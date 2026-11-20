# C# 11 Features

C# 11 shipped with .NET 7 in November 2022. This tab covers the features in your course list plus a short overview of the rest.

## 1. Overview of C# 11

| Feature | One-line summary |
| --- | --- |
| Raw string literals | `"""..."""` strings with no escaping, multi-line, clean indentation |
| List patterns | Match arrays/lists by elements: `[1, 2, ..]` |
| Slice pattern | `..` inside a list pattern matches zero or more elements, optionally captured |
| `var` pattern in lists | Capture elements: `[var first, .., var last]` |
| File-local types | `file class Helper` is visible only inside its source file |
| Required members | `required` properties must be set when the object is created |
| Auto-default structs | Struct constructors no longer have to assign every field |
| Ref fields | `ref` fields inside `ref struct` types; `scoped` keyword |
| Also in C# 11 | Generic math (`static abstract` interface members), UTF-8 literals `"abc"u8`, generic attributes `[MyAttr<T>]`, newlines inside interpolation holes, `nameof` of parameters in attributes, unsigned right shift `>>>`, `checked` user-defined operators |

## 2. Raw String Literals

A raw string starts and ends with **three or more double quotes**. Everything inside is taken literally: no escape sequences, quotes and backslashes allowed as-is. Perfect for JSON, SQL, XML, regex and file paths.

```csharp
string json = """
    {
      "claimId": "C1",
      "path": "C:\Claims\2026",
      "note": "Patient said "urgent""
    }
    """;
// Output starts at '{': the indentation before the closing """ is removed from every line

string sql = """
    SELECT Id, Amount
    FROM Claims
    WHERE Status = 'Approved'
    """;

string single = """He said "hi" and left C:\temp""";    // single-line form
```

**Rules for multi-line raw strings**

- The opening `"""` must be followed by a newline; the content starts on the next line.
- The closing `"""` must be on its own line; **its column sets the indentation that's removed** from every line (lines can't be indented less than it).
- The first and last newlines are not part of the value.
- To include `"""` inside, start and end with more quotes: `""""` … `""""`.

### Interpolated raw strings

```csharp
string name = "Rohit"; decimal amt = 25000m;

string a = $"""
    Patient: {name}
    Amount: {amt:N0}
    """;

// Need literal { } in the text (JSON!)? Use $$: then {{ }} marks interpolation, single braces are literal
string b = $$"""
    { "patient": "{{name}}", "amount": {{amt}} }
    """;
// { "patient": "Rohit", "amount": 25000 }
```

The number of `$` signs = how many braces start an interpolation hole.

|  | Regular `"..."` | Verbatim `@"..."` | Raw `"""..."""` |
| --- | --- | --- | --- |
| Escapes like `\n` | Processed | Not processed | Not processed |
| Backslashes | Must be `\\` | Literal | Literal |
| Double quotes | `\"` | `""` | Literal |
| Multi-line | No | Yes (keeps indentation) | Yes (indentation trimmed) |
| Literal braces with interpolation | `{{`/`}}` | `{{`/`}}` | Use `$$` |

## 3. List, Slice and var Patterns

### List pattern

Match a sequence **element by element** inside `[ ]`. Works on arrays, `List<T>`, strings, spans: anything with `Length`/`Count` and an `int` indexer.

```csharp
int[] nums = { 1, 2, 3 };

nums is [1, 2, 3];          // true: exact elements, exact length
nums is [1, _, 3];          // true: _ matches any single element
nums is [_, _];             // false: length must match (3 ≠ 2)
nums is [> 0, > 0, > 0];    // true: each element can be any pattern
nums is [1 or 2, ..];       // true

string Describe(int[] a) => a switch
{
    []        => "Empty",
    [var x]   => $"One item: {x}",
    [var x, var y] => $"Two items: {x}, {y}",
    _         => $"{a.Length} items"
};
```

### Slice pattern (`..`)

`..` matches **zero or more** elements, and can appear **once** per list pattern.

```csharp
nums is [1, ..];            // starts with 1
nums is [.., 3];            // ends with 3
nums is [1, .., 3];         // starts with 1 and ends with 3 (any length ≥ 2)
nums is [..];               // any sequence (not null)

// Capture the slice into a variable (needs Range support: arrays, strings, spans)
if (nums is [var first, .. var middle, var last])
    Console.WriteLine($"{first} | {string.Join(",", middle)} | {last}");   // 1 | 2 | 3

// Strings are char sequences
string code = "CLM-2026";
if (code is ['C', 'L', 'M', '-', .. var year]) Console.WriteLine(year);   // 2026
```

### var pattern

`var x` always matches (including null) and **captures** the value. Most useful inside list, positional and property patterns.

```csharp
string[] args = { "--year", "2026" };
string msg = args switch
{
    ["--year", var y] when int.TryParse(y, out var yr) => $"Year {yr}",
    ["--help", ..]    => "Help",
    [var cmd, ..]     => $"Unknown command {cmd}",
    []                => "No arguments"
};

if (claim is { Amount: var amt } && amt > 50_000) { }    // capture a property value
```

**Practical uses:** parsing command-line arguments, CSV rows (`[var id, var name, var amount]`), protocol messages, checking prefixes/suffixes, recursive list processing (`[var head, .. var tail]`).

## 4. File-Local Types and Required Members

### File-local types

The `file` modifier limits a type's visibility to **the single source file** where it's declared. Other files (even in the same project) can't see it, so two files can each have a private helper with the same name.

```csharp
// ClaimExporter.cs
public class ClaimExporter
{
    public string Export(Claim c) => CsvFormatter.Format(c);
}

file static class CsvFormatter                     // only usable inside ClaimExporter.cs
{
    public static string Format(Claim c) => $"{c.Id},{c.Amount}";
}

// PolicyExporter.cs
file static class CsvFormatter { /* different helper, no name clash */ }
```

**Rules:** applies to top-level types (class, struct, interface, enum, record, delegate); a file-local type can't appear in the signature of a public member of a non-file type; mainly used by **source generators** to emit helpers without colliding with user code.

### Required members

`required` forces callers to set a property or field in the **object initializer**; forgetting it is a compile error, not a runtime null.

```csharp
public class Patient
{
    public required string Uhid { get; init; }
    public required string Name { get; init; }
    public int Age { get; init; }                 // optional
}

var ok = new Patient { Uhid = "U1", Name = "Rohit" };   // fine
// var bad = new Patient { Name = "Rohit" };           // ERROR CS9035: required member 'Uhid' must be set
```

**With a constructor that sets them**, tell the compiler with `[SetsRequiredMembers]`:

```csharp
using System.Diagnostics.CodeAnalysis;

public class Patient
{
    public required string Uhid { get; init; }
    public required string Name { get; init; }

    public Patient() { }

    [SetsRequiredMembers]
    public Patient(string uhid, string name) => (Uhid, Name) = (uhid, name);
}

var p = new Patient("U2", "Priya");           // OK: constructor promises to set them
```

**Rules:** works on properties and fields (with `init` or `set`); can't be used on static members or in interfaces; a required member must be at least as accessible as its type; derived classes inherit the requirement. Combine with nullable reference types: `required string Name` removes the "non-nullable not initialised" warning without `= null!`.

## 5. Auto-Default Structs and Ref Fields

### Auto-default structs

Before C# 11, a struct constructor had to assign **every** field (error CS0171). Now the compiler automatically sets any field you don't assign to its default value.

```csharp
public struct Measurement
{
    public double Value;
    public string Unit;
    public DateTime TakenAt;

    public Measurement(double value)
    {
        Value = value;          // Unit and TakenAt not assigned:
    }                           // C# 10: error CS0171; C# 11: auto-set to null and DateTime.MinValue
}
```

This makes struct constructors as convenient as class constructors and simplifies code with field initializers (C# 10).

### Ref fields

A `ref struct` (a stack-only struct, like `Span<T>`) can now contain **`ref` fields**: fields that store a reference to another variable rather than a copy. This is how `Span<T>` itself is implemented.

```csharp
public ref struct CounterRef
{
    private ref int _value;                       // ref field: points at someone else's int

    public CounterRef(ref int target) => _value = ref target;

    public void Increment() => _value++;          // changes the original variable
    public int Current => _value;
}

int visits = 0;
var counter = new CounterRef(ref visits);
counter.Increment();
counter.Increment();
Console.WriteLine(visits);                       // 2: the original was modified
```

**`scoped` keyword:** marks a ref or ref-struct parameter/local as not escaping the current method, so the compiler can prove safety:

```csharp
void Process(scoped Span<int> buffer) { /* buffer can't be stored in a field or returned */ }
```

**Rules:** ref fields are allowed **only in `ref struct` types**; a `ref struct` can't be boxed, stored in a class field, used in async methods or lambdas that capture it; `ref readonly` fields give read-only access. This is a low-level, performance feature mostly for library authors (parsers, serializers, `Span`-based APIs).

## 6. Cheat Sheet and Interview Questions

```csharp
string raw = """
    {"path": "C:\temp"}
    """;
string json = $$"""{"name": "{{name}}"}""";

bool b1 = arr is [1, .., 9];
if (arr is [var head, .. var tail]) { }
var desc = list switch { [] => "empty", [_] => "one", _ => "many" };

file class Helper { }
public required string Name { get; init; }
[SetsRequiredMembers] public Patient(string n) => Name = n;

public ref struct R { private ref int _x; }
```

**Interview questions (short answers)**

1. **What is a raw string literal?** A `"""`-delimited string with no escape processing, ideal for JSON/SQL/regex.
2. **How is indentation handled in raw strings?** The closing `"""`'s column is stripped from every line.
3. **How do you interpolate JSON with braces in a raw string?** Use `$$"""`; then `{{x}}` is interpolation and single braces are literal.
4. **What is a list pattern?** A pattern that matches a sequence's elements: `[1, 2, 3]`.
5. **What does `..` do in a list pattern?** Matches zero or more elements (once per pattern); `.. var s` captures the slice.
6. **What is the var pattern?** `var x` matches anything and captures it, useful inside other patterns.
7. **What is a file-local type?** A type declared with `file`, visible only in its source file.
8. **What does `required` do?** Forces callers to set the member in the object initializer at compile time.
9. **What is `[SetsRequiredMembers]`?** Marks a constructor that sets all required members so callers can skip the initializer.
10. **What are auto-default structs?** Struct constructors no longer need to assign every field; unassigned ones get defaults.
11. **What is a ref field?** A field in a `ref struct` that holds a reference to another variable (how Span\<T> works).
12. **Which .NET version shipped C# 11?** .NET 7 (November 2022).
