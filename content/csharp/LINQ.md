# LINQ

Language Integrated Query: one consistent way to filter, sort, shape, group, join and summarise data from lists, arrays, databases, XML and more. Covers every LINQ topic in your course list plus the rest of the standard operators.

## 0. Sample Data Used in Every Example

All outputs on this tab come from these two lists, so you can check each result by hand.

```csharp
public record Employee(int Id, string Name, string Dept, decimal Salary, int Age, string City);
public record Department(string Name, string Building);

var employees = new List<Employee>
{
    new(1, "Rohit", "IT",      60000, 30, "Pune"),
    new(2, "Priya", "HR",      45000, 41, "Mumbai"),
    new(3, "Amit",  "IT",      75000, 28, "Pune"),
    new(4, "Neha",  "Finance", 52000, 35, "Delhi"),
    new(5, "Kabir", "IT",      48000, 24, "Mumbai"),
    new(6, "Meera", "HR",      58000, 39, "Pune")
};

var departments = new List<Department>
{
    new("IT", "Tech Park"), new("HR", "HQ"), new("Finance", "HQ"), new("Legal", "Annex")   // Legal has no employees
};

int[] numbers = { 5, 3, 8, 1, 9, 2 };
```

Add `using System.Linq;` (included automatically by implicit usings in .NET 6+).

## 1. LINQ Basics

LINQ adds query operators (`Where`, `Select`, `OrderBy`, `GroupBy`…) as **extension methods on `IEnumerable<T>`** (and `IQueryable<T>`), so the same syntax works on arrays, lists, dictionaries, EF Core database tables and XML. Queries are strongly typed and checked by the compiler.

### Two syntaxes, same result

```csharp
// Query syntax: SQL-like, compiled into method calls
var q1 = from e in employees
         where e.Salary > 50000
         orderby e.Name
         select e.Name;

// Method syntax: chains of extension methods + lambdas (more common, covers every operator)
var q2 = employees
         .Where(e => e.Salary > 50000)
         .OrderBy(e => e.Name)
         .Select(e => e.Name);

// both: Amit, Meera, Neha, Rohit
```

|  | Query syntax | Method syntax |
| --- | --- | --- |
| Looks like | SQL (`from … where … select`) | Method chain with lambdas |
| Must start/end with | `from` … `select` / `group` | Anything |
| All operators available | No (`Count`, `First`, `Take`, `Distinct` need methods) | Yes |
| Easier for | Joins, `let`, multiple `from` | Everything else |

You can mix them: `(from e in employees where e.Dept == "IT" select e).Count()`.

### Deferred execution

&#91;embedded content: Deferred execution in a LINQ pipeline\]

Most operators return a query object, not results. The data is read only when you loop or call an immediate operator such as `ToList`, `Count`, `First` or `Sum`, and it's read again every time you do.

```csharp
var highPaid = employees.Where(e => e.Salary > 70000);   // nothing runs yet
employees.Add(new(7, "Isha", "IT", 90000, 33, "Delhi"));
Console.WriteLine(highPaid.Count());                      // 2: Amit AND Isha (sees the new item)

var snapshot = employees.Where(e => e.Salary > 70000).ToList();   // runs now, frozen copy
```

(Later examples use the original six employees.)

**The building blocks:** lambdas (`e => e.Salary > 50000`), extension methods, `var`/anonymous types for results, and `Func<T, TResult>` delegates. All are covered on earlier tabs.

## 2. Filtering and Projection: Where, OfType, Select, SelectMany

**`Where`** keeps the items that match a condition; **`Select`** transforms each item into something else (a property, a calculation, a new object).

### Where

```csharp
var itStaff  = employees.Where(e => e.Dept == "IT");                       // Rohit, Amit, Kabir
var puneHigh = employees.Where(e => e.City == "Pune" && e.Salary > 55000); // Rohit, Amit, Meera
var everyOther = employees.Where((e, index) => index % 2 == 0);            // index overload: Rohit, Amit, Kabir

// query syntax
var q = from e in employees where e.Age < 30 select e;                      // Amit, Kabir
```

### OfType

Filters by **type** (useful with mixed or non-generic collections):

```csharp
object[] mixed = { 1, "two", 3.0, 4, "five" };
var ints = mixed.OfType<int>();          // 1, 4
var strs = mixed.OfType<string>();       // two, five
```

### Select

```csharp
var names   = employees.Select(e => e.Name);                         // Rohit, Priya, ...
var raised  = employees.Select(e => e.Salary * 1.10m);               // calculated values
var summary = employees.Select(e => new { e.Name, Annual = e.Salary * 12 });   // anonymous type
var dtos    = employees.Select(e => new EmployeeDto(e.Id, e.Name));  // map to another class
var ranked  = employees.Select((e, i) => $"{i + 1}. {e.Name}");      // with index: "1. Rohit"

// query syntax with let (an intermediate variable)
var q2 = from e in employees
         let annual = e.Salary * 12
         where annual > 600000
         select new { e.Name, annual };                               // Rohit, Amit, Neha, Meera
```

### SelectMany

Flattens a "list of lists" into one sequence.

```csharp
var teams = new[]
{
    new { Team = "A", Members = new[] { "Rohit", "Amit" } },
    new { Team = "B", Members = new[] { "Priya" } }
};

var nested = teams.Select(t => t.Members);          // IEnumerable<string[]>: 2 arrays
var flat   = teams.SelectMany(t => t.Members);      // Rohit, Amit, Priya
var pairs  = teams.SelectMany(t => t.Members, (t, m) => $"{t.Team}:{m}");   // A:Rohit, A:Amit, B:Priya

// query syntax: two from clauses
var q3 = from t in teams from m in t.Members select m;
```

|  | `Select` | `SelectMany` |
| --- | --- | --- |
| One input item gives | Exactly one output item | Zero or more output items |
| Result for nested lists | Sequence of sequences | One flat sequence |
| SQL feel | Choosing columns | Like a CROSS APPLY / flatten |

## 3. Ordering: OrderBy, OrderByDescending, ThenBy, Reverse

`OrderBy` sorts ascending by a key; `OrderByDescending` sorts high to low; `ThenBy` / `ThenByDescending` break ties. All return a **new sorted sequence** and leave the source untouched; LINQ sorting is **stable** (equal keys keep their original order).

```csharp
var bySalary = employees.OrderBy(e => e.Salary);
// Priya 45000, Kabir 48000, Neha 52000, Meera 58000, Rohit 60000, Amit 75000

var oldestFirst = employees.OrderByDescending(e => e.Age);
// Priya 41, Meera 39, Neha 35, Rohit 30, Amit 28, Kabir 24

var byDeptThenName = employees.OrderBy(e => e.Dept).ThenBy(e => e.Name);
// Finance: Neha | HR: Meera, Priya | IT: Amit, Kabir, Rohit

var byDeptThenPayDesc = employees.OrderBy(e => e.Dept).ThenByDescending(e => e.Salary);
// Neha | Meera, Priya | Amit, Rohit, Kabir

var reversed = employees.Select(e => e.Name).Reverse();   // Meera, Kabir, Neha, Amit, Priya, Rohit

var sortedNums = numbers.Order();                 // .NET 7+: sort by the value itself: 1 2 3 5 8 9
var caseless = names.OrderBy(n => n, StringComparer.OrdinalIgnoreCase);   // custom comparer
```

**Query syntax:**

```csharp
var q = from e in employees
        orderby e.Dept, e.Salary descending
        select e;
```

**Rules and gotchas**

- Don't call `OrderBy` twice to sort by two keys: the second call re-sorts everything. Use `ThenBy`.
- `ThenBy` only exists after `OrderBy`/`OrderByDescending` (it works on `IOrderedEnumerable<T>`).
- `List<T>.Sort()` sorts **in place** and is not stable; LINQ `OrderBy` makes a new sequence and is stable.
- Sorting is O(n log n) and needs all items, so it runs only when the query executes.

## 4. Element Operators: First, Last, ElementAt, Single (and OrDefault)

These return **one element**, and execute immediately. The plain versions **throw** when there's no suitable element; the `OrDefault` versions return `default` (`null` for classes, `0` for numbers) instead.

### First and FirstOrDefault

```csharp
Employee first    = employees.First();                          // Rohit
Employee firstIT  = employees.First(e => e.Dept == "IT");       // Rohit
Employee? legal   = employees.FirstOrDefault(e => e.Dept == "Legal");   // null
// employees.First(e => e.Dept == "Legal");                    // InvalidOperationException: no match

int firstBig = numbers.FirstOrDefault(n => n > 100);           // 0
int withFallback = numbers.FirstOrDefault(n => n > 100, -1);   // -1 (custom default, .NET 6+)
```

### Last and LastOrDefault

```csharp
Employee lastIT = employees.Last(e => e.Dept == "IT");          // Kabir
Employee? lastLegal = employees.LastOrDefault(e => e.Dept == "Legal");  // null
Employee oldestByAge = employees.OrderBy(e => e.Age).Last();    // Priya (oldest)
```

### ElementAt and ElementAtOrDefault

```csharp
Employee third = employees.ElementAt(2);              // Amit (zero-based)
Employee? none = employees.ElementAtOrDefault(10);    // null
// employees.ElementAt(10);                           // ArgumentOutOfRangeException
Employee lastOne = employees.ElementAt(^1);           // Meera (Index from end, .NET 6+)
```

### Single and SingleOrDefault

Use when there **must be exactly one** match (like looking up by a unique ID). They also throw if there's **more than one**.

```csharp
Employee amit = employees.Single(e => e.Id == 3);                   // Amit
Employee? missing = employees.SingleOrDefault(e => e.Id == 99);     // null
// employees.Single(e => e.Dept == "IT");                           // throws: 3 matches
// employees.SingleOrDefault(e => e.Dept == "IT");                  // ALSO throws: more than one
// employees.Single(e => e.Id == 99);                               // throws: no match
```

| Method | 0 matches | 1 match | 2+ matches |
| --- | --- | --- | --- |
| `First` | Throws | Returns it | Returns the first |
| `FirstOrDefault` | `default` | Returns it | Returns the first |
| `Last` | Throws | Returns it | Returns the last |
| `LastOrDefault` | `default` | Returns it | Returns the last |
| `Single` | Throws | Returns it | **Throws** |
| `SingleOrDefault` | `default` | Returns it | **Throws** |
| `ElementAt(i)` | Throws if index out of range |  |  |
| `ElementAtOrDefault(i)` | `default` if index out of range |  |  |

**Which to use:** `FirstOrDefault` for "find one if it exists"; `Single` when duplicates would be a data bug (IDs, emails); `First`/`Last` only when the sequence is guaranteed non-empty. Always null-check an `OrDefault` result before using it. With EF Core, `Single` makes the database fetch up to 2 rows to check uniqueness.

## 5. Aggregates: Min, Max, Count, Sum, Average (and more)

Aggregate operators reduce a sequence to **one value** and execute immediately.

```csharp
int count       = employees.Count();                          // 6
int itCount     = employees.Count(e => e.Dept == "IT");       // 3
long big        = employees.LongCount();                      // for huge sequences

decimal total   = employees.Sum(e => e.Salary);               // 338000
decimal avg     = employees.Average(e => e.Salary);           // 56333.33...
decimal lowest  = employees.Min(e => e.Salary);               // 45000
decimal highest = employees.Max(e => e.Salary);               // 75000
double avgAge   = employees.Average(e => e.Age);              // 32.83...

int maxNum = numbers.Max();                                   // 9 (no selector for simple values)

// MinBy / MaxBy (.NET 6+): return the ELEMENT, not the value
Employee lowestPaid = employees.MinBy(e => e.Salary)!;        // Priya
Employee oldest     = employees.MaxBy(e => e.Age)!;           // Priya

// Aggregate: your own accumulator
string all = employees.Select(e => e.Name).Aggregate((a, b) => $"{a}, {b}");
// "Rohit, Priya, Amit, Neha, Kabir, Meera"   (string.Join(", ", ...) is simpler here)
int product = numbers.Aggregate(1, (acc, n) => acc * n);      // 2160 (seed = 1)
```

| Method | Returns | On an empty sequence |
| --- | --- | --- |
| `Count()` / `Count(pred)` | `int` | 0 |
| `Sum(selector)` | Same numeric type | 0 |
| `Average(selector)` | `double` (or `decimal` for decimal input) | **Throws** `InvalidOperationException` |
| `Min` / `Max` | The value | **Throws** for value types; `null` for nullable/reference types |
| `MinBy` / `MaxBy` | The element | `null` / default |
| `Aggregate(seed, func)` | Seed type | The seed |

**Tips:** guard `Average`/`Min`/`Max` with `Any()` or use `DefaultIfEmpty()` (e.g. `employees.Where(...).Select(e => e.Salary).DefaultIfEmpty().Max()`); for `List<T>` or arrays, the `.Count` / `.Length` property is cheaper than `Count()`; `Sum` on `int` can overflow, so select `long` or `decimal` for big totals.

## 6. Quantifiers and Partitioning: Any, All, Contains, Take, Skip, Chunk

### Quantifiers: return bool

```csharp
bool anyHigh   = employees.Any(e => e.Salary > 70000);     // true (Amit)
bool hasItems  = employees.Any();                          // true: list not empty (prefer over Count() > 0)
bool allAdults = employees.All(e => e.Age >= 18);          // true
bool allPune   = employees.All(e => e.City == "Pune");     // false
bool has8      = numbers.Contains(8);                      // true
bool same      = numbers.SequenceEqual(new[] { 5, 3, 8, 1, 9, 2 });   // true: same items, same order
```

`Any` stops at the first match; `All` stops at the first failure. `All` on an empty sequence is `true`.

### Partitioning: take or skip part of a sequence

```csharp
var firstTwo = employees.Take(2);                       // Rohit, Priya
var afterFour = employees.Skip(4);                      // Kabir, Meera
var lastTwo = employees.TakeLast(2);                    // Kabir, Meera
var middle = employees.Take(2..4);                      // Amit, Neha (range, .NET 6+)

var untilFinance = employees.TakeWhile(e => e.Dept != "Finance");   // Rohit, Priya, Amit (stops at Neha)
var fromFinance  = employees.SkipWhile(e => e.Dept != "Finance");   // Neha, Kabir, Meera

foreach (Employee[] batch in employees.Chunk(4))        // .NET 6+: [Rohit..Neha], [Kabir, Meera]
    Console.WriteLine(batch.Length);                    // 4, then 2
```

**Paging (the classic use):**

```csharp
int page = 2, pageSize = 2;
var pageItems = employees
    .OrderBy(e => e.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize);                                    // Amit, Neha
```

**Note:** `TakeWhile`/`SkipWhile` stop testing at the first item that breaks the condition, unlike `Where`, which checks every item. Always `OrderBy` before paging so pages are consistent.

## 7. Set Operations: Distinct, DistinctBy, Union, Intersect, Except

```csharp
var depts  = employees.Select(e => e.Dept).Distinct();      // IT, HR, Finance
var cities = employees.Select(e => e.City).Distinct();      // Pune, Mumbai, Delhi
var onePerCity = employees.DistinctBy(e => e.City);         // .NET 6+: Rohit, Priya, Neha (first per city)

int[] a = { 1, 2, 3, 4 };
int[] b = { 3, 4, 5 };

var union     = a.Union(b);        // 1, 2, 3, 4, 5   in either (no duplicates)
var intersect = a.Intersect(b);    // 3, 4            in both
var except    = a.Except(b);       // 1, 2            in a but not b
var concat    = a.Concat(b);       // 1, 2, 3, 4, 3, 4, 5   keeps duplicates (not a set op)

var deptsWithStaff = departments.Select(d => d.Name).Intersect(depts);   // IT, HR, Finance
var emptyDepts     = departments.Select(d => d.Name).Except(depts);      // Legal

// .NET 6+ "By" versions compare by a key
var newHires = employees.ExceptBy(new[] { 1, 2 }, e => e.Id);           // everyone except Ids 1 and 2
```

| Operator | Result | Duplicates |
| --- | --- | --- |
| `Distinct` | Unique items | Removed |
| `Union` | Items in A or B | Removed |
| `Intersect` | Items in both | Removed |
| `Except` | Items in A not in B | Removed |
| `Concat` | A then B | Kept |

**For your own classes** these use `Equals`/`GetHashCode`, so records work out of the box; for classes either implement `IEquatable<T>`, pass an `IEqualityComparer<T>`, or use the `...By(key)` versions. Distinct keeps the first occurrence's order.

## 8. Grouping and Joining: GroupBy, ToLookup, Join, GroupJoin, Left Join

### GroupBy

Splits items into groups that share a key. Each group is an `IGrouping<TKey, T>`: it has a `.Key` and is itself a sequence of the items.

```csharp
var byDept = employees.GroupBy(e => e.Dept);

foreach (var g in byDept)
{
    Console.WriteLine($"{g.Key}: {g.Count()} staff, avg {g.Average(e => e.Salary):N0}");
    foreach (var e in g) Console.WriteLine($"   {e.Name}");
}
// IT: 3 staff, avg 61,000      (Rohit, Amit, Kabir)
// HR: 2 staff, avg 51,500      (Priya, Meera)
// Finance: 1 staff, avg 52,000 (Neha)

var summary = employees
    .GroupBy(e => e.Dept)
    .Select(g => new { Dept = g.Key, Total = g.Sum(e => e.Salary), Top = g.MaxBy(e => e.Salary)!.Name })
    .OrderByDescending(x => x.Total);           // IT 183000 Amit, HR 103000 Meera, Finance 52000 Neha

var byDeptCity = employees.GroupBy(e => new { e.Dept, e.City });   // composite key
var bigDepts = employees.GroupBy(e => e.Dept).Where(g => g.Count() >= 2);   // like SQL HAVING: IT, HR

var countByCity = employees.CountBy(e => e.City);   // .NET 9+: (Pune, 3), (Mumbai, 2), (Delhi, 1)

// query syntax
var q = from e in employees
        group e by e.Dept into g
        select new { g.Key, Count = g.Count() };
```

**ToLookup** does the same grouping but **immediately**, returning an `ILookup<TKey, T>` you can index by key (missing keys give an empty sequence, not an exception):

```csharp
ILookup<string, Employee> lookup = employees.ToLookup(e => e.Dept);
foreach (var e in lookup["HR"]) Console.WriteLine(e.Name);   // Priya, Meera
Console.WriteLine(lookup["Legal"].Count());                   // 0
```

### Join (inner join)

Matches items from two sequences on equal keys; items without a match on either side are dropped.

```csharp
var withBuilding = employees.Join(
    departments,
    e => e.Dept,                 // outer key
    d => d.Name,                 // inner key
    (e, d) => new { e.Name, d.Building });
// Rohit Tech Park, Priya HQ, Amit Tech Park, Neha HQ, Kabir Tech Park, Meera HQ   (Legal not present)

// query syntax
var q2 = from e in employees
         join d in departments on e.Dept equals d.Name
         select new { e.Name, d.Building };
```

### GroupJoin and left outer join

`GroupJoin` pairs each outer item with the **group** of matching inner items (possibly empty), which is how you get a left join.

```csharp
var deptStaff = departments.GroupJoin(
    employees,
    d => d.Name,
    e => e.Dept,
    (d, staff) => new { d.Name, Count = staff.Count() });
// IT 3, HR 2, Finance 1, Legal 0   (Legal kept!)

// Left outer join: every department, employee or null
var leftJoin = from d in departments
               join e in employees on d.Name equals e.Dept into staff
               from s in staff.DefaultIfEmpty()
               select new { Dept = d.Name, Employee = s?.Name ?? "(none)" };
// ..., Finance Neha, Legal (none)

// .NET 10 adds a LeftJoin operator that does this in one call.
```

|  | `Join` | `GroupJoin` | Left join |
| --- | --- | --- | --- |
| SQL equivalent | INNER JOIN | (no direct one) | LEFT OUTER JOIN |
| Unmatched outer items | Dropped | Kept with empty group | Kept with null |
| Result shape | Flat pairs | Outer item + its group | Flat pairs |

**Composite join keys:** `on new { e.Dept, e.City } equals new { Dept = d.Name, d.City }` (property names and types must match).

## 9. Conversion and Generation

### Conversion (execute immediately)

```csharp
List<Employee> list   = employees.Where(e => e.Age > 30).ToList();
Employee[] arr        = employees.ToArray();
Dictionary<int, Employee> byId = employees.ToDictionary(e => e.Id);          // key must be unique
var nameById          = employees.ToDictionary(e => e.Id, e => e.Name);      // key + value selector
HashSet<string> citySet = employees.Select(e => e.City).ToHashSet();
ILookup<string, Employee> byCity = employees.ToLookup(e => e.City);        // many per key
// employees.ToDictionary(e => e.Dept);   // ArgumentException: duplicate key "IT"
```

### Type conversion (deferred)

```csharp
ArrayList old = new ArrayList { 1, 2, 3 };
IEnumerable<int> ints = old.Cast<int>();        // throws InvalidCastException if any item isn't int
IEnumerable<int> safe = old.OfType<int>();      // skips items that aren't int
IEnumerable<Employee> seq = employees.AsEnumerable();   // switch from IQueryable to in-memory LINQ
```

### Generation

```csharp
var oneToTen = Enumerable.Range(1, 10);           // 1..10
var squares  = Enumerable.Range(1, 5).Select(n => n * n);   // 1 4 9 16 25
var stars    = Enumerable.Repeat("*", 3);         // *, *, *
var nothing  = Enumerable.Empty<Employee>();      // safe empty sequence (better than null)
```

### Combining sequences

```csharp
var all = employees.Take(2).Concat(employees.TakeLast(1));    // Rohit, Priya, Meera
var withHead = employees.Select(e => e.Name).Prepend("HEAD").Append("TAIL");

string[] codes = { "E1", "E2", "E3" };
var zipped = codes.Zip(employees, (c, e) => $"{c}={e.Name}");   // E1=Rohit, E2=Priya, E3=Amit (stops at shorter)

var safeMax = employees.Where(e => e.Dept == "Legal").Select(e => e.Salary).DefaultIfEmpty(0).Max();   // 0, no exception

var indexed = employees.Index();   // .NET 9+: (Index, Item) pairs
```

## 10. Deferred vs Immediate Execution, IEnumerable vs IQueryable, Performance

| Deferred (lazy): returns a query | Immediate: runs now |
| --- | --- |
| `Where`, `Select`, `SelectMany`, `OrderBy`, `ThenBy`, `GroupBy`, `Join`, `Skip`, `Take`, `Distinct`, `Union`, `Concat`, `Zip`, `Cast`, `OfType`, `Reverse` | `ToList`, `ToArray`, `ToDictionary`, `ToLookup`, `ToHashSet`, `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`, `First`, `Last`, `Single`, `ElementAt`, `Any`, `All`, `Contains`, `foreach` |

**Multiple enumeration trap:** each loop over a deferred query re-runs it (re-reading a file or re-hitting the database).

```csharp
var q = employees.Where(e => Expensive(e));
int c = q.Count();                 // runs the filter over everything
var first = q.First();             // runs it AGAIN
var cached = q.ToList();           // run once, then reuse cached
```

### IEnumerable\<T> vs IQueryable\<T>

|  | `IEnumerable<T>` (LINQ to Objects) | `IQueryable<T>` (LINQ to Entities / EF Core) |
| --- | --- | --- |
| Data | In memory | Database or other provider |
| Lambdas become | Compiled delegates (`Func`) | Expression trees (`Expression<Func>`) |
| Runs | In your app, item by item | Translated to SQL, runs on the server |
| Filter location | After loading everything | In the SQL `WHERE` |

```csharp
// EF Core: good, the filter, sort and paging happen in SQL
var page = db.Employees.Where(e => e.Dept == "IT").OrderBy(e => e.Name).Take(20).ToList();

// Bad: ToList() first loads the WHOLE table, then filters in memory
var slow = db.Employees.ToList().Where(e => e.Dept == "IT");
```

**Performance tips**

- Filter early (`Where` before `Select`/`OrderBy`) and project only the columns you need.
- Use `Any()` instead of `Count() > 0`; `FirstOrDefault` instead of `Where(...).FirstOrDefault()` is equal in result but shorter.
- Materialise with `ToList()` when a query will be enumerated more than once.
- For repeated lookups by key, build a `Dictionary`/`ToLookup` once instead of calling `First` inside a loop (O(n²) → O(n)).
- In very hot loops, a plain `for` can beat LINQ; measure before optimising.
- **PLINQ:** `employees.AsParallel().Where(...)` spreads CPU-heavy work across cores (order not guaranteed unless `AsOrdered()`).

## 11. Cheat Sheet, Checklist and Interview Questions

```csharp
// Filter / shape / sort
employees.Where(e => e.Salary > 50000)
         .OrderBy(e => e.Dept).ThenByDescending(e => e.Salary)
         .Select(e => new { e.Name, e.Dept });

// One element
employees.First(); employees.FirstOrDefault(e => e.Id == 9);
employees.Last(p);  employees.LastOrDefault(p);
employees.ElementAt(2); employees.ElementAtOrDefault(9);
employees.Single(e => e.Id == 3); employees.SingleOrDefault(e => e.Id == 9);

// Aggregates
employees.Count(); employees.Sum(e => e.Salary); employees.Average(e => e.Salary);
employees.Min(e => e.Salary); employees.Max(e => e.Salary); employees.MaxBy(e => e.Age);

// Quantify / partition / sets
employees.Any(p); employees.All(p); numbers.Contains(8);
employees.Skip(10).Take(10); employees.Chunk(50);
names.Distinct(); a.Union(b); a.Intersect(b); a.Except(b);

// Group / join / convert
employees.GroupBy(e => e.Dept).Select(g => new { g.Key, Count = g.Count() });
employees.Join(departments, e => e.Dept, d => d.Name, (e, d) => new { e.Name, d.Building });
employees.ToDictionary(e => e.Id); employees.ToLookup(e => e.City);
Enumerable.Range(1, 10); Enumerable.Repeat(0, 5);
```

(`p` = any predicate lambda such as `e => e.Dept == "IT"`.)

**Checklist**

- [ ] Write the same query in query syntax and method syntax
- [ ] Explain deferred execution and name five immediate operators
- [ ] Sort by two keys with OrderBy + ThenBy
- [ ] Predict exceptions for First, Last, Single, ElementAt and their OrDefault versions
- [ ] Compute Count, Sum, Average, Min, Max and use MinBy/MaxBy
- [ ] Use Select and SelectMany; explain the difference
- [ ] Page results with Skip/Take
- [ ] Use Distinct, Union, Intersect, Except
- [ ] Group with GroupBy and filter groups (HAVING)
- [ ] Write an inner join and a left join
- [ ] Explain IEnumerable vs IQueryable

**Interview questions (short answers)**

1. **What is LINQ?** A set of query operators and syntax built into C# for querying any data source in a uniform, type-safe way.
2. **Query syntax vs method syntax?** Same result; query syntax is SQL-like and compiles to method calls; method syntax supports every operator.
3. **What is deferred execution?** A query runs only when enumerated or when an immediate operator is called; it reflects the data at that moment.
4. **First vs FirstOrDefault?** First throws if nothing matches; FirstOrDefault returns default/null.
5. **First vs Single?** First returns the first match even if there are many; Single throws if there isn't exactly one.
6. **SingleOrDefault with 2 matches?** Throws InvalidOperationException.
7. **ElementAt vs ElementAtOrDefault?** ElementAt throws for an invalid index; OrDefault returns default.
8. **Select vs SelectMany?** Select maps 1-to-1; SelectMany flattens nested sequences.
9. **OrderBy twice vs ThenBy?** A second OrderBy re-sorts everything; ThenBy sorts ties within the first order.
10. **What does Average do on an empty sequence?** Throws; use Any() or DefaultIfEmpty().
11. **Any vs Count() > 0?** Any stops at the first item; Count may scan everything.
12. **Join vs GroupJoin?** Join gives matched pairs (inner join); GroupJoin gives each outer item with its matching group, enabling left joins.
13. **GroupBy vs ToLookup?** GroupBy is deferred; ToLookup runs immediately and returns an indexable lookup.
14. **IEnumerable vs IQueryable?** IEnumerable runs in memory with delegates; IQueryable builds expression trees translated by a provider (e.g. to SQL).
15. **How do you implement paging?** `OrderBy(...).Skip((page - 1) * size).Take(size)`.
16. **What is the `let` keyword?** Stores an intermediate value inside a query-syntax expression.
17. **Union vs Concat?** Union removes duplicates; Concat keeps them.
