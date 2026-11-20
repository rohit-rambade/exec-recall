# DateTime, Math & Regular Expressions

Dates and times (creating, formatting, adding and subtracting), the `Math` class, and pattern matching text with regular expressions.

## 1. DateTime Basics

`DateTime` is an immutable **struct** holding a date and time of day (to 100-nanosecond "ticks"), from year 1 to 9999. Like strings, its methods return new values.

### Part 1: creating

```csharp
DateTime now   = DateTime.Now;          // local date + time
DateTime utc   = DateTime.UtcNow;       // UTC: use for storing/logging
DateTime today = DateTime.Today;        // today at 00:00:00

var dob      = new DateTime(1996, 3, 15);              // y, m, d
var meeting  = new DateTime(2026, 11, 20, 14, 30, 0);  // + h, m, s
DateTime min = DateTime.MinValue;                       // 01-01-0001
DateTime def = default;                                 // same as MinValue
// new DateTime(2026, 2, 30);           // ArgumentOutOfRangeException: no 30 Feb
```

### Part 2: reading parts

```csharp
var d = new DateTime(2026, 11, 20, 14, 30, 45);
d.Year;  d.Month;  d.Day;               // 2026, 11, 20
d.Hour;  d.Minute;  d.Second;           // 14, 30, 45
d.DayOfWeek;                            // Friday (enum)
d.DayOfYear;                            // 324
d.Date;                                 // 20-11-2026 00:00:00 (time removed)
d.TimeOfDay;                            // 14:30:45 (TimeSpan)
d.Kind;                                 // Unspecified / Local / Utc

DateTime.IsLeapYear(2028);              // true
DateTime.DaysInMonth(2026, 2);          // 28
d.ToUniversalTime();  d.ToLocalTime();
```

**Related types:** `DateOnly` and `TimeOnly` (.NET 6) for a pure date (birthdays, policy start dates) or pure time; `DateTimeOffset` stores the time **plus its UTC offset** (best for timestamps across time zones); `TimeSpan` is a duration (section 3).

```csharp
DateOnly policyStart = new DateOnly(2026, 4, 1);
TimeOnly opens = new TimeOnly(9, 30);
DateTimeOffset stamp = DateTimeOffset.Now;   // e.g. 20-11-2026 14:30 +05:30
```

## 2. DateTime Formats and Parsing

### Part 1: standard formats (one letter)

Using `d = 20 Nov 2026, 14:30:45` with en-IN culture (output depends on culture):

| Code | Name | Example |
| --- | --- | --- |
| `d` | Short date | `20-11-2026` |
| `D` | Long date | `Friday, 20 November, 2026` |
| `t` | Short time | `2:30 pm` |
| `T` | Long time | `2:30:45 pm` |
| `f` | Full, short time | `Friday, 20 November, 2026 2:30 pm` |
| `g` | General | `20-11-2026 2:30 pm` |
| `M` | Month/day | `20 November` |
| `Y` | Year/month | `November 2026` |
| `o` | Round-trip ISO 8601 | `2026-11-20T14:30:45.0000000` |
| `s` | Sortable | `2026-11-20T14:30:45` |
| `u` | Universal sortable | `2026-11-20 14:30:45Z` |

### Part 2: custom formats

| Specifier | Meaning | Example |
| --- | --- | --- |
| `d` / `dd` | Day, 1–31 / 01–31 | `5` / `05` |
| `ddd` / `dddd` | Day name short / full | `Fri` / `Friday` |
| `M` / `MM` | Month 1–12 / 01–12 | `11` |
| `MMM` / `MMMM` | Month name short / full | `Nov` / `November` |
| `yy` / `yyyy` | Year 2 / 4 digits | `26` / `2026` |
| `h` / `hh` | Hour 1–12 / 01–12 | `2` / `02` |
| `H` / `HH` | Hour 0–23 / 00–23 | `14` |
| `mm` | Minutes | `30` |
| `ss` | Seconds | `45` |
| `fff` | Milliseconds | `000` |
| `tt` | AM/PM | `PM` |
| `zzz` | UTC offset | `+05:30` |

**Careful:** `MM` = month, `mm` = minutes; `HH` = 24-hour, `hh` = 12-hour.

```csharp
d.ToString("dd-MM-yyyy");              // 20-11-2026
d.ToString("dd MMM yyyy, hh:mm tt");   // 20 Nov 2026, 02:30 PM
d.ToString("dddd");                    // Friday
d.ToString("yyyyMMdd_HHmmss");         // 20261120_143045  (good for file names)
$"{d:dd/MM/yyyy}";                     // in interpolation ("/" uses the culture's date separator)
d.ToString("dd-MM-yyyy", CultureInfo.InvariantCulture);   // culture-independent
```

### Parsing strings into dates

```csharp
DateTime a = DateTime.Parse("2026-11-20");                       // flexible, culture-dependent
DateTime b = DateTime.ParseExact("20-11-2026", "dd-MM-yyyy", CultureInfo.InvariantCulture);
bool ok = DateTime.TryParseExact("31-02-2026", "dd-MM-yyyy",
              CultureInfo.InvariantCulture, DateTimeStyles.None, out DateTime c);   // false: invalid date
bool ok2 = DateTime.TryParse(userInput, out DateTime d2);       // safe for user input
```

Use `ParseExact`/`TryParseExact` whenever you know the exact format (files, APIs); `"01-02-2026"` means 1 Feb in India but 2 Jan in the US with plain `Parse`.

## 3. Date Subtraction, Date Addition and TimeSpan

### Date addition

All `Add...` methods return a **new** DateTime; pass negative numbers to go back.

```csharp
var admit = new DateTime(2026, 1, 31, 10, 0, 0);
admit.AddDays(10);          // 10-02-2026 10:00
admit.AddMonths(1);         // 28-02-2026: clamps to the last valid day
admit.AddYears(1);          // 31-01-2027
admit.AddHours(-3);         // 31-01-2026 07:00
admit.AddMinutes(90);       // 31-01-2026 11:30
admit.Add(TimeSpan.FromDays(1.5));   // 01-02-2026 22:00

DateTime policyEnd = new DateTime(2026, 4, 1).AddYears(1).AddDays(-1);   // 31-03-2027
DateTime dueDate   = DateTime.Today.AddDays(30);
```

### Date subtraction

Subtracting two dates gives a **`TimeSpan`** (a duration).

```csharp
var admitted   = new DateTime(2026, 11, 10, 9, 0, 0);
var discharged = new DateTime(2026, 11, 14, 17, 30, 0);

TimeSpan stay = discharged - admitted;          // same as discharged.Subtract(admitted)
stay.Days;              // 4      (whole-day component)
stay.Hours;             // 8      (hours component, 0–23)
stay.Minutes;           // 30
stay.TotalDays;         // 4.354... (whole duration in days)
stay.TotalHours;        // 104.5

DateTime earlier = discharged.Subtract(TimeSpan.FromHours(2));   // date minus a duration = date
int daysBetween = (new DateTime(2026, 12, 31) - DateTime.Today).Days;
```

**Age calculation (correct way)**

```csharp
int AgeOn(DateTime dob, DateTime on)
{
    int age = on.Year - dob.Year;
    if (on < dob.AddYears(age)) age--;      // birthday not reached yet this year
    return age;
}
AgeOn(new DateTime(1996, 12, 25), new DateTime(2026, 11, 20));   // 29
```

### TimeSpan

```csharp
var t1 = new TimeSpan(2, 30, 0);          // 2 h 30 m
var t2 = TimeSpan.FromMinutes(45);
var total = t1 + t2;                      // 03:15:00
total.TotalMinutes;                       // 195
total.ToString(@"hh\:mm");                // "03:15"
TimeSpan.Zero;  TimeSpan.MaxValue;
```

| Expression | Result type |
| --- | --- |
| `DateTime - DateTime` | `TimeSpan` |
| `DateTime + TimeSpan` / `DateTime - TimeSpan` | `DateTime` |
| `TimeSpan + TimeSpan` | `TimeSpan` |
| `DateTime + DateTime` | Compile error |

**Comparing dates:** `<`, `>`, `==`, `DateTime.Compare(a, b)`, `a.Date == b.Date` (ignore time).

## 4. The Math Class

`System.Math` is a static class of constants and mathematical functions. Most methods take and return `double`; many also have `decimal`, `int` and `long` overloads.

### Part 1: rounding, absolute values, min/max

```csharp
Math.Abs(-7.5);                 // 7.5
Math.Max(4, 9);  Math.Min(4, 9);   // 9, 4
Math.Clamp(120, 0, 100);        // 100: force into a range
Math.Sign(-3);                  // -1  (0 or 1 for zero/positive)

Math.Round(2.567, 2);           // 2.57
Math.Round(2.5);                // 2   banker's rounding (to even) by default!
Math.Round(3.5);                // 4
Math.Round(2.5, MidpointRounding.AwayFromZero);   // 3   the "school" way
Math.Ceiling(4.1);              // 5   always up
Math.Floor(4.9);                // 4   always down
Math.Truncate(-4.9);            // -4  drop decimals (toward zero)
Math.Floor(-4.1);               // -5  (floor goes down, even for negatives)
```

### Part 2: powers, roots, trig, logs

```csharp
Math.Pow(2, 10);                // 1024
Math.Sqrt(144);                 // 12
Math.Cbrt(27);                  // 3
Math.PI;  Math.E;               // 3.14159..., 2.71828...
Math.Sin(Math.PI / 2);          // 1   (angles in radians: deg * Math.PI / 180)
Math.Log(Math.E);               // 1   natural log
Math.Log10(1000);               // 3
Math.Log2(8);                   // 3
Math.DivRem(17, 5, out int rem);   // 3, rem = 2

// Practical
double emi(double p, double annualRate, int months)
{
    double r = annualRate / 12 / 100;
    return p * r * Math.Pow(1 + r, months) / (Math.Pow(1 + r, months) - 1);
}
double circleArea = Math.PI * Math.Pow(5, 2);   // 78.54
```

| Method | Does | Example |
| --- | --- | --- |
| `Round(x, n)` | Nearest, n decimals (banker's) | `Round(2.345, 2)` → `2.34` or `2.35`\* |
| `Ceiling` / `Floor` | Up / down to an integer | `Ceiling(1.2)` → `2` |
| `Truncate` | Remove decimals | `Truncate(1.9)` → `1` |
| `Abs`, `Sign` | Absolute value, sign |  |
| `Max`, `Min`, `Clamp` | Compare / limit |  |
| `Pow`, `Sqrt`, `Cbrt` | Power, square/cube root |  |
| `Log`, `Log10`, `Log2`, `Exp` | Logarithms, e^x |  |
| `Sin`, `Cos`, `Tan` | Trigonometry (radians) |  |

\*`double` can't store 2.345 exactly, so `Math.Round(2.345, 2)` may give 2.34. Use `decimal` (`Math.Round(2.345m, 2, MidpointRounding.AwayFromZero)` → 2.35) for money.

**Related:** `Random.Shared.Next(1, 7)` for random numbers (1–6), `int.MaxValue`, `checked` for overflow, `MathF` for `float`, `BigInteger` for huge integers.

## 5. Regular Expressions

A regular expression (regex) is a **pattern** describing text. `System.Text.RegularExpressions.Regex` uses it to validate, search, extract, replace and split strings.

### Pattern syntax

| Pattern | Matches | Example pattern | Matches |
| --- | --- | --- | --- |
| `\d` / `\D` | Digit / non-digit | `\d\d` | `42` |
| `\w` / `\W` | Word char (letter, digit, \_) / not | `\w+` | `Rohit_1` |
| `\s` / `\S` | Whitespace / not | `a\sb` | `a b` |
| `.` | Any char except newline | `c.t` | `cat`, `cot` |
| `[abc]` / `[^abc]` | One of / none of | `[aeiou]` | a vowel |
| `[a-z]`, `[A-Z0-9]` | Range | `[A-Z]{5}` | `ABCDE` |
| `^` / `$` | Start / end of string | `^\d+$` | only digits |
| `*` `+` `?` | 0+, 1+, 0 or 1 | `colou?r` | `color`, `colour` |
| `{n}` `{n,}` `{n,m}` | Exactly n, at least n, n to m | `\d{6}` | `411001` |
| `(...)` | Group / capture | `(\d{3})-(\d{4})` | captures parts |
| `(?<name>...)` | Named group | `(?<year>\d{4})` |  |
| `a\|b` | Either | `cat\|dog` |  |
| `\b` | Word boundary | `\bTPA\b` | whole word TPA |
| `\.` `\+` | Escaped special char | `\.com$` | ends with .com |

Use verbatim strings `@"..."` so backslashes aren't C# escapes.

### The main methods

```csharp
using System.Text.RegularExpressions;

// IsMatch: validate
bool pan    = Regex.IsMatch("ABCDE1234F", @"^[A-Z]{5}\d{4}[A-Z]$");     // PAN card: true
bool mobile = Regex.IsMatch("9876543210", @"^[6-9]\d{9}$");             // Indian mobile: true
bool pin    = Regex.IsMatch("411001", @"^[1-9]\d{5}$");                 // PIN code: true
bool email  = Regex.IsMatch("a@b.com", @"^[^@\s]+@[^@\s]+\.[^@\s]+$");  // simple email check

// Match: first occurrence + groups
Match m = Regex.Match("Claim CLM-2026-00123 approved", @"CLM-(?<year>\d{4})-(?<no>\d+)");
if (m.Success)
{
    Console.WriteLine(m.Value);                  // CLM-2026-00123
    Console.WriteLine(m.Groups["year"].Value);   // 2026
    Console.WriteLine(m.Groups["no"].Value);     // 00123
}

// Matches: all occurrences
foreach (Match x in Regex.Matches("Pay 120, 45 and 300", @"\d+"))
    Console.Write(x.Value + " ");                // 120 45 300

// Replace
string masked = Regex.Replace("9876543210", @"\d(?=\d{4})", "*");      // ******3210
string clean  = Regex.Replace("a   b    c", @"\s+", " ");              // "a b c"
string swap   = Regex.Replace("2026-11-20", @"(\d+)-(\d+)-(\d+)", "$3/$2/$1");   // 20/11/2026

// Split
string[] parts = Regex.Split("a1b22c333d", @"\d+");                    // a, b, c, d
```

**Options and performance**

```csharp
Regex.IsMatch("TPA", "tpa", RegexOptions.IgnoreCase);    // true
var compiled = new Regex(@"^\d{6}$", RegexOptions.Compiled);   // reuse one instance

// .NET 7+: source-generated regex (fastest, checked at compile time)
public partial class Validators
{
    [GeneratedRegex(@"^[A-Z]{5}\d{4}[A-Z]$")]
    public static partial Regex Pan();
}
Validators.Pan().IsMatch("ABCDE1234F");
```

**Tips:** anchor validation patterns with `^...$` (otherwise "abc411001xyz" passes a PIN check); test patterns on regex101.com; add a timeout (`TimeSpan.FromSeconds(1)`) for patterns run on user input; don't use regex for things with a parser (JSON, HTML, dates).

## 6. Cheat Sheet, Checklist and Interview Questions

```csharp
// DateTime
var d = new DateTime(2026, 11, 20); DateTime.Now; DateTime.UtcNow; DateTime.Today;
d.AddDays(7); d.AddMonths(-1); TimeSpan gap = d2 - d1; gap.TotalDays;
d.ToString("dd-MMM-yyyy HH:mm");
DateTime.TryParseExact(s, "dd-MM-yyyy", CultureInfo.InvariantCulture, DateTimeStyles.None, out var r);

// Math
Math.Round(2.5, MidpointRounding.AwayFromZero); Math.Ceiling(x); Math.Floor(x);
Math.Pow(2, 8); Math.Sqrt(81); Math.Abs(-3); Math.Max(a, b); Math.Clamp(v, 0, 100);

// Regex
Regex.IsMatch(s, @"^\d{6}$"); Regex.Match(s, @"(?<y>\d{4})").Groups["y"].Value;
Regex.Matches(s, @"\d+"); Regex.Replace(s, @"\s+", " "); Regex.Split(s, @"[,;]");
```

**Checklist**

- [ ] Create DateTime values and read their parts
- [ ] Format dates with standard and custom specifiers; avoid MM/mm mix-ups
- [ ] Parse dates safely with TryParseExact
- [ ] Add months across month-ends and explain the clamping
- [ ] Subtract dates and use TimeSpan Days vs TotalDays
- [ ] Calculate age correctly
- [ ] Explain banker's rounding and Ceiling/Floor/Truncate on negatives
- [ ] Write regexes for PAN, mobile and PIN; extract named groups; replace and split

**Interview questions (short answers)**

1. **Is DateTime a value or reference type?** Value type (struct), immutable.
2. **DateTime.Now vs UtcNow?** Local time vs Coordinated Universal Time; store UTC, display local.
3. **DateTime vs DateTimeOffset?** DateTimeOffset also stores the offset from UTC, so the exact instant is unambiguous.
4. **What does subtracting two dates return?** A TimeSpan.
5. **TimeSpan.Days vs TotalDays?** Days is the whole-day component; TotalDays is the entire duration as a double.
6. **Parse vs ParseExact?** Parse guesses the format from culture; ParseExact requires the given format.
7. **MM vs mm?** Month vs minutes.
8. **What does Math.Round(2.5) return?** 2: banker's rounding rounds midpoints to the nearest even number.
9. **Ceiling vs Floor vs Truncate for -4.5?** -4, -5, -4.
10. **What is a regular expression?** A pattern language for matching, extracting and transforming text.
11. **IsMatch vs Match vs Matches?** Bool check; first match with groups; all matches.
12. **Why use ^ and $?** To make the whole string match, not just part of it.
13. **How do you improve regex performance?** Reuse a Regex instance, RegexOptions.Compiled, or \[GeneratedRegex\] in .NET 7+.
