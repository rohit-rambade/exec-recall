# Modern C# Features (C# 8, 9, 10)

The language features from your "C# 9 features" section that aren't pattern matching or records (those have the next tab): leaner program files, nullable reference types, newer lambda and interface features, and indexes and ranges.

## 1. Program Structure: Top-Level Statements, File-Scoped Namespaces, Global Using, Module Initializers

### Top-level statements (C# 9)

Write the program's code directly in one file without `class Program` and `static void Main`. The compiler generates them for you.

```csharp
// Program.cs: the whole program
using System.Text.Json;

Console.WriteLine("Claims processor starting");
var claims = LoadClaims();
Console.WriteLine($"{claims.Count} claims loaded");

if (args.Length > 0) Console.WriteLine($"First arg: {args[0]}");   // 'args' is available
await Task.Delay(100);                                               // await allowed: Main becomes async
return 0;                                                            // optional exit code

List<string> LoadClaims() => new() { "C1", "C2" };   // local functions allowed
class Helper { }                                     // types must come AFTER the statements
```

**Rules:** only **one** file per project can have top-level statements; statements must come before any type declarations in that file; you can still use `args`, `await` and `return int`. Great for console apps, scripts, samples and ASP.NET Core `Program.cs`.

### File-scoped namespaces (C# 10)

```csharp
namespace HealthIndia.Claims;     // applies to the whole file: no braces, one less indent

public class ClaimService { }
```

Only one per file, and it can't be mixed with block-scoped namespaces in that file.

### Global using (C# 10)

```csharp
// GlobalUsings.cs
global using System.Text.Json;
global using HealthIndia.Common;
global using static System.Math;
global using Money = System.Decimal;
```

Applies to every file in the project. **Implicit usings** (`<ImplicitUsings>enable</ImplicitUsings>`) auto-generate common ones (`System`, `System.Linq`, `System.Collections.Generic`, `System.IO`, `System.Threading.Tasks`…). Add or remove in the `.csproj`: `<Using Include="HealthIndia.Common" />`.

### Module initializers (C# 9)

A method marked `[ModuleInitializer]` runs **automatically once when the assembly (module) is first loaded**, before any other code in it, with no call needed.

```csharp
using System.Runtime.CompilerServices;

internal static class Startup
{
    [ModuleInitializer]
    internal static void Init()
    {
        AppConfig.Load();                         // e.g. register converters, warm caches
        Console.WriteLine("Module initialised");
    }
}
```

**Rules:** the method must be `static`, parameterless, return `void`, non-generic, and accessible inside the module (`internal` or `public`) in a non-generic class. Several are allowed; their order isn't guaranteed. Mostly used by libraries and source generators; in apps prefer explicit startup code.

|  | Static constructor | Module initializer |
| --- | --- | --- |
| Runs | Before first use of that **class** | When the **assembly** loads |
| Scope | One class | Whole module |
| Trigger | Touching the class | Any code in the module running |

## 2. Command Line Arguments

Values typed after the program name are passed to `Main(string[] args)` (or the `args` variable in top-level statements) as an array of strings split on spaces.

### Part 1: reading arguments

```bash
dotnet run -- report.csv 2026 --verbose
ClaimsApp.exe report.csv 2026 --verbose
ClaimsApp.exe "C:\My Files\report.csv"     # quotes keep spaces in one argument
```

```csharp
static int Main(string[] args)
{
    Console.WriteLine($"Count: {args.Length}");          // 3
    for (int i = 0; i < args.Length; i++)
        Console.WriteLine($"args[{i}] = {args[i]}");     // report.csv, 2026, --verbose

    if (args.Length < 2)
    {
        Console.Error.WriteLine("Usage: ClaimsApp <file> <year> [--verbose]");
        return 1;                                        // non-zero exit code = error
    }
    string file = args[0];
    if (!int.TryParse(args[1], out int year)) { Console.Error.WriteLine("Year must be a number"); return 2; }
    bool verbose = args.Contains("--verbose");
    return 0;
}
```

The program name is **not** in `args` (unlike C); `args` is never null, just empty. In Visual Studio set test arguments in Project Properties > Debug > Command line arguments (or `launchSettings.json`).

### Part 2: options and environment

```csharp
// Simple --key value parsing
var options = new Dictionary<string, string>();
for (int i = 0; i < args.Length - 1; i++)
    if (args[i].StartsWith("--")) options[args[i][2..]] = args[i + 1];
// ClaimsApp --year 2026 --city Pune  ->  options["year"] = "2026", options["city"] = "Pune"

string[] all = Environment.GetCommandLineArgs();   // [0] is the program path, then the args
Environment.ExitCode = 0;
string? env = Environment.GetEnvironmentVariable("CLAIMS_DB");
```

For real tools use the `System.CommandLine` library (options, help text, validation) or `Microsoft.Extensions.Configuration` (command line + appsettings + environment).

## 3. Nullable Reference Types and the Null-Forgiving Operator

With nullable reference types (C# 8, on by default in new projects), the compiler tracks which references **may be null** and warns before you dereference them. A plain `string` means "never null"; `string?` means "may be null". It's compile-time analysis only; runtime behaviour doesn't change.

```csharp
// .csproj: <Nullable>enable</Nullable>   or per file: #nullable enable

public class Patient
{
    public string Name { get; set; }            // warning CS8618: not initialised (non-nullable)
    public string? MiddleName { get; set; }     // OK: may be null
    public Patient(string name) => Name = name; // fixes the warning
}

string? nick = GetNickname();
Console.WriteLine(nick.Length);                 // warning CS8602: possible null dereference
if (nick != null) Console.WriteLine(nick.Length);   // OK: compiler knows it's not null here
Console.WriteLine(nick?.Length ?? 0);           // OK

string name = null;                             // warning CS8600: null to non-nullable
void Greet(string name) { }
Greet(nick);                                    // warning CS8604: possible null argument
```

**The compiler's flow analysis** understands `if (x != null)`, `is not null`, `??`, `?.`, early `return`/`throw`, and `ArgumentNullException.ThrowIfNull(x)`.

### Null-forgiving operator `!`

`x!` tells the compiler "I know this isn't null here, stop warning". It does **nothing at run time**; if you're wrong, you still get a `NullReferenceException`.

```csharp
string? fromConfig = Environment.GetEnvironmentVariable("CLAIMS_DB");
string conn = fromConfig!;                         // silence the warning (risky if unset)

var c = claims.FirstOrDefault(x => x.Id == id)!;   // you've already checked it exists
public Customer Customer { get; set; } = null!;    // common for EF Core navigation properties
```

Use `!` sparingly; prefer a real check (`?? throw new InvalidOperationException(...)`).

**Helpful attributes** (`System.Diagnostics.CodeAnalysis`) teach the compiler about your methods: `[NotNullWhen(true)]` on `TryGet` out parameters, `[MaybeNull]`, `[NotNull]`, `[MemberNotNull]`.

| Syntax | Meaning |
| --- | --- |
| `string` | Should never be null (warnings if it might be) |
| `string?` | May be null; check before use |
| `x!` | Null-forgiving: suppress the warning |
| `x?.Member` | Null-conditional access |
| `x ?? y` | Use `y` when `x` is null |
| `#nullable enable / disable` | Turn checking on/off for part of a file |

## 4. Target-Typed new Expressions (C# 9)

When the type is already known from the left side (or the parameter, return type or property), write `new()` without repeating it.

```csharp
List<Claim> claims = new();                                   // instead of new List<Claim>()
Dictionary<string, List<int>> map = new();
Patient p = new("Rohit", 30);                                 // constructor arguments still allowed
Patient q = new() { Name = "Priya" };                         // with an object initializer
private readonly Dictionary<int, string> _cache = new();     // fields
Point[] points = { new(1, 2), new(3, 4) };                   // array elements

void Admit(Patient patient) { }
Admit(new("Amit", 8));                                        // parameter type known
Patient Create() => new("Neha", 35);                          // return type known
```

**vs `var`:** `var claims = new List<Claim>();` puts the type on the right; `List<Claim> claims = new();` puts it on the left. Both are fine; target-typed `new` is the only option for **fields and properties** (where `var` isn't allowed).

**Not allowed** where the type can't be inferred: `var x = new();` (error), and it can't create an interface or abstract class (`IList<int> l = new();` fails).

## 5. Functions: Partial Method Return Types, Static Anonymous Functions, Lambda Return Types, Constant Interpolated Strings

### Partial methods with return types (C# 9)

A **partial method** is declared in one part of a partial class and implemented in another (often one part is written by a code generator). Originally they had to be `private void` with no `out` parameters, and the call was removed if unimplemented. Since C# 9 they can return values, have `out` parameters and access modifiers, but then **an implementation is required**.

```csharp
// ClaimValidator.generated.cs (e.g. from a source generator)
public partial class ClaimValidator
{
    public partial bool IsValid(Claim c, out string reason);    // declaration only
    partial void OnValidated(Claim c);                         // classic style: optional
}

// ClaimValidator.cs (your code)
public partial class ClaimValidator
{
    public partial bool IsValid(Claim c, out string reason)    // implementation: REQUIRED
    {
        reason = c.Amount > 0 ? "" : "Amount must be positive";
        OnValidated(c);                                         // removed by compiler if not implemented
        return reason == "";
    }
}
```

|  | Classic partial method | Extended (C# 9) |
| --- | --- | --- |
| Return type | `void` only | Any |
| `out` parameters | No | Yes |
| Access modifier | None (implicitly private) | Required (`public`, `private`…) |
| Implementation | Optional (call removed if missing) | Required |

### Static anonymous functions (C# 9)

Adding `static` to a lambda or anonymous method forbids capturing local variables or `this`. That avoids hidden allocations and accidental closures.

```csharp
int threshold = 1000;
Func<int, bool> a = x => x > threshold;            // captures 'threshold' (allocates a closure)
// Func<int, bool> b = static x => x > threshold;  // ERROR: static lambda can't capture
Func<int, bool> c = static x => x > 1000;          // OK: uses only parameters/constants
const int Limit = 1000;
Func<int, bool> d = static x => x > Limit;         // OK: constants are allowed

var adults = patients.Where(static p => p.Age >= 18);
```

### Lambda improvements (C# 10)

```csharp
var parse = (string s) => int.Parse(s);          // natural type: Func<string, int>, so var works
var log = () => Console.WriteLine("hi");         // Action

var choose = object (bool b) => b ? 1 : "one";   // explicit RETURN TYPE before the parameters
var maybe  = int? (string s) => int.TryParse(s, out var n) ? n : null;

var handler = [Obsolete] (int x) => x * 2;       // attributes on lambdas
// var bad = x => x;                             // ERROR: parameter type can't be inferred
```

The explicit return type helps when the compiler can't infer one (different branch types, `null`).

### Constant interpolated strings (C# 10)

An interpolated string can be `const` if **every** placeholder is itself a constant **string**.

```csharp
const string Company = "HealthIndia";
const string Product = "TPA";
const string AppTitle = $"{Company} {Product} Portal";      // OK in C# 10
const string ApiBase = $"/api/{Version}";                   // Version must be a const string
const string Version = "v2";

[Obsolete($"Use {nameof(NewMethod)} instead")]              // usable in attributes
void OldMethod() { }
void NewMethod() { }

// const string Bad = $"Max {100}";                        // ERROR: 100 is an int, not a string
```

## 6. Interface Default Methods, Modifiers, Private Methods and Static Members (C# 8)

Before C# 8 an interface could only declare signatures. Now it can contain **code**, which lets library authors add members to a published interface without breaking every class that implements it.

### Default methods

```csharp
public interface INotifier
{
    void Send(string to, string msg);                                   // abstract: must implement

    void SendBulk(IEnumerable<string> recipients, string msg)           // default implementation
    {
        foreach (var r in recipients) Send(r, msg);
    }
}

public class SmsNotifier : INotifier
{
    public void Send(string to, string msg) => Console.WriteLine($"SMS {to}: {msg}");
    // SendBulk not written: the default is used
}

INotifier n = new SmsNotifier();
n.SendBulk(new[] { "98..", "97.." }, "Claim approved");   // OK through the interface
// new SmsNotifier().SendBulk(...);                       // ERROR: defaults are only visible via the interface type
```

A class can override the default simply by implementing the method itself.

### Modifiers, private and static members

```csharp
public interface IClaimRules
{
    // static members (fields, methods, properties)
    static decimal DefaultLimit { get; } = 500_000m;
    static bool IsHighValue(decimal amount) => amount > DefaultLimit;

    decimal Limit => DefaultLimit;                       // default property

    bool Validate(decimal amount)                        // public by default
    {
        Log($"Validating {amount}");
        return amount > 0 && amount <= Limit;
    }

    private void Log(string m) => Console.WriteLine(m);  // private helper: shared by defaults only
    protected static string Prefix => "[Rules]";         // visible to derived interfaces
    sealed void Audit() => Log("audit");                 // sealed: implementers can't override
}

bool big = IClaimRules.IsHighValue(750_000m);            // call static members via the interface name
```

| Modifier on an interface member | Meaning |
| --- | --- |
| (none) / `public` | Default; part of the contract |
| `abstract` | No body; implementers must provide it (the default for bodiless members) |
| `virtual` | Has a body; implementers may replace it (default for members with a body) |
| `sealed` | Has a body; can't be overridden |
| `private` | Helper for other default members; invisible to implementers |
| `protected`, `internal` | Limited visibility (derived interfaces / same assembly) |
| `static` | Belongs to the interface itself: `IName.Member` |
| `static abstract` (C# 11) | Implementing types must provide a static member (generic math) |

**Still not allowed:** instance fields and instance constructors. Interfaces describe behaviour; state belongs to classes.

**Abstract class vs interface now:** interfaces still can't hold instance state or constructors, and a class can implement many interfaces but inherit only one class. Default methods are mainly for evolving APIs, not a replacement for base classes.

## 7. Index-From-End Operator and the Range Struct (C# 8)

`^n` means "n from the end" (`^1` is the last element). `start..end` builds a **Range** (start inclusive, end **exclusive**). Both are real types: `System.Index` and `System.Range`.

```csharp
int[] a = { 10, 20, 30, 40, 50 };
//  index:   0   1   2   3   4
//  from end ^5  ^4  ^3  ^2  ^1

int last = a[^1];             // 50
int secondLast = a[^2];       // 40
// int bad = a[^0];           // IndexOutOfRangeException: ^0 is one past the end

int[] mid    = a[1..4];       // 20, 30, 40   (end exclusive)
int[] first2 = a[..2];        // 10, 20
int[] from3  = a[3..];        // 40, 50
int[] last3  = a[^3..];       // 30, 40, 50
int[] all    = a[..];         // copy of everything
int[] inner  = a[1..^1];      // 20, 30, 40  (drop first and last)

string id = "CLM-2026-00123";
string year = id[4..8];       // "2026"
string num  = id[^5..];       // "00123"

Index i = ^2;                 // store them in variables
Range r = 1..3;
int x = a[i];                 // 40
int[] slice = a[r];           // 20, 30
Console.WriteLine(r.Start.Value);   // 1

var list = new List<int>(a);
int lastItem = list[^1];      // Index works on List<T>
// var sub = list[1..3];      // Range on List<T> is not supported; use GetRange or LINQ Take(1..3)
Span<int> span = a.AsSpan()[1..3];   // slicing a span doesn't copy
```

**Rules:** works on arrays, strings, `Span<T>` and any type with `Length`/`Count` + indexer (Index) or a `Slice` method (Range). **Slicing arrays and strings creates a copy**; slicing a `Span<T>`/`Memory<T>` doesn't. `a[..^0]` equals `a[..]`.

## 8. Which Version Added What, Cheat Sheet and Interview Questions

| Feature | C# version | .NET |
| --- | --- | --- |
| Nullable reference types, `!` operator | 8 | .NET Core 3.x |
| Default / private / static interface members | 8 | .NET Core 3.x |
| Index `^` and Range `..` | 8 | .NET Core 3.x |
| Switch expressions, property/tuple/positional patterns | 8 | .NET Core 3.x |
| Top-level statements | 9 | .NET 5 |
| Module initializers | 9 | .NET 5 |
| Target-typed `new()` | 9 | .NET 5 |
| Records, `init`, relational/logical patterns | 9 | .NET 5 |
| Static anonymous functions, extended partial methods | 9 | .NET 5 |
| File-scoped namespaces, global using | 10 | .NET 6 |
| Record structs, parameterless struct constructors | 10 | .NET 6 |
| Lambda natural types and return types | 10 | .NET 6 |
| Constant interpolated strings, extended property patterns | 10 | .NET 6 |

**Cheat sheet**

```csharp
// Program.cs (top-level)
using static System.Console;
WriteLine($"args: {args.Length}");

namespace App;                          // file-scoped (in other files)
global using System.Text.Json;          // GlobalUsings.cs
[ModuleInitializer] internal static void Init() { }

string? maybe = null;  string sure = maybe ?? "default";  string forced = maybe!;
List<int> nums = new();
Func<int, int> sq = static x => x * x;
var parse = int (string s) => int.Parse(s);
const string Title = $"{Company} Portal";
interface IX { void M() => Console.WriteLine("default"); static int Count => 0; private void Help() { } }
int last = arr[^1];  int[] part = arr[1..^1];
```

**Interview questions (short answers)**

1. **What are top-level statements?** Writing program code directly in one file; the compiler generates `Main`. Only one file per project.
2. **File-scoped vs block-scoped namespace?** `namespace X;` applies to the whole file without braces.
3. **What is global using?** A using directive that applies to all files in the project.
4. **What is a module initializer?** A `[ModuleInitializer]` static method that runs once when the assembly loads.
5. **What are nullable reference types?** Compile-time annotations (`string?`) and warnings that help prevent NullReferenceException.
6. **What does `!` do?** Suppresses a nullable warning; it has no runtime effect.
7. **What is target-typed new?** `new()` without the type when the compiler already knows it.
8. **What changed for partial methods in C# 9?** They can return values, have out parameters and access modifiers, but must then be implemented.
9. **What is a static lambda?** A lambda marked `static` that can't capture locals or `this`.
10. **Can a lambda declare its return type?** Yes since C# 10: `var f = int (string s) => ...;`.
11. **What is a constant interpolated string?** A `const string` built with `$"..."` from other constant strings (C# 10).
12. **Why default interface methods?** To add members to an interface without breaking existing implementers.
13. **Can interfaces have private or static members?** Yes since C# 8; still no instance fields or constructors.
14. **What do `^1` and `1..^1` mean?** The last element; everything except the first and last.
15. **Is the end of a range inclusive?** No, exclusive.
