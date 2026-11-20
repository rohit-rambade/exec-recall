# Methods & Encapsulation

How to write methods, protect an object's data, and pass values in and out: every parameter modifier, overloading, local functions and recursion.

## 1. Understanding Methods

A method is a named block of code that does one job. Define it once and call it as often as you like; it can take inputs (parameters) and give back one output (return value).

```csharp
// access   modifier  return type  name     parameters
   public   static    int          Add     (int a, int b)
{
    return a + b;      // body; 'return' sends the value back and exits
}

int total = Add(3, 4);   // calling it: 3 and 4 are arguments
```

**Anatomy**

| Part | Meaning |
| --- | --- |
| Access modifier | Who can call it (`public`, `private`, …). Default is `private`. |
| `static` (optional) | Belongs to the class, not an object |
| Return type | Type of the value sent back; `void` means nothing |
| Name | PascalCase verb: `CalculateTax`, `GetUser` |
| Parameter list | Inputs, each with a type; may be empty `()` |
| Body | Statements in `{ }` |
| Signature | Name + parameter types and order (return type is **not** part of it) |

**Parameters vs arguments:** parameters are the variables in the definition (`int a`); arguments are the actual values passed at the call (`3`).

**Kinds of methods**

```csharp
void Greet() => Console.WriteLine("Hi");              // no input, no output
void GreetUser(string name) => Console.WriteLine($"Hi {name}");  // input only
int GetYear() => DateTime.Now.Year;                   // output only
double Area(double r) => Math.PI * r * r;             // input and output
```

`=>` makes an **expression-bodied** method: a one-line shortcut for `{ return ...; }`.

**Rules**

- A non-void method must return a value on every code path (error CS0161 otherwise).
- `return;` alone can exit a `void` method early.
- A method returns only one value; for more, use a tuple `(int min, int max)`, an object, or `out` parameters.
- Keep methods short and focused on one task: easier to test and reuse.

## 2. Encapsulation (Parts 1 and 2)

Encapsulation hides an object's data behind `private` fields and allows changes only through methods or properties that enforce the rules. Outside code can't put the object in an invalid state.

**The problem it solves**

```csharp
public class Account { public double Balance; }
var acc = new Account();
acc.Balance = -50000;      // nothing stops invalid data
```

**Part 1: private field + getter/setter methods (the classic way)**

```csharp
public class Account
{
    private double _balance;               // hidden

    public double GetBalance() => _balance; // read access

    public void Deposit(double amount)     // controlled write
    {
        if (amount <= 0)
            throw new ArgumentException("Deposit must be positive");
        _balance += amount;
    }
}
```

**Part 2: properties (the C# way)**

A property looks like a field to the caller but runs `get`/`set` code behind the scenes.

```csharp
public class Employee
{
    private int _age;

    public int Age
    {
        get { return _age; }
        set
        {
            if (value < 18 || value > 65)
                throw new ArgumentOutOfRangeException(nameof(value));
            _age = value;            // 'value' = the assigned value
        }
    }

    public string Name { get; set; }          // auto-property: compiler makes the field
    public string Id { get; private set; }    // anyone reads, only this class writes
    public decimal Bonus => Salary * 0.1m;    // read-only computed property
    public decimal Salary { get; init; }      // set only during creation
}

var e = new Employee { Name = "Rohit", Salary = 50000 };
e.Age = 30;      // runs the setter
// e.Age = 10;   // throws: validation works
```

| Property form | Read | Write |
| --- | --- | --- |
| `{ get; set; }` | Anyone | Anyone |
| `{ get; private set; }` | Anyone | Only inside the class |
| `{ get; }` | Anyone | Only in the constructor |
| `{ get; init; }` | Anyone | Constructor or object initialiser |
| `=> expression` | Anyone | Never (computed) |

**Benefits:** validation in one place, freedom to change the inner design later, read-only or write-only data, and easier debugging.

## 3. Local Variables, Parameters and the this Keyword

Local variables and parameters exist only while the method runs; fields live as long as the object. `this` lets you reach the object's own members when names collide.

|  | Field | Parameter | Local variable |
| --- | --- | --- | --- |
| Declared | In the class | In the method signature | Inside the method body |
| Lifetime | As long as the object | During the call | Until its block ends |
| Default value | Yes (`0`, `null`, `false`) | Set by the caller | None: must assign before use |
| Access modifiers | Yes | No | No |

```csharp
public class Counter
{
    private int count;                  // field

    public void Add(int step)           // parameter
    {
        int previous = count;           // local variable
        count += step;
        Console.WriteLine($"{previous} -> {count}");
    }   // 'step' and 'previous' are gone here; 'count' remains
}
```

### this keyword

`this` is a reference to the current object.

```csharp
public class Student
{
    private string name;
    private int age;

    public Student(string name, int age)
    {
        this.name = name;   // this.name = field, name = parameter
        this.age = age;
    }

    public Student(string name) : this(name, 18) { }   // 2. chain constructors

    public Student Rename(string n) { name = n; return this; }  // 3. return the object (fluent style)

    public void Register(School s) => s.Add(this);     // 4. pass the current object
}
```

**Uses of `this`:** (1) tell a field apart from a parameter with the same name, (2) call another constructor, (3) return or pass the current object, (4) declare extension methods (`this string s` as the first parameter) and indexers.

**Not allowed:** `this` inside a `static` method, because there is no current object there.

## 4. Static Methods

A static method belongs to the class, not to an object: call it as `ClassName.Method()` with no `new`. Use it for work that needs no object state, such as helpers and calculations.

```csharp
public class MathHelper
{
    public static double Pi = 3.14159;           // static field
    private int instanceValue = 5;

    public static double CircleArea(double r)    // static method
    {
        return Pi * r * r;
        // return instanceValue;   // ERROR: no object, so no instance members
    }

    public int Double() => instanceValue * 2 + (int)Pi;   // instance method CAN use static members
}

double a = MathHelper.CircleArea(5);   // no object needed
```

|  | Static method | Instance method |
| --- | --- | --- |
| Called on | The class: `Math.Max(a, b)` | An object: `car.Start()` |
| Needs `new`? | No | Yes |
| Can use instance fields | No (only via an object passed in) | Yes |
| Can use static fields | Yes | Yes |
| `this` available | No | Yes |
| Examples | `Console.WriteLine`, `int.Parse`, `Math.Sqrt` | `list.Add`, `str.ToUpper` |

**Static class:** `public static class Utils { }` holds only static members and cannot be instantiated or inherited. Good for helpers (like `System.Math`).

**`Main` is static** so the runtime can call it before any object exists.

## 5. Object References as Arguments

When you pass an object, C# copies the **reference**, not the object. The method can change the object's fields and the caller sees it, but reassigning the parameter to a new object does not affect the caller.

```csharp
public class Product { public string Name; public decimal Price; }

void ApplyDiscount(Product p)
{
    p.Price = p.Price * 0.9m;     // changes the SAME object: caller sees it
}

void Replace(Product p)
{
    p = new Product { Name = "New", Price = 1 };   // only the local copy now points elsewhere
}

var item = new Product { Name = "Pen", Price = 100 };
ApplyDiscount(item);
Console.WriteLine(item.Price);   // 90
Replace(item);
Console.WriteLine(item.Name);    // Pen (unchanged)
```

| Passing | What the method gets | Change a field / value inside | Reassign inside |
| --- | --- | --- | --- |
| Value type (`int`, `struct`) | A copy of the value | Caller does **not** see it | Caller does not see it |
| Reference type (class) | A copy of the reference | Caller **sees** it | Caller does not see it |
| Anything with `ref` | An alias of the caller's variable | Caller sees it | Caller **sees** it |

**`string` is a special case:** it is a reference type but immutable, so `s += "x"` inside a method creates a new string and the caller's string is unchanged.

## 6. Default and Named Arguments

**Default (optional) arguments** let callers skip parameters that have a sensible default. **Named arguments** let callers pass values by parameter name, in any order.

```csharp
void CreateOrder(string item, int qty = 1, bool express = false, string note = "")
{
    Console.WriteLine($"{item} x{qty}, express={express}, note='{note}'");
}

CreateOrder("Pen");                          // Pen x1, express=False
CreateOrder("Pen", 5);                       // qty = 5
CreateOrder("Pen", 5, true);                 // all positional
CreateOrder("Pen", express: true);           // skip qty using a name
CreateOrder(note: "Gift", item: "Book");     // any order when all named
CreateOrder("Pen", qty: 2, note: "Urgent");  // mix: positional first
```

**Rules for default arguments**

- Optional parameters must come **after** all required ones.
- The default must be a compile-time constant: a literal, `const`, `null`, `default`, or `new()` for a struct. Not `DateTime.Now`.
- The default value is baked into the **caller** at compile time. Changing it in a library means callers must recompile.

**Rules for named arguments**

- Names must match the parameter names exactly (case-sensitive).
- Positional arguments can follow named ones only if the named ones are in their correct positions (C# 7.2).
- Great for readability: `Resize(width: 800, height: 600)` beats `Resize(800, 600)`; and for skipping defaults in the middle.

## 7. Method Overloading

Overloading means several methods with the **same name but different parameter lists** in the same class. The compiler picks the right one from the arguments at compile time (compile-time polymorphism).

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;                    // 2 ints
    public int Add(int a, int b, int c) => a + b + c;         // different count
    public double Add(double a, double b) => a + b;           // different types
    public string Add(string a, int b) => a + b;              // different order/types
    public string Add(int b, string a) => b + a;

    // public double Add(int a, int b) => a + b;  // ERROR: differs only by return type
}

var c = new Calculator();
c.Add(2, 3);        // int version
c.Add(2, 3, 4);     // three-int version
c.Add(2.5, 3.1);    // double version
```

**What makes a valid overload:** different number of parameters, different types, different order of types, or a different modifier (`ref`/`out`/`in` vs none).

**What does NOT count:** return type, parameter names, `params` alone, or `ref` vs `out` vs `in` against each other.

**Watch for ambiguity:** with `void F(int a, double b)` and `void F(double a, int b)`, the call `F(1, 1)` matches both equally and fails to compile.

**Real examples:** `Console.WriteLine` has many overloads (for `int`, `string`, `bool`, …); so do constructors, which are overloaded the same way.

## 8. Parameter Modifiers: default, ref, out, in

Modifiers control whether a method gets a copy of the caller's variable or the variable itself, and which way data may flow.

| Modifier | Data flow | Must caller initialise? | Must method assign? | Method can change it? | Call syntax |
| --- | --- | --- | --- | --- | --- |
| none (default) | In (a copy) | Yes | No | Only its own copy | `F(x)` |
| `ref` | In and out | **Yes** | No | Yes, caller sees it | `F(ref x)` |
| `out` | Out only | No | **Yes, before returning** | Yes, caller sees it | `F(out x)` |
| `in` | In only, by reference | Yes | No | **No (read-only)** | `F(in x)` or `F(x)` |

### Default: pass by value

```csharp
void AddTen(int n) { n += 10; }
int a = 5;
AddTen(a);
Console.WriteLine(a);   // 5: the method changed its own copy
```

### ref: pass by reference (two-way)

```csharp
void AddTen(ref int n) { n += 10; }
int a = 5;
AddTen(ref a);           // 'ref' needed at the call too
Console.WriteLine(a);   // 15

void Swap(ref int x, ref int y) { (x, y) = (y, x); }   // classic use
int p = 1, q = 2;
Swap(ref p, ref q);      // p = 2, q = 1
```

### out: return extra values (one-way out)

```csharp
void Divide(int a, int b, out int quotient, out int remainder)
{
    quotient = a / b;    // MUST assign every out parameter
    remainder = a % b;
}

int q, r;                // no need to initialise
Divide(17, 5, out q, out r);   // q = 3, r = 2
```

Best-known example: `int.TryParse("123", out int n)` returns `bool` for success and gives the number through `out`.

### Out variable declaration (C# 7)

Declare the variable right in the call instead of on a separate line:

```csharp
Divide(17, 5, out int quotient, out var remainder);   // declared inline
if (int.TryParse(input, out int age)) Console.WriteLine(age);
Divide(17, 5, out int onlyQ, out _);                  // '_' discards a value you don't need
```

### in: read-only reference (C# 7.2)

```csharp
struct BigPoint { public double X, Y, Z; /* ... many fields ... */ }

double Length(in BigPoint p)
{
    // p.X = 5;     // ERROR: 'in' parameters are read-only
    return Math.Sqrt(p.X * p.X + p.Y * p.Y + p.Z * p.Z);
}
```

Use `in` for **large structs** to avoid copying them while guaranteeing the method won't modify them. For small types like `int` it brings no benefit.

**Restrictions:** `ref`, `out` and `in` cannot be used in `async` methods, iterators (`yield`), or with properties as arguments (`F(ref obj.Prop)` fails; use a local variable).

## 9. ref Returns

A `ref` return (C# 7) hands back a **reference to a storage location** (an array element or field) instead of a copy, so the caller can read or change the original directly.

```csharp
int[] scores = { 40, 75, 90 };

ref int FindScore(int[] arr, int target)
{
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] == target)
            return ref arr[i];      // return a reference to the element
    throw new InvalidOperationException("Not found");
}

ref int slot = ref FindScore(scores, 75);   // 'ref local' points at scores[1]
slot = 80;                                   // changes the array itself
Console.WriteLine(scores[1]);                // 80

int copy = FindScore(scores, 90);            // without 'ref' you just get a copy
copy = 0;                                    // array unchanged
```

**Rules**

- Mark it in three places: `ref` return type, `return ref ...`, and `ref` at the call + `ref` local.
- You can only return a reference to something that outlives the method: array elements, fields, or `ref` parameters. **Never a local variable** (it is destroyed when the method ends; compile error).
- `ref readonly` returns give read-only access without copying.
- Mostly used in performance-critical code (games, large struct arrays, `Span<T>`).

## 10. params

`params` lets a method accept **any number of arguments** of one type; the caller can pass them separated by commas or as an array.

```csharp
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers) total += n;
    return total;
}

Sum();                   // 0: empty array, not null
Sum(5);                  // 5
Sum(1, 2, 3, 4);         // 10
Sum(new[] { 7, 8 });     // 15: an array works too

void Log(string level, params string[] messages)
{
    foreach (var m in messages) Console.WriteLine($"[{level}] {m}");
}
Log("INFO", "Started", "Connected", "Ready");
```

**Rules**

- Only **one** `params` parameter per method, and it must be the **last**.
- Up to C# 12 it must be a single-dimensional array (`params int[]`). C# 13 allows any collection type, such as `params List<int>` or `params ReadOnlySpan<int>`.
- `params object[] args` accepts values of mixed types (this is how `string.Format` and `Console.WriteLine(format, args)` work).
- If an overload without `params` matches exactly, the compiler prefers it.

## 11. Local Functions and Static Local Functions

A local function (C# 7) is a method declared **inside another method**. Only that method can call it, which keeps a small helper close to where it is used.

```csharp
public double OrderTotal(double[] prices, double taxRate)
{
    double subtotal = 0;
    foreach (var p in prices) subtotal += p;
    return AddTax(subtotal);

    // local function: can be declared after the return
    double AddTax(double amount) => amount + amount * taxRate;   // uses outer 'taxRate'
}
```

**Features**

- Can use (capture) the outer method's local variables and parameters, like `taxRate` above.
- Can be called before its declaration in the method.
- Can be generic, recursive, `async`, and use `ref`/`out`/`params`.
- No access modifiers (they are always private to the method).

### Static local functions (C# 8)

Adding `static` forbids capturing outer variables, so the function can only use its own parameters. This avoids hidden dependencies and hidden memory allocations.

```csharp
public int Process(int x)
{
    int factor = 3;
    return Triple(x);

    static int Triple(int n) => n * 3;          // OK: uses only its parameter
    // static int Bad(int n) => n * factor;     // ERROR: cannot capture 'factor'
}
```

|  | Local function | Static local function | Lambda |
| --- | --- | --- | --- |
| Captures outer variables | Yes | **No** | Yes |
| Callable before declaration | Yes | Yes | No |
| Recursion | Easy | Easy | Awkward |
| Allocation when capturing | Usually none | None | Often a heap allocation |
| Use when | A private helper for one method | Pure helper with no side inputs | Passing behaviour to LINQ, events |

## 12. Recursion

A recursive method calls itself on a smaller version of the problem until it reaches a **base case** that answers directly. Every recursive method needs both parts.

```csharp
long Factorial(int n)
{
    if (n <= 1) return 1;            // base case: stops the recursion
    return n * Factorial(n - 1);     // recursive case: smaller problem
}
```

&#91;embedded content: Recursion: the call stack for Factorial(4)\]

Each call waits on the stack until the one below it returns; then the results multiply on the way back up.

**More classic examples**

```csharp
int Fibonacci(int n) => n <= 1 ? n : Fibonacci(n - 1) + Fibonacci(n - 2);   // 0 1 1 2 3 5 8

int SumDigits(int n) => n == 0 ? 0 : n % 10 + SumDigits(n / 10);          // 1234 -> 10

int Power(int b, int e) => e == 0 ? 1 : b * Power(b, e - 1);              // 2^5 = 32

string Reverse(string s) => s.Length <= 1 ? s : Reverse(s[1..]) + s[0];   // "abc" -> "cba"
```

|  | Recursion | Loop (iteration) |
| --- | --- | --- |
| Code | Short, mirrors the maths | Often longer |
| Memory | One stack frame per call | Constant |
| Risk | `StackOverflowException` if too deep or no base case | Infinite loop |
| Best for | Trees, folders, divide and conquer, backtracking | Simple counting and repetition |

**Watch out:** naive `Fibonacci` recomputes the same values exponentially; for large n use a loop or cache results (memoisation). A missing or unreachable base case crashes the program with a stack overflow, which cannot be caught.

## 13. Important Points about Methods + Cheat Sheet

**Points to remember**

1. Every method lives inside a type (class, struct, record or interface); top-level statements are compiled into a hidden `Main`.
2. The **signature** is name + parameter types/order/modifiers. Return type and parameter names are not part of it.
3. Methods are `private` by default; make public only what callers need (encapsulation).
4. Arguments are passed **by value** unless you write `ref`, `out` or `in`. For objects, the value passed is the reference.
5. `out` parameters must be assigned inside the method; `ref` arguments must be initialised before the call.
6. Optional parameters go last; `params` goes very last; only one `params` per method.
7. Overloads differ by parameters, never by return type alone.
8. Static methods cannot use instance members or `this`.
9. Local functions are private to their method; `static` ones cannot capture outer variables.
10. Recursion needs a reachable base case, or the program crashes with `StackOverflowException`.
11. A non-void method must return a value on every path.
12. Keep methods small, named with a verb, and doing one thing.

**Cheat sheet**

```csharp
// DEFINE
public int Add(int a, int b) { return a + b; }
public int Add(int a, int b, int c) => a + b + c;          // overload, expression-bodied
public static double Area(double r) => Math.PI * r * r;    // static

// ENCAPSULATION
private int _age;
public int Age { get => _age; set => _age = value >= 0 ? value : 0; }
public string Name { get; private set; }

// ARGUMENTS
void Order(string item, int qty = 1, bool express = false) { }
Order("Pen");  Order("Pen", express: true);                // default + named

// MODIFIERS
void Inc(ref int x) => x++;            Inc(ref n);
void Get(out int x) => x = 42;         Get(out int v);  Get(out _);
void Show(in BigStruct s) { }          Show(in big);
int Sum(params int[] xs) => xs.Sum();  Sum(1, 2, 3);       // Sum() needs using System.Linq
ref int Find(int[] arr) => ref arr[0]; ref int slot = ref Find(arr);

// LOCAL FUNCTIONS
int Outer(int x) { return Square(x); static int Square(int n) => n * n; }

// RECURSION
long Fact(int n) => n <= 1 ? 1 : n * Fact(n - 1);
```

## 14. Interview Questions and Checklist

Tick each item when you can explain it and write the code without notes.

- [ ] Write methods of all four shapes (no input/output, input only, output only, both)
- [ ] Encapsulate a class with private fields, validation and properties
- [ ] Explain field vs parameter vs local variable lifetime
- [ ] Four uses of `this`
- [ ] Static vs instance methods, and what a static class is
- [ ] Predict what happens when an object is passed and modified or reassigned
- [ ] Use default and named arguments together
- [ ] Write three valid overloads and one invalid one
- [ ] Compare by-value, `ref`, `out`, `in` from memory (the table in section 8)
- [ ] Use an out variable declaration and a discard
- [ ] Write a ref return and explain why a local can't be returned by ref
- [ ] Write a `params` method
- [ ] Write a local function and a static local function
- [ ] Write factorial, Fibonacci and sum of digits recursively

**Interview questions (short answers)**

1. **What is a method signature?** Name plus parameter types, order and modifiers; not the return type.
2. **What is encapsulation?** Hiding data with `private` and exposing controlled access via methods/properties.
3. **Field vs property?** A property wraps a field with `get`/`set` logic; callers use it like a field.
4. **`ref` vs `out`?** `ref` must be initialised by the caller and is two-way; `out` need not be initialised but must be assigned by the method.
5. **What is `in`?** Pass by reference but read-only; avoids copying large structs.
6. **Can you overload by return type?** No.
7. **Overloading vs overriding?** Overloading: same name, different parameters, same class, compile time. Overriding: same signature in a derived class with `virtual`/`override`, run time.
8. **Why can't a static method use `this`?** There is no current object in a static context.
9. **Are objects passed by reference in C#?** No: the reference is passed by value. The method can change the object but not the caller's variable, unless `ref` is used.
10. **What is `params`?** A modifier for the last parameter that accepts a variable number of arguments.
11. **Rules for optional parameters?** After required ones; default must be a compile-time constant.
12. **What are named arguments for?** Clarity and skipping optional parameters, in any order.
13. **Local function vs lambda?** Local functions can be called before declaration, recurse easily and often avoid allocations; lambdas are values you can pass around.
14. **What is a static local function?** A local function that cannot capture outer variables.
15. **What is recursion and its risk?** A method calling itself with a base case; risk is stack overflow when too deep or with no base case.
16. **What is a ref return?** Returning a reference to an existing storage location so the caller can modify it in place.
