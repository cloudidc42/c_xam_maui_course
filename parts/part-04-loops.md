# Part 04: Loops - for, while, foreach, do-while
## Steps 31-40: การทำซ้ำใน C#

---

## Step 31: for Loop

```csharp
// ============================================
// for Loop พื้นฐาน
// ============================================

// รูปแบบ: for (initializer; condition; iterator)
for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"i = {i}");
}
// Output: 0 1 2 3 4

// นับถอยหลัง
for (int i = 10; i >= 0; i--)
{
    Console.Write($"{i} ");
}
Console.WriteLine(); // 10 9 8 7 6 5 4 3 2 1 0

// นับทีละ 2
for (int i = 0; i <= 20; i += 2)
{
    Console.Write($"{i} ");
}
// 0 2 4 6 8 10 12 14 16 18 20

// ============================================
// for Loop กับ Array
// ============================================

string[] fruits = { "แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง", "สับปะรด" };

// แบบปกติ
for (int i = 0; i < fruits.Length; i++)
{
    Console.WriteLine($"[{i}] {fruits[i]}");
}

// Loop แบบกลับ
for (int i = fruits.Length - 1; i >= 0; i--)
{
    Console.WriteLine($"[{i}] {fruits[i]}");
}

// ============================================
// for Loop หลาย Variables
// ============================================

for (int i = 0, j = 10; i < j; i++, j--)
{
    Console.WriteLine($"i={i}, j={j}");
}
// i=0, j=10
// i=1, j=9
// i=2, j=8
// i=3, j=7
// i=4, j=6

// ============================================
// Infinite Loop
// ============================================

// for(;;) = infinite loop
// ต้องมี break ข้างใน
int counter = 0;
for (;;)
{
    counter++;
    if (counter >= 5) break;
}
Console.WriteLine($"counter = {counter}"); // 5
```

---

## Step 32: while Loop

```csharp
// ============================================
// while Loop พื้นฐาน
// ============================================

// รูปแบบ: while (condition) { ... }

int count = 0;
while (count < 5)
{
    Console.WriteLine($"count = {count}");
    count++;
}

// ตรวจสอบ condition ก่อน loop
// ถ้า condition เป็น false ตั้งแต่ต้น จะไม่ทำงานเลย
int x = 10;
while (x < 5) // condition เป็น false
{
    Console.WriteLine("จะไม่แสดงเลย");
}

// ============================================
// while กับ User Input
// ============================================

Console.WriteLine("ป้อนตัวเลข (0 เพื่อหยุด):");
int sum = 0;
int inputCount = 0;

while (true)
{
    Console.Write($"ตัวเลขที่ {inputCount + 1}: ");
    if (!int.TryParse(Console.ReadLine(), out int input))
    {
        Console.WriteLine("ใส่ตัวเลขเท่านั้น");
        continue;
    }
    
    if (input == 0) break;
    
    sum += input;
    inputCount++;
}

if (inputCount > 0)
    Console.WriteLine($"ผลรวม: {sum}, เฉลี่ย: {(double)sum / inputCount:F2}");
else
    Console.WriteLine("ไม่มีข้อมูล");

// ============================================
// while กับ File Reading (simulation)
// ============================================

// Simulate reading lines
var lines = new Queue<string>(new[] { "line 1", "line 2", "line 3" });

while (lines.Count > 0)
{
    string line = lines.Dequeue();
    Console.WriteLine($"Processing: {line}");
}
```

---

## Step 33: do-while Loop

```csharp
// ============================================
// do-while Loop
// ============================================

// รูปแบบ: do { ... } while (condition);
// ทำงานอย่างน้อย 1 ครั้งเสมอ ก่อนตรวจสอบ condition

int number = 0;
do
{
    Console.WriteLine($"number = {number}");
    number++;
} while (number < 5);
// 0 1 2 3 4

// ต่างจาก while:
// do-while ทำงานก่อนตรวจสอบ condition
int y = 10;
do
{
    Console.WriteLine("จะแสดง 1 ครั้งเสมอ แม้ condition false");
} while (y < 5); // condition false แต่ loop ทำงาน 1 ครั้ง

// ============================================
// do-while สำหรับ Menu
// ============================================

int choice;
do
{
    Console.WriteLine("\n=== เมนูหลัก ===");
    Console.WriteLine("1. ดูข้อมูล");
    Console.WriteLine("2. เพิ่มข้อมูล");
    Console.WriteLine("3. แก้ไขข้อมูล");
    Console.WriteLine("4. ลบข้อมูล");
    Console.WriteLine("0. ออก");
    Console.Write("เลือก: ");
    
    if (!int.TryParse(Console.ReadLine(), out choice))
    {
        Console.WriteLine("กรุณาเลือกตัวเลข");
        choice = -1; // ให้ loop ทำงานต่อ
        continue;
    }
    
    switch (choice)
    {
        case 1: Console.WriteLine("แสดงข้อมูล..."); break;
        case 2: Console.WriteLine("เพิ่มข้อมูล..."); break;
        case 3: Console.WriteLine("แก้ไขข้อมูล..."); break;
        case 4: Console.WriteLine("ลบข้อมูล..."); break;
        case 0: Console.WriteLine("กำลังออกจากโปรแกรม..."); break;
        default: Console.WriteLine("ตัวเลือกไม่ถูกต้อง"); break;
    }
} while (choice != 0);
```

---

## Step 34: foreach Loop

```csharp
// ============================================
// foreach Loop พื้นฐาน
// ============================================

// รูปแบบ: foreach (type variable in collection)
// ใช้กับ IEnumerable collections

string[] countries = { "ไทย", "ญี่ปุ่น", "เกาหลี", "จีน", "อเมริกา" };

foreach (string country in countries)
{
    Console.WriteLine(country);
}

// ============================================
// foreach กับ List
// ============================================

var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// หาผลรวม
int total = 0;
foreach (int num in numbers)
{
    total += num;
}
Console.WriteLine($"ผลรวม: {total}"); // 55

// หาตัวเลขคู่
Console.Write("ตัวเลขคู่: ");
foreach (int num in numbers)
{
    if (num % 2 == 0)
        Console.Write($"{num} ");
}

// ============================================
// foreach กับ Dictionary
// ============================================

var studentScores = new Dictionary<string, int>
{
    { "สมชาย", 85 },
    { "สมหญิง", 92 },
    { "สมศรี", 78 },
    { "สมพงษ์", 88 }
};

// Iterate ทั้ง key และ value
foreach (var (name, score) in studentScores)
{
    string grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "D";
    Console.WriteLine($"{name}: {score} ({grade})");
}

// Iterate เฉพาะ Keys
foreach (string name in studentScores.Keys)
{
    Console.WriteLine(name);
}

// Iterate เฉพาะ Values
foreach (int score in studentScores.Values)
{
    Console.Write($"{score} ");
}

// ============================================
// foreach กับ String
// ============================================

string text = "Hello, World!";
int vowelCount = 0;

foreach (char c in text)
{
    if ("aeiouAEIOU".Contains(c))
        vowelCount++;
}
Console.WriteLine($"Vowels in '{text}': {vowelCount}");

// ============================================
// foreach ด้วย Index (C# 7.3+)
// ============================================

// วิธีที่ 1: ใช้ for loop
for (int i = 0; i < countries.Length; i++)
{
    Console.WriteLine($"{i}: {countries[i]}");
}

// วิธีที่ 2: LINQ Select with index
foreach (var (country, index) in countries.Select((c, i) => (c, i)))
{
    Console.WriteLine($"{index}: {country}");
}

// วิธีที่ 3: ใช้ Index struct
foreach (var item in countries.Index())
{
    Console.WriteLine($"{item.Index}: {item.Item}");
}
```

---

## Step 35: Loop Control - break, continue, goto

```csharp
// ============================================
// break - ออกจาก loop
// ============================================

// หาตัวเลขแรกที่หาร 7 ลงตัว
for (int i = 1; i <= 100; i++)
{
    if (i % 7 == 0)
    {
        Console.WriteLine($"ตัวเลขแรกที่หาร 7 ลงตัว: {i}");
        break; // ออกจาก loop ทันที
    }
}

// break ใน while
int search = 42;
int[] array = { 10, 25, 42, 73, 8, 91 };
int foundIndex = -1;
int j = 0;

while (j < array.Length)
{
    if (array[j] == search)
    {
        foundIndex = j;
        break; // พบแล้ว หยุดค้นหา
    }
    j++;
}

if (foundIndex != -1)
    Console.WriteLine($"พบ {search} ที่ index {foundIndex}");
else
    Console.WriteLine($"ไม่พบ {search}");

// ============================================
// continue - ข้ามการทำซ้ำปัจจุบัน
// ============================================

// แสดงเฉพาะตัวเลขคี่
for (int i = 1; i <= 10; i++)
{
    if (i % 2 == 0) continue; // ข้ามตัวเลขคู่
    Console.Write($"{i} ");
}
// 1 3 5 7 9

Console.WriteLine();

// ข้ามค่า null/empty
string[] names = { "Alice", "", "Bob", null, "Charlie", " ", "David" };
foreach (string? name in names)
{
    if (string.IsNullOrWhiteSpace(name)) continue; // ข้ามค่าว่าง
    Console.WriteLine($"Hello, {name}!");
}

// ============================================
// Nested Loop Control
// ============================================

// break ออกจากแค่ loop ที่ใกล้ที่สุด
for (int i = 0; i < 3; i++)
{
    for (int k = 0; k < 3; k++)
    {
        if (k == 1) break; // ออกจาก inner loop เท่านั้น
        Console.Write($"({i},{k}) ");
    }
}
// (0,0) (1,0) (2,0)
Console.WriteLine();

// ============================================
// การออกจาก Outer Loop (ไม่แนะนำ goto)
// ============================================

// วิธีที่ 1: ใช้ flag variable
bool found = false;
int targetRow = -1, targetCol = -1;
int[,] matrix = { { 1, 2, 3 }, { 4, 5, 6 }, { 7, 8, 9 } };

for (int row = 0; row < 3 && !found; row++)
{
    for (int col = 0; col < 3 && !found; col++)
    {
        if (matrix[row, col] == 5)
        {
            targetRow = row;
            targetCol = col;
            found = true;
        }
    }
}
Console.WriteLine($"พบ 5 ที่ row={targetRow}, col={targetCol}");

// วิธีที่ 2: Extract to method (แนะนำมากที่สุด)
var (row2, col2) = FindInMatrix(matrix, 5);
Console.WriteLine($"พบ 5 ที่ row={row2}, col={col2}");

(int Row, int Col) FindInMatrix(int[,] mat, int target)
{
    for (int r = 0; r < mat.GetLength(0); r++)
    {
        for (int c = 0; c < mat.GetLength(1); c++)
        {
            if (mat[r, c] == target)
                return (r, c);
        }
    }
    return (-1, -1);
}
```

---

## Step 36: Nested Loops

```csharp
// ============================================
// Nested Loops
// ============================================

// สร้าง Multiplication Table
Console.WriteLine("ตารางสูตรคูณ:");
Console.Write("    ");
for (int i = 1; i <= 10; i++)
    Console.Write($"{i,4}");
Console.WriteLine();

for (int i = 1; i <= 10; i++)
{
    Console.Write($"{i,3}|");
    for (int j = 1; j <= 10; j++)
    {
        Console.Write($"{i * j,4}");
    }
    Console.WriteLine();
}

// ============================================
// สร้าง Pattern ด้วย Nested Loops
// ============================================

// Pattern 1: สามเหลี่ยม
int n = 5;
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= i; j++)
        Console.Write("* ");
    Console.WriteLine();
}
/*
* 
* * 
* * * 
* * * * 
* * * * * 
*/

// Pattern 2: สามเหลี่ยมกลับหัว
for (int i = n; i >= 1; i--)
{
    for (int j = 1; j <= i; j++)
        Console.Write("* ");
    Console.WriteLine();
}

// Pattern 3: ตัวเลขสามเหลี่ยม
for (int i = 1; i <= n; i++)
{
    for (int j = 1; j <= i; j++)
        Console.Write($"{j} ");
    Console.WriteLine();
}
/*
1 
1 2 
1 2 3 
1 2 3 4 
1 2 3 4 5 
*/

// Pattern 4: Diamond
int size = 5;
// Upper half
for (int i = 1; i <= size; i++)
{
    // Spaces
    for (int s = 1; s <= size - i; s++)
        Console.Write(" ");
    // Stars
    for (int j = 1; j <= 2 * i - 1; j++)
        Console.Write("*");
    Console.WriteLine();
}
// Lower half
for (int i = size - 1; i >= 1; i--)
{
    // Spaces
    for (int s = 1; s <= size - i; s++)
        Console.Write(" ");
    // Stars
    for (int j = 1; j <= 2 * i - 1; j++)
        Console.Write("*");
    Console.WriteLine();
}
```

---

## Step 37: Iterators และ yield

```csharp
// ============================================
// yield return - สร้าง Custom Iterator
// ============================================

// สร้าง sequence ของ Fibonacci
IEnumerable<int> Fibonacci(int count)
{
    int a = 0, b = 1;
    for (int i = 0; i < count; i++)
    {
        yield return a; // ส่งค่าแล้วหยุดชั่วคราว
        (a, b) = (b, a + b); // Tuple swap
    }
}

// ใช้งาน
foreach (int fib in Fibonacci(10))
{
    Console.Write($"{fib} ");
}
// 0 1 1 2 3 5 8 13 21 34

// ============================================
// yield return กับ infinite sequence
// ============================================

IEnumerable<int> NaturalNumbers()
{
    int i = 1;
    while (true) // ไม่มีวันสิ้นสุด
    {
        yield return i++;
    }
}

// ต้องใช้ Take() เพื่อจำกัดจำนวน
foreach (int num in NaturalNumbers().Take(10))
{
    Console.Write($"{num} ");
}
// 1 2 3 4 5 6 7 8 9 10

// ============================================
// yield break - หยุด iterator
// ============================================

IEnumerable<int> NumbersUntil(int limit)
{
    for (int i = 1; ; i++)
    {
        if (i > limit) yield break; // หยุดส่งค่า
        yield return i;
    }
}

foreach (int n2 in NumbersUntil(5))
{
    Console.Write($"{n2} ");
}
// 1 2 3 4 5

// ============================================
// Iterator Class
// ============================================

class Range : IEnumerable<int>
{
    private readonly int _start, _end, _step;
    
    public Range(int start, int end, int step = 1)
    {
        _start = start;
        _end = end;
        _step = step;
    }
    
    public IEnumerator<int> GetEnumerator()
    {
        for (int i = _start; i <= _end; i += _step)
            yield return i;
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        => GetEnumerator();
}

// ใช้งาน
var range = new Range(1, 20, 3);
foreach (int n3 in range)
{
    Console.Write($"{n3} ");
}
// 1 4 7 10 13 16 19
```

---

## Step 38: Loop Optimization

```csharp
// ============================================
// Performance Considerations
// ============================================

using System.Diagnostics;

// ทดสอบ performance ของ Loop แบบต่างๆ

int[] bigArray = Enumerable.Range(1, 1_000_000).ToArray();
var sw = Stopwatch.StartNew();

// ============================================
// วิธีที่ 1: for loop (เร็วที่สุด)
// ============================================

sw.Restart();
long sum1 = 0;
for (int i = 0; i < bigArray.Length; i++)
{
    sum1 += bigArray[i];
}
sw.Stop();
Console.WriteLine($"for: {sw.ElapsedMilliseconds}ms, sum={sum1}");

// ============================================
// วิธีที่ 2: foreach loop
// ============================================

sw.Restart();
long sum2 = 0;
foreach (int n in bigArray)
{
    sum2 += n;
}
sw.Stop();
Console.WriteLine($"foreach: {sw.ElapsedMilliseconds}ms, sum={sum2}");

// ============================================
// วิธีที่ 3: LINQ (เข้าใจง่ายที่สุด แต่ช้ากว่า)
// ============================================

sw.Restart();
long sum3 = bigArray.Sum(x => (long)x);
sw.Stop();
Console.WriteLine($"LINQ: {sw.ElapsedMilliseconds}ms, sum={sum3}");

// ============================================
// วิธีที่ 4: Parallel (เร็วบน multi-core)
// ============================================

sw.Restart();
long sum4 = 0;
object lockObj = new();
Parallel.ForEach(bigArray, n =>
{
    Interlocked.Add(ref sum4, n);
});
sw.Stop();
Console.WriteLine($"Parallel: {sw.ElapsedMilliseconds}ms, sum={sum4}");

// ============================================
// Loop Unrolling (สำหรับ performance critical)
// ============================================

// แทนที่จะทำ 1 operation ต่อรอบ ทำ 4 operations
int[] array = Enumerable.Range(1, 100).ToArray();
int total = 0;
int len = array.Length;
int i2 = 0;

// Process 4 items per iteration
for (; i2 <= len - 4; i2 += 4)
{
    total += array[i2] + array[i2 + 1] + array[i2 + 2] + array[i2 + 3];
}

// Process remaining items
for (; i2 < len; i2++)
{
    total += array[i2];
}

Console.WriteLine($"Total: {total}");

// ============================================
// Span<T> สำหรับ high-performance loops
// ============================================

int[] numbers2 = { 1, 2, 3, 4, 5 };
Span<int> span = numbers2;

int spanSum = 0;
foreach (ref int n2 in span) // ref เพื่อ modify ใน-place
{
    n2 *= 2; // คูณ 2 ทุกตัว
    spanSum += n2;
}
Console.WriteLine($"Span sum: {spanSum}");
```

---

## Step 39: Async Loops

```csharp
// ============================================
// Async/Await ใน Loops
// ============================================

using System.Net.Http;

// ❌ Anti-pattern: Sequential async (ช้ามาก)
async Task<string[]> DownloadSequential(string[] urls)
{
    var results = new string[urls.Length];
    for (int i = 0; i < urls.Length; i++)
    {
        results[i] = await DownloadAsync(urls[i]); // รอทีละ URL
    }
    return results;
}

// ✅ Best practice: Parallel async
async Task<string[]> DownloadParallel(string[] urls)
{
    // สร้าง Task ทั้งหมดก่อน แล้วรอพร้อมกัน
    var tasks = urls.Select(url => DownloadAsync(url));
    return await Task.WhenAll(tasks);
}

async Task<string> DownloadAsync(string url)
{
    await Task.Delay(100); // simulate download
    return $"Content from {url}";
}

// ============================================
// Async IAsyncEnumerable (C# 8+)
// ============================================

async IAsyncEnumerable<int> GenerateNumbersAsync()
{
    for (int i = 1; i <= 10; i++)
    {
        await Task.Delay(100); // Simulate async work
        yield return i;
    }
}

// ใช้งาน await foreach
async Task ProcessNumbersAsync()
{
    await foreach (int number in GenerateNumbersAsync())
    {
        Console.WriteLine($"Got: {number}");
    }
}

// ============================================
// Cancellation ใน Async Loop
// ============================================

async IAsyncEnumerable<int> GenerateWithCancel(
    [System.Runtime.CompilerServices.EnumeratorCancellation] 
    CancellationToken cancellationToken = default)
{
    for (int i = 1; ; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(100, cancellationToken);
        yield return i;
    }
}

async Task ProcessWithCancellationAsync()
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(1));
    
    try
    {
        await foreach (int n in GenerateWithCancel(cts.Token))
        {
            Console.WriteLine($"Number: {n}");
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Operation cancelled after 1 second");
    }
}

// ============================================
// Rate-limited Loop
// ============================================

async Task ProcessWithRateLimitAsync(List<string> items, int maxConcurrent = 3)
{
    using var semaphore = new SemaphoreSlim(maxConcurrent, maxConcurrent);
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try
        {
            await ProcessItemAsync(item);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    await Task.WhenAll(tasks);
}

async Task ProcessItemAsync(string item)
{
    await Task.Delay(100); // Simulate processing
    Console.WriteLine($"Processed: {item}");
}
```

---

## Step 40: Loop Patterns ขั้นสูง

```csharp
// ============================================
// Sliding Window
// ============================================

int[] data = { 1, 3, 5, 7, 2, 8, 4, 6, 9, 10 };
int windowSize = 3;

Console.WriteLine("Moving Average (window=3):");
for (int i = 0; i <= data.Length - windowSize; i++)
{
    double average = 0;
    for (int j = i; j < i + windowSize; j++)
        average += data[j];
    average /= windowSize;
    
    Console.WriteLine($"[{i}..{i + windowSize - 1}]: {average:F2}");
}

// ============================================
// Accumulator Pattern
// ============================================

var transactions = new List<(string Description, decimal Amount)>
{
    ("เงินเดือน", 25000),
    ("ค่าเช่า", -8000),
    ("อาหาร", -3000),
    ("บันเทิง", -1500),
    ("ออมเงิน", -5000),
    ("โบนัส", 10000)
};

decimal balance = 0;
decimal totalIncome = 0;
decimal totalExpense = 0;

Console.WriteLine("\nรายการธุรกรรม:");
Console.WriteLine(new string('-', 45));
foreach (var (desc, amount) in transactions)
{
    balance += amount;
    if (amount > 0) totalIncome += amount;
    else totalExpense += Math.Abs(amount);
    
    string type = amount > 0 ? "รับ" : "จ่าย";
    Console.WriteLine($"{desc,-12} {type} {Math.Abs(amount),10:N2}  ยอดคงเหลือ: {balance,10:N2}");
}
Console.WriteLine(new string('-', 45));
Console.WriteLine($"รายรับรวม:  {totalIncome,10:N2}");
Console.WriteLine($"รายจ่ายรวม: {totalExpense,10:N2}");
Console.WriteLine($"ยอดสุทธิ:   {balance,10:N2}");

// ============================================
// Two-pointer Technique
// ============================================

// หาคู่ตัวเลขที่รวมกันได้ target sum
int[] sortedArray = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
int target = 11;

int left = 0, right = sortedArray.Length - 1;
var pairs = new List<(int, int)>();

while (left < right)
{
    int sum = sortedArray[left] + sortedArray[right];
    
    if (sum == target)
    {
        pairs.Add((sortedArray[left], sortedArray[right]));
        left++;
        right--;
    }
    else if (sum < target)
    {
        left++;
    }
    else
    {
        right--;
    }
}

Console.WriteLine($"\nคู่ตัวเลขที่รวมกันได้ {target}:");
foreach (var (a, b) in pairs)
{
    Console.WriteLine($"{a} + {b} = {target}");
}

// ============================================
// Fibonacci ด้วยวิธีต่างๆ
// ============================================

// วิธีที่ 1: Iterative (ดีที่สุด)
long FibIterative(int n)
{
    if (n <= 1) return n;
    long a = 0, b = 1;
    for (int i = 2; i <= n; i++)
        (a, b) = (b, a + b);
    return b;
}

// วิธีที่ 2: ด้วย array
long[] FibArray(int count)
{
    long[] fib = new long[count];
    fib[0] = 0;
    if (count > 1) fib[1] = 1;
    for (int i = 2; i < count; i++)
        fib[i] = fib[i - 1] + fib[i - 2];
    return fib;
}

Console.WriteLine("\nFibonacci:");
for (int i = 0; i < 15; i++)
    Console.Write($"{FibIterative(i)} ");
```

---

## แบบฝึกหัด Part 04

### แบบฝึกหัดที่ 4.1: FizzBuzz
พิมพ์ตัวเลข 1-100:
- ถ้าหาร 3 ลงตัว พิมพ์ "Fizz"
- ถ้าหาร 5 ลงตัว พิมพ์ "Buzz"
- ถ้าหาร 15 ลงตัว พิมพ์ "FizzBuzz"

### แบบฝึกหัดที่ 4.2: ตรวจสอบจำนวนเฉพาะ
หาตัวเลขจำนวนเฉพาะทั้งหมดตั้งแต่ 1-100 ด้วย Sieve of Eratosthenes

### แบบฝึกหัดที่ 4.3: เกมทายตัวเลขขั้นสูง
สร้างเกมที่:
- สุ่มตัวเลข 1-1000
- ให้ทาย 10 ครั้ง
- บอก "ร้อน/อุ่น/เย็น" ตามระยะห่าง
- เก็บ high score

---

## เฉลยแบบฝึกหัด Part 04

### เฉลย 4.1: FizzBuzz

```csharp
for (int i = 1; i <= 100; i++)
{
    string output = (i % 15 == 0) ? "FizzBuzz" :
                    (i % 3 == 0)  ? "Fizz" :
                    (i % 5 == 0)  ? "Buzz" :
                    i.ToString();
    Console.Write($"{output,-10}");
    if (i % 10 == 0) Console.WriteLine();
}
```

### เฉลย 4.2: Sieve of Eratosthenes

```csharp
int limit = 100;
bool[] isPrime = new bool[limit + 1];
Array.Fill(isPrime, true);
isPrime[0] = isPrime[1] = false;

for (int i = 2; i * i <= limit; i++)
{
    if (isPrime[i])
    {
        for (int j = i * i; j <= limit; j += i)
        {
            isPrime[j] = false;
        }
    }
}

var primes = new List<int>();
for (int i = 2; i <= limit; i++)
{
    if (isPrime[i]) primes.Add(i);
}

Console.WriteLine($"จำนวนเฉพาะระหว่าง 1-{limit} ({primes.Count} ตัว):");
for (int i = 0; i < primes.Count; i++)
{
    Console.Write($"{primes[i],4}");
    if ((i + 1) % 10 == 0) Console.WriteLine();
}
Console.WriteLine();
```

### เฉลย 4.3: เกมทายตัวเลขขั้นสูง

```csharp
using System;
using System.Collections.Generic;

Console.WriteLine("=== เกมทายตัวเลขขั้นสูง ===");

Random random = new();
int highScore = int.MaxValue;
bool playAgain = true;

while (playAgain)
{
    int secret = random.Next(1, 1001);
    int attempts = 0;
    const int MAX_ATTEMPTS = 10;
    bool won = false;
    var guessHistory = new List<int>();
    
    Console.WriteLine($"\nทายตัวเลข 1-1000 ใน {MAX_ATTEMPTS} ครั้ง!");
    
    while (attempts < MAX_ATTEMPTS && !won)
    {
        Console.Write($"\nครั้งที่ {attempts + 1}: ");
        if (!int.TryParse(Console.ReadLine(), out int guess) || guess < 1 || guess > 1000)
        {
            Console.WriteLine("กรุณาใส่ตัวเลข 1-1000");
            continue;
        }
        
        guessHistory.Add(guess);
        attempts++;
        
        if (guess == secret)
        {
            won = true;
            Console.WriteLine($"🎉 ถูกต้อง! ใช้ {attempts} ครั้ง");
            
            if (attempts < highScore)
            {
                highScore = attempts;
                Console.WriteLine($"🏆 High Score ใหม่: {highScore} ครั้ง!");
            }
            else
            {
                Console.WriteLine($"High Score ปัจจุบัน: {highScore} ครั้ง");
            }
        }
        else
        {
            int diff = Math.Abs(guess - secret);
            string temperature = diff switch
            {
                <= 10  => "🔥 ร้อนมาก!",
                <= 30  => "🌡️ ร้อน",
                <= 70  => "🌤️ อุ่น",
                <= 150 => "🌨️ เย็น",
                _      => "🧊 เย็นมาก"
            };
            
            string direction = guess < secret ? "⬆️ มากกว่านี้" : "⬇️ น้อยกว่านี้";
            Console.WriteLine($"{direction} | {temperature} (ห่าง {diff})");
            Console.WriteLine($"คะแนนสูงสุด: [{string.Join(", ", guessHistory)}]");
        }
    }
    
    if (!won)
    {
        Console.WriteLine($"\n😔 หมดครั้ง! คำตอบคือ {secret}");
    }
    
    Console.Write("\nเล่นอีกไหม? (y/n): ");
    playAgain = Console.ReadLine()?.ToLower() == "y";
}

Console.WriteLine($"\nสถิติดีที่สุด: {(highScore == int.MaxValue ? "ยังไม่เคยชนะ" : $"{highScore} ครั้ง")}");
```

---

## สรุป Part 04

ในส่วนนี้เราได้เรียนรู้:

1. **for Loop** - iteration แบบพื้นฐาน, nested
2. **while Loop** - loop ตาม condition
3. **do-while Loop** - loop อย่างน้อย 1 ครั้ง
4. **foreach Loop** - iterate ผ่าน collection
5. **break/continue** - control loop flow
6. **Nested Loops** - patterns, matrix
7. **yield return** - custom iterators
8. **IAsyncEnumerable** - async streaming
9. **Performance** - for vs foreach vs LINQ
10. **Loop Patterns** - sliding window, two-pointer

## ขั้นต่อไป

ใน Part 05 เราจะเรียนรู้เกี่ยวกับ:
- Methods/Functions
- Parameters (value, ref, out, in, params)
- Method overloading
- Optional parameters
- Named parameters
- Local functions
- Extension methods

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 04 | Steps 31-40*
