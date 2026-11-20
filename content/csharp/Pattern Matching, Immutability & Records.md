# Pattern Matching, Immutability & Records

Testing the shape of data concisely with patterns, designing objects that can't change, and records: the C# type built for immutable data.

## 1. Why Pattern Matching; Type, Switch-Case and When Patterns

Pattern matching lets you **test a value's type, shape or contents and extract parts of it in one expression**, replacing chains of `is` + cast + `if` with short, declarative code. The compiler also checks that patterns are reachable and (for switch expressions) exhaustive.

**Without patterns (old style)**

```csharp
if (shape is Circle)
{
    Circle c = (Circle)shape;                 // test, then cast separately
    if (c.Radius > 10) Console.WriteLine("Big circle");
}
else if (shape is Rectangle) { var r = (Rectangle)shape; /* ... */ }
```

**With patterns**

```csharp
if (shape is Circle { Radius: > 10 } c) Console.WriteLine($"Big circle {c.Radius}");
```

Sample types used below:

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Rectangle(double Width, double Height) : Shape;
public record Triangle(double Base, double Height) : Shape;
```

### Type pattern (with declaration)

```csharp
object o = 42;
if (o is int n) Console.WriteLine(n + 1);        // test + cast + new variable n
if (o is not string) Console.WriteLine("Not text");
if (o is null) { }                                 // null check that can't be fooled by an overloaded ==
if (shape is Circle or Rectangle) { }              // C# 9: combine types
```

The variable (`n`) is only definitely assigned where the pattern matched.

### Switch-case pattern (switch statement with patterns)

```csharp
switch (shape)
{
    case Circle c:
        Console.WriteLine($"Circle area {Math.PI * c.Radius * c.Radius:F1}");
        break;
    case Rectangle r:
        Console.WriteLine($"Rectangle area {r.Width * r.Height}");
        break;
    case null:
        Console.WriteLine("No shape");
        break;
    default:
        Console.WriteLine("Unknown shape");
        break;
}
```

Cases are tested **in order**; the compiler errors if a case can never match because an earlier one covers it.

### When pattern (case guards)

```csharp
switch (shape)
{
    case Rectangle r when r.Width == r.Height:
        Console.WriteLine("Square");
        break;
    case Rectangle r:
        Console.WriteLine("Rectangle");
        break;
    case Circle c when c.Radius == 0:
        Console.WriteLine("Just a point");
        break;
}
```

`when` adds any boolean condition after the pattern matches; put guarded cases before the general case of the same type.

## 2. Switch Expression, Relational/Logical, Property, Extended Property, Tuple and Positional Patterns

### Switch expression pattern (C# 8)

A `switch` that **returns a value**, with `pattern => result` arms and `_` as the catch-all.

```csharp
double Area(Shape s) => s switch
{
    Circle c    => Math.PI * c.Radius * c.Radius,
    Rectangle r => r.Width * r.Height,
    Triangle t  => 0.5 * t.Base * t.Height,
    null        => throw new ArgumentNullException(nameof(s)),
    _           => throw new NotSupportedException()
};
```

If no arm matches at run time, it throws `SwitchExpressionException`; the compiler warns when arms aren't exhaustive.

### Relational and logical patterns (C# 9)

Relational: `<`, `<=`, `>`, `>=` against constants. Logical: `and`, `or`, `not`, plus parentheses.

```csharp
string AgeGroup(int age) => age switch
{
    < 0               => throw new ArgumentOutOfRangeException(nameof(age)),
    < 18              => "Minor",
    >= 18 and < 60    => "Adult",
    _                 => "Senior"
};

bool IsLetter(char c) => c is (>= 'a' and <= 'z') or (>= 'A' and <= 'Z');
bool valid = input is not null and not "";
if (score is > 90 or 100) { }
```

### Property pattern (C# 8)

Match on an object's properties with `{ Property: pattern }`. It also implies a **non-null** check.

```csharp
public record Claim(string Id, decimal Amount, string Status, Patient Patient);
public record Patient(string Name, int Age, Address Address);
public record Address(string City, string State);

decimal CoPay(Claim c) => c switch
{
    { Status: "Rejected" }                 => 0m,
    { Amount: > 100_000, Patient: { Age: >= 60 } } => 0.05m,   // nested property pattern
    { Amount: > 100_000 }                  => 0.10m,
    { }                                    => 0.20m,           // any non-null claim
    null                                   => throw new ArgumentNullException(nameof(c))
};

if (claim is { Status: "Pending", Amount: > 50_000 } big) Console.WriteLine(big.Id);
```

### Extended property pattern (C# 10)

Reach nested properties with dots instead of nested braces:

```csharp
// C# 8/9
if (claim is { Patient: { Address: { City: "Pune" } } }) { }
// C# 10
if (claim is { Patient.Address.City: "Pune" }) { }

string Region(Claim c) => c switch
{
    { Patient.Address.State: "MH" or "GA" } => "West",
    { Patient.Address.State: "DL" }         => "North",
    _                                       => "Other"
};
```

### Tuple pattern (C# 8)

Switch on **several values at once** by putting them in a tuple.

```csharp
string RockPaperScissors(string a, string b) => (a, b) switch
{
    ("rock", "scissors") or ("scissors", "paper") or ("paper", "rock") => "Player A wins",
    var (x, y) when x == y => "Draw",
    _ => "Player B wins"
};

string Discount(bool isMember, int qty) => (isMember, qty) switch
{
    (true, >= 10) => "20%",
    (true, _)     => "10%",
    (false, >= 10) => "5%",
    _             => "0%"
};
```

### Positional pattern (C# 8)

Uses a type's `Deconstruct` method (records have one automatically) to match by position.

```csharp
string Classify(Shape s) => s switch
{
    Circle(0)                    => "Point",
    Circle(> 100)                => "Huge circle",
    Rectangle(var w, var h) when w == h => $"Square {w}",
    Rectangle(_, 0) or Rectangle(0, _)  => "Line",
    Rectangle(var w, var h)      => $"Rectangle {w}x{h}",
    _                            => "Other"
};

public record Point(int X, int Y);
string Quadrant(Point p) => p switch
{
    (0, 0)           => "Origin",
    ( > 0, > 0)      => "Q1",
    ( < 0, > 0)      => "Q2",
    ( < 0, < 0)      => "Q3",
    ( > 0, < 0)      => "Q4",
    _                => "On an axis"
};
```

### All patterns at a glance

| Pattern | Syntax example | Since |
| --- | --- | --- |
| Constant | `x is 0`, `case "Pune":`, `null` | C# 7 |
| Type / declaration | `o is int n`, `case Circle c:` | C# 7 |
| `var` | `case var x when x > 5:` | C# 7 |
| Discard | `_` | C# 8 |
| Switch expression | `x switch { ... => ..., _ => ... }` | C# 8 |
| Property | `{ Amount: > 1000 }` | C# 8 |
| Tuple | `(a, b) switch { (1, 2) => ... }` | C# 8 |
| Positional | `Point(0, _)` | C# 8 |
| Relational | `< 18`, `>= 60` | C# 9 |
| Logical | `and`, `or`, `not` | C# 9 |
| Extended property | `{ Patient.Address.City: "Pune" }` | C# 10 |
| List | `[1, 2, ..]`, `[var first, ..]` | C# 11 |

## 3. Immutability

An **immutable** object can't change after it's created; to "change" it you make a new object. `string`, `DateTime` and records are immutable.

### Need of immutability

```csharp
public class Policy { public decimal Premium { get; set; } }

var p = new Policy { Premium = 5000 };
var cache = new Dictionary<string, Policy> { ["P1"] = p };
ApplyDiscount(p);                           // some method quietly sets p.Premium = 0
// every holder of p (the cache, other threads, UI) now sees 0, and nobody knows who changed it
```

**Benefits:** thread-safe sharing without locks; safe as dictionary keys and in hash sets; easier reasoning and debugging (state can't drift); validation happens once in the constructor; no defensive copies needed; natural fit for value objects (Money, Address, DateRange) and DTOs.

### Immutable classes (classic way)

```csharp
public sealed class Money
{
    public decimal Amount { get; }               // get-only: set only in the constructor
    public string Currency { get; }
    private readonly List<string> _notes;        // even private state must not change

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Amount = amount; Currency = currency;
        _notes = new List<string>();
    }

    public Money Add(Money other) => new(Amount + other.Amount, Currency);   // returns NEW object
    public IReadOnlyList<string> Notes => _notes.AsReadOnly();              // never expose the mutable list
}
```

**Recipe:** get-only or `init` properties; `readonly` fields; validate in the constructor; methods return new instances; don't expose mutable collections (use `IReadOnlyList<T>` or `ImmutableList<T>`); `sealed` stops a subclass adding mutable state.

### Init-only properties (C# 9)

`init` allows setting a property **in an object initializer or constructor**, then locks it. Immutable objects no longer need a big constructor.

```csharp
public class Patient
{
    public string Uhid { get; init; } = "";
    public string Name { get; init; } = "";
    public required DateOnly Dob { get; init; }      // C# 11: must be set
}

var p = new Patient { Uhid = "U1", Name = "Rohit", Dob = new(1996, 3, 15) };
// p.Name = "X";                                    // ERROR CS8852: init-only
```

### Readonly structs (C# 7.2)

```csharp
public readonly struct Coordinate
{
    public double Lat { get; }
    public double Lng { get; }
    public Coordinate(double lat, double lng) => (Lat, Lng) = (lat, lng);
    public Coordinate WithLat(double lat) => new(lat, Lng);
}
```

The compiler enforces that all fields are `readonly`, and it can skip defensive copies when passing with `in`. (More in the Structures tab.)

### Parameterless struct constructors (C# 10)

Structs can now declare their own parameterless constructor and field initializers:

```csharp
public struct Pager
{
    public int PageSize { get; init; } = 20;          // field/property initializer (C# 10)
    public int Page { get; init; }
    public Pager() { Page = 1; }                      // custom parameterless constructor
}

var a = new Pager();          // PageSize 20, Page 1: constructor runs
var b = default(Pager);       // PageSize 0,  Page 0: default NEVER runs constructors
var arr = new Pager[3];       // all elements are default (0, 0)
```

It must be `public`; remember `default` and arrays bypass it.

## 4. Records

A **record** (C# 9) is a class designed for immutable data. In one line the compiler generates properties, a constructor, value-based equality, `ToString`, `Deconstruct` and support for `with` expressions.

### Basics

```csharp
public record Patient(string Uhid, string Name, int Age);   // positional record: that's the whole type

var p = new Patient("U1", "Rohit", 30);
Console.WriteLine(p.Name);           // Rohit: public init-only properties were generated
Console.WriteLine(p);                // Patient { Uhid = U1, Name = Rohit, Age = 30 }

// Nominal (non-positional) record: write properties yourself
public record Address
{
    public string City { get; init; } = "";
    public string Pin { get; init; } = "";
}
```

**What the compiler generates for a positional record:** a primary constructor, public `init` properties, `Equals`/`GetHashCode`/`==`/`!=` (value-based), `ToString`, `Deconstruct`, a copy constructor + clone method for `with`, and `PrintMembers`.

### Nested records

```csharp
public record Hospital(string Name, Address Address);
public record Claim(string Id, Patient Patient, Hospital Hospital, decimal Amount);

var claim = new Claim("C1", p, new Hospital("City Care", new Address { City = "Pune" }), 25000m);
Console.WriteLine(claim.Hospital.Address.City);   // Pune
Console.WriteLine(claim);   // nested ToString: Claim { Id = C1, Patient = Patient { ... }, Hospital = ... }
```

### Immutability

```csharp
// p.Name = "X";        // ERROR: init-only property
```

Records are **shallowly** immutable: a `List<string>` property can't be replaced, but its contents can still change. Use `IReadOnlyList<T>`/immutable collections for full immutability. You can also add mutable `{ get; set; }` properties to a record, but that defeats the purpose.

### Equality

```csharp
var a = new Patient("U1", "Rohit", 30);
var b = new Patient("U1", "Rohit", 30);

Console.WriteLine(a == b);                    // True: compares all property VALUES
Console.WriteLine(a.Equals(b));               // True
Console.WriteLine(ReferenceEquals(a, b));     // False: still two objects
int count = new HashSet<Patient> { a, b }.Count;   // 1
```

Equality also requires the **same runtime type** (via a generated `EqualityContract`), so a `Patient` never equals a subclass instance with the same values. Collections inside records compare by reference.

### with expression (non-destructive mutation)

```csharp
var older = a with { Age = 31 };              // copy of a, Age changed; a is untouched
var moved = claim with { Hospital = claim.Hospital with { Name = "Ruby Hall" } };   // nested change
var clone = a with { };                       // shallow copy
```

`with` works on records, record structs, structs and anonymous types (C# 10). It copies, then runs the `init` setters you list.

### Deconstruct

```csharp
var (uhid, name, age) = a;                    // generated Deconstruct for positional records
if (a is ("U1", _, >= 18)) Console.WriteLine("Adult U1");   // positional pattern
```

### ToString

```csharp
Console.WriteLine(a);    // Patient { Uhid = U1, Name = Rohit, Age = 30 }

public record Card(string Number)
{
    public override string ToString() => $"Card ****{Number[^4..]}";        // replace completely
}
public record Member(string Name, string Pan)
{
    protected virtual bool PrintMembers(StringBuilder sb)                   // control which members print
    {
        sb.Append($"Name = {Name}");                                         // hide Pan
        return true;
    }
}
```

### Constructors

```csharp
public record Policy(string Number, decimal Premium)
{
    // validate the positional parameters
    public decimal Premium { get; init; } = Premium >= 0 ? Premium : throw new ArgumentOutOfRangeException(nameof(Premium));

    public string Plan { get; init; } = "Basic";             // extra property

    public Policy(string number) : this(number, 1000m) { }  // extra constructor MUST chain to the primary one
}

var p1 = new Policy("P1", 5000m) { Plan = "Gold" };
var p2 = new Policy("P2");                                  // Premium 1000
```

Additional constructors must call the primary constructor with `: this(...)`. You can redeclare a generated property (as with `Premium` above) to add validation or change its accessor.

## 5. Record Inheritance, Sealed ToString and Record Structs

### Inheritance

A record can inherit from another record (not from a class, and a class can't inherit from a record).

```csharp
public abstract record Person(string Name);
public record Doctor(string Name, string Speciality) : Person(Name);
public record Nurse(string Name, string Ward) : Person(Name);

Person d = new Doctor("Neha", "Cardiology");
Console.WriteLine(d);                            // Doctor { Name = Neha, Speciality = Cardiology }: real type printed

var d2 = d with { Name = "Neha S" };             // still a Doctor: with keeps the runtime type
Console.WriteLine(d2 is Doctor);                 // True

Person n = new Nurse("Neha", "ICU");
Console.WriteLine(d == n);                       // False: different runtime types, even if Name matches
```

Equality, `ToString` and `with` all respect the **runtime** type, which is why records use a hidden `EqualityContract` and virtual clone method.

### sealed ToString (C# 10)

If a base record overrides `ToString`, every derived record would normally regenerate its own and hide yours. Mark it `sealed` to force all derived records to use the base version:

```csharp
public abstract record Entity(Guid Id)
{
    public sealed override string ToString() => $"{GetType().Name}#{Id.ToString()[..8]}";
}
public record Invoice(Guid Id, decimal Total) : Entity(Id);

Console.WriteLine(new Invoice(Guid.NewGuid(), 500m));   // Invoice#3f2a9c1e: base format used
```

### Record structs (C# 10)

```csharp
public record struct Point(int X, int Y);             // value type, properties are { get; set; } (mutable!)
public readonly record struct Money(decimal Amount, string Currency);   // value type, immutable

var p = new Point(1, 2);
p.X = 5;                                              // allowed: record struct is mutable by default
var m = new Money(100, "INR");
// m.Amount = 5;                                      // ERROR: readonly record struct
var m2 = m with { Amount = 200 };
Console.WriteLine(m == new Money(100, "INR"));        // True: value equality
```

`record` alone (or `record class`, C# 10) is a reference type.

|  | `record` / `record class` | `record struct` | `readonly record struct` | Plain `class` |
| --- | --- | --- | --- | --- |
| Kind | Reference | Value | Value | Reference |
| Positional properties | `init` (immutable) | `get; set;` (mutable) | `init` (immutable) | n/a |
| `==` compares | Values | Values | Values | References |
| `with` | Yes | Yes | Yes | No |
| Inheritance | Records only | No | No | Yes |
| Null allowed | Yes | No | No | Yes |
| Use for | DTOs, domain values, API models | Small mutable value data | Small immutable values (Money, Point) | Entities with identity and behaviour |

**Record vs class:** use a record when two objects with the same data should be equal and the data shouldn't change; use a class when identity matters (two patients named Rohit are different people) or the object has rich mutable behaviour.

## 6. Cheat Sheet and Interview Questions

```csharp
// Patterns
if (o is int n) { }    if (x is not null) { }    if (c is { Status: "Pending", Amount: > 5000 }) { }
var label = age switch { < 18 => "Minor", >= 60 => "Senior", _ => "Adult" };
var r = (a, b) switch { (true, true) => 1, (true, _) => 2, _ => 0 };
if (claim is { Patient.Address.City: "Pune" }) { }
switch (shape) { case Circle c when c.Radius > 10: break; }

// Immutability
public class Money { public decimal Amount { get; init; } }
public readonly struct Point { public int X { get; } }

// Records
public record Patient(string Name, int Age);
var p2 = p1 with { Age = 31 };  var (name, age) = p1;  bool same = p1 == p2;
public record Doctor(string Name, string Dept) : Person(Name);
public readonly record struct Pin(string Code);
```

**Checklist**

- [ ] Rewrite an is-cast-if chain using a type pattern
- [ ] Write a switch statement with when guards and a switch expression
- [ ] Use relational, logical, property, extended property, tuple and positional patterns
- [ ] Explain why immutability helps; build an immutable class with init
- [ ] Show that default/arrays skip a struct's parameterless constructor
- [ ] Create a positional record and use equality, with, Deconstruct and ToString
- [ ] Add validation and a second constructor to a record
- [ ] Build a record hierarchy and explain sealed ToString
- [ ] Compare record, record struct and readonly record struct

**Interview questions (short answers)**

1. **What is pattern matching?** Testing a value's type/shape/contents and extracting data in one expression.
2. **Switch statement vs switch expression?** The expression returns a value, uses `=>` arms and `_`, and must be exhaustive.
3. **What is a property pattern?** `{ Prop: pattern }`, matching on property values (implies not null).
4. **What is an extended property pattern?** `{ A.B.C: value }`, dotted access to nested properties (C# 10).
5. **What are relational and logical patterns?** `< > <= >=` comparisons combined with `and`, `or`, `not` (C# 9).
6. **What does a positional pattern need?** A `Deconstruct` method (records have one).
7. **Why prefer immutable objects?** Thread safety, predictable state, safe sharing and hashing.
8. **What is an init-only property?** Settable only during construction/object initialization (C# 9).
9. **What is a record?** A reference type with value-based equality, immutability by default, `with`, `ToString` and `Deconstruct` generated.
10. **Record vs class equality?** Records compare values; classes compare references by default.
11. **What does `with` do?** Creates a copy with some properties changed (non-destructive mutation).
12. **Are records fully immutable?** Shallowly: referenced collections/objects can still change.
13. **Can records inherit?** Yes, from other records only; equality also checks the runtime type.
14. **What is sealed ToString in records?** Stops derived records regenerating ToString, so the base format is used (C# 10).
15. **record struct vs readonly record struct?** Both value types; record struct properties are mutable, readonly record struct is immutable.
