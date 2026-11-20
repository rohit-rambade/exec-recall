# Type Conversion

How to turn a value of one type into another safely: implicit and explicit casts, parsing strings, and the Convert class.

## 1. Overview of Type Conversion

Type conversion turns a value of one type into another. Because C# is strongly typed, you must convert whenever types don't match, for example to do maths with user input (always a `string`).

| Way | Syntax | Use when | If it can't convert |
| --- | --- | --- | --- |
| Implicit cast | `long l = i;` | Small → big numeric type, no data loss | Never fails (compiler allows only safe ones) |
| Explicit cast | `int i = (int)d;` | Big → small, or between numeric types | Loses data silently (truncates or wraps) |
| Parse / TryParse | `int.Parse(s)`, `int.TryParse(s, out n)` | String → number / date / bool | `Parse` throws; `TryParse` returns `false` |
| Convert class | `Convert.ToInt32(x)` | Between many base types, including `null` | Throws on bad format; `null` → 0 |
| ToString | `n.ToString("N2")` | Anything → string | Never fails |

**Rule of thumb:** casts for number ↔ number, `TryParse` for user input text, `Convert` for mixed or nullable sources, `ToString` for display.

```csharp
string input = "25";
// int age = input;          // ERROR: cannot convert string to int
int age = int.Parse(input);  // OK
```

## 2. Implicit Casting

Implicit casting happens automatically, with no syntax, when the target type can hold every value of the source type (a **widening** conversion). The compiler only allows it when no data can be lost.

```csharp
int i = 100;
long l = i;        // int -> long
float f = l;       // long -> float
double d = f;      // float -> double
decimal m = i;     // int -> decimal
int code = 'A';    // char -> int: 65
```

&#91;embedded content: Implicit (widening) numeric conversions\]

Any path along the arrows is allowed, so `byte` to `double` works in one step too (`double d = myByte;`).

**Also implicit**

- Derived class → base class: `Animal a = new Dog();` (every Dog is an Animal).
- Any type → `object`: `object o = 42;` (for value types this is **boxing**).
- Value type → its nullable: `int? n = 5;`

**Not implicit (compile error)**

```csharp
double d = 9.7;
// int i = d;        // ERROR CS0266: possible data loss, use a cast
// float f = d;      // ERROR: double is wider than float
// decimal m = d;    // ERROR: double and decimal never convert implicitly
// char c = 65;      // ERROR: int to char needs (char)65
```

## 3. Explicit Casting

An explicit cast `(targetType)value` forces a **narrowing** conversion. You are telling the compiler "I accept possible data loss", and it will not warn you again.

```csharp
double price = 99.99;
int whole = (int)price;          // 99: decimals are TRUNCATED, not rounded
int neg = (int)-7.8;             // -7: truncates toward zero

long big = 3_000_000_000;
int wrapped = (int)big;          // -1294967296: overflow wraps around silently

int code = 66;
char letter = (char)code;        // 'B'

double d = 1.23456789;
float f = (float)d;              // 1.234568: precision lost
decimal money = (decimal)d;      // double -> decimal needs a cast
```

**Three ways data gets lost**

| Problem | Example | Result |
| --- | --- | --- |
| Truncation | `(int)9.99` | `9` |
| Overflow / wrap-around | `int n = 300; (byte)n` | `44` (300 − 256) |
| Precision loss | `(float)1.23456789` | `1.234568` |

**Catch overflow with `checked`**

```csharp
long big = 3_000_000_000;
try
{
    int safe = checked((int)big);   // throws OverflowException instead of wrapping
}
catch (OverflowException) { Console.WriteLine("Too big for int"); }

checked
{
    int x = int.MaxValue;
    x++;                             // throws inside a checked block
}
```

`unchecked` does the opposite (forces wrap-around). Projects can turn on overflow checking for everything with `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` in the `.csproj`.

**Need rounding instead of truncation?** Use `(int)Math.Round(9.6)` = 10, or `Convert.ToInt32(9.6)` = 10 (banker's rounding: `Convert.ToInt32(2.5)` = 2).

**Casting objects (reference types)**

```csharp
object o = "hello";
string s = (string)o;            // OK
// int n = (int)o;               // InvalidCastException at run time

string s2 = o as string;         // 'as': returns null instead of throwing
if (o is string s3) { }          // 'is' pattern: test and cast in one step (safest)
```

## 4. Parse and TryParse

Both convert a **string** into a number, date or bool. `Parse` throws an exception on bad input; `TryParse` returns `false` instead, which makes it the right choice for anything a user types.

### Parse

```csharp
int age = int.Parse("25");                 // 25
double rate = double.Parse("7.5");         // 7.5
bool ok = bool.Parse("true");              // true (case-insensitive)
DateTime dt = DateTime.Parse("2026-11-20");

int bad = int.Parse("25abc");      // FormatException
int big = int.Parse("9999999999"); // OverflowException
int nul = int.Parse(null);         // ArgumentNullException
int sp  = int.Parse(" 42 ");       // 42: surrounding spaces are allowed
```

### TryParse

Signature: `bool int.TryParse(string s, out int result)`. On failure `result` is set to `0` (the default).

```csharp
Console.Write("Enter age: ");
string input = Console.ReadLine();

if (int.TryParse(input, out int age))
    Console.WriteLine($"Next year you will be {age + 1}");
else
    Console.WriteLine("Please enter a whole number");

// Keep asking until valid
int qty;
while (!int.TryParse(Console.ReadLine(), out qty) || qty <= 0)
    Console.Write("Invalid. Enter a positive number: ");
```

|  | `Parse` | `TryParse` |
| --- | --- | --- |
| Returns | The value | `bool` success flag; value via `out` |
| Bad input | Throws `FormatException` | Returns `false`, result = 0 |
| Too big | Throws `OverflowException` | Returns `false` |
| `null` | Throws `ArgumentNullException` | Returns `false` |
| Speed on bad input | Slow (exceptions are costly) | Fast |
| Use for | Data you trust (config, constants) | User input, files, web requests |

**Culture matters:** `double.Parse("7,5")` depends on the machine's regional settings (comma vs dot). For fixed formats use `double.Parse(s, CultureInfo.InvariantCulture)`. Other options: `NumberStyles.Currency`, `NumberStyles.AllowThousands`, and `DateTime.ParseExact(s, "dd-MM-yyyy", null)`.

## 5. Conversion Methods

The `System.Convert` class converts between all base types (numbers, `bool`, `char`, `string`, `DateTime`) and handles `null` gracefully. `ToString()` goes the other way, turning any value into text.

```csharp
int a = Convert.ToInt32("123");      // 123: from string
int b = Convert.ToInt32(9.6);        // 10: ROUNDS (cast would give 9)
int c = Convert.ToInt32(2.5);        // 2: banker's rounding (to even)
int d = Convert.ToInt32(null);       // 0: null becomes 0, no exception
int e = Convert.ToInt32(true);       // 1
bool f = Convert.ToBoolean("False"); // false
bool g = Convert.ToBoolean(5);       // true: non-zero is true
double h = Convert.ToDouble("3.14");
string s = Convert.ToString(42);     // "42"
string bin = Convert.ToString(10, 2);   // "1010": number to base 2, 8 or 16
int fromHex = Convert.ToInt32("FF", 16); // 255

// int x = Convert.ToInt32("abc");   // FormatException
// int y = Convert.ToInt32(1e12);    // OverflowException (Convert checks overflow, a cast does not)
```

**Common methods:** `ToInt16`, `ToInt32`, `ToInt64`, `ToDouble`, `ToDecimal`, `ToSingle`, `ToBoolean`, `ToChar`, `ToString`, `ToDateTime`, `ToByte`, plus `Convert.ToBase64String` / `FromBase64String` for bytes.

### Cast vs Parse vs Convert

| Input | `(int)` cast | `int.Parse` | `Convert.ToInt32` |
| --- | --- | --- | --- |
| `9.6` (double) | `9` (truncates) | n/a (string only) | `10` (rounds) |
| `"123"` (string) | Compile error | `123` | `123` |
| `null` string | Compile error / exception | `ArgumentNullException` | `0` |
| `"abc"` | Compile error | `FormatException` | `FormatException` |
| Overflow | Wraps silently | `OverflowException` | `OverflowException` |

### ToString and formatting

```csharp
double amt = 1234567.891;
amt.ToString();          // "1234567.891"
amt.ToString("F2");      // "1234567.89"      fixed decimals
amt.ToString("N2");      // "1,234,567.89"    thousands separators
amt.ToString("C");       // currency symbol by culture, e.g. "₹12,34,567.89" in India
0.256.ToString("P1");    // "25.6%"
42.ToString("D5");       // "00042"
255.ToString("X");       // "FF"
DateTime.Now.ToString("dd-MM-yyyy");
$"{amt:N2}";             // same format codes inside interpolation
```

## 6. Important Points, Cheat Sheet and Interview Questions

**Points to remember**

1. Implicit conversion is allowed only when no data can be lost (widening); everything else needs an explicit cast or a method.
2. A numeric cast **truncates** decimals and **wraps** on overflow without any error. Use `checked` or `Convert` when overflow must be caught.
3. You can't cast a `string` to a number. Use `Parse`, `TryParse` or `Convert`.
4. Prefer `TryParse` for user input: no exceptions, clear `bool` result.
5. `Convert.ToInt32` rounds (banker's rounding) and turns `null` into `0`; a cast truncates.
6. `float`/`double` ↔ `decimal` always needs an explicit cast.
7. Parsing depends on culture; pass `CultureInfo.InvariantCulture` for fixed formats.
8. Reference types: use `is` / `as` for safe casting; a bad direct cast throws `InvalidCastException`.
9. Boxing (value type → `object`) is implicit; unboxing needs an explicit cast to the **exact** type: `(int)obj` works, `(long)obj` on a boxed int throws.

**Cheat sheet**

```csharp
long l = myInt;                          // implicit (widening)
int i = (int)myDouble;                   // explicit: truncates
int c = checked((int)myLong);            // explicit, throws on overflow
int p = int.Parse("42");                 // string -> int, throws on bad input
bool ok = int.TryParse(s, out int n);    // safe; n = 0 if it fails
int v = Convert.ToInt32(x);              // rounds, null -> 0
string t = n.ToString("N2");             // number -> formatted text
string str = obj as string;              // reference cast, null if wrong type
if (obj is int num) { }                  // type test + cast
```

**Checklist**

- [ ] Name the four ways to convert and when to use each
- [ ] Draw the implicit-conversion arrows from memory
- [ ] Predict results of `(int)9.99`, `(int)-7.8`, `(byte)` on 300
- [ ] Use `checked` to catch overflow
- [ ] Write a TryParse input loop
- [ ] Explain cast vs Parse vs Convert for `9.6`, `"123"` and `null`
- [ ] Format numbers with F2, N2, C, P, D5

**Interview questions (short answers)**

1. **Implicit vs explicit conversion?** Implicit is automatic and safe (widening); explicit needs `(type)` and may lose data (narrowing).
2. **What does `(int)9.9` return?** `9`: casting truncates toward zero.
3. **Parse vs TryParse?** Parse throws on invalid input; TryParse returns `false` and sets the out value to 0.
4. **`int.Parse` vs `Convert.ToInt32`?** Both convert strings; Convert also accepts other types, returns 0 for `null`, and rounds doubles.
5. **What is `checked`?** It makes arithmetic and conversions throw `OverflowException` instead of wrapping.
6. **Can you convert `double` to `decimal` implicitly?** No; neither direction is implicit.
7. **`is` vs `as`?** `is` tests the type (and can cast with a pattern); `as` casts and returns `null` on failure. Neither throws.
8. **What is boxing and unboxing?** Boxing wraps a value type in an `object` on the heap (implicit); unboxing extracts it with an explicit cast to the exact type.
9. **Why does `Convert.ToInt32(2.5)` return 2?** It uses banker's rounding (round half to even).
10. **What exception does a bad reference cast throw?** `InvalidCastException`.
