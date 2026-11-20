# Multithreading: Threads

Running work at the same time: what threads are, the `Thread` class and its methods, thread states, passing data in and out, and the thread pool. Synchronization and concurrent collections are on the next tab.

## 1. Concurrent Execution and Threading

**Concurrent execution** means making progress on several tasks during the same period, so a slow task (a file download, a database call) doesn't freeze everything else. A **thread** is the unit of execution the OS schedules on a CPU core; a program can run many threads at once.

| Term | Meaning |
| --- | --- |
| Process | A running program with its own memory space (e.g. `ClaimsApp.exe`) |
| Thread | A path of execution inside a process; threads of one process **share** its memory |
| Main thread | The first thread, which runs `Main` |
| Concurrency | Tasks overlap in time (may take turns on one core) |
| Parallelism | Tasks run at literally the same instant on different cores |
| Context switch | The OS pausing one thread and resuming another |

**Why use threads:** keep UIs responsive; do several independent jobs at once (process files, call APIs); use all CPU cores for heavy calculations; serve many users (web servers).

**Costs:** each thread uses memory (about 1 MB stack by default) and switching costs CPU; shared data needs synchronization (next tab); bugs like race conditions and deadlocks are hard to reproduce.

**Where threading fits in modern .NET:** `Thread` (this tab) is the low-level building block. Most new code uses the **thread pool** via `Task`/`Task.Run`, `async`/`await` for I/O, and `Parallel`/PLINQ for CPU work. Learning `Thread` first makes those easier to understand.

## 2. Main Thread, Thread Class, Single-Threaded vs Multiple Threads

### Main thread

```csharp
using System.Threading;

Thread main = Thread.CurrentThread;
main.Name = "Main";
Console.WriteLine($"{main.Name}, id {main.ManagedThreadId}, priority {main.Priority}");
Console.WriteLine(main.IsBackground);   // False: the app stays alive while it runs
```

### The Thread class

```csharp
void PrintNumbers()
{
    for (int i = 1; i <= 3; i++)
        Console.WriteLine($"{Thread.CurrentThread.Name}: {i}");
}

var t = new Thread(PrintNumbers)    // ThreadStart: void method, no parameters
{
    Name = "Worker",
    IsBackground = false            // foreground (default): app waits for it to finish
};
t.Start();                          // begins running PrintNumbers on the new thread
```

| Member | Purpose |
| --- | --- |
| `new Thread(method)` | Create (doesn't start yet) |
| `Start()` / `Start(obj)` | Begin running |
| `Join()` | Wait for it to finish |
| `Thread.Sleep(ms)` | Pause the current thread |
| `Name`, `ManagedThreadId` | Identify it |
| `IsAlive`, `ThreadState` | Is it running? Which state? |
| `IsBackground` | Background threads are killed when the app's foreground threads end |
| `Priority` | Scheduling hint |
| `Interrupt()` | Wake it from Sleep/Join/Wait with an exception |
| `Thread.CurrentThread` | The thread running this code |

### Single-threaded app

```csharp
var sw = Stopwatch.StartNew();
DownloadReport("Jan");      // 2 s
DownloadReport("Feb");      // 2 s
DownloadReport("Mar");      // 2 s
Console.WriteLine(sw.Elapsed);   // ~6 s: each waits for the previous one

void DownloadReport(string month) { Thread.Sleep(2000); Console.WriteLine($"{month} done"); }
```

### Multiple threads

```csharp
var sw = Stopwatch.StartNew();
var threads = new List<Thread>();
foreach (var m in new[] { "Jan", "Feb", "Mar" })
{
    var th = new Thread(() => DownloadReport(m)) { Name = m };
    threads.Add(th);
    th.Start();
}
foreach (var th in threads) th.Join();     // wait for all
Console.WriteLine(sw.Elapsed);             // ~2 s: they ran at the same time
// Output order varies run to run: Feb done, Jan done, Mar done ...
```

**Key point:** the OS decides the order; never assume threads run or finish in the order you started them.

## 3. Sleep, Join, Priority and Interrupt

### Sleep

`Thread.Sleep(ms)` pauses **the current thread** (it's static) and gives the CPU to others.

```csharp
for (int i = 3; i > 0; i--)
{
    Console.WriteLine(i);
    Thread.Sleep(1000);            // 1 second
}
Thread.Sleep(0);                   // yield the rest of this time slice
Thread.Sleep(TimeSpan.FromSeconds(2));
```

In async code use `await Task.Delay(ms)` instead: it doesn't block a thread.

### Join

`t.Join()` blocks the calling thread until `t` finishes.

```csharp
var worker = new Thread(() => { Thread.Sleep(2000); Console.WriteLine("Worker done"); });
worker.Start();
Console.WriteLine("Main waiting...");
worker.Join();                              // Main pauses here
Console.WriteLine("Main continues");        // always after "Worker done"

bool finished = worker.Join(500);           // wait at most 500 ms; false if still running
```

### Priority

A **hint** to the OS scheduler: `Lowest`, `BelowNormal`, `Normal` (default), `AboveNormal`, `Highest`.

```csharp
var report = new Thread(BuildReport) { Priority = ThreadPriority.BelowNormal };
var ui     = new Thread(RefreshScreen) { Priority = ThreadPriority.AboveNormal };
```

Priority only matters when cores are busy; results differ by OS and core count. Don't rely on it for correctness, and avoid `Highest` (it can starve other threads).

### Interrupt

`t.Interrupt()` wakes a thread that is (or next becomes) blocked in `Sleep`, `Join` or `Monitor.Wait` by throwing `ThreadInterruptedException` inside it.

```csharp
var sleeper = new Thread(() =>
{
    try
    {
        Console.WriteLine("Sleeping for 10 s");
        Thread.Sleep(10_000);
    }
    catch (ThreadInterruptedException)
    {
        Console.WriteLine("Woken up early");
    }
});
sleeper.Start();
Thread.Sleep(1000);
sleeper.Interrupt();             // "Woken up early" after ~1 s
```

If the thread is running (not blocked), the interrupt waits until it next blocks. `Thread.Abort` is obsolete and throws `PlatformNotSupportedException` in .NET Core/5+; for cooperative stopping use a `CancellationToken` or a flag.

## 4. Thread States

A thread moves through a small set of states during its life, reported by `t.ThreadState` (a flags enum) and summarised by `t.IsAlive`.

&#91;embedded content: Thread states\]

A thread can go between Running and WaitSleepJoin many times, but once Stopped it can never be started again: calling `Start()` twice throws `ThreadStateException`.

```csharp
var t = new Thread(() => Thread.Sleep(500));
Console.WriteLine(t.ThreadState);   // Unstarted
t.Start();
Thread.Sleep(100);
Console.WriteLine(t.ThreadState);   // WaitSleepJoin (it's inside Sleep)
t.Join();
Console.WriteLine(t.ThreadState);   // Stopped
Console.WriteLine(t.IsAlive);       // False
```

| State | Meaning |
| --- | --- |
| `Unstarted` | Created, `Start()` not called yet |
| `Running` | Started and not blocked (may be waiting for a CPU core) |
| `WaitSleepJoin` | Blocked in `Sleep`, `Join`, `Monitor.Wait`, a lock or a wait handle |
| `Stopped` | Method finished (normally or by an unhandled exception) |
| `Background` | Flag combined with others when `IsBackground = true` |
| `SuspendRequested`, `Suspended`, `AbortRequested`, `Aborted` | Legacy, from obsolete `Suspend`/`Abort` |

**An unhandled exception in any thread crashes the whole process**, so wrap a thread's work in try/catch.

## 5. Thread Parameters, ParameterizedThreadStart, Custom Thread Object and Callbacks

`ThreadStart` takes no arguments and returns nothing. These are the ways to pass data **into** a thread and get results **out**.

### ParameterizedThreadStart: one `object` argument

```csharp
void Download(object? state)
{
    string month = (string)state!;               // must cast
    Console.WriteLine($"Downloading {month}");
}

var t = new Thread(Download);                    // matches ParameterizedThreadStart
t.Start("Jan");                                  // the argument goes to Start

// several values: wrap them in an object or tuple
var t2 = new Thread(o => { var (m, y) = ((string, int))o!; Console.WriteLine($"{m} {y}"); });
t2.Start(("Feb", 2026));
```

Drawback: not type-safe (`object` + cast).

### Lambda capture (simplest, type-safe)

```csharp
string month = "Mar"; int year = 2026;
var t3 = new Thread(() => DownloadReport(month, year));   // pass any number of typed values
t3.Start();

// Classic bug: capturing a loop variable that keeps changing
for (int i = 0; i < 3; i++)
{
    int copy = i;                                       // take a copy per iteration
    new Thread(() => Console.WriteLine(copy)).Start();  // without 'copy' you may print 3, 3, 3
}
```

(`foreach` variables are safe since C# 5; `for` counters are not.)

### Custom thread object: data and result in a class

```csharp
public class ReportJob
{
    public string Month { get; }
    public int RowCount { get; private set; }      // result
    public ReportJob(string month) => Month = month;

    public void Run()                              // ThreadStart-compatible instance method
    {
        Thread.Sleep(1000);
        RowCount = Month.Length * 100;
    }
}

var job = new ReportJob("April");
var t4 = new Thread(job.Run);
t4.Start();
t4.Join();                                         // wait, then read the result safely
Console.WriteLine($"{job.Month}: {job.RowCount} rows");   // April: 500 rows
```

Type-safe, holds many inputs and outputs, and the result is safe to read after `Join`.

### Callback: the thread reports back when done

```csharp
public class SumJob
{
    private readonly int[] _numbers;
    private readonly Action<int> _onDone;                // callback delegate

    public SumJob(int[] numbers, Action<int> onDone) { _numbers = numbers; _onDone = onDone; }

    public void Run()
    {
        int total = _numbers.Sum();
        _onDone(total);                                  // "call back" with the result
    }
}

var job2 = new SumJob(new[] { 1, 2, 3, 4 }, total => Console.WriteLine($"Sum = {total}"));
new Thread(job2.Run).Start();                            // prints Sum = 10 when finished
Console.WriteLine("Main keeps working...");
```

**Note:** the callback runs **on the worker thread**, not the main thread, so anything it touches must be thread-safe (and UI apps must marshal back to the UI thread). Modern code returns results with `Task<T>` instead: `int total = await Task.Run(() => numbers.Sum());`.

| Technique | Type-safe | Many inputs | Gets a result back |
| --- | --- | --- | --- |
| `ParameterizedThreadStart` | No (`object`) | Wrap them | No |
| Lambda capture | Yes | Yes | Via a captured variable (after Join) |
| Custom thread object | Yes | Yes | Properties, read after Join |
| Callback delegate | Yes | Yes | Pushed to you when done |
| `Task<T>` (modern) | Yes | Yes | `await` / `.Result` |

## 6. Thread Pool

Creating a `Thread` is expensive (stack memory, OS setup). The **thread pool** keeps a set of ready, reusable **background** worker threads and hands them short work items. .NET grows and shrinks the pool automatically. `Task.Run`, `async`/`await` continuations, `Parallel`, timers and ASP.NET Core requests all run on it.

```csharp
// Classic API
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine($"Processing {state} on pool thread {Thread.CurrentThread.ManagedThreadId}, " +
                      $"pool? {Thread.CurrentThread.IsThreadPoolThread}");
}, "Claim C1");

for (int i = 1; i <= 5; i++)
{
    int n = i;
    ThreadPool.QueueUserWorkItem(_ => Console.WriteLine($"Item {n}"));   // reuse a few threads for many items
}

ThreadPool.GetMinThreads(out int minWorkers, out int minIo);
ThreadPool.GetAvailableThreads(out int freeWorkers, out int freeIo);

// Modern way: Task.Run uses the pool and gives you a result, exceptions and await
int total = await Task.Run(() => Enumerable.Range(1, 1_000_000).Sum());
```

Pool threads are background threads: they're stopped when the app's last foreground thread ends, so `Join` isn't available. Wait for work with `Task`/`await`, `CountdownEvent` or a `ManualResetEvent` (next tab).

|  | `new Thread` | Thread pool (`QueueUserWorkItem` / `Task.Run`) |
| --- | --- | --- |
| Creation cost | High every time | Threads reused |
| Foreground/background | Your choice (foreground default) | Always background |
| Name, priority, Join | Yes | No (don't change pool threads) |
| Best for | Long-running dedicated work, special settings | Many short tasks |
| Results / exceptions | Manual | Easy with `Task<T>` |

**Don't block pool threads** for long (`Thread.Sleep`, synchronous I/O); it starves other work. For a long-running job use `new Thread` or `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)`.

## 7. Cheat Sheet and Interview Questions

```csharp
var t = new Thread(Work) { Name = "W1", IsBackground = true, Priority = ThreadPriority.Normal };
t.Start();  t.Join();  t.Join(500);  t.Interrupt();  bool alive = t.IsAlive;
Thread.Sleep(1000);  Thread.CurrentThread.ManagedThreadId;
new Thread(o => Use((string)o!)).Start("data");          // ParameterizedThreadStart
new Thread(() => Use(x, y)).Start();                     // lambda capture
var job = new Job(); new Thread(job.Run).Start();        // custom thread object
new Thread(() => onDone(Compute())).Start();             // callback
ThreadPool.QueueUserWorkItem(_ => Work());
int r = await Task.Run(() => Compute());                 // modern
```

**Checklist**

- [ ] Explain process vs thread and concurrency vs parallelism
- [ ] Name and configure the main thread
- [ ] Compare a single-threaded and a multi-threaded run with Stopwatch
- [ ] Use Sleep, Join (with timeout), Priority and Interrupt
- [ ] Draw the thread state diagram
- [ ] Pass data with ParameterizedThreadStart, a lambda and a custom object
- [ ] Return a result with a callback
- [ ] Explain when to use the thread pool

**Interview questions (short answers)**

1. **Process vs thread?** A process has its own memory; threads run inside a process and share its memory.
2. **What is multithreading?** Running several threads concurrently within one process.
3. **Concurrency vs parallelism?** Overlapping progress vs truly simultaneous execution on multiple cores.
4. **Foreground vs background thread?** The process waits for foreground threads; background threads are killed when the last foreground thread ends.
5. **What does Join do?** Blocks the caller until the thread finishes.
6. **Sleep vs Join?** Sleep pauses the current thread for a time; Join waits for another thread to end.
7. **Does a higher priority guarantee earlier completion?** No, it's only a scheduler hint.
8. **What does Interrupt do?** Throws ThreadInterruptedException in a thread blocked in Sleep/Join/Wait.
9. **Can you restart a stopped thread?** No; create a new Thread.
10. **How do you pass parameters to a thread?** `Start(object)`, a lambda capturing variables, or an instance method of a custom class.
11. **How do you get a result from a thread?** Read a field after Join, use a callback, or use `Task<T>`.
12. **What is the thread pool?** A managed set of reusable background threads for short work items.
13. **Thread vs Task?** Thread is a low-level OS thread; Task is a higher-level unit of work (usually on the pool) with results, exceptions, continuations and await.
14. **What happens on an unhandled exception in a thread?** The whole process terminates.
