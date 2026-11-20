# Nullable Types & Null Operators

How C# represents "no value" safely: nullable types, and the `??`, `??=`, `?.` and `?[]` operators that replace long null checks.

## 1. Nullable Types

Value types like `int`, `bool` and `DateTime` can't normally be `null`. Add `?` (`int?`) to get a **nullable value type** that holds either a value or `null`. That's essential for optional data, such as a database column that may be empty.

```csharp
// int age = null;          // ERROR: int can't be null
int? age = null;            // OK: no value yet
age = 30;

DateTime? dischargeDate = null;   // patient not discharged yet
bool? isApproved = null;          // three states: true, false, unknown
decimal? claimAmount = 25000m;
```

`int?` is shorthand for the struct `Nullable<int>`, which has two key members:

| Member | Returns |
| --- | --- |
| `HasValue` | `true` if a value is present |
| `Value` | The value; throws `InvalidOperationException` if null |
| `GetValueOrDefault()` | The value, or `default` (0, false…) if null |
| `GetValueOrDefault(x)` | The value, or `x` if null |

```csharp
int? marks = null;

if (marks.HasValue)                     // classic check
    Console.WriteLine(marks.Value);

if (marks is int m)                     // pattern: check + unwrap
    Console.WriteLine(m);

int safe = marks.GetValueOrDefault();   // 0
int withDefault = marks ?? -1;          // -1 (see section 2)

// int bad = marks;                     // ERROR: int? to int needs a cast or ??
int forced = (int)marks;                // compiles, but throws if marks is null
```

**Arithmetic and comparison with null ("lifted operators")**

```csharp
int? a = 5, b = null;
Console.WriteLine(a + b);      // (blank): null; any arithmetic with null gives null
Console.WriteLine(a > b);      // False: comparisons with null are false
Console.WriteLine(b == null);  // True
```

### Nullable reference types (C# 8+)

Reference types (`string`, classes) could always be `null`. With `<Nullable>enable</Nullable>` (on by default in new projects), the compiler **warns** you about possible nulls:

```csharp
string name = null;        // warning CS8600: non-nullable variable set to null
string? nickname = null;   // OK: '?' says null is allowed

Console.WriteLine(nickname.Length);    // warning CS8602: possible null dereference
Console.WriteLine(nickname!.Length);   // '!' null-forgiving: "trust me" (still crashes if null)
```

This is only compile-time checking; at run time `string?` and `string` are the same type.

## 2. Null Coalescing Operator ?? and ??=

`a ?? b` returns `a` if it's not null, otherwise `b`. It's a one-line replacement for "if null, use a default".

```csharp
string? input = null;
string name = input ?? "Guest";             // "Guest"

int? discount = null;
int applied = discount ?? 0;                // 0: also turns int? into int

// same as:
string name2 = input != null ? input : "Guest";
```

**Chaining:** the first non-null value wins.

```csharp
string display = user.NickName ?? user.FirstName ?? user.Email ?? "Unknown";
```

**Throw if missing (C# 7):**

```csharp
public Policy(string number) =>
    Number = number ?? throw new ArgumentNullException(nameof(number));
```

### Null-coalescing assignment ??= (C# 8)

`x ??= value` assigns only if `x` is currently null. Great for lazy initialisation.

```csharp
List<string>? tags = null;
tags ??= new List<string>();     // created because tags was null
tags ??= new List<string>();     // ignored: tags already has a value

private Dictionary<string, int>? _cache;
public Dictionary<string, int> Cache => _cache ??= LoadCache();   // load once, on first use
```

| Operator | Meaning | Example |
| --- | --- | --- |
| `a ?? b` | `a` if not null, else `b` | `name ?? "Guest"` |
| `a ?? throw ...` | `a` if not null, else throw | `id ?? throw new ArgumentNullException()` |
| `a ??= b` | If `a` is null, set `a = b` | `list ??= new()` |

**Rules:** the left side must be a nullable or reference type; `??` is right-associative (`a ?? b ?? c` = `a ?? (b ?? c)`); `b` is evaluated only when `a` is null.

## 3. Null Propagation Operator ?. and ?\[\]

`a?.Member` accesses `Member` only if `a` isn't null; if `a` is null, the whole expression returns `null` instead of throwing `NullReferenceException`. Also called the **null-conditional** or "Elvis" operator (C# 6).

```csharp
Patient? p = GetPatient("U100");     // may return null

// Old way
string? city1 = null;
if (p != null && p.Address != null)
    city1 = p.Address.City;

// With ?.: stops at the first null in the chain
string? city2 = p?.Address?.City;

// Combine with ?? for a default
string city = p?.Address?.City ?? "Not provided";
```

**Null-conditional indexer `?[]`**

```csharp
string[]? names = null;
string? first = names?[0];             // null, no exception
int count = names?.Length ?? 0;        // 0

Dictionary<string, string>? map = null;
string? v = map?["key"];
```

**Calling methods and events safely**

```csharp
p?.PrintSummary();                     // method runs only if p isn't null
OnSaved?.Invoke(this, EventArgs.Empty);  // standard thread-safe way to raise an event
int? len = p?.Name.Length;             // value-type result becomes nullable: int?
```

**Rules to remember**

- The result type becomes nullable: `p?.Age` is `int?` even if `Age` is `int`.
- **Short-circuits:** if `p` is null, nothing to the right runs (no method calls, no argument evaluation).
- `p?.Name.Length` checks only `p`; if `Name` itself can be null, write `p?.Name?.Length`.
- Since C# 14 it can also be used on the left of an assignment: `p?.Name = "X";` assigns only if `p` isn't null.
- Don't overuse it: if a value should never be null, a crash (or an explicit check that throws) finds the bug faster than silently getting `null`.

| Operator | Name | Does |
| --- | --- | --- |
| `?.` | Null-conditional member access | Access member if not null, else null |
| `?[]` | Null-conditional index | Index if not null, else null |
| `??` | Null-coalescing | Use left if not null, else right |
| `??=` | Null-coalescing assignment | Assign if currently null |
| `!` | Null-forgiving | Silence nullable warnings (no runtime effect) |

## 4. Important Points, Cheat Sheet, Checklist and Interview Questions

**Points to remember**

1. `T?` on a value type = `Nullable<T>`: a struct with `HasValue` and `Value`.
2. Reading `.Value` on a null nullable throws `InvalidOperationException`; use `??`, `GetValueOrDefault` or a pattern instead.
3. Arithmetic with null gives null; comparisons with null give false (except `==`/`!=`).
4. `string?` (nullable reference type) is a compile-time annotation only.
5. `??` supplies a default; `??=` assigns only when null; `?.`/`?[]` stop at the first null.
6. `?.` makes value-type results nullable, so pair it with `??` to get a plain value.
7. These operators evaluate the left side only once.

**Cheat sheet**

```csharp
int? n = null;                         // nullable value type
bool has = n.HasValue;                 // false
int a = n ?? 0;                        // default if null
int b = n.GetValueOrDefault(10);       // 10
if (n is int v) { }                    // safe unwrap

string? s = null;                      // nullable reference type
int len = s?.Length ?? 0;              // 0
char? c = s?[0];                       // null
s ??= "default";                       // assign if null
string t = s ?? throw new ArgumentNullException(nameof(s));
handler?.Invoke(this, EventArgs.Empty);
```

**Checklist**

- [ ] Declare nullable value types and use `HasValue`, `Value`, `GetValueOrDefault`
- [ ] Predict results of arithmetic and comparisons with null
- [ ] Explain nullable reference types and `!`
- [ ] Replace an if/else default with `??` and a lazy init with `??=`
- [ ] Replace nested null checks with `?.` and `?[]`
- [ ] Raise an event with `?.Invoke`

**Interview questions (short answers)**

1. **What is a nullable type?** A value type that can also hold `null`, written `T?` (`Nullable<T>`).
2. **Why are they needed?** To represent missing values, such as nullable database columns or optional fields.
3. **`HasValue` vs `Value`?** `HasValue` tells if there's a value; `Value` returns it or throws if null.
4. **What does `??` do?** Returns the left operand if not null, otherwise the right operand.
5. **What does `??=` do?** Assigns the right side only if the left variable is null.
6. **What is `?.`?** Null-conditional access: returns null instead of throwing when the object is null.
7. **What is the type of `p?.Age` when Age is int?** `int?`.
8. **What are nullable reference types?** A C# 8 feature where `string?` marks references that may be null and the compiler warns on unsafe use.
9. **What is the `!` operator?** Null-forgiving: suppresses nullable warnings; no runtime check.
10. **What is `null + 5` for `int?`?** `null`.
11. **How do you safely raise an event?** `MyEvent?.Invoke(this, args);`
