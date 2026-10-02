# Part 06: Arrays และ Collections
## Steps 51-60: การจัดการข้อมูลหลายชิ้น

---

## Step 51: Arrays พื้นฐาน

```csharp
// ============================================
// Array Declaration
// ============================================

// สร้าง array แบบต่างๆ
int[] nums1 = new int[5];              // กำหนดขนาด, ค่า default = 0
int[] nums2 = new int[] { 1, 2, 3, 4, 5 }; // กำหนดค่าเริ่มต้น
int[] nums3 = { 10, 20, 30, 40, 50 }; // Shorthand
string[] names = { "Alice", "Bob", "Charlie" };

// Array Properties
Console.WriteLine(nums3.Length);   // 5
Console.WriteLine(nums3.Rank);     // 1 (มิติเดียว)

// Indexing
Console.WriteLine(nums3[0]);       // 10 (index เริ่มที่ 0)
Console.WriteLine(nums3[^1]);      // 50 (index สุดท้าย)
Console.WriteLine(nums3[^2]);      // 40 (นับจากท้าย)

// Slicing (Range)
int[] slice = nums3[1..4];         // { 20, 30, 40 } (index 1 ถึง 3)
int[] from2 = nums3[2..];         // { 30, 40, 50 } (จาก index 2)
int[] to3 = nums3[..3];           // { 10, 20, 30 } (ถึง index 2)
int[] copy = nums3[..];           // copy ทั้งหมด

Console.WriteLine(string.Join(", ", slice)); // 20, 30, 40

// ============================================
// Array Manipulation
// ============================================

// Copy
int[] source = { 1, 2, 3, 4, 5 };
int[] destination = new int[5];
Array.Copy(source, destination, source.Length);

// CopyTo
source.CopyTo(destination, 0); // copy ไปที่ index 0

// Clone (shallow copy)
int[] clone = (int[])source.Clone();

// Sort
int[] unsorted = { 5, 3, 8, 1, 9, 2, 7, 4, 6 };
Array.Sort(unsorted);
Console.WriteLine(string.Join(", ", unsorted)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Sort descending
Array.Sort(unsorted, (a, b) => b.CompareTo(a));
Console.WriteLine(string.Join(", ", unsorted)); // 9, 8, 7, 6, 5, 4, 3, 2, 1

// Reverse
Array.Reverse(unsorted);
Console.WriteLine(string.Join(", ", unsorted)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Search
int[] sorted = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
int idx = Array.BinarySearch(sorted, 7);
Console.WriteLine($"Found 7 at index: {idx}"); // 6

int idx2 = Array.IndexOf(sorted, 5);
Console.WriteLine($"Found 5 at index: {idx2}"); // 4

// Fill
int[] zeros = new int[5];
Array.Fill(zeros, 42);
Console.WriteLine(string.Join(", ", zeros)); // 42, 42, 42, 42, 42

// ============================================
// Array Methods (LINQ-based)
// ============================================

int[] data = { 3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5 };

Console.WriteLine($"Length: {data.Length}");
Console.WriteLine($"Max: {data.Max()}");
Console.WriteLine($"Min: {data.Min()}");
Console.WriteLine($"Sum: {data.Sum()}");
Console.WriteLine($"Average: {data.Average():F2}");
Console.WriteLine($"Contains 9: {data.Contains(9)}");
Console.WriteLine($"Count of 5: {data.Count(x => x == 5)}");
```

---

## Step 52: Multidimensional Arrays

```csharp
// ============================================
// 2D Array (Rectangular)
// ============================================

// สร้าง 2D array
int[,] matrix = new int[3, 4]; // 3 rows, 4 columns
int[,] matrix2 = { 
    { 1, 2, 3, 4 }, 
    { 5, 6, 7, 8 }, 
    { 9, 10, 11, 12 } 
};

// Dimensions
Console.WriteLine(matrix2.GetLength(0)); // 3 (rows)
Console.WriteLine(matrix2.GetLength(1)); // 4 (columns)
Console.WriteLine(matrix2.Rank);         // 2

// Access
Console.WriteLine(matrix2[1, 2]); // 7 (row 1, col 2)

// Iterate
for (int row = 0; row < matrix2.GetLength(0); row++)
{
    for (int col = 0; col < matrix2.GetLength(1); col++)
    {
        Console.Write($"{matrix2[row, col],4}");
    }
    Console.WriteLine();
}

// ============================================
// Jagged Array (Array of Arrays)
// ============================================

// Jagged = แต่ละ row มีขนาดต่างกันได้
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2, 3 };
jagged[1] = new int[] { 4, 5 };
jagged[2] = new int[] { 6, 7, 8, 9, 10 };

// Iterate
for (int i = 0; i < jagged.Length; i++)
{
    Console.Write($"Row {i}: ");
    for (int j = 0; j < jagged[i].Length; j++)
    {
        Console.Write($"{jagged[i][j]} ");
    }
    Console.WriteLine();
}

// ============================================
// Matrix Operations
// ============================================

// Matrix Addition
int[,] AddMatrices(int[,] a, int[,] b)
{
    int rows = a.GetLength(0);
    int cols = a.GetLength(1);
    var result = new int[rows, cols];
    
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            result[i, j] = a[i, j] + b[i, j];
    
    return result;
}

// Matrix Transpose
int[,] Transpose(int[,] matrix3)
{
    int rows = matrix3.GetLength(0);
    int cols = matrix3.GetLength(1);
    var result = new int[cols, rows];
    
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            result[j, i] = matrix3[i, j];
    
    return result;
}

void PrintMatrix(int[,] m)
{
    for (int i = 0; i < m.GetLength(0); i++)
    {
        for (int j = 0; j < m.GetLength(1); j++)
            Console.Write($"{m[i, j],4}");
        Console.WriteLine();
    }
}

int[,] mat1 = { { 1, 2, 3 }, { 4, 5, 6 } };
Console.WriteLine("Original:");
PrintMatrix(mat1);
Console.WriteLine("Transposed:");
PrintMatrix(Transpose(mat1));
```

---

## Step 53: List<T>

```csharp
// ============================================
// List<T> - Dynamic size array
// ============================================

// สร้าง List
var emptyList = new List<string>();
var initList = new List<int> { 1, 2, 3, 4, 5 };
var fromArray = new List<string>(new[] { "a", "b", "c" });

// Properties
Console.WriteLine(initList.Count);    // 5
Console.WriteLine(initList.Capacity); // 8 (internal buffer)

// Add
var fruits = new List<string>();
fruits.Add("แอปเปิ้ล");
fruits.Add("กล้วย");
fruits.Add("ส้ม");
fruits.AddRange(new[] { "มะม่วง", "สับปะรด" });

// Insert
fruits.Insert(1, "อะโวคาโด"); // แทรกที่ index 1

// Remove
fruits.Remove("กล้วย");          // ลบตัวแรกที่เจอ
fruits.RemoveAt(0);              // ลบที่ index 0
fruits.RemoveAll(f => f.Length > 4); // ลบตามเงื่อนไข

// Search
bool hasApple = fruits.Contains("ส้ม");
int index = fruits.IndexOf("ส้ม");
string? found = fruits.Find(f => f.StartsWith("ม"));    // หาตัวแรก
List<string> foundAll = fruits.FindAll(f => f.Length <= 4); // หาทั้งหมด

// Sort
fruits.Sort();                           // ตาม default comparison
fruits.Sort((a, b) => a.Length.CompareTo(b.Length)); // custom

// Convert
string[] array = fruits.ToArray();
IEnumerable<string> enumerable = fruits.AsEnumerable();
IReadOnlyList<string> readOnly = fruits.AsReadOnly();

// ============================================
// List Performance Tips
// ============================================

// กำหนด capacity ล่วงหน้าเพื่อป้องกัน reallocation
var optimizedList = new List<int>(1000);
for (int i = 0; i < 1000; i++)
    optimizedList.Add(i);

// ใช้ Span<T> สำหรับ read operations ที่ต้องการ performance
var listData = new List<int> { 1, 2, 3, 4, 5 };
Span<int> span = System.Runtime.InteropServices.CollectionsMarshal.AsSpan(listData);

// ============================================
// List Patterns
// ============================================

class StudentList
{
    private readonly List<Student> _students = new();
    
    public void Add(Student student) => _students.Add(student);
    
    public Student? FindById(int id) 
        => _students.Find(s => s.Id == id);
    
    public List<Student> GetByGrade(string grade)
        => _students.FindAll(s => s.Grade == grade);
    
    public List<Student> GetTopStudents(int count)
        => _students
            .OrderByDescending(s => s.Score)
            .Take(count)
            .ToList();
    
    public void RemoveStudent(int id)
    {
        int idx = _students.FindIndex(s => s.Id == id);
        if (idx >= 0) _students.RemoveAt(idx);
    }
    
    public double AverageScore()
        => _students.Count > 0 ? _students.Average(s => s.Score) : 0;
}

record Student(int Id, string Name, string Grade, double Score);
```

---

## Step 54: Dictionary<TKey, TValue>

```csharp
// ============================================
// Dictionary พื้นฐาน
// ============================================

// สร้าง Dictionary
var emptyDict = new Dictionary<string, int>();
var initDict = new Dictionary<string, int>
{
    { "สมชาย", 85 },
    { "สมหญิง", 92 },
    { "สมศรี", 78 }
};

// Collection initializer ใหม่
var dict2 = new Dictionary<string, int>
{
    ["Alice"] = 90,
    ["Bob"]   = 85,
    ["Carol"] = 95
};

// ============================================
// CRUD Operations
// ============================================

var scores = new Dictionary<string, int>();

// Add
scores.Add("Player1", 100);
scores["Player2"] = 200;  // Add หรือ Update
scores.TryAdd("Player1", 999); // ไม่ override ถ้ามีอยู่แล้ว

// Update
scores["Player1"] = 150;

// Read
int p1score = scores["Player1"];   // throw ถ้าไม่มี key
bool gotP3 = scores.TryGetValue("Player3", out int p3score); // safe

// Delete
scores.Remove("Player1");
scores.Clear();

// ============================================
// Dictionary Querying
// ============================================

var inventory = new Dictionary<string, int>
{
    { "แอปเปิ้ล", 50 },
    { "กล้วย", 30 },
    { "ส้ม", 0 },
    { "มะม่วง", 15 },
    { "สับปะรด", 8 }
};

// Check
Console.WriteLine(inventory.ContainsKey("กล้วย"));      // True
Console.WriteLine(inventory.ContainsValue(0));           // True (ส้ม)

// Get all keys/values
foreach (string key in inventory.Keys)
    Console.Write($"{key} ");

foreach (int value in inventory.Values)
    Console.Write($"{value} ");

// KVP iteration
foreach (var kvp in inventory)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");

// Deconstruction
foreach (var (name, qty) in inventory)
    Console.WriteLine($"{name}: {qty}");

// LINQ on Dictionary
var inStock = inventory
    .Where(kvp => kvp.Value > 0)
    .OrderByDescending(kvp => kvp.Value)
    .ToDictionary(kvp => kvp.Key, kvp => kvp.Value);

// ============================================
// Advanced Dictionary Patterns
// ============================================

// GroupBy to Dictionary
var students2 = new List<(string Name, string Class, int Score)>
{
    ("Alice", "A", 90), ("Bob", "B", 85), ("Carol", "A", 92),
    ("Dave", "B", 78), ("Eve", "A", 88)
};

var byClass = students2
    .GroupBy(s => s.Class)
    .ToDictionary(g => g.Key, g => g.ToList());

foreach (var (cls, slist) in byClass)
{
    Console.WriteLine($"Class {cls}: {string.Join(", ", slist.Select(s => s.Name))}");
}

// Word frequency counter
string text = "the quick brown fox jumps over the lazy dog the fox";
var wordFreq = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);

foreach (string word in text.Split(' '))
{
    wordFreq[word] = wordFreq.GetValueOrDefault(word, 0) + 1;
}

foreach (var (word, count) in wordFreq.OrderByDescending(kvp => kvp.Value))
    Console.WriteLine($"{word}: {count}");

// ============================================
// Concurrent Dictionary
// ============================================

using System.Collections.Concurrent;

var concurrentDict = new ConcurrentDictionary<string, int>();
concurrentDict.TryAdd("key1", 1);
concurrentDict.AddOrUpdate("key1", 1, (key, old) => old + 1);
int value = concurrentDict.GetOrAdd("key2", key => key.Length);
```

---

## Step 55: HashSet<T> และ Collections อื่นๆ

```csharp
// ============================================
// HashSet<T> - Unique items, O(1) lookup
// ============================================

var set1 = new HashSet<int> { 1, 2, 3, 4, 5 };
var set2 = new HashSet<int> { 3, 4, 5, 6, 7 };

// Add - ถ้ามีอยู่แล้ว return false
bool added = set1.Add(6); // true
bool duplicate = set1.Add(1); // false (มีอยู่แล้ว)

// Contains - O(1)
Console.WriteLine(set1.Contains(3)); // True

// Set Operations
var union = new HashSet<int>(set1);
union.UnionWith(set2); // set1 ∪ set2

var intersection = new HashSet<int>(set1);
intersection.IntersectWith(set2); // set1 ∩ set2

var difference = new HashSet<int>(set1);
difference.ExceptWith(set2); // set1 - set2

var symmetric = new HashSet<int>(set1);
symmetric.SymmetricExceptWith(set2); // (set1 ∪ set2) - (set1 ∩ set2)

Console.WriteLine($"Union: {string.Join(", ", union.OrderBy(x => x))}");
Console.WriteLine($"Intersection: {string.Join(", ", intersection.OrderBy(x => x))}");
Console.WriteLine($"Difference: {string.Join(", ", difference.OrderBy(x => x))}");
Console.WriteLine($"Symmetric: {string.Join(", ", symmetric.OrderBy(x => x))}");

// Subset/Superset
Console.WriteLine(set1.IsSubsetOf(union));    // True
Console.WriteLine(union.IsSupersetOf(set1));  // True

// ============================================
// Queue<T> - FIFO
// ============================================

var queue = new Queue<string>();

// Enqueue (ใส่ท้าย)
queue.Enqueue("Task 1");
queue.Enqueue("Task 2");
queue.Enqueue("Task 3");

Console.WriteLine($"Count: {queue.Count}"); // 3
Console.WriteLine($"Peek: {queue.Peek()}");  // Task 1 (ดูแต่ไม่ลบ)

// Dequeue (ดึงออกจากหน้า)
while (queue.Count > 0)
{
    string task = queue.Dequeue();
    Console.WriteLine($"Processing: {task}");
}

// ============================================
// Stack<T> - LIFO
// ============================================

var stack = new Stack<int>();

// Push (ใส่บนสุด)
stack.Push(1);
stack.Push(2);
stack.Push(3);

Console.WriteLine($"Peek: {stack.Peek()}"); // 3 (บนสุด)

// Pop (ดึงจากบนสุด)
while (stack.Count > 0)
{
    Console.WriteLine(stack.Pop());
}
// 3, 2, 1

// ============================================
// SortedDictionary และ SortedList
// ============================================

var sortedDict = new SortedDictionary<string, int>
{
    { "Banana", 3 },
    { "Apple", 5 },
    { "Cherry", 2 },
    { "Date", 8 }
};

// iterate โดยเรียงตาม key อัตโนมัติ
foreach (var (fruit, qty) in sortedDict)
    Console.WriteLine($"{fruit}: {qty}");
// Apple: 5, Banana: 3, Cherry: 2, Date: 8

var sortedList = new SortedList<int, string>
{
    { 3, "Third" },
    { 1, "First" },
    { 2, "Second" }
};

// ============================================
// LinkedList<T>
// ============================================

var linked = new LinkedList<int>();
linked.AddFirst(1);
linked.AddLast(3);
linked.AddAfter(linked.First!, 2); // ใส่หลัง node แรก

Console.WriteLine(string.Join(" → ", linked)); // 1 → 2 → 3

// Traverse
var current = linked.First;
while (current != null)
{
    Console.Write($"{current.Value} ");
    current = current.Next;
}
```

---

## Step 56: IEnumerable และ LINQ พื้นฐาน

```csharp
// ============================================
// IEnumerable<T> Interface
// ============================================

// IEnumerable คือ base interface ของ collections ทั้งหมด
// ทำให้สามารถใช้ foreach ได้

IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;
}

foreach (int n in GetEvenNumbers(10))
    Console.Write($"{n} "); // 0 2 4 6 8 10

// ============================================
// LINQ Methods พื้นฐาน
// ============================================

var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// Where - Filter
var evens = numbers.Where(n => n % 2 == 0);
Console.WriteLine(string.Join(", ", evens)); // 2, 4, 6, 8, 10

// Select - Transform
var squares = numbers.Select(n => n * n);
Console.WriteLine(string.Join(", ", squares)); // 1, 4, 9, 16, 25, 36, 49, 64, 81, 100

// OrderBy / OrderByDescending
var sorted = numbers.OrderByDescending(n => n);
Console.WriteLine(string.Join(", ", sorted));

// Take / Skip
var first3 = numbers.Take(3);
var skip3 = numbers.Skip(3);
var page2 = numbers.Skip(3).Take(3); // pagination

// First / Last / Single
int first = numbers.First();                    // 1
int last = numbers.Last();                      // 10
int? firstEven = numbers.FirstOrDefault(n => n > 50); // null (not found)

// Any / All / None
bool hasNegative = numbers.Any(n => n < 0);    // False
bool allPositive = numbers.All(n => n > 0);    // True

// Count / Sum / Min / Max / Average
Console.WriteLine(numbers.Count(n => n % 2 == 0)); // 5
Console.WriteLine(numbers.Sum());                    // 55
Console.WriteLine(numbers.Min());                    // 1
Console.WriteLine(numbers.Max());                    // 10
Console.WriteLine(numbers.Average());                // 5.5

// Distinct
var withDups = new[] { 1, 2, 2, 3, 3, 3, 4 };
var unique = withDups.Distinct();
Console.WriteLine(string.Join(", ", unique)); // 1, 2, 3, 4

// ============================================
// LINQ Query Syntax (SQL-like)
// ============================================

var students3 = new List<(string Name, int Age, double Score)>
{
    ("Alice", 20, 90.5), ("Bob", 22, 85.0), 
    ("Carol", 21, 92.0), ("Dave", 20, 78.5),
    ("Eve", 22, 95.0)
};

// Method syntax
var topStudents = students3
    .Where(s => s.Score >= 90)
    .OrderByDescending(s => s.Score)
    .Select(s => new { s.Name, s.Score });

// Query syntax (SQL-like)
var topStudentsQuery = 
    from s in students3
    where s.Score >= 90
    orderby s.Score descending
    select new { s.Name, s.Score };

foreach (var s in topStudents)
    Console.WriteLine($"{s.Name}: {s.Score}");

// Group By
var byAge = students3.GroupBy(s => s.Age);
foreach (var group in byAge)
{
    Console.WriteLine($"\nAge {group.Key}:");
    foreach (var s in group)
        Console.WriteLine($"  {s.Name}: {s.Score}");
}
```

---

## Step 57: Collection Initialization Patterns

```csharp
// ============================================
// Collection Expressions (C# 12)
// ============================================

// ก่อน C# 12
int[] old1 = new int[] { 1, 2, 3 };
List<int> old2 = new List<int> { 1, 2, 3 };

// C# 12: Collection expressions
int[] arr = [1, 2, 3, 4, 5];
List<int> list = [1, 2, 3, 4, 5];
HashSet<string> set = ["a", "b", "c"];

// Spread operator (..)
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]

// ============================================
// Immutable Collections
// ============================================

using System.Collections.Immutable;

// ImmutableList
var immutableList = ImmutableList.Create(1, 2, 3, 4, 5);
var withNew = immutableList.Add(6);    // สร้าง list ใหม่
var withRemoved = immutableList.Remove(3); // สร้าง list ใหม่

Console.WriteLine(string.Join(", ", immutableList)); // 1, 2, 3, 4, 5 (ไม่เปลี่ยน)
Console.WriteLine(string.Join(", ", withNew));       // 1, 2, 3, 4, 5, 6

// ImmutableDictionary
var immutableDict = ImmutableDictionary.Create<string, int>()
    .Add("a", 1)
    .Add("b", 2)
    .Add("c", 3);

var updated = immutableDict.SetItem("b", 20); // สร้าง dict ใหม่

// ============================================
// Custom Collection Class
// ============================================

class BoundedQueue<T>
{
    private readonly Queue<T> _queue = new();
    private readonly int _maxSize;
    
    public BoundedQueue(int maxSize) => _maxSize = maxSize;
    
    public int Count => _queue.Count;
    public bool IsFull => _queue.Count >= _maxSize;
    public bool IsEmpty => _queue.Count == 0;
    
    public bool TryEnqueue(T item)
    {
        if (IsFull) return false;
        _queue.Enqueue(item);
        return true;
    }
    
    public T? TryDequeue()
    {
        if (IsEmpty) return default;
        return _queue.Dequeue();
    }
    
    public void EnqueueWithEviction(T item)
    {
        if (IsFull) _queue.Dequeue(); // ลบตัวเก่าสุด
        _queue.Enqueue(item);
    }
}

// ============================================
// Observable Collection (MAUI/WPF)
// ============================================

using System.Collections.ObjectModel;

// ObservableCollection แจ้ง UI ให้ update อัตโนมัติ
var observableList = new ObservableCollection<string>();
observableList.CollectionChanged += (sender, e) =>
{
    Console.WriteLine($"Collection changed: {e.Action}");
};

observableList.Add("Item 1"); // "Collection changed: Add"
observableList.Remove("Item 1"); // "Collection changed: Remove"
```

---

## Step 58: Span<T> และ Memory<T>

```csharp
// ============================================
// Span<T> - High Performance, Zero allocation
// ============================================

// Span คือ struct ที่ reference memory โดยไม่ copy
int[] array = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

Span<int> span = array;           // Wrap array
Span<int> slice = array[2..7];    // Slice โดยไม่ copy

// Modify ผ่าน Span
slice[0] = 99; // แก้ไข array[2]
Console.WriteLine(array[2]); // 99

// ============================================
// ReadOnlySpan<T>
// ============================================

string text = "Hello, World!";
ReadOnlySpan<char> chars = text;
ReadOnlySpan<char> hello = chars[..5]; // "Hello"

// ไม่ต้อง allocate string ใหม่
bool startsWithH = hello[0] == 'H'; // True

// ============================================
// Memory<T> - Async-compatible Span
// ============================================

Memory<int> memory = new int[] { 1, 2, 3, 4, 5 };
ReadOnlyMemory<int> readonlyMem = memory;

// Slice
Memory<int> slice2 = memory.Slice(1, 3); // [2, 3, 4]

// ใช้ใน async method
async Task ProcessMemoryAsync(Memory<byte> buffer)
{
    // สามารถส่ง Memory ข้ามไป async method ได้ (Span ไม่ได้)
    await Task.Delay(100); // Simulate async work
    var span = buffer.Span;
    for (int i = 0; i < span.Length; i++)
        span[i] = (byte)(span[i] + 1);
}

// ============================================
// ArrayPool<T> - Reuse arrays
// ============================================

using System.Buffers;

// แทนที่จะ allocate array ใหม่ทุกครั้ง ให้ borrow จาก pool
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    // ใช้ buffer
    Array.Clear(buffer, 0, buffer.Length);
    // ... process ...
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer); // คืน pool
}
```

---

## Step 59: Collection Performance

```csharp
// ============================================
// Collection Comparison
// ============================================

using System.Diagnostics;

int N = 100_000;
var random = new Random(42);

// ============================================
// Search Performance
// ============================================

var list = Enumerable.Range(0, N).ToList();
var hashset = new HashSet<int>(list);
var sortedset = new SortedSet<int>(list);

int target = N / 2;
var sw = Stopwatch.StartNew();

// List.Contains: O(n) - Linear search
sw.Restart();
for (int i = 0; i < 1000; i++) list.Contains(target);
Console.WriteLine($"List.Contains: {sw.ElapsedMilliseconds}ms");

// HashSet.Contains: O(1) - Hash lookup
sw.Restart();
for (int i = 0; i < 1000; i++) hashset.Contains(target);
Console.WriteLine($"HashSet.Contains: {sw.ElapsedMilliseconds}ms");

// ============================================
// Add Performance
// ============================================

sw.Restart();
var listTest = new List<int>();
for (int i = 0; i < N; i++) listTest.Add(i);
Console.WriteLine($"List.Add: {sw.ElapsedMilliseconds}ms");

sw.Restart();
var listPrealloc = new List<int>(N); // Pre-allocate
for (int i = 0; i < N; i++) listPrealloc.Add(i);
Console.WriteLine($"List.Add (pre-alloc): {sw.ElapsedMilliseconds}ms");

// ============================================
// Collection Selection Guide
// ============================================

/*
 * เลือก Collection ให้เหมาะสม:
 * 
 * ต้องการ:            ใช้:
 * ------------------------------------------------
 * เพิ่ม/ลบ/อ่านบ่อย  List<T>
 * ค้นหาเร็ว           HashSet<T>, Dictionary<K,V>
 * เรียง key อัตโนมัติ SortedDictionary<K,V>
 * FIFO queue          Queue<T>
 * LIFO stack          Stack<T>
 * Thread-safe         ConcurrentDictionary<K,V>
 * ไม่เปลี่ยนแปลง     ImmutableList<T>
 * Performance สูง     Span<T>, ArrayPool<T>
 * ขนาดใหญ่มาก        IEnumerable (lazy)
 */
```

---

## Step 60: Collections ใน MAUI Context

```csharp
// ============================================
// ObservableCollection สำหรับ Data Binding
// ============================================

using System.Collections.ObjectModel;

// ViewModel ใน MAUI
class ProductViewModel
{
    // ObservableCollection: เมื่อ add/remove, UI update อัตโนมัติ
    public ObservableCollection<Product2> Products { get; } = new();
    
    public void LoadProducts()
    {
        Products.Clear();
        var data = new List<Product2>
        {
            new(1, "iPhone 16", 39900),
            new(2, "Samsung S24", 35900),
            new(3, "Pixel 9", 29900)
        };
        
        foreach (var product in data)
            Products.Add(product);
    }
    
    public void AddProduct(Product2 product)
        => Products.Add(product);
    
    public void RemoveProduct(int id)
    {
        var product = Products.FirstOrDefault(p => p.Id == id);
        if (product != null) Products.Remove(product);
    }
    
    // สำหรับ Filter
    public ObservableCollection<Product2> GetFiltered(string query)
    {
        if (string.IsNullOrWhiteSpace(query))
            return Products;
        
        var filtered = Products
            .Where(p => p.Name.Contains(query, StringComparison.OrdinalIgnoreCase))
            .ToList();
        
        return new ObservableCollection<Product2>(filtered);
    }
}

record Product2(int Id, string Name, decimal Price);

// ============================================
// Grouping สำหรับ MAUI ListView/CollectionView
// ============================================

class ContactGroup : ObservableCollection<Contact>
{
    public string GroupName { get; }
    
    public ContactGroup(string name, IEnumerable<Contact> contacts) : base(contacts)
    {
        GroupName = name;
    }
}

class ContactsViewModel
{
    private List<Contact> _allContacts = new();
    
    public ObservableCollection<ContactGroup> GroupedContacts { get; } = new();
    
    public void LoadAndGroup(List<Contact> contacts)
    {
        _allContacts = contacts;
        
        var groups = contacts
            .GroupBy(c => c.Name[0].ToString().ToUpper())
            .OrderBy(g => g.Key)
            .Select(g => new ContactGroup(g.Key, g.OrderBy(c => c.Name)));
        
        GroupedContacts.Clear();
        foreach (var group in groups)
            GroupedContacts.Add(group);
    }
}

record Contact(string Name, string Phone, string Email);
```

---

## แบบฝึกหัด Part 06

### แบบฝึกหัดที่ 6.1: Student Grade System
สร้างระบบที่:
- เก็บนักเรียนด้วย Dictionary (id → Student)
- เพิ่ม/ลบ/แก้ไข/ค้นหานักเรียน
- แสดงอันดับ Top 5
- คำนวณเกรดเฉลี่ย

### แบบฝึกหัดที่ 6.2: Inventory System
สร้างระบบ inventory ที่:
- เก็บสินค้า (ชื่อ, จำนวน, ราคา)
- เพิ่ม/ลด stock
- ค้นหาสินค้าที่ใกล้หมด (< 10)
- คำนวณมูลค่ารวม

### แบบฝึกหัดที่ 6.3: Custom Data Structure
สร้าง MinMaxStack ที่:
- Push/Pop ปกติ
- GetMin() - O(1)
- GetMax() - O(1)

---

## เฉลยแบบฝึกหัด Part 06

### เฉลย 6.1: Student Grade System

```csharp
class StudentGradeSystem
{
    record Student3(int Id, string Name, Dictionary<string, double> Scores)
    {
        public double Average => Scores.Count > 0 ? Scores.Values.Average() : 0;
        public string Grade => Average switch
        {
            >= 90 => "A", >= 80 => "B", >= 70 => "C", >= 60 => "D", _ => "F"
        };
    }
    
    private readonly Dictionary<int, Student3> _students = new();
    private int _nextId = 1;
    
    public Student3 AddStudent(string name)
    {
        var student = new Student3(_nextId++, name, new Dictionary<string, double>());
        _students[student.Id] = student;
        return student;
    }
    
    public bool AddScore(int id, string subject, double score)
    {
        if (!_students.TryGetValue(id, out var student)) return false;
        student.Scores[subject] = score;
        return true;
    }
    
    public List<Student3> GetTopStudents(int n)
        => _students.Values.OrderByDescending(s => s.Average).Take(n).ToList();
    
    public double ClassAverage()
        => _students.Count > 0 ? _students.Values.Average(s => s.Average) : 0;
    
    public void PrintReport()
    {
        Console.WriteLine("\n=== รายงานคะแนน ===");
        Console.WriteLine($"{"ลำดับ",-5} {"ชื่อ",-15} {"เฉลี่ย",-10} {"เกรด",-5}");
        Console.WriteLine(new string('-', 40));
        
        int rank = 1;
        foreach (var s in GetTopStudents(_students.Count))
        {
            Console.WriteLine($"{rank++,-5} {s.Name,-15} {s.Average,-10:F2} {s.Grade,-5}");
        }
        
        Console.WriteLine(new string('-', 40));
        Console.WriteLine($"คะแนนเฉลี่ยชั้นเรียน: {ClassAverage():F2}");
    }
}
```

### เฉลย 6.3: MinMaxStack

```csharp
class MinMaxStack<T> where T : IComparable<T>
{
    private readonly Stack<T> _mainStack = new();
    private readonly Stack<T> _minStack = new();
    private readonly Stack<T> _maxStack = new();
    
    public int Count => _mainStack.Count;
    public bool IsEmpty => _mainStack.Count == 0;
    
    public void Push(T item)
    {
        _mainStack.Push(item);
        
        if (_minStack.Count == 0 || item.CompareTo(_minStack.Peek()) <= 0)
            _minStack.Push(item);
        else
            _minStack.Push(_minStack.Peek());
        
        if (_maxStack.Count == 0 || item.CompareTo(_maxStack.Peek()) >= 0)
            _maxStack.Push(item);
        else
            _maxStack.Push(_maxStack.Peek());
    }
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        _minStack.Pop();
        _maxStack.Pop();
        return _mainStack.Pop();
    }
    
    public T GetMin()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _minStack.Peek();
    }
    
    public T GetMax()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _maxStack.Peek();
    }
}

var s = new MinMaxStack<int>();
s.Push(5); s.Push(3); s.Push(8); s.Push(1); s.Push(6);
Console.WriteLine($"Min: {s.GetMin()}, Max: {s.GetMax()}"); // Min: 1, Max: 8
s.Pop(); // ลบ 6
s.Pop(); // ลบ 1
Console.WriteLine($"Min: {s.GetMin()}, Max: {s.GetMax()}"); // Min: 3, Max: 8
```

---

## สรุป Part 06

ในส่วนนี้เราได้เรียนรู้:

1. **Arrays** - 1D, 2D, Jagged, Span, Range
2. **List<T>** - Dynamic collections
3. **Dictionary<K,V>** - Key-value storage
4. **HashSet<T>** - Unique items, set operations
5. **Queue/Stack** - FIFO/LIFO
6. **SortedDictionary** - Ordered keys
7. **IEnumerable/LINQ** - Query collections
8. **Span<T>/Memory<T>** - High performance
9. **ObservableCollection** - MAUI data binding
10. **ImmutableCollections** - Thread-safe immutable

## ขั้นต่อไป

ใน Part 07 เราจะเรียนรู้เกี่ยวกับ:
- Classes และ Objects
- Constructors
- Properties
- Fields
- Access Modifiers
- Object initialization

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 06 | Steps 51-60*
