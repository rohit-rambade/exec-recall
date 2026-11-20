# C# 12 Features

C# 12 shipped with .NET 8 (LTS) in November 2023.

## 1. Primary Constructors in Classes

Records had primary constructors since C# 9; C# 12 brings them to **every class and struct**. Parameters declared after the type name are in scope **throughout the class body**.

```csharp
// Before
public class ClaimService
{
    private readonly IClaimRepository _repo;
    private readonly ILogger<ClaimService> _logger;
    public ClaimService(IClaimRepository repo, ILogger<ClaimService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
    public Claim? Get(string id) { _logger.LogInformation("Get {Id}", id); return _repo.Find(id); }
}

// C# 12
public class ClaimService(IClaimRepository repo, ILogger<ClaimService> logger)
{
    public Claim? Get(string id)
    {
        logger.LogInformation("Get {Id}", id);     // parameters used directly in members
        return repo.Find(id);
    }
}
```

**Initialising properties and validating**

```csharp
public class Policy(string number, decimal premium)
{
    public string Number { get; } = number;                       // copy into a property
    public decimal Premium { get; } = premium >= 0 ? premium
        : throw new ArgumentOutOfRangeException(nameof(premium));
    public string Plan { get; init; } = "Basic";

    public Policy(string number) : this(number, 1000m) { }        // other ctors MUST call the primary one
}

public class Doctor(string name, string dept) : Person(name)      // pass to a base class
{
    public string Dept => dept;
}
```

**Class vs record primary constructors**

|  | Record `record P(string Name)` | Class `class P(string name)` |
| --- | --- | --- |
| Generates public properties | Yes | **No**: parameters are just captured variables |
| Mutable | Properties are `init` | Parameters can be reassigned inside the class |
| `readonly` | n/a | Can't be marked readonly (assign to a `readonly` field if needed) |
| Equality, ToString, Deconstruct | Generated | Not generated |

**Gotchas:** if you both copy a parameter into a property **and** use the parameter in a method, the class stores it twice and the two can drift apart (compiler warning CS9124); naming convention is camelCase for these parameters; great for dependency injection in services and controllers.

## 2. Collection Expressions

A single, uniform syntax `[ ... ]` creates arrays, lists, spans and other collections. The **target type** decides what gets built.

```csharp
int[] a = [1, 2, 3];                       // array
List<string> cities = ["Pune", "Delhi"];   // List<T>
Span<int> s = [1, 2, 3];                   // Span (may be stack-allocated)
IEnumerable<int> e = [4, 5];               // interface: compiler picks a type
ImmutableArray<int> im = [1, 2];           // immutable collections too
HashSet<int> set = [1, 2, 2];              // {1, 2}
int[] empty = [];                          // empty collection (no allocation for arrays)

// Spread operator .. inlines another collection
int[] first = [1, 2];
int[] second = [3, 4];
int[] all = [.. first, .. second, 5];      // 1, 2, 3, 4, 5
List<string> withHeader = ["Id,Amount", .. rows];

// As arguments and return values
PrintAll(["a", "b"]);
static int[] Empty() => [];

void PrintAll(IEnumerable<string> items) { foreach (var i in items) Console.WriteLine(i); }
```

**Rules:** there must be a target type (`var x = [1, 2];` is an error, since there's no natural type yet); works for arrays, `Span<T>`/`ReadOnlySpan<T>`, types with a collection initializer (`Add` method), and types marked with `[CollectionBuilder]`; dictionaries aren't supported with this syntax yet.

| Old | C# 12 |
| --- | --- |
| `new int[] { 1, 2, 3 }` | `[1, 2, 3]` |
| `new List<int> { 1, 2 }` | `[1, 2]` |
| `Array.Empty<int>()` / `new List<int>()` | `[]` |
| `a.Concat(b).ToArray()` | `[.. a, .. b]` |

## 3. Default Parameters in Lambdas and Alias Any Type

### Default (optional) parameters in lambda expressions

Lambdas can now have default parameter values (and `params` arrays), just like methods.

```csharp
var addTax = (decimal amount, decimal rate = 0.18m) => amount * (1 + rate);
addTax(1000);          // 1180
addTax(1000, 0.05m);   // 1050

var greet = (string name = "Guest") => $"Hello, {name}";
greet();               // Hello, Guest

var sum = (params int[] values) => values.Sum();
sum(1, 2, 3);          // 6
```

The lambda's natural type becomes a compiler-generated delegate that keeps the default; assigning it to `Func<decimal, decimal, decimal>` loses the default (Func has no optional parameters).

### Alias any type

`using` aliases (previously only for named types and namespaces) can now name **any type**: tuples, arrays, pointers, generic instantiations, nullable value types.

```csharp
using Point = (int X, int Y);
using Grid = int[,];
using ClaimMap = System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<Claim>>;
using MaybeInt = int?;
global using Money = decimal;                       // project-wide

Point origin = (0, 0);
Point Move(Point p, int dx) => (p.X + dx, p.Y);
Grid board = new int[3, 3];
Money fee = 500m;
```

**Rules:** can't alias nullable **reference** types (`using S = string?;` is an error); aliases are file-scoped unless `global`; unsafe pointer aliases need `using unsafe`.

## 4. Other C# 12 Features, Cheat Sheet and Interview Questions

**Also in C# 12:** inline arrays (`[InlineArray(10)] struct Buffer { int _e; }`, fixed-size stack buffers), `ref readonly` parameters, the `[Experimental]` attribute, and interceptors (preview, used by source generators).

```csharp
public class Service(IRepo repo) { public Item? Get(int id) => repo.Find(id); }   // primary ctor
int[] nums = [1, 2, 3];  List<int> more = [.. nums, 4];  int[] none = [];        // collections
var f = (int x, int y = 10) => x + y;                                            // default lambda param
using Point = (int X, int Y);                                                    // alias any type
```

**Interview questions (short answers)**

1. **What is a primary constructor in a class?** Parameters on the class declaration, available throughout the class body (C# 12).
2. **Do class primary constructors create properties?** No; only records do. Assign them to properties yourself.
3. **What must other constructors do?** Chain to the primary constructor with `: this(...)`.
4. **What is a collection expression?** `[a, b, c]` syntax that builds whatever collection type is expected.
5. **What does `..` do inside a collection expression?** Spreads (inlines) another collection's elements.
6. **Why can't you write `var x = [1, 2];`?** Collection expressions need a target type.
7. **Can lambdas have default parameters?** Yes, since C# 12: `(int x = 5) => x`.
8. **What does alias any type allow?** `using` aliases for tuples, arrays, generics and nullable value types.
9. **Which .NET shipped C# 12?** .NET 8 (LTS).
