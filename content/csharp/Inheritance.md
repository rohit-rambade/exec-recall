# Inheritance

How one class reuses and extends another: inheritance syntax and types, `base`, constructor chaining, hiding vs overriding, and `sealed`.

## 1. Creating Inheritance

Inheritance lets a new class (child / derived) reuse the members of an existing class (parent / base) and add or change behaviour. Write it with a colon: `class Child : Parent`. It models an **is-a** relationship: a Car *is a* Vehicle.

```csharp
public class Vehicle                       // base (parent) class
{
    public string Brand { get; set; }
    public int Speed { get; protected set; }

    public void Accelerate(int amount) => Speed += amount;
    public string Describe() => $"{Brand} at {Speed} km/h";
}

public class Car : Vehicle                 // derived (child) class
{
    public int Doors { get; set; }         // new member
    public void Honk() => Console.WriteLine("Beep!");
}

var car = new Car { Brand = "Tata", Doors = 4 };
car.Accelerate(60);                        // inherited method
Console.WriteLine(car.Describe());         // Tata at 60 km/h
car.Honk();                                // its own method

Vehicle v = car;                           // a Car can be treated as a Vehicle
// v.Honk();                               // ERROR: Vehicle doesn't know Honk
```

**What the child gets**

| Parent member | Visible in child? |
| --- | --- |
| `public` | Yes, and to everyone |
| `protected` | Yes (child only, not outside code) |
| `internal` | Yes, if in the same assembly |
| `private` | No: inherited in memory but not accessible by name |
| Constructors | No: never inherited (but called, see section 3) |
| Static members | Accessible through the class, not truly per-object |

**Why use it:** code reuse, a common type for related classes (`List<Vehicle>` can hold cars and bikes), and the foundation for polymorphism.

**Every class inherits from `System.Object`**, directly or indirectly, so every object has `ToString()`, `Equals()`, `GetHashCode()` and `GetType()`.

**Prefer composition when it's has-a:** a Car *has an* Engine, so give Car an `Engine` property rather than inheriting from Engine.

## 2. Types of Inheritance

C# supports single, multilevel and hierarchical inheritance between classes. Multiple inheritance (two parent classes) is **not** allowed for classes; you get the same effect by implementing several interfaces.

&#91;embedded content: Types of inheritance in C#\]

Hybrid inheritance (a mix of the shapes) is possible only when every class still has one parent class, with any extra "parents" being interfaces.

| Type | Shape | Example | Allowed for classes? |
| --- | --- | --- | --- |
| Single | One child, one parent | `Car : Vehicle` | Yes |
| Multilevel | Chain of parents | `SportsCar : Car`, `Car : Vehicle` | Yes |
| Hierarchical | Many children, one parent | `Car : Vehicle`, `Bike : Vehicle` | Yes |
| Multiple | One child, many parents | `Smartphone : Phone, Camera` | **No**: use interfaces |
| Hybrid | Mix of the above | Classes + interfaces | Only via interfaces |

```csharp
// Multilevel
class Animal { public void Eat() => Console.WriteLine("Eating"); }
class Dog : Animal { public void Bark() => Console.WriteLine("Woof"); }
class Puppy : Dog { public void Weep() => Console.WriteLine("Whimper"); }
new Puppy().Eat();   // inherited through two levels

// Multiple via interfaces
interface ICamera { void TakePhoto(); }
interface IPhone  { void Call(string number); }
class Smartphone : ICamera, IPhone
{
    public void TakePhoto() => Console.WriteLine("Click");
    public void Call(string number) => Console.WriteLine($"Calling {number}");
}

// One class + many interfaces: the class must come first
class Tablet : Device, ICamera, IPhone { /* ... */ }
```

**Why no multiple class inheritance?** The **diamond problem**: if B and C both inherit from A and override the same method, and D inherits from both, which version does D get? C# avoids the ambiguity by allowing only one base class.

## 3. The base Keyword and the Parent Class's Constructor

`base` refers to the parent-class part of the current object. Use it to call a parent's constructor, or to reach a parent member that the child has hidden or overridden.

### Calling a parent member

```csharp
class Employee
{
    public virtual decimal Pay() => 30000;
}

class Manager : Employee
{
    public override decimal Pay() => base.Pay() + 20000;   // reuse parent logic, then add
}

Console.WriteLine(new Manager().Pay());   // 50000
```

### Parent class's constructor

Creating a child object always runs the **parent constructor first**, then the child's. If you don't say which parent constructor, C# calls the parent's parameterless one.

```csharp
class Person
{
    public string Name { get; }
    public Person(string name)            // no parameterless constructor here
    {
        Name = name;
        Console.WriteLine("1. Person constructor");
    }
}

class Student : Person
{
    public int RollNo { get; }
    public Student(string name, int rollNo) : base(name)   // pass data up to the parent
    {
        RollNo = rollNo;
        Console.WriteLine("2. Student constructor");
    }
}

var s = new Student("Neha", 12);
// 1. Person constructor
// 2. Student constructor
```

**Rule to remember:** if the parent has **no parameterless constructor**, every child constructor must call `: base(...)` explicitly; otherwise you get error CS7036.

In a multilevel chain (`Puppy : Dog : Animal`), constructors run from the top down: Animal, then Dog, then Puppy.

|  | `this` | `base` |
| --- | --- | --- |
| Refers to | The current object | The parent-class part of the current object |
| Constructor call | `: this(...)`: another constructor in the same class | `: base(...)`: a parent constructor |
| Member access | `this.Name` | `base.Pay()`: the parent's version |
| In static members | Not allowed | Not allowed |

## 4. Method Hiding, Method Overriding, and Hiding vs Overriding

Both let a child redefine a parent method with the same signature. The difference is which version runs when the object is accessed through a **parent-type variable**: overriding runs the child's version, hiding runs the parent's.

### Method hiding (`new`)

```csharp
class Parent
{
    public void Show() => Console.WriteLine("Parent.Show");
}
class Child : Parent
{
    public new void Show() => Console.WriteLine("Child.Show");   // 'new' hides the parent method
}
```

Without `new` it still compiles but gives warning CS0108; `new` says the hiding is intentional.

### Method overriding (`virtual` + `override`)

```csharp
class Shape
{
    public virtual double Area() => 0;          // parent allows overriding
}
class Circle : Shape
{
    public double R { get; set; }
    public override double Area() => Math.PI * R * R;   // child replaces it
}
```

### The difference in one example

```csharp
Parent p = new Child();
p.Show();                 // Parent.Show   <- hiding: decided by the VARIABLE type

Shape s = new Circle { R = 1 };
Console.WriteLine(s.Area());   // 3.14...  <- overriding: decided by the OBJECT type

((Child)p).Show();        // Child.Show: hidden method needs a child-type reference
```

|  | Method hiding | Method overriding |
| --- | --- | --- |
| Keywords | `new` in the child (parent needs nothing) | `virtual` / `abstract` in the parent, `override` in the child |
| Which method runs via a parent variable | Parent's | Child's |
| Decided at | Compile time (by variable type) | Run time (by actual object type) |
| Polymorphism? | No | Yes: run-time polymorphism |
| Parent version reachable | Yes, through a parent-type variable or `base` | Only via `base.Method()` inside the child |
| Typical use | Rare: you can't change the parent, or names clash | Common: specialise behaviour in subclasses |

**Overriding rules:** the signature and return type must match; the access level must match; you can't override a non-virtual, static or private method; `override` methods are themselves virtual, so a grandchild can override again unless `sealed`.

## 5. Sealed Classes and Sealed Methods

`sealed` stops further inheritance. A sealed class can't be a parent; a sealed method can't be overridden again further down the chain.

### Sealed class

```csharp
public sealed class PaymentGateway
{
    public void Charge(decimal amount) { /* ... */ }
}

// class FakeGateway : PaymentGateway { }   // ERROR CS0509: cannot derive from sealed type
```

**When to seal a class:** to stop others changing security-sensitive or carefully tuned behaviour, for utility or immutable types, and for a small speed gain (the runtime can call methods directly). `string` is sealed in .NET.

A sealed class **can** inherit from another class; it just can't be inherited from. A `static` class is automatically sealed.

### Sealed method

Only an `override` method can be sealed. It stops classes further down from overriding it again.

```csharp
class Account
{
    public virtual decimal Interest() => 0.03m;
}

class SavingsAccount : Account
{
    public sealed override decimal Interest() => 0.04m;   // final version
}

class SeniorSavings : SavingsAccount
{
    // public override decimal Interest() => 0.05m;     // ERROR: sealed in SavingsAccount
    public new decimal Interest() => 0.05m;             // hiding is still possible
}
```

|  | Sealed class | Sealed method |
| --- | --- | --- |
| Syntax | `sealed class X` | `sealed override void M()` |
| Stops | Any class from inheriting | Further overriding of that method |
| Applies to | Any class | Only methods that are `override` |
| Still possible | Inheriting *from* another class | Hiding with `new` |

## 6. Important Points, Cheat Sheet, Checklist and Interview Questions

**Points to remember**

1. Syntax is `class Child : Parent`; a class can have **one** base class but any number of interfaces (class listed first).
2. Inheritance means **is-a**. Use composition for **has-a**.
3. Private members are inherited but not accessible; use `protected` for child access.
4. Constructors are not inherited. The parent constructor runs first; use `: base(...)` to pick one, and you must if the parent lacks a parameterless constructor.
5. A parent-type variable can hold a child object (`Vehicle v = new Car();`), but sees only parent members.
6. Overriding needs `virtual`/`abstract` in the parent and `override` in the child; it is run-time polymorphism.
7. Hiding uses `new`, is resolved by variable type at compile time, and is usually a design smell.
8. `base.Method()` calls the parent's version from inside the child.
9. `sealed` classes can't be inherited; `sealed override` methods can't be overridden further.
10. Static, private and non-virtual methods can't be overridden.
11. All classes ultimately derive from `System.Object`; structs can't inherit from classes (only implement interfaces).

**Cheat sheet**

```csharp
class Animal
{
    protected string Name;
    public Animal(string name) => Name = name;
    public virtual string Speak() => "...";            // can be overridden
    public void Sleep() => Console.WriteLine("Zzz");   // non-virtual
}

class Dog : Animal
{
    public Dog(string name) : base(name) { }           // parent constructor
    public override string Speak() => "Woof";          // overriding
    public new void Sleep() => Console.WriteLine("Dog naps");  // hiding
    public string Both() => base.Speak() + " " + Speak();      // base access
}

sealed class Puppy : Dog                               // nobody can inherit Puppy
{
    public Puppy(string n) : base(n) { }
    public sealed override string Speak() => "Yip";    // final version
}

Animal a = new Dog("Rex");
a.Speak();   // "Woof"  (override: object type wins)
a.Sleep();   // "Zzz"   (hiding: variable type wins)
```

**Checklist**

- [ ] Write a parent and child class and use inherited members
- [ ] Draw the four inheritance shapes and say why multiple class inheritance is banned
- [ ] Call a parent constructor with `: base(...)` and predict the print order
- [ ] Use `base.Method()` inside an override
- [ ] Predict output for hiding vs overriding through a parent variable
- [ ] Seal a class and a method; explain what each blocks

**Interview questions (short answers)**

1. **What is inheritance?** A mechanism where a class reuses and extends another class's members (is-a relationship).
2. **Does C# support multiple inheritance?** Not for classes; a class can implement multiple interfaces.
3. **What is the diamond problem?** Ambiguity when two parents provide the same member; C# avoids it with single class inheritance.
4. **Are constructors inherited?** No. The parent constructor is called (first), but not inherited.
5. **What does `base` do?** Calls a parent constructor (`: base(...)`) or accesses the parent's member version (`base.M()`).
6. **In what order do constructors run?** Parent first, then child (top of the chain down).
7. **Method hiding vs overriding?** Hiding (`new`) picks the method by variable type at compile time; overriding (`virtual`/`override`) picks by object type at run time.
8. **Can you override a non-virtual method?** No; you can only hide it with `new`.
9. **What is a sealed class?** A class that can't be inherited, such as `string`.
10. **Can a sealed class inherit from another class?** Yes; it just can't be a base class.
11. **What is a sealed method?** An `override` method marked `sealed` so derived classes can't override it again.
12. **Can a static class be inherited?** No; static classes are implicitly sealed.
13. **Can a private member be accessed in a child?** No. Use `protected` instead.
14. **Why prefer composition over inheritance?** Looser coupling and more flexibility; inheritance ties the child to the parent's implementation.
