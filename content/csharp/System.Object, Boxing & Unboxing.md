# System.Object, Boxing & Unboxing

Every type in .NET derives from `System.Object`. This tab covers its members, how to override them properly, and what happens when value types are boxed into objects and unboxed back.

## 1. Overview of System.Object

`System.Object` (keyword `object`) is the ultimate base class of every type in .NET: classes, structs, enums, arrays, delegates and records. Even when you write `class Claim { }`, the compiler makes it `class Claim : object`. So every value can be stored in an `object` variable, and every object has the members below.

| Member | Kind | Default behaviour | Override? |
| --- | --- | --- | --- |
| `ToString()` | public virtual | Returns the full type name, e.g. `"HealthIndia.Claim"` | Yes, commonly |
| `Equals(object obj)` | public virtual | Classes: reference equality. Structs: compares all fields (via reflection, slow) | Yes, for value-like types |
| `GetHashCode()` | public virtual | Number used by `Dictionary`/`HashSet` buckets | Yes, **whenever you override Equals** |
| `GetType()` | public, non-virtual | Returns the runtime `Type` of the object | No (can't) |
| `MemberwiseClone()` | protected | Shallow copy of the object | No; call it inside a `Clone()` method |
| `Finalize()` | protected virtual | Cleanup before GC (written as `~ClassName()`) | Rarely; prefer `IDisposable` |
| `Equals(object a, object b)` | public static | Null-safe equality check | No |
| `ReferenceEquals(a, b)` | public static | True only if both refer to the same object | No |

```csharp
object[] things = { 42, "hi", 3.5, new DateTime(2026, 11, 20), new int[] { 1 } };
foreach (object t in things)
    Console.WriteLine($"{t.GetType().Name}: {t}");   // every type has GetType and ToString
// Int32: 42
// String: hi
// Double: 3.5
// DateTime: 20-11-2026 00:00:00
// Int32[]: System.Int32[]
```

## 2. Understanding and Overriding Object Methods

Override `ToString` to give a readable description, and override `Equals` + `GetHashCode` together when two different objects with the same data should count as equal.

### ToString

```csharp
public class Patient
{
    public string Uhid { get; init; }
    public string Name { get; init; }

    public override string ToString() => $"{Name} ({Uhid})";
}

var p = new Patient { Uhid = "U100", Name = "Rohit" };
Console.WriteLine(p);           // Rohit (U100): WriteLine calls ToString()
string s = $"Patient: {p}";     // interpolation calls it too
```

Without the override, `Console.WriteLine(p)` prints `Patient` (namespace-qualified type name).

### Equals and GetHashCode

```csharp
var a = new Patient { Uhid = "U100", Name = "Rohit" };
var b = new Patient { Uhid = "U100", Name = "Rohit" };
Console.WriteLine(a.Equals(b));   // False by default: two different objects
```

Override both so patients with the same UHID are equal:

```csharp
public class Patient
{
    public string Uhid { get; init; }
    public string Name { get; init; }

    public override bool Equals(object? obj)
    {
        if (ReferenceEquals(this, obj)) return true;      // same object
        if (obj is not Patient other) return false;       // null or different type
        return Uhid == other.Uhid;                        // compare what defines identity
    }

    public override int GetHashCode() => Uhid.GetHashCode();   // same fields as Equals

    // optional: make == and != agree with Equals
    public static bool operator ==(Patient? l, Patient? r) => Equals(l, r);
    public static bool operator !=(Patient? l, Patient? r) => !Equals(l, r);

    public override string ToString() => $"{Name} ({Uhid})";
}

Console.WriteLine(a.Equals(b));        // True
var set = new HashSet<Patient> { a, b };
Console.WriteLine(set.Count);          // 1: treated as the same patient
```

**The Equals / GetHashCode contract**

- If `a.Equals(b)` is true, `a.GetHashCode()` **must** equal `b.GetHashCode()`. Break this and `Dictionary`/`HashSet` lose or duplicate items.
- Equal hash codes don't require equal objects (collisions are fine).
- Equals must be reflexive, symmetric, transitive, and return false for `null`.
- For several fields use `HashCode.Combine(Uhid, Name)`.
- Base hash codes on fields that don't change while the object is in a collection.

**Shortcut:** a `record` generates `Equals`, `GetHashCode`, `==`, `!=` and `ToString` for you based on its properties: `public record Patient(string Uhid, string Name);`

### GetType vs typeof vs is

```csharp
object o = new Patient();
bool exact = o.GetType() == typeof(Patient);   // exact runtime type check
bool isP = o is Patient;          // true for Patient OR any subclass
Console.WriteLine(o.GetType().Name);   // Patient
```

## 3. Important Points about the Object Class

1. Every type derives from `System.Object`, directly or indirectly; structs via `System.ValueType`, enums via `System.Enum`.
2. Only three instance methods are usually overridden: `ToString`, `Equals`, `GetHashCode`.
3. Always override `GetHashCode` when you override `Equals` (compiler warning CS0659 otherwise).
4. `GetType()` can't be overridden; it always reports the true runtime type.
5. Default `Equals` for classes = reference equality; for structs = field-by-field (slow, so override it for performance).
6. `==` on classes compares references unless the operator is overloaded; `string` overloads it to compare text.
7. `object.ReferenceEquals(a, b)` is the guaranteed identity check, ignoring overrides.
8. Storing a value type in an `object` boxes it (section 4).
9. `MemberwiseClone` makes a shallow copy: reference-type fields still point to the same objects.

## 4. Boxing and Unboxing

**Boxing** wraps a value type in an object on the heap so it can be treated as `object` (or an interface). **Unboxing** extracts the value back with an explicit cast. Both make a **copy**.

&#91;embedded content: Boxing and unboxing\]

After boxing, `i` and the boxed value are independent: changing `i` later doesn't change `o`.

### Boxing (implicit)

```csharp
int i = 42;
object o = i;              // boxing: new heap object holding a copy of 42
i = 100;
Console.WriteLine(o);      // 42: the box has its own copy

IComparable c = 7;         // boxing to an interface
ArrayList old = new ArrayList();
old.Add(5);                // boxing: non-generic collections store object
string s = string.Format("{0}", 3.5);   // the double is boxed
```

### Unboxing (explicit)

```csharp
object o = 42;
int j = (int)o;            // unboxing: type check + copy back to the stack

// long l = (long)o;       // InvalidCastException: must unbox to the EXACT type
long l = (long)(int)o;     // OK: unbox to int first, then convert

object n = null;
// int k = (int)n;         // NullReferenceException
int? k = (int?)n;          // OK: null

if (o is int value)        // safest: test and unbox in one step
    Console.WriteLine(value);
```

|  | Boxing | Unboxing |
| --- | --- | --- |
| Direction | Value type → `object` / interface | `object` / interface → value type |
| Syntax | Implicit: `object o = i;` | Explicit cast: `int j = (int)o;` |
| Memory | Allocates a new heap object | Copies the value out |
| Can fail? | No | Yes: `InvalidCastException`, `NullReferenceException` |
| Rule | Any value type | Must cast to the exact boxed type (or its nullable) |

**Why it matters:** each box is a heap allocation for the garbage collector to clean up, and the type checks cost time. In loops this adds up.

```csharp
// Slow: 1,000,000 boxes
ArrayList list = new ArrayList();
for (int x = 0; x < 1_000_000; x++) list.Add(x);

// Fast: no boxing, generics store ints directly
List<int> fast = new List<int>();
for (int x = 0; x < 1_000_000; x++) fast.Add(x);
```

**Avoid boxing by:** using generic collections (`List<T>`, `Dictionary<K,V>`), generic methods with `T`, string interpolation (modern .NET avoids most boxing there), and overriding `ToString`/`Equals` on structs.

## 5. Cheat Sheet, Checklist and Interview Questions

```csharp
public class Policy
{
    public string No { get; init; }
    public decimal Premium { get; init; }

    public override string ToString() => $"{No}: {Premium:N0}";
    public override bool Equals(object? obj) => obj is Policy p && p.No == No;
    public override int GetHashCode() => No.GetHashCode();
}

object o = 42;                    // boxing
int n = (int)o;                   // unboxing (exact type)
if (o is int v) { }               // safe unbox
Type t = o.GetType();             // System.Int32
bool same = ReferenceEquals(a, b);   // identity check
// prefer List<int> over ArrayList to avoid boxing
```

**Checklist**

- [ ] List the members of `System.Object` and which are virtual
- [ ] Override `ToString` on a class
- [ ] Override `Equals` + `GetHashCode` and prove it with a `HashSet`
- [ ] Explain the Equals/GetHashCode contract
- [ ] Draw boxing and unboxing between stack and heap
- [ ] Show the exact-type unboxing rule and its exception
- [ ] Rewrite an `ArrayList` loop with `List<int>`

**Interview questions (short answers)**

1. **What is `System.Object`?** The root base class of every .NET type.
2. **Which Object methods can be overridden?** `ToString`, `Equals`, `GetHashCode` (and `Finalize` via a finalizer).
3. **Why override `GetHashCode` with `Equals`?** Hash-based collections use the hash first; equal objects must have equal hash codes.
4. **`==` vs `Equals`?** For classes, `==` compares references unless overloaded; `Equals` can be overridden for value equality. `string` overrides both.
5. **`ReferenceEquals` vs `Equals`?** `ReferenceEquals` always checks identity and can't be overridden.
6. **`GetType()` vs `typeof`?** `GetType()` gets the runtime type of an instance; `typeof(T)` gets a type known at compile time.
7. **What is boxing?** Implicitly converting a value type to `object`/interface by copying it into a new heap object.
8. **What is unboxing?** Explicitly extracting the value from a boxed object back into a value type.
9. **What happens if you unbox to the wrong type?** `InvalidCastException`; you must unbox to the exact type.
10. **Why avoid boxing?** Heap allocations, GC pressure and type-check overhead.
11. **How do generics avoid boxing?** `List<int>` stores ints directly instead of as `object`.
12. **What does `MemberwiseClone` do?** A shallow copy; nested reference objects are shared.
