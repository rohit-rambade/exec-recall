# Thread Synchronization & Concurrent Collections

Keeping shared data correct when several threads touch it: locks, signalling with events and Wait/Pulse, semaphores, mutexes, countdown events, and thread-safe collections, with two practice projects.

## 1. Shared Resources, Race Conditions and the Synchronization Toolbox

Threads in one process share memory. When two threads read and write the same data at the same time, the result depends on timing, which is a **race condition**. Even `count++` is three steps (read, add, write), so updates get lost.

```csharp
int counter = 0;
void Work() { for (int i = 0; i < 100_000; i++) counter++; }   // NOT atomic

var t1 = new Thread(Work); var t2 = new Thread(Work);
t1.Start(); t2.Start(); t1.Join(); t2.Join();
Console.WriteLine(counter);     // expected 200000, often prints something like 136482
```

**Thread synchronization** means coordinating threads so shared data stays correct: allowing only one thread into a **critical section** at a time (mutual exclusion), or making threads wait for a **signal** from another.

| Tool | Kind | What it does | Cross-process |
| --- | --- | --- | --- |
| `lock` / `Monitor` | Lock | One thread at a time in a code block | No |
| `Monitor.Wait` / `Pulse` | Signalling | Wait inside a lock until another thread pulses | No |
| `ManualResetEvent(Slim)` | Signalling | A gate: Set opens for everyone until Reset | Yes (non-Slim) |
| `AutoResetEvent` | Signalling | A turnstile: Set lets exactly one waiter through, then closes | Yes |
| `SemaphoreSlim` / `Semaphore` | Throttle | At most N threads at once | Semaphore only |
| `Mutex` | Lock | Like lock, but can work across processes | Yes |
| `CountdownEvent` | Signalling | Wait until N signals have arrived | No |
| `Interlocked` | Atomic op | Thread-safe `Increment`, `Add`, `Exchange` on one variable | No |
| Concurrent collections | Data structure | Thread-safe queue, dictionary, bag, stack | No |

**Quick fix for the counter:** `Interlocked.Increment(ref counter);` or put `counter++` inside a `lock`.

**Hazards to remember:** **deadlock** (two threads each hold a lock the other needs, so both wait forever), **starvation** (a thread never gets a turn), and **over-locking** (everything serialised, so no benefit from threads).

## 2. Monitor and lock

### Monitor

`Monitor.Enter(obj)` takes an exclusive lock on an object; any other thread calling `Enter` on the same object **waits** until the owner calls `Monitor.Exit(obj)`.

```csharp
private static readonly object _sync = new();
private static int _counter;

void Work()
{
    for (int i = 0; i < 100_000; i++)
    {
        bool taken = false;
        try
        {
            Monitor.Enter(_sync, ref taken);     // acquire
            _counter++;                          // critical section: one thread at a time
        }
        finally
        {
            if (taken) Monitor.Exit(_sync);      // ALWAYS release, even on exceptions
        }
    }
}
// now the result is always 200000

if (Monitor.TryEnter(_sync, TimeSpan.FromSeconds(1)))   // give up instead of waiting forever
{
    try { /* work */ } finally { Monitor.Exit(_sync); }
}
```

### lock (shorthand for Monitor)

```csharp
lock (_sync)
{
    _counter++;
}
// compiles to exactly the Monitor.Enter / try / finally / Monitor.Exit pattern above
```

**Rules for the lock object**

- Use a **private, readonly, reference-type** object dedicated to locking: `private readonly object _sync = new();`.
- Never lock on `this`, a `Type` (`typeof(X)`), a string literal or a public object: outside code could lock on it too and cause deadlocks.
- Never lock on a value type (it gets boxed into a new object each time, so nothing is actually locked; the compiler rejects `lock(int)`).
- Locks are **reentrant**: the same thread can enter a lock it already holds.
- Keep critical sections short; never `await` inside `lock` (compile error); use `SemaphoreSlim` for async.
- .NET 9 adds `System.Threading.Lock` (`private readonly Lock _lock = new(); lock (_lock) { }`), a faster dedicated lock type.

**Avoiding deadlocks:** always take multiple locks in the same order; prefer one lock; use `TryEnter` with a timeout; don't call unknown code (events, callbacks) while holding a lock.

## 3. ManualResetEvent and AutoResetEvent

Event wait handles let threads **wait for a signal**. Think of a gate: `WaitOne()` waits at the gate, `Set()` opens it, `Reset()` closes it.

### ManualResetEvent, part 1: one signal releases everyone

```csharp
var ready = new ManualResetEvent(false);     // false = start closed

for (int i = 1; i <= 3; i++)
{
    int id = i;
    new Thread(() =>
    {
        Console.WriteLine($"Worker {id} waiting for config");
        ready.WaitOne();                     // blocks until the gate opens
        Console.WriteLine($"Worker {id} started");
    }).Start();
}

Thread.Sleep(2000);                          // pretend to load configuration
Console.WriteLine("Config loaded");
ready.Set();                                 // opens the gate: ALL 3 workers continue
```

### Part 2: the gate stays open until Reset

```csharp
ready.Set();
ready.WaitOne();        // returns immediately: still open
ready.Reset();          // close again; later WaitOne() calls block
bool signalled = ready.WaitOne(TimeSpan.FromSeconds(5));   // timeout version: false if not set in time
```

### Part 3: pause and resume workers

```csharp
var running = new ManualResetEventSlim(true);    // Slim: faster, in-process only
var cts = new CancellationTokenSource();

var worker = new Thread(() =>
{
    while (!cts.IsCancellationRequested)
    {
        running.Wait();                          // blocks while paused
        Console.WriteLine($"Processing claim at {DateTime.Now:T}");
        Thread.Sleep(500);
    }
});
worker.Start();

Thread.Sleep(2000); running.Reset();   Console.WriteLine("Paused");
Thread.Sleep(2000); running.Set();     Console.WriteLine("Resumed");
Thread.Sleep(1000); cts.Cancel();      worker.Join();
```

### AutoResetEvent: one signal releases ONE waiter

```csharp
var turnstile = new AutoResetEvent(false);

for (int i = 1; i <= 3; i++)
{
    int id = i;
    new Thread(() => { turnstile.WaitOne(); Console.WriteLine($"Customer {id} served"); }).Start();
}

for (int i = 0; i < 3; i++)
{
    Thread.Sleep(1000);
    turnstile.Set();        // lets exactly one waiting thread through, then auto-closes
}
// Customers are served one per second
```

|  | ManualResetEvent | AutoResetEvent |
| --- | --- | --- |
| Analogy | Gate | Turnstile |
| One `Set()` releases | All waiting threads (and later ones) | Exactly one thread |
| After releasing | Stays open until `Reset()` | Closes automatically |
| Typical use | "Start now", pause/resume, "initialisation done" | Hand work items over one at a time |
| Lightweight version | `ManualResetEventSlim` | (use `SemaphoreSlim(0)`) |

Both derive from `WaitHandle`, so `WaitHandle.WaitAll(handles)` / `WaitAny(handles)` can wait on several at once. Dispose them when done (`using`).

## 4. Wait and Pulse; Monitor with ManualResetEvent

`Monitor.Wait` and `Monitor.Pulse` let threads that share a lock **signal each other**. A thread inside the lock calls `Wait` to release the lock and sleep until another thread calls `Pulse`.

### Part 1: the three methods

| Method | Must hold the lock? | Effect |
| --- | --- | --- |
| `Monitor.Wait(obj)` | Yes | Releases the lock, blocks until pulsed, then **re-acquires the lock** before returning |
| `Monitor.Pulse(obj)` | Yes | Moves **one** waiting thread to the ready queue (it runs once the lock is free) |
| `Monitor.PulseAll(obj)` | Yes | Moves **all** waiting threads to the ready queue |

Calling them without holding the lock throws `SynchronizationLockException`.

### Part 2: ping-pong (two threads taking turns)

```csharp
object sync = new();
bool pingTurn = true;

void Player(string word, bool isPing)
{
    for (int i = 0; i < 3; i++)
    {
        lock (sync)
        {
            while (pingTurn != isPing) Monitor.Wait(sync);   // not my turn: wait
            Console.WriteLine(word);
            pingTurn = !pingTurn;                            // hand over the turn
            Monitor.Pulse(sync);                             // wake the other player
        }
    }
}
var a = new Thread(() => Player("Ping", true));
var b = new Thread(() => Player("Pong", false));
a.Start(); b.Start(); a.Join(); b.Join();      // Ping Pong Ping Pong Ping Pong
```

### Part 3: producer-consumer with a bounded buffer

```csharp
public class ClaimBuffer
{
    private readonly Queue<string> _queue = new();
    private readonly object _sync = new();
    private const int Capacity = 3;

    public void Produce(string claim)
    {
        lock (_sync)
        {
            while (_queue.Count == Capacity) Monitor.Wait(_sync);   // buffer full: wait
            _queue.Enqueue(claim);
            Console.WriteLine($"Produced {claim}");
            Monitor.PulseAll(_sync);                                 // wake consumers
        }
    }

    public string Consume()
    {
        lock (_sync)
        {
            while (_queue.Count == 0) Monitor.Wait(_sync);           // buffer empty: wait
            string claim = _queue.Dequeue();
            Monitor.PulseAll(_sync);                                 // wake producers
            return claim;
        }
    }
}

var buffer = new ClaimBuffer();
var producer = new Thread(() => { for (int i = 1; i <= 6; i++) buffer.Produce($"C{i}"); });
var consumer = new Thread(() => { for (int i = 1; i <= 6; i++) { Console.WriteLine($"  Consumed {buffer.Consume()}"); Thread.Sleep(300); } });
producer.Start(); consumer.Start(); producer.Join(); consumer.Join();
```

### Part 4: rules that prevent bugs

- Always call `Wait` inside a **`while`** loop that rechecks the condition (not `if`): another thread may have changed it before you got the lock back, and spurious wake-ups can happen.
- `Pulse` isn't remembered: if nobody is waiting yet, the signal is lost. That's why the condition variable (`pingTurn`, `Count`) is the real source of truth.
- Prefer `PulseAll` when producers and consumers wait on the same object, so the right kind of thread wakes up.
- Modern alternative: `BlockingCollection<T>` or `System.Threading.Channels` implement producer-consumer for you.

### Monitor with ManualResetEvent

Combine them when a group of workers must **wait to start** (event) and then **safely share data** (lock):

```csharp
var start = new ManualResetEvent(false);
object sync = new();
int processed = 0;

var workers = Enumerable.Range(1, 4).Select(id => new Thread(() =>
{
    start.WaitOne();                                // all wait for the starting signal
    for (int i = 0; i < 1000; i++)
        lock (sync) processed++;                    // safe shared update
})).ToList();

workers.ForEach(w => w.Start());
Console.WriteLine("All workers ready...");
start.Set();                                        // release them together
workers.ForEach(w => w.Join());
Console.WriteLine(processed);                       // always 4000
```

## 5. Semaphore, Mutex and CountdownEvent

### Semaphore: allow at most N threads at once

A semaphore holds a count of free slots. `WaitOne()` / `Wait()` takes a slot (blocking if none are free); `Release()` returns one. Use it to throttle access to a limited resource (database connections, API rate limits, printers).

```csharp
var pool = new SemaphoreSlim(initialCount: 3, maxCount: 3);   // 3 concurrent downloads max

for (int i = 1; i <= 8; i++)
{
    int id = i;
    new Thread(() =>
    {
        pool.Wait();                                 // take a slot
        try
        {
            Console.WriteLine($"Download {id} started (free slots: {pool.CurrentCount})");
            Thread.Sleep(1000);
        }
        finally { pool.Release(); }                  // always give it back
    }).Start();
}
// Downloads run in waves of 3

await pool.WaitAsync();                              // SemaphoreSlim also works with async/await
var named = new Semaphore(2, 2, "Global\\ClaimsPrinters");   // Semaphore (non-Slim) can be shared across processes
```

### Mutex: a lock that can work across processes

Only one thread (in any process, if named) can own it. The owner must release it. Classic use: allow only **one instance** of an application.

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: "Global\\HealthIndia.ClaimsApp");

if (!mutex.WaitOne(TimeSpan.Zero))         // try to take it without waiting
{
    Console.WriteLine("The app is already running.");
    return;
}
try
{
    Console.WriteLine("Running... press Enter to exit");
    Console.ReadLine();
}
finally { mutex.ReleaseMutex(); }          // only the owning thread can release
```

Within one process, prefer `lock` (much faster). A mutex abandoned by a crashed owner throws `AbandonedMutexException` to the next waiter.

### CountdownEvent: wait until N things have finished

```csharp
string[] files = { "jan.csv", "feb.csv", "mar.csv" };
using var done = new CountdownEvent(files.Length);    // count = 3

foreach (var f in files)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        try { Console.WriteLine($"Processed {f}"); Thread.Sleep(500); }
        finally { done.Signal(); }                    // count - 1
    });
}

done.Wait();                                          // blocks until count reaches 0
Console.WriteLine("All files processed");
// done.AddCount(); done.Reset(); done.CurrentCount;
```

Perfect for waiting on thread-pool work (which has no `Join`).

|  | `lock` / Monitor | Mutex | SemaphoreSlim / Semaphore | CountdownEvent |
| --- | --- | --- | --- | --- |
| Threads allowed in | 1 | 1 | N | (not a lock) |
| Cross-process | No | Yes (named) | Semaphore only | No |
| Owner must release | Yes | Yes | Any thread may release | n/a |
| Async support | No | No | `SemaphoreSlim.WaitAsync` | No |
| Speed | Fastest | Slow (kernel) | Slim fast / Semaphore slow | Fast |
| Typical use | Protect shared data | Single app instance | Throttle concurrency | Wait for N tasks |

## 6. Concurrent Collections

`List<T>`, `Dictionary<K,V>` and `Queue<T>` are **not thread-safe**: concurrent writes can corrupt them or throw. `System.Collections.Concurrent` provides collections that handle locking internally (often lock-free), so many threads can use them safely without your own `lock`.

### ConcurrentQueue\<T>

```csharp
using System.Collections.Concurrent;

var queue = new ConcurrentQueue<string>();

var producers = Enumerable.Range(1, 3).Select(p => new Thread(() =>
{
    for (int i = 1; i <= 5; i++) queue.Enqueue($"P{p}-Claim{i}");   // thread-safe add
})).ToList();
producers.ForEach(t => t.Start());
producers.ForEach(t => t.Join());

int handled = 0;
var consumers = Enumerable.Range(1, 2).Select(c => new Thread(() =>
{
    while (queue.TryDequeue(out string? claim))                       // thread-safe take
        Interlocked.Increment(ref handled);
})).ToList();
consumers.ForEach(t => t.Start());
consumers.ForEach(t => t.Join());
Console.WriteLine(handled);                                             // 15, every time

queue.TryPeek(out var next);  bool empty = queue.IsEmpty;  int n = queue.Count;
```

Note the `Try...` methods: with many threads, "check then act" (`if (Count > 0) Dequeue()`) is unsafe, so the check and the action happen in one atomic call.

### Other concurrent collections

```csharp
// ConcurrentDictionary: atomic add/update
var visits = new ConcurrentDictionary<string, int>();
visits.AddOrUpdate("Pune", 1, (city, old) => old + 1);     // insert or increment atomically
int cnt = visits.GetOrAdd("Delhi", 0);
visits.TryAdd("Goa", 5);  visits.TryRemove("Goa", out _);  visits.TryUpdate("Pune", 10, 2);

// ConcurrentStack: LIFO
var stack = new ConcurrentStack<int>();
stack.Push(1); stack.PushRange(new[] { 2, 3 }); stack.TryPop(out int top);

// ConcurrentBag: unordered, fastest when the same threads add and take
var bag = new ConcurrentBag<string>();
bag.Add("x"); bag.TryTake(out var item);

// BlockingCollection: producer-consumer with blocking and bounded capacity
using var jobs = new BlockingCollection<string>(boundedCapacity: 100);
var consumer = new Thread(() =>
{
    foreach (var job in jobs.GetConsumingEnumerable())   // blocks until items arrive, ends after CompleteAdding
        Console.WriteLine($"Processing {job}");
});
consumer.Start();
for (int i = 1; i <= 3; i++) jobs.Add($"Job{i}");      // blocks if full
jobs.CompleteAdding();
consumer.Join();
```

| Collection | Like | Order | Key methods |
| --- | --- | --- | --- |
| `ConcurrentQueue<T>` | `Queue<T>` | FIFO | `Enqueue`, `TryDequeue`, `TryPeek` |
| `ConcurrentStack<T>` | `Stack<T>` | LIFO | `Push`, `TryPop`, `PushRange` |
| `ConcurrentBag<T>` | Unordered list | None | `Add`, `TryTake` |
| `ConcurrentDictionary<K,V>` | `Dictionary<K,V>` | None | `TryAdd`, `GetOrAdd`, `AddOrUpdate`, `TryRemove` |
| `BlockingCollection<T>` | Wrapper (queue by default) | FIFO | `Add`, `Take`, `GetConsumingEnumerable`, `CompleteAdding` |

**Caveats:** individual operations are thread-safe, but a **sequence** of operations isn't automatically atomic; `GetOrAdd`'s value factory may run more than once under contention; `Count` and enumeration are snapshots. For async pipelines, `System.Threading.Channels` is the modern choice.

## 7. Projects: Multi-Threaded Banking Simulation and CSV Processing with Threads

### Multi-threaded banking operations simulation

Goal: many threads deposit, withdraw and transfer at the same time; the total money in the bank must never change and no balance may go negative.

```csharp
public class Account
{
    private readonly object _sync = new();
    public int Id { get; }
    public decimal Balance { get; private set; }
    public Account(int id, decimal opening) { Id = id; Balance = opening; }

    public void Deposit(decimal amt) { lock (_sync) Balance += amt; }

    public bool Withdraw(decimal amt)
    {
        lock (_sync)
        {
            if (amt > Balance) return false;      // check and update in ONE lock: no overdraw
            Balance -= amt;
            return true;
        }
    }

    // Transfer: lock both accounts, ALWAYS in Id order to avoid deadlock
    public static bool Transfer(Account from, Account to, decimal amt)
    {
        var first  = from.Id < to.Id ? from : to;
        var second = from.Id < to.Id ? to : from;
        lock (first._sync)
        lock (second._sync)
        {
            if (from.Balance < amt) return false;
            from.Balance -= amt;
            to.Balance += amt;
            return true;
        }
    }
}

var accounts = Enumerable.Range(1, 5).Select(i => new Account(i, 1000m)).ToArray();
decimal before = accounts.Sum(a => a.Balance);              // 5000

var threads = Enumerable.Range(0, 10).Select(_ => new Thread(() =>
{
    var rnd = new Random();
    for (int i = 0; i < 1000; i++)
    {
        var a = accounts[rnd.Next(5)];
        var b = accounts[rnd.Next(5)];
        if (a != b) Account.Transfer(a, b, rnd.Next(1, 100));
    }
})).ToList();

threads.ForEach(t => t.Start());
threads.ForEach(t => t.Join());
Console.WriteLine($"Before {before}, after {accounts.Sum(a => a.Balance)}");   // always equal
Console.WriteLine(accounts.All(a => a.Balance >= 0));                         // True
```

**What it teaches:** lock per account (not one global lock) for better concurrency; check-then-act inside the same lock; consistent lock ordering prevents the A→B / B→A transfer deadlock. Remove the locks and the totals drift: that's the race condition made visible.

### CSV processing with threads (parts 1 to 4)

Goal: read a large claims CSV, process rows on several worker threads, and write results safely.

```csharp
// Part 1: split the work: one reader thread, N worker threads, one shared output
using var lines = new BlockingCollection<string>(boundedCapacity: 1000);
var results = new ConcurrentBag<(string Id, decimal Payable)>();
int errors = 0;

// Part 2: the reader (producer) streams lines without loading the whole file
var reader = new Thread(() =>
{
    foreach (var line in File.ReadLines("claims.csv").Skip(1))   // skip header
        lines.Add(line);
    lines.CompleteAdding();                                      // tell workers no more data
});

// Part 3: workers (consumers) parse and calculate in parallel
var workers = Enumerable.Range(1, Environment.ProcessorCount).Select(_ => new Thread(() =>
{
    foreach (var line in lines.GetConsumingEnumerable())
    {
        var cols = line.Split(',');
        if (cols.Length < 3 || !decimal.TryParse(cols[2], out var amount))
        {
            Interlocked.Increment(ref errors);                  // thread-safe counter
            continue;
        }
        decimal payable = amount * 0.9m;                         // 10% co-pay
        results.Add((cols[0], payable));
    }
})).ToList();

reader.Start();
workers.ForEach(w => w.Start());
reader.Join();
workers.ForEach(w => w.Join());

// Part 4: write output once all workers are done (single thread, no locking needed)
using var writer = new StreamWriter("payables.csv");
writer.WriteLine("Id,Payable");
foreach (var r in results.OrderBy(r => r.Id))
    writer.WriteLine($"{r.Id},{r.Payable:F2}");
Console.WriteLine($"{results.Count} processed, {errors} bad rows");
```

**Design points:** bounded queue so the reader can't run far ahead and fill memory; worker count = CPU cores for CPU-bound work; thread-safe output (`ConcurrentBag`) or a single writer thread; count errors with `Interlocked`; order results at the end because threads finish in any order. In modern code the same pipeline can be written with `Parallel.ForEach(File.ReadLines(...), ...)` or Channels.

## 8. Cheat Sheet and Interview Questions

```csharp
lock (_sync) { shared++; }                               // mutual exclusion
Interlocked.Increment(ref counter);                      // atomic counter
Monitor.Enter(o); try { } finally { Monitor.Exit(o); }   // what lock compiles to
lock (o) { while (!ready) Monitor.Wait(o); }             // wait for a condition
lock (o) { ready = true; Monitor.PulseAll(o); }          // signal it
var gate = new ManualResetEvent(false); gate.WaitOne(); gate.Set(); gate.Reset();
var turn = new AutoResetEvent(false); turn.WaitOne(); turn.Set();
var sem = new SemaphoreSlim(3); sem.Wait(); try { } finally { sem.Release(); }
using var m = new Mutex(false, "Global\\App"); m.WaitOne(); m.ReleaseMutex();
var cd = new CountdownEvent(3); cd.Signal(); cd.Wait();
var q = new ConcurrentQueue<int>(); q.Enqueue(1); q.TryDequeue(out var x);
var d = new ConcurrentDictionary<string, int>(); d.AddOrUpdate("k", 1, (_, v) => v + 1);
```

**Checklist**

- [ ] Reproduce a race condition and fix it with lock and with Interlocked
- [ ] Explain what lock compiles to and the rules for the lock object
- [ ] Release a group with ManualResetEvent; serve one at a time with AutoResetEvent
- [ ] Write ping-pong and a bounded producer-consumer with Wait/PulseAll
- [ ] Throttle with SemaphoreSlim; single-instance app with Mutex; wait for N with CountdownEvent
- [ ] Use ConcurrentQueue, ConcurrentDictionary and BlockingCollection
- [ ] Build the banking simulation with deadlock-free transfers
- [ ] Build the multi-threaded CSV pipeline

**Interview questions (short answers)**

1. **What is a race condition?** A bug where the result depends on the timing of threads accessing shared data.
2. **What is a critical section?** Code that only one thread should execute at a time.
3. **lock vs Monitor?** `lock` is shorthand for Monitor.Enter/Exit in try/finally; Monitor adds TryEnter, Wait and Pulse.
4. **Why not lock(this)?** Outside code can lock the same object and cause deadlocks; use a private object.
5. **What is a deadlock and how do you avoid it?** Threads waiting on each other's locks forever; take locks in a consistent order, keep locks short, use timeouts.
6. **Wait vs Sleep?** Monitor.Wait releases the lock and waits for a pulse; Sleep keeps any locks and just pauses.
7. **Pulse vs PulseAll?** Wakes one waiter vs all waiters.
8. **ManualResetEvent vs AutoResetEvent?** Manual stays signalled and releases all until Reset; Auto releases one waiter and resets itself.
9. **Semaphore vs Mutex?** Semaphore allows N threads; Mutex allows one and is owned (must be released by the owner).
10. **Mutex vs lock?** Mutex can work across processes but is slower; lock is in-process only.
11. **What is CountdownEvent?** A signal that's set after a given number of Signal() calls.
12. **Why concurrent collections?** Standard collections aren't thread-safe; concurrent ones handle synchronization internally.
13. **What is Interlocked?** Atomic operations (Increment, Add, Exchange, CompareExchange) on shared variables without a lock.
14. **What is the producer-consumer pattern?** Producers add work to a shared buffer; consumers take it; coordination via Wait/Pulse, BlockingCollection or Channels.
