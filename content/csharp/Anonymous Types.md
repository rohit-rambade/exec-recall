# Anonymous Types

Quick, unnamed, read-only types created on the spot with `new { ... }`, plus nesting them and putting them in arrays.

## 1. Anonymous Types

An anonymous type is a class the **compiler creates for you** from a `new { ... }` expression. It has no name you can write, so you hold it with `var`. It's ideal for a quick, local bundle of values, especially LINQ results.

```csharp
var patient = new { Id = 101, Name = "Rohit", Age = 30 };

Console.WriteLine(patient.Name);        // Rohit: strongly typed, IntelliSense works
Console.WriteLine(patient);             // { Id = 101, Name = Rohit, Age = 30 }: generated ToString
Console.WriteLine(patient.GetType().Name);   // <>f__AnonymousType0`3: compiler-made name

// patient.Age = 31;                    // ERROR: properties are read-only
```

**Property names can be inferred** from variables or members (projection initializers):

```csharp
string city = "Pune";
int pin = 411001;
var address = new { city, pin };        // properties named city and pin
var summary = new { patient.Name, patient.Age };   // Name and Age
```

**Non-destructive change with `with` (C# 10):**

```csharp
var older = patient with { Age = 31 };  // new object; patient unchanged
```

**Value equality:** the compiler generates `Equals` and `GetHashCode` comparing every property, and two anonymous objects with the **same property names, types and order** in the same assembly share one type.

```csharp
var a = new { Id = 1, Name = "X" };
var b = new { Id = 1, Name = "X" };
var c = new { Name = "X", Id = 1 };     // different ORDER = different type

Console.WriteLine(a.Equals(b));          // True: compares values
Console.WriteLine(a == b);               // False: == compares references for anonymous types
// a = c;                                // ERROR: different types
```

**Main use: LINQ projections**

```csharp
var report = claims
    .Where(c => c.Status == "Approved")
    .Select(c => new { c.Id, c.PatientName, Gst = c.Amount * 0.18m })
    .ToList();

foreach (var r in report) Console.WriteLine($"{r.Id}: {r.PatientName} {r.Gst}");

var byCity = patients.GroupBy(p => new { p.City, p.Gender });   // composite keys
```

**Rules**

- Only **properties** (public, read-only); no methods, events or fields of your own.
- Values can't be `null` literals without a type (`new { X = null }` fails; use `(string?)null`), and can't be method groups or pointers.
- The type derives directly from `object` and is `internal` and sealed.
- You can't use it as a method's parameter or return type by name: returning one means `object` or `dynamic`, so create a real class, record or tuple instead when data leaves the method.

## 2. Nested Anonymous Types

A property of an anonymous type can itself be an anonymous object, giving you quick hierarchical data, like a small inline JSON document.

```csharp
var claim = new
{
    Id = "CLM-101",
    Amount = 25000m,
    Patient = new                      // nested anonymous type
    {
        Name = "Rohit",
        Age = 30,
        Address = new { City = "Pune", Pin = "411001" }   // nested again
    },
    Hospital = new { Name = "City Care", Network = true }
};

Console.WriteLine(claim.Patient.Name);            // Rohit
Console.WriteLine(claim.Patient.Address.City);    // Pune
Console.WriteLine(claim);
// { Id = CLM-101, Amount = 25000, Patient = { Name = Rohit, Age = 30, Address = { City = Pune, Pin = 411001 } }, Hospital = { ... } }

// Great for building API payloads
string json = JsonSerializer.Serialize(claim);
```

**Points:** every level is read-only; equality is deep (nested anonymous objects compare their own properties too); LINQ often produces them when grouping: `.Select(g => new { Dept = g.Key, Stats = new { Count = g.Count(), Avg = g.Average(x => x.Salary) } })`.

## 3. Anonymous Arrays

An implicitly typed array `new[] { ... }` lets the compiler infer the element type. Combined with anonymous types, it gives you an array of anonymous objects, which is handy for quick test data or lookup tables.

```csharp
var numbers = new[] { 1, 2, 3 };                 // int[]
var mixed   = new[] { 1, 2.5, 3 };               // double[]: best common type
// var bad  = new[] { 1, "two" };                // ERROR: no best type

var plans = new[]
{
    new { Code = "GOLD",   Premium = 12000m, Cover = 1_000_000 },
    new { Code = "SILVER", Premium = 8000m,  Cover = 500_000 },
    new { Code = "BRONZE", Premium = 5000m,  Cover = 300_000 }
};

foreach (var p in plans)
    Console.WriteLine($"{p.Code}: {p.Premium:N0} for {p.Cover:N0}");

var cheapest = plans.OrderBy(p => p.Premium).First();   // LINQ works
Console.WriteLine(plans.Length);                         // 3
plans[0] = new { Code = "PLATINUM", Premium = 20000m, Cover = 2_000_000 };   // OK: same shape
```

**Rule:** every element must have the **same property names, types and order**, so they're all the same anonymous type. If one element writes `Premium = 8000` (an `int`) while the others use `decimal`, the compiler sees two different types and refuses to build the array.

Anonymous objects can also go into lists via LINQ (`.ToList()`) or with a template trick, but if you need a growable collection you'd usually define a record.

## 4. Anonymous Types vs Tuples vs Records, Cheat Sheet and Interview Questions

|  | Anonymous type | Value tuple | Record |
| --- | --- | --- | --- |
| Syntax | `new { Id = 1, Name = "X" }` | `(Id: 1, Name: "X")` | `record Person(int Id, string Name);` |
| Has a name you can write | No (use `var`) | Yes: `(int, string)` | Yes |
| Kind | Class (reference) | Struct (value) | Class or struct |
| Mutable | No | Yes (fields) | No by default |
| Value equality | `Equals` yes, `==` no | Yes | Yes, including `==` |
| Return from a method | Not practically | Yes | Yes |
| Best for | LINQ projections inside a method | Returning 2–3 values quickly | Real data models, DTOs |

**Cheat sheet**

```csharp
var p = new { Id = 1, Name = "Rohit" };              // anonymous type
var q = new { p.Name, City = "Pune" };               // inferred property name
var r = p with { Name = "Priya" };                    // modified copy
var n = new { Id = 2, Info = new { Age = 30 } };     // nested
var arr = new[] { new { A = 1 }, new { A = 2 } };    // anonymous array
var proj = list.Select(x => new { x.Id, Total = x.Qty * x.Price });   // LINQ
bool same = p.Equals(new { Id = 1, Name = "Rohit" }); // true
```

**Checklist**

- [ ] Create an anonymous type, read its properties and print it
- [ ] Explain why properties are read-only and how `with` helps
- [ ] Show that `Equals` is true but `==` is false for equal anonymous objects
- [ ] Use an anonymous type in a LINQ Select and GroupBy
- [ ] Build a nested anonymous object
- [ ] Build an anonymous array and explain the "same shape" rule
- [ ] Choose between anonymous type, tuple and record

**Interview questions (short answers)**

1. **What is an anonymous type?** A compiler-generated, unnamed class created with `new { ... }`, holding read-only properties.
2. **Why must you use `var`?** The type has no name you can write in code.
3. **Can you modify its properties?** No; they're read-only. Use `with` for a modified copy.
4. **Can it have methods?** No, only properties (the compiler adds Equals, GetHashCode, ToString).
5. **How is equality handled?** `Equals` compares all property values; `==` compares references.
6. **When are two anonymous objects the same type?** Same property names, types and order, in the same assembly.
7. **Can you return an anonymous type from a method?** Only as `object`/`dynamic`; use a tuple or record instead.
8. **Where are anonymous types used most?** LINQ projections (`Select`) and composite GroupBy keys.
9. **What is an anonymous (implicitly typed) array?** `new[] { ... }` with the element type inferred; with anonymous objects, all elements must share one shape.
10. **Anonymous type vs tuple?** Anonymous type: reference type, read-only, method-local. Tuple: value type, mutable, can be returned.
