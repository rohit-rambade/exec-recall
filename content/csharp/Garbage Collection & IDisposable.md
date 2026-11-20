# Garbage Collection & IDisposable

How .NET frees memory automatically, and how you release everything it can't: destructors, `IDisposable` and `using`.

## 1. Garbage Collection and Generations

The garbage collector (GC) is the part of the CLR that automatically frees heap memory used by objects that nothing references any more. You never call `delete` or `free` in C#.

**How a collection works**

1. **Mark:** starting from the "roots" (local variables, static fields, CPU registers), the GC finds every object still reachable.
2. **Sweep:** everything not reached is garbage, and its memory is reclaimed.
3. **Compact:** surviving objects are moved together to remove gaps, and references are updated.

**When it runs:** when Gen 0's budget fills up, when the system is low on memory, or when you call `GC.Collect()` (almost never do this).

&#91;embedded content: GC generations\]

Survivors are promoted one generation each time they live through a collection. Because most objects (temporary strings, LINQ results, DTOs) die in Gen 0, most collections are fast and touch little memory.

| Generation | Holds | Collected |
| --- | --- | --- |
| Gen 0 | Newly created objects | Very often, very fast |
| Gen 1 | Objects that survived one GC | Less often |
| Gen 2 | Long-lived objects (caches, singletons, static data) | Rarely: a "full" GC, slowest |
| LOH | Objects of 85,000 bytes or more (big arrays) | With Gen 2; not compacted by default |

```csharp
var temp = new byte[100];
Console.WriteLine(GC.GetGeneration(temp));   // 0
GC.Collect();                                 // for demos only
Console.WriteLine(GC.GetGeneration(temp));   // 1 (survived, so promoted)
Console.WriteLine(GC.GetGeneration(new byte[100_000]));  // 2 (LOH)
Console.WriteLine(GC.GetTotalMemory(false)); // bytes currently allocated
```

**Managed vs unmanaged resources:** the GC handles **managed memory** only. It does **not** close files, database connections, network sockets, or OS handles promptly. Those are **unmanaged resources**, and you release them yourself with `Dispose` (sections 3 and 4).

**Tips:** avoid needless allocations in hot loops, don't keep references longer than needed (static lists that grow forever leak memory), and don't call `GC.Collect()` in production code.

## 2. Destructors (Finalizers)

A destructor `~ClassName()` is a method the GC calls **some time before** reclaiming an object, as a last chance to release unmanaged resources. You can't call it yourself, and you can't predict when (or on app exit, whether) it will run.

```csharp
public class TempFile
{
    private readonly string _path;
    public TempFile(string path) { _path = path; File.WriteAllText(path, ""); }

    ~TempFile()                         // destructor: tilde + class name
    {
        Console.WriteLine("Finalizer running");
        try { File.Delete(_path); } catch { }   // never throw from a finalizer
    }
}
```

The compiler turns it into an override of `Object.Finalize()` that also calls the base finalizer.

**Rules**

- Only in **classes** (not structs); one per class; **no parameters, no access modifier, no return type**; can't be overloaded, inherited or called directly.
- Runs on a separate **finalizer thread**, at a time chosen by the GC.
- Objects with finalizers live longer: they survive one extra collection (they're put on a finalization queue), so they get promoted and cost more.
- Don't touch other managed objects inside it: they may already be finalized.

**Bottom line:** you almost never write a finalizer. Use `IDisposable` for deterministic cleanup, and wrap raw OS handles in `SafeHandle`, which already has a correct finalizer. Add a finalizer only as a safety net when your class directly owns an unmanaged resource.

## 3. IDisposable and the Dispose Pattern

`IDisposable` has one method, `Dispose()`, that releases resources **right now** instead of waiting for the GC. Any class that owns a file, stream, database connection, socket, timer or another `IDisposable` should implement it, and callers must call `Dispose` when finished (usually via `using`).

**Simple version (class owns only managed disposables; most common)**

```csharp
public sealed class ClaimExporter : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public ClaimExporter(string path) => _writer = new StreamWriter(path);

    public void Write(string line)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);   // .NET 7+
        _writer.WriteLine(line);
    }

    public void Dispose()
    {
        if (_disposed) return;          // safe to call more than once
        _writer.Dispose();              // release what we own
        _disposed = true;
    }
}
```

**Full dispose pattern (class may be inherited or owns unmanaged resources)**

```csharp
public class ResourceHolder : IDisposable
{
    private IntPtr _handle;                       // unmanaged resource
    private readonly MemoryStream _stream = new(); // managed disposable
    private bool _disposed;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);                // finalizer no longer needed
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            _stream.Dispose();                    // managed: only when called from Dispose()
        }
        // unmanaged: always
        if (_handle != IntPtr.Zero) { /* CloseHandle(_handle); */ _handle = IntPtr.Zero; }
        _disposed = true;
    }

    ~ResourceHolder() => Dispose(false);          // safety net if Dispose was forgotten
}
```

|  | Finalizer (`~Class`) | `Dispose()` |
| --- | --- | --- |
| Called by | The GC, at an unknown time | Your code (or `using`), immediately |
| Deterministic | No | Yes |
| Cleans up | Unmanaged resources only | Managed and unmanaged |
| Cost | Delays collection, extra GC work | Cheap |
| Typical use | Safety net | Normal cleanup |

**Rules:** make `Dispose` safe to call many times; throw `ObjectDisposedException` if the object is used after disposal; don't throw from `Dispose`; call `GC.SuppressFinalize(this)` only if a finalizer exists (or may exist in a subclass). For async cleanup there is `IAsyncDisposable` with `DisposeAsync()` and `await using`.

## 4. using Statement and using Declaration

`using` guarantees `Dispose()` is called when you're done, **even if an exception is thrown**. The using **declaration** (C# 8) is a shorter form that disposes at the end of the enclosing scope.

**What you'd have to write by hand**

```csharp
var reader = new StreamReader("claims.csv");
try
{
    Console.WriteLine(reader.ReadLine());
}
finally
{
    reader?.Dispose();      // always runs
}
```

**using statement (block form)**

```csharp
using (var reader = new StreamReader("claims.csv"))
{
    Console.WriteLine(reader.ReadLine());
}   // reader.Dispose() called here

using (var conn = new SqlConnection(cs))
using (var cmd = new SqlCommand(sql, conn))   // stacked usings
{
    conn.Open();
    cmd.ExecuteNonQuery();
}   // disposed in reverse order: cmd, then conn
```

**using declaration (C# 8): no braces**

```csharp
void ExportClaims(string path)
{
    using var writer = new StreamWriter(path);   // disposed when the method ends
    using var exporter = new ClaimExporter(path + ".bak");

    writer.WriteLine("id,amount");
    exporter.Write("backup");
}   // exporter.Dispose(), then writer.Dispose(): reverse order of declaration
```

**Async version**

```csharp
await using var stream = File.OpenWrite("report.json");   // calls DisposeAsync()
await JsonSerializer.SerializeAsync(stream, data);
```

|  | using statement | using declaration |
| --- | --- | --- |
| Syntax | `using (var x = ...) { ... }` | `using var x = ...;` |
| Disposed | At the end of its own block | At the end of the enclosing scope (method, block) |
| Nesting | Indents each level | Flat |
| Best when | You want to release early, mid-method | The resource is needed until the method ends |

**Rules:** the type must implement `IDisposable` (or `IAsyncDisposable` for `await using`); the variable is read-only inside the scope; if the expression is `null`, `Dispose` is simply skipped. Don't confuse this with the `using` **directive** that imports namespaces.

## 5. Cheat Sheet, Checklist and Interview Questions

```csharp
// GC info
int gen = GC.GetGeneration(obj);
long bytes = GC.GetTotalMemory(false);

// Destructor (rarely needed)
class Native { ~Native() { /* free unmanaged */ } }

// IDisposable
sealed class Exporter : IDisposable
{
    private readonly StreamWriter _w = new("out.txt");
    private bool _done;
    public void Dispose() { if (_done) return; _w.Dispose(); _done = true; }
}

// using: always disposes, even on exceptions
using (var e1 = new Exporter()) { }       // statement
using var e2 = new Exporter();            // declaration: end of scope
await using var s = File.OpenRead("a");   // async
```

**Checklist**

- [ ] Explain mark, sweep and compact
- [ ] Draw the three generations and the LOH
- [ ] Show an object being promoted with `GC.GetGeneration`
- [ ] Write a destructor and list its restrictions
- [ ] Implement `IDisposable` (simple and full pattern)
- [ ] Explain finalizer vs Dispose
- [ ] Rewrite try/finally as a using statement and as a using declaration

**Interview questions (short answers)**

1. **What is garbage collection?** Automatic reclamation of heap memory used by objects that are no longer reachable.
2. **What are generations?** Gen 0 (new), Gen 1 (survived once), Gen 2 (long-lived); younger generations are collected more often.
3. **Why generations?** Most objects die young, so scanning only Gen 0 most of the time is fast.
4. **What is the Large Object Heap?** A heap for objects of 85,000 bytes or more, collected with Gen 2.
5. **Should you call `GC.Collect()`?** Almost never; the GC tunes itself.
6. **What is a destructor/finalizer?** `~ClassName()`, called by the GC before reclaiming the object; non-deterministic.
7. **Can you call a destructor or overload it?** No to both.
8. **What is `IDisposable`?** An interface with `Dispose()` for deterministic release of resources.
9. **Finalize vs Dispose?** Finalize: GC-called, unpredictable, unmanaged only. Dispose: you call it, immediate, everything.
10. **What does `GC.SuppressFinalize` do?** Tells the GC not to run the finalizer because cleanup is already done.
11. **What does `using` do?** Calls `Dispose()` automatically at the end of its scope, even if an exception occurs.
12. **using statement vs using declaration?** The statement disposes at the end of its own block; the declaration (C# 8) at the end of the enclosing scope.
13. **Managed vs unmanaged resources?** Managed: memory the GC tracks. Unmanaged: OS resources like file handles and connections the GC can't release promptly.
14. **What is a memory leak in .NET?** Objects kept reachable unintentionally (static lists, unremoved event handlers), so the GC can't free them.
