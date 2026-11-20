---
title: C# & .NET Fundamentals
---

# C# & .NET Fundamentals — Revision Notes

## 1. Introduction to .NET

.NET is Microsoft's free, open-source developer platform for building and running apps (web, desktop, mobile, cloud, games, IoT) in languages like C#, F# and VB.NET.

**What the platform gives you**

- **A runtime** (the CLR) that runs your compiled code, manages memory and handles security and exceptions.
- **A huge class library** (BCL) for files, collections, networking, LINQ, threading, JSON and more.
- **Language compilers** (e.g. Roslyn for C#) that turn source code into IL.
- **Tooling**: the `dotnet` CLI, Visual Studio, VS Code, NuGet package manager.

**Three flavours you will hear about**

| Flavour | Runs on | Status |
| --- | --- | --- |
| .NET Framework (1.0 to 4.8.1) | Windows only | Maintenance only, no new features |
| .NET Core (1.0 to 3.1) | Windows, Linux, macOS | Replaced by .NET 5+ |
| .NET (5, 6, 7, 8, 9, 10 …) | Cross-platform | The current, unified platform |

**Key idea for interviews:** .NET is language-independent. Code written in C# and VB.NET compiles to the same Intermediate Language (IL), so the two can call each other freely.

**Remember:** .NET is the platform; C# is a language that runs on it.

## 2. CLI

"CLI" means two different things in .NET. Know both.

### a) Common Language Infrastructure (the standard)

The CLI is an open specification (ECMA-335 / ISO 23271) that describes how .NET-style code is compiled, stored and executed, independent of language and OS. The CLR is Microsoft's implementation of it; Mono is another.

Its four parts:

| Part | What it defines |
| --- | --- |
| CTS – Common Type System | The data types every .NET language shares (e.g. C# `int` = `System.Int32`). |
| CLS – Common Language Specification | The minimum rules a language must follow so libraries work across languages. |
| Metadata | Self-describing info about types, stored inside the assembly. |
| VES – Virtual Execution System | The engine that loads and runs IL (the CLR is a VES). |

### b) .NET CLI (the `dotnet` command-line tool)

The cross-platform tool used to create, build, run and publish projects without Visual Studio.

```bash
dotnet --version            # installed SDK version
dotnet new console -n HelloApp   # create a console project
cd HelloApp
dotnet build                # compile to IL (.dll)
dotnet run                  # build + run
dotnet add package Newtonsoft.Json   # add a NuGet package
dotnet test                 # run unit tests
dotnet publish -c Release   # produce deployable output
```

**Tip:** `dotnet new list` shows every project template (console, classlib, webapi, mvc, xunit …).

## 3. CLR (Common Language Runtime)

The CLR is the execution engine of .NET: it takes the IL inside your assembly, JIT-compiles it to native code, and runs it while managing memory, types, security and exceptions. Code that runs under the CLR is called **managed code**.

&#91;embedded content: C# compilation and execution flow\]

Your C# is compiled once to IL at build time; the CLR's JIT turns each method into native code the first time it is called and reuses it afterwards.

**Main responsibilities of the CLR**

| Component | Job |
| --- | --- |
| Class Loader | Loads assemblies and types into memory when needed. |
| JIT Compiler | Converts IL to native CPU instructions at run time. |
| Garbage Collector (GC) | Frees objects no longer referenced. Uses generations 0, 1, 2. |
| Type Checker / CTS | Enforces type safety; no unsafe casts. |
| Exception Manager | Structured `try / catch / finally` across languages. |
| Security | Code access checks and verification of IL. |
| Thread Support | Manages threads and synchronisation. |

**Managed vs unmanaged code**

- **Managed:** runs under the CLR (C#, VB.NET). Memory is handled by the GC.
- **Unmanaged:** runs directly on the OS (C/C++, COM). You free memory yourself; .NET talks to it via P/Invoke or COM Interop.

**Key terms**

- **IL / MSIL / CIL:** the CPU-independent intermediate code inside a .dll or .exe.
- **Assembly:** the unit of deployment (.dll or .exe) = IL + metadata + manifest.
- **AOT (Ahead-of-Time):** compile to native before running (ReadyToRun, Native AOT) for faster startup.

## 4. .NET Framework Architecture

The .NET Framework is a stack: languages compile against shared rules (CLS/CTS), use ready-made libraries (BCL and app frameworks), and everything runs on the CLR above the OS.

&#91;embedded content: .NET Framework architecture · 6 layers\]

Read it bottom-up: the OS hosts the CLR, the CLR runs the libraries, and your language code sits on top.

**Layer by layer**

| Layer | Remember this |
| --- | --- |
| CLR | Runtime engine: loads, JIT-compiles, runs and garbage-collects. |
| BCL / FCL | BCL = core types (`System`, `System.IO`, `System.Collections`). FCL = BCL + bigger frameworks (ASP.NET, ADO.NET, WinForms, WPF). |
| ADO.NET | Data access to SQL Server and other databases. |
| ASP.NET | Web apps and web services. |
| WinForms / WPF | Windows desktop UI. |
| CTS | Defines all types; value types vs reference types. |
| CLS | Subset of CTS rules so a library written in one language works from another. |

**CTS: value types vs reference types**

|  | Value types | Reference types |
| --- | --- | --- |
| Examples | `int`, `double`, `bool`, `char`, `struct`, `enum` | `class`, `string`, `object`, arrays, `interface`, `delegate` |
| Holds | The data itself | A reference (address) to data on the heap |
| Stored | Usually the stack (or inline in an object) | Heap, cleaned up by the GC |
| Copy | Copies the value | Copies the reference; both point to the same object |

**Example of CLS:** C# is case-sensitive, VB.NET is not. A public C# class with both `Add()` and `add()` is not CLS-compliant, because VB.NET could not tell them apart.

## 5. Versions of .NET Framework

.NET Framework went from 1.0 (2002) to 4.8.1 (2022), the final version. It is Windows-only and now gets only security and bug fixes; new work goes to .NET 5+.

| Version | Year | Visual Studio | Headline additions |
| --- | --- | --- | --- |
| 4.8.1 | 2022 | VS 2022 | Native Arm64 support, accessibility fixes. **Last version.** |
| 4.8 | 2019 | VS 2019 | JIT and GC improvements, high-DPI WinForms/WPF |
| 4.7.x | 2017 | VS 2017 | TLS improvements, high-DPI support |
| 4.6.x | 2015 | VS 2015 | RyuJIT 64-bit compiler, Roslyn compiler |
| 4.5.x | 2012 | VS 2012 | `async` / `await`, improved GC |
| 4.0 | 2010 | VS 2010 | Task Parallel Library (TPL), PLINQ, `dynamic`, new CLR 4 |
| 3.5 | 2007 | VS 2008 | **LINQ**, lambda expressions, ASP.NET AJAX |
| 3.0 | 2006 | — | WPF, WCF, WF (Workflow), CardSpace |
| 2.0 | 2005 | VS 2005 | **Generics**, nullable types, partial classes, CLR 2 |
| 1.1 | 2003 | VS .NET 2003 | Mobile ASP.NET, IPv6, side-by-side execution |
| 1.0 | 2002 | VS .NET | First release: CLR, BCL, ASP.NET, WinForms, ADO.NET |

**CLR versions to remember:** only three CLRs ever shipped with the Framework: CLR 1.0 (for 1.0/1.1), CLR 2.0 (for 2.0, 3.0, 3.5) and CLR 4 (for 4.0 to 4.8.1). Versions 3.0 and 3.5 added libraries on top of CLR 2.0.

**Interview one-liner:** generics arrived in 2.0, LINQ in 3.5, TPL in 4.0, async/await in 4.5.

*Release years are from memory and approximate.*

## 6. Overview of .NET Core

.NET Core (2016) is the cross-platform, open-source rewrite of .NET. From version 5 (2020) it dropped the word "Core" and became simply **.NET**, the single platform going forward.

**Why it was created**

- Run on Windows, Linux and macOS, including containers and the cloud.
- Open source on GitHub (dotnet/runtime) with community contributions.
- Much faster than .NET Framework and modular (packages via NuGet).
- Side-by-side installs: each app can target its own runtime version.

**.NET Framework vs .NET Core / .NET 5+**

|  | .NET Framework | .NET Core / .NET 5+ |
| --- | --- | --- |
| Platforms | Windows only | Windows, Linux, macOS |
| Source | Mostly closed | Open source (MIT) |
| Web | ASP.NET (Web Forms, MVC) | ASP.NET Core |
| Desktop | WinForms, WPF | WinForms, WPF (Windows only), MAUI |
| Deployment | Installed machine-wide | Side-by-side or self-contained |
| CLI | Limited | Full `dotnet` CLI |
| Future | Maintenance only | Active, a new version every November |

**Release rhythm:** a new major version every November. Even numbers are **LTS** (3 years of support); odd numbers are **STS** (2 years).

| Version | Released | Support | C# version |
| --- | --- | --- | --- |
| .NET 10 | Nov 2025 | LTS | C# 14 |
| .NET 9 | Nov 2024 | STS | C# 13 |
| .NET 8 | Nov 2023 | LTS | C# 12 |
| .NET 7 | Nov 2022 | STS | C# 11 |
| .NET 6 | Nov 2021 | LTS | C# 10 |
| .NET 5 | Nov 2020 | STS | C# 9 |
| .NET Core 3.1 | Dec 2019 | LTS | C# 8 |
| .NET Core 2.x | 2017–2018 | — | C# 7.x |
| .NET Core 1.0 | Jun 2016 | — | C# 6/7 |

**Why no ".NET Core 4"?** It was skipped to avoid confusion with .NET Framework 4.x.

*Open question:* .NET 11 (with C# 15) is scheduled for November 2026. I could not check whether it has shipped yet, so confirm on Microsoft's .NET download page.

## 7. Introducing Visual Studio

Visual Studio is Microsoft's full IDE (Integrated Development Environment) for .NET: you write, build, debug, test and publish from one tool.

**Editions**

| Edition | Cost | Who it is for |
| --- | --- | --- |
| Community | Free | Students, open source, individuals, small teams |
| Professional | Paid | Professional developers and small teams |
| Enterprise | Paid | Large teams; adds advanced testing and architecture tools |
| VS Code | Free | Lightweight cross-platform editor, with the C# Dev Kit extension |

**Solution vs project**

- **Solution (.sln / .slnx):** a container that groups one or more related projects.
- **Project (.csproj):** one buildable unit that produces one assembly (.exe or .dll).
- Example: `ShopSolution` holds `Shop.Web` (web app), `Shop.Core` (class library) and `Shop.Tests` (unit tests).

**Windows you will use daily**

- **Solution Explorer:** browse files and projects.
- **Code editor + IntelliSense:** auto-complete and inline docs.
- **Error List / Output:** compiler errors and build output.
- **Properties:** settings of the selected item.
- **NuGet Package Manager:** add libraries.

**Useful shortcuts**

| Shortcut | Action |
| --- | --- |
| F5 | Start with debugging |
| Ctrl + F5 | Start without debugging |
| F9 | Toggle breakpoint |
| F10 / F11 | Step over / step into |
| Ctrl + Shift + B | Build solution |
| Ctrl + . | Quick actions (add `using`, fix errors) |
| Ctrl + K, Ctrl + D | Format document |
| F12 | Go to definition |

**First project steps:** File > New > Project > Console App > name it > pick the .NET version > Create > press Ctrl + F5.

## 8. Introduction to C#

C# ("C sharp") is a modern, type-safe, object-oriented language created by Anders Hejlsberg at Microsoft and released in 2002 with .NET Framework 1.0.

**Key characteristics**

- **Object-oriented:** encapsulation, inheritance, polymorphism, abstraction.
- **Strongly and statically typed:** type errors are caught at compile time.
- **Managed:** memory is cleaned up by the garbage collector.
- **Multi-paradigm:** also supports functional style (lambdas, LINQ) and async programming.
- **Cross-platform** on modern .NET; used for web (ASP.NET Core), desktop, mobile (MAUI), cloud and games (Unity).

**Hello World: classic style**

```csharp
using System;                 // import a namespace

namespace HelloApp            // group related types
{
    class Program             // every method lives in a type
    {
        static void Main(string[] args)   // entry point
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

**Hello World: modern style (C# 9+ top-level statements)**

```csharp
Console.WriteLine("Hello, World!");
```

The compiler generates the `Program` class and `Main` method for you, and implicit `global using` directives bring in `System`.

**Anatomy of the classic program**

| Part | Meaning |
| --- | --- |
| `using System;` | Lets you write `Console` instead of `System.Console`. |
| `namespace HelloApp` | Logical container that avoids name clashes. |
| `class Program` | A type; C# code lives inside types. |
| `static void Main` | Entry point. `static` = no object needed; `void` = returns nothing. |
| `Console.WriteLine` | Prints a line to the console. |
| `;` and `{ }` | Statements end with `;`; blocks are wrapped in braces. |

## 9. Object, Class, Fields and Methods

A **class** is a blueprint; an **object** is a real instance built from it. **Fields** hold an object's data and **methods** define what it can do.

| Term | Plain meaning | Example |
| --- | --- | --- |
| Class | Blueprint / template | `class Car { }` |
| Object | An instance in memory, created with `new` | `Car myCar = new Car();` |
| Field | Variable inside a class (state) | `private string _model;` |
| Property | Controlled access to a field | `public string Model { get; set; }` |
| Method | Function inside a class (behaviour) | `public void Drive() { }` |
| Constructor | Special method that runs on `new` | `public Car(string model) { }` |

**Full example**

```csharp
public class Car
{
    // Fields: data stored in each object
    private string _model;
    private int _speed;

    // Constructor: runs when you write new Car(...)
    public Car(string model)
    {
        _model = model;
        _speed = 0;
    }

    // Method with a parameter, no return value
    public void Accelerate(int amount)
    {
        _speed += amount;
    }

    // Method that returns a value
    public string GetInfo()
    {
        return $"{_model} is going {_speed} km/h";
    }
}

// Using the class (e.g. inside Main)
Car car1 = new Car("Swift");   // object 1
Car car2 = new Car("Nexon");   // object 2, separate data
car1.Accelerate(40);
Console.WriteLine(car1.GetInfo());  // Swift is going 40 km/h
Console.WriteLine(car2.GetInfo());  // Nexon is going 0 km/h
```

**Points to remember**

- Each object gets its own copy of instance fields; `car1` and `car2` do not share `_speed`.
- A `static` field or method belongs to the class itself, not to any object (`Math.Sqrt()`).
- Classes are **reference types**: `Car a = b;` copies the reference, not the object.
- Default access for class members is `private`; for top-level classes it is `internal`.
- Method signature = name + parameter list. Overloading = same name, different parameters.

**Access modifiers at a glance**

| Modifier | Visible to |
| --- | --- |
| `public` | Everyone |
| `private` | Only the same class |
| `protected` | Same class and derived classes |
| `internal` | Same assembly (project) |
| `protected internal` | Same assembly or derived classes |
| `private protected` | Derived classes in the same assembly |

## 10. Namespaces

A namespace is a named container that groups related types and prevents name clashes, like folders for classes.

```csharp
namespace HealthApp.Claims        // dotted names show hierarchy
{
    public class Claim { }
}

namespace HealthApp.Billing
{
    public class Claim { }        // same name, different namespace: no clash
}
```

**Ways to use a type from another namespace (in a real file, all using lines go at the top)**

```csharp
using HealthApp.Claims;                          // 1. using directive
var c1 = new Claim();

var c2 = new HealthApp.Billing.Claim();          // 2. fully qualified name

using Bill = HealthApp.Billing;                  // 3. alias
var c3 = new Bill.Claim();

using static System.Math;                        // 4. using static (C# 6)
double r = Sqrt(16);                             //    no "Math." prefix needed
```

**Modern shortcuts**

- **File-scoped namespace (C# 10):** `namespace HealthApp.Claims;` at the top applies to the whole file, saving one level of indentation.
- **Global using (C# 10):** `global using System.Text;` in one file applies to every file in the project.
- **Implicit usings (.NET 6+):** common namespaces such as `System`, `System.Linq` and `System.Collections.Generic` are imported automatically.

**Remember**

- Namespaces are logical; assemblies (.dll) are physical. One assembly can hold many namespaces, and one namespace can span several assemblies.
- Convention: `Company.Product.Feature`, in PascalCase.
- Common built-in ones: `System`, `System.IO`, `System.Collections.Generic`, `System.Linq`, `System.Threading.Tasks`.

## 11. C# Language Tokens

A token is the smallest meaningful unit the compiler reads. Every C# statement breaks down into five kinds of token.

```csharp
int total = price * 2;   // a comment is not a token
```

Broken into tokens: `int` (keyword) · `total` (identifier) · `=` (operator) · `price` (identifier) · `*` (operator) · `2` (literal) · `;` (punctuator).

| Token type | What it is | Examples |
| --- | --- | --- |
| Keywords | Reserved words with fixed meaning | `class`, `int`, `if`, `return`, `public`, `static`, `void`, `new` |
| Contextual keywords | Keywords only in certain places; usable as names elsewhere | `var`, `async`, `await`, `get`, `set`, `value`, `record`, `yield` |
| Identifiers | Names you choose for variables, methods, classes | `total`, `GetInfo`, `_speed`, `Car` |
| Literals | Fixed values written in code | `42`, `3.14`, `'A'`, `"Hello"`, `true`, `null`, `10m`, `5L` |
| Operators | Perform operations | `+ - * / %`, `== != < >`, `&& \|\| !`, `=`, `++`, `??`, `?.`, `=>` |
| Punctuators / separators | Structure the code | `;` `,` `.` `{ }` `( )` `[ ]` `:` |

**Identifier rules**

- Start with a letter or underscore, then letters, digits or underscores (`_count`, `item2`).
- Cannot start with a digit (`2item` is invalid) or contain spaces or symbols like `-` or `#`.
- Case-sensitive: `Total` and `total` are different.
- Cannot be a keyword unless prefixed with `@`: `int @class = 5;` is valid.

**Literal suffixes worth knowing**

| Suffix | Type | Example |
| --- | --- | --- |
| none (whole number) | `int` | `100` |
| `L` | `long` | `100L` |
| `F` | `float` | `1.5F` |
| none (decimal point) | `double` | `1.5` |
| `M` | `decimal` (use for money) | `99.99M` |
| `U` | `uint` | `10U` |

Also: verbatim strings `@"C:\temp"`, interpolated strings `$"Hi {name}"`, raw strings `"""..."""` (C# 11), digit separators `1_000_000`.

**Comments** (ignored by the compiler): `// single line`, `/* multi-line */`, `/// XML doc comment`.

## 12. Versions of C#

C# has gone from 1.0 (2002) to 14 (Nov 2025, with .NET 10), and a new version now ships every November alongside .NET. Learn the bold features; they come up most in interviews.

| Version | Year | Ships with | Key features |
| --- | --- | --- | --- |
| C# 14 | 2025 | .NET 10 | Extension members, `field` keyword in properties, null-conditional assignment (`a?.B = x`) |
| C# 13 | 2024 | .NET 9 | `params` collections, new `Lock` type, `\e` escape |
| C# 12 | 2023 | .NET 8 | **Primary constructors** for classes, **collection expressions** `[1, 2, 3]`, default lambda parameters |
| C# 11 | 2022 | .NET 7 | **Raw string literals**, `required` members, list patterns, generic math |
| C# 10 | 2021 | .NET 6 | **Global usings**, **file-scoped namespaces**, `record struct` |
| C# 9 | 2020 | .NET 5 | **Records**, `init` setters, **top-level statements**, target-typed `new()` |
| C# 8 | 2019 | .NET Core 3.x | **Nullable reference types**, switch expressions, async streams, ranges `^1`, `..`, default interface methods |
| C# 7.0–7.3 | 2017–18 | VS 2017 | **Tuples**, **pattern matching**, `out var`, local functions, `ref` returns |
| C# 6 | 2015 | VS 2015 | **String interpolation** `$""`, **null-conditional** `?.`, `nameof`, expression-bodied members |
| C# 5 | 2012 | .NET 4.5 | **`async` / `await`**, caller info attributes |
| C# 4 | 2010 | .NET 4.0 | `dynamic`, named and optional parameters |
| C# 3 | 2007 | .NET 3.5 | **LINQ**, **lambda expressions**, `var`, extension methods, anonymous types, auto-properties |
| C# 2 | 2005 | .NET 2.0 | **Generics**, nullable types `int?`, iterators `yield`, partial classes, anonymous methods |
| C# 1 | 2002 | .NET 1.0 | Classes, structs, interfaces, events, delegates, properties |

**Quick memory hook:** 2 = generics, 3 = LINQ, 4 = dynamic, 5 = async, 6 = interpolation, 7 = tuples, 8 = nullable refs, 9 = records, 10 = global usings.

**Check your version:** each .NET SDK has a default C# version. Override it in the `.csproj` with `<LangVersion>latest</LangVersion>`.

*Open question:* C# 15 is expected with .NET 11 in November 2026; I could not confirm its final feature list.

## 13. C# Naming Conventions

Use PascalCase for anything public or type-like, and camelCase for locals and parameters. Microsoft's guidelines are followed almost everywhere.

| Element | Convention | Example |
| --- | --- | --- |
| Namespace | PascalCase, dotted | `HealthIndia.Claims.Services` |
| Class, struct, record | PascalCase noun | `ClaimProcessor`, `PatientRecord` |
| Interface | `I` + PascalCase | `IClaimRepository` |
| Method | PascalCase verb | `CalculatePremium()`, `GetById()` |
| Async method | ends in `Async` | `SaveClaimAsync()` |
| Property | PascalCase | `FirstName`, `TotalAmount` |
| Public field / constant | PascalCase | `MaxRetries`, `DefaultTimeout` |
| Private field | `_` + camelCase | `_claimRepository`, `_count` |
| Static private field | `s_` + camelCase (.NET runtime style) | `s_instance` |
| Local variable | camelCase | `totalAmount`, `claimId` |
| Parameter | camelCase | `void Add(int itemCount)` |
| Enum type / values | PascalCase, singular type | `enum Status { Pending, Approved }` |
| Flags enum | PascalCase, plural | `[Flags] enum Permissions` |
| Generic type parameter | `T` or `T` + name | `T`, `TKey`, `TValue` |
| Event | PascalCase verb phrase | `ClaimSubmitted` |
| Delegate / event handler | ends in `Handler` / `EventHandler` | `ClaimEventHandler` |
| Attribute class | ends in `Attribute` | `RequiredAttribute` |
| Exception class | ends in `Exception` | `InvalidClaimException` |

**Good habits**

- Choose clear names over short ones: `customerCount` beats `cc`.
- Booleans read as yes/no questions: `isActive`, `hasPolicy`, `canApprove`.
- Avoid Hungarian notation (`strName`, `intAge`); the type is already known.
- Avoid abbreviations except well-known ones (`Id`, `Xml`, `Http`). Two-letter acronyms stay upper (`IO`); longer ones use PascalCase (`HtmlParser`, not `HTMLParser`).
- One public class per file, file name = class name (`ClaimProcessor.cs`).

**Spot the mistakes**

```csharp
// Bad                          // Good
class claim_processor { }       class ClaimProcessor { }
interface ClaimRepo { }         interface IClaimRepository { }
int TotalAmount = 0;            int totalAmount = 0;      // local
private string name;            private string _name;     // field
public void getdata() { }       public void GetData() { }
```

## 14. Revision Checklist and Interview Questions

Tick each topic once you can explain it out loud without notes.

- [ ] What .NET is, and the difference between .NET Framework, .NET Core and .NET 5+
- [ ] CLI the standard (CTS, CLS, metadata, VES) vs the `dotnet` CLI tool
- [ ] CLR flow: source → IL → JIT → native; managed vs unmanaged code
- [ ] The six layers of the .NET Framework architecture
- [ ] Framework milestones: 2.0 generics, 3.5 LINQ, 4.0 TPL, 4.5 async, 4.8.1 last
- [ ] .NET Core benefits and the LTS / STS release rhythm
- [ ] Visual Studio editions, solution vs project, F5 vs Ctrl + F5
- [ ] Write Hello World both ways (classic `Main` and top-level statements)
- [ ] Write a class with fields, a constructor and two methods, then create two objects
- [ ] Namespaces: using, alias, fully qualified, file-scoped, global using
- [ ] The five token types and the identifier rules
- [ ] One headline feature for each C# version from 2 to 14
- [ ] Naming: PascalCase vs camelCase vs `_camelCase`, `I` prefix, `Async` suffix

**Common interview questions (short answers)**

1. **What is the CLR?** The .NET runtime that JIT-compiles IL to native code and manages memory, types, exceptions and security.
2. **What is IL / MSIL?** CPU-independent intermediate code produced by the C# compiler, stored in assemblies.
3. **What is JIT?** Just-In-Time compiler; converts IL to native code method by method at run time.
4. **CTS vs CLS?** CTS defines all .NET types; CLS is the subset of rules languages must follow to interoperate.
5. **What is an assembly?** The compiled unit (.dll or .exe) containing IL, metadata and a manifest.
6. **Managed vs unmanaged code?** Managed runs under the CLR with garbage collection; unmanaged runs directly on the OS.
7. **What does the garbage collector do?** Frees memory of unreachable objects, using generations 0, 1 and 2.
8. **.NET Framework vs .NET Core?** Windows-only and maintenance mode vs cross-platform, open source and actively developed.
9. **Class vs object?** A class is the blueprint; an object is an instance created with `new`.
10. **Field vs property?** A field stores data; a property wraps it with `get`/`set` for controlled access.
11. **Value type vs reference type?** Value types hold data directly (usually stack); reference types hold an address to heap data.
12. **Why use namespaces?** To organise code and avoid name clashes between types with the same name.
13. **What is `static void Main`?** The program's entry point; `static` means it runs without creating an object.
14. **Keyword vs contextual keyword?** Keywords are always reserved; contextual ones (`var`, `async`) are reserved only in certain positions.
15. **What is LTS?** Long-Term Support: even-numbered .NET releases supported for 3 years.
