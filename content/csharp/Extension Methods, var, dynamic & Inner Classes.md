# Extension Methods, var, dynamic & Inner Classes

Four language features: adding methods to existing types, letting the compiler infer types (`var`), skipping compile-time checks (`dynamic`), and classes declared inside classes.

## 1. Extension Methods

An extension method adds a new method to an existing type (even `string`, `int` or a sealed class you don't own) **without** changing or inheriting it. It's a static method whose first parameter is marked with `this`.

```csharp
namespace HealthIndia.Extensions;

public static class StringExtensions             // 1. static class (non-generic, top-level)
{
    public static bool IsValidPan(this string s)  // 2. static method, 3. 'this' on first parameter
        => s.Length == 10 && char.IsLetter(s[0]);

    public static string Mask(this string s, int visible = 4)
        => new string('*', Math.Max(0, s.Length - visible)) + s[^Math.Min(visible, s.Length)..];

    public static int WordCount(this string s)
        => s.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
}

public static class NumberExtensions
{
    public static bool IsEven(this int n) => n % 2 == 0;
    public static decimal WithGst(this decimal amount, decimal rate = 0.18m) => amount * (1 + rate);
}
```

```csharp
using HealthIndia.Extensions;                    // 4. import the namespace to see them

"ABCDE1234F".IsValidPan();          // true: looks like an instance method
"9876543210".Mask();                // ******3210
"Claim approved today".WordCount(); // 3
7.IsEven();                         // false
1000m.WithGst();                    // 1180

StringExtensions.Mask("9876543210");   // what the compiler actually calls
```

**Rules**

- Must be in a **static, non-nested, non-generic** class; the method must be **static**.
- First parameter has `this` + the type being extended; extra parameters come after it.
- Only visible where its namespace is imported.
- Can access only the **public** (or internal) members of the extended type; no private access, so it isn't real inheritance.
- If the type already has an instance method with the same signature, the **instance method wins**; the extension is silently ignored.
- Works on interfaces too: that's how **LINQ** works (`Where`, `Select` are extensions on `IEnumerable<T>`).
- Generic extensions are allowed: `public static bool IsIn<T>(this T x, params T[] set) => set.Contains(x);`

**C# 14 extension members:** the new `extension` block syntax can also add extension **properties** and static members; the classic `this` syntax above still works and is what most code uses.

## 2. Implicitly Typed (var) and Dynamically Typed (dynamic) Variables

`var` lets the **compiler infer** a fixed type from the initializer; it's still fully statically typed. `dynamic` **turns off** compile-time type checking; the type is resolved at run time, and mistakes only show up as runtime exceptions.

### var (C# 3)

```csharp
var count = 10;                       // int
var name = "Rohit";                   // string
var claims = new List<Claim>();       // List<Claim>
var total = 10 * 1.5;                 // double

// count = "ten";                     // ERROR: count is int forever
// var x;                             // ERROR: must be initialised
// var y = null;                      // ERROR: no type to infer
// var a = 1, b = 2;                  // ERROR: one variable per var

var anon = new { Id = 1, Name = "Pen" };   // anonymous type: var is REQUIRED
foreach (var c in claims) { }              // common in loops
var q = claims.Where(c => c.Amount > 1000); // LINQ results
```

**Use `var`** when the type is obvious from the right side (`new`, casts) or very long; **avoid it** when it hides the type (`var x = Calculate();` — what is x?). It's only for local variables: not fields, parameters or return types.

### dynamic (C# 4)

```csharp
dynamic d = 10;            // currently int
d = d + 5;                 // 15: resolved at run time
d = "now a string";        // allowed: type can change
Console.WriteLine(d.ToUpper());   // works because d is a string now

d.FlyToMoon();             // COMPILES, but throws RuntimeBinderException at run time

dynamic bag = new System.Dynamic.ExpandoObject();   // add members at run time
bag.Name = "Rohit";
bag.Age = 30;
Console.WriteLine(bag.Name);   // Rohit
// classic use: Office COM automation, e.g. dynamic excel = Activator.CreateInstance(type);
```

**Use `dynamic`** for COM interop (Excel/Word automation), `ExpandoObject`, interop with dynamic languages (IronPython), or reflection-heavy code. **Avoid it** in normal code: no IntelliSense, no compile-time errors, slower calls, and value types get boxed.

|  | `var` | `dynamic` | `object` |
| --- | --- | --- | --- |
| Type decided | Compile time (inferred) | Run time | Compile time (`object`) |
| Can change type later | No | Yes | Can hold anything, but static type stays `object` |
| Must initialise | Yes | No | No |
| Member calls checked | At compile time | At run time only | Only `object` members; cast first |
| IntelliSense | Full | None | Only `object` members |
| Errors show up | Compile time | Run time (`RuntimeBinderException`) | Run time (bad casts) |
| Allowed as field/param/return | No (locals only) | Yes | Yes |
| Performance | Same as explicit type | Slower (runtime binder) | Boxing/casting cost |

## 3. Inner (Nested) Classes

An inner class is a class declared **inside another class**. Use it for a helper type that only makes sense as part of its outer class, keeping it hidden from the rest of the code.

```csharp
public class Hospital
{
    private readonly List<Ward> _wards = new();
    private static int _totalBeds = 0;

    public Ward AddWard(string name, int beds)
    {
        var w = new Ward(this, name, beds);
        _wards.Add(w);
        return w;
    }

    public class Ward                           // nested class: full name Hospital.Ward
    {
        private readonly Hospital _owner;       // needs a reference to reach the outer object
        public string Name { get; }
        public int Beds { get; }

        public Ward(Hospital owner, string name, int beds)
        {
            _owner = owner; Name = name; Beds = beds;
            _totalBeds += beds;                 // CAN access outer private static members
            Console.WriteLine(_owner._wards.Count);   // and private instance members via a reference
        }
    }

    private class AuditLog { }                  // private nested: invisible outside Hospital
}

var h = new Hospital();
Hospital.Ward icu = h.AddWard("ICU", 12);      // refer to it as Outer.Inner
var general = new Hospital.Ward(h, "General", 40);
// var log = new Hospital.AuditLog();          // ERROR: private nested class
```

**Rules**

- Nested types default to **`private`** (top-level classes default to `internal`). Any access modifier is allowed.
- The inner class can access **all** members of the outer class, including private ones, but needs an instance reference for instance members (C# inner classes aren't tied to an outer object automatically, unlike Java's).
- The outer class can only reach the inner class's public/internal members.
- From outside, write `Outer.Inner` (or import with `using static Outer;`).
- Inheriting the outer class doesn't give you a copy of the inner class; it's just a type name scoped inside the outer one.
- Structs, interfaces, enums and records can be nested too: `public enum Status` inside `Claim` is common.

**Typical uses:** builder classes (`Order.Builder`), private helper/node types (`LinkedList<T>.Node`), enumerators, and grouping constants (`Claim.Status.Pending`).

## 4. Cheat Sheet, Checklist and Interview Questions

```csharp
// EXTENSION METHOD
public static class Ext { public static bool IsAdult(this int age) => age >= 18; }
bool ok = 21.IsAdult();

// VAR: inferred, fixed type
var list = new List<string>();
var person = new { Name = "Rohit", Age = 30 };   // anonymous type

// DYNAMIC: resolved at run time
dynamic d = 5;  d = "text";  int len = d.Length;  // compiles; fails at run time if wrong

// NESTED CLASS
public class Order { public class Line { public string Item = ""; } }
var line = new Order.Line();
```

**Checklist**

- [ ] Write two extension methods and call them like instance methods
- [ ] State the three requirements for an extension method
- [ ] Explain why LINQ is built on extension methods
- [ ] List where `var` is not allowed
- [ ] Show a `dynamic` call that compiles but fails at run time
- [ ] Compare var, dynamic and object
- [ ] Write a nested class that uses the outer class's private member

**Interview questions (short answers)**

1. **What is an extension method?** A static method with `this` on its first parameter that can be called as if it were an instance method of that type.
2. **Requirements?** Static non-generic non-nested class, static method, `this` on the first parameter, namespace imported.
3. **Can extension methods access private members?** No; only accessible members of the extended type.
4. **Extension vs instance method with the same signature?** The instance method wins.
5. **What is `var`?** An implicitly typed local; the compiler infers a fixed type from the initializer.
6. **Is `var` dynamically typed?** No; it's statically typed at compile time.
7. **When is `var` required?** For anonymous types.
8. **What is `dynamic`?** A type whose operations are resolved at run time, bypassing compile-time checks.
9. **`var` vs `dynamic`?** var: compile-time type, can't change, IntelliSense; dynamic: run-time binding, can change, errors at run time.
10. **`dynamic` vs `object`?** object requires casts to use members; dynamic lets you call any member and checks it at run time.
11. **What is a nested class?** A class declared inside another class; it can access the outer class's private members.
12. **Default access of a nested class?** `private`.
