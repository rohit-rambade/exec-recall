# Abstraction & Interfaces

How to define what an object can do without fixing how it does it: abstract classes, abstract methods and interfaces, and how they enable polymorphism.

## 1. Abstraction

Abstraction means showing **what** an object does and hiding **how** it does it. Callers work with a simple, stable contract; the messy details can change behind it.

**Real-world example:** you drive a car with a steering wheel, pedals and a gear lever. You don't need to know how fuel injection or the gearbox works, and the same controls work on a petrol, diesel or electric car.

**In C# you get abstraction from**

- **Abstract classes:** a partial blueprint that can hold shared code plus abstract (unimplemented) members.
- **Interfaces:** a pure contract listing what a type can do.
- **Access modifiers:** `private` hides details (this overlaps with encapsulation).

|  | Abstraction | Encapsulation |
| --- | --- | --- |
| Focus | Design level: what to expose | Implementation level: how to protect data |
| Tool | Abstract classes, interfaces | `private` fields, properties |
| Question it answers | What can this object do? | Who may touch this data? |
| Example | `IPayment.Pay(amount)` | `private decimal _balance` |

## 2. Abstract Classes and Abstract Methods

An **abstract class** is an incomplete base class: you can't create objects of it, only of classes that inherit it. An **abstract method** has a signature but no body; every non-abstract child **must** override it.

```csharp
public abstract class Employee                  // abstract class
{
    public string Name { get; }
    protected Employee(string name) => Name = name;   // constructors are allowed

    public abstract decimal MonthlyPay();         // abstract method: no body, ends with ;

    public virtual string Role => "Employee";     // virtual: child MAY override

    public string Payslip() =>                    // normal method: shared by all children
        $"{Name} ({Role}): {MonthlyPay():N0}";
}

public class Permanent : Employee
{
    public decimal Salary { get; init; }
    public Permanent(string name) : base(name) { }
    public override decimal MonthlyPay() => Salary;     // MUST implement
}

public class Contractor : Employee
{
    public int Hours { get; init; }
    public decimal Rate { get; init; }
    public Contractor(string name) : base(name) { }
    public override decimal MonthlyPay() => Hours * Rate;
    public override string Role => "Contractor";
}

// var e = new Employee("X");        // ERROR CS0144: cannot create an abstract class
Employee[] staff =
{
    new Permanent("Rohit") { Salary = 60000 },
    new Contractor("Priya") { Hours = 120, Rate = 500 }
};
foreach (var e in staff) Console.WriteLine(e.Payslip());
// Rohit (Employee): 60,000
// Priya (Contractor): 60,000
```

**Rules for abstract classes**

- Declared with `abstract`; can't be instantiated or `sealed`.
- Can contain fields, constructors, properties, normal, virtual and abstract methods.
- A class with even one abstract member must itself be abstract.
- A child that doesn't implement all abstract members must also be abstract.

**Rules for abstract methods**

- No body: `public abstract void Draw();`
- Allowed only inside an abstract class.
- Implicitly virtual: implemented with `override` in the child.
- Can't be `private`, `static` or `sealed`; properties and indexers can be abstract too.

|  | `abstract` method | `virtual` method |
| --- | --- | --- |
| Has a body | No | Yes |
| Child must override | Yes | Optional |
| Allowed in | Abstract classes only | Any non-sealed class |

## 3. Practice: Gaming Characters using Inheritance

A classic exercise that combines an abstract base class, shared state, abstract and virtual methods, and polymorphism. Try writing it yourself first.

**Task:** every character has a name, health and attack power. Warriors hit hard, mages cast spells that cost mana, archers sometimes land a critical hit. A battle loop treats them all as `Character`.

```csharp
public abstract class Character
{
    public string Name { get; }
    public int Health { get; protected set; }
    protected int Power;

    protected Character(string name, int health, int power)
    {
        Name = name; Health = health; Power = power;
    }

    public bool IsAlive => Health > 0;

    public abstract int Attack();                       // each type attacks differently

    public virtual void TakeDamage(int amount)          // shared, can be customised
    {
        Health = Math.Max(0, Health - amount);
        Console.WriteLine($"{Name} takes {amount} damage, health {Health}");
    }
}

public class Warrior : Character
{
    public Warrior(string name) : base(name, 150, 20) { }
    public override int Attack() => Power;
    public override void TakeDamage(int amount) => base.TakeDamage(amount / 2);   // armour halves damage
}

public class Mage : Character
{
    private int _mana = 30;
    public Mage(string name) : base(name, 80, 35) { }
    public override int Attack()
    {
        if (_mana < 10) return 5;          // out of mana: weak staff hit
        _mana -= 10;
        return Power;
    }
}

public class Archer : Character
{
    private static readonly Random Rng = new();
    public Archer(string name) : base(name, 100, 18) { }
    public override int Attack() => Rng.Next(100) < 25 ? Power * 2 : Power;   // 25% critical
}

// Battle loop: written once, works for every character type
var party = new List<Character> { new Warrior("Arjun"), new Mage("Meera"), new Archer("Kabir") };
var boss = new Warrior("Boss");

while (boss.IsAlive)
{
    foreach (var hero in party.Where(h => h.IsAlive))
    {
        boss.TakeDamage(hero.Attack());    // polymorphism: the right Attack() runs
        if (!boss.IsAlive) break;
    }
}
Console.WriteLine("Boss defeated!");
```

**What this shows:** shared code lives once in `Character`; each subclass supplies only what differs; adding a `Healer` needs no change to the battle loop. **Extend it:** add a `Healer` with a `Heal(Character target)` method, or an `ISpellCaster` interface for characters with mana.

## 4. Creating Interfaces

An interface is a **contract**: it lists members (methods, properties, events, indexers) that any implementing class or struct promises to provide. It says nothing about how.

```csharp
public interface INotifier                    // names start with I (convention)
{
    string Channel { get; }                   // property
    bool Send(string to, string message);     // method: no body, no access modifier
}

public class EmailNotifier : INotifier
{
    public string Channel => "Email";
    public bool Send(string to, string message)        // must be public
    {
        Console.WriteLine($"Email to {to}: {message}");
        return true;
    }
}

public class SmsNotifier : INotifier
{
    public string Channel => "SMS";
    public bool Send(string to, string message)
    {
        if (message.Length > 160) return false;
        Console.WriteLine($"SMS to {to}: {message}");
        return true;
    }
}

INotifier n = new SmsNotifier();      // variable of interface type
n.Send("98xxxxxx10", "Claim approved");
// var x = new INotifier();           // ERROR: can't create an interface
```

**Rules**

- Declared with `interface`; members are `public` by default.
- No instance fields and no constructors.
- The implementing class must provide **every** member, as `public`, with matching signatures (otherwise error CS0535).
- A class can implement many interfaces; a struct can implement interfaces too.

**Since C# 8, interfaces can also have:** default method bodies, static members, and private helper methods. Use defaults to add a member later without breaking existing implementers:

```csharp
public interface ILogger
{
    void Log(string msg);
    void LogError(string msg) => Log("ERROR: " + msg);   // default implementation
}
```

## 5. Dynamic Polymorphism with Interfaces

Dynamic (run-time) polymorphism means one call written against an interface runs different code depending on the **actual object** behind it. The caller never needs `if (type == ...)` checks, and new types plug in without changing the caller.

&#91;embedded content: Dynamic polymorphism through an interface\]

The caller depends only on `IPayment`. Adding a `NetBankingPayment` later is one new class; `Checkout` doesn't change.

```csharp
public interface IPayment
{
    bool Pay(decimal amount);
}

public class UpiPayment : IPayment
{
    public bool Pay(decimal amount) { Console.WriteLine($"UPI: {amount}"); return true; }
}
public class CardPayment : IPayment
{
    public bool Pay(decimal amount) { Console.WriteLine($"Card: {amount}"); return true; }
}
public class WalletPayment : IPayment
{
    public decimal Balance { get; set; } = 500;
    public bool Pay(decimal amount)
    {
        if (amount > Balance) return false;
        Balance -= amount; Console.WriteLine($"Wallet: {amount}"); return true;
    }
}

void Checkout(IPayment p, decimal amount)        // written once
{
    Console.WriteLine(p.Pay(amount) ? "Paid" : "Failed");
}

Checkout(new UpiPayment(), 1200);
Checkout(new WalletPayment(), 1200);             // Failed

List<IPayment> methods = new() { new UpiPayment(), new CardPayment() };
foreach (var m in methods) m.Pay(100);           // each runs its own Pay
```

**Why it matters:** this is the basis of **loose coupling** and **dependency injection**. In ASP.NET Core you register `IPayment` → `UpiPayment` once, and every class asks only for `IPayment`. It also makes unit testing easy: pass a fake `IPayment`.

|  | Compile-time (static) polymorphism | Run-time (dynamic) polymorphism |
| --- | --- | --- |
| How | Method / operator overloading | Overriding virtual/abstract methods, implementing interfaces |
| Decided | By the compiler from argument types | By the runtime from the object's real type |
| Example | `Add(int, int)` vs `Add(double, double)` | `IPayment p = new CardPayment(); p.Pay(...)` |

## 6. Multiple Inheritance with Interfaces and Interface Inheritance

A class can have only one base class but can implement **any number of interfaces**. That's how C# gives you the benefits of multiple inheritance without the diamond problem.

```csharp
public interface IPrintable { void Print(); }
public interface IScannable { string Scan(); }
public interface IFaxable  { void Fax(string number); }

public class Device { public string Model { get; set; } }

// one base class first, then any number of interfaces
public class AllInOnePrinter : Device, IPrintable, IScannable, IFaxable
{
    public void Print() => Console.WriteLine("Printing...");
    public string Scan() => "scanned.pdf";
    public void Fax(string number) => Console.WriteLine($"Faxing {number}");
}

var mfp = new AllInOnePrinter { Model = "HP 4103" };
IPrintable p = mfp;   p.Print();           // seen only as a printer
IScannable s = mfp;   s.Scan();            // seen only as a scanner
if (mfp is IFaxable f) f.Fax("0222");      // check a capability at run time
```

**Design idea (Interface Segregation):** many small, focused interfaces (`IPrintable`, `IScannable`) beat one huge `IMachine`, because a simple printer shouldn't have to implement `Fax`.

### Interface Inheritance

An interface can inherit from one or more other interfaces. A class that implements the child interface must implement **all** members of the whole chain.

```csharp
public interface IReadable  { string Read(int id); }
public interface IWritable  { void Write(int id, string data); }
public interface IRepository : IReadable, IWritable      // inherits both
{
    void Delete(int id);
}

public class ClaimRepository : IRepository
{
    private readonly Dictionary<int, string> _db = new();
    public string Read(int id) => _db.GetValueOrDefault(id, "none");   // from IReadable
    public void Write(int id, string data) => _db[id] = data;        // from IWritable
    public void Delete(int id) => _db.Remove(id);                     // from IRepository
}

IReadable reader = new ClaimRepository();   // usable through any interface in the chain
```

|  | Class inheritance | Interface inheritance |
| --- | --- | --- |
| Parents allowed | One class | Many interfaces |
| Inherits | Implementation (code + data) | Only the contract (signatures) |
| Syntax | `class B : A` | `interface IB : IA1, IA2` |

## 7. Explicit Interface Implementation

Explicit implementation writes a member as `InterfaceName.Member` with **no access modifier**. It's then callable only through a variable of that interface type, not through the class. Use it to resolve name clashes or to hide interface-only members.

### Problem: two interfaces with the same member

```csharp
public interface IEnglishReport { string Title(); }
public interface IHindiReport   { string Title(); }

public class ClaimReport : IEnglishReport, IHindiReport
{
    string IEnglishReport.Title() => "Claim Summary";     // explicit: no 'public'
    string IHindiReport.Title()   => "Daava Saaraansh";

    public string Title() => "Default title";              // optional normal method
}

var r = new ClaimReport();
Console.WriteLine(r.Title());                      // Default title
Console.WriteLine(((IEnglishReport)r).Title());    // Claim Summary
IHindiReport h = r;
Console.WriteLine(h.Title());                      // Daava Saaraansh
```

### Hiding a member from the class's public surface

```csharp
public class ReadOnlyList : IList<int>
{
    // ...
    void ICollection<int>.Add(int item) => throw new NotSupportedException();   // not visible on ReadOnlyList
    // ...
}
```

|  | Implicit implementation | Explicit implementation |
| --- | --- | --- |
| Syntax | `public void Print()` | `void IPrintable.Print()` |
| Access modifier | Must be `public` | None allowed |
| Callable via class variable | Yes | **No** |
| Callable via interface variable | Yes | Yes |
| Use when | Normal case | Name clashes, or member should be hidden |

**Rules:** can't be `virtual`, `abstract` or have access modifiers; implements exactly one interface's member; a class can mix implicit and explicit implementations.

## 8. Abstract Class vs Interface, Cheat Sheet, Checklist and Interview Questions

**Abstract class vs interface (the most-asked comparison)**

|  | Abstract class | Interface |
| --- | --- | --- |
| Keyword | `abstract class` | `interface` |
| Instantiate? | No | No |
| How many per class | One (single inheritance) | Many |
| Fields | Yes | No instance fields |
| Constructors | Yes | No |
| Method bodies | Yes (normal + virtual), plus abstract | Only default methods (C# 8+) |
| Access modifiers on members | Any | Public by default |
| Relationship | is-a (shared identity and code) | can-do (capability) |
| Use when | Related classes share state and code | Unrelated classes share a capability |
| Example | `abstract class Employee` | `IComparable`, `IDisposable`, `IPayment` |

**Rule of thumb:** start with an interface for the contract; add an abstract base class when several implementations need the same code.

**Cheat sheet**

```csharp
// ABSTRACT CLASS
abstract class Shape
{
    public string Name { get; }
    protected Shape(string n) => Name = n;
    public abstract double Area();                  // must override
    public virtual string Info() => $"{Name}: {Area():F2}";   // may override
}
class Circle : Shape
{
    double r; public Circle(double r) : base("Circle") => this.r = r;
    public override double Area() => Math.PI * r * r;
}

// INTERFACES
interface IDrawable { void Draw(); }
interface IResizable { void Resize(double f); }
interface IShape2D : IDrawable, IResizable { }      // interface inheritance

class Square : Shape, IShape2D                      // one class + interfaces
{
    double s; public Square(double s) : base("Square") => this.s = s;
    public override double Area() => s * s;
    public void Draw() => Console.WriteLine("[]");  // implicit
    void IResizable.Resize(double f) => s *= f;     // explicit
}

// POLYMORPHISM
List<Shape> shapes = new() { new Circle(1), new Square(2) };
foreach (var sh in shapes) Console.WriteLine(sh.Info());
IResizable rs = new Square(3); rs.Resize(2);        // explicit member via interface
```

**Checklist**

- [ ] Define abstraction and contrast it with encapsulation
- [ ] Write an abstract class with an abstract method, a virtual method and a constructor
- [ ] Build the gaming characters program from memory
- [ ] Create an interface and two implementations
- [ ] Write a method that takes an interface and works with any implementation
- [ ] Implement three interfaces in one class
- [ ] Make an interface inherit from two others
- [ ] Resolve a name clash with explicit implementation
- [ ] Recite abstract class vs interface

**Interview questions (short answers)**

1. **What is abstraction?** Exposing what an object does while hiding how; done with abstract classes and interfaces.
2. **Can you instantiate an abstract class?** No; only its non-abstract subclasses.
3. **Can an abstract class have a constructor?** Yes; it runs when a subclass object is created.
4. **Can an abstract class have non-abstract methods?** Yes, plus fields and properties.
5. **What happens if a child doesn't implement an abstract method?** Compile error, unless the child is also abstract.
6. **Abstract vs virtual method?** Abstract has no body and must be overridden; virtual has a body and may be overridden.
7. **What is an interface?** A contract of members that implementing types must provide.
8. **Can interfaces have fields or constructors?** No. Since C# 8 they can have default methods and static members.
9. **How does C# achieve multiple inheritance?** By implementing multiple interfaces.
10. **What is explicit interface implementation?** Implementing a member as `IName.Member` so it's callable only through that interface; used for name clashes.
11. **Can an interface inherit another interface?** Yes, one or many.
12. **Abstract class vs interface?** See the table: one vs many, state and constructors vs none, is-a vs can-do.
13. **What is dynamic polymorphism?** The method that runs is chosen at run time by the object's actual type (overriding / interface calls).
14. **Why program to an interface?** Loose coupling, easy swapping of implementations, testability and dependency injection.
15. **Can a struct implement an interface?** Yes (but it can't inherit from a class).
