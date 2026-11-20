# File I/O & Serialization

Reading and writing files and folders with `System.IO`, streams for text and binary data, and saving objects as binary, JSON or XML.

## 1. Introduction to System.IO

The `System.IO` namespace contains everything for working with the file system: classes to **manage** files and folders, **streams** to read and write data, and (with `System.Text.Json` and `System.Xml.Serialization`) **serializers** that turn objects into files.

&#91;embedded content: System.IO: which class for which job\]

Pick the left column to move files around, the middle one to read or write their contents, and the right one to save whole objects.

**Path helpers** (no disk access, just string work):

```csharp
string full = Path.Combine(@"C:\Claims", "2026", "report.csv");   // C:\Claims\2026\report.csv
Path.GetFileName(full);                  // report.csv
Path.GetFileNameWithoutExtension(full);  // report
Path.GetExtension(full);                 // .csv
Path.GetDirectoryName(full);             // C:\Claims\2026
Path.ChangeExtension(full, ".json");     // ...\report.json
Path.GetTempPath();  Path.GetRandomFileName();
```

Always build paths with `Path.Combine` (it handles separators on Windows/Linux) instead of string concatenation.

## 2. File Class: Managing, Reading and Writing

`File` is a **static** class: call methods directly with a path. Ideal for one-off operations and small files (read/write the whole file in one call).

### Managing files

```csharp
string path = @"C:\Claims\report.txt";

File.Exists(path);                               // true/false
File.Create(path).Dispose();                     // create empty (Create returns an open FileStream!)
File.Copy(path, @"C:\Backup\report.txt", overwrite: true);
File.Move(path, @"C:\Claims\old.txt");           // move or rename
File.Delete(@"C:\Claims\old.txt");               // no error if missing
File.GetCreationTime(path);  File.GetLastWriteTime(path);
File.SetAttributes(path, FileAttributes.ReadOnly);
```

### Read and write

```csharp
// Write (creates or OVERWRITES)
File.WriteAllText(path, "Claim CLM-1 approved\n");
File.WriteAllLines(path, new[] { "id,amount", "C1,1200", "C2,5000" });
File.WriteAllBytes(@"C:\Claims\logo.png", imageBytes);

// Append (creates if missing)
File.AppendAllText(path, "C3,800\n");
File.AppendAllLines(path, new[] { "C4,900" });

// Read
string all      = File.ReadAllText(path);         // whole file as one string
string[] lines  = File.ReadAllLines(path);        // array of lines
byte[] bytes    = File.ReadAllBytes(@"C:\Claims\logo.png");

// Read lazily, line by line (best for big files)
foreach (string line in File.ReadLines(path).Skip(1))   // skip header
{
    var parts = line.Split(',');
    Console.WriteLine($"{parts[0]} = {parts[1]}");
}

// Async versions for apps/servers
string text = await File.ReadAllTextAsync(path);
await File.WriteAllTextAsync(path, text);
```

| Method | Use |
| --- | --- |
| `WriteAllText` / `ReadAllText` | Small text file, all at once |
| `WriteAllLines` / `ReadAllLines` | Small file as an array of lines |
| `ReadLines` | Large file: streams lines lazily |
| `AppendAllText` / `AppendAllLines` | Add to the end (logs) |
| `ReadAllBytes` / `WriteAllBytes` | Images, PDFs, any binary file |

**Errors to expect:** `FileNotFoundException`, `DirectoryNotFoundException` (folder missing), `UnauthorizedAccessException` (permissions), `IOException` (file locked by another process). Wrap file code in try/catch.

## 3. FileInfo Class and Properties

`FileInfo` is the **object** version of `File`: create it once for a path, then read its properties and call methods without repeating the path. Better when you do several operations on the same file.

```csharp
var fi = new FileInfo(@"C:\Claims\report.txt");

if (!fi.Exists)
    using (StreamWriter w = fi.CreateText()) w.WriteLine("header");

fi.Refresh();                                     // properties are cached: refresh after changes

// Properties
fi.Name;               // report.txt
fi.FullName;           // C:\Claims\report.txt
fi.Extension;          // .txt
fi.DirectoryName;      // C:\Claims
fi.Directory;          // DirectoryInfo of the folder
fi.Length;             // size in bytes
fi.CreationTime;  fi.LastWriteTime;  fi.LastAccessTime;
fi.IsReadOnly;  fi.Attributes;

// Methods
FileInfo copy = fi.CopyTo(@"C:\Backup\report.txt", overwrite: true);
fi.MoveTo(@"C:\Claims\report-2026.txt");
using StreamReader r = fi.OpenText();
using FileStream fs = fi.Open(FileMode.Open, FileAccess.Read);
fi.AppendText();  fi.Delete();

Console.WriteLine($"{fi.Name}: {fi.Length / 1024.0:F1} KB");
```

|  | `File` (static) | `FileInfo` (instance) |
| --- | --- | --- |
| Usage | `File.Copy(path, dest)` | `new FileInfo(path).CopyTo(dest)` |
| Path repeated | Every call (security check each time) | Once |
| Size, name, extension | No (use `new FileInfo(path).Length`) | Properties |
| Best for | One quick operation | Many operations on one file |

## 4. Directory, DirectoryInfo and DriveInfo

### Directory (static)

```csharp
string root = @"C:\Claims";

Directory.CreateDirectory(@"C:\Claims\2026\Nov");   // creates all missing levels; no error if exists
Directory.Exists(root);
Directory.GetCurrentDirectory();

string[] files = Directory.GetFiles(root);                              // files only
string[] csvs  = Directory.GetFiles(root, "*.csv", SearchOption.AllDirectories);   // recursive with pattern
string[] dirs  = Directory.GetDirectories(root);
foreach (var f in Directory.EnumerateFiles(root, "*.pdf"))              // lazy: better for big folders
    Console.WriteLine(Path.GetFileName(f));

Directory.Move(@"C:\Claims\Old", @"C:\Archive\Old");
Directory.Delete(@"C:\Claims\Temp");                 // must be empty...
Directory.Delete(@"C:\Claims\Temp", recursive: true); // ...or delete everything inside
```

### DirectoryInfo (instance) and its properties

```csharp
var di = new DirectoryInfo(@"C:\Claims");

di.Name;  di.FullName;  di.Parent;  di.Root;  di.Exists;
di.CreationTime;  di.LastWriteTime;  di.Attributes;

FileInfo[] pdfs = di.GetFiles("*.pdf");
DirectoryInfo[] subs = di.GetDirectories();
DirectoryInfo nov = di.CreateSubdirectory("2026\\Nov");

// Folder size and largest files (FileInfo objects make this easy)
long totalBytes = di.EnumerateFiles("*", SearchOption.AllDirectories).Sum(f => f.Length);
var biggest = di.GetFiles().OrderByDescending(f => f.Length).Take(3);

// Delete files older than 30 days
foreach (var f in di.GetFiles("*.log").Where(f => f.LastWriteTime < DateTime.Now.AddDays(-30)))
    f.Delete();
```

### DriveInfo

```csharp
foreach (DriveInfo d in DriveInfo.GetDrives())
{
    if (!d.IsReady) continue;            // e.g. empty DVD drive
    Console.WriteLine($"{d.Name} {d.DriveType} {d.DriveFormat} " +
        $"free {d.AvailableFreeSpace / 1_073_741_824.0:F1} GB of {d.TotalSize / 1_073_741_824.0:F1} GB");
}
var c = new DriveInfo("C");   c.VolumeLabel;  c.RootDirectory;
```

| Class | Kind | For |
| --- | --- | --- |
| `Directory` | Static | Quick folder operations |
| `DirectoryInfo` | Instance | Repeated operations, properties, returns `FileInfo` objects |
| `DriveInfo` | Instance | Drive type, format, free and total space |

## 5. FileStream

A **stream** is a sequence of bytes you read or write piece by piece, with a current **position**. `FileStream` is the stream for files; it works only with `byte[]`, so text and typed data use reader/writer wrappers (section 6). Streams hold an OS file handle, so always dispose them (`using`).

### Part 1: opening with FileMode, FileAccess, FileShare

```csharp
using var fs = new FileStream(@"C:\Claims\data.bin",
                              FileMode.OpenOrCreate,   // what to do if the file exists / doesn't
                              FileAccess.ReadWrite,    // what this stream may do
                              FileShare.Read);         // what OTHER processes may do meanwhile
```

| `FileMode` | If file exists | If file doesn't exist |
| --- | --- | --- |
| `CreateNew` | Throws `IOException` | Creates |
| `Create` | Truncates (empties) | Creates |
| `Open` | Opens | Throws `FileNotFoundException` |
| `OpenOrCreate` | Opens | Creates |
| `Truncate` | Opens and empties | Throws |
| `Append` | Opens, position at end (write-only) | Creates |

`FileAccess`: `Read`, `Write`, `ReadWrite`. `FileShare`: `None` (lock), `Read`, `Write`, `ReadWrite`, `Delete`.

### Part 2: writing and reading bytes

```csharp
string path = @"C:\Claims\note.txt";

using (var fs = new FileStream(path, FileMode.Create, FileAccess.Write))
{
    byte[] data = Encoding.UTF8.GetBytes("Claim approved");
    fs.Write(data, 0, data.Length);     // array, offset, count
    fs.WriteByte((byte)'!');
    fs.Flush();                          // push buffered bytes to disk (Dispose also flushes)
}

using (var fs = new FileStream(path, FileMode.Open, FileAccess.Read))
{
    byte[] buffer = new byte[fs.Length];
    int read = fs.Read(buffer, 0, buffer.Length);   // returns how many bytes were actually read
    Console.WriteLine(Encoding.UTF8.GetString(buffer, 0, read));   // Claim approved!
}
```

### Part 3: position, seeking and copying in chunks

```csharp
using var fs = new FileStream(path, FileMode.Open, FileAccess.ReadWrite);
Console.WriteLine(fs.Length);        // 15 bytes
fs.Position = 6;                     // jump to byte 6
fs.Seek(-1, SeekOrigin.End);         // 1 byte before the end
int b = fs.ReadByte();               // '!' (or -1 at end of stream)
fs.CanRead;  fs.CanWrite;  fs.CanSeek;

// Copy a large file in 80 KB chunks (never loads it all into memory)
using var src = File.OpenRead(@"C:\big.zip");
using var dst = File.Create(@"D:\big.zip");
byte[] chunk = new byte[81920];
int n;
while ((n = src.Read(chunk, 0, chunk.Length)) > 0)
    dst.Write(chunk, 0, n);
// or simply: src.CopyTo(dst);   await src.CopyToAsync(dst);
```

**Other streams share the same `Stream` API:** `MemoryStream` (bytes in RAM), `NetworkStream`, `GZipStream` (compression), `CryptoStream`, `BufferedStream`. Code written against `Stream` works with all of them.

## 6. StreamWriter / StreamReader and BinaryWriter / BinaryReader

These classes wrap a stream and handle the conversion between bytes and data for you: **StreamWriter/Reader** for **text** (with an encoding, UTF-8 by default), **BinaryWriter/Reader** for **typed binary values**.

### StreamWriter and StreamReader, part 1: writing and reading text

```csharp
string path = @"C:\Claims\claims.csv";

using (var writer = new StreamWriter(path, append: false, Encoding.UTF8))
{
    writer.WriteLine("Id,Patient,Amount");
    writer.WriteLine($"C1,Rohit,{25000m}");
    writer.Write("C2,");                    // no newline
    writer.WriteLine("Priya,4000");
}                                           // Dispose flushes and closes the file

using (var reader = new StreamReader(path))
{
    string? header = reader.ReadLine();     // "Id,Patient,Amount"
    string? line;
    while ((line = reader.ReadLine()) != null)   // null at end of file
    {
        var cols = line.Split(',');
        Console.WriteLine($"{cols[1]} claimed {cols[2]}");
    }
}
```

### Part 2: more options

```csharp
using var log = new StreamWriter(@"C:\Claims\app.log", append: true) { AutoFlush = true };   // append mode, write immediately
log.WriteLine($"{DateTime.Now:s} Started");

using var r = new StreamReader(path);
string everything = r.ReadToEnd();          // whole rest of the file
int ch = r.Peek();                          // next char without consuming (-1 at end)
bool done = r.EndOfStream;

// Wrapping an existing stream
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
using var sr = new StreamReader(fs);

// Async
await using var w = new StreamWriter("out.txt");
await w.WriteLineAsync("hello");
```

### BinaryWriter and BinaryReader, parts 1 and 2

Store values in their compact binary form (an `int` is always 4 bytes, a `decimal` 16). You must **read back in exactly the same order and types** you wrote.

```csharp
string bin = @"C:\Claims\claim.dat";

using (var bw = new BinaryWriter(File.Open(bin, FileMode.Create)))
{
    bw.Write(101);              // int     4 bytes
    bw.Write("Rohit");          // string  length prefix + UTF-8 bytes
    bw.Write(25000.50m);        // decimal 16 bytes
    bw.Write(true);             // bool    1 byte
    bw.Write(DateTime.Today.ToBinary());   // long 8 bytes
}

using (var br = new BinaryReader(File.OpenRead(bin)))
{
    int id        = br.ReadInt32();
    string name   = br.ReadString();
    decimal amt   = br.ReadDecimal();
    bool approved = br.ReadBoolean();
    DateTime date = DateTime.FromBinary(br.ReadInt64());
    Console.WriteLine($"{id} {name} {amt} {approved} {date:d}");
}
```

|  | StreamWriter / Reader | BinaryWriter / Reader |
| --- | --- | --- |
| Stores | Characters (text) | Raw typed values |
| Human-readable file | Yes | No |
| Size | Larger (numbers as digits) | Compact, fixed sizes |
| Reading | Lines / whole text, then parse | `ReadInt32`, `ReadString`... in the same order |
| Typical use | CSV, logs, reports | Custom file formats, game saves, fast caches |

## 7. Serialization: Binary, JSON and XML

**Serialization** turns an object (and the objects it references) into a format you can save or send: bytes, JSON text or XML. **Deserialization** rebuilds the object from it. Example class used below:

```csharp
public class Claim
{
    public string Id { get; set; } = "";
    public string Patient { get; set; } = "";
    public decimal Amount { get; set; }
    public DateTime SubmittedOn { get; set; }
    public List<string> Documents { get; set; } = new();
}
var claim = new Claim { Id = "C1", Patient = "Rohit", Amount = 25000m, SubmittedOn = new(2026, 11, 20), Documents = { "bill.pdf" } };
```

### Binary serialization, parts 1 and 2 (legacy)

Older courses use `[Serializable]` with `BinaryFormatter`:

```csharp
[Serializable]
public class LegacyClaim { public string Id = ""; [NonSerialized] public string TempNote = ""; }

// var bf = new BinaryFormatter();
// bf.Serialize(stream, obj);  var back = (LegacyClaim)bf.Deserialize(stream);
```

**Important:** `BinaryFormatter` is **insecure** (deserializing untrusted data can run attacker code). It was marked obsolete in .NET 5 and **removed in .NET 9**, where it throws `PlatformNotSupportedException`. Know the concepts (`[Serializable]`, `[NonSerialized]`, object graphs) for interviews and old code, but for binary today use `BinaryWriter` (section 6), or libraries such as MessagePack or protobuf-net.

### JSON serialization (System.Text.Json): the modern default

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var options = new JsonSerializerOptions
{
    WriteIndented = true,
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase      // "patient" instead of "Patient"
};

string json = JsonSerializer.Serialize(claim, options);
File.WriteAllText("claim.json", json);
/*
{
  "id": "C1",
  "patient": "Rohit",
  "amount": 25000,
  "submittedOn": "2026-11-20T00:00:00",
  "documents": [ "bill.pdf" ]
}
*/
Claim? back = JsonSerializer.Deserialize<Claim>(File.ReadAllText("claim.json"), options);
List<Claim>? many = JsonSerializer.Deserialize<List<Claim>>(jsonArray);

// Streams and async
await using var fs = File.Create("claims.json");
await JsonSerializer.SerializeAsync(fs, claims, options);
```

Customise with attributes: `[JsonPropertyName("claim_id")]`, `[JsonIgnore]`, `[JsonRequired]`, `[JsonConverter(typeof(JsonStringEnumConverter))]` (enums as text). Only **public properties** are serialized by default; a parameterless or matching constructor is needed for deserialization.

### XML serialization (XmlSerializer)

```csharp
using System.Xml.Serialization;

var xs = new XmlSerializer(typeof(Claim));
using (var w = new StreamWriter("claim.xml"))
    xs.Serialize(w, claim);
/*
<Claim>
  <Id>C1</Id>
  <Patient>Rohit</Patient>
  <Amount>25000</Amount>
  <SubmittedOn>2026-11-20T00:00:00</SubmittedOn>
  <Documents><string>bill.pdf</string></Documents>
</Claim>
*/
using (var r = new StreamReader("claim.xml"))
{
    var fromXml = (Claim)xs.Deserialize(r)!;
}
```

Rules: the class must be **public** with a **public parameterless constructor**; only public fields/properties are written; customise with `[XmlRoot("ClaimDoc")]`, `[XmlElement("PatientName")]`, `[XmlAttribute]`, `[XmlIgnore]`, `[XmlArray]`/`[XmlArrayItem]`.

|  | Binary (BinaryFormatter) | JSON | XML |
| --- | --- | --- | --- |
| Status | Removed (.NET 9), insecure | Recommended | Supported |
| Human-readable | No | Yes | Yes (verbose) |
| Size | Compact | Small | Largest |
| Typical use | Old desktop apps | Web APIs, config, files | SOAP, legacy systems, some config |
| .NET API | `BinaryFormatter` | `JsonSerializer` | `XmlSerializer` |
| What's included | All fields (private too) of `[Serializable]` types | Public properties | Public fields and properties |

## 8. Cheat Sheet and Interview Questions

```csharp
File.WriteAllText(p, s);  File.ReadAllText(p);  File.ReadLines(p);  File.AppendAllText(p, s);
File.Exists(p);  File.Copy(a, b, true);  File.Move(a, b);  File.Delete(p);
var fi = new FileInfo(p); fi.Length; fi.Extension; fi.CopyTo(b);
Directory.CreateDirectory(d); Directory.GetFiles(d, "*.csv", SearchOption.AllDirectories);
new DirectoryInfo(d).GetFiles(); DriveInfo.GetDrives();
using var fs = new FileStream(p, FileMode.OpenOrCreate, FileAccess.ReadWrite);
using var sw = new StreamWriter(p, append: true); sw.WriteLine("x");
using var sr = new StreamReader(p); while (sr.ReadLine() is { } line) { }
using var bw = new BinaryWriter(File.Create(p)); bw.Write(42);
string json = JsonSerializer.Serialize(obj); var o = JsonSerializer.Deserialize<T>(json);
new XmlSerializer(typeof(T)).Serialize(writer, obj);
```

**Checklist**

- [ ] Pick the right class for a task using the diagram in section 1
- [ ] Read and write a file with File, then with StreamReader/Writer
- [ ] Explain File vs FileInfo and Directory vs DirectoryInfo
- [ ] List folder contents recursively and compute folder size
- [ ] Explain every FileMode value
- [ ] Read a big file in chunks with FileStream
- [ ] Write and read typed data with BinaryWriter/Reader
- [ ] Serialize and deserialize an object as JSON and XML

**Interview questions (short answers)**

1. **File vs FileInfo?** File is static (pass a path each call); FileInfo is an object for one file with properties.
2. **ReadAllLines vs ReadLines?** ReadAllLines loads every line into an array; ReadLines streams them lazily.
3. **What is a stream?** A sequence of bytes read or written sequentially, with a position (files, memory, network).
4. **FileMode.Create vs CreateNew?** Create overwrites an existing file; CreateNew throws if it exists.
5. **Why use `using` with streams?** To release the file handle and flush data even if an exception occurs.
6. **StreamWriter vs BinaryWriter?** Text characters vs raw typed binary values.
7. **What does Flush do?** Writes buffered data to the underlying file immediately.
8. **What is serialization?** Converting an object into a storable/transmittable format; deserialization rebuilds it.
9. **Why is BinaryFormatter not used anymore?** It's insecure and was removed in .NET 9.
10. **JSON vs XML serialization?** JSON is lighter and the modern default; XML is more verbose, used by SOAP and legacy systems.
11. **What does XmlSerializer require?** A public class with a public parameterless constructor; serializes public members.
12. **How do you rename a JSON property?** `[JsonPropertyName("name")]` or a naming policy.
