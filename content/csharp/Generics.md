# Generics

Write a class or method once and use it with any type, with full type safety and no boxing: generic classes, multiple type parameters, constraints and generic methods.

## 1. Generic Classes

A generic class takes a **type parameter** (usually `T`) that the caller fills in: `Box<int>`, `Box<string>`. You write the logic once, and the compiler checks types for each use with no casting or boxing.

**The problem generics solve**

```csharp
// Option A: one class per type: duplicated code
class IntStore    { public int Value; }
class StringStore { public string Value; }

// Option B: use object: no type safety, boxing, casts everywhere
class ObjectStore { public object Value; }
var s = new ObjectStore { Value = 42 };     // boxing
string oops = (string)s.Value;              // compiles, crashes at run time
```

**The generic way**

```csharp
public class Store<T>                       // T = type parameter
{
    private readonly List<T> _items = new();

    public void Add(T item) => _items.Add(item);
    public T Get(int index) => _items[index];
    public int Count => _items.Count;
    public T? FirstOrDefault() => _items.Count > 0 ? _items[0] : default;   // default(T)
}

var numbers = new Store<int>();             // T = int
numbers.Add(10);
int n = numbers.Get(0);                     // no cast, no boxing
// numbers.Add("ten");                      // compile-time ERROR: type safety

var names = new Store<string>();            // T = string
names.Add("Rohit");
```

**Vocabulary**

| Term | Example | Meaning |
| --- | --- | --- |
| Generic type definition | `Store<T>` | The template |
| Type parameter | `T` | Placeholder in the definition |
| Type argument | `int` in `Store<int>` | Actual type supplied by the caller |
| Constructed (closed) type | `Store<int>` | A real, usable type |

**Benefits:** type safety at compile time, reuse of one implementation, no boxing for value types (faster), and IntelliSense knows the exact type.

**You already use them:** `List<T>`, `Dictionary<TKey, TValue>`, `Nullable<T>`, `Task<T>`, `IEnumerable<T>`.

**`default(T)` / `default`:** gives `0` for numbers, `false` for bool, `null` for reference types: the safe "empty" value for any `T`. Generic interfaces, structs and records work the same way: `interface IRepository<T>`, `struct Pair<T>`.

## 2. Multiple Generic Parameters

A generic type can take several type parameters, separated by commas. Name them descriptively with a `T` prefix when there's more than one: `TKey`, `TValue`, `TResult`.

```csharp
public class Pair<TFirst, TSecond>
{
    public TFirst First { get; }
    public TSecond Second { get; }
    public Pair(TFirst first, TSecond second) => (First, Second) = (first, second);
    public override string ToString() => $"({First}, {Second})";
}

var p1 = new Pair<string, int>("Rohit", 30);
var p2 = new Pair<int, DateTime>(101, DateTime.Today);

public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();
    public void Set(TKey key, TValue value) => _data[key] = value;
    public bool TryGet(TKey key, out TValue? value) => _data.TryGetValue(key, out value);
}

var claimCache = new Cache<string, decimal>();
claimCache.Set("CLM-01", 25000m);
```

**Points**

- Each parameter is independent: they can be the same type (`Pair<int, int>`) or different.
- Overloading by number of type parameters (arity) is allowed: `Pair<T>` and `Pair<T1, T2>` are different types (like `Action<T>` and `Action<T1, T2>`).
- Built-in examples: `Dictionary<TKey, TValue>`, `KeyValuePair<TKey, TValue>`, `Func<T, TResult>`, `Tuple<T1, T2, T3>`.
- For quick throwaway pairs, value tuples are simpler: `(string Name, int Age) p = ("Rohit", 30);`

## 3. Generic Constraints

A constraint (`where T : ...`) limits which types can be used for `T`. In return, the compiler lets you use the members those types are guaranteed to have, such as calling `CompareTo` or `new T()`.

**Without a constraint you can only treat `T` as `object`:**

```csharp
public class Ranker<T>
{
    public T Max(T a, T b) => a.CompareTo(b) > 0 ? a : b;   // ERROR: T might not have CompareTo
}
```

**With a constraint:**

```csharp
public class Ranker<T> where T : IComparable<T>
{
    public T Max(T a, T b) => a.CompareTo(b) > 0 ? a : b;   // OK
}

new Ranker<int>().Max(3, 9);          // 9
new Ranker<string>().Max("a", "z");   // "z"
// new Ranker<object>();              // ERROR: object isn't IComparable<object>
```

**All constraint kinds**

| Constraint | Means T must be | Lets you |
| --- | --- | --- |
| `where T : class` | A reference type | Use `null`, `as` |
| `where T : class?` | A nullable or non-nullable reference type | Same, in nullable contexts |
| `where T : struct` | A non-nullable value type | Use `T?` as `Nullable<T>` |
| `where T : new()` | Has a public parameterless constructor | Write `new T()` |
| `where T : BaseClass` | That class or a subclass | Use the base class's members |
| `where T : IInterface` | Implements the interface | Call interface members |
| `where T : U` | Same as or derived from another type parameter U | Treat T as U |
| `where T : notnull` | Any non-nullable type | Safe as a dictionary key |
| `where T : unmanaged` | A value type with no references (int, double, simple structs) | Pointers, `sizeof`, low-level code |
| `where T : Enum` / `Delegate` | An enum / delegate type | Enum/delegate APIs |

**Combining constraints**

```csharp
public class Repository<TEntity, TKey>
    where TEntity : class, IEntity<TKey>, new()    // order: class/struct first, new() last
    where TKey : notnull
{
    private readonly Dictionary<TKey, TEntity> _db = new();

    public TEntity Create(TKey id)
    {
        var e = new TEntity { Id = id };           // allowed because of new()
        _db[id] = e;
        return e;
    }
}

public interface IEntity<TKey> { TKey Id { get; set; } }
public class Claim : IEntity<string> { public string Id { get; set; } = ""; }

var repo = new Repository<Claim, string>();
repo.Create("CLM-01");
```

**Ordering rule:** `class` / `struct` / `notnull` / `unmanaged` / a base class come **first**, then interfaces, and `new()` **last**. Each type parameter gets its own `where` clause.

## 4. Generic Methods

A generic method has its own type parameter, written after the method name: `Swap<T>(...)`. It can live in a normal (non-generic) class, and the compiler usually infers `T` from the arguments.

```csharp
public static class Utils
{
    public static void Swap<T>(ref T a, ref T b)
    {
        T temp = a;
        a = b;
        b = temp;
    }

    public static T Max<T>(T a, T b) where T : IComparable<T>      // with a constraint
        => a.CompareTo(b) >= 0 ? a : b;

    public static void PrintAll<T>(IEnumerable<T> items)
    {
        foreach (var item in items) Console.WriteLine(item);
    }

    public static TResult[] ConvertAll<TInput, TResult>(TInput[] input, Func<TInput, TResult> convert)
    {
        var result = new TResult[input.Length];
        for (int i = 0; i < input.Length; i++) result[i] = convert(input[i]);
        return result;
    }
}

int x = 1, y = 2;
Utils.Swap(ref x, ref y);                 // T inferred as int: x = 2, y = 1
Utils.Swap<int>(ref x, ref y);            // explicit type argument also fine

string longer = Utils.Max("apple", "mango");   // "mango"
Utils.PrintAll(new[] { 1.5, 2.5 });
string[] labels = Utils.ConvertAll(new[] { 1, 2, 3 }, n => $"Item {n}");
```

**When inference fails** you must write the type: `Utils.Create<Claim>()` (no argument to infer from) or when you want a different type than the arguments suggest: `Max<long>(3, 4)`.

**Generic method in a generic class:** the method can add its own parameters beyond the class's `T`:

```csharp
public class Store<T>
{
    private readonly List<T> _items = new();
    public List<TOut> Map<TOut>(Func<T, TOut> f) => _items.Select(f).ToList();
}
```

**Built-in generic methods you use daily:** LINQ (`Where<T>`, `Select<T, TResult>`), `Array.Sort<T>`, `Enumerable.Empty<T>()`, `JsonSerializer.Deserialize<T>(json)`.

## 5. Important Points, Cheat Sheet, Checklist and Interview Questions

**Points to remember**

1. Generics give type safety at compile time, code reuse and no boxing for value types.
2. Type parameters are named `T` or `T` + a meaningful word (`TKey`, `TEntity`).
3. `default(T)` gives the zero/null value for any `T`.
4. Without constraints, `T` supports only `object` members; constraints unlock more.
5. Constraint order: `class`/`struct`/base class first, interfaces next, `new()` last.
6. Generic methods can be in non-generic classes; type arguments are usually inferred.
7. Each closed type (`Store<int>`, `Store<string>`) is a separate type; static fields are **not** shared between them.
8. Prefer `List<T>` and `Dictionary<K,V>` over `ArrayList` and `Hashtable`.
9. Operators like `+` don't work on a plain `T`; .NET 7+ generic math (`where T : INumber<T>`) solves this.

**Cheat sheet**

```csharp
class Box<T> { public T Value { get; set; } }                         // generic class
class Pair<TKey, TValue> { public TKey Key; public TValue Val; }      // multiple params
class Repo<T> where T : class, IEntity, new() { T Make() => new T(); }   // constraints
static T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) > 0 ? a : b;  // generic method
static T Sum<T>(IEnumerable<T> xs) where T : INumber<T>                // generic math (.NET 7+)
{ T total = T.Zero; foreach (var x in xs) total += x; return total; }

var b = new Box<int> { Value = 5 };
int m = Max(3, 9);                    // T inferred
T? empty = default;                   // inside generic code
```

**Checklist**

- [ ] Explain why `object`-based code is worse than generics (safety + boxing)
- [ ] Write a generic `Store<T>` class
- [ ] Write a class with two type parameters
- [ ] Name all constraint kinds and their ordering rule
- [ ] Use `new()`, an interface and `class` constraints together
- [ ] Write `Swap<T>` and `Max<T>` generic methods
- [ ] Explain type inference and when it fails

**Interview questions (short answers)**

1. **What are generics?** Classes, interfaces and methods with type parameters, so one implementation works type-safely with many types.
2. **Advantages of generics?** Compile-time type safety, reuse, better performance (no boxing/casting).
3. **What is a type parameter vs type argument?** The placeholder `T` in the definition vs the real type supplied (`int`).
4. **What is `default(T)`?** The default value of T: 0, false or null depending on the type.
5. **What are generic constraints?** `where` clauses that restrict type arguments and enable using their members.
6. **What does `where T : new()` do?** Requires a public parameterless constructor so you can write `new T()`.
7. **`where T : class` vs `where T : struct`?** Reference types only vs non-nullable value types only.
8. **Can a generic method exist in a non-generic class?** Yes.
9. **What is type inference?** The compiler working out `T` from the method arguments.
10. **Are static fields shared across `Store<int>` and `Store<string>`?** No; each closed type has its own.
11. **`List<T>` vs `ArrayList`?** `List<T>` is type-safe and avoids boxing; `ArrayList` stores `object`.
12. **Can you use `+` on a generic `T`?** Not directly; use generic math interfaces (`INumber<T>`) in .NET 7+.
