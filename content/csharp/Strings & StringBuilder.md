# Strings & StringBuilder

Working with text: how strings are stored, every common conversion and check, formatting, classic string programs with loops, and `StringBuilder` for heavy editing.

## 1. String Basics and How String Objects Are Created

`string` (alias of `System.String`) is a **reference type** that holds a sequence of UTF-16 `char`s. It is **immutable**: every "change" creates a brand-new string object, and the original is never modified.

```csharp
string a = "Hello";              // literal
String b = "Hello";              // same type (System.String)
string empty = string.Empty;     // "" (prefer over "" for readability)
string? none = null;             // no object at all

string path  = @"C:\temp\new";              // verbatim: backslashes literal
string quote = "He said \"hi\"\n";          // escapes: \" \n \t \\
string json  = """{ "id": 1 }""";           // raw string literal (C# 11)
string name = "Rohit";
string msg   = $"Hi {name}";                 // interpolation
char first   = name[0];                      // 'R': index access (read-only)
int len      = name.Length;                  // 5
// name[0] = 'r';                            // ERROR: strings are immutable
```

### How string objects are created

```csharp
string s1 = "Pune";                 // literal: stored in the intern pool
string s2 = "Pune";                 // reuses the SAME object from the pool
Console.WriteLine(ReferenceEquals(s1, s2));   // True

string s3 = new string(new[] { 'P', 'u', 'n', 'e' });   // new object at run time
Console.WriteLine(ReferenceEquals(s1, s3));   // False
Console.WriteLine(s1 == s3);                   // True: == on strings compares TEXT

string s4 = string.Intern(s3);      // fetch the pooled copy
Console.WriteLine(ReferenceEquals(s1, s4));   // True

string s = "a";
s += "b";                           // creates a new "ab"; old "a" becomes garbage
s = s.ToUpper();                    // another new object "AB"
```

**Ways a string is created:** literal (interned at compile time), `new string(char[])` / `new string('*', 5)`, concatenation `+`, `string.Concat`/`Join`/`Format`, interpolation, `ToString()`, methods like `Substring`/`Replace` (all return **new** strings), and reading files/input.

**Why immutable?** Thread-safe sharing, safe use as dictionary keys, interning saves memory, and security (a path/connection string can't be changed after validation). The cost: repeated concatenation in loops creates many temporary objects, so use `StringBuilder` there (section 6).

**`string` is special:** it's a reference type, but `==` compares contents (operator overloaded) and it behaves like a value when passed around because it can't change.

## 2. Converting Strings

Every method below **returns a new string** (or other value); the original is untouched, so always use the return value.

### Part 1: case, trimming, padding

```csharp
string s = "  Health India TPA  ";
s.ToUpper();               // "  HEALTH INDIA TPA  "
s.ToLower();               // "  health india tpa  "
s.Trim();                  // "Health India TPA"
s.TrimStart();  s.TrimEnd();
"--abc--".Trim('-');       // "abc"
"42".PadLeft(5, '0');      // "00042"
"Name".PadRight(10) + "|"; // "Name      |"
s = s.Trim();              // remember: assign the result!
```

### Part 2: extracting and splitting

```csharp
string id = "CLM-2026-00123";
id.Substring(4);           // "2026-00123"   from index 4 to end
id.Substring(4, 4);        // "2026"         start, length
id[9..];                   // "00123"        range (C# 8)
id[^5..];                  // "00123"        last 5 chars

string[] parts = id.Split('-');                       // ["CLM", "2026", "00123"]
string[] words = "a, b,,c".Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);  // ["a","b","c"]
string joined  = string.Join(" | ", parts);           // "CLM | 2026 | 00123"
char[] chars   = "abc".ToCharArray();                 // ['a','b','c']
string back    = new string(chars);                   // "abc"
```

### Part 3: converting to and from other types

```csharp
int n = int.Parse("123");                  // throws on bad input
bool ok = int.TryParse("12a", out int m);  // false, m = 0
decimal amt = decimal.Parse("1,250.75", CultureInfo.InvariantCulture);
DateTime d = DateTime.Parse("2026-11-20");
string t = 3.5.ToString();                 // "3.5"
string c = Convert.ToString(42);           // "42"
string concat = string.Concat("Pune", "-", 411001);  // "Pune-411001"
byte[] bytes = Encoding.UTF8.GetBytes("हेलो"); // string -> bytes (see encoding tab)
string fromBytes = Encoding.UTF8.GetString(bytes);
```

| Method | Does | Example result |
| --- | --- | --- |
| `ToUpper` / `ToLower` | Change case | `"abc".ToUpper()` → `"ABC"` |
| `Trim` / `TrimStart` / `TrimEnd` | Remove whitespace (or given chars) | `" a ".Trim()` → `"a"` |
| `PadLeft` / `PadRight` | Pad to a width | `"7".PadLeft(3,'0')` → `"007"` |
| `Substring(start, len)` | Extract part | `"Hello".Substring(1,3)` → `"ell"` |
| `Split` | String → array | `"a,b".Split(',')` |
| `string.Join` | Array → string | `Join("-", ["a","b"])` → `"a-b"` |
| `ToCharArray` | String → `char[]` |  |
| `string.Concat` | Join pieces with no separator |  |

## 3. Checking Strings

### Part 1: empty checks, searching, comparing

```csharp
string.IsNullOrEmpty(null);         // true
string.IsNullOrEmpty("");           // true
string.IsNullOrWhiteSpace("   ");   // true   (best for validating user input)

string email = "Rohit@HealthIndia.com";
email.Contains("@");                                    // true
email.Contains("health", StringComparison.OrdinalIgnoreCase);   // true
email.StartsWith("Rohit");                              // true
email.EndsWith(".com");                                 // true
email.IndexOf('@');                                     // 5  (-1 if not found)
email.IndexOf("India");                                 // 12
email.LastIndexOf('.');                                 // 17
email.IndexOfAny(new[] { '#', '@' });                   // 5

"abc" == "abc";                                         // true: compares text
"abc".Equals("ABC", StringComparison.OrdinalIgnoreCase); // true
string.Compare("apple", "banana");                      // negative: apple sorts first
"b".CompareTo("a");                                     // positive
```

### Part 2: checking characters

```csharp
char.IsDigit('7');        char.IsLetter('a');       char.IsLetterOrDigit('_');   // true, true, false
char.IsUpper('A');        char.IsLower('a');        char.IsWhiteSpace(' ');      // true, true, true
char.IsPunctuation('!');  char.ToUpper('q');        // true, 'Q'

bool allDigits = "411001".All(char.IsDigit);                     // true (LINQ)
bool strongPwd = pwd.Length >= 8 && pwd.Any(char.IsUpper)
                 && pwd.Any(char.IsDigit) && pwd.Any(ch => !char.IsLetterOrDigit(ch));
```

| Need | Use |
| --- | --- |
| Null, empty or spaces? | `string.IsNullOrWhiteSpace(s)` |
| Contains text (ignore case) | `s.Contains(x, StringComparison.OrdinalIgnoreCase)` |
| Position of text | `IndexOf` / `LastIndexOf` (-1 if missing) |
| Equal ignoring case | `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)` |
| Sort order | `string.Compare(a, b)` / `CompareTo` |

**Tip:** prefer `StringComparison.Ordinal` / `OrdinalIgnoreCase` for IDs, codes, file paths and keys; use culture-aware comparison only for text shown to users. Avoid `s.ToLower() == x.ToLower()` (extra allocations, culture issues).

## 4. Formatting and Modifying Strings

### Formatting

```csharp
string name = "Rohit"; decimal amt = 25000.5m; double pct = 0.256; DateTime d = new(2026, 11, 20);

string a = string.Format("{0} claimed {1:C}", name, amt);  // composite formatting
string b = $"{name} claimed {amt:N2}";                       // interpolation: "Rohit claimed 25,000.50"
string c = $"{name,-10}|{amt,12:N0}|";                       // alignment: left 10, right 12
string e = $"{pct:P1} on {d:dd-MMM-yyyy}";                   // "25.6% on 20-Nov-2026"
string f = $"{{literal braces}}";                            // "{literal braces}"
string g = $"{(amt > 20000 ? "High" : "Low")}";              // expression inside
```

| Specifier | Meaning | `1234.567` → |
| --- | --- | --- |
| `N2` | Number with separators, 2 decimals | `1,234.57` |
| `F1` | Fixed-point | `1234.6` |
| `C` | Currency (culture) | `₹1,234.57` (en-IN) |
| `P0` | Percent (×100) | for 0.25 → `25%` |
| `E2` | Scientific | `1.23E+003` |
| `D6` | Integer with zero padding (ints only) | 42 → `000042` |
| `X` | Hexadecimal (ints only) | 255 → `FF` |
| `0.00`, `#,##0` | Custom patterns | `1234.57`, `1,235` |

**Culture:** `C` and `N` depend on the current culture; for fixed output use `amt.ToString("N2", CultureInfo.InvariantCulture)` or a specific one like `new CultureInfo("en-IN")` (gives Indian grouping 12,34,567).

### Modifying (always returns a new string)

```csharp
string s = "Health TPA";
s.Insert(6, " India");          // "Health India TPA"
s.Remove(6);                    // "Health"           (from index 6 to end)
s.Remove(0, 7);                 // "TPA"              (start, count)
s.Replace("TPA", "Insurance");  // "Health Insurance"
s.Replace(' ', '_');            // "Health_TPA"

string.Concat("a", "b", "c");   // "abc"
"ab" + "cd";                    // "abcd"
new string('-', 20);            // "--------------------"
```

## 5. Strings with For Loops: Classic Programs

A string can be read character by character like an array (`s[i]`, `s.Length`). These are the standard interview programs; try each before reading the solution.

### Part 1: reverse, palindrome, vowels

```csharp
// Reverse a string
string Reverse(string s)
{
    string result = "";
    for (int i = s.Length - 1; i >= 0; i--)
        result += s[i];                       // fine for short strings; StringBuilder for long
    return result;
}
Reverse("Pune");                              // "enuP"
// shortcuts: new string(s.Reverse().ToArray()) or Array.Reverse on a char[]

// Palindrome (two pointers, ignore case)
bool IsPalindrome(string s)
{
    s = s.ToLower();
    for (int i = 0, j = s.Length - 1; i < j; i++, j--)
        if (s[i] != s[j]) return false;
    return true;
}
IsPalindrome("Madam");                        // true

// Count vowels and consonants
int vowels = 0, consonants = 0;
foreach (char ch in "Health India".ToLower())
{
    if (!char.IsLetter(ch)) continue;
    if ("aeiou".Contains(ch)) vowels++; else consonants++;
}                                             // vowels 5, consonants 6
```

### Part 2: words

```csharp
string text = "  claim  approved for   patient ";
int wordCount = text.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;   // 4

// Capitalise each word: "claim approved" -> "Claim Approved"
string TitleCase(string s)
{
    char[] c = s.ToLower().ToCharArray();
    for (int i = 0; i < c.Length; i++)
        if (i == 0 || c[i - 1] == ' ')
            c[i] = char.ToUpper(c[i]);
    return new string(c);
}

// Reverse the order of words
string.Join(" ", "I love C#".Split(' ').Reverse());   // "C# love I"
```

### Part 3: counting and removing

```csharp
// Character frequency
var freq = new Dictionary<char, int>();
foreach (char ch in "banana")
    freq[ch] = freq.GetValueOrDefault(ch) + 1;        // b:1, a:3, n:2

// First non-repeating character
char? FirstUnique(string s)
{
    for (int i = 0; i < s.Length; i++)
        if (s.IndexOf(s[i]) == s.LastIndexOf(s[i])) return s[i];
    return null;
}
FirstUnique("swiss");                                  // 'w'

// Remove duplicate characters, keep first occurrence
string RemoveDupes(string s)
{
    string result = "";
    foreach (char ch in s)
        if (!result.Contains(ch)) result += ch;
    return result;
}
RemoveDupes("programming");                            // "progamin"

// Anagram check
bool IsAnagram(string a, string b) =>
    a.Length == b.Length && a.OrderBy(c => c).SequenceEqual(b.OrderBy(c => c));
IsAnagram("listen", "silent");                         // true
```

## 6. StringBuilder

`System.Text.StringBuilder` is a **mutable** text buffer. Appending changes it in place (it grows an internal array as needed), so building a string from many pieces doesn't create a new string object each time.

### Part 1: why it exists

```csharp
// Slow: 10,000 iterations create 10,000 intermediate strings (O(n²) copying)
string s = "";
for (int i = 0; i < 10_000; i++) s += i + ",";

// Fast: one growing buffer
var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) sb.Append(i).Append(',');
string result = sb.ToString();       // create the final string once
```

### Part 2: creating and capacity

```csharp
var sb1 = new StringBuilder();                 // default capacity (16 chars)
var sb2 = new StringBuilder("Start");          // initial text
var sb3 = new StringBuilder(capacity: 1000);   // pre-size if you know roughly how big
Console.WriteLine(sb3.Capacity);               // 1000
Console.WriteLine(sb2.Length);                 // 5
```

### Methods of StringBuilder

```csharp
var sb = new StringBuilder("Health TPA");
sb.Append(" Ltd");                    // "Health TPA Ltd"
sb.AppendLine("!");                   // adds "!" + newline
sb.AppendFormat("Id: {0:D4}", 7);     // "Id: 0007"
sb.Insert(7, "India ");               // "Health India TPA Ltd!\nId: 0007"
sb.Replace("TPA", "Insurance");       // replace all occurrences
sb.Remove(0, 7);                      // remove 7 chars from index 0
sb[0] = 'i';                          // change one char in place (allowed!)
sb.Clear();                           // empty it, keep capacity

// Methods return the same StringBuilder, so you can chain
string line = new StringBuilder().Append("A").Append('-').Append(42).ToString();   // "A-42"
```

| Method | Does |
| --- | --- |
| `Append(x)` | Add any value at the end |
| `AppendLine(x)` | Add + newline |
| `AppendFormat(fmt, args)` | Add formatted text |
| `AppendJoin(sep, items)` | Add items with a separator |
| `Insert(index, x)` | Insert at a position |
| `Remove(start, length)` | Delete a range |
| `Replace(old, new)` | Replace text or chars |
| `Clear()` | Empty the buffer |
| `ToString()` | Produce the final string |
| `Length`, `Capacity`, `this[i]` | Size, allocated space, char access |

|  | `string` | `StringBuilder` |
| --- | --- | --- |
| Mutable | No | Yes |
| Each change | New object | Same buffer |
| Best for | Few changes, fixed text, keys | Many appends/edits in loops |
| Thread-safe | Yes (immutable) | No |
| Namespace | `System` | `System.Text` |

**Rule of thumb:** for a handful of concatenations, `+` or interpolation is fine (the compiler optimises them); inside loops or when building large text (reports, CSV, HTML), use `StringBuilder`. `string.Join` is best for joining a collection.

## 7. Cheat Sheet, Checklist and Interview Questions

```csharp
string s = "  Hello World  ";
s = s.Trim();  s.ToUpper();  s.Substring(0, 5);  s[6..];
s.Split(' ');  string.Join(",", arr);  s.Replace("World", "C#");
s.Contains("lo");  s.StartsWith("He");  s.IndexOf('o');  s.LastIndexOf('o');
string.IsNullOrWhiteSpace(s);  s.Equals(t, StringComparison.OrdinalIgnoreCase);
$"{amount:N2} {date:dd-MM-yyyy} {name,-10}";
var sb = new StringBuilder(); sb.Append("x").AppendLine().Insert(0, ">"); sb.ToString();
```

**Checklist**

- [ ] Explain immutability and string interning with ReferenceEquals
- [ ] Use every conversion method and assign its result
- [ ] Validate input with IsNullOrWhiteSpace and char checks
- [ ] Format numbers and dates with interpolation and specifiers
- [ ] Write reverse, palindrome, vowel count, word count, frequency and anagram programs
- [ ] Rewrite a concatenation loop with StringBuilder

**Interview questions (short answers)**

1. **Is string a value or reference type?** Reference type, but immutable and `==` compares content.
2. **What does immutable mean?** Once created, its contents can't change; methods return new strings.
3. **What is string interning?** Reusing one shared object for identical literals via the intern pool.
4. **`string` vs `String`?** Same type; `string` is the C# alias.
5. **`==` vs `Equals` for strings?** Both compare text; `Equals` overloads accept `StringComparison` options; `ReferenceEquals` checks identity.
6. **`string.Empty` vs `""` vs `null`?** Empty and `""` are the same empty string; `null` is no object.
7. **IsNullOrEmpty vs IsNullOrWhiteSpace?** The second also treats spaces/tabs-only as empty.
8. **String vs StringBuilder?** String is immutable (new object per change); StringBuilder is a mutable buffer for many edits.
9. **When to use StringBuilder?** Building strings in loops or many append operations.
10. **Is StringBuilder thread-safe?** No.
11. **Substring vs Split?** Substring extracts by position; Split breaks by separators into an array.
12. **How do you reverse a string?** Loop from the end, `Array.Reverse` on a char array, or `new string(s.Reverse().ToArray())`.
13. **What does `$` do?** String interpolation: embeds expressions with optional format/alignment.
14. **What is a verbatim string?** `@"..."`: backslashes are literal and it can span lines.
