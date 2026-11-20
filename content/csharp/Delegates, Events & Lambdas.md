# Delegates, Events & Lambdas

Treating methods as values: delegates, multicast, events, anonymous methods and lambdas, the built-in `Func`/`Action`/`Predicate`, expression trees, and the expression-style syntax (`=>` members and switch expressions).

## 1. Creating Delegates and Multicast Delegates

A delegate is a **type-safe reference to a method**. Declare the signature once, then store any matching method in a delegate variable and call it later, pass it as an argument, or swap it at run time.

```csharp
// 1. Declare a delegate type (the signature)
public delegate decimal DiscountRule(decimal amount);

// 2. Methods with a matching signature
static decimal NoDiscount(decimal a)     => a;
static decimal Festive(decimal a)        => a * 0.9m;
static decimal SeniorCitizen(decimal a)  => a * 0.8m;

// 3. Create and invoke
DiscountRule rule = Festive;              // method group conversion (same as new DiscountRule(Festive))
decimal price = rule(1000);               // 900: same as rule.Invoke(1000)

rule = SeniorCitizen;                     // swap behaviour at run time
Console.WriteLine(rule(1000));            // 800

// 4. Pass behaviour into a method
static decimal Checkout(decimal amt, DiscountRule r) => r(amt);
Checkout(1000, NoDiscount);               // 1000
```

**Key facts:** delegates are reference types derived from `System.MulticastDelegate`; they can point to static or instance methods; the signature (parameters + return type) must match.

### Multicast delegates

One delegate can hold **several methods**. Add with `+=`, remove with `-=`; invoking it calls them all, in the order added.

```csharp
public delegate void Notify(string msg);

static void ToEmail(string m) => Console.WriteLine($"Email: {m}");
static void ToSms(string m)   => Console.WriteLine($"SMS: {m}");
static void ToLog(string m)   => Console.WriteLine($"Log: {m}");

Notify notify = ToEmail;
notify += ToSms;
notify += ToLog;
notify("Claim approved");    // Email, SMS, Log: all three run

notify -= ToSms;
notify("Claim paid");        // Email, Log

foreach (Delegate d in notify.GetInvocationList())
    Console.WriteLine(d.Method.Name);   // ToEmail, ToLog
```

**Multicast rules:** if the delegate returns a value, you get only the **last** method's result; if one method throws, the rest don't run (loop over `GetInvocationList()` to handle each); invoking a null delegate throws, so use `notify?.Invoke(...)`.

## 2. Events, Auto-Implemented Events and EventHandler

An **event** is a delegate wrapped so that outside code can only **subscribe** (`+=`) or **unsubscribe** (`-=`); only the declaring class can **raise** it. It implements the publisher/subscriber (observer) pattern.

&#91;embedded content: Events: publisher and subscribers\]

The publisher raises once; every attached handler runs. Adding a new subscriber needs no change to `ClaimService`.

### Auto-implemented (field-like) event with EventHandler

```csharp
// 1. Data to send (optional): derive from EventArgs
public class ClaimEventArgs : EventArgs
{
    public string ClaimId { get; }
    public decimal Amount { get; }
    public ClaimEventArgs(string id, decimal amt) { ClaimId = id; Amount = amt; }
}

// 2. Publisher
public class ClaimService
{
    public event EventHandler<ClaimEventArgs>? ClaimApproved;   // auto-implemented event

    public void Approve(string id, decimal amount)
    {
        // ... business logic ...
        OnClaimApproved(new ClaimEventArgs(id, amount));
    }

    protected virtual void OnClaimApproved(ClaimEventArgs e)
        => ClaimApproved?.Invoke(this, e);     // raise: null-safe, sender = this
}

// 3. Subscribers
var service = new ClaimService();
service.ClaimApproved += (sender, e) => Console.WriteLine($"Email: {e.ClaimId} approved");
service.ClaimApproved += (sender, e) => Console.WriteLine($"SMS: Rs {e.Amount}");
service.ClaimApproved += AuditLog;

service.Approve("CLM-01", 25000);   // all three handlers run

service.ClaimApproved -= AuditLog;  // unsubscribe (works for named methods, not inline lambdas)

static void AuditLog(object? sender, ClaimEventArgs e) => Console.WriteLine($"Audit: {e.ClaimId}");
```

**EventHandler** is the standard built-in delegate for events: `delegate void EventHandler(object? sender, EventArgs e)` and the generic `EventHandler<TEventArgs>`. Use `EventArgs.Empty` when there's no data.

**Custom delegate event (older style)**

```csharp
public delegate void StockLowHandler(string item, int left);
public event StockLowHandler? StockLow;
```

**Explicit add/remove (custom event accessors)**: rare; gives control over storage:

```csharp
private EventHandler? _changed;
public event EventHandler Changed
{
    add    { _changed += value; Console.WriteLine("subscribed"); }
    remove { _changed -= value; }
}
```

|  | Plain delegate field | Event |
| --- | --- | --- |
| Outside code can `+=` / `-=` | Yes | Yes |
| Outside code can assign `=` (wiping others) | **Yes** | No |
| Outside code can invoke it | **Yes** | No: only the declaring class |
| Purpose | Pass behaviour around | Notify subscribers |

**Conventions:** name events with a verb (`ClaimApproved`, `Closing`), raise through a `protected virtual OnXxx` method, pass `this` as sender, and **unsubscribe** long-lived publishers' events to avoid memory leaks.

## 3. Anonymous Methods, Lambda Expressions and Inline Lambdas

Instead of writing a separate named method for every delegate, you can write the code **inline**. Anonymous methods (C# 2) did this first; lambda expressions (C# 3) are the shorter, modern form.

### Anonymous methods

```csharp
Notify n = delegate (string msg)
{
    Console.WriteLine($"Anon: {msg}");
};
n("hello");

DiscountRule half = delegate (decimal a) { return a / 2; };
```

### Lambda expressions

Syntax: `(parameters) => expression` or `(parameters) => { statements }`. Read `=>` as "goes to".

```csharp
DiscountRule festive = a => a * 0.9m;                    // one parameter: brackets optional
Func<int, int, int> add = (x, y) => x + y;              // several parameters
Action greet = () => Console.WriteLine("Hi");            // no parameters
Func<int, bool> isAdult = age =>                         // statement body
{
    if (age < 0) throw new ArgumentException();
    return age >= 18;
};
Func<decimal, decimal> withTax = (decimal a) => a * 1.18m;   // explicit parameter type
```

### Inline lambdas (passed directly as arguments)

The most common use: hand a lambda straight to a method that expects a delegate, especially LINQ.

```csharp
var claims = new List<Claim> { /* ... */ };

var big      = claims.Where(c => c.Amount > 50_000);         // filter
var names    = claims.Select(c => c.PatientName);            // project
var sorted   = claims.OrderBy(c => c.SubmittedOn);           // sort key
bool anyRej  = claims.Any(c => c.Status == "Rejected");
claims.ForEach(c => Console.WriteLine(c.Id));

button.Click += (sender, e) => Console.WriteLine("Clicked");   // event handler
```

**Closures (captured variables):** a lambda can use local variables from its surrounding method; it captures the **variable**, not a copy of its value.

```csharp
int threshold = 1000;
Func<int, bool> above = x => x > threshold;
threshold = 5000;
Console.WriteLine(above(2000));   // False: uses the current value 5000
```

|  | Anonymous method | Lambda |
| --- | --- | --- |
| Syntax | `delegate (int x) { return x * 2; }` | `x => x * 2` |
| Parameter types | Must be written | Usually inferred |
| Expression body | No | Yes |
| Can omit parameter list entirely | Yes | No |
| Can become an expression tree | No | Yes (expression lambdas) |
| Today | Rarely used | Standard |

`static` lambdas (C# 9), such as `static x => x * 2`, can't capture locals, which avoids accidental closures.

## 4. Func, Action and Predicate

.NET ships ready-made generic delegates, so you rarely need to declare your own: **Func** returns a value, **Action** returns nothing, **Predicate** returns `bool`.

| Delegate | Signature | Returns | Example |
| --- | --- | --- | --- |
| `Func<TResult>` | no inputs | `TResult` | `Func<DateTime> now = () => DateTime.Now;` |
| `Func<T1, …, TResult>` | up to 16 inputs; **last** type is the return | `TResult` | `Func<int, int, int> add = (a, b) => a + b;` |
| `Action` | no inputs | `void` | `Action beep = () => Console.Beep();` |
| `Action<T1, …>` | up to 16 inputs | `void` | `Action<string> log = m => Console.WriteLine(m);` |
| `Predicate<T>` | one input | `bool` | `Predicate<int> isEven = n => n % 2 == 0;` |

```csharp
// Func: compute something
Func<decimal, decimal, decimal> premium = (sum, rate) => sum * rate / 100;
decimal p = premium(500_000, 1.2m);                 // 6000

// Action: do something
Action<string, int> printRow = (name, age) => Console.WriteLine($"{name,-10}{age}");
printRow("Rohit", 30);

// Predicate: test something
Predicate<string> isPan = s => s.Length == 10;
List<string> ids = new() { "ABCDE1234F", "X1" };
List<string> valid = ids.FindAll(isPan);           // List<T>.FindAll takes Predicate<T>
int firstBad = ids.FindIndex(s => !isPan(s));      // 1

// Passing behaviour into your own methods
static void Retry(Action work, int times)
{
    for (int i = 1; i <= times; i++)
    {
        try { work(); return; }
        catch when (i < times) { Console.WriteLine($"Retry {i}"); }
    }
}
Retry(() => CallInsurerApi(), 3);
```

**Where you meet them:** LINQ uses `Func` (`Where(Func<T,bool>)`, `Select(Func<T,TResult>)`); `List<T>.Find/FindAll/RemoveAll/Exists` use `Predicate<T>`; `List<T>.ForEach` and `Task.Run` use `Action`.

**`Predicate<T>` vs `Func<T, bool>`:** same shape, but they're different types and not interchangeable. LINQ uses `Func<T,bool>`, older `List<T>` methods use `Predicate<T>`.

**Custom delegate or Func/Action?** Prefer Func/Action. Declare your own only for clearer names in public APIs, or for `ref`/`out` parameters (Func/Action don't support them).

## 5. Expression Trees, Expression-Bodied Members and Switch Expressions

### Expression trees

Assign a lambda to `Expression<TDelegate>` instead of a delegate and the compiler builds a **data structure describing the code** instead of compiled code. Libraries can then inspect it and translate it, which is how **Entity Framework turns `c => c.Amount > 1000` into SQL**.

```csharp
using System.Linq.Expressions;

Func<int, bool>             asCode = x => x > 5;    // compiled, runnable
Expression<Func<int, bool>> asData = x => x > 5;    // tree you can inspect

Console.WriteLine(asData);              // x => (x > 5)
Console.WriteLine(asData.Body.NodeType); // GreaterThan
var body = (BinaryExpression)asData.Body;
Console.WriteLine(body.Left);           // x
Console.WriteLine(body.Right);          // 5

Func<int, bool> compiled = asData.Compile();   // turn the tree into code
Console.WriteLine(compiled(9));         // True

// Build a tree by hand: (a, b) => a + b
var a = Expression.Parameter(typeof(int), "a");
var b = Expression.Parameter(typeof(int), "b");
var add = Expression.Lambda<Func<int, int, int>>(Expression.Add(a, b), a, b).Compile();
Console.WriteLine(add(2, 3));           // 5
```

**Rules:** only **expression lambdas** (single expression, no `{ }` body) can become expression trees. `IQueryable<T>` (EF Core) takes `Expression<Func<...>>`; `IEnumerable<T>` (in-memory LINQ) takes plain `Func<...>`.

### Expression-bodied members

Any member whose body is a single expression can use `=>` instead of `{ return ...; }`.

```csharp
public class Patient
{
    private string _name = "";
    public Patient(string name) => _name = name;                  // constructor (C# 7)
    ~Patient() => Console.WriteLine("Finalized");                 // finalizer (C# 7)
    public string Name { get => _name; set => _name = value.Trim(); }   // accessors (C# 7)
    public int NameLength => _name.Length;                        // read-only property (C# 6)
    public string Greet(string greeting) => $"{greeting}, {_name}";   // method (C# 6)
    public override string ToString() => _name;
    public char this[int i] => _name[i];                          // indexer
}
```

Use them for short one-liners; switch back to a block body when logic grows.

### Switch expressions (C# 8)

A `switch` that **returns a value**, with patterns instead of `case` labels.

```csharp
string Category(int age) => age switch
{
    < 0          => throw new ArgumentOutOfRangeException(nameof(age)),
    < 18         => "Minor",
    >= 18 and < 60 => "Adult",
    _            => "Senior"                 // _ = discard, like default
};

decimal Copay(Claim c) => c switch
{
    { Type: "Cashless", Amount: > 100_000 } => 0.10m,    // property pattern
    { Type: "Cashless" }                    => 0.05m,
    { Type: "Reimbursement" }               => 0.20m,
    null                                    => throw new ArgumentNullException(nameof(c)),
    _                                       => 0m
};

string Quadrant((int X, int Y) p) => p switch
{
    (> 0, > 0) => "Q1",                     // tuple / positional pattern
    (< 0, > 0) => "Q2",
    _          => "Other"
};
```

|  | switch statement | switch expression |
| --- | --- | --- |
| Produces a value | No | Yes |
| Syntax | `case x: ... break;` | `pattern => result,` |
| Default | `default:` | `_` |
| Non-exhaustive | Allowed | Compiler warning; throws `SwitchExpressionException` if nothing matches |

## 6. Cheat Sheet, Checklist and Interview Questions

```csharp
delegate int Op(int a, int b);                     // custom delegate
Op add = (a, b) => a + b;                          // lambda
Op mul = delegate (int a, int b) { return a * b; }; // anonymous method
Action<string> log = Console.WriteLine;            // method group
log += s => File.AppendAllText("log.txt", s);      // multicast

Func<int, int> square = x => x * x;
Predicate<int> isOdd = x => x % 2 != 0;
Expression<Func<int, bool>> tree = x => x > 10;    // expression tree

public event EventHandler<MyArgs>? Done;           // event
Done?.Invoke(this, new MyArgs());                  // raise
obj.Done += (s, e) => { };                         // subscribe

public int Total => Items.Sum(i => i.Price);       // expression-bodied
string size = n switch { < 10 => "S", < 100 => "M", _ => "L" };   // switch expression
```

**Checklist**

- [ ] Declare a delegate, assign two methods and invoke it
- [ ] Build a multicast delegate and explain return values and exceptions
- [ ] Write an event with `EventHandler<T>` and a custom `EventArgs`
- [ ] Explain why events are safer than public delegate fields
- [ ] Convert an anonymous method into a lambda
- [ ] Use Func, Action and Predicate in one program
- [ ] Inspect an expression tree and explain how EF uses it
- [ ] Rewrite a switch statement as a switch expression

**Interview questions (short answers)**

1. **What is a delegate?** A type-safe reference to a method with a specific signature.
2. **What is a multicast delegate?** A delegate holding several methods (`+=`), all invoked in order.
3. **Return value of a multicast delegate?** The last method's return value.
4. **Delegate vs event?** An event restricts outside code to `+=`/`-=`; only the owner can raise it or reset it.
5. **What is `EventHandler`?** The standard event delegate `(object? sender, EventArgs e)`; generic `EventHandler<T>` for custom data.
6. **What is an anonymous method?** An unnamed inline method written with the `delegate` keyword.
7. **What is a lambda expression?** A concise anonymous function: `(params) => expression`.
8. **What is a closure?** A lambda capturing outer variables; it sees their current values.
9. **Func vs Action vs Predicate?** Func returns a value (last type param), Action returns void, Predicate returns bool.
10. **What is an expression tree?** A lambda represented as data (`Expression<T>`), which can be inspected, translated (e.g. to SQL) or compiled.
11. **What are expression-bodied members?** Members written with `=>` and a single expression.
12. **What is a switch expression?** An expression form of switch that returns a value using patterns (C# 8).
13. **How do you avoid memory leaks with events?** Unsubscribe (`-=`) handlers when the subscriber's lifetime is shorter than the publisher's.
14. **How do you safely raise an event?** `MyEvent?.Invoke(this, args);`
