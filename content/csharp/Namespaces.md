# Namespaces

How to organise types into namespaces and bring them into scope with `using`, aliases and `using static`. (Tab 1, section 10 has the short version.)

## 1. Creating Namespaces

A namespace is a named container for related types (classes, interfaces, structs, enums, delegates). It keeps names unique and code organised; the type's **full name** is `Namespace.TypeName`.

**Block-scoped (classic)**

```csharp
namespace HealthIndia.Claims
{
    public class Claim { }
    public interface IClaimValidator { }
    public enum ClaimStatus { Pending, Approved, Rejected }
}
// full name: HealthIndia.Claims.Claim
```

**File-scoped (C# 10, recommended)**: one line at the top; applies to the whole file and saves an indent level.

```csharp
namespace HealthIndia.Claims;

public class Claim { }
public class ClaimService { }
```

**Same name, different namespaces: no clash**

```csharp
namespace HealthIndia.Claims  { public class Report { } }
namespace HealthIndia.Billing { public class Report { } }
// HealthIndia.Claims.Report and HealthIndia.Billing.Report are different types
```

**Conventions**

- `Company.Product.Feature` in PascalCase: `HealthIndia.Tpa.Claims`.
- Match the folder structure: `Claims/Validation/RuleEngine.cs` → `namespace HealthIndia.Tpa.Claims.Validation;` (Visual Studio does this by default from the project's root namespace).
- Don't name a namespace the same as a class inside it (`Claims.Claims` causes confusing errors).
- Types declared outside any namespace go into the **global namespace**: fine for tiny demos, avoid in real projects.

**Rules:** a namespace can span many files and many assemblies; a file can declare several block-scoped namespaces but only one file-scoped namespace (and can't mix the two); namespaces have no access modifiers (they are always public).

## 2. Nested Namespaces

Namespaces can be nested to build a hierarchy. Writing dots in the name is the same as nesting blocks.

```csharp
// Way 1: nested blocks
namespace HealthIndia
{
    namespace Claims
    {
        namespace Validation
        {
            public class AmountRule { }
        }
    }
}

// Way 2: dotted name (same result, preferred)
namespace HealthIndia.Claims.Validation
{
    public class AmountRule { }   // full name: HealthIndia.Claims.Validation.AmountRule
}
```

**Access between levels**

```csharp
namespace HealthIndia
{
    public class Config { }

    namespace Claims
    {
        public class ClaimService
        {
            Config c = new Config();            // OK: outer namespace is visible from inner
            // AmountRule r;                    // ERROR: inner namespace needs using or full name
            Validation.AmountRule r = new();    // OK: relative name from HealthIndia.Claims
        }

        namespace Validation { public class AmountRule { } }
    }
}
```

**Remember:** code inside an inner namespace automatically sees types from its outer namespaces, but not the other way round. Importing `HealthIndia` with `using` does **not** import `HealthIndia.Claims`; each namespace is imported separately.

**Real .NET example:** `System` → `System.Collections` → `System.Collections.Generic` (`List<T>`, `Dictionary<K,V>`).

## 3. Importing Namespaces

A `using` directive imports a namespace so you can write `Claim` instead of `HealthIndia.Claims.Claim`. Without it, you must use the fully qualified name.

```csharp
using System;
using System.Collections.Generic;
using HealthIndia.Claims;              // our own namespace

var list = new List<Claim>();         // short names work
var full = new HealthIndia.Billing.Report();   // fully qualified: no using needed
```

**Ways to bring types into scope**

| Way | Syntax | Scope |
| --- | --- | --- |
| Fully qualified name | `System.IO.File.ReadAllText(...)` | That one use |
| `using` directive | `using System.IO;` | The current file |
| `global using` (C# 10) | `global using System.IO;` | Every file in the project |
| Implicit usings (.NET 6+) | `<ImplicitUsings>enable</ImplicitUsings>` in `.csproj` | Common namespaces for the project type, auto-generated |
| Using alias | `using Io = System.IO;` | The current file (section 4) |
| `using static` | `using static System.Math;` | Static members of one type (section 5) |

**Global usings in practice:** put them in one file such as `GlobalUsings.cs`:

```csharp
global using System.Text.Json;
global using HealthIndia.Claims;
global using HealthIndia.Common;
```

Implicit usings for a console app include `System`, `System.IO`, `System.Linq`, `System.Collections.Generic`, `System.Net.Http`, `System.Threading` and `System.Threading.Tasks`.

**Rules**

- `using` directives go at the top of the file, before any namespace or type declarations (or at the top of a namespace block).
- `using` imports types of **one** namespace only, not its nested namespaces.
- If two imported namespaces have a type with the same name, using the short name is ambiguous (error CS0104). Fix it with a full name or an alias.
- `global::` forces lookup from the root, for when your own namespace shadows a .NET one: `global::System.Console.WriteLine()`.
- Don't confuse the `using` **directive** (imports) with the `using` **statement** (`using var file = ...;`), which disposes resources.

## 4. Creating a Using Alias

A using alias gives a namespace or type a short local name: `using Alias = Full.Name;`. Use it to resolve name clashes or shorten long names.

**Resolve a clash**

```csharp
using Claims = HealthIndia.Claims;
using Billing = HealthIndia.Billing;

var r1 = new Claims.Report();     // HealthIndia.Claims.Report
var r2 = new Billing.Report();    // HealthIndia.Billing.Report
```

**Alias a single type**

```csharp
using WinTimer = System.Windows.Forms.Timer;
using ThreadTimer = System.Threading.Timer;   // two Timer classes, no ambiguity

using ClaimMap = System.Collections.Generic.Dictionary<string, HealthIndia.Claims.Claim>;
var map = new ClaimMap();                     // shortens a long generic type
```

**Alias any type (C# 12):** tuples, arrays and other types too.

```csharp
using Point = (int X, int Y);
using Grid = int[,];
Point p = (3, 4);
```

**Rules:** an alias applies to its file only (or the whole project with `global using Alias = ...;`); it can't reference another alias in the same group of usings; the `::` qualifier (`Claims::Report`) says explicitly "this is an alias".

## 5. Using Static

`using static` (C# 6) imports the **static members** of a single type, so you can call them without the type name.

```csharp
using static System.Console;
using static System.Math;

WriteLine("Area calculator");            // Console.WriteLine
double r = 5;
double area = PI * Pow(r, 2);            // Math.PI, Math.Pow
WriteLine($"Area = {Round(area, 2)}");   // Math.Round
```

**Also works with**

- Your own static classes: `using static HealthIndia.Common.Validators;` then `IsValidPan(pan)`.
- Enums: `using static HealthIndia.Claims.ClaimStatus;` then write `Approved` instead of `ClaimStatus.Approved`.
- Nested types and constants of the imported type.

**Rules**

- Takes a **type**, not a namespace: `using static System.Math;` works, `using static System;` doesn't.
- Imports only static members (and nested types); instance members aren't affected.
- Extension methods are not imported as plain calls this way.
- If two imported types have a static member with the same name, calls become ambiguous; use the type name.
- Use sparingly: `Max(a, b)` is less clear than `Math.Max(a, b)` in large files.

## 6. Important Points, Cheat Sheet, Checklist and Interview Questions

**Points to remember**

1. Namespaces are **logical** grouping; assemblies (.dll) are **physical**. They're independent: one assembly can hold many namespaces and vice versa.
2. Prefer file-scoped namespaces (`namespace X;`) and match them to folders.
3. `using` imports one namespace, not its children.
4. Inner namespaces see outer ones automatically; outer ones don't see inner ones.
5. Resolve clashes with a fully qualified name or an alias.
6. `global using` and implicit usings cut repeated imports across files.
7. `using static` takes a type and imports its static members and nested types.
8. `global::` starts name lookup at the root namespace.
9. Namespaces have no access modifiers; the types inside them do.

**Cheat sheet**

```csharp
global using System.Text.Json;                 // whole project
using System.Collections.Generic;              // this file
using Billing = HealthIndia.Billing;           // namespace alias
using ClaimMap = System.Collections.Generic.Dictionary<string, int>;   // type alias
using static System.Math;                      // static members of Math

namespace HealthIndia.Claims;                  // file-scoped namespace

public class ClaimService
{
    public double Score(double x) => Sqrt(x);           // via using static
    public Billing.Report Bill() => new();              // via alias
    public void Log(string m) => global::System.Console.WriteLine(m);   // root lookup
}
```

**Checklist**

- [ ] Create block-scoped and file-scoped namespaces
- [ ] Nest namespaces both ways and explain visibility between levels
- [ ] List five ways to bring a type into scope
- [ ] Fix a CS0104 ambiguity with an alias
- [ ] Use `using static` with `Math`, `Console` and an enum

**Interview questions (short answers)**

1. **What is a namespace?** A logical container that groups related types and prevents naming conflicts.
2. **Namespace vs assembly?** Namespace is logical organisation; assembly is the physical compiled file (.dll/.exe).
3. **What is a nested namespace?** A namespace inside another, written as nested blocks or dotted names.
4. **What does `using` do?** Imports a namespace's types so you can use short names in that file.
5. **Does `using System;` import `System.IO`?** No; nested namespaces need their own `using`.
6. **What is a using alias?** A short local name for a namespace or type, mainly to resolve conflicts.
7. **What is `using static`?** Imports static members of a type so they're callable without the type name.
8. **What is `global using`?** A using directive that applies to every file in the project (C# 10).
9. **What are file-scoped namespaces?** `namespace X;` declarations that apply to the whole file without braces (C# 10).
10. **What is the global namespace?** The unnamed root namespace that holds types declared outside any namespace; reached with `global::`.
11. **Can namespaces have access modifiers?** No; they're implicitly public.
12. **`using` directive vs `using` statement?** The directive imports namespaces; the statement disposes an `IDisposable` object at the end of its scope.
