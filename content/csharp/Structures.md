# Structures

Structs are lightweight value types: how to create them, how they differ from classes in practice, readonly structs, and why `int` and `double` are structs too.

## 1. Creating Structures

A struct is declared like a class with `struct` instead of `class`, but it is a **value type**: the variable holds the data itself, and assigning or passing it makes a copy.

```csharp
public struct Point
{
    public int X;                              // fields
    public int Y;

    public double DistanceFromOrigin()         // methods
        => Math.Sqrt(X * X + Y * Y);

    public override string ToString() => $"({X}, {Y})";
}

// Way 1: without new: assign every field before use
Point p1;
p1.X = 3;
p1.Y = 4;
Console.WriteLine(p1.DistanceFromOrigin());   // 5

// Way 2: with new: fields start at default (0)
Point p2 = new Point();          // (0, 0)
var p3 = new Point { X = 1, Y = 2 };          // object initializer works too
```

**A struct can contain:** fields, properties, methods, constructors, indexers, events, operators, constants, and it can implement interfaces.

**A struct cannot:** inherit from another struct or class (it always derives from `System.ValueType`), be inherited from (structs are implicitly `sealed`), or declare a finalizer. Members can't be `protected` or `virtual` (no inheritance).

**Good candidates for a struct:** small (roughly 16 bytes or less), short-lived, logically a single value, ideally immutable: `Point`, `Money`, `Coordinate`, `DateRange`, `RgbColor`. .NET's own: `DateTime`, `TimeSpan`, `Guid`, `KeyValuePair<K,V>`.

## 2. Structures with Constructors

Structs can have constructors to set their fields in one step. Every struct also always has an implicit parameterless constructor that sets all fields to their defaults.

```csharp
public struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)    // parameterized constructor
    {
        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other) =>
        other.Currency == Currency
            ? new Money(Amount + other.Amount, Currency)
            : throw new InvalidOperationException("Currency mismatch");
}

var fee = new Money(500, "INR");
var total = fee.Add(new Money(250, "INR"));      // 750 INR
var empty = new Money();                          // Amount 0, Currency null
Money zero = default;                             // same as new Money()
```

**Rules (C# 11+)**

- A struct constructor must leave every field assigned; since C# 11, any field you don't set is auto-set to its default.
- Since C# 10 you can write your own parameterless constructor and field initializers: `public int Count = 1;`
- `default(Money)` and `new Money[5]` **don't** run your parameterless constructor; they always produce all-zero values.
- Constructor chaining with `: this(...)` works; `: base(...)` doesn't (no inheritance).
- No static constructor restrictions beyond those for classes; a struct can have one.

**Older C# (before 10/11):** a parameterless constructor couldn't be written, and every field had to be assigned explicitly in each constructor. You'll still see this rule in older tutorials and interviews.

## 3. Structure vs Class in Practice

The one difference that matters day to day: assigning or passing a struct **copies the whole value**, while a class copies only the **reference** to one shared object.

&#91;embedded content: Assignment: struct copies vs class references\]

Run this yourself with `struct PointS` and `class PointC` side by side:

```csharp
struct PointS { public int X, Y; }
class  PointC { public int X, Y; }

var s1 = new PointS { X = 1 };
var s2 = s1;  s2.X = 99;
Console.WriteLine(s1.X);   // 1   (copy)

var c1 = new PointC { X = 1 };
var c2 = c1;  c2.X = 99;
Console.WriteLine(c1.X);   // 99  (shared)

void Move(PointS p) => p.X = 500;        // changes the method's copy only
void Move(PointC p) => p.X = 500;        // changes the caller's object

// PointS ns = null;   // ERROR: a struct can't be null (use PointS? for nullable)
PointC nc = null;   // fine
```

|  | Struct | Class |
| --- | --- | --- |
| Kind | Value type | Reference type |
| Variable holds | The data | A reference to data on the heap |
| Stored | Inline: stack for locals, inside the containing object/array | Heap |
| Assignment / passing | Copies all fields | Copies the reference |
| `null` | Not allowed (unless `T?`) | Allowed |
| Default value | All fields zero | `null` |
| Inheritance | No (only interfaces) | Yes |
| `protected`, `virtual`, `abstract` | Not allowed | Allowed |
| Finalizer | No | Yes |
| `==` by default | Not defined (use `Equals`, which compares fields) | Compares references |
| Garbage collection | Not individually collected | Collected by the GC |
| Best for | Small, immutable values | Entities with identity and behaviour |

**Performance trade-off:** structs avoid heap allocation and GC pressure (great in big arrays of points), but large structs are slow to copy. Keep them small or pass with `in`.

**Watch out for boxing:** putting a struct in an `object` or non-generic collection copies it to the heap. `object o = s1;` creates a separate boxed copy.

## 4. Readonly Structures

A `readonly struct` (C# 7.2) is **immutable**: every field must be `readonly` and every property get-only or `init`. Once created, its value never changes, so copies can never drift apart.

```csharp
public readonly struct Coordinate
{
    public double Lat { get; }
    public double Lng { get; }

    public Coordinate(double lat, double lng)
    {
        Lat = lat;
        Lng = lng;
    }

    // "Changing" returns a NEW value instead of mutating
    public Coordinate WithLat(double lat) => new Coordinate(lat, Lng);

    // public double Alt;                 // ERROR: fields must be readonly
    // public void Move() { Lat = 0; }    // ERROR: can't assign in a readonly struct
}

var pune = new Coordinate(18.52, 73.85);
var north = pune.WithLat(19.0);          // pune is unchanged
```

**Why use it**

- **Safety:** no accidental changes to a copy you thought was the original.
- **Performance:** passing a readonly struct with `in` avoids hidden "defensive copies" the compiler otherwise makes to protect it.
- **Clarity:** it tells readers this type is a value, like `int` or `DateTime`.

**Related features**

- `readonly` **members** (C# 8): mark single methods on a normal struct as non-mutating: `public readonly double Length() => ...;`
- `with` expressions work on structs (C# 10): `var b = a with { X = 5 };` creates a modified copy.
- `readonly record struct` (C# 10) gives you an immutable struct with value equality, `ToString` and `with` for free:

```csharp
public readonly record struct Money(decimal Amount, string Currency);
var a = new Money(100, "INR");
var b = a with { Amount = 200 };
Console.WriteLine(a == b);     // False: compares values
```

## 5. Primitive Types as Structures

Every C# numeric type, `bool` and `char` is actually a **struct** in the .NET library. The keyword is just an alias: `int` is `System.Int32`. That's why primitives are value types and have methods and constants.

| C# keyword | .NET struct |
| --- | --- |
| `int` | `System.Int32` |
| `long` | `System.Int64` |
| `short` / `byte` | `System.Int16` / `System.Byte` |
| `double` / `float` | `System.Double` / `System.Single` |
| `decimal` | `System.Decimal` |
| `bool` | `System.Boolean` |
| `char` | `System.Char` |
| Also structs | `DateTime`, `TimeSpan`, `Guid`, `Nullable<T>` |
| **Not** structs | `string`, `object`, arrays: these are classes (reference types) |

```csharp
int a = 5;
System.Int32 b = 5;                  // exactly the same type
Console.WriteLine(a.GetType());      // System.Int32
Console.WriteLine(typeof(int).IsValueType);   // True

// Because they're structs, they have members:
int max = int.MaxValue, min = int.MinValue;   // constants
int.Parse("42");  int.TryParse(s, out var n);   // static methods
42.ToString("D5");  5.CompareTo(7);  // instance methods
char.IsDigit('7');  double.IsNaN(x);

// Value-type behaviour
int x = 10;
int y = x;  y = 20;
Console.WriteLine(x);                // 10: copied, not shared
```

**Interview point:** primitives are structs that the runtime and compiler treat specially (literals, CPU instructions for arithmetic, constant folding), but conceptually they follow every struct rule: value semantics, no inheritance, sealed, default value 0/false. `int?` is shorthand for the struct `Nullable<int>`.

## 6. Important Points, Cheat Sheet, Checklist and Interview Questions

**Points to remember**

1. A struct is a **value type**; assignment and parameter passing copy the whole value.
2. Structs implicitly derive from `System.ValueType` and are implicitly `sealed`; they can implement interfaces but not inherit.
3. A struct variable can't be `null` unless declared nullable (`Point?`).
4. Default value of a struct = all fields zero/false/null; `default` and arrays never run your constructor.
5. Structs can have constructors (parameterless ones since C# 10), properties, methods, operators and static members.
6. Keep structs small (around 16 bytes) and preferably immutable: `readonly struct`.
7. Large structs are costly to copy: pass with `in`.
8. Boxing a struct (to `object` or an interface) copies it to the heap.
9. All primitive types except `string` and `object` are structs (`int` = `System.Int32`).
10. Use a class when you need identity, inheritance, shared mutable state, or large data.

**Cheat sheet**

```csharp
public readonly struct Temperature : IComparable<Temperature>
{
    public double Celsius { get; }
    public Temperature(double c) => Celsius = c;
    public double Fahrenheit => Celsius * 9 / 5 + 32;
    public int CompareTo(Temperature other) => Celsius.CompareTo(other.Celsius);
    public override string ToString() => $"{Celsius:F1} °C";
}

var t1 = new Temperature(36.6);
var t2 = t1;                       // full copy
Temperature t0 = default;          // 0 °C, constructor not run
Temperature? none = null;          // nullable struct
public readonly record struct Pin(string Code);   // immutable + value equality
```

**Checklist**

- [ ] Create a struct with fields and methods; use it with and without `new`
- [ ] Add a parameterized constructor and explain the C# 10/11 constructor rules
- [ ] Predict output of struct vs class assignment (the diagram in section 3)
- [ ] Write a `readonly struct` with a `With...` method
- [ ] Map five keywords to their .NET struct names
- [ ] List when to choose a struct over a class

**Interview questions (short answers)**

1. **What is a struct?** A user-defined value type that holds its data directly.
2. **Struct vs class?** Value type, copied on assignment, no inheritance, can't be null vs reference type, shared reference, inheritance, can be null.
3. **Where are structs stored?** Inline: on the stack as locals, or inside the object/array that contains them.
4. **Can a struct inherit?** No; it can only implement interfaces.
5. **Can a struct have a parameterless constructor?** Yes, since C# 10; before that, no.
6. **Can a struct be null?** Only as `Nullable<T>` (`T?`).
7. **What is a readonly struct?** An immutable struct whose fields are all readonly; avoids defensive copies.
8. **Why is `int` a struct?** `int` is an alias for `System.Int32`, a value type defined as a struct.
9. **Is `string` a struct?** No, it's a class (immutable reference type).
10. **When should you use a struct?** For small, immutable, value-like data that is created often.
11. **What is boxing?** Converting a value type to `object` (or an interface), copying it to the heap.
12. **What is `default` for a struct?** An instance with every field set to its default value.
