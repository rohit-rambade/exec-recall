---
title: OOP, Classes & Objects
---

# Section 4: OOP, Classes and Objects

Why object-oriented programming exists, how to build classes and objects in C#, and what really happens in memory when you write `new`.

## 1. Understanding OOP

Object-Oriented Programming organises code around **objects**, things that hold their own data (state) and the actions that work on it (behaviour), instead of loose functions passing data around.

**Real-world mapping:** a *Car* has data (colour, speed, fuel) and behaviour (Start, Accelerate, Brake). In code, the `Car` class describes that, and each actual car is an object.

&#91;embedded content: The four pillars of OOP\]

Every pillar is built from classes and objects, which is why this section starts there; later sections cover each pillar in depth.

**Procedural vs object-oriented**

|  | Procedural (C, early BASIC) | Object-oriented (C#, Java) |
| --- | --- | --- |
| Built around | Functions and steps | Objects that combine data + behaviour |
| Data | Often global, shared by any function | Hidden inside objects (encapsulated) |
| Reuse | Copy or call functions | Inheritance and composition |
| Change | One change can ripple everywhere | Changes stay inside one class |
| Scales to | Small programs | Large, long-lived systems |

**The four pillars in one line each**

- **Encapsulation:** keep fields `private` and expose safe access through methods and properties.
- **Abstraction:** show only the essentials (`car.Start()`), hide the details (fuel injection, ignition).
- **Inheritance:** a child class reuses and extends a parent (`ElectricCar : Car`). "Is-a" relationship.
- **Polymorphism:** the same call behaves differently by type: method overloading (compile time) and overriding (run time).

**Benefits:** reusability, easier maintenance, security of data, modelling of real-world problems, and teamwork (each class is a unit someone can own).

## 2. Creating Classes & Objects

Define a class once with `class Name { ... }`, then create as many objects as you need with `new Name()`. Each object gets its own copy of the instance fields.

**What can go inside a class**

| Member | Purpose | Example |
| --- | --- | --- |
| Field | Stores data | `private double _balance;` |
| Property | Safe get/set access to data | `public string Owner { get; set; }` |
| Constructor | Sets up a new object | `public Account(string owner) { }` |
| Method | Behaviour | `public void Deposit(double amt) { }` |
| Constant / static member | Shared by the whole class | `public static int Count;` |
| Event, indexer, nested type | Covered in later sections | — |

**Full example: a bank account**

```csharp
public class BankAccount
{
    // Fields (private: encapsulated)
    private double _balance;

    // Static field: shared by all objects
    public static int TotalAccounts;

    // Properties
    public string AccountNumber { get; }          // read-only after construction
    public string Owner { get; set; }
    public double Balance => _balance;            // read-only, computed

    // Constructor
    public BankAccount(string accountNumber, string owner, double opening = 0)
    {
        AccountNumber = accountNumber;
        Owner = owner;
        _balance = opening;
        TotalAccounts++;
    }

    // Methods
    public void Deposit(double amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }

    public bool Withdraw(double amount)
    {
        if (amount > _balance) return false;
        _balance -= amount;
        return true;
    }
}
```

**Creating and using objects**

```csharp
BankAccount acc1 = new BankAccount("A101", "Rohit", 5000);
var acc2 = new BankAccount("A102", "Priya");      // opening = 0 (optional parameter)
BankAccount acc3 = new("A103", "Amit");          // target-typed new (C# 9)

acc1.Deposit(1500);
acc1.Withdraw(2000);
Console.WriteLine(acc1.Balance);                 // 4500
Console.WriteLine(BankAccount.TotalAccounts);    // 3 (accessed via the class)
// acc1._balance = 1000000;   // ERROR: private
```

### Constructors

- Same name as the class, **no return type**, runs automatically on `new`.
- If you write none, C# gives a free parameterless **default constructor**. Write any constructor and that free one disappears.
- Overload constructors and chain them with `this(...)`:

```csharp
public class Student
{
    public string Name;
    public int Age;

    public Student() : this("Unknown", 18) { }        // calls the other constructor
    public Student(string name, int age)
    {
        this.Name = name;    // 'this' = the current object; resolves name clash
        this.Age = age;
    }
}
```

### Object initialiser (C# 3)

Set public properties right after construction, without a matching constructor:

```csharp
var s = new Student { Name = "Neha", Age = 21 };
```

### Properties instead of public fields

```csharp
private int _age;
public int Age                     // full property with validation
{
    get { return _age; }
    set { if (value >= 0) _age = value; }   // 'value' = what was assigned
}

public string City { get; set; }           // auto-property
public string Code { get; init; }          // settable only at creation (C# 9)
public required string Email { get; set; } // must be set in initialiser (C# 11)
```

## 3. Important Points to Remember about Objects

A class variable does not hold the object; it holds a **reference** to an object on the heap. Most object "surprises" come from that one fact.

&#91;embedded content: Object references: stack vs heap\]

Changing the object through `b` also changes what `a` sees, because there is only one object.

```csharp
Car a = new Car("Swift");
Car b = a;            // copies the reference, NOT the object
b.Model = "Baleno";
Console.WriteLine(a.Model);   // Baleno
```

**The points to remember**

1. **A class is a blueprint and takes no memory for data; an object does.** Memory is allocated on the heap only when `new` runs.
2. **`new` does three things:** allocates heap memory, sets fields to defaults (`0`, `false`, `null`), then runs the constructor and returns the reference.
3. **Declaring is not creating.** `Car c;` makes a variable only. Using an uninitialised local is a compile error; a field defaults to `null`.
4. **Calling a member on `null` throws `NullReferenceException`.** Guard with `if (c != null)` or `c?.Start()`.
5. **Every object is independent.** Two `new` calls give two objects with separate field values, even with identical data.
6. **`==` on classes compares references** by default, not contents. Two separate objects with equal data are not `==` unless the class overrides equality (records do this for you).
7. **Passing an object to a method passes a copy of the reference.** The method can change the object's fields, and the caller sees the change.
8. **Static members belong to the class, not to objects.** Access them as `ClassName.Member`; there is one copy no matter how many objects exist.
9. **The garbage collector frees objects automatically** once no reference points to them. Setting `a = null;` only drops that one reference.
10. **Every class ultimately inherits from `System.Object`,** so every object has `ToString()`, `Equals()`, `GetHashCode()` and `GetType()`.

```csharp
var p1 = new Point { X = 1 };
var p2 = new Point { X = 1 };
Console.WriteLine(p1 == p2);          // False: two different objects
Console.WriteLine(p1.Equals(p2));     // False unless Equals is overridden
Console.WriteLine(p1.GetType().Name); // Point
```

|  | Class (reference type) | Struct (value type) |
| --- | --- | --- |
| Variable holds | A reference to the object | The data itself |
| `b = a` | Both point to one object | Independent copy |
| Can be `null` | Yes | No (unless `Nullable<T>`) |
| Inheritance | Yes | No (interfaces only) |
| Use for | Most entities with identity and behaviour | Small, immutable values (Point, Money) |

## 4. Section Cheat Sheet

The syntax for this whole section in one block.

```csharp
// CLASS SKELETON
public class Employee
{
    private decimal _salary;                       // field (private)
    public static int Count;                       // static: one per class
    public const string Company = "HealthIndia";   // constant

    public int Id { get; }                         // read-only auto-property
    public string Name { get; set; }               // auto-property
    public decimal Salary                          // full property
    {
        get => _salary;
        set => _salary = value < 0 ? 0 : value;
    }

    public Employee() : this(0, "New") { }         // chained constructor
    public Employee(int id, string name)           // parameterised constructor
    {
        Id = id; Name = name; Count++;
    }

    public decimal AnnualPay() => Salary * 12;     // expression-bodied method
    public override string ToString() => $"{Id}: {Name}";
}

// OBJECTS
Employee e1 = new Employee(1, "Rohit");    // constructor
var e2 = new Employee { Name = "Priya" };  // initialiser (Id = 0)
Employee e3 = new(3, "Amit");              // target-typed new
Employee e4 = e1;                          // same object as e1
e1?.AnnualPay();                           // null-safe call
int total = Employee.Count;                // static via class name
```

**Key facts**

| Topic | Remember |
| --- | --- |
| Class vs object | Blueprint vs instance created by `new` |
| Default access | Members `private`, top-level classes `internal` |
| Default constructor | Free only if you write no constructor |
| `this` | Current object; also chains constructors |
| `static` | Belongs to the class; no `this` inside |
| Assignment `b = a` | Copies the reference |
| `==` on classes | Compares references, not data |
| Unused objects | Freed by the garbage collector |
| Base of every class | `System.Object` |

## 5. Section 4 Interview Questions and Checklist

Tick each item once you can explain it and write the code without notes.

- [ ] Explain OOP and how it differs from procedural programming
- [ ] Name and define the four pillars, with one example each
- [ ] Write a class with fields, properties, constructors and methods
- [ ] Create objects three ways: constructor, initialiser, target-typed `new`
- [ ] Chain constructors with `this(...)` and explain the default constructor rule
- [ ] Draw the stack and heap for `b = a`
- [ ] Explain `NullReferenceException` and how to avoid it
- [ ] Static vs instance members
- [ ] Class vs struct

**Interview questions (short answers)**

1. **What is OOP?** A style that models software as objects combining data and behaviour, built on encapsulation, abstraction, inheritance and polymorphism.
2. **Class vs object?** A class is the blueprint (a type); an object is a runtime instance of it, created with `new` and stored on the heap.
3. **What is a constructor?** A special method with the class's name and no return type that initialises a new object.
4. **Can a constructor be private?** Yes. It stops outside code from using `new`; used in singletons and static-factory classes.
5. **When is the default constructor provided?** Only when the class declares no constructors at all.
6. **What is `this`?** A reference to the current object; used to resolve name clashes and chain constructors.
7. **Field vs property?** A field stores data; a property controls access with `get`/`set` and can validate.
8. **Static vs instance members?** Static belongs to the class (one copy); instance belongs to each object.
9. **Where are objects stored?** On the managed heap; the variable (on the stack, for locals) holds a reference.
10. **What happens with `Car b = a;`?** Both variables reference the same object; a change through one is seen through the other.
11. **How is memory freed?** The garbage collector reclaims objects that are no longer reachable.
12. **Does `==` compare object contents?** Not by default for classes; it compares references unless overridden (records compare values).
13. **Can you create an object of an abstract class or a static class?** No to both.
14. **What is an object initialiser?** `new T { Prop = value }`, which sets accessible properties right after construction.
15. **Class vs struct?** Class: reference type, heap, inheritance, can be null. Struct: value type, copied on assignment, no inheritance.
