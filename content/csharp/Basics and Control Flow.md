---
title: Basics & Control Flow
---

# Section 2: Basics and Control Flow

Variables, types and operators, then every way C# makes decisions and repeats work, with the practice programs from the course.

## 1. Variables

A variable is a named storage location with a fixed type. Declare it with `type name = value;` and C# will not let you put a different type in it.

```csharp
int age;                 // declaration
age = 25;                // assignment
int marks = 90;          // declaration + initialisation
string name = "Rohit";
double salary = 45000.50;
bool isActive = true;
char grade = 'A';

int x = 1, y = 2, z = 3; // several on one line
```

**`var` (implicit typing, C# 3)**

```csharp
var city = "Pune";    // compiler infers string
var count = 10;       // infers int
// var total;         // ERROR: var must be initialised
// city = 5;          // ERROR: still strongly typed (string)
```

**Constants and read-only**

| Keyword | Value set | Example |
| --- | --- | --- |
| `const` | At compile time; must be initialised at declaration | `const double Pi = 3.14159;` |
| `readonly` | At declaration or in the constructor (fields only) | `readonly int _maxUsers;` |

**Rules to remember**

- A local variable must be assigned before it is used; otherwise compile error CS0165.
- Fields get default values automatically (`0`, `false`, `null`); locals do not.
- **Scope:** a variable lives only inside the `{ }` block where it is declared.
- Names follow the identifier rules: start with a letter or `_`, no spaces, case-sensitive, camelCase for locals.

```csharp
if (true)
{
    int inside = 5;
}
// Console.WriteLine(inside);  // ERROR: out of scope
```

## 2. Primitive Types

C# has built-in value types for numbers, characters and booleans; each keyword is an alias for a .NET type (`int` = `System.Int32`).

| C# keyword | .NET type | Size | Range / precision | Default |
| --- | --- | --- | --- | --- |
| `byte` | Byte | 1 byte | 0 to 255 | 0 |
| `sbyte` | SByte | 1 byte | -128 to 127 | 0 |
| `short` | Int16 | 2 bytes | -32,768 to 32,767 | 0 |
| `ushort` | UInt16 | 2 bytes | 0 to 65,535 | 0 |
| `int` | Int32 | 4 bytes | about ±2.1 billion | 0 |
| `uint` | UInt32 | 4 bytes | 0 to about 4.3 billion | 0 |
| `long` | Int64 | 8 bytes | about ±9.2 × 10^18 | 0L |
| `ulong` | UInt64 | 8 bytes | 0 to about 1.8 × 10^19 | 0 |
| `float` | Single | 4 bytes | \~6–9 digits | 0.0F |
| `double` | Double | 8 bytes | \~15–17 digits | 0.0 |
| `decimal` | Decimal | 16 bytes | 28–29 digits, exact base-10 (money) | 0.0M |
| `char` | Char | 2 bytes | one UTF-16 character | '\\0' |
| `bool` | Boolean | 1 byte | `true` / `false` | false |

`string` and `object` are built-in too, but they are **reference types**, not primitives.

Check limits in code: `int.MaxValue`, `int.MinValue`, `sizeof(long)`.

**Type conversion**

| Kind | When | Example |
| --- | --- | --- |
| Implicit | Small to big, no data loss | `int i = 10; long l = i; double d = i;` |
| Explicit (cast) | Big to small, may lose data | `double d = 9.78; int i = (int)d; // 9` |
| Convert class | Between unrelated types | `int n = Convert.ToInt32("123");` |
| Parse | String to number | `int n = int.Parse("123");` |
| TryParse | Safe parse, no exception | `bool ok = int.TryParse(input, out int n);` |
| ToString | Anything to string | `string s = 45.ToString();` |

**Watch out**

- Integer division drops decimals: `7 / 2` is `3`; `7 / 2.0` is `3.5`.
- Overflow wraps silently: `int.MaxValue + 1` becomes `int.MinValue`. Use `checked { }` to throw instead.
- `int.Parse("abc")` throws `FormatException`; prefer `TryParse` for user input.
- Use `decimal` for money: `0.1 + 0.2` in `double` is `0.30000000000000004`.

## 3. Operators

Operators act on values (operands) to compute, compare or combine them.

| Category | Operators | Example (a = 10, b = 3) |
| --- | --- | --- |
| Arithmetic | `+  -  *  /  %` | `a / b` = 3, `a % b` = 1 |
| Increment / decrement | `++  --` | `a++` uses then adds; `++a` adds then uses |
| Assignment | `=  +=  -=  *=  /=  %=  ??=` | `a += 5` makes a = 15 |
| Relational | `==  !=  >  <  >=  <=` | `a > b` is `true` |
| Logical | `&&  \|\|  !` | `a > 5 && b > 5` is `false` |
| Bitwise | `&  \|  ^  ~  <<  >>` | `a << 1` = 20 |
| Ternary | `condition ? x : y` | `a > b ? "a" : "b"` gives "a" |
| Null | `??  ?.  ??=` | `name ?? "Guest"` |
| Type | `is  as  typeof  sizeof` | `obj is string s` |

**Prefix vs postfix**

```csharp
int i = 5;
int a = i++;   // a = 5, then i becomes 6
int b = ++i;   // i becomes 7, then b = 7
```

**Short-circuit:** with `&&`, if the left side is `false` the right side is never evaluated; with `||`, if the left is `true` it stops. That is why `if (s != null && s.Length > 0)` is safe.

**Precedence (high to low), most-used rows**

1. `()` brackets, `x++`, `x--`, `.` member access
2. Unary: `!`, `++x`, `--x`, `-x`, casts
3. `*  /  %`
4. `+  -`
5. `<<  >>`
6. `<  >  <=  >=  is  as`
7. `==  !=`
8. `&`, then `^`, then `|`
9. `&&`, then `||`
10. `??`, then `?:` ternary
11. Assignment `=  +=  ...` (right to left)

Example: `2 + 3 * 4` = 14, not 20. When in doubt, add brackets.

## 4. Practice Programs: Variables and Operators

Four small programs that use input, conversion and arithmetic. Try each yourself first, then compare.

### Area of Circle

Formula: area = π × r²

```csharp
Console.Write("Enter radius: ");
double radius = double.Parse(Console.ReadLine());
double area = Math.PI * radius * radius;   // or Math.Pow(radius, 2)
Console.WriteLine($"Area = {area:F2}");     // F2 = 2 decimal places
// Input 5  ->  Area = 78.54
```

### Feet to Centimeters

1 foot = 30.48 cm

```csharp
Console.Write("Enter feet: ");
double feet = double.Parse(Console.ReadLine());
const double CmPerFoot = 30.48;
double cm = feet * CmPerFoot;
Console.WriteLine($"{feet} ft = {cm} cm");
// Input 6  ->  6 ft = 182.88 cm
```

### Nearest Thousand

Round a number to the nearest 1000 (e.g. 4499 → 4000, 4500 → 5000).

```csharp
Console.Write("Enter number: ");
int n = int.Parse(Console.ReadLine());

// Way 1: integer maths (positive numbers)
int nearest = (n + 500) / 1000 * 1000;

// Way 2: Math.Round (AwayFromZero avoids banker's rounding: 2.5 -> 3, not 2)
int nearest2 = (int)(Math.Round(n / 1000.0, MidpointRounding.AwayFromZero) * 1000);

Console.WriteLine($"Nearest thousand: {nearest}");
// Input 4567  ->  5000
```

**Why way 1 works:** adding 500 pushes anything at or above the halfway mark into the next thousand; integer division by 1000 then drops the remainder.

### Convert Seconds into Minutes

```csharp
Console.Write("Enter seconds: ");
int totalSeconds = int.Parse(Console.ReadLine());
int minutes = totalSeconds / 60;   // integer division
int seconds = totalSeconds % 60;   // remainder
Console.WriteLine($"{minutes} min {seconds} sec");
// Input 135  ->  2 min 15 sec

// Bonus: hours too
int h = totalSeconds / 3600, m = (totalSeconds % 3600) / 60, s = totalSeconds % 60;
// Or: TimeSpan.FromSeconds(totalSeconds).ToString(@"hh\:mm\:ss")
```

**Pattern to notice:** `/` gives the whole units and `%` gives the leftover. The same trick splits digits (`n % 10`, `n / 10`).

## 5. If, If-else, Else-if, Nested-if

An `if` runs a block only when its condition is `true`; `else if` checks further conditions in order, and the first true one wins.

| Form | Use when |
| --- | --- |
| `if` | One optional action |
| `if-else` | Exactly one of two actions |
| `else-if` ladder | One of many ranges or cases |
| Nested `if` | A second check only makes sense after the first passes |

```csharp
int marks = 72;

if (marks >= 35)                          // if
    Console.WriteLine("Pass");

if (marks % 2 == 0)                       // if-else
    Console.WriteLine("Even");
else
    Console.WriteLine("Odd");

if (marks >= 75)      Console.WriteLine("Distinction");   // else-if ladder
else if (marks >= 60) Console.WriteLine("First class");
else if (marks >= 35) Console.WriteLine("Pass");
else                  Console.WriteLine("Fail");

bool hasId = true;                        // nested if
if (marks >= 35)
{
    if (hasId) Console.WriteLine("Collect certificate");
    else       Console.WriteLine("Bring your ID");
}
```

**Tips:** always use braces for multi-line blocks; order else-if checks from most to least specific; a short if-else can become a ternary: `string r = marks >= 35 ? "Pass" : "Fail";`

### Height Category

Read a height in cm and print its category (thresholds below are a common version; use your course's if they differ).

```csharp
Console.Write("Enter height (cm): ");
double h = double.Parse(Console.ReadLine());

if (h < 150)       Console.WriteLine("Dwarf");
else if (h < 165)  Console.WriteLine("Average height");
else if (h <= 195) Console.WriteLine("Taller");
else               Console.WriteLine("Abnormal height");
```

Because the ladder stops at the first true condition, each `else if` only needs the upper limit.

### Largest of Three Numbers

```csharp
int a = 12, b = 45, c = 30;

// Way 1: logical operators
if (a >= b && a >= c)      Console.WriteLine($"Largest: {a}");
else if (b >= a && b >= c) Console.WriteLine($"Largest: {b}");
else                       Console.WriteLine($"Largest: {c}");

// Way 2: nested if
if (a > b)
{
    if (a > c) Console.WriteLine(a); else Console.WriteLine(c);
}
else
{
    if (b > c) Console.WriteLine(b); else Console.WriteLine(c);
}

// Way 3: built-in
int max = Math.Max(a, Math.Max(b, c));
```

Use `>=` in way 1 so equal values are handled correctly.

## 6. Switch-Case

`switch` picks one branch by matching a single value against constant cases. It is cleaner than a long else-if ladder when you compare one variable to fixed values.

```csharp
int day = 3;
switch (day)
{
    case 1:
        Console.WriteLine("Monday");
        break;
    case 2:
        Console.WriteLine("Tuesday");
        break;
    case 6:
    case 7:                              // stacked cases share one body
        Console.WriteLine("Weekend");
        break;
    default:                             // runs when nothing matches
        Console.WriteLine("Mid-week");
        break;
}
```

**Rules that trip people up**

- **No fall-through:** every non-empty case must end with `break`, `return`, `throw` or `goto case`; otherwise compile error CS0163.
- Case labels must be constants; duplicates are not allowed.
- Works with `int`, `char`, `string`, `enum`, `bool` and (C# 7+) any type via patterns.
- `default` is optional and can be placed anywhere, but put it last for readability.

**Pattern matching (C# 7+)**

```csharp
switch (marks)
{
    case >= 75: Console.WriteLine("Distinction"); break;   // relational pattern, C# 9
    case >= 35: Console.WriteLine("Pass"); break;
    default:    Console.WriteLine("Fail"); break;
}
```

**Switch expression (C# 8+): returns a value**

```csharp
string dayName = day switch
{
    1 => "Monday",
    2 => "Tuesday",
    6 or 7 => "Weekend",
    _ => "Mid-week"          // _ = discard, acts like default
};
```

**switch vs if-else:** use `switch` for one value against many constants; use `if-else` for ranges or conditions on different variables (or a switch with relational patterns on modern C#).

## 7. While, Do-While

Both repeat a block while a condition is true. The only difference is when the condition is checked: `while` checks before each pass, `do-while` checks after, so its body always runs at least once.

&#91;embedded content: while vs do-while flow\]

If the condition is false from the start, `while` runs zero times and `do-while` runs once.

```csharp
// while: count 1 to 5
int i = 1;
while (i <= 5)
{
    Console.WriteLine(i);
    i++;                 // forget this and the loop never ends
}

// do-while: menu that must show at least once
int choice;
do
{
    Console.WriteLine("1. Add  2. View  0. Exit");
    choice = int.Parse(Console.ReadLine());
} while (choice != 0);   // note the semicolon
```

**Classic while example: sum of digits**

```csharp
int n = 1234, sum = 0;
while (n > 0)
{
    sum += n % 10;   // last digit
    n /= 10;         // drop last digit
}
Console.WriteLine(sum);  // 10
```

|  | `while` | `do-while` |
| --- | --- | --- |
| Condition checked | Before the body (entry-controlled) | After the body (exit-controlled) |
| Minimum runs | 0 | 1 |
| Semicolon after `while(...)` | No | Yes |
| Typical use | Unknown number of repeats, may be none | Menus, input validation, retry prompts |

**Infinite loop:** `while (true) { ... }` runs until a `break` or `return` inside it.

## 8. For Loop and Nested For Loops

A `for` loop packs start, condition and step into one line, so use it when you know how many times to repeat.

```csharp
//   init      condition   step
for (int i = 1; i <= 5;     i++)
{
    Console.WriteLine(i);      // 1 2 3 4 5
}
```

**Order of execution:** init runs once → check condition → run body → step → check condition again → … until the condition is false.

**Variations**

```csharp
for (int i = 10; i >= 1; i--) { }          // count down
for (int i = 0; i <= 20; i += 2) { }       // even numbers, step 2
for (int i = 0, j = 10; i < j; i++, j--) { }   // two variables
for (;;) { }                               // infinite loop (needs break)

int[] nums = { 4, 8, 15 };
for (int i = 0; i < nums.Length; i++)      // index loop over an array
    Console.WriteLine(nums[i]);
foreach (int n in nums)                    // simpler when the index is not needed
    Console.WriteLine(n);
```

**Common for-loop programs**

```csharp
// Multiplication table of 7
for (int i = 1; i <= 10; i++)
    Console.WriteLine($"7 x {i} = {7 * i}");

// Factorial of 5
long fact = 1;
for (int i = 1; i <= 5; i++) fact *= i;    // 120

// Sum of 1 to 100
int sum = 0;
for (int i = 1; i <= 100; i++) sum += i;   // 5050
```

### Nested For Loops

A loop inside a loop: the inner loop finishes all its passes for every single pass of the outer loop. Total runs = outer count × inner count.

```csharp
for (int row = 1; row <= 3; row++)          // outer: rows
{
    for (int col = 1; col <= 4; col++)      // inner: columns
    {
        Console.Write($"({row},{col}) ");
    }
    Console.WriteLine();                     // new line after each row
}
// (1,1) (1,2) (1,3) (1,4)
// (2,1) (2,2) (2,3) (2,4)
// (3,1) (3,2) (3,3) (3,4)   -> 3 x 4 = 12 runs
```

**Rule of thumb:** outer loop = rows, inner loop = columns. This is the basis of every pattern-printing program (section 10).

| Loop | Best for |
| --- | --- |
| `for` | Known number of repeats, counters, index access |
| `foreach` | Reading every item of an array or collection |
| `while` | Repeat until something changes; count unknown |
| `do-while` | Must run at least once |

## 9. Break, Continue and Goto

These are jump statements: `break` leaves the loop, `continue` skips to the next pass, and `goto` jumps to a label (use it rarely).

| Statement | Effect | Works in |
| --- | --- | --- |
| `break` | Exits the nearest loop or `switch` immediately | `for`, `foreach`, `while`, `do-while`, `switch` |
| `continue` | Skips the rest of this pass, goes to the next pass | Loops only |
| `goto label;` | Jumps to a labelled statement | Anywhere in the same method |
| `return` | Exits the whole method | Methods |

### Break

```csharp
// Stop at the first number divisible by 7
for (int i = 1; i <= 100; i++)
{
    if (i % 7 == 0)
    {
        Console.WriteLine($"Found {i}");   // Found 7
        break;                            // leave the loop
    }
}
```

In nested loops, `break` exits only the **inner** loop. To exit both, use a flag variable, put the loops in a method and `return`, or use `goto`.

### Continue

```csharp
// Print odd numbers only
for (int i = 1; i <= 10; i++)
{
    if (i % 2 == 0)
        continue;           // skip even numbers
    Console.Write(i + " "); // 1 3 5 7 9
}
```

**Careful in `while`:** if `continue` runs before the counter update, the loop never ends.

```csharp
int i = 0;
while (i < 10)
{
    i++;                    // update FIRST
    if (i % 2 == 0) continue;
    Console.Write(i + " ");
}
```

### Goto

```csharp
// Exit two nested loops at once
for (int r = 1; r <= 5; r++)
{
    for (int c = 1; c <= 5; c++)
    {
        if (r * c == 12)
            goto Done;
        Console.Write($"{r * c} ");
    }
}
Done:
Console.WriteLine("Exited both loops");

// In a switch
switch (level)
{
    case 1: Console.WriteLine("Basic"); break;
    case 2: Console.WriteLine("Pro"); goto case 1;   // also print Basic
}
```

**Why avoid goto:** it makes the flow hard to follow ("spaghetti code"). Acceptable uses: breaking out of deeply nested loops and `goto case` in a switch. You cannot jump *into* a loop or block from outside.

## 10. Pattern Printing

Every pattern is a nested loop: the outer loop picks the row, the inner loop(s) print what goes in that row. Work out "how many of what on row *i*" and the code follows.

**Method:** (1) count the rows, n; (2) for row i, write how many spaces and how many symbols; (3) turn each count into an inner loop; (4) `Console.WriteLine()` at the end of each row.

### 1. Square

```csharp
int n = 4;
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= n; j++) Console.Write("* ");
    Console.WriteLine();
}
// * * * *
// * * * *
// * * * *
// * * * *
```

### 2. Right triangle (row i has i stars)

```csharp
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= i; j++) Console.Write("* ");
    Console.WriteLine();
}
// *
// * *
// * * *
// * * * *
```

### 3. Inverted triangle (row i has n − i + 1 stars)

```csharp
for (int i = n; i >= 1; i--)
{
    for (int j = 1; j <= i; j++) Console.Write("* ");
    Console.WriteLine();
}
// * * * *
// * * *
// * *
// *
```

### 4. Number triangle

```csharp
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= i; j++) Console.Write(j + " ");
    Console.WriteLine();
}
// 1
// 1 2
// 1 2 3
// 1 2 3 4
```

Variant: print `i` instead of `j` to get 1 / 2 2 / 3 3 3 / 4 4 4 4.

### 5. Floyd's triangle (keep counting across rows)

```csharp
int k = 1;
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= i; j++) Console.Write(k++ + " ");
    Console.WriteLine();
}
// 1
// 2 3
// 4 5 6
// 7 8 9 10
```

### 6. Pyramid (n − i spaces, then 2i − 1 stars)

```csharp
for (int i = 1; i <= n; i++)
{
    for (int s = 1; s <= n - i; s++)     Console.Write(" ");
    for (int j = 1; j <= 2 * i - 1; j++) Console.Write("*");
    Console.WriteLine();
}
//    *
//   ***
//  *****
// *******
```

### 7. Diamond (pyramid + inverted pyramid)

```csharp
for (int i = 1; i <= n; i++)          // top half
{
    Console.Write(new string(' ', n - i));
    Console.WriteLine(new string('*', 2 * i - 1));
}
for (int i = n - 1; i >= 1; i--)      // bottom half
{
    Console.Write(new string(' ', n - i));
    Console.WriteLine(new string('*', 2 * i - 1));
}
```

`new string('*', count)` repeats a character: a handy shortcut once you understand the loop version.

| Pattern | Spaces on row i | Symbols on row i |
| --- | --- | --- |
| Right triangle | 0 | i |
| Inverted triangle | 0 | n − i + 1 |
| Right-aligned triangle | n − i | i |
| Pyramid | n − i | 2i − 1 |
| Inverted pyramid | i − 1 | 2(n − i) + 1 |

## 11. Section Cheat Sheet

Everything in this section on one screen.

```csharp
// VARIABLES & TYPES
int i = 10;  long l = 10L;  double d = 1.5;  float f = 1.5F;  decimal m = 9.99M;
char c = 'A';  bool b = true;  string s = "text";  var v = 42;  const double Pi = 3.14;

// INPUT / CONVERSION
string input = Console.ReadLine();
int n = int.Parse(input);              // throws on bad input
int.TryParse(input, out int safe);     // returns false on bad input
int x = (int)3.9;                      // 3 (explicit cast truncates)
string t = $"Value: {d:F2}";           // interpolation + format

// OPERATORS
7 / 2 == 3;   7 % 2 == 1;   7 / 2.0 == 3.5;
a += 1;  a++;  ++a;
&& || !        == != < > <= >=        cond ? x : y        name ?? "default"

// DECISIONS
if (a > b) { } else if (a == b) { } else { }
switch (v) { case 1: ...; break; default: ...; break; }
string r = v switch { 1 => "one", 2 or 3 => "few", _ => "many" };

// LOOPS
for (int k = 0; k < 10; k++) { }
foreach (var item in list) { }
while (cond) { }
do { } while (cond);

// JUMPS
break;       // leave loop / switch
continue;    // next iteration
goto Label;  // jump (use rarely)
return;      // leave method
```

**Handy one-liners**

| Task | Code |
| --- | --- |
| Last digit of n | `n % 10` |
| Remove last digit | `n / 10` |
| Is even? | `n % 2 == 0` |
| Max of three | `Math.Max(a, Math.Max(b, c))` |
| Power / square root | `Math.Pow(2, 10)`, `Math.Sqrt(81)` |
| Round to 2 decimals | `Math.Round(x, 2)` |
| Repeat a character | `new string('*', 5)` |
| Swap two values | `(a, b) = (b, a);` |

## 12. Section 2 Interview Questions and Checklist

Tick each item once you can write it from memory without running the code.

- [ ] Declare and initialise each primitive type; explain `var` and `const`
- [ ] Size and range of `int`, `long`, `double`, `decimal`, `char`, `bool`
- [ ] Implicit vs explicit conversion; `Parse` vs `TryParse` vs `Convert`
- [ ] Predict `i++` vs `++i` and integer division results
- [ ] Write all four practice programs (circle, feet to cm, nearest thousand, seconds)
- [ ] Height category and largest of three, with if/else-if
- [ ] A switch statement and the same logic as a switch expression
- [ ] while vs do-while with an example of each
- [ ] Multiplication table, factorial and sum of digits with loops
- [ ] break, continue and goto in a nested loop
- [ ] Print triangle, pyramid, Floyd's triangle and diamond patterns

**Interview questions (short answers)**

1. **Value of `int x = 5; int y = x++ + ++x;`?** y = 5 + 7 = 12; x ends as 7.
2. **`var` vs `dynamic`?** `var` is typed at compile time and fixed; `dynamic` is resolved at run time.
3. **`const` vs `readonly`?** `const` is fixed at compile time and implicitly static; `readonly` is set once at declaration or in the constructor.
4. **Why `decimal` for money?** It stores base-10 values exactly; `double` has binary rounding errors.
5. **What does `7 / 2` return?** 3, because both operands are `int`. Use `7 / 2.0` for 3.5.
6. **What happens on `int.MaxValue + 1`?** It wraps to `int.MinValue` unless inside `checked`, which throws `OverflowException`.
7. **`&&` vs `&`?** `&&` short-circuits (skips the right side if the left is false); `&` always evaluates both.
8. **`==` vs `=`?** `=` assigns; `==` compares.
9. **Is fall-through allowed in a C# switch?** No. Each non-empty case must end with `break`, `return`, `throw` or `goto case`.
10. **while vs do-while?** `while` checks first (may run 0 times); `do-while` checks after (runs at least once).
11. **break vs continue?** `break` exits the loop; `continue` skips to the next iteration.
12. **How do you exit nested loops at once?** A flag variable, `return` from a method, or `goto` to a label after the loops.
13. **for vs foreach?** `for` uses an index and can modify elements by index; `foreach` reads every item and cannot reassign the loop variable.
14. **Can you write an infinite loop?** Yes: `for (;;)` or `while (true)`, exited with `break` or `return`.
15. **Why is goto discouraged?** It makes control flow hard to read and maintain; structured loops and methods are clearer.
