# Part 14: File I/O และ Serialization
## Steps 131-140: การอ่าน/เขียนไฟล์และแปลงข้อมูล

---

## Step 131: File Operations พื้นฐาน

```csharp
using System.IO;

// ============================================
// File Class Methods
// ============================================

// เขียนไฟล์
File.WriteAllText("hello.txt", "สวัสดี ชาวโลก!");
File.WriteAllLines("list.txt", new[] { "บรรทัดที่ 1", "บรรทัดที่ 2", "บรรทัดที่ 3" });
File.WriteAllBytes("data.bin", new byte[] { 0x48, 0x65, 0x6C, 0x6C, 0x6F });

// อ่านไฟล์
string content = File.ReadAllText("hello.txt");
string[] lines = File.ReadAllLines("list.txt");
byte[] bytes = File.ReadAllBytes("data.bin");

// Append
File.AppendAllText("log.txt", $"[{DateTime.Now}] Log entry\n");

// Async versions
await File.WriteAllTextAsync("async.txt", "Async content");
string asyncContent = await File.ReadAllTextAsync("async.txt");

// ============================================
// File Info
// ============================================

var file = new FileInfo("hello.txt");
Console.WriteLine($"Name: {file.Name}");
Console.WriteLine($"Size: {file.Length} bytes");
Console.WriteLine($"Created: {file.CreationTime}");
Console.WriteLine($"Modified: {file.LastWriteTime}");
Console.WriteLine($"Extension: {file.Extension}");
Console.WriteLine($"Directory: {file.DirectoryName}");
Console.WriteLine($"Exists: {file.Exists}");
Console.WriteLine($"ReadOnly: {file.IsReadOnly}");

// ============================================
// File Operations
// ============================================

// Copy
File.Copy("source.txt", "dest.txt");
File.Copy("source.txt", "dest.txt", overwrite: true);
file.CopyTo("newlocation/dest.txt", overwrite: true);

// Move/Rename
File.Move("old.txt", "new.txt");
File.Move("old.txt", "new.txt", overwrite: true);

// Delete
File.Delete("temp.txt");
if (File.Exists("maybe.txt"))
    File.Delete("maybe.txt");

// Check existence
bool exists = File.Exists("data.txt");

// Attributes
File.SetAttributes("readonly.txt", FileAttributes.ReadOnly);
FileAttributes attrs = File.GetAttributes("file.txt");
bool isHidden = (attrs & FileAttributes.Hidden) != 0;
```

---

## Step 132: Directory Operations

```csharp
// ============================================
// Directory Class
// ============================================

// สร้าง directory
Directory.CreateDirectory("output");
Directory.CreateDirectory("data/raw/2024"); // recursive

// ลบ directory
Directory.Delete("temp");
Directory.Delete("temp_with_files", recursive: true); // ลบทั้ง subdirectories

// List contents
string[] files2 = Directory.GetFiles(".");                     // ไฟล์ใน current dir
string[] txts = Directory.GetFiles(".", "*.txt");              // .txt เท่านั้น
string[] allTxts = Directory.GetFiles(".", "*.txt", 
    SearchOption.AllDirectories); // recursive

string[] subdirs = Directory.GetDirectories(".");
string[] allDirs = Directory.GetDirectories(".", "*", 
    SearchOption.AllDirectories);

// Enumerate (lazy - ดีกว่าสำหรับ large directories)
foreach (var f in Directory.EnumerateFiles(".", "*.cs", SearchOption.AllDirectories))
    Console.WriteLine(f);

// DirectoryInfo
var dir = new DirectoryInfo(".");
Console.WriteLine($"Path: {dir.FullName}");
Console.WriteLine($"Parent: {dir.Parent?.Name}");

foreach (var f in dir.GetFiles("*.txt"))
    Console.WriteLine($"  {f.Name} ({f.Length} bytes)");

foreach (var d in dir.GetDirectories())
    Console.WriteLine($"  [{d.Name}]");

// Move directory
Directory.Move("source", "destination");

// ============================================
// Path Class
// ============================================

string fullPath = Path.Combine("users", "documents", "report.pdf");
Console.WriteLine(fullPath); // users/documents/report.pdf หรือ users\documents\report.pdf

string dir2 = Path.GetDirectoryName(fullPath)!; // users/documents
string file3 = Path.GetFileName(fullPath);       // report.pdf
string name = Path.GetFileNameWithoutExtension(fullPath); // report
string ext = Path.GetExtension(fullPath);        // .pdf
string absolute = Path.GetFullPath(fullPath);    // absolute path

// Temp files
string tempPath = Path.GetTempPath();
string tempFile = Path.GetTempFileName(); // creates actual empty file

// Random file name
string random = Path.GetRandomFileName(); // e.g., "3k5gxd2y.wf4"

// Path separators
Console.WriteLine(Path.DirectorySeparatorChar); // / on Unix, \ on Windows
Console.WriteLine(Path.PathSeparator);          // : on Unix, ; on Windows
```

---

## Step 133: Stream Operations

```csharp
// ============================================
// FileStream - Low-level file access
// ============================================

// เขียนด้วย FileStream
using (var stream = new FileStream("binary.dat", FileMode.Create, FileAccess.Write))
{
    byte[] header = System.Text.Encoding.UTF8.GetBytes("MYFILE");
    stream.Write(header);
    stream.WriteByte(0x01); // version
    
    byte[] data = System.BitConverter.GetBytes(12345);
    stream.Write(data);
}

// อ่านด้วย FileStream
using (var stream = new FileStream("binary.dat", FileMode.Open, FileAccess.Read))
{
    byte[] headerBytes = new byte[6];
    stream.Read(headerBytes, 0, 6);
    string header = System.Text.Encoding.UTF8.GetString(headerBytes);
    
    int version = stream.ReadByte();
    
    byte[] numBytes = new byte[4];
    stream.Read(numBytes);
    int number = System.BitConverter.ToInt32(numBytes);
    
    Console.WriteLine($"Header: {header}, Version: {version}, Number: {number}");
}

// ============================================
// StreamReader/Writer - Text files
// ============================================

// เขียน
using (var writer = new StreamWriter("text.txt", append: false, 
    System.Text.Encoding.UTF8))
{
    writer.WriteLine("บรรทัดแรก");
    writer.WriteLine("บรรทัดสอง");
    writer.Write("ไม่มี newline");
}

// อ่านทีละบรรทัด
using (var reader = new StreamReader("text.txt", System.Text.Encoding.UTF8))
{
    string? line;
    int lineNum = 0;
    while ((line = reader.ReadLine()) != null)
    {
        lineNum++;
        Console.WriteLine($"{lineNum:D3}: {line}");
    }
}

// ============================================
// MemoryStream - in-memory stream
// ============================================

// ใช้เมื่อต้องการทำงานกับข้อมูลใน memory แบบ stream
using var ms = new MemoryStream();
using var writer2 = new StreamWriter(ms, leaveOpen: true);

writer2.WriteLine("Line 1");
writer2.WriteLine("Line 2");
writer2.Flush();

// อ่านจาก beginning
ms.Position = 0;
using var reader2 = new StreamReader(ms);
string allContent = reader2.ReadToEnd();
Console.WriteLine(allContent);

// Convert to byte array
byte[] bytes2 = ms.ToArray();

// ============================================
// Large File Processing
// ============================================

async Task ProcessLargeFileAsync(string inputPath, string outputPath)
{
    using var input = File.OpenRead(inputPath);
    using var output = File.OpenWrite(outputPath);
    
    // Process in 64KB chunks
    const int bufferSize = 64 * 1024;
    var buffer = new byte[bufferSize];
    int bytesRead;
    long totalBytes = 0;
    
    while ((bytesRead = await input.ReadAsync(buffer)) > 0)
    {
        // Transform data (e.g., simple XOR encryption)
        for (int i = 0; i < bytesRead; i++)
            buffer[i] ^= 0xFF;
        
        await output.WriteAsync(buffer.AsMemory(0, bytesRead));
        totalBytes += bytesRead;
    }
    
    Console.WriteLine($"Processed {totalBytes:N0} bytes");
}
```

---

## Step 134: JSON Serialization

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// ============================================
// System.Text.Json (built-in, fast)
// ============================================

public record Person2(string Name, int Age, string Email);
public class Company
{
    public string Name { get; set; } = string.Empty;
    public List<Person2> Employees { get; set; } = new();
    public Address2 HQ { get; set; } = new();
}
public class Address2
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string Country { get; set; } = "Thailand";
}

// Serialize
var company = new Company
{
    Name = "Acme Corp",
    Employees = new()
    {
        new("สมชาย", 30, "somchai@acme.com"),
        new("สมหญิง", 28, "somying@acme.com")
    },
    HQ = new Address2 { Street = "ถนนสุขุมวิท", City = "กรุงเทพ" }
};

string json = JsonSerializer.Serialize(company);
Console.WriteLine(json);

// Pretty print
var options = new JsonSerializerOptions { WriteIndented = true };
string prettyJson = JsonSerializer.Serialize(company, options);
Console.WriteLine(prettyJson);

// Deserialize
Company? loaded = JsonSerializer.Deserialize<Company>(json);
Console.WriteLine($"Company: {loaded?.Name}, Employees: {loaded?.Employees.Count}");

// ============================================
// JSON Options
// ============================================

var jsonOptions = new JsonSerializerOptions
{
    WriteIndented = true,
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase, // camelCase output
    PropertyNameCaseInsensitive = true, // case-insensitive deserialize
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    NumberHandling = JsonNumberHandling.AllowReadingFromString,
    Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping,
};

// ============================================
// JSON Attributes
// ============================================

public class User4
{
    [JsonPropertyName("user_id")]
    public int Id { get; set; }
    
    [JsonPropertyName("full_name")]
    public string Name { get; set; } = string.Empty;
    
    [JsonIgnore]
    public string PasswordHash { get; set; } = string.Empty;
    
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? Nickname { get; set; }
    
    [JsonConverter(typeof(JsonStringEnumConverter))]
    public UserRole Role { get; set; }
}

public enum UserRole { Admin, User, Moderator }

// ============================================
// Custom JSON Converter
// ============================================

public class DateOnlyConverter : JsonConverter<DateOnly>
{
    public override DateOnly Read(ref Utf8JsonReader reader, 
        Type typeToConvert, JsonSerializerOptions options)
        => DateOnly.Parse(reader.GetString()!);
    
    public override void Write(Utf8JsonWriter writer, 
        DateOnly value, JsonSerializerOptions options)
        => writer.WriteStringValue(value.ToString("yyyy-MM-dd"));
}

// ============================================
// Serialize to File
// ============================================

await using var fileStream = File.OpenWrite("company.json");
await JsonSerializer.SerializeAsync(fileStream, company, jsonOptions);

await using var readStream = File.OpenRead("company.json");
var fromFile = await JsonSerializer.DeserializeAsync<Company>(readStream, jsonOptions);
```

---

## Step 135: CSV Processing

```csharp
// ============================================
// CSV Reading/Writing
// ============================================

public class CsvHelper
{
    public static async IAsyncEnumerable<Dictionary<string, string>> ReadCsvAsync(
        string filePath,
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken ct = default)
    {
        using var reader = new StreamReader(filePath);
        
        string? headerLine = await reader.ReadLineAsync(ct);
        if (headerLine == null) yield break;
        
        string[] headers = ParseCsvLine(headerLine);
        
        string? line;
        while ((line = await reader.ReadLineAsync(ct)) != null)
        {
            string[] values = ParseCsvLine(line);
            var record = new Dictionary<string, string>();
            
            for (int i = 0; i < Math.Min(headers.Length, values.Length); i++)
                record[headers[i]] = values[i];
            
            yield return record;
        }
    }
    
    private static string[] ParseCsvLine(string line)
    {
        var fields = new List<string>();
        bool inQuotes = false;
        var current = new System.Text.StringBuilder();
        
        for (int i = 0; i < line.Length; i++)
        {
            char c = line[i];
            
            if (c == '"')
            {
                if (inQuotes && i + 1 < line.Length && line[i + 1] == '"')
                {
                    current.Append('"');
                    i++;
                }
                else
                {
                    inQuotes = !inQuotes;
                }
            }
            else if (c == ',' && !inQuotes)
            {
                fields.Add(current.ToString());
                current.Clear();
            }
            else
            {
                current.Append(c);
            }
        }
        
        fields.Add(current.ToString());
        return fields.ToArray();
    }
    
    public static async Task WriteCsvAsync<T>(
        string filePath,
        IEnumerable<T> records,
        bool includeHeaders = true)
    {
        var props = typeof(T).GetProperties();
        
        await using var writer = new StreamWriter(filePath);
        
        if (includeHeaders)
        {
            await writer.WriteLineAsync(string.Join(",", 
                props.Select(p => EscapeCsvField(p.Name))));
        }
        
        foreach (var record in records)
        {
            var values = props.Select(p => EscapeCsvField(p.GetValue(record)?.ToString() ?? ""));
            await writer.WriteLineAsync(string.Join(",", values));
        }
    }
    
    private static string EscapeCsvField(string field)
    {
        if (field.Contains(',') || field.Contains('"') || field.Contains('\n'))
            return $"\"{field.Replace("\"", "\"\"")}\"";
        return field;
    }
}

// ใช้งาน
var students2 = new[]
{
    new { Name = "อลิส", Age = 22, GPA = 3.8, Major = "CS" },
    new { Name = "บ็อบ", Age = 20, GPA = 3.2, Major = "IT" },
    new { Name = "แครอล, Jr.", Age = 23, GPA = 3.9, Major = "CS" },
};

// Write CSV
await CsvHelper.WriteCsvAsync("students.csv", students2);

// Read CSV
await foreach (var record in CsvHelper.ReadCsvAsync("students.csv"))
{
    Console.WriteLine($"Name: {record["Name"]}, GPA: {record["GPA"]}");
}
```

---

## Step 136: XML Handling

```csharp
using System.Xml;
using System.Xml.Serialization;
using System.Xml.Linq;

// ============================================
// XML Serialization
// ============================================

[XmlRoot("Catalog")]
public class BookCatalog
{
    [XmlElement("Book")]
    public List<Book2> Books { get; set; } = new();
}

public class Book2
{
    [XmlAttribute("id")]
    public int Id { get; set; }
    
    [XmlElement("Title")]
    public string Title { get; set; } = string.Empty;
    
    [XmlElement("Author")]
    public string Author { get; set; } = string.Empty;
    
    [XmlElement("Price")]
    public decimal Price { get; set; }
    
    [XmlIgnore]
    public string InternalCode { get; set; } = string.Empty;
}

var catalog = new BookCatalog
{
    Books = new()
    {
        new Book2 { Id = 1, Title = "Clean Code", Author = "Robert Martin", Price = 599 },
        new Book2 { Id = 2, Title = "Design Patterns", Author = "Gang of Four", Price = 799 },
    }
};

// Serialize to XML
var serializer = new XmlSerializer(typeof(BookCatalog));
var settings = new XmlWriterSettings { Indent = true, Encoding = System.Text.Encoding.UTF8 };

using (var writer3 = XmlWriter.Create("catalog.xml", settings))
    serializer.Serialize(writer3, catalog);

// Deserialize
using (var reader3 = XmlReader.Create("catalog.xml"))
{
    var loaded2 = (BookCatalog?)serializer.Deserialize(reader3);
    Console.WriteLine($"Books: {loaded2?.Books.Count}");
}

// ============================================
// LINQ to XML (Modern approach)
// ============================================

var xml = XDocument.Load("catalog.xml");

// Query
var expensiveBooks = xml.Descendants("Book")
    .Where(b => decimal.Parse(b.Element("Price")?.Value ?? "0") > 600)
    .Select(b => new {
        Id = (int)b.Attribute("id")!,
        Title = (string)b.Element("Title")!,
        Price = decimal.Parse(b.Element("Price")!.Value)
    });

foreach (var book in expensiveBooks)
    Console.WriteLine($"[{book.Id}] {book.Title}: {book.Price:C}");

// Modify
foreach (var book in xml.Descendants("Book"))
{
    var price = decimal.Parse(book.Element("Price")!.Value);
    book.Element("Price")!.Value = (price * 1.1m).ToString("F0"); // 10% increase
}

xml.Save("catalog_updated.xml");
```

---

## Step 137: Configuration Files

```csharp
// ============================================
// appsettings.json Pattern
// ============================================

// appsettings.json
/*
{
    "Database": {
        "ConnectionString": "Server=localhost;Database=mydb",
        "CommandTimeout": 30,
        "MaxRetries": 3
    },
    "Cache": {
        "DefaultDuration": "00:30:00",
        "MaxItems": 1000
    },
    "Logging": {
        "Level": "Information"
    }
}
*/

public class DatabaseConfig
{
    public string ConnectionString { get; set; } = string.Empty;
    public int CommandTimeout { get; set; } = 30;
    public int MaxRetries { get; set; } = 3;
}

public class CacheConfig
{
    public TimeSpan DefaultDuration { get; set; } = TimeSpan.FromMinutes(30);
    public int MaxItems { get; set; } = 1000;
}

public class AppConfig
{
    public DatabaseConfig Database { get; set; } = new();
    public CacheConfig Cache { get; set; } = new();
    public string LogLevel { get; set; } = "Information";
}

// Simple JSON config loader
public class ConfigLoader
{
    public static T Load<T>(string path) where T : new()
    {
        if (!File.Exists(path))
            return new T();
        
        string json = File.ReadAllText(path);
        return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        }) ?? new T();
    }
    
    public static void Save<T>(T config, string path)
    {
        string json = JsonSerializer.Serialize(config, new JsonSerializerOptions
        {
            WriteIndented = true
        });
        File.WriteAllText(path, json);
    }
}
```

---

## Step 138: Compression

```csharp
using System.IO.Compression;

// ============================================
// GZip Compression
// ============================================

async Task<byte[]> CompressAsync(byte[] data)
{
    using var output = new MemoryStream();
    await using var gzip = new GZipStream(output, CompressionLevel.Optimal);
    await gzip.WriteAsync(data);
    return output.ToArray();
}

async Task<byte[]> DecompressAsync(byte[] compressed)
{
    using var input = new MemoryStream(compressed);
    await using var gzip = new GZipStream(input, CompressionMode.Decompress);
    using var output = new MemoryStream();
    await gzip.CopyToAsync(output);
    return output.ToArray();
}

// ใช้งาน
string text = string.Concat(Enumerable.Repeat("Hello, World! ", 1000));
byte[] original = System.Text.Encoding.UTF8.GetBytes(text);
byte[] compressed = await CompressAsync(original);
byte[] decompressed = await DecompressAsync(compressed);

Console.WriteLine($"Original: {original.Length:N0} bytes");
Console.WriteLine($"Compressed: {compressed.Length:N0} bytes ({100.0 * compressed.Length / original.Length:F1}%)");
Console.WriteLine($"Decompressed matches: {original.SequenceEqual(decompressed)}");

// ============================================
// ZIP Archive
// ============================================

// สร้าง ZIP
using (var zip = ZipFile.Open("archive.zip", ZipArchiveMode.Create))
{
    zip.CreateEntryFromFile("file1.txt", "docs/file1.txt");
    zip.CreateEntryFromFile("file2.txt", "docs/file2.txt");
    
    // Add entry from memory
    var entry = zip.CreateEntry("readme.txt");
    using var stream = entry.Open();
    using var writer4 = new StreamWriter(stream);
    writer4.WriteLine("This is the readme");
}

// แตก ZIP
ZipFile.ExtractToDirectory("archive.zip", "extracted");

// อ่าน ZIP entries โดยไม่แตก
using (var zip2 = ZipFile.OpenRead("archive.zip"))
{
    foreach (var entry in zip2.Entries)
    {
        Console.WriteLine($"{entry.FullName}: {entry.Length:N0} bytes");
        
        using var stream = entry.Open();
        using var reader4 = new StreamReader(stream);
        Console.WriteLine(reader4.ReadToEnd());
    }
}
```

---

## Step 139: File Watching

```csharp
// ============================================
// FileSystemWatcher
// ============================================

public class DirectoryMonitor : IDisposable
{
    private readonly FileSystemWatcher _watcher;
    private readonly Action<string> _onChanged;
    
    public DirectoryMonitor(string path, string filter, Action<string> onChanged)
    {
        _onChanged = onChanged;
        
        _watcher = new FileSystemWatcher(path, filter)
        {
            NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.FileName | NotifyFilters.Size,
            IncludeSubdirectories = true,
            EnableRaisingEvents = true
        };
        
        _watcher.Created += OnEvent;
        _watcher.Changed += OnEvent;
        _watcher.Deleted += OnEvent;
        _watcher.Renamed += OnRenamed;
        _watcher.Error += OnError;
    }
    
    private void OnEvent(object sender, FileSystemEventArgs e)
        => _onChanged($"[{e.ChangeType}] {e.FullPath}");
    
    private void OnRenamed(object sender, RenamedEventArgs e)
        => _onChanged($"[Renamed] {e.OldFullPath} -> {e.FullPath}");
    
    private void OnError(object sender, ErrorEventArgs e)
        => Console.Error.WriteLine($"Watcher error: {e.GetException().Message}");
    
    public void Dispose() => _watcher.Dispose();
}

// ใช้งาน
Directory.CreateDirectory("watched");

using var monitor = new DirectoryMonitor("watched", "*.txt", change =>
    Console.WriteLine($"File changed: {change}"));

// ทดสอบ
File.WriteAllText("watched/test1.txt", "content");
await Task.Delay(500);
File.WriteAllText("watched/test1.txt", "updated content");
await Task.Delay(500);
File.Delete("watched/test1.txt");
await Task.Delay(500);
```

---

## Step 140: Complete File System Manager

```csharp
// ============================================
// File System Manager
// ============================================

public class FileSystemManager
{
    private readonly string _basePath;
    
    public FileSystemManager(string basePath)
    {
        _basePath = Path.GetFullPath(basePath);
        Directory.CreateDirectory(_basePath);
    }
    
    private string ResolvePath(string relativePath)
    {
        var fullPath = Path.GetFullPath(Path.Combine(_basePath, relativePath));
        
        // Security: prevent directory traversal
        if (!fullPath.StartsWith(_basePath))
            throw new UnauthorizedAccessException("Access outside base path not allowed");
        
        return fullPath;
    }
    
    // Text files
    public async Task WriteTextAsync(string path, string content)
    {
        var fullPath = ResolvePath(path);
        Directory.CreateDirectory(Path.GetDirectoryName(fullPath)!);
        await File.WriteAllTextAsync(fullPath, content, System.Text.Encoding.UTF8);
    }
    
    public async Task<string> ReadTextAsync(string path)
    {
        var fullPath = ResolvePath(path);
        if (!File.Exists(fullPath))
            throw new FileNotFoundException($"File not found: {path}");
        return await File.ReadAllTextAsync(fullPath);
    }
    
    // JSON
    public async Task WriteJsonAsync<T>(string path, T data)
    {
        string json = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
        await WriteTextAsync(path, json);
    }
    
    public async Task<T?> ReadJsonAsync<T>(string path)
    {
        string json = await ReadTextAsync(path);
        return JsonSerializer.Deserialize<T>(json);
    }
    
    // Directory listing
    public IEnumerable<FileInfo> ListFiles(string? relativePath = null, string pattern = "*")
    {
        var dirPath = relativePath != null ? ResolvePath(relativePath) : _basePath;
        if (!Directory.Exists(dirPath)) return Enumerable.Empty<FileInfo>();
        
        return Directory.EnumerateFiles(dirPath, pattern, SearchOption.AllDirectories)
            .Select(f => new FileInfo(f));
    }
    
    // Cleanup
    public void DeleteOldFiles(string pattern, TimeSpan maxAge)
    {
        var cutoff = DateTime.Now - maxAge;
        var oldFiles = ListFiles(pattern: pattern)
            .Where(f => f.LastWriteTime < cutoff);
        
        foreach (var file in oldFiles)
        {
            file.Delete();
            Console.WriteLine($"Deleted: {file.Name}");
        }
    }
    
    // Backup
    public async Task BackupAsync(string sourcePath, string backupPath)
    {
        var sourceDir = sourcePath != null ? ResolvePath(sourcePath) : _basePath;
        var backupDir = ResolvePath(backupPath);
        
        foreach (var file in Directory.EnumerateFiles(sourceDir, "*", 
            SearchOption.AllDirectories))
        {
            var relative = Path.GetRelativePath(sourceDir, file);
            var destPath = Path.Combine(backupDir, relative);
            
            Directory.CreateDirectory(Path.GetDirectoryName(destPath)!);
            File.Copy(file, destPath, overwrite: true);
        }
        
        Console.WriteLine($"Backup complete: {sourceDir} -> {backupDir}");
        await Task.CompletedTask;
    }
}

// ใช้งาน
var fs = new FileSystemManager("app_data");

// เก็บ settings
await fs.WriteJsonAsync("config/settings.json", new
{
    Theme = "dark",
    Language = "th",
    AutoSave = true
});

// โหลด settings
var settings = await fs.ReadJsonAsync<dynamic>("config/settings.json");

// เก็บ log
await fs.WriteTextAsync($"logs/{DateTime.Now:yyyy-MM-dd}.log", 
    $"[{DateTime.Now:HH:mm:ss}] Application started\n");

// List files
foreach (var file in fs.ListFiles(pattern: "*.json"))
    Console.WriteLine($"  {file.Name}: {file.Length} bytes");

// Cleanup old logs
fs.DeleteOldFiles("*.log", TimeSpan.FromDays(30));
```

---

## สรุป Part 14

ใน Part 14 เราได้เรียนรู้:

1. **File Operations** - Read, Write, Copy, Move, Delete
2. **Directory** - Create, Delete, Enumerate
3. **Path** - Combine, GetFileName, GetExtension
4. **Streams** - FileStream, StreamReader, MemoryStream
5. **JSON** - System.Text.Json, Serialize, Deserialize
6. **CSV** - Custom CSV parser
7. **XML** - XmlSerializer, LINQ to XML
8. **Config** - JSON configuration files
9. **Compression** - GZip, ZIP Archive
10. **FileSystemWatcher** - Monitor file changes

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 14 | Steps 131-140*
