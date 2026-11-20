# Exception Handling

Handling run-time errors safely: try/catch/finally, the exception classes you'll meet most, inner exceptions and stack traces, filters, custom exceptions and logging.

## 1. Try-Catch-Finally

An **exception** is an object describing a run-time error. When one is thrown, normal execution stops and .NET looks for a matching `catch`, first in the current method, then in each caller up the call stack. If none matches, the program crashes. `try`/`catch`/`finally` lets you handle it and always clean up.

&#91;embedded content: How try, catch and finally run\]

If no catch matches, `finally` still runs before the exception continues up to the caller.

```csharp
StreamReader? reader = null;
try
{
    reader = new StreamReader("claims.csv");
    int count = int.Parse(reader.ReadLine()!);         // may throw FormatException
    Console.WriteLine(100 / count);                    // may throw DivideByZeroException
}
catch (FormatException ex)                             // most specific first
{
    Console.WriteLine($"Bad number: {ex.Message}");
}
catch (DivideByZeroException)
{
    Console.WriteLine("Count was zero");
}
catch (Exception ex)                                   // general catch-all LAST
{
    Console.WriteLine($"Unexpected: {ex.Message}");
    throw;                                             // rethrow, keeping the original stack trace
}
finally
{
    reader?.Dispose();                                 // runs in every case
    Console.WriteLine("Cleanup done");
}
```

**Rules**

- A `try` needs at least one `catch` or a `finally` (`try`/`finally` alone is valid).
- Catch blocks are checked **top to bottom**: put specific exceptions before general ones (the compiler rejects a derived type after its base).
- `finally` runs after `try`/`catch` whether or not an exception happened, even after `return`. It doesn't run if the process is killed or on `Environment.FailFast`.
- `throw;` rethrows and **keeps** the stack trace; `throw ex;` resets it (avoid).
- `throw new XyzException("msg")` raises your own exceptions; `throw` can also be an expression: `name ?? throw new ArgumentNullException(nameof(name))`.
- `using` is shorthand for try/finally + Dispose, so prefer it for cleanup.

## 2. System.Exception and the Exception Hierarchy

Every exception derives from `System.Exception`. Catching a base type also catches all its subclasses.

**Key members of `System.Exception`**

| Member | Holds |
| --- | --- |
| `Message` | Human-readable description |
| `StackTrace` | Method calls that led to the error, with file/line in debug builds |
| `InnerException` | The original exception that caused this one (section 4) |
| `Source` | Assembly/app where it occurred |
| `TargetSite` | Method that threw it |
| `Data` | Dictionary for extra key/value details |
| `HResult` | Numeric error code |
| `GetType().Name` | The exception's class, e.g. `FormatException` |
| `ToString()` | Type + message + inner exceptions + stack trace (best for logs) |

**The hierarchy (the parts you'll meet)**

```text
System.Object
  System.Exception
    System.SystemException                 (thrown by the runtime / .NET)
      ArgumentException
        ArgumentNullException
        ArgumentOutOfRangeException
      FormatException
      IndexOutOfRangeException
      NullReferenceException
      InvalidOperationException
        ObjectDisposedException
      InvalidCastException
      ArithmeticException
        DivideByZeroException
        OverflowException
      IOException
        FileNotFoundException
        DirectoryNotFoundException
      KeyNotFoundException
      NotImplementedException / NotSupportedException
      OutOfMemoryException, StackOverflowException   (can't really be handled)
    System.ApplicationException            (old convention for app exceptions; not recommended now)
    YourCustomException                    (derive from Exception or a specific type)
```

Because `ArgumentNullException` derives from `ArgumentException`, `catch (ArgumentException)` catches both.

## 3. Common Exceptions

| Exception | Thrown when | Typical cause | Prevent with |
| --- | --- | --- | --- |
| `FormatException` | A string isn't in the expected format | `int.Parse("12a")`, bad date string | `TryParse` |
| `IndexOutOfRangeException` | Array index < 0 or ≥ Length | Off-by-one loop `i <= arr.Length` | Check bounds, use `foreach` |
| `ArgumentOutOfRangeException` | An argument is outside the allowed range | `list[10]` on a short List, `Substring(50)`, negative quantity | Validate inputs, check `Count` |
| `NullReferenceException` | Using a member of a `null` reference | Uninitialised object, missing lookup result | Null checks, `?.`, `??`, nullable reference types |
| `ArgumentNullException` | A required argument is `null` | Caller passed null | Guard clauses |
| `ArgumentException` | An argument is invalid in another way | Empty name, duplicate dictionary key | Validate inputs |
| `InvalidOperationException` | The object's current state doesn't allow the call | `First()` on empty list, modifying a list during foreach, `Value` on null nullable | Check state first (`Any()`, `HasValue`) |

```csharp
// FormatException
try { int age = int.Parse("thirty"); }
catch (FormatException) { Console.WriteLine("Enter digits only"); }

// IndexOutOfRangeException
int[] marks = { 70, 80, 90 };
// for (int i = 0; i <= marks.Length; i++) Console.WriteLine(marks[i]);   // fails at i = 3

// NullReferenceException
string? city = null;
// int n = city.Length;              // crash
int n = city?.Length ?? 0;           // safe

// InvalidOperationException
var empty = new List<int>();
// int first = empty.First();        // "Sequence contains no elements"
```

### Throwing argument exceptions yourself (guard clauses)

Validate at the start of public methods so bad input fails fast with a clear message:

```csharp
public void Admit(Patient patient, int bedNo, string ward)
{
    ArgumentNullException.ThrowIfNull(patient);                       // .NET 6+
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(bedNo);         // .NET 8+
    ArgumentException.ThrowIfNullOrWhiteSpace(ward);                  // .NET 8+

    // classic form
    if (bedNo > 50)
        throw new ArgumentOutOfRangeException(nameof(bedNo), bedNo, "Ward has only 50 beds");
    if (_isClosed)
        throw new InvalidOperationException("Ward is closed for admissions");
}
```

**Rule of thumb:** `Argument...` exceptions = **the caller** passed bad data; `InvalidOperationException` = the arguments are fine, but **the object** is in the wrong state; `NullReferenceException` and `IndexOutOfRangeException` = **bugs** to fix, not to catch. Always pass `nameof(param)` so the message names the parameter.

## 4. Inner Exceptions, Stack Trace and Exception Filters

### Inner exception

When you catch a low-level exception and throw a more meaningful one, pass the original as the **inner exception** so no detail is lost.

```csharp
public Policy LoadPolicy(string number)
{
    try
    {
        string json = File.ReadAllText($"policies/{number}.json");
        return JsonSerializer.Deserialize<Policy>(json)!;
    }
    catch (Exception ex) when (ex is IOException or JsonException)
    {
        throw new PolicyLoadException($"Could not load policy {number}", ex);   // wrap
    }
}

try { LoadPolicy("P9"); }
catch (PolicyLoadException ex)
{
    Console.WriteLine(ex.Message);                     // Could not load policy P9
    Console.WriteLine(ex.InnerException?.GetType());   // System.IO.FileNotFoundException
    Console.WriteLine(ex.GetBaseException().Message);  // deepest original message
}
```

`AggregateException` (from tasks/`Parallel`) holds several inner exceptions in `InnerExceptions`.

### Stack trace

`StackTrace` lists the chain of method calls, most recent first, which tells you exactly where an error started.

```csharp
catch (Exception ex)
{
    Console.WriteLine(ex.StackTrace);
    //   at ClaimService.Validate(Claim c) in ClaimService.cs:line 42
    //   at ClaimService.Submit(Claim c) in ClaimService.cs:line 18
    //   at Program.Main() in Program.cs:line 7
    Console.WriteLine(ex.ToString());   // type + message + stack trace + inner exceptions
}
```

Line numbers appear when PDB symbol files are available (debug builds, or release with symbols). `throw;` keeps the original trace; `throw ex;` replaces it with the current line, hiding where the problem really happened.

### Catch when (exception filters, C# 6)

A `when` clause adds a condition; the catch runs only if it's true. Otherwise the exception keeps looking for another handler, with the **stack untouched**.

```csharp
try
{
    await http.GetStringAsync(url);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    Console.WriteLine("Not found");
}
catch (HttpRequestException ex) when (ex.StatusCode >= HttpStatusCode.InternalServerError)
{
    Console.WriteLine("Server error, retry later");
}

// Log without catching: the filter returns false, so the exception continues unchanged
catch (Exception ex) when (Log(ex)) { }
static bool Log(Exception ex) { Console.WriteLine(ex); return false; }
```

**Why filters beat `if` + `throw;` inside a catch:** the stack isn't unwound unless the filter matches, so debuggers and crash dumps show the original state, and multiple catch blocks for the same type become possible.

## 5. Custom Exceptions and an Exception Logger

### Custom exceptions

Create your own exception when callers need to recognise and handle a **specific business error** (claim limit exceeded, policy expired). Name it `...Exception`, derive from `Exception` (or a closer match like `InvalidOperationException`), and provide the standard constructors.

```csharp
public class ClaimLimitExceededException : Exception
{
    public string ClaimId { get; }
    public decimal Requested { get; }
    public decimal Limit { get; }

    public ClaimLimitExceededException() { }
    public ClaimLimitExceededException(string message) : base(message) { }
    public ClaimLimitExceededException(string message, Exception inner) : base(message, inner) { }

    public ClaimLimitExceededException(string claimId, decimal requested, decimal limit)
        : base($"Claim {claimId}: requested {requested:N0} exceeds limit {limit:N0}")
    {
        ClaimId = claimId; Requested = requested; Limit = limit;
    }
}

public void Approve(Claim c, decimal limit)
{
    if (c.Amount > limit)
        throw new ClaimLimitExceededException(c.Id, c.Amount, limit);
}

try { Approve(claim, 100_000m); }
catch (ClaimLimitExceededException ex)
{
    Console.WriteLine(ex.Message);                              // Claim C1: requested 125,000 exceeds limit 100,000
    Console.WriteLine($"Over by {ex.Requested - ex.Limit:N0}");
}
```

**Guidelines:** only create one if callers will catch it specifically (otherwise use a built-in type); keep it serializable-friendly (public properties); don't derive from `ApplicationException`; put useful data in properties, not just the message.

### Exception logger

Log every unexpected exception with full details in one place, so problems can be diagnosed later.

```csharp
public static class ExceptionLogger
{
    private static readonly string LogPath = Path.Combine(AppContext.BaseDirectory, "errors.log");
    private static readonly object _lock = new();

    public static void Log(Exception ex, string? context = null)
    {
        var sb = new StringBuilder();
        sb.AppendLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] {context}");
        for (Exception? e = ex; e != null; e = e.InnerException)     // walk the inner chain
        {
            sb.AppendLine($"  {e.GetType().FullName}: {e.Message}");
        }
        sb.AppendLine(ex.StackTrace);
        sb.AppendLine(new string('-', 60));

        lock (_lock) File.AppendAllText(LogPath, sb.ToString());   // thread-safe append
    }
}

// Use it in catch blocks...
try { ProcessClaims(); }
catch (Exception ex)
{
    ExceptionLogger.Log(ex, "Nightly claim processing");
    Console.WriteLine("Something went wrong. Please contact support.");   // friendly message for users
}

// ...and as a last-resort global handler
AppDomain.CurrentDomain.UnhandledException += (s, e) =>
    ExceptionLogger.Log((Exception)e.ExceptionObject, "Unhandled");
```

In real applications use a logging framework (`ILogger` with Serilog or NLog): `logger.LogError(ex, "Failed to process claim {ClaimId}", id);` gives structured logs, levels and multiple outputs. ASP.NET Core adds global exception-handling middleware.

## 6. Best Practices, Cheat Sheet and Interview Questions

**Best practices**

1. Catch only what you can handle meaningfully; let the rest bubble up to a global handler.
2. Catch specific exceptions before general ones; avoid empty `catch { }` that swallows errors.
3. Use `throw;` (not `throw ex;`) to rethrow.
4. Wrap with an inner exception when translating to a higher-level error.
5. Prevent predictable errors instead of catching them: `TryParse`, null checks, bounds checks, `Any()`.
6. Release resources with `using`/`finally`.
7. Validate arguments at method entry (guard clauses) and throw the right `Argument...` exception.
8. Don't use exceptions for normal control flow; they're slow compared to an `if`.
9. Log `ex.ToString()` (everything) but show users a friendly message without internal details.

**Cheat sheet**

```csharp
try { ... }
catch (FormatException ex) { ... }                       // specific first
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) { ... }   // filter
catch (Exception ex) { Log(ex); throw; }                 // general last, rethrow
finally { cleanup(); }

throw new ArgumentNullException(nameof(x));
ArgumentNullException.ThrowIfNull(x);
throw new InvalidOperationException("State not valid");
throw new MyException("msg", innerException);
ex.Message; ex.StackTrace; ex.InnerException; ex.GetBaseException(); ex.ToString();
```

**Checklist**

- [ ] Draw the try/catch/finally flow from memory
- [ ] Order catch blocks correctly and explain why
- [ ] Name the trigger and prevention for each of the 7 common exceptions
- [ ] Write guard clauses with ArgumentNullException / ArgumentOutOfRangeException
- [ ] Wrap an exception with an inner exception and read it back
- [ ] Read a stack trace; explain throw vs throw ex
- [ ] Use a `when` filter
- [ ] Write a custom exception with the standard constructors and extra properties
- [ ] Build an exception logger that walks inner exceptions

**Interview questions (short answers)**

1. **What is an exception?** An object representing a run-time error that interrupts normal flow until handled.
2. **What is the base class of all exceptions?** `System.Exception`.
3. **Can a try block exist without catch?** Yes, with `finally`.
4. **When doesn't finally run?** Process killed, `Environment.FailFast`, stack overflow, power loss.
5. **throw vs throw ex?** `throw;` preserves the stack trace; `throw ex;` resets it.
6. **Why order catch blocks from specific to general?** The first matching block runs; a general catch first would hide specific ones (compile error).
7. **What is an inner exception?** The original exception wrapped inside a new one, available via `InnerException`.
8. **ArgumentException vs InvalidOperationException?** Bad argument vs object in the wrong state for the call.
9. **NullReferenceException vs ArgumentNullException?** The first happens when null is used; the second is thrown deliberately when a null argument is passed.
10. **IndexOutOfRangeException vs ArgumentOutOfRangeException?** Arrays throw the first; collections and methods validating arguments throw the second.
11. **What is an exception filter?** `catch (...) when (condition)`: catch only if the condition is true, without unwinding otherwise.
12. **How do you create a custom exception?** Derive from Exception, name it `...Exception`, add the standard constructors and any extra properties.
13. **What does StackTrace show?** The call chain from where the exception was thrown up to where it was caught.
14. **Checked vs unchecked exceptions in C#?** C# has no checked exceptions; methods don't declare what they throw.
