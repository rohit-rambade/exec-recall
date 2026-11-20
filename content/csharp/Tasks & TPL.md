# Tasks & the Task Parallel Library (TPL)

The higher-level way to run work concurrently: `Task` and `Task<T>`, waiting, continuations, exceptions, cancellation and file I/O, building on the Threads tabs.

## 1. Drawbacks of Threading, Introduction to Tasks, Advantages of TPL

### Drawbacks of using Thread directly

- **Expensive:** each `Thread` allocates its own stack (\~1 MB) and OS resources; creating hundreds is slow and wasteful.
- **No return value:** you need shared fields, callbacks or Join + property tricks to get results.
- **Exceptions are lost or fatal:** an unhandled exception in a thread crashes the process, and the caller can't easily catch it.
- **No built-in continuations:** "when this finishes, do that" needs manual signalling.
- **No built-in cancellation**, progress or composition (wait for any/all).
- **Manual scaling:** you decide how many threads, regardless of CPU cores.

### Introduction to tasks

A **Task** represents an asynchronous operation: a unit of work that will complete in the future, possibly producing a value. Tasks usually run on **thread-pool** threads, and the **Task Parallel Library (TPL)** (`System.Threading.Tasks`, .NET 4) schedules them efficiently across CPU cores.

```csharp
Task t = Task.Run(() => Console.WriteLine($"Running on pool thread {Environment.CurrentManagedThreadId}"));
t.Wait();

Task<int> sum = Task.Run(() => Enumerable.Range(1, 100).Sum());
Console.WriteLine(sum.Result);   // 5050
```

### Advantages of TPL

| Need | Thread | Task |
| --- | --- | --- |
| Reuse threads (cheap) | No | Yes (thread pool) |
| Return a value | Manual | `Task<T>.Result` / `await` |
| Catch exceptions in the caller | Hard | `AggregateException` / `try` around `await` |
| Continue when done | Manual signalling | `ContinueWith`, `await` |
| Wait for all / any | Join each | `WaitAll`, `WhenAll`, `WaitAny`, `WhenAny` |
| Cancellation | Manual flag | `CancellationToken` built in |
| Works with `async`/`await` | No | Yes |
| Data parallelism | Manual | `Parallel.For`, `Parallel.ForEach`, PLINQ |

### The Task class

| Member | Purpose |
| --- | --- |
| `Task.Run(action)` / `Task.Run(func)` | Queue work on the pool and start it (most common) |
| `new Task(action)` + `Start()` | Create first, start later (rare) |
| `Task.Factory.StartNew(...)` | Advanced creation options |
| `Wait()`, `Task.WaitAll`, `Task.WaitAny` | Block until done |
| `Task.WhenAll`, `Task.WhenAny` | Non-blocking (awaitable) versions |
| `Result` | Value of a `Task<T>` (blocks until ready) |
| `ContinueWith(...)` | Run something after it completes |
| `Status`, `IsCompleted`, `IsFaulted`, `IsCanceled`, `IsCompletedSuccessfully` | State |
| `Exception` | `AggregateException` if it faulted |
| `Id`, `Task.CurrentId` | Identify tasks |
| `Task.Delay(ms)`, `Task.CompletedTask`, `Task.FromResult(x)` | Helpers |

**Task statuses:** `Created` → `WaitingToRun` → `Running` → `RanToCompletion` / `Faulted` / `Canceled`.

## 2. Creating Tasks: Task.Run, Task.Factory.StartNew and Stopwatch

### Task.Run

The standard way to start CPU-bound work on the thread pool.

```csharp
Task t1 = Task.Run(() => ProcessFile("jan.csv"));             // Action
Task<int> t2 = Task.Run(() => CountRows("feb.csv"));          // Func<int>: Task<int>
Task t3 = Task.Run(async () => await SendEmailsAsync());       // async lambda: unwrapped automatically

int rows = t2.Result;                                          // waits, then returns the value

// several at once
var tasks = new[] { "jan", "feb", "mar" }
    .Select(m => Task.Run(() => ProcessFile($"{m}.csv")))
    .ToArray();
Task.WaitAll(tasks);
```

### Task.Factory.StartNew

The older, more configurable API. Use it only when you need its options.

```csharp
Task longJob = Task.Factory.StartNew(
    () => MonitorQueueForever(),
    CancellationToken.None,
    TaskCreationOptions.LongRunning,      // hint: use a dedicated thread, not a pool thread
    TaskScheduler.Default);

Task<string> withState = Task.Factory.StartNew(state => $"Hello {state}", "Rohit");   // pass a state object
```

|  | `Task.Run` | `Task.Factory.StartNew` |
| --- | --- | --- |
| Introduced | .NET 4.5 (simple wrapper) | .NET 4.0 |
| Defaults | Pool, `DenyChildAttach`, default scheduler | Current scheduler (may surprise you) |
| Options (LongRunning, AttachedToParent, scheduler, state) | No | Yes |
| Async lambdas | Unwrapped automatically | Returns `Task<Task>`; needs `.Unwrap()` |
| Use | Almost always | Long-running work or special options |

### Stopwatch: measuring performance

```csharp
using System.Diagnostics;

var sw = Stopwatch.StartNew();
for (int i = 0; i < 3; i++) Thread.Sleep(1000);         // sequential
Console.WriteLine($"Sequential: {sw.ElapsedMilliseconds} ms");   // ~3000

sw.Restart();
Task.WaitAll(
    Task.Run(() => Thread.Sleep(1000)),
    Task.Run(() => Thread.Sleep(1000)),
    Task.Run(() => Thread.Sleep(1000)));
Console.WriteLine($"Tasks: {sw.ElapsedMilliseconds} ms");        // ~1000

sw.Stop();  TimeSpan e = sw.Elapsed;  long ticks = sw.ElapsedTicks;  sw.Reset();
```

`Stopwatch` uses a high-resolution timer, so prefer it over subtracting `DateTime.Now` values. For serious benchmarks use BenchmarkDotNet (warm-up, many iterations, statistics).

## 3. Waiting: Wait, WaitAll, WaitAny and Task.Delay

```csharp
Task a = Task.Run(() => { Thread.Sleep(1000); Console.WriteLine("A done"); });
Task b = Task.Run(() => { Thread.Sleep(3000); Console.WriteLine("B done"); });

// Task.Wait: block until ONE task finishes
a.Wait();
bool finished = b.Wait(500);                 // timeout: false if not done in 500 ms

// Task.WaitAll: block until EVERY task finishes
Task.WaitAll(a, b);
bool allIn5s = Task.WaitAll(new[] { a, b }, TimeSpan.FromSeconds(5));

// Task.WaitAny: block until the FIRST finishes; returns its index
Task<string>[] mirrors =
{
    Task.Run(() => { Thread.Sleep(800); return "Mirror 1"; }),
    Task.Run(() => { Thread.Sleep(300); return "Mirror 2"; })
};
int winner = Task.WaitAny(mirrors);
Console.WriteLine($"Fastest: {mirrors[winner].Result}");   // Mirror 2
```

### Task.Delay

A task that completes after a time, **without blocking a thread** (unlike `Thread.Sleep`).

```csharp
await Task.Delay(2000);                                     // pause this async method for 2 s
Task.Delay(1000).Wait();                                    // blocking version (avoid in async code)

// Timeout pattern: whichever finishes first
Task<string> work = DownloadReportAsync();
if (await Task.WhenAny(work, Task.Delay(5000)) == work)
    Console.WriteLine(await work);
else
    Console.WriteLine("Timed out");

await Task.Delay(1000, cancellationToken);                 // cancellable delay
```

| Blocking (thread waits) | Non-blocking (`await`, thread is freed) | Completes when |
| --- | --- | --- |
| `t.Wait()` | `await t` | That task finishes |
| `Task.WaitAll(ts)` | `await Task.WhenAll(ts)` | All finish (returns all results for `Task<T>`) |
| `Task.WaitAny(ts)` | `await Task.WhenAny(ts)` | The first finishes (returns that task) |
| `Thread.Sleep(ms)` | `await Task.Delay(ms)` | Time passes |

**Rule:** in console demos `Wait` is fine; in UI apps and ASP.NET use `await` (blocking with `.Wait()`/`.Result` can deadlock UI threads and wastes server threads).

## 4. Generic Task\<T> and Returning Complex Types

`Task<TResult>` is a task that **produces a value**. Get it with `.Result` (blocks) or `await` (doesn't).

```csharp
Task<int> countTask = Task.Run(() => File.ReadLines("claims.csv").Count());
int count = countTask.Result;                 // blocks until ready

Task<decimal> totalTask = Task.Run(() =>
{
    decimal total = 0;
    foreach (var line in File.ReadLines("claims.csv").Skip(1))
        total += decimal.Parse(line.Split(',')[2]);
    return total;                             // return type sets T = decimal
});
Console.WriteLine(await totalTask);

// WhenAll with Task<T> returns an array of results
int[] counts = await Task.WhenAll(
    Task.Run(() => CountRows("jan.csv")),
    Task.Run(() => CountRows("feb.csv")));
Console.WriteLine(counts.Sum());
```

### Returning complex types

A task can return any type: objects, collections, tuples or records.

```csharp
public record ClaimSummary(int Count, decimal Total, decimal Average, string Largest);

Task<ClaimSummary> summaryTask = Task.Run(() =>
{
    var claims = LoadClaims();                            // List<Claim>
    var largest = claims.MaxBy(c => c.Amount)!;
    return new ClaimSummary(claims.Count, claims.Sum(c => c.Amount),
                            claims.Average(c => c.Amount), largest.Id);
});

ClaimSummary s = await summaryTask;
Console.WriteLine($"{s.Count} claims, total {s.Total:N0}, largest {s.Largest}");

// Tuples and collections work too
Task<(int Ok, int Failed)> validate = Task.Run(() => (Ok: 95, Failed: 5));
var (ok, failed) = await validate;

Task<List<string>> namesTask = Task.Run(() => LoadClaims().Select(c => c.Patient).Distinct().ToList());
```

Returning an immutable object (record) from a task is the cleanest way to hand results between threads: no shared mutable state, no locks.

**Helpers:** `Task.FromResult(value)` gives an already-completed `Task<T>` (useful for caching and tests); `Task.CompletedTask` is a finished non-generic task.

## 5. ContinueWith and Continuation Chains

`ContinueWith` schedules another task to run **after** a task (the *antecedent*) finishes, without blocking anyone while waiting. The continuation receives the finished antecedent, so it can read `Result`, `Exception` or `Status`.

```csharp
Task<List<Claim>> load = Task.Run(() => LoadClaims());

Task<decimal> total = load.ContinueWith(prev => prev.Result.Sum(c => c.Amount));

total.ContinueWith(prev => Console.WriteLine($"Total: {prev.Result:N0}"));
```

### Continuation options

&#91;embedded content: Conditional continuations\]

Use `TaskContinuationOptions` to run different follow-ups for success, failure and cancellation.

```csharp
Task<List<Claim>> load = Task.Run(() => LoadClaims());

load.ContinueWith(t => Console.WriteLine($"{t.Result.Count} claims loaded"),
                  TaskContinuationOptions.OnlyOnRanToCompletion);

load.ContinueWith(t => Log(t.Exception!.InnerException!),
                  TaskContinuationOptions.OnlyOnFaulted);

load.ContinueWith(t => Console.WriteLine("Load cancelled"),
                  TaskContinuationOptions.OnlyOnCanceled);
```

Other useful options: `NotOnFaulted`, `ExecuteSynchronously` (run on the same thread if possible), `LongRunning`, `AttachedToParent`. `Task.Factory.ContinueWhenAll(tasks, ...)` / `ContinueWhenAny` continue after a group.

### Continuation chain

Each `ContinueWith` returns a new task, so they can be chained into a pipeline:

```csharp
Task.Run(() => File.ReadAllLines("claims.csv"))                 // 1. read
    .ContinueWith(t => t.Result.Skip(1)
                              .Select(l => decimal.Parse(l.Split(',')[2]))
                              .ToList())                       // 2. parse
    .ContinueWith(t => t.Result.Sum())                          // 3. total
    .ContinueWith(t => File.WriteAllText("total.txt", t.Result.ToString()))   // 4. save
    .ContinueWith(t =>
    {
        if (t.IsFaulted) Console.WriteLine($"Failed: {t.Exception!.GetBaseException().Message}");
        else Console.WriteLine("Report saved");
    })
    .Wait();
```

**Caution:** if step 1 fails, step 2's `t.Result` rethrows the error (wrapped again), so faults travel down the chain; check `t.IsFaulted` or use options.

**Modern equivalent with await** (same steps, much easier to read and debug):

```csharp
try
{
    string[] lines = await File.ReadAllLinesAsync("claims.csv");
    var amounts = lines.Skip(1).Select(l => decimal.Parse(l.Split(',')[2])).ToList();
    await File.WriteAllTextAsync("total.txt", amounts.Sum().ToString());
    Console.WriteLine("Report saved");
}
catch (Exception ex) { Console.WriteLine($"Failed: {ex.Message}"); }
```

`await` is built on the same continuation mechanism; prefer it in new code, but know `ContinueWith` for interviews and older code.

## 6. Task Exception Handling

An exception thrown inside a task **doesn't crash the program immediately**. The task becomes `Faulted` and stores the exception, which is rethrown when you observe the task: `Wait()`, `.Result`, `WaitAll`, or `await`.

### Part 1: Wait/Result throw AggregateException

```csharp
Task t = Task.Run(() => throw new InvalidOperationException("Claim file missing"));

try
{
    t.Wait();                                         // or t.Result for Task<T>
}
catch (AggregateException ae)                         // the real error is INSIDE
{
    foreach (var ex in ae.InnerExceptions)
        Console.WriteLine($"{ex.GetType().Name}: {ex.Message}");   // InvalidOperationException: Claim file missing
}

Console.WriteLine(t.Status);       // Faulted
Console.WriteLine(t.IsFaulted);    // True
Console.WriteLine(t.Exception?.InnerException?.Message);
```

`AggregateException` wraps exceptions because several tasks (or child tasks) can fail at once.

### Part 2: many tasks, Handle, Flatten and await

```csharp
var tasks = new[]
{
    Task.Run(() => throw new FileNotFoundException("jan.csv")),
    Task.Run(() => throw new FormatException("bad amount in feb.csv")),
    Task.Run(() => Console.WriteLine("mar.csv ok"))
};

try { Task.WaitAll(tasks); }
catch (AggregateException ae)
{
    Console.WriteLine($"{ae.InnerExceptions.Count} tasks failed");   // 2

    ae.Flatten().Handle(ex =>                       // Flatten: unwrap nested AggregateExceptions
    {
        if (ex is FileNotFoundException) { Console.WriteLine($"Missing: {ex.Message}"); return true; }   // handled
        if (ex is FormatException)       { Console.WriteLine($"Bad data: {ex.Message}"); return true; }
        return false;                                // unhandled ones are rethrown in a new AggregateException
    });
}

// Check each task individually
foreach (var task in tasks)
    Console.WriteLine($"{task.Id}: {task.Status}");   // Faulted, Faulted, RanToCompletion

// With await: the FIRST inner exception is thrown directly (no AggregateException)
try { await Task.Run(() => throw new FormatException("bad")); }
catch (FormatException ex) { Console.WriteLine(ex.Message); }

// await Task.WhenAll throws the first error, but all errors are on the WhenAll task
var all = Task.WhenAll(tasks);
try { await all; }
catch { foreach (var ex in all.Exception!.InnerExceptions) Console.WriteLine(ex.Message); }
```

| How you wait | What's thrown |
| --- | --- |
| `t.Wait()`, `t.Result`, `Task.WaitAll` | `AggregateException` (look at `InnerExceptions`) |
| `await t` | The original exception (first one) |
| `await Task.WhenAll(...)` | First exception; all are in `whenAllTask.Exception` |
| Never observed | Nothing (the error is silently lost; `TaskScheduler.UnobservedTaskException` fires later) |

**Always observe tasks**: await them, Wait on them, or attach an `OnlyOnFaulted` continuation; otherwise failures disappear.

## 7. Task Cancellation

.NET uses **cooperative cancellation**: the caller requests cancellation through a `CancellationTokenSource`; the task's code checks the `CancellationToken` and stops itself cleanly. Nothing is forcibly killed.

```csharp
using var cts = new CancellationTokenSource();
CancellationToken token = cts.Token;

Task<int> job = Task.Run(() =>
{
    int processed = 0;
    foreach (var line in File.ReadLines("claims.csv"))
    {
        token.ThrowIfCancellationRequested();     // throws OperationCanceledException when cancelled
        // or: if (token.IsCancellationRequested) { cleanup(); return processed; }
        Thread.Sleep(10);                         // simulate work
        processed++;
    }
    return processed;
}, token);                                        // passing the token lets Run skip the task if already cancelled

Console.WriteLine("Press C to cancel");
if (Console.ReadKey(true).Key == ConsoleKey.C) cts.Cancel();     // request cancellation

try
{
    Console.WriteLine($"Processed {await job} rows");
}
catch (OperationCanceledException)
{
    Console.WriteLine($"Cancelled. Status: {job.Status}");   // Canceled
}
```

**Other ways to cancel**

```csharp
var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(5));   // auto-cancel after 5 s
cts.CancelAfter(2000);                                                // same, later
await Task.Delay(10_000, token);                                     // built-in APIs accept tokens
await File.ReadAllTextAsync("big.txt", token);
await httpClient.GetStringAsync(url, token);

using var linked = CancellationTokenSource.CreateLinkedTokenSource(userToken, timeout.Token);   // either one cancels
token.Register(() => Console.WriteLine("Cancellation requested"));    // callback when cancelled
```

**Rules**

- Pass the token **down** to every method and API call that accepts one.
- Throw `OperationCanceledException` (via `ThrowIfCancellationRequested`) with the same token so the task ends `Canceled`, not `Faulted`.
- Check the token at regular points in loops; long blocking calls without a token can't be cancelled.
- Dispose `CancellationTokenSource` (it may hold a timer).
- `Thread.Abort` is gone in .NET Core/5+; cooperative cancellation is the only supported way.

## 8. File IO with Tasks

File work is mostly **I/O-bound** (waiting for the disk), so use the built-in **async** file APIs with `await`; no thread sits blocked while the disk works. Use `Task.Run` only for the **CPU-bound** part (parsing, calculations).

### Async reading and writing

```csharp
string text = await File.ReadAllTextAsync("claims.csv");
string[] lines = await File.ReadAllLinesAsync("claims.csv");
await File.WriteAllTextAsync("summary.txt", "Total: 25000");
await File.AppendAllLinesAsync("audit.log", new[] { $"{DateTime.Now:s} done" });

await using var writer = new StreamWriter("report.csv");
await writer.WriteLineAsync("Id,Amount");

using var reader = new StreamReader("claims.csv");
while (await reader.ReadLineAsync() is { } line)         // stream line by line
    Console.WriteLine(line);

await foreach (var line in File.ReadLinesAsync("claims.csv"))   // .NET 7+: IAsyncEnumerable<string>
    Console.WriteLine(line);
```

### Processing many files concurrently

```csharp
string[] files = Directory.GetFiles(@"C:\Claims\Monthly", "*.csv");
var sw = Stopwatch.StartNew();

// Start all reads at once, then wait for all
Task<(string File, decimal Total)>[] jobs = files.Select(async f =>
{
    string[] rows = await File.ReadAllLinesAsync(f);                     // I/O: await
    decimal total = await Task.Run(() => rows.Skip(1)                    // CPU: Task.Run
        .Sum(r => decimal.Parse(r.Split(',')[2])));
    return (Path.GetFileName(f), total);
}).ToArray();

var results = await Task.WhenAll(jobs);
foreach (var (file, total) in results.OrderBy(r => r.File))
    Console.WriteLine($"{file}: {total:N0}");
Console.WriteLine($"{files.Length} files in {sw.ElapsedMilliseconds} ms");

await File.WriteAllLinesAsync("totals.csv", results.Select(r => $"{r.File},{r.Total}"));
```

### Limiting concurrency

Opening hundreds of files at once can overwhelm the disk. Throttle with `SemaphoreSlim` or `Parallel.ForEachAsync`:

```csharp
await Parallel.ForEachAsync(files, new ParallelOptions { MaxDegreeOfParallelism = 4 }, async (f, ct) =>
{
    var content = await File.ReadAllTextAsync(f, ct);
    await File.WriteAllTextAsync(Path.ChangeExtension(f, ".upper"), content.ToUpper(), ct);
});
```

**Tips:** pass a `CancellationToken` to every async call; don't write to the **same** file from several tasks at once (collect results, write once, or use one writer behind a lock/channel); catch `IOException` per file so one bad file doesn't stop the batch.

## 9. Cheat Sheet and Interview Questions

```csharp
Task t = Task.Run(() => Work());
Task<int> r = Task.Run(() => Compute());  int v = r.Result;  int v2 = await r;
Task.Factory.StartNew(LongWork, TaskCreationOptions.LongRunning);
t.Wait();  Task.WaitAll(t1, t2);  int i = Task.WaitAny(t1, t2);
await Task.WhenAll(t1, t2);  Task first = await Task.WhenAny(t1, t2);  await Task.Delay(1000);
r.ContinueWith(p => Use(p.Result), TaskContinuationOptions.OnlyOnRanToCompletion);
try { t.Wait(); } catch (AggregateException ae) { ae.Flatten().Handle(e => true); }
var cts = new CancellationTokenSource(); Task.Run(() => { cts.Token.ThrowIfCancellationRequested(); }, cts.Token); cts.Cancel();
string s = await File.ReadAllTextAsync(path, cts.Token);
var sw = Stopwatch.StartNew(); /* ... */ sw.Stop(); Console.WriteLine(sw.ElapsedMilliseconds);
```

**Checklist**

- [ ] List the drawbacks of raw threads and the advantages of TPL
- [ ] Start tasks with Task.Run and Task.Factory.StartNew; know when each fits
- [ ] Time sequential vs parallel work with Stopwatch
- [ ] Use Wait, WaitAll, WaitAny, Task.Delay and their await equivalents
- [ ] Return a record from a Task\<T>
- [ ] Write conditional continuations and a continuation chain
- [ ] Catch AggregateException, use Flatten/Handle, and compare with await
- [ ] Cancel a task with CancellationTokenSource
- [ ] Read and process several files concurrently with async APIs

**Interview questions (short answers)**

1. **Thread vs Task?** A thread is an OS execution path; a Task is a higher-level unit of work, usually run on pool threads, with results, exceptions, continuations and cancellation.
2. **What is the TPL?** The Task Parallel Library: Task, Task\<T>, Parallel and related types for concurrent and parallel programming.
3. **Task.Run vs Task.Factory.StartNew?** Task.Run is the simple, safe default; StartNew offers options (LongRunning, schedulers, state) and needs Unwrap for async lambdas.
4. **Wait vs await?** Wait blocks the thread; await frees it and resumes later.
5. **WaitAll vs WhenAll?** WaitAll blocks; WhenAll returns a task you await (and gives results).
6. **WaitAny vs WhenAny?** Same difference; both complete when the first task finishes.
7. **Thread.Sleep vs Task.Delay?** Sleep blocks a thread; Delay is a timer-based task that doesn't.
8. **How do you return a value from a task?** Use Task\<T>; read `.Result` or `await` it.
9. **What is ContinueWith?** Schedules work to run after a task completes, optionally only on success/fault/cancel.
10. **What is AggregateException?** A wrapper holding one or more exceptions thrown by tasks, raised by Wait/Result/WaitAll.
11. **How does await handle task exceptions?** It rethrows the first inner exception directly.
12. **How do you cancel a task?** Pass a CancellationToken, check it (ThrowIfCancellationRequested), and call Cancel on the source.
13. **What are the task statuses?** Created, WaitingToRun, Running, RanToCompletion, Faulted, Canceled (plus a few waiting states).
14. **CPU-bound vs I/O-bound?** CPU-bound: use Task.Run/Parallel. I/O-bound: use async APIs with await, no extra thread.
15. **What happens to an unobserved faulted task?** The exception is lost (UnobservedTaskException event); it doesn't crash modern .NET apps.
