# Tuples, Deconstruction & Discards

Grouping a few values without writing a class: the old `Tuple` class, modern value tuples, unpacking them with deconstruction, and ignoring what you don't need with discards.

## 1. The Tuple Class

`System.Tuple<T1, …>` (.NET 4) is a generic **class** that holds a fixed number of values of possibly different types. Its elements are read-only and always named `Item1`, `Item2`, …, which makes code hard to read.

```csharp
Tuple<int, string, decimal> claim = new Tuple<int, string, decimal>(101, "Rohit", 25000m);
var same = Tuple.Create(101, "Rohit", 25000m);          // factory infers types

Console.WriteLine(claim.Item1);     // 101
Console.WriteLine(claim.Item2);     // Rohit
Console.WriteLine(claim);           // (101, Rohit, 25000)
// claim.Item1 = 102;               // ERROR: read-only

Console.WriteLine(claim.Equals(same));   // True: value equality via Equals

// Returning several values (old style)
Tuple<int, int> MinMax(int[] a) => Tuple.Create(a.Min(), a.Max());
var mm = MinMax(new[] { 4, 9, 1 });
Console.WriteLine($"{mm.Item1} - {mm.Item2}");      // 1 - 9
```

**Limits:** up to 7 elements directly; the 8th parameter `TRest` must be another tuple for more. It's a reference type (heap allocation). Element names can't be changed from `Item1`… It's mostly seen in older code; use **value tuples** today.

## 2. Value Tuples

Value tuples (C# 7, `System.ValueTuple`) are lightweight **structs** with built-in syntax `( , )`, **named elements**, and mutable fields. They're the modern way to return or pass a few values together.

```csharp
(int, string) t1 = (101, "Rohit");                    // unnamed: Item1, Item2 still work
(int Id, string Name) t2 = (101, "Rohit");            // named elements
var t3 = (Id: 101, Name: "Rohit", Amount: 25000m);    // names on the right

Console.WriteLine(t2.Name);            // Rohit
Console.WriteLine(t2.Item2);           // Rohit: ItemN always available
t2.Name = "Priya";                     // mutable fields

string city = "Pune"; int pin = 411001;
var loc = (city, pin);                 // names inferred: loc.city, loc.pin (C# 7.1)
```

**Returning multiple values: the main use**

```csharp
(decimal Min, decimal Max, decimal Avg) Stats(IEnumerable<decimal> amounts)
{
    var list = amounts.ToList();
    return (list.Min(), list.Max(), list.Average());
}

var s = Stats(new[] { 1200m, 5000m, 800m });
Console.WriteLine($"{s.Min} {s.Max} {s.Avg:F0}");    // 800 5000 2333
```

**More uses**

```csharp
(a, b) = (b, a);                                     // swap without a temp variable
var lookup = new Dictionary<(string City, int Year), int>();   // composite key
lookup[("Pune", 2026)] = 42;
foreach (var (key, value) in lookup) { }
List<(string Name, int Age)> people = [("Rohit", 30), ("Priya", 62)];
```

**Equality:** `==` and `!=` compare element by element (C# 7.3); names are ignored, only positions and values matter: `(1, "a") == (Id: 1, Name: "a")` is `true`.

|  | `Tuple<…>` (class) | `ValueTuple` / `( , )` |
| --- | --- | --- |
| Kind | Reference type (heap) | Value type (struct) |
| Syntax | `Tuple.Create(1, "a")` | `(1, "a")` |
| Element names | Only `Item1`… | Custom names + `Item1`… |
| Mutable | No (read-only properties) | Yes (public fields) |
| `==` compares values | No (references) | Yes |
| Deconstruction | Yes | Yes |
| Performance | Allocation per tuple | No allocation |
| Use today | Legacy APIs | Default choice |

**When not to use tuples:** if the data is used in many places, crosses public APIs, or needs behaviour, create a `record` or class with proper names instead.

## 3. Deconstructing

Deconstruction unpacks a tuple (or any object that supports it) into separate variables in one statement.

```csharp
var (min, max, avg) = Stats(amounts);          // new variables, types inferred
(decimal lo, decimal hi, decimal mean) = Stats(amounts);   // explicit types

decimal x, y, z;
(x, y, z) = Stats(amounts);                     // into existing variables

var (id, name) = (101, "Rohit");                // directly from a tuple literal

foreach (var (city, count) in cityCounts) { }   // in loops (dictionaries, lists of tuples)
```

### Deconstructing your own types

Add a `Deconstruct` method with `out` parameters, and any class or struct becomes deconstructable.

```csharp
public class Patient
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public string City { get; init; } = "";

    public void Deconstruct(out string name, out int age) { name = Name; age = Age; }
    public void Deconstruct(out string name, out int age, out string city)   // overload by count
        => (name, age, city) = (Name, Age, City);
}

var p = new Patient { Name = "Rohit", Age = 30, City = "Pune" };
var (n, a) = p;              // uses the 2-parameter Deconstruct
var (n2, a2, c2) = p;        // uses the 3-parameter one
```

Records with positional parameters get `Deconstruct` automatically: `public record Plan(string Code, decimal Premium);` then `var (code, premium) = plan;`. `KeyValuePair<K,V>` and `DateTime`-like types in newer .NET also provide one. An extension method named `Deconstruct` adds it to types you don't own.

**Pattern matching uses it too:** `if (p is ("Rohit", > 18)) { }` (positional pattern).

## 4. Discards

A discard `_` is a write-only placeholder for a value you **don't need**. It makes intent clear and avoids unused variables.

```csharp
var (_, max, _) = Stats(amounts);                 // only need max

if (int.TryParse(input, out _))                   // only care whether it parses
    Console.WriteLine("Valid number");

_ = SendEmailAsync();                             // deliberately ignore a returned value / task

string size = qty switch
{
    < 10 => "Small",
    < 100 => "Medium",
    _ => "Large"                                  // discard pattern = "anything else"
};

if (obj is Patient { Age: > 60 } _) { }            // type check without a variable

button.Click += (_, _) => Console.WriteLine("Clicked");   // unused lambda params (C# 9)
```

**Rules:** `_` holds nothing and can't be read; you can use several `_` in one statement; if a real variable named `_` is already in scope, `_` refers to that variable instead (avoid naming variables `_`).

## 5. Cheat Sheet, Checklist and Interview Questions

```csharp
var old = Tuple.Create(1, "a");            // Tuple class: old.Item1
(int Id, string Name) t = (1, "Rohit");    // value tuple with names
var u = (Id: 2, Name: "Priya");
(int Min, int Max) MinMax(int[] a) => (a.Min(), a.Max());   // return several values
var (lo, hi) = MinMax([4, 9, 1]);          // deconstruct
(a, b) = (b, a);                           // swap
var (_, onlyHi) = MinMax([3, 7]);          // discard
bool ok = int.TryParse("5", out _);
void Deconstruct(out string n, out int a) { n = Name; a = Age; }   // in your class
```

**Checklist**

- [ ] Create a Tuple class instance and read Item1/Item2
- [ ] Create named value tuples and return one from a method
- [ ] List four differences between Tuple and ValueTuple
- [ ] Deconstruct into new and existing variables
- [ ] Add a Deconstruct method to a class
- [ ] Use discards in deconstruction, out parameters, switch expressions and lambdas

**Interview questions (short answers)**

1. **What is a tuple?** A lightweight grouping of a fixed number of values, possibly of different types.
2. **Tuple vs ValueTuple?** Tuple: class, read-only, only Item1… names. ValueTuple: struct, mutable, custom names, `==` support, faster.
3. **How do you return multiple values from a method?** A value tuple (preferred), `out` parameters, or a class/record.
4. **Are value tuple names kept at run time?** No; they're compile-time only, the fields are still Item1, Item2…
5. **What is deconstruction?** Unpacking a tuple or object into separate variables: `var (a, b) = t;`.
6. **How do you make a class deconstructable?** Add a `Deconstruct` method with `out` parameters.
7. **What is a discard?** `_`, a placeholder for a value you intentionally ignore.
8. **How do you swap two variables with tuples?** `(a, b) = (b, a);`
9. **Tuple vs anonymous type?** A tuple can be returned and is a mutable struct; an anonymous type is a read-only class used locally.
10. **When should you use a record instead of a tuple?** When the data is shared widely, part of a public API, or needs behaviour.
