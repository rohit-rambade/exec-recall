# Constructors, Properties & Indexers

How objects get set up (instance and static constructors, overloading, initializers) and how they expose data safely (properties and indexers).

## 1. Instance Constructors

An instance constructor runs once for every object, at the moment `new` creates it. Its job is to put the new object into a valid starting state.

```csharp
public class Patient
{
    public string Name;
    public int Age;
    public DateTime RegisteredOn;

    // Instance constructor: same name as the class, no return type
    public Patient(string name, int age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name is required");
        Name = name;
        Age = age;
        RegisteredOn = DateTime.Now;   // values the caller shouldn't have to supply
    }
}

var p = new Patient("Rohit", 30);   // constructor runs here
// var q = new Patient();           // ERROR: no parameterless constructor exists now
```

**Rules**

- Name must match the class exactly; **no return type**, not even `void`.
- Runs automatically; you can't call it like a normal method (`p.Patient()` is invalid).
- Can be `public`, `private`, `protected` or `internal`, and can be overloaded.
- If a class declares **no** constructor, C# adds a public parameterless one that sets fields to defaults. As soon as you write any constructor, that free one disappears.
- Can't be `virtual`, `abstract`, `override` or inherited. A derived class calls the base one with `: base(...)`.

**Expression-bodied constructor** (one-liner):

```csharp
public Point(int x, int y) => (X, Y) = (x, y);
```

**Primary constructor (C# 12)**: parameters on the class itself, usable anywhere in the class body.

```csharp
public class Invoice(string number, decimal amount)
{
    public string Number { get; } = number;
    public decimal Total => amount * 1.18m;   // parameter captured
}
```

## 2. Static Constructor

A static constructor initialises a class's **static** data. The runtime calls it automatically, exactly **once**, just before the class is first used (first object created or first static member accessed).

```csharp
public class AppConfig
{
    public static readonly string ConnectionString;
    public static readonly DateTime LoadedAt;
    public static int Instances;

    static AppConfig()                       // no access modifier, no parameters
    {
        ConnectionString = File.ReadAllText("db.txt");
        LoadedAt = DateTime.Now;
        Console.WriteLine("Static constructor ran");
    }

    public AppConfig() { Instances++; }      // instance constructor
}

var a = new AppConfig();   // prints "Static constructor ran", then creates a
var b = new AppConfig();   // static constructor does NOT run again
```

&#91;embedded content: What runs, in order, when an object is created\]

Steps 1 and 2 happen only before the class's first use; steps 3 to 6 repeat for every object. (In a derived class, its own field initializers run before the base constructor.)

**Rules**

- Declared as `static ClassName()`: **no access modifier, no parameters**, so it can't be overloaded. Only one per class.
- Can't be called directly; you don't control exactly when it runs, only that it runs before first use.
- Can access only static members.
- If it throws, the type becomes unusable for the rest of the program (`TypeInitializationException`).
- Typical uses: load config once, set up a cache or logger, initialise `static readonly` values that need code.

|  | Instance constructor | Static constructor |
| --- | --- | --- |
| Runs | Every `new` | Once per class |
| Called by | `new` | The runtime, automatically |
| Parameters | Allowed | Not allowed |
| Access modifier | Any | None allowed |
| Overloading | Yes | No |
| Can touch | Static and instance members | Static members only |

## 3. Types of Constructor, Overloading and Object Initializer

C# has five commonly named kinds of constructor. They are not different keywords, just different shapes of the same feature.

| Type | What it looks like | Purpose |
| --- | --- | --- |
| Default | Parameterless; supplied by C# if you write none | Create with default values |
| Parameterized | Takes parameters | Create with caller-supplied values |
| Copy | Takes an object of the same class | Clone another object's values |
| Static | `static ClassName()` | Initialise static data once |
| Private | `private ClassName()` | Block outside `new` (singletons, static-only classes) |

```csharp
public class Policy
{
    public string Number;
    public decimal Premium;

    public Policy() { Number = "NEW"; }                       // default (user-written)
    public Policy(string number, decimal premium)             // parameterized
    {
        Number = number; Premium = premium;
    }
    public Policy(Policy other)                               // copy
    {
        Number = other.Number; Premium = other.Premium;
    }
}

// Private constructor: Singleton pattern
public sealed class Logger
{
    private static readonly Logger _instance = new Logger();
    private Logger() { }                                      // nobody else can call new
    public static Logger Instance => _instance;
}
var log = Logger.Instance;    // the one and only object
```

### Constructor Overloading

Several constructors with different parameter lists. Use `: this(...)` so they share one main constructor instead of repeating code.

```csharp
public class Claim
{
    public string Id { get; }
    public decimal Amount { get; }
    public string Status { get; }

    public Claim(string id, decimal amount, string status)    // main constructor
    {
        Id = id; Amount = amount; Status = status;
    }
    public Claim(string id, decimal amount) : this(id, amount, "Pending") { }
    public Claim(string id) : this(id, 0) { }
}

var c1 = new Claim("C1");                   // amount 0, Pending
var c2 = new Claim("C2", 5000);             // Pending
var c3 = new Claim("C3", 5000, "Approved");
```

The chained constructor (`this(...)`) runs **first**, then the calling constructor's own body.

### Object Initializer

Sets accessible fields/properties right after a constructor runs, in one expression, without needing a constructor for every combination.

```csharp
public class Address { public string City { get; set; } public string Pin { get; set; } }
public class Member
{
    public string Name { get; set; }
    public int Age { get; set; }
    public Address Home { get; set; }
    public List<string> Tags { get; } = new();
}

var m = new Member
{
    Name = "Priya",
    Age = 28,
    Home = new Address { City = "Pune", Pin = "411001" },  // nested initializer
    Tags = { "gold", "family" }                             // collection initializer adds items
};

var p = new Policy("P9", 1200m) { Premium = 1500m };   // constructor + initializer together
```

- Runs **after** the constructor, so it can override values the constructor set.
- Works only with accessible members that have a `set` or `init` accessor.
- `()` is optional when calling the parameterless constructor: `new Member { ... }`.

## 3b. Constructor Chaining and Private Constructors (in depth)

**Constructor chaining** means one constructor calls another before running its own body: `: this(...)` for a constructor in the same class, `: base(...)` for the parent's. It keeps all initialisation logic in **one** main constructor.

### Chaining with this(...)

```csharp
public class Appointment
{
    public string Patient { get; }
    public DateTime At { get; }
    public string Doctor { get; }
    public int DurationMin { get; }

    // Main ("master") constructor: all validation lives here
    public Appointment(string patient, DateTime at, string doctor, int durationMin)
    {
        if (string.IsNullOrWhiteSpace(patient)) throw new ArgumentException("Patient required");
        if (durationMin <= 0) throw new ArgumentOutOfRangeException(nameof(durationMin));
        Patient = patient; At = at; Doctor = doctor; DurationMin = durationMin;
        Console.WriteLine("main ctor");
    }

    public Appointment(string patient, DateTime at, string doctor)
        : this(patient, at, doctor, 15)                  // default duration
    {
        Console.WriteLine("3-arg ctor");
    }

    public Appointment(string patient, DateTime at)
        : this(patient, at, "Duty Doctor")              // chains to the 3-arg one, which chains on
    {
        Console.WriteLine("2-arg ctor");
    }
}

var a = new Appointment("Rohit", DateTime.Today.AddHours(10));
// main ctor
// 3-arg ctor
// 2-arg ctor     <- called constructor's body runs LAST
```

### Chaining with base(...) across inheritance

```csharp
public class Person
{
    public string Name { get; }
    public Person(string name) { Name = name; Console.WriteLine("Person"); }
}
public class Doctor : Person
{
    public string Speciality { get; }
    public Doctor(string name, string speciality) : base(name)   // parent first
    {
        Speciality = speciality; Console.WriteLine("Doctor");
    }
    public Doctor(string name) : this(name, "General") { }        // this + base together
}

new Doctor("Neha");   // Person, Doctor
```

**Order when creating an object:** field initializers → chained `base(...)` constructor (up to `object`) → chained `this(...)` constructors, innermost first → the called constructor's body → object initializer `{ ... }`.

**Rules**

- Only one initializer per constructor: either `: this(...)` or `: base(...)`, not both (but the target can chain further).
- No cycles: A → B → A is compile error CS0768.
- Arguments can be expressions but can't use instance members (`this` isn't ready yet); static methods are fine: `: this(Normalize(name))`.
- Without an explicit initializer, every constructor implicitly calls `base()`; if the parent has no parameterless constructor, you must write `: base(...)`.
- **Chaining vs optional parameters:** `public Appointment(string p, DateTime at, string doctor = "Duty Doctor", int duration = 15)` can replace several overloads; chaining is better when each overload needs different logic, or for public library APIs (defaults are baked into callers).

### Private constructors

A `private` constructor can be called **only from inside the class**, so outside code can't use `new`. Use it to control how and how many objects get created.

**1. Singleton: exactly one instance**

```csharp
public sealed class AppSettings
{
    private static readonly Lazy<AppSettings> _instance = new(() => new AppSettings());
    public static AppSettings Instance => _instance.Value;   // thread-safe, created on first use

    public string ConnectionString { get; }
    private AppSettings() => ConnectionString = File.ReadAllText("db.txt");   // private
}

var s1 = AppSettings.Instance;
var s2 = AppSettings.Instance;
Console.WriteLine(ReferenceEquals(s1, s2));   // True
// var s3 = new AppSettings();                // ERROR CS0122: inaccessible
```

**2. Static factory methods: named, validated creation**

```csharp
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    private Money(decimal amount, string currency) => (Amount, Currency) = (amount, currency);

    public static Money Rupees(decimal amount) => new(amount, "INR");
    public static Money FromPaise(long paise) => new(paise / 100m, "INR");
    public static bool TryParse(string s, out Money? m)
    {
        m = decimal.TryParse(s, out var d) && d >= 0 ? new Money(d, "INR") : null;
        return m is not null;
    }
}
var fee = Money.Rupees(500);   // clearer than new Money(500, "INR")
```

**3. Utility classes that should never be instantiated** (before `static class` existed, or when you need a non-static class): `private Helpers() { }`.

**4. Restricting inheritance:** a class whose constructors are all private can't be inherited from outside (a derived class must call a base constructor), though nested classes inside it still can.

**5. Mixing access levels:** a private "master" constructor plus public ones that chain to it with `: this(...)`.

|  | Private constructor | Static class | Sealed class |
| --- | --- | --- | --- |
| Can create objects | Only from inside the class | Never | Yes |
| Instance members allowed | Yes | No | Yes |
| Can be inherited | Not from outside | No | No |
| Typical use | Singleton, factories | Pure helpers (`Math`) | Prevent inheritance only |

**Interview quick answers:** *Can a class have only private constructors?* Yes: then only its own members (or nested types) can create instances. *Does a private constructor stop static members being used?* No. *Is a private constructor the same as a static constructor?* No: a static constructor initialises static data once and has no access modifier.

## 4. Important Points about Constructors

1. A constructor has the class's name and **no return type**.
2. The free default constructor exists only while you write no constructor yourself.
3. Constructors can be overloaded; chain them with `: this(...)` to avoid duplicate code.
4. A derived class's constructor calls the base constructor first, with `: base(...)` or implicitly the parameterless one. If the base has no parameterless constructor, you must call `base(...)` explicitly.
5. Constructors are not inherited and can't be `virtual`, `abstract` or `override`.
6. A static constructor runs once, takes no parameters or access modifier, and can touch only static members.
7. A private constructor stops outside creation; a class with only private constructors can't be inherited from outside it.
8. Object initializers run after the constructor.
9. `readonly` fields can be assigned only in their declaration or in a constructor.
10. Keep constructors light: validate and assign. Avoid long-running work such as network calls.
11. Structs always have a parameterless constructor; since C# 10 you can write your own.

## 5. Creating Properties; Read-only and Write-only Properties

A property is a member that looks like a field to callers (`obj.Age = 5`) but runs a `get` accessor when read and a `set` accessor when written. It's the standard way to encapsulate a private field.

```csharp
public class Member
{
    private string _name;            // backing field
    private int _age;

    public string Name
    {
        get { return _name; }
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name required");
            _name = value.Trim();     // 'value' holds what the caller assigned
        }
    }

    public int Age
    {
        get => _age;                  // expression-bodied accessors
        set => _age = value < 0 ? 0 : value;
    }
}

var m = new Member();
m.Name = "  Rohit ";     // calls set: stored as "Rohit"
Console.WriteLine(m.Name); // calls get
m.Age = -4;              // stored as 0
```

**What a property can do that a public field can't:** validate input, compute a value on the fly, raise a change notification, log access, be read-only or write-only, be part of an interface, and support data binding.

### Read-only and Write-only Properties

| Kind | Accessors | Example use |
| --- | --- | --- |
| Read-write | `get` + `set` | Name, Address |
| Read-only | `get` only | Id, CreatedOn, computed totals |
| Write-only | `set` only (rare) | Password, PIN: store, never read back |

```csharp
public class UserAccount
{
    private readonly string _id = Guid.NewGuid().ToString();
    private string _passwordHash;
    private decimal _price = 100, _qty = 3;

    public string Id { get { return _id; } }          // read-only
    public decimal Total => _price * _qty;            // read-only, computed each time

    public string Password                            // write-only
    {
        set { _passwordHash = Hash(value); }
    }

    private static string Hash(string s) => Convert.ToBase64String(
        System.Security.Cryptography.SHA256.HashData(System.Text.Encoding.UTF8.GetBytes(s)));
}

var u = new UserAccount();
Console.WriteLine(u.Id);      // OK
// u.Id = "X";                // ERROR: no set accessor
u.Password = "secret";        // OK
// Console.WriteLine(u.Password);  // ERROR: no get accessor
```

Write-only properties are unusual and many style guides prefer a method (`SetPassword(...)`), but know them for interviews.

## 6. Auto-Properties, Accessor Accessibility and Initializers

An auto-implemented property (C# 3) lets the compiler create the hidden backing field for you when the accessors need no extra logic.

```csharp
public class Employee
{
    public string Name { get; set; }        // compiler creates a private field
}
```

### Accessor accessibility

Give one accessor a more restrictive modifier than the property itself, typically a public `get` with a private `set`:

```csharp
public class Order
{
    public int Id { get; private set; }              // read anywhere, write only inside Order
    public string Status { get; protected set; }     // derived classes can also write
    public decimal Total { get; internal set; }      // same assembly can write

    public Order(int id) { Id = id; Status = "New"; }
    public void Ship() => Status = "Shipped";        // allowed inside the class
}

var o = new Order(7);
// o.Status = "Lost";    // ERROR: setter is protected
```

**Rules:** only one accessor may carry a modifier; it must be **more restrictive** than the property; both accessors must exist to use one.

### `init` accessor (C# 9)

```csharp
public class Patient
{
    public string Uhid { get; init; }       // set in constructor or object initializer, then frozen
    public required string Name { get; init; }   // C# 11: caller MUST set it
}

var p = new Patient { Uhid = "U100", Name = "Amit" };
// p.Uhid = "U200";    // ERROR: init-only
```

### Auto-property initializers (C# 6)

Give an auto-property a starting value right where it is declared:

```csharp
public class Settings
{
    public string Theme { get; set; } = "Light";
    public int PageSize { get; set; } = 20;
    public DateTime CreatedOn { get; } = DateTime.Now;      // read-only auto-property with initializer
    public List<string> Roles { get; } = new List<string>(); // avoids null collections
}
```

- Initializers run **before** the constructor body, so the constructor can still change them.
- A get-only auto-property (`{ get; }`) can be assigned only by its initializer or in a constructor.

| Syntax | Read | Write |
| --- | --- | --- |
| `{ get; set; }` | Anyone | Anyone |
| `{ get; private set; }` | Anyone | Inside the class |
| `{ get; protected set; }` | Anyone | Class + derived classes |
| `{ get; }` | Anyone | Initializer or constructor |
| `{ get; init; }` | Anyone | Constructor or object initializer |
| `=> expr` | Anyone | Never (computed) |

## 7. Important Points about Properties

1. A property is really a pair of methods (`get_Name()` and `set_Name(value)`) that the compiler generates; callers use field syntax.
2. Inside `set`, the keyword `value` holds the assigned value and has the property's type.
3. Keep fields `private` and expose them through properties; public fields are discouraged.
4. Auto-properties have a hidden backing field you can't name. Switch to a full property when you need logic. In C# 14 the `field` keyword lets you add logic without declaring one: `set => field = value.Trim();`
5. Accessor modifiers can only make one accessor more restrictive, never more open.
6. Properties can be `static`, `virtual`, `abstract`, `override`, and can be declared in interfaces.
7. A property can't be passed as a `ref` or `out` argument.
8. Getters should be fast and have no side effects. Use a method for slow or state-changing work (`GetReport()`, not `Report`).
9. Use `init` + `required` for objects that should be fully set at creation and immutable after.
10. Property names are PascalCase nouns; backing fields are `_camelCase`.

## 8. Creating Indexers and Indexer Overloading

An indexer lets callers use **array syntax on your object**: `obj[0]` or `obj["key"]`. It works like a property with parameters, declared with the keyword `this`.

```csharp
public class Ward
{
    private readonly string[] _beds = new string[10];

    public string this[int bedNo]                 // indexer
    {
        get
        {
            if (bedNo < 0 || bedNo >= _beds.Length)
                throw new IndexOutOfRangeException("No such bed");
            return _beds[bedNo] ?? "Empty";
        }
        set
        {
            if (bedNo < 0 || bedNo >= _beds.Length)
                throw new IndexOutOfRangeException("No such bed");
            _beds[bedNo] = value;
        }
    }

    public int Count => _beds.Length;
}

var ward = new Ward();
ward[3] = "Rohit";               // calls set
Console.WriteLine(ward[3]);       // Rohit (calls get)
Console.WriteLine(ward[4]);       // Empty
```

### Indexer Overloading

A class can have several indexers as long as their **parameter lists differ** (type or count). The return type alone doesn't count.

```csharp
public class EmployeeDirectory
{
    private readonly List<(int Id, string Name, string Dept)> _staff = new()
    {
        (101, "Rohit", "IT"), (102, "Priya", "HR"), (103, "Amit", "IT")
    };

    public string this[int id]                    // by id
        => _staff.FirstOrDefault(e => e.Id == id).Name ?? "Not found";

    public int this[string name]                  // by name -> id
        => _staff.FirstOrDefault(e => e.Name == name).Id;

    public string this[string dept, int position] // two parameters
        => _staff.Where(e => e.Dept == dept).ElementAtOrDefault(position).Name ?? "None";
}

var dir = new EmployeeDirectory();
Console.WriteLine(dir[102]);         // Priya
Console.WriteLine(dir["Amit"]);      // 103
Console.WriteLine(dir["IT", 1]);     // Amit
```

|  | Property | Indexer |
| --- | --- | --- |
| Accessed by | Name: `obj.Name` | Index: `obj[i]` |
| Declared with | Its own name | `this[...]` |
| Parameters | None | One or more (any type) |
| Static allowed? | Yes | No: always instance |
| Overloading | No (names must be unique) | Yes, by parameter list |

**Points to remember:** indexers can be read-only (`get` only), can't be `static`, can't use `ref`/`out` parameters, and are what make `List<T>[i]`, `Dictionary<K,V>[key]` and `string[i]` work.

## 9. Section Cheat Sheet, Checklist and Interview Questions

```csharp
public class Product
{
    // STATIC
    public static int Count;
    static Product() { Count = 0; }                       // static constructor: once

    // FIELDS & PROPERTIES
    private decimal _price;
    private readonly string[] _tags = new string[5];
    public int Id { get; private set; }                   // public get, private set
    public string Name { get; set; } = "Unnamed";         // auto + initializer
    public string Sku { get; init; }                      // init-only
    public decimal Price                                  // full property
    {
        get => _price;
        set => _price = value >= 0 ? value : throw new ArgumentException("Negative price");
    }
    public decimal PriceWithTax => Price * 1.18m;         // read-only computed

    // CONSTRUCTORS
    public Product() : this(0, "Unnamed") { }             // chained default
    public Product(int id, string name) { Id = id; Name = name; Count++; }
    public Product(Product other) : this(other.Id, other.Name) { Price = other.Price; }  // copy

    // INDEXERS
    public string this[int i] { get => _tags[i]; set => _tags[i] = value; }
    public bool this[string tag] => Array.IndexOf(_tags, tag) >= 0;   // overload
}

var p = new Product(1, "Pen") { Price = 20, Sku = "PN-01" };   // ctor + initializer
p[0] = "stationery";
bool has = p["stationery"];   // true
```

**Checklist**

- [ ] Write an instance constructor with validation
- [ ] Explain when a static constructor runs and its restrictions
- [ ] Name the five constructor types with an example of each
- [ ] Overload constructors and chain them with `this(...)`
- [ ] Use an object initializer with nested and collection initializers
- [ ] Write full, read-only, write-only and computed properties
- [ ] Use `private set`, `init` and `required`
- [ ] Write an indexer and overload it
- [ ] Recite the six-step creation order

**Interview questions (short answers)**

1. **What is a constructor?** A special method with the class name and no return type that initialises a new object.
2. **When does C# provide a default constructor?** Only when the class declares no constructors.
3. **What is a static constructor?** A parameterless, modifier-less constructor that initialises static data once, before the class is first used.
4. **Can a static constructor be overloaded or called directly?** No to both.
5. **What is a private constructor used for?** Preventing outside instantiation: singletons, factory methods, static-like classes.
6. **What is constructor chaining?** Calling another constructor of the same class with `: this(...)`, or the base class with `: base(...)`.
7. **Is there a copy constructor in C#?** Not built in; you write one that takes an object of the same type.
8. **Object initializer vs constructor?** The constructor runs first and enforces required setup; the initializer then sets accessible properties.
9. **Property vs field?** A field stores data; a property controls access to it through `get`/`set`.
10. **What is an auto-implemented property?** A property whose backing field is generated by the compiler.
11. **How do you make a property read-only?** Omit `set`, use `private set`, use `init`, or make it expression-bodied.
12. **What is `init`?** An accessor that allows setting only during object creation.
13. **What is an indexer?** A member declared with `this[...]` that lets an object be accessed like an array.
14. **Can indexers be overloaded or static?** Overloaded yes (different parameters); static no.
15. **Indexer vs property?** A property is accessed by name with no parameters; an indexer by index with parameters.
