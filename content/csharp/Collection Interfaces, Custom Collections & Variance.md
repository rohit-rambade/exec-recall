# Collection Interfaces, Custom Collections & Variance

What sits underneath every collection: the interface hierarchy, how `foreach` really works, `yield return`, building your own collections, equality and sorting interfaces, and generic variance.

## 1. Collection Classes and Interfaces Hierarchy

Every collection class implements a stack of interfaces, each adding abilities on top of the one below. Knowing the hierarchy tells you what any collection can do and which type to accept in a method parameter.

&#91;embedded content: Collection interfaces hierarchy\]

`IDictionary<K,V>` is itself an `ICollection` of `KeyValuePair<K,V>`. Read-only views (`IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `IReadOnlyDictionary<K,V>`) sit alongside these.

| Interface | Adds | Key members |
| --- | --- | --- |
| `IEnumerable<T>` | Can be looped with `foreach`, LINQ works | `GetEnumerator()` |
| `ICollection<T>` | Size and modification | `Count`, `Add`, `Remove`, `Contains`, `Clear`, `CopyTo`, `IsReadOnly` |
| `IList<T>` | Position / index | `this[int]`, `IndexOf`, `Insert`, `RemoveAt` |
| `ISet<T>` | Uniqueness and set maths | `UnionWith`, `IntersectWith`, `IsSubsetOf`, `Add` returns bool |
| `IDictionary<K,V>` | Key lookup | `this[key]`, `Keys`, `Values`, `ContainsKey`, `TryGetValue` |
| `IReadOnlyList<T>` | Index + count, no changes | `this[int]`, `Count` |

**Design rule: accept the least, return the most useful.** A method that only loops should take `IEnumerable<T>` so it works with lists, arrays, sets and LINQ queries; a method that needs `Count` and indexing takes `IReadOnlyList<T>` or `IList<T>`.

```csharp
decimal Total(IEnumerable<Claim> claims) => claims.Sum(c => c.Amount);   // works for any collection

Total(new List<Claim>());
Total(new Claim[0]);
Total(new HashSet<Claim>());
Total(claims.Where(c => c.Status == "Approved"));
```

**Namespaces:** generic interfaces and classes are in `System.Collections.Generic`; the non-generic originals (`IEnumerable`, `ICollection`, `IList`, `IDictionary`, `ArrayList`, `Hashtable`) are in `System.Collections`; thread-safe versions (`ConcurrentDictionary`, `ConcurrentQueue`) are in `System.Collections.Concurrent`.

## 2. IEnumerable and IEnumerator

`IEnumerable<T>` means "you can loop over me": it has one method, `GetEnumerator()`, which returns an `IEnumerator<T>`, a cursor that walks the items one at a time. `foreach` is just compiler shorthand for using that cursor.

```csharp
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<out T> : IDisposable, IEnumerator
{
    T Current { get; }      // item at the cursor
    bool MoveNext();        // advance; false when no more items
    void Reset();           // back to start (rarely supported)
}
```

**What `foreach` compiles to**

```csharp
var names = new List<string> { "Rohit", "Priya" };

foreach (var n in names)
    Console.WriteLine(n);

// is roughly:
using (IEnumerator<string> e = names.GetEnumerator())
{
    while (e.MoveNext())          // cursor starts BEFORE the first item
    {
        var n = e.Current;
        Console.WriteLine(n);
    }
}
```

|  | `IEnumerable<T>` | `IEnumerator<T>` |
| --- | --- | --- |
| Role | The collection: "can be enumerated" | The cursor: "where am I?" |
| Members | `GetEnumerator()` | `Current`, `MoveNext()`, `Reset()`, `Dispose()` |
| State | None | Holds position |
| Many loops at once | Yes: each `foreach` gets its own enumerator | No: one position |

**Rules:** modifying a `List<T>` while enumerating it throws `InvalidOperationException`; enumerators should be disposed (foreach does this); `foreach` actually only needs a public `GetEnumerator()` method (duck typing), but implementing the interface is what makes LINQ work.

## 3. Iterators and yield return

Writing an `IEnumerator` by hand is tedious. An **iterator** method uses `yield return` to produce items one at a time, and the compiler generates the enumerator class for you.

```csharp
IEnumerable<int> EvenNumbers(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;               // hand back one item, pause here
}

foreach (int n in EvenNumbers(10))
    Console.Write(n + " ");           // 0 2 4 6 8 10
```

**Lazy (deferred) execution:** nothing runs when you call `EvenNumbers(10)`. Code runs only as `foreach` (or LINQ) asks for the next item, and it resumes exactly where it paused.

```csharp
IEnumerable<string> ReadClaims()
{
    Console.WriteLine("start");
    yield return "C1";
    Console.WriteLine("after C1");
    yield return "C2";
    Console.WriteLine("end");
}

var seq = ReadClaims();               // prints nothing yet
foreach (var c in seq) Console.WriteLine(c);
// start, C1, after C1, C2, end

string first = ReadClaims().First();  // start, then stops after C1 is returned
```

**yield break** ends the sequence early:

```csharp
IEnumerable<string> ReadUntilBlank(TextReader r)
{
    while (true)
    {
        string? line = r.ReadLine();
        if (string.IsNullOrEmpty(line)) yield break;   // stop
        yield return line;
    }
}
```

**Infinite and streaming sequences** are easy because items are produced on demand:

```csharp
IEnumerable<long> Fibonacci()
{
    long a = 0, b = 1;
    while (true) { yield return a; (a, b) = (b, a + b); }
}
var firstTen = Fibonacci().Take(10).ToList();
```

**Rules:** the return type must be `IEnumerable<T>`, `IEnumerator<T>` (or non-generic versions) or `IAsyncEnumerable<T>` (with `await foreach`); no `ref`/`out` parameters; `yield return` can't be inside a `catch` block or a `try` that has a `catch`; argument checks inside an iterator run only on first `MoveNext`, so validate in a wrapper method if you want errors immediately. Each `foreach` re-runs the iterator from the start.

## 4. Custom Collections with IEnumerable, ICollection and IList

Wrap a `List<T>` inside your own class to add rules (validation, limits, events) while still working with `foreach`, LINQ and collection initializers. Implement only as much of the hierarchy as callers need.

| Implement | Callers get | You write |
| --- | --- | --- |
| `IEnumerable<T>` | `foreach`, LINQ | `GetEnumerator()` (2 methods) |
| `ICollection<T>` | + `Count`, `Add`, `Remove`, `Contains`, `Clear`, `CopyTo`, collection initializer | \~8 members |
| `IList<T>` | + indexer, `IndexOf`, `Insert`, `RemoveAt` | \~13 members |
| Inherit `Collection<T>` | All of IList\<T> with override hooks | Only the hooks you need |

### With IEnumerable\<T>: read-only, loopable

```csharp
public class Ward : IEnumerable<Patient>
{
    private readonly List<Patient> _beds = new();
    public int Capacity { get; }
    public Ward(int capacity) => Capacity = capacity;

    public void Admit(Patient p)
    {
        if (_beds.Count >= Capacity) throw new InvalidOperationException("Ward full");
        _beds.Add(p);
    }

    public IEnumerator<Patient> GetEnumerator() => _beds.GetEnumerator();   // reuse the list's
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();             // non-generic, explicit
}

var icu = new Ward(2);
icu.Admit(new Patient { Name = "Rohit" });
foreach (var p in icu) Console.WriteLine(p.Name);
var names = icu.Select(p => p.Name);         // LINQ works
```

You could also write `GetEnumerator` with `yield return` (e.g. to skip discharged patients).

### With ICollection\<T>: add/remove with rules

```csharp
public class UniqueEmailList : ICollection<string>
{
    private readonly List<string> _items = new();

    public int Count => _items.Count;
    public bool IsReadOnly => false;

    public void Add(string email)
    {
        if (!email.Contains('@')) throw new ArgumentException("Invalid email");
        if (!Contains(email)) _items.Add(email.ToLower());
    }
    public bool Remove(string email) => _items.Remove(email.ToLower());
    public bool Contains(string email) => _items.Contains(email.ToLower());
    public void Clear() => _items.Clear();
    public void CopyTo(string[] array, int index) => _items.CopyTo(array, index);
    public IEnumerator<string> GetEnumerator() => _items.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

var emails = new UniqueEmailList { "a@x.com", "A@X.com" };   // initializer calls Add: 1 item
```

### With IList\<T>: plus index access

```csharp
public class LimitedList<T> : IList<T>
{
    private readonly List<T> _items = new();
    private readonly int _max;
    public LimitedList(int max) => _max = max;

    public T this[int index] { get => _items[index]; set => _items[index] = value; }
    public int IndexOf(T item) => _items.IndexOf(item);
    public void Insert(int index, T item) { Guard(); _items.Insert(index, item); }
    public void RemoveAt(int index) => _items.RemoveAt(index);

    public void Add(T item) { Guard(); _items.Add(item); }
    public int Count => _items.Count;
    public bool IsReadOnly => false;
    public bool Contains(T item) => _items.Contains(item);
    public bool Remove(T item) => _items.Remove(item);
    public void Clear() => _items.Clear();
    public void CopyTo(T[] array, int i) => _items.CopyTo(array, i);
    public IEnumerator<T> GetEnumerator() => _items.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    private void Guard() { if (_items.Count >= _max) throw new InvalidOperationException("Limit reached"); }
}
```

**Shortcut:** inherit `System.Collections.ObjectModel.Collection<T>` and override `InsertItem`, `RemoveItem`, `SetItem`, `ClearItems` to add rules with far less code. `ReadOnlyCollection<T>` and `ObservableCollection<T>` (raises change events for UI binding) come from the same namespace. Don't inherit `List<T>` itself: its methods aren't virtual, so your rules can be bypassed.

## 5. IEquatable\<T>

`IEquatable<T>` adds a strongly typed `Equals(T other)` method so collections can tell whether two objects are "the same" by **value** instead of by reference, with no casting and no boxing for structs. `List.Contains/IndexOf/Remove`, `Dictionary` keys and `HashSet` all use it.

```csharp
public class Policy : IEquatable<Policy>
{
    public string Number { get; init; } = "";
    public string Holder { get; init; } = "";

    public bool Equals(Policy? other)                     // IEquatable<Policy>
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return Number == other.Number;                    // identity = policy number
    }

    public override bool Equals(object? obj) => Equals(obj as Policy);   // keep object.Equals consistent
    public override int GetHashCode() => Number.GetHashCode();          // MUST match Equals

    public static bool operator ==(Policy? a, Policy? b) => Equals(a, b);
    public static bool operator !=(Policy? a, Policy? b) => !Equals(a, b);
}

var list = new List<Policy> { new() { Number = "P1", Holder = "Rohit" } };
var probe = new Policy { Number = "P1", Holder = "R. Rambade" };

list.Contains(probe);              // true: same policy number
list.IndexOf(probe);               // 0
int n = new HashSet<Policy> { probe, list[0] }.Count;   // 1
```

**Without IEquatable** (and without overriding Equals), `Contains` returns `false` because it compares references.

**Checklist when implementing:** implement `Equals(T)`; override `Equals(object)` to call it; override `GetHashCode` using the same fields; optionally overload `==`/`!=`; handle `null`. Records (`public record Policy(string Number, string Holder);`) generate all of this automatically, comparing every property.

**Custom equality without changing the class:** pass an `IEqualityComparer<T>` to the collection, e.g. `new HashSet<string>(StringComparer.OrdinalIgnoreCase)` or your own comparer class implementing `Equals(x, y)` and `GetHashCode(obj)`.

## 6. IComparable\<T> and IComparer\<T>

Both tell sorting code how to order objects. **`IComparable<T>`** is implemented *by the class itself* and defines its one natural order. **`IComparer<T>`** is a *separate class* that defines an extra order you choose at sort time, so you can have as many as you like.

Both return an `int`: **negative** = first comes before second, **0** = equal, **positive** = first comes after.

### IComparable\<T>: the default order

```csharp
public class Patient : IComparable<Patient>
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public int Priority { get; init; }       // 1 = most urgent

    public int CompareTo(Patient? other)
    {
        if (other is null) return 1;             // non-null sorts after null
        return Age.CompareTo(other.Age);         // natural order: by age
    }
    public override string ToString() => $"{Name} ({Age}, P{Priority})";
}

var ward = new List<Patient>
{
    new() { Name = "Rohit", Age = 30, Priority = 2 },
    new() { Name = "Amit",  Age = 8,  Priority = 1 },
    new() { Name = "Priya", Age = 62, Priority = 1 }
};
ward.Sort();                    // uses CompareTo: Amit, Rohit, Priya
// Without IComparable, Sort() throws InvalidOperationException
```

### IComparer\<T>, part 1: separate orders

```csharp
public class ByName : IComparer<Patient>
{
    public int Compare(Patient? x, Patient? y) =>
        string.Compare(x?.Name, y?.Name, StringComparison.OrdinalIgnoreCase);
}

public class ByPriorityThenAge : IComparer<Patient>
{
    public int Compare(Patient? x, Patient? y)
    {
        if (x is null || y is null) return Comparer<Patient?>.Default.Compare(x, y);
        int p = x.Priority.CompareTo(y.Priority);
        return p != 0 ? p : y.Age.CompareTo(x.Age);    // tie-break: older first
    }
}

ward.Sort(new ByName());               // Amit, Priya, Rohit
ward.Sort(new ByPriorityThenAge());    // Priya (P1, 62), Amit (P1, 8), Rohit (P2)
```

### IComparer\<T>, part 2: more ways and more places

```csharp
ward.Sort((a, b) => b.Age.CompareTo(a.Age));                        // Comparison<T> lambda: oldest first
var cmp = Comparer<Patient>.Create((a, b) => a.Name.Length.CompareTo(b.Name.Length));  // lambda -> IComparer
ward.Sort(cmp);

var sortedSet  = new SortedSet<Patient>(new ByName());               // collections that keep order
var sortedDict = new SortedDictionary<string, int>(StringComparer.OrdinalIgnoreCase);
Array.Sort(patientsArray, new ByPriorityThenAge());
int idx = ward.BinarySearch(target, new ByName());                   // same comparer used for sorting

var linq = ward.OrderBy(p => p.Priority).ThenByDescending(p => p.Age);   // LINQ alternative
// for a reversed comparer, swap x and y inside Compare
```

|  | `IComparable<T>` | `IComparer<T>` |
| --- | --- | --- |
| Implemented by | The class being sorted | A separate comparer class |
| Method | `CompareTo(T other)` | `Compare(T x, T y)` |
| How many orders | One natural order | As many as you write |
| Used by | `Sort()`, `SortedSet<T>()`, `OrderBy(x => x)` with no comparer | `Sort(comparer)`, `new SortedSet<T>(comparer)`, `OrderBy(key, comparer)` |
| Need to edit the class? | Yes | No |
| Namespace | `System` | `System.Collections.Generic` |

**Tip:** compare with the built-in `CompareTo` of the field (`Age.CompareTo(...)`) rather than subtracting (`a.Age - b.Age`), which can overflow.

## 7. Covariance and Contravariance

Variance decides whether a generic type built on a **derived** type can be used where one built on a **base** type is expected (or the reverse). It applies only to generic **interfaces and delegates**, only to reference types, and only when the type parameter is marked `out` or `in`.

Example classes: `class Animal { }` and `class Dog : Animal { }`. Every Dog is an Animal.

### Covariance (`out T`): derived → base, for things that only **produce** T

```csharp
IEnumerable<Dog> dogs = new List<Dog>();
IEnumerable<Animal> animals = dogs;        // OK: IEnumerable<out T> is covariant

Func<Dog> makeDog = () => new Dog();
Func<Animal> makeAnimal = makeDog;         // OK: Func<out TResult>

// List<Animal> list = new List<Dog>();    // ERROR: List<T> is invariant
```

Why safe: anything you **read out** of a sequence of dogs is certainly an animal. Why `List<T>` can't be covariant: if it were, you could `Add(new Cat())` into a list of dogs.

### Contravariance (`in T`): base → derived, for things that only **consume** T

```csharp
Action<Animal> feedAnimal = a => Console.WriteLine("Feeding");
Action<Dog> feedDog = feedAnimal;          // OK: Action<in T> is contravariant

IComparer<Animal> byWeight = Comparer<Animal>.Create((a, b) => 0);
IComparer<Dog> dogComparer = byWeight;     // OK: IComparer<in T>
var dogList = new List<Dog>();
dogList.Sort(byWeight);                    // an animal comparer can sort dogs
```

Why safe: something that can handle **any animal** can certainly handle a dog you pass **in**.

### Declaring your own variant interfaces

```csharp
public interface IProducer<out T> { T Produce(); }          // T only in output positions
public interface IConsumer<in T>  { void Consume(T item); } // T only in input positions

// public interface IBad<out T> { void Take(T x); }  // ERROR: 'out' T used as an input
```

|  | Covariance | Contravariance | Invariance |
| --- | --- | --- | --- |
| Keyword | `out T` | `in T` | none |
| Direction | `I<Dog>` → `I<Animal>` | `I<Animal>` → `I<Dog>` | Exact type only |
| T used as | Return values only | Parameters only | Both |
| Built-in examples | `IEnumerable<T>`, `IReadOnlyList<T>`, `Func<T>` result, `IEnumerator<T>` | `Action<T>`, `IComparer<T>`, `IEqualityComparer<T>`, `Predicate<T>`, `Func<T, …>` inputs | `List<T>`, `IList<T>`, `Dictionary<K,V>` |

**Memory hook:** **out** = things coming **out** can be treated as a more general type; **in** = things going **in** can be handled by something more general. Arrays are covariant too (`Animal[] a = new Dog[1];`), but unsafely: storing a `Cat` throws `ArrayTypeMismatchException` at run time.

## 8. Cheat Sheet, Checklist and Interview Questions

```csharp
// Iterator
IEnumerable<int> Evens(int max) { for (int i = 0; i <= max; i += 2) yield return i; }

// Custom collection
class Bag<T> : IEnumerable<T>
{
    private readonly List<T> _items = new();
    public void Add(T x) => _items.Add(x);
    public IEnumerator<T> GetEnumerator() => _items.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// Equality and ordering
class Item : IEquatable<Item>, IComparable<Item>
{
    public int Id { get; init; }
    public bool Equals(Item? o) => o is not null && o.Id == Id;
    public override bool Equals(object? o) => Equals(o as Item);
    public override int GetHashCode() => Id;
    public int CompareTo(Item? o) => o is null ? 1 : Id.CompareTo(o.Id);
}
class ByIdDesc : IComparer<Item> { public int Compare(Item? x, Item? y) => (y?.Id ?? 0).CompareTo(x?.Id ?? 0); }

// Variance
IEnumerable<object> objs = new List<string>();   // covariance (out)
Action<string> act = (Action<object>)(o => { });   // contravariance (in)
```

**Checklist**

- [ ] Draw the IEnumerable → ICollection → IList/ISet/IDictionary hierarchy
- [ ] Rewrite a foreach loop using GetEnumerator/MoveNext/Current
- [ ] Write an iterator with yield return and yield break; explain lazy execution
- [ ] Build a custom collection with IEnumerable, then ICollection, then IList (or Collection\<T>)
- [ ] Implement IEquatable\<T> with matching GetHashCode and show Contains working
- [ ] Sort with IComparable (natural order) and two IComparer classes
- [ ] Explain covariance and contravariance with Animal/Dog examples

**Interview questions (short answers)**

1. **What is IEnumerable\<T>?** The interface that makes a type loopable with foreach and usable with LINQ; it has `GetEnumerator()`.
2. **IEnumerable vs IEnumerator?** IEnumerable is the collection; IEnumerator is the cursor with `Current` and `MoveNext()`.
3. **IEnumerable vs ICollection vs IList?** Loop only; + Count/Add/Remove; + index access/Insert.
4. **What does yield return do?** Returns one item and pauses; the compiler generates the enumerator and execution is lazy.
5. **What is deferred execution?** The iterator's code runs only when items are requested, not when the method is called.
6. **yield break?** Ends the iterator early.
7. **Why implement IEquatable\<T>?** Typed, boxing-free value equality used by List, HashSet and Dictionary.
8. **What must you override along with Equals?** GetHashCode, using the same fields.
9. **IComparable vs IComparer?** IComparable: the class's own single natural order (`CompareTo`). IComparer: separate classes for any number of orders (`Compare`).
10. **What does CompareTo return?** Negative, zero or positive for less, equal, greater.
11. **What is covariance?** Using a more derived type argument where a base one is expected, for output-only (`out`) type parameters: `IEnumerable<Dog>` as `IEnumerable<Animal>`.
12. **What is contravariance?** Using a more general type argument where a derived one is expected, for input-only (`in`) parameters: `Action<Animal>` as `Action<Dog>`.
13. **Why isn't List\<T> covariant?** It accepts T as input (Add), so covariance would allow adding the wrong type.
14. **Why not inherit from List\<T> for a custom collection?** Its methods aren't virtual; use Collection\<T> or wrap a List.
