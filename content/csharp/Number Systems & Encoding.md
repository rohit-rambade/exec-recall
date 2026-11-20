# Number Systems & Character Encoding

How computers store numbers (binary, octal, hex) and text (ASCII, Unicode), and how C# exposes both.

## 1. Number Systems

A number system is defined by its **base** (radix): how many digits it uses. Computers store everything in **binary** (base 2); octal and hexadecimal are compact ways for humans to write binary.

| System | Base | Digits | C# literal | Typical use |
| --- | --- | --- | --- | --- |
| Decimal | 10 | 0–9 | `255` | Everyday numbers |
| Binary | 2 | 0, 1 | `0b1111_1111` | Bits, flags, low-level data |
| Octal | 8 | 0–7 | (no literal) | Unix file permissions (`755`) |
| Hexadecimal | 16 | 0–9, A–F | `0xFF` | Colours, memory addresses, bytes, GUIDs |

**Place values:** each position is a power of the base. Decimal 345 = 3×10² + 4×10¹ + 5×10⁰.

| Decimal | Binary | Octal | Hex |
| --- | --- | --- | --- |
| 0 | 0000 | 0 | 0 |
| 1 | 0001 | 1 | 1 |
| 2 | 0010 | 2 | 2 |
| 5 | 0101 | 5 | 5 |
| 7 | 0111 | 7 | 7 |
| 8 | 1000 | 10 | 8 |
| 9 | 1001 | 11 | 9 |
| 10 | 1010 | 12 | A |
| 11 | 1011 | 13 | B |
| 12 | 1100 | 14 | C |
| 13 | 1101 | 15 | D |
| 14 | 1110 | 16 | E |
| 15 | 1111 | 17 | F |
| 16 | 1 0000 | 20 | 10 |
| 255 | 1111 1111 | 377 | FF |

### Binary

One binary digit = a **bit**; 8 bits = a **byte** (0–255).

**Decimal → binary:** divide by 2 repeatedly and read the remainders **bottom-up**.

```text
13 / 2 = 6  remainder 1
 6 / 2 = 3  remainder 0
 3 / 2 = 1  remainder 1
 1 / 2 = 0  remainder 1     ->  13 = 1101
```

**Binary → decimal:** add the place values where there's a 1: `1101` = 8 + 4 + 0 + 1 = **13**. Places: 128 64 32 16 8 4 2 1.

### Octal

Base 8. **Binary ↔ octal:** group bits in threes from the right. `110 110` → `6 6` = octal **66** = decimal 54. **Octal → decimal:** `755` = 7×64 + 5×8 + 5 = **493**. (Unix `chmod 755`: owner rwx = 7, group r-x = 5, others r-x = 5.)

### Hexadecimal

Base 16; A=10 … F=15. **Binary ↔ hex:** group bits in fours. `1011 0110` → `B 6` = **0xB6** = 182. Two hex digits = one byte. **Hex → decimal:** `2AF` = 2×256 + 10×16 + 15 = **687**. Colours: `#FF8800` = red 255, green 136, blue 0.

### In C#

```csharp
int a = 0b1101;              // 13: binary literal (C# 7)
int b = 0xFF;                // 255: hex literal
int c = 1_000_000;           // digit separators also work: 0b1111_0000

Convert.ToString(13, 2);     // "1101"   to base 2, 8, 10 or 16
Convert.ToString(493, 8);    // "755"
Convert.ToString(255, 16);   // "ff"
255.ToString("X");           // "FF"     hex format; "X4" -> "00FF"
13.ToString("B8");           // "00001101" binary format (.NET 8+)

Convert.ToInt32("1101", 2);  // 13      from base 2, 8, 10 or 16
Convert.ToInt32("755", 8);   // 493
Convert.ToInt32("2AF", 16);  // 687
int.Parse("FF", NumberStyles.HexNumber);   // 255

// Bitwise operators work on the binary form
int flags = 0b0101;
bool bit0 = (flags & 1) != 0;   // true: test a bit
flags |= 0b1000;                // set a bit   -> 0b1101
flags &= ~0b0100;               // clear a bit -> 0b1001
int shifted = 1 << 4;           // 16
```

## 2. Character Encoding

Computers only store numbers, so every character is given a number (its **code point**), and an **encoding** decides how that number is written as bytes. Reading text with the wrong encoding produces garbage like `Ã©` or `�`.

### ASCII

ASCII (1963) uses **7 bits = 128 characters**: English letters, digits, punctuation and control codes. Each character fits in one byte.

| Range | Characters | Examples |
| --- | --- | --- |
| 0–31, 127 | Control codes | 9 = tab, 10 = line feed `\n`, 13 = carriage return `\r` |
| 32 | Space |  |
| 48–57 | Digits | `'0'` = 48 … `'9'` = 57 |
| 65–90 | Uppercase | `'A'` = 65 … `'Z'` = 90 |
| 97–122 | Lowercase | `'a'` = 97 … `'z'` = 122 |

Handy tricks: lowercase = uppercase + 32; a digit's value = `ch - '0'`.

```csharp
int code = 'A';                 // 65: char converts to int implicitly
char ch = (char)97;             // 'a'
int digit = '7' - '0';          // 7
char next = (char)('A' + 1);    // 'B'
for (char c = 'A'; c <= 'E'; c++) Console.Write(c);   // ABCDE

byte[] ascii = Encoding.ASCII.GetBytes("Hi");          // [72, 105]
Encoding.ASCII.GetBytes("₹");                          // [63] = '?': not representable
```

**Limitation:** no room for Hindi, Marathi, ₹, accents or emoji. Many incompatible 8-bit "extended ASCII" code pages tried to fill the gap.

### Unicode

Unicode gives **every character in every script one code point**, written `U+XXXX` (over 1.1 million possible; about 155,000 assigned). The first 128 code points are identical to ASCII. Unicode is the character set; UTF-8/16/32 are ways to encode it as bytes.

| Encoding | Bytes per character | Notes |
| --- | --- | --- |
| UTF-8 | 1–4 (ASCII = 1, Devanagari/₹ = 3, emoji = 4) | Web, files, JSON default; ASCII-compatible |
| UTF-16 | 2 or 4 (surrogate pair) | How .NET stores `string` and `char` in memory |
| UTF-32 | Always 4 | Simple but wasteful |

```csharp
string s = "A₹ह";                              // A, rupee sign U+20B9, Devanagari ha U+0939
Encoding.UTF8.GetBytes(s).Length;              // 7  = 1 + 3 + 3
Encoding.Unicode.GetBytes(s).Length;           // 6  = 2 + 2 + 2 (UTF-16)
Encoding.UTF32.GetBytes(s).Length;             // 12 = 4 × 3

int rupee = '₹';                               // 8377 = 0x20B9
string fromCode = char.ConvertFromUtf32(0x1F600);   // "😀"
"😀".Length;                                     // 2! one emoji = surrogate pair of two chars
"😀".EnumerateRunes().Count();                  // 1: Rune = one real code point

File.WriteAllText("out.txt", s, Encoding.UTF8);    // be explicit when writing files
string back = Encoding.UTF8.GetString(Encoding.UTF8.GetBytes(s));   // round trip
```

**Rules of thumb:** use **UTF-8** for files and network data (the .NET default for `File` and streams); remember a C# `char` is a UTF-16 code unit, not always a whole character; `string.Length` counts chars, not visible characters.

## 3. Cheat Sheet and Interview Questions

```csharp
int b = 0b1010; int h = 0x1F;                         // literals
Convert.ToString(n, 2);  Convert.ToInt32(s, 16);   // base can be 2, 8, 10 or 16
n.ToString("X2");  n.ToString("B8");                   // hex / binary format
int code = 'A'; char c = (char)66;                    // char <-> code
Encoding.UTF8.GetBytes(text); Encoding.UTF8.GetString(bytes);
```

**Checklist**

- [ ] Convert decimal ↔ binary by hand (divide-by-2 and place values)
- [ ] Convert binary ↔ octal (groups of 3) and binary ↔ hex (groups of 4)
- [ ] Use C# literals and Convert.ToString / Convert.ToInt32 with a base
- [ ] Recite ASCII codes for '0', 'A', 'a' and space
- [ ] Explain Unicode vs UTF-8 vs UTF-16 and count bytes for a Hindi string

**Interview questions (short answers)**

1. **Why do computers use binary?** Hardware has two reliable states (on/off), which map to 1 and 0.
2. **Why use hexadecimal?** It's a compact way to write binary: one hex digit = 4 bits, two = 1 byte.
3. **What is 0xFF in decimal and binary?** 255, 11111111.
4. **How do you convert an int to a binary string in C#?** `Convert.ToString(n, 2)` (or `n.ToString("B")` in .NET 8+).
5. **What is ASCII?** A 7-bit encoding of 128 English characters and control codes.
6. **What is Unicode?** A universal character set assigning a code point to every character in every script.
7. **Unicode vs UTF-8?** Unicode is the list of characters and numbers; UTF-8 is one way of storing those numbers as bytes.
8. **How does .NET store strings?** As UTF-16 code units (`char` = 2 bytes).
9. **Why can `"😀".Length` be 2?** Characters beyond U+FFFF need a surrogate pair of two UTF-16 chars.
10. **Which encoding should files use?** UTF-8 in almost all cases.
