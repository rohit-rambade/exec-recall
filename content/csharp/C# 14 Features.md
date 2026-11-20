# C# 14 Features

C# 14 shipped with .NET 10 (LTS) in November 2025. Like your course, this page starts from an existing order report and improves it one feature at a time.

## 1. The Starting Point: the Existing Order Report

This is a typical pre-C# 14 report. Read it first and spot the pain points; each later section fixes one of them.

```csharp
public class Address { public string City { get; set; } = ""; }

public class Order
{
    private string _customer = "";                        // (1) manual backing field just to trim
    public string Customer
    {
        get => _customer;
        set => _customer = value.Trim();
    }
    public string Id { get; set; } = "";
    public decimal Amount { get; set; }
    public Address? ShippingAddress { get; set; }
}

public static class OrderHelpers                         // (2) helpers called as static methods
{
    public static bool IsLarge(Order o) => o.Amount > 10_000;
    public static decimal Total(IEnumerable<Order> orders) => orders.Sum(o => o.Amount);
    public static Order Empty() => new() { Customer = "N/A" };
}

delegate bool OrderIdParser(string text, out int id);

void PrintReport(List<Order> orders)
{
    OrderIdParser parse = (string text, out int id) => int.TryParse(text, out id);   // (3) types required

    foreach (var o in orders)
    {
        if (o.ShippingAddress != null)                    // (4) null check before assigning
            o.ShippingAddress.City = o.ShippingAddress.City.ToUpper();

        string code = o.Customer.Substring(0, 3);         // (5) allocates a new string per order
        string flag = OrderHelpers.IsLarge(o) ? "LARGE" : "";
        parse(o.Id[4..], out int number);
        Console.WriteLine($"{number,5} {code} {o.Amount,10:N0} {flag}");
    }
    Console.WriteLine($"Total: {OrderHelpers.Total(orders):N0}");
}
```

| # | Pain point | C# 14 fix |
| --- | --- | --- |
| 1 | Backing field only to add logic | `field` keyword |
| 2 | Helpers read as `OrderHelpers.IsLarge(o)` | Extension properties and static extension members |
| 3 | Lambda must repeat parameter types to use `out` | Lambda parameter modifiers without types |
| 4 | `if (x != null) x.Prop = ...` | Null-conditional assignment |
| 5 | Substring allocations in a loop | Spans, now with implicit conversions |

## 2. Extension Blocks: Methods, Properties and Static Members

Classic extension methods (`this Order o`) could only add **instance methods**. C# 14's **extension blocks** (`extension(receiver) { ... }`) can add instance methods, **properties**, and **static members** to existing types, all grouped by the type they extend.

### Extension blocks and methods

```csharp
public static class OrderExtensions
{
    extension(Order order)                             // receiver: an Order instance named 'order'
    {
        public bool IsAbove(decimal limit) => order.Amount > limit;   // extension METHOD
        public string Describe() => $"{order.Id} for {order.Customer}";
    }
}

order.IsAbove(5000);      // called like an instance method
order.Describe();
```

The old syntax still works and means the same thing: `public static bool IsAbove(this Order order, decimal limit)`. You can mix both styles; extension blocks just let you write more kinds of members.

### Extension properties and static members

```csharp
public static class OrderExtensions
{
    extension(Order order)
    {
        public bool IsLarge => order.Amount > 10_000;                  // extension PROPERTY
        public string ShortCode => order.Customer.Length >= 3
            ? order.Customer[..3].ToUpper() : order.Customer.ToUpper();
    }

    extension(IEnumerable<Order> orders)                               // extend a collection type
    {
        public decimal Total => orders.Sum(o => o.Amount);
        public IEnumerable<Order> LargeOnly => orders.Where(o => o.IsLarge);
    }

    extension(Order)                                                    // no name: STATIC extension members
    {
        public static Order Empty => new() { Customer = "N/A" };
        public static Order FromCsv(string line)
        {
            var p = line.Split(',');
            return new Order { Id = p[0], Customer = p[1], Amount = decimal.Parse(p[2]) };
        }
    }

    extension<T>(IEnumerable<T> source)                                 // generic extension block
    {
        public bool IsEmpty => !source.Any();
    }
}

bool big = order.IsLarge;                    // property, no parentheses
decimal total = orders.Total;
Order blank = Order.Empty;                    // static member: looks like it belongs to Order
Order parsed = Order.FromCsv("ORD-1001,Rohit,12500");
bool none = orders.IsEmpty;
```

**Rules:** extension blocks live inside a non-generic, top-level `static class`; a named receiver (`extension(Order order)`) gives instance members, an unnamed one (`extension(Order)`) gives static members; extension properties can't have backing storage (no auto-properties or `field`); like all extensions, they only see public members and lose to a real member with the same name.

## 3. The Released field Keyword and Null-Conditional Assignment

### field keyword (now standard)

Previewed in C# 13, `field` is a normal C# 14 feature: it names the property's hidden backing field.

```csharp
public class Order
{
    public string Customer                       // no _customer field needed
    {
        get;
        set => field = value.Trim();
    } = "";

    public decimal Amount
    {
        get;
        set => field = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
    }

    public string Status { get; set => field = value.ToUpperInvariant(); } = "NEW";

    public List<string> Notes => field ??= new();   // lazy initialisation in a getter
}
```

### Null-conditional assignment

`?.` and `?[]` can now appear on the **left** of `=` and compound assignments (`+=`, `-=`…). If the object is null, the assignment is skipped, and the right side **isn't even evaluated**.

```csharp
// Before
if (order.ShippingAddress != null)
    order.ShippingAddress.City = order.ShippingAddress.City.ToUpper();

// C# 14
order.ShippingAddress?.City = order.ShippingAddress.City.ToUpper();

customer?.LoyaltyPoints += 50;                  // compound assignment
orders?[0].Status = "PRIORITY";                // index then member
handlers?.OnSaved += LogSave;                   // events too
```

**Rules:** works with `=` and compound operators, not with `++`/`--`; the right-hand side is evaluated only when the target isn't null (so it can safely use the same chain); the expression's result is null when skipped.

## 4. Span Basics and C# 14 Span Improvements

### Span basics

`Span<T>` and `ReadOnlySpan<T>` are **views over contiguous memory** (an array, part of a string, stack memory) that let you slice and read data **without copying or allocating**. They're `ref struct`s, so they live only on the stack.

```csharp
string id = "ORD-2026-00123";

string yearCopy = id.Substring(4, 4);            // allocates a new string
ReadOnlySpan<char> year = id.AsSpan(4, 4);       // no allocation: a window onto id
int y = int.Parse(year);                          // many APIs accept spans directly

int[] amounts = { 100, 200, 300, 400 };
Span<int> middle = amounts.AsSpan(1, 2);         // [200, 300]
middle[0] = 999;                                  // writes into the ORIGINAL array
Console.WriteLine(amounts[1]);                    // 999

Span<byte> buffer = stackalloc byte[256];         // stack memory, no GC
bool same = "Pune".AsSpan().SequenceEqual("Pune");
```

| | `string.Substring` / array copy | `Span<T>` / `ReadOnlySpan<T>` |
| --- | --- | --- |
| Allocates | Yes | No |
| Can modify source | No (copy) | `Span<T>` yes, `ReadOnlySpan<T>` no |
| Where it can live | Anywhere | Stack only: not in class fields, async methods, lambdas that capture it |
| Use for | Simple code | Parsing, hot loops, high-performance libraries |

### C# 14: first-class span conversions

C# 14 makes spans first-class in the language: the compiler now applies **implicit conversions** `T[]` → `Span<T>` / `ReadOnlySpan<T>`, `Span<T>` → `ReadOnlySpan<T>`, and `string` → `ReadOnlySpan<char>` in more places, including **extension method receivers** and **generic type inference**.

```csharp
int[] codes = { 3, 7, 9 };

// MemoryExtensions methods are defined on spans; arrays can now call them directly
bool starts = codes.StartsWith([3, 7]);          // array converts to ReadOnlySpan<int> as receiver
int idx = codes.IndexOf(9);                       // span-based overload chosen

static int Sum(ReadOnlySpan<int> xs) { int t = 0; foreach (var x in xs) t += x; return t; }
Sum(codes);                                       // array -> ReadOnlySpan, no AsSpan() needed

// In the order report:
ReadOnlySpan<char> code = order.Customer.AsSpan(0, 3);   // instead of Substring: no allocation per order
```

The practical effect: fewer `.AsSpan()` calls, span overloads picked automatically (faster, allocation-free), and APIs can offer a single span-based extension that works for arrays, spans and strings alike.

## 5. Lambda Parameter Modifiers

Before C# 14, adding a modifier such as `out`, `ref`, `in`, `scoped` or `ref readonly` to one lambda parameter forced you to write **explicit types for every parameter**. Now you can use modifiers with inferred types.

```csharp
delegate bool OrderIdParser(string text, out int id);

// Before C# 14: types required because of 'out'
OrderIdParser p1 = (string text, out int id) => int.TryParse(text, out id);

// C# 14: modifiers without types
OrderIdParser p2 = (text, out id) => int.TryParse(text, out id);

delegate void Doubler(ref int value);
Doubler d = (ref v) => v *= 2;
int n = 5; d(ref n);                     // n = 10

delegate bool TryConvert<T>(string s, out T result);
TryConvert<decimal> parseAmount = (s, out result) => decimal.TryParse(s, out result);
```

**Rule:** `params` still requires an explicit type; all other modifiers now work with implicitly typed parameters. It's a small change that removes noise from callbacks and `Try...` style delegates.

## 6. The Refactored Order Report

```csharp
public class Order
{
    public string Customer { get; set => field = value.Trim(); } = "";      // (1) field
    public string Id { get; set; } = "";
    public decimal Amount { get; set; }
    public Address? ShippingAddress { get; set; }
}

public static class OrderExtensions                                           // (2) extension blocks
{
    extension(Order o) { public bool IsLarge => o.Amount > 10_000; }
    extension(IEnumerable<Order> orders) { public decimal Total => orders.Sum(x => x.Amount); }
    extension(Order) { public static Order Empty => new() { Customer = "N/A" }; }
}

void PrintReport(List<Order> orders)
{
    OrderIdParser parse = (text, out id) => int.TryParse(text, out id);       // (3) modifiers without types

    foreach (var o in orders)
    {
        o.ShippingAddress?.City = o.ShippingAddress.City.ToUpper();          // (4) null-conditional assignment
        ReadOnlySpan<char> code = o.Customer.AsSpan(0, 3);                    // (5) span, no allocation
        parse(o.Id[4..], out int number);
        Console.WriteLine($"{number,5} {code} {o.Amount,10:N0} {(o.IsLarge ? "LARGE" : "")}");
    }
    Console.WriteLine($"Total: {orders.Total:N0}");
}
```

### Other C# 14 features

- `nameof` with unbound generic types: `nameof(List<>)` → "List".
- Partial constructors and partial events (for source generators).
- User-defined compound assignment operators (`public void operator +=(Money m)`) for efficient in-place updates.

## 7. Cheat Sheet, Checklist and Interview Questions

```csharp
static class Ext
{
    extension(string s) { public bool IsBlank => string.IsNullOrWhiteSpace(s); public string Shout() => s.ToUpper() + "!"; }
    extension(string) { public static string Dash => "-"; }
}
public string Name { get; set => field = value.Trim(); } = "";
person?.Address = newAddress;    total?.Value += 5;
int[] arr = [1, 2, 3]; bool ok = arr.StartsWith([1]);        // array as span receiver
TryParser p = (s, out v) => int.TryParse(s, out v);
string n = nameof(Dictionary<,>);
```

**Checklist**

- [ ] Write an extension block with a method, a property and a static member
- [ ] Replace a manual backing field with `field`
- [ ] Rewrite an `if (x != null)` assignment using `?.=`
- [ ] Parse part of a string with `AsSpan` instead of `Substring`
- [ ] Write a `TryParse`-style lambda with `out` and no parameter types

**Interview questions (short answers)**

1. **What are extension blocks?** C# 14 syntax `extension(Type receiver) { ... }` inside a static class that groups extension members for a type.
2. **What can extension blocks add that extension methods couldn't?** Properties and static members (e.g. `Order.Empty`).
3. **How do you declare static extension members?** Use an unnamed receiver: `extension(Order) { public static ... }`.
4. **Can extension properties store data?** No; they compute values from the receiver.
5. **What is the `field` keyword?** A reference to a property's compiler-generated backing field, standard from C# 14.
6. **What is null-conditional assignment?** `a?.B = x`: assigns only if `a` isn't null, and skips evaluating `x` otherwise.
7. **What is Span&lt;T&gt;?** A stack-only view over contiguous memory that allows slicing without allocating.
8. **What changed for spans in C# 14?** Implicit conversions from arrays and strings to spans now apply to extension receivers and type inference.
9. **What are lambda parameter modifiers?** `out`, `ref`, `in` and `scoped` on lambda parameters without writing their types.
10. **Which .NET shipped C# 14?** .NET 10 (LTS), November 2025.
