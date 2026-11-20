# Arrays

Fixed-size collections of one type: creating and looping, the `System.Array` helper methods, multi-dimensional and jagged arrays, arrays of objects, and copying them correctly.

## 1. Creating Arrays; Looping with for and foreach

An array is a **fixed-size**, zero-indexed list of elements of one type, stored together in memory. Arrays are reference types (they live on the heap), even when they hold `int`s.

```csharp
int[] marks = new int[5];                     // 5 elements, all 0
string[] names = new string[3];               // all null

int[] primes = new int[] { 2, 3, 5, 7 };      // size inferred
int[] evens = { 2, 4, 6, 8 };                 // shorthand
int[] odds = [1, 3, 5];                       // collection expression (C# 12)
var scores = new[] { 90, 85, 77 };            // type inferred: int[]

marks[0] = 72;                                // write
int first = marks[0];                         // read
int last = primes[^1];                        // 7: index from end (C# 8)
int[] middle = primes[1..3];                  // { 3, 5 }: range (C# 8)
Console.WriteLine(primes.Length);             // 4
// marks[5] = 1;                              // IndexOutOfRangeException
```

**Default values:** `0` for numbers, `false` for bool, `'\0'` for char, `null` for reference types.

### for loop: when you need the index or want to modify elements

```csharp
int[] prices = { 100, 250, 75 };
for (int i = 0; i < prices.Length; i++)
{
    prices[i] = prices[i] * 2;                // modify in place
    Console.WriteLine($"Item {i}: {prices[i]}");
}

for (int i = prices.Length - 1; i >= 0; i--)  // reverse order
    Console.Write(prices[i] + " ");
```

### foreach loop: simplest way to read every element

```csharp
int total = 0;
foreach (int p in prices)
    total += p;                               // read only

foreach (var name in new[] { "Rohit", "Priya" })
    Console.WriteLine(name.ToUpper());

// foreach (int p in prices) p = 0;           // ERROR: loop variable is read-only
```

|  | `for` | `foreach` |
| --- | --- | --- |
| Index available | Yes | No (unless you track it) |
| Modify elements | Yes: `arr[i] = ...` | No: loop variable is read-only |
| Iterate backwards / skip | Yes | No |
| Readability | More code | Cleaner |
| Works on | Arrays, lists (indexable) | Anything `IEnumerable` |

## 2. The System.Array Class

Every array implicitly derives from the abstract class `System.Array`, which provides properties (`Length`, `Rank`) and static helper methods for searching, sorting and resizing.

```csharp
int[] data = { 40, 10, 30, 20, 10 };

// IndexOf / LastIndexOf: linear search, -1 if not found
int pos  = Array.IndexOf(data, 10);        // 1
int last = Array.LastIndexOf(data, 10);    // 4
int none = Array.IndexOf(data, 99);        // -1

// Sort: ascending, in place
Array.Sort(data);                          // { 10, 10, 20, 30, 40 }

// BinarySearch: fast, but the array MUST be sorted first
int idx = Array.BinarySearch(data, 30);    // 3
int miss = Array.BinarySearch(data, 25);   // negative: not found (~miss = insert position)

// Reverse: in place
Array.Reverse(data);                       // { 40, 30, 20, 10, 10 }

// Clear: reset a range to default values (doesn't shrink)
Array.Clear(data, 0, 2);                   // { 0, 0, 20, 10, 10 }

// Resize: creates a NEW array and copies; needs ref
Array.Resize(ref data, 7);                 // { 0, 0, 20, 10, 10, 0, 0 }
Array.Resize(ref data, 3);                 // { 0, 0, 20 }: extra elements dropped
```

| Method | What it does | Speed | Notes |
| --- | --- | --- | --- |
| `Array.IndexOf(arr, x)` | First index of x, or -1 | O(n) | Works on unsorted arrays |
| `Array.BinarySearch(arr, x)` | Index of x, or a negative number | O(log n) | Array must be sorted |
| `Array.Clear(arr, start, len)` | Sets a range to defaults | O(len) | Length unchanged |
| `Array.Resize(ref arr, n)` | Makes a new array of size n, copies elements | O(n) | Old array becomes garbage |
| `Array.Sort(arr)` | Sorts in place | O(n log n) | Custom order: `Array.Sort(arr, (a, b) => b.CompareTo(a))` |
| `Array.Reverse(arr)` | Reverses in place | O(n) | Also `Array.Reverse(arr, start, len)` |

**Other useful members:** `arr.Length`, `arr.Rank` (number of dimensions), `Array.Exists(arr, x => x > 50)`, `Array.Find`, `Array.FindAll`, `Array.TrueForAll`, `Array.Fill(arr, value)`, `Array.ConvertAll(arr, x => x.ToString())`, `Array.Copy`, `Array.Empty<T>()`.

**Sorting objects:** `Array.Sort(claims, (a, b) => a.Amount.CompareTo(b.Amount));` or make the class implement `IComparable<T>`.

## 3. Multi-Dimensional and Jagged Arrays

A **multi-dimensional** (rectangular) array is one block of rows × columns where every row has the same length: `int[,]`. A **jagged** array is an array of arrays, where each row is a separate array that can have its own length: `int[][]`.

### Multi-dimensional (rectangular)

```csharp
int[,] grid = new int[3, 4];                  // 3 rows, 4 columns, all 0
int[,] matrix =
{
    { 1, 2, 3 },
    { 4, 5, 6 }
};                                            // 2 x 3

matrix[1, 2] = 60;                            // row 1, column 2
Console.WriteLine(matrix.Rank);               // 2 dimensions
Console.WriteLine(matrix.GetLength(0));       // 2 rows
Console.WriteLine(matrix.GetLength(1));       // 3 columns
Console.WriteLine(matrix.Length);             // 6 elements total

for (int r = 0; r < matrix.GetLength(0); r++)
{
    for (int c = 0; c < matrix.GetLength(1); c++)
        Console.Write($"{matrix[r, c],4}");
    Console.WriteLine();
}

int[,,] cube = new int[2, 3, 4];              // 3D works the same way
```

### Jagged (array of arrays)

```csharp
int[][] weeklyVisits = new int[3][];          // 3 rows, each not yet created
weeklyVisits[0] = new int[] { 5, 8 };
weeklyVisits[1] = new int[] { 2, 4, 6, 9 };   // rows can differ in length
weeklyVisits[2] = new int[] { 7 };

int[][] seats =
{
    new[] { 1, 2, 3 },
    new[] { 4, 5 }
};

Console.WriteLine(weeklyVisits[1][3]);        // 9: [row][column]
Console.WriteLine(weeklyVisits.Length);       // 3 rows
Console.WriteLine(weeklyVisits[1].Length);    // 4 in row 1

foreach (int[] row in weeklyVisits)
    Console.WriteLine(string.Join(", ", row));
```

|  | Multi-dimensional `int[,]` | Jagged `int[][]` |
| --- | --- | --- |
| Shape | Rectangular: all rows same length | Rows can differ |
| Memory | One contiguous block | One array per row + an array of references |
| Access | `a[r, c]` | `a[r][c]` |
| Row count | `a.GetLength(0)` | `a.Length` |
| `Length` means | Total elements | Number of rows |
| Rows must be created separately | No | Yes (else `NullReferenceException`) |
| Performance | Slightly slower indexing | Usually faster (runtime optimises 1-D arrays) |
| Use for | Grids, matrices, game boards | Uneven data: seats per row, items per order |

## 4. Array of Objects

An array of a class type stores **references**, not objects. `new Patient[3]` creates three empty slots (all `null`); you must create each object with `new` before using it.

```csharp
public class Patient
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public override string ToString() => $"{Name} ({Age})";
}

Patient[] ward = new Patient[3];          // { null, null, null }
// Console.WriteLine(ward[0].Name);       // NullReferenceException

ward[0] = new Patient { Name = "Rohit", Age = 30 };
ward[1] = new Patient { Name = "Priya", Age = 62 };
ward[2] = new Patient { Name = "Amit",  Age = 8 };

// Or initialise in one go
Patient[] icu =
{
    new() { Name = "Neha", Age = 45 },
    new() { Name = "Kabir", Age = 71 }
};

foreach (Patient p in ward)
    Console.WriteLine(p);                 // uses ToString()

// Search, filter, sort
Patient? senior = Array.Find(ward, p => p.Age >= 60);       // Priya
Patient[] kids  = Array.FindAll(ward, p => p.Age < 18);     // Amit
Array.Sort(ward, (a, b) => a.Age.CompareTo(b.Age));         // by age
int total = ward.Sum(p => p.Age);                           // LINQ works on arrays
```

**Remember:** `ward[0].Age = 31;` changes the object itself; any other variable or array holding the same reference sees the change. That's why copying arrays of objects needs care (section 5).

**Polymorphic arrays:** an array of a base type can hold derived objects: `Employee[] staff = { new Manager(), new Clerk() };` and each element's overridden methods run correctly.

## 5. CopyTo, Clone and Deep Copy

`Clone()` and `CopyTo()` make a **shallow copy**: a new array, but for reference-type elements both arrays point to the **same objects**. A **deep copy** also duplicates every object, so the two arrays are fully independent.

&#91;embedded content: Assignment vs shallow copy vs deep copy\]

With value types (`int[]`), shallow and deep copies behave the same, because the elements are the values themselves. The difference only bites with arrays of objects.

### Clone and CopyTo

```csharp
int[] a = { 1, 2, 3 };

int[] c = (int[])a.Clone();       // Clone returns object, so cast; creates the new array for you

int[] d = new int[5];
a.CopyTo(d, 2);                   // copy into an EXISTING array starting at index 2: { 0, 0, 1, 2, 3 }

int[] e = new int[2];
Array.Copy(a, 1, e, 0, 2);        // source, srcIndex, dest, destIndex, count: { 2, 3 }

int[] f = a[..];                  // range copy (C# 8): also a shallow copy

c[0] = 99;
Console.WriteLine(a[0]);          // 1: value types are independent
```

|  | `Clone()` | `CopyTo(dest, index)` | `Array.Copy(...)` |
| --- | --- | --- | --- |
| Creates the new array | Yes | No: you supply it | No: you supply it |
| Returns | `object` (cast needed) | `void` | `void` |
| Partial copy | No | From a start index in dest | Any range |
| Copy type | Shallow | Shallow | Shallow |

### The shallow-copy trap with objects

```csharp
Patient[] original = { new() { Name = "Rohit", Age = 30 } };
Patient[] copy = (Patient[])original.Clone();

copy[0].Age = 99;                        // changes the SHARED object
Console.WriteLine(original[0].Age);      // 99 (!)

copy[0] = new Patient { Name = "New" };  // replaces a slot in copy only
Console.WriteLine(original[0].Name);     // Rohit
```

### Deep copy

Copy the array **and** create a new object for every element.

```csharp
public class Patient
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public Address Home { get; set; } = new();

    public Patient DeepCopy() => new Patient
    {
        Name = Name,                            // strings are immutable: safe to share
        Age = Age,
        Home = new Address { City = Home.City } // nested objects copied too
    };
}
public class Address { public string City { get; set; } = ""; }

// Way 1: loop
Patient[] deep = new Patient[original.Length];
for (int i = 0; i < original.Length; i++)
    deep[i] = original[i].DeepCopy();

// Way 2: LINQ
Patient[] deep2 = original.Select(p => p.DeepCopy()).ToArray();

// Way 3: serialize and deserialize (simple, slower; needs public properties)
string json = JsonSerializer.Serialize(original);
Patient[] deep3 = JsonSerializer.Deserialize<Patient[]>(json)!;

deep[0].Age = 1;
Console.WriteLine(original[0].Age);      // unchanged
```

**Other options:** implement `ICloneable` (but its `Clone()` doesn't say if it's shallow or deep, so a named `DeepCopy()` is clearer); use a copy constructor `new Patient(other)`; use `MemberwiseClone()` inside the class for a shallow member copy, then deep-copy reference fields; records give `with` expressions (shallow).

|  | Assignment `b = a` | Shallow copy | Deep copy |
| --- | --- | --- | --- |
| New array | No | Yes | Yes |
| New element objects | No | No | Yes |
| Change `b[0].Age` affects `a`? | Yes | Yes | No |
| Replace `b[0] = x` affects `a`? | Yes | No | No |
| How | `=` | `Clone`, `CopyTo`, `Array.Copy`, ranges | Loop/LINQ + per-object copy, serialization |

## 6. Cheat Sheet, Checklist and Interview Questions

```csharp
int[] a = { 5, 3, 8 };                 // create
int[] b = new int[10];                 // sized, defaults to 0
int n = a.Length;  int last = a[^1];   // length, index from end

for (int i = 0; i < a.Length; i++) a[i] *= 2;   // modify
foreach (var x in a) Console.Write(x);          // read

Array.Sort(a);  Array.Reverse(a);
int i1 = Array.IndexOf(a, 8);          // linear search
int i2 = Array.BinarySearch(a, 8);     // sorted arrays only
Array.Clear(a, 0, 1);  Array.Resize(ref a, 5);

int[,] grid = new int[2, 3];   grid[1, 2] = 7;        // rectangular
int[][] jag = { new[] { 1 }, new[] { 2, 3 } };   int v = jag[1][0];   // jagged

var shallow = (int[])a.Clone();        // shallow copy
a.CopyTo(b, 0);                        // into existing array
var deep = people.Select(p => p.DeepCopy()).ToArray();   // deep copy
```

**Checklist**

- [ ] Create arrays four ways and explain default values
- [ ] Loop with for (modify) and foreach (read)
- [ ] Use IndexOf, BinarySearch, Clear, Resize, Sort, Reverse and predict each result
- [ ] Build and loop a 2D array and a jagged array; explain `Length` for each
- [ ] Create an array of objects and avoid `NullReferenceException`
- [ ] Show the shallow-copy trap and fix it with a deep copy

**Interview questions (short answers)**

1. **What is an array?** A fixed-size, zero-indexed collection of elements of one type, stored contiguously.
2. **Are arrays value or reference types?** Reference types (derived from `System.Array`), even `int[]`.
3. **Can you change an array's size?** No; `Array.Resize` creates a new array and copies. Use `List<T>` for dynamic size.
4. **for vs foreach?** for gives the index and can modify elements; foreach is simpler and read-only.
5. **IndexOf vs BinarySearch?** IndexOf: linear, works unsorted, O(n). BinarySearch: needs sorted data, O(log n).
6. **What does `Array.Clear` do?** Sets a range of elements to their default values; length doesn't change.
7. **Multi-dimensional vs jagged array?** `int[,]` is one rectangular block; `int[][]` is an array of arrays whose rows can differ in length.
8. **What is `Length` for a 2D array?** The total number of elements; use `GetLength(0)` for rows.
9. **What's inside `new Patient[3]`?** Three `null` references; each object must be created.
10. **Clone vs CopyTo?** Clone creates and returns a new array; CopyTo copies into an existing array at an index. Both are shallow.
11. **Shallow vs deep copy?** Shallow copies references (objects shared); deep copies the objects too (fully independent).
12. **How do you deep copy an array of objects?** Create a new array and copy each element with a copy method/constructor (or serialize and deserialize).
13. **What happens when you access an invalid index?** `IndexOutOfRangeException`.
