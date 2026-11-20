# C# 13 Features

C# 13 shipped with .NET 9 in November 2024.

## 1. Params Collections

Before C# 13, `params` only worked with arrays (`params int[]`). Now it works with **any collection type** that a collection expression can create: `Span<T>`, `ReadOnlySpan<T>`, `List<T>`, `IEnumerable<T>`, `IReadOnlyList<T>`, and more.

```csharp
// Old: always allocates an array on the heap
static int SumOld(params int[] values) => values.Sum();

// C# 13: ReadOnlySpan can live on the stack: no allocation for small argument lists
static int Sum(params ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (var v in values) total += v;
    return total;
}

// Accept any sequence, including LINQ results
static void LogAll(params IEnumerable<string> messages)
{
    foreach (var m in messages) Console.WriteLine(m);
}

static void AddTags(params List<string> tags) => tags.ForEach(Console.WriteLine);

Sum(1, 2, 3);                         // 6: arguments packed into a span
Sum();                                // 0
LogAll("start", "processing", "done");
LogAll(claims.Select(c => c.Id));     // pass a whole sequence directly
AddTags("urgent", "cashless");
```

**Why it matters:** `params ReadOnlySpan<T>` avoids a heap allocation on every call, which helps hot paths. .NET 9 added span overloads to many BCL methods (`string.Join`, `Task.WhenAll`, `Path.Combine`), so existing code gets faster just by recompiling.

**Rules:** still one `params` parameter and it must be last; the type must be a span, an array, or a type usable with collection expressions; if overloads exist for both, the compiler prefers span over array.

## 2. Partial Properties and Indexers

Partial **methods** have existed for years; C# 13 allows partial **properties** and **indexers**. One part of a `partial` class **declares** the property (often generated code or a source-generator contract), and another part **implements** it.

```csharp
// Patient.Declarations.cs: the declaring part (looks like an auto-property but isn't one)
public partial class Patient
{
    public partial string Name { get; set; }
    public partial string this[int index] { get; }
}

// Patient.cs: the implementing part
public partial class Patient
{
    private string _name = "";
    private readonly string[] _tags = { "vip", "cashless" };

    public partial string Name
    {
        get => _name;
        set => _name = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
    }

    public partial string this[int index] => _tags[index];
}
```

**Rules:** exactly one declaring and one implementing declaration; both must have the same type, accessors and modifiers; the implementing part can't be an auto-property (it needs bodies, or use `field` in C# 14); attributes from both parts are merged.

**Main use:** source generators. For example, the MVVM Toolkit's `[ObservableProperty] public partial string Name { get; set; }` lets the generator write the change-notification code, and `[GeneratedRegex]` can now be applied to a partial property: `[GeneratedRegex(@"^\d{6}$")] private static partial Regex PinRegex { get; }`.

## 3. The field Keyword (Preview in C# 13, Released in C# 14)

`field` refers to the **compiler-generated backing field** of a property, so you can add logic to one accessor while keeping the auto-property convenience, with no manual `_name` field.

```csharp
// Before: you had to declare the backing field yourself
private string _name = "";
public string Name
{
    get => _name;
    set => _name = value.Trim();
}

// With field
public string Name
{
    get;                                   // auto-implemented getter
    set => field = value.Trim();           // 'field' is the hidden backing field
} = "";

public int Age
{
    get;
    set => field = value is >= 0 and <= 120 ? value : throw new ArgumentOutOfRangeException(nameof(value));
}

public string Email { get => field ?? "not provided"; set; }   // logic in the getter only
```

In C# 13 (.NET 9) this needed `<LangVersion>preview</LangVersion>`; it's a standard feature from C# 14 (.NET 10). See the C# 14 tab for more examples.

**Watch out:** if your class already has a member named `field`, use `@field` or `this.field` to refer to your own member inside accessors.

## 4. Implicit Index Access in Object Initializers

The **from-end index operator `^`** can now be used inside object initializers to set elements of a collection member.

```csharp
public class Countdown
{
    public int[] Slots { get; } = new int[5];
}

var timer = new Countdown
{
    Slots =
    {
        [^1] = 0,      // last element
        [^2] = 1,
        [^3] = 2,
        [^4] = 3,
        [^5] = 4       // first element
    }
};
// Slots = { 4, 3, 2, 1, 0 }

// Before C# 13 this was an error inside initializers; you had to write
// var t = new Countdown(); t.Slots[^1] = 0; ...
```

It works with any member that supports an `Index` indexer (arrays, `List<T>`, spans inside ref structs). It's a small convenience, mainly for initialising buffers or fixed-size arrays from the end.

## 5. Other C# 13 Features, Cheat Sheet and Interview Questions

**Also in C# 13:** a new `System.Threading.Lock` type used automatically by `lock (myLock)`; the `\e` escape for the ESC character; `ref` locals and `unsafe` code allowed in async methods and iterators (with limits); `ref struct` types can implement interfaces and be used as generic arguments via the `allows ref struct` anti-constraint; `[OverloadResolutionPriority]` attribute; better method-group natural types.

```csharp
static int Sum(params ReadOnlySpan<int> xs) { int t = 0; foreach (var x in xs) t += x; return t; }
public partial string Name { get; set; }               // declaring part
public partial string Name { get => _n; set => _n = value; }   // implementing part
public string Code { get; set => field = value.ToUpper(); }   // field keyword (preview in 13)
var c = new Countdown { Slots = { [^1] = 0 } };         // ^ in initializers
private readonly Lock _lock = new(); lock (_lock) { }   // new Lock type
```

**Interview questions (short answers)**

1. **What are params collections?** `params` with spans, lists or other collection types, not just arrays (C# 13).
2. **Why use `params ReadOnlySpan<T>`?** Avoids allocating an array on every call.
3. **What are partial properties?** Properties declared in one part of a partial type and implemented in another, mainly for source generators.
4. **What does the `field` keyword refer to?** The compiler-generated backing field of a property (preview in C# 13, released in C# 14).
5. **What does implicit index access in initializers allow?** Using `^` (from-end) indexes inside object initializers.
6. **What is the new Lock type?** `System.Threading.Lock`, a dedicated, faster lock object that `lock` uses directly.
7. **Which .NET shipped C# 13?** .NET 9.
