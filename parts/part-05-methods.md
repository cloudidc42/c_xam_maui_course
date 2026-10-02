# Part 05: Functions และ Methods
## Steps 41-50: การสร้างและใช้งาน Methods ใน C#

---

## Step 41: Methods พื้นฐาน

```csharp
// ============================================
// Method Declaration
// ============================================

// รูปแบบ:
// [access_modifier] [return_type] MethodName([parameters])
// {
//     // method body
//     return [value]; // ถ้า return type ไม่ใช่ void
// }

// Method ที่ไม่ return ค่า
void SayHello()
{
    Console.WriteLine("สวัสดี!");
}

// Method ที่ return ค่า
int Add(int a, int b)
{
    return a + b;
}

// Method ที่รับ parameter
void PrintName(string name)
{
    Console.WriteLine($"ชื่อ: {name}");
}

// เรียกใช้ method
SayHello();              // สวัสดี!
int result = Add(3, 5);  // result = 8
PrintName("สมชาย");      // ชื่อ: สมชาย

// ============================================
// Expression-bodied Methods (C# 6+)
// ============================================

// แบบปกติ
int Square(int n)
{
    return n * n;
}

// แบบ expression-bodied (กระชับกว่า)
int Square2(int n) => n * n;
void Greet(string name) => Console.WriteLine($"สวัสดี {name}!");
bool IsEven(int n) => n % 2 == 0;

Console.WriteLine(Square2(5));   // 25
Greet("สมศรี");                  // สวัสดี สมศรี!
Console.WriteLine(IsEven(4));    // True

// ============================================
// Static vs Instance Methods
// ============================================

class MathHelper
{
    // Static method - เรียกผ่าน class name
    public static double CircleArea(double radius)
        => Math.PI * radius * radius;
    
    public static int Factorial(int n)
    {
        if (n <= 1) return 1;
        return n * Factorial(n - 1);
    }
    
    // Instance method - ต้องสร้าง object ก่อน
    private double _value;
    
    public MathHelper(double value) => _value = value;
    
    public double Sqrt() => Math.Sqrt(_value);
    public double Square() => _value * _value;
}

// ใช้งาน
double area = MathHelper.CircleArea(5);         // Static: ไม่ต้อง new
Console.WriteLine($"Area: {area:F2}");

int fact = MathHelper.Factorial(5);             // Static
Console.WriteLine($"5! = {fact}");

var helper = new MathHelper(16);               // Instance: ต้อง new
Console.WriteLine($"√16 = {helper.Sqrt()}");   // 4
Console.WriteLine($"16² = {helper.Square()}"); // 256
```

---

## Step 42: Parameters

```csharp
// ============================================
// Value Parameters (Default)
// ============================================

void IncrementByValue(int x)
{
    x++; // แก้ไข copy - ไม่กระทบ original
    Console.WriteLine($"Inside: {x}");
}

int num = 5;
IncrementByValue(num);
Console.WriteLine($"After: {num}"); // ยังเป็น 5

// ============================================
// ref Parameters - ส่ง reference
// ============================================

void IncrementByRef(ref int x)
{
    x++; // แก้ไข original
    Console.WriteLine($"Inside: {x}");
}

int num2 = 5;
IncrementByRef(ref num2); // ต้องใส่ ref ตอนเรียกด้วย
Console.WriteLine($"After: {num2}"); // 6

// ============================================
// out Parameters - รับค่ากลับหลายตัว
// ============================================

bool TryDivide(int a, int b, out int result, out string error)
{
    if (b == 0)
    {
        result = 0;
        error = "Cannot divide by zero";
        return false;
    }
    
    result = a / b;
    error = string.Empty;
    return true;
}

if (TryDivide(10, 2, out int quotient, out string errorMsg))
{
    Console.WriteLine($"10 / 2 = {quotient}");
}

// ไม่สนใจ out บางตัว ใช้ _ (discard)
if (TryDivide(10, 0, out _, out string err))
{
    // ...
}
else
{
    Console.WriteLine($"Error: {err}");
}

// ============================================
// in Parameters - ส่ง readonly reference
// ============================================

// in = ส่ง reference แต่ห้ามแก้ไข (ประหยัด memory สำหรับ struct ขนาดใหญ่)
struct BigStruct
{
    public double X, Y, Z, W; // 32 bytes
    public double A, B, C, D; // 32 bytes รวม 64 bytes
}

double ProcessStruct(in BigStruct s)
{
    // s.X = 1; // ERROR: ไม่สามารถแก้ไข in parameter ได้
    return s.X + s.Y + s.Z + s.W;
}

// ============================================
// params - จำนวน argument ไม่จำกัด
// ============================================

int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers)
        total += n;
    return total;
}

Console.WriteLine(Sum(1));              // 1
Console.WriteLine(Sum(1, 2, 3));       // 6
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 15

// ส่ง array ก็ได้
int[] arr = { 10, 20, 30 };
Console.WriteLine(Sum(arr)); // 60

// params collection (C# 13+)
string Join(string separator, params ReadOnlySpan<string> items)
{
    return string.Join(separator, items.ToArray());
}
```

---

## Step 43: Optional และ Named Parameters

```csharp
// ============================================
// Optional Parameters (Default Values)
// ============================================

// ต้องวางไว้ท้าย parameter list
void CreateUser(
    string name, 
    int age = 0, 
    string email = "", 
    bool isActive = true,
    string role = "User")
{
    Console.WriteLine($"Name: {name}, Age: {age}, Email: {email}, Active: {isActive}, Role: {role}");
}

// เรียกแบบต่างๆ
CreateUser("สมชาย");                              // ใช้ค่า default ทั้งหมด
CreateUser("สมหญิง", 25);                          // กำหนด age
CreateUser("สมศรี", 30, "somsr@email.com");       // กำหนด age และ email
CreateUser("สมพงษ์", 35, role: "Admin");          // ข้าม email, กำหนด role

// ============================================
// Named Parameters
// ============================================

// Named parameters ช่วยให้ code อ่านง่ายขึ้น
CreateUser(
    name: "ทดสอบ",
    role: "Manager",
    isActive: false,
    age: 40
    // email ใช้ default
);

// ============================================
// Method Overloading
// ============================================

class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public decimal Add(decimal a, decimal b) => a + b;
    public int Add(int a, int b, int c) => a + b + c;
    public string Add(string a, string b) => a + b;
    
    // Print overloads
    public void Print(int value) => Console.WriteLine($"int: {value}");
    public void Print(double value) => Console.WriteLine($"double: {value}");
    public void Print(string value) => Console.WriteLine($"string: {value}");
    public void Print(bool value) => Console.WriteLine($"bool: {value}");
}

var calc = new Calculator();
Console.WriteLine(calc.Add(1, 2));           // 3 (int)
Console.WriteLine(calc.Add(1.5, 2.5));       // 4.0 (double)
Console.WriteLine(calc.Add(1, 2, 3));        // 6 (3 params)
Console.WriteLine(calc.Add("Hello", " World")); // Hello World (string)
```

---

## Step 44: Local Functions

```csharp
// ============================================
// Local Functions (C# 7+)
// ============================================

// Local function คือ function ที่ประกาศภายใน method อื่น

void ProcessData(int[] data)
{
    // Local function - เห็นได้เฉพาะใน ProcessData
    int Square(int n) => n * n;
    bool IsOdd(int n) => n % 2 != 0;
    
    foreach (int item in data)
    {
        if (IsOdd(item))
            Console.WriteLine($"{item}² = {Square(item)}");
    }
}

ProcessData(new[] { 1, 2, 3, 4, 5 });

// ============================================
// Static Local Functions (C# 8+)
// ============================================

void OuterMethod()
{
    int multiplier = 3;
    
    // ❌ ไม่แนะนำ: อ่าน outer scope (hidden dependency)
    int MultiplyBad(int x) => x * multiplier;
    
    // ✅ แนะนำ: static local function - ไม่สามารถ capture outer scope
    static int MultiplyGood(int x, int mult) => x * mult;
    
    Console.WriteLine(MultiplyBad(5));           // 15
    Console.WriteLine(MultiplyGood(5, multiplier)); // 15
}

// ============================================
// Recursive Local Functions
// ============================================

long ComputeFactorial(int n)
{
    // Guard clause
    if (n < 0) throw new ArgumentException("n must be non-negative");
    
    // Local recursive function
    static long Factorial(int n) => n <= 1 ? 1 : n * Factorial(n - 1);
    
    return Factorial(n);
}

Console.WriteLine(ComputeFactorial(10)); // 3628800

// ============================================
// Local function กับ yield return
// ============================================

IEnumerable<T> FilterItems<T>(IEnumerable<T> items, Func<T, bool> predicate)
{
    // Validate ก่อน
    ArgumentNullException.ThrowIfNull(items);
    ArgumentNullException.ThrowIfNull(predicate);
    
    // Local function สำหรับ iterator
    return FilterCore(items, predicate);
    
    static IEnumerable<T> FilterCore(IEnumerable<T> items, Func<T, bool> pred)
    {
        foreach (var item in items)
        {
            if (pred(item))
                yield return item;
        }
    }
}

var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
var evens = FilterItems(numbers, n => n % 2 == 0);
Console.WriteLine(string.Join(", ", evens)); // 2, 4, 6, 8, 10
```

---

## Step 45: Extension Methods

```csharp
// ============================================
// Extension Methods
// ============================================

// Extension method ต้องอยู่ใน static class
// และ parameter แรกต้องใช้ this keyword

static class StringExtensions
{
    // Truncate string ถ้ายาวเกิน
    public static string Truncate(this string str, int maxLength, string suffix = "...")
    {
        if (str.Length <= maxLength) return str;
        return str[..(maxLength - suffix.Length)] + suffix;
    }
    
    // Title Case
    public static string ToTitleCase(this string str)
    {
        if (string.IsNullOrEmpty(str)) return str;
        return System.Globalization.CultureInfo.CurrentCulture
            .TextInfo.ToTitleCase(str.ToLower());
    }
    
    // Remove special characters
    public static string ToSlug(this string str)
    {
        return str.ToLower()
            .Replace(" ", "-")
            .Replace("_", "-");
    }
    
    // ตรวจสอบว่าเป็น email
    public static bool IsValidEmail(this string str)
    {
        return System.Text.RegularExpressions.Regex.IsMatch(
            str, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    }
    
    // Repeat string
    public static string Repeat(this string str, int count)
        => string.Concat(Enumerable.Repeat(str, count));
}

// ใช้งาน
string longText = "สวัสดีทุกคน ยินดีต้อนรับสู่หลักสูตร C# Programming";
Console.WriteLine(longText.Truncate(20));        // สวัสดีทุกคน ยินดีต้อ...
Console.WriteLine("hello world".ToTitleCase());  // Hello World
Console.WriteLine("My Article Title".ToSlug());  // my-article-title
Console.WriteLine("test@email.com".IsValidEmail()); // True
Console.WriteLine("=".Repeat(20));               // ====================

// ============================================
// Extension Methods บน Collections
// ============================================

static class CollectionExtensions
{
    // Chunk list ออกเป็น batch
    public static IEnumerable<IEnumerable<T>> Chunk<T>(
        this IEnumerable<T> source, int chunkSize)
    {
        var chunk = new List<T>(chunkSize);
        foreach (var item in source)
        {
            chunk.Add(item);
            if (chunk.Count == chunkSize)
            {
                yield return chunk.AsReadOnly();
                chunk = new List<T>(chunkSize);
            }
        }
        if (chunk.Count > 0)
            yield return chunk.AsReadOnly();
    }
    
    // Shuffle list
    public static IList<T> Shuffle<T>(this IList<T> list)
    {
        var result = list.ToList();
        var random = new Random();
        for (int i = result.Count - 1; i > 0; i--)
        {
            int j = random.Next(i + 1);
            (result[i], result[j]) = (result[j], result[i]);
        }
        return result;
    }
    
    // Flatten nested collections
    public static IEnumerable<T> Flatten<T>(this IEnumerable<IEnumerable<T>> source)
        => source.SelectMany(x => x);
    
    // Safe FirstOrDefault with default
    public static T FirstOrDefault<T>(
        this IEnumerable<T> source, 
        Func<T, bool> predicate, 
        T defaultValue)
    {
        foreach (var item in source)
        {
            if (predicate(item)) return item;
        }
        return defaultValue;
    }
}

var items = Enumerable.Range(1, 10).ToList();

// Chunk
foreach (var chunk in items.Chunk(3))
    Console.WriteLine(string.Join(", ", chunk));
// 1, 2, 3
// 4, 5, 6
// 7, 8, 9
// 10

// Shuffle
var shuffled = items.Shuffle();
Console.WriteLine(string.Join(", ", shuffled));

// Extension บน numeric
static class NumberExtensions
{
    public static bool IsBetween(this int value, int min, int max)
        => value >= min && value <= max;
    
    public static int Clamp(this int value, int min, int max)
        => Math.Max(min, Math.Min(max, value));
    
    public static string ToOrdinal(this int n)
    {
        return (n % 100) switch
        {
            11 or 12 or 13 => $"{n}th",
            _ => (n % 10) switch
            {
                1 => $"{n}st",
                2 => $"{n}nd",
                3 => $"{n}rd",
                _ => $"{n}th"
            }
        };
    }
}

Console.WriteLine(15.IsBetween(10, 20)); // True
Console.WriteLine(25.Clamp(0, 20));       // 20
Console.WriteLine(1.ToOrdinal());          // 1st
Console.WriteLine(2.ToOrdinal());          // 2nd
Console.WriteLine(3.ToOrdinal());          // 3rd
Console.WriteLine(11.ToOrdinal());         // 11th
```

---

## Step 46: Delegates และ Functional Parameters

```csharp
// ============================================
// Func<> Delegate
// ============================================

// Func<input1, input2, ..., output>
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isLong = s => s.Length > 10;
Func<double, double, double> power = Math.Pow;

Console.WriteLine(square(5));         // 25
Console.WriteLine(add(3, 7));         // 10
Console.WriteLine(isLong("Hello"));   // False

// ============================================
// Action<> Delegate
// ============================================

// Action ไม่มี return value
Action<string> print = msg => Console.WriteLine(msg);
Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);

print("Hello!");
printSum(3, 4);

// ============================================
// Predicate<T> Delegate
// ============================================

// Predicate = Func<T, bool>
Predicate<int> isPositive = n => n > 0;
Predicate<string> isNotEmpty = s => !string.IsNullOrEmpty(s);

var numbers = new List<int> { -3, -1, 0, 2, 5, -2, 8 };
var positiveNumbers = numbers.FindAll(isPositive);
Console.WriteLine(string.Join(", ", positiveNumbers)); // 2, 5, 8

// ============================================
// Higher-Order Functions
// ============================================

// Function ที่รับ function เป็น parameter
T Apply<T>(T value, Func<T, T> transform) => transform(value);

int doubled = Apply(5, x => x * 2);     // 10
string upper = Apply("hello", s => s.ToUpper()); // HELLO

// Function ที่ return function (currying)
Func<int, int> CreateMultiplier(int multiplier)
    => x => x * multiplier;

var triple = CreateMultiplier(3);
var quadruple = CreateMultiplier(4);

Console.WriteLine(triple(5));    // 15
Console.WriteLine(quadruple(5)); // 20

// ============================================
// Method ที่รับ function parameter
// ============================================

class DataProcessor
{
    // Transform data
    public static List<TResult> Transform<T, TResult>(
        List<T> items, 
        Func<T, TResult> transformer)
    {
        return items.Select(transformer).ToList();
    }
    
    // Filter data
    public static List<T> Filter<T>(
        List<T> items, 
        Func<T, bool> predicate)
    {
        return items.Where(predicate).ToList();
    }
    
    // Process with side effects
    public static void Process<T>(
        List<T> items, 
        Action<T> action)
    {
        foreach (var item in items)
            action(item);
    }
    
    // Aggregate
    public static TAccumulate Reduce<T, TAccumulate>(
        List<T> items,
        TAccumulate seed,
        Func<TAccumulate, T, TAccumulate> accumulator)
    {
        return items.Aggregate(seed, accumulator);
    }
}

var nums = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// Transform: คูณ 2
var doubled2 = DataProcessor.Transform(nums, n => n * 2);
Console.WriteLine(string.Join(", ", doubled2));

// Filter: เฉพาะตัวคู่
var evens2 = DataProcessor.Filter(nums, n => n % 2 == 0);
Console.WriteLine(string.Join(", ", evens2));

// Process: แสดงผล
DataProcessor.Process(evens2, n => Console.Write($"{n} "));
Console.WriteLine();

// Reduce: รวมทั้งหมด
int sum2 = DataProcessor.Reduce(nums, 0, (acc, n) => acc + n);
Console.WriteLine($"Sum: {sum2}");
```

---

## Step 47: Recursive Methods

```csharp
// ============================================
// Recursion พื้นฐาน
// ============================================

// Factorial
long Factorial(int n)
{
    if (n < 0) throw new ArgumentException("n must be >= 0");
    if (n <= 1) return 1;         // Base case
    return n * Factorial(n - 1);  // Recursive case
}

Console.WriteLine(Factorial(10)); // 3628800

// Power
double Power(double base2, int exp)
{
    if (exp == 0) return 1;        // Base case
    if (exp < 0) return 1.0 / Power(base2, -exp);
    return base2 * Power(base2, exp - 1); // Recursive
}

Console.WriteLine(Power(2, 10)); // 1024

// ============================================
// Tree Traversal
// ============================================

class TreeNode
{
    public int Value { get; }
    public TreeNode? Left { get; set; }
    public TreeNode? Right { get; set; }
    
    public TreeNode(int value) => Value = value;
}

// Inorder traversal (Left, Root, Right)
void InOrder(TreeNode? node)
{
    if (node == null) return;
    InOrder(node.Left);
    Console.Write($"{node.Value} ");
    InOrder(node.Right);
}

// Preorder traversal (Root, Left, Right)
void PreOrder(TreeNode? node)
{
    if (node == null) return;
    Console.Write($"{node.Value} ");
    PreOrder(node.Left);
    PreOrder(node.Right);
}

// สร้าง Binary Search Tree
TreeNode root = new(5)
{
    Left = new TreeNode(3) 
    { 
        Left = new TreeNode(1), 
        Right = new TreeNode(4) 
    },
    Right = new TreeNode(7) 
    { 
        Left = new TreeNode(6), 
        Right = new TreeNode(9) 
    }
};

Console.Write("InOrder: ");
InOrder(root);    // 1 3 4 5 6 7 9
Console.WriteLine();

Console.Write("PreOrder: ");
PreOrder(root);   // 5 3 1 4 7 6 9
Console.WriteLine();

// ============================================
// Memoization - Cache ผลลัพธ์
// ============================================

class Memoizer
{
    private readonly Dictionary<long, long> _cache = new();
    
    public long Fibonacci(long n)
    {
        if (n <= 1) return n;
        
        if (_cache.TryGetValue(n, out long cached))
            return cached;
        
        long result = Fibonacci(n - 1) + Fibonacci(n - 2);
        _cache[n] = result;
        return result;
    }
}

var memo = new Memoizer();
Console.WriteLine(memo.Fibonacci(50));  // 12586269025

// ============================================
// Tail Recursion
// ============================================

// ไม่ tail recursive (stack อาจ overflow)
long SumNaive(int n) => n <= 0 ? 0 : n + SumNaive(n - 1);

// Tail recursive (ประหยัด stack ในบางภาษา แต่ C# ไม่ optimize)
long SumTail(int n, long accumulator = 0) 
    => n <= 0 ? accumulator : SumTail(n - 1, accumulator + n);

Console.WriteLine(SumNaive(100));  // 5050
Console.WriteLine(SumTail(100));   // 5050
```

---

## Step 48: Method Chaining และ Fluent Interface

```csharp
// ============================================
// Method Chaining
// ============================================

class QueryBuilder
{
    private string _table = string.Empty;
    private List<string> _conditions = new();
    private List<string> _columns = new();
    private int? _limit;
    private string? _orderBy;
    private bool _orderDesc;
    
    public QueryBuilder From(string table)
    {
        _table = table;
        return this; // return this เพื่อ chaining
    }
    
    public QueryBuilder Select(params string[] columns)
    {
        _columns.AddRange(columns);
        return this;
    }
    
    public QueryBuilder Where(string condition)
    {
        _conditions.Add(condition);
        return this;
    }
    
    public QueryBuilder OrderBy(string column, bool descending = false)
    {
        _orderBy = column;
        _orderDesc = descending;
        return this;
    }
    
    public QueryBuilder Limit(int count)
    {
        _limit = count;
        return this;
    }
    
    public string Build()
    {
        var columns = _columns.Count > 0 ? string.Join(", ", _columns) : "*";
        var query = $"SELECT {columns} FROM {_table}";
        
        if (_conditions.Count > 0)
            query += $" WHERE {string.Join(" AND ", _conditions)}";
        
        if (_orderBy != null)
            query += $" ORDER BY {_orderBy}{(_orderDesc ? " DESC" : "")}";
        
        if (_limit.HasValue)
            query += $" LIMIT {_limit}";
        
        return query;
    }
}

// ใช้งาน - อ่านเหมือนภาษาธรรมชาติ
string sql = new QueryBuilder()
    .From("users")
    .Select("id", "name", "email")
    .Where("age >= 18")
    .Where("is_active = true")
    .OrderBy("name")
    .Limit(10)
    .Build();

Console.WriteLine(sql);
// SELECT id, name, email FROM users WHERE age >= 18 AND is_active = true ORDER BY name LIMIT 10

// ============================================
// Fluent Validation
// ============================================

class UserValidator
{
    private readonly string _username;
    private readonly List<string> _errors = new();
    
    public UserValidator(string username) => _username = username;
    
    public UserValidator MinLength(int min)
    {
        if (_username.Length < min)
            _errors.Add($"Username must be at least {min} characters");
        return this;
    }
    
    public UserValidator MaxLength(int max)
    {
        if (_username.Length > max)
            _errors.Add($"Username must be at most {max} characters");
        return this;
    }
    
    public UserValidator NoSpaces()
    {
        if (_username.Contains(' '))
            _errors.Add("Username cannot contain spaces");
        return this;
    }
    
    public UserValidator AlphanumericOnly()
    {
        if (!_username.All(c => char.IsLetterOrDigit(c) || c == '_'))
            _errors.Add("Username can only contain letters, numbers, and underscores");
        return this;
    }
    
    public (bool IsValid, IReadOnlyList<string> Errors) Validate()
        => (!_errors.Any(), _errors.AsReadOnly());
}

var (isValid, errors) = new UserValidator("john doe 123!")
    .MinLength(3)
    .MaxLength(20)
    .NoSpaces()
    .AlphanumericOnly()
    .Validate();

Console.WriteLine($"Valid: {isValid}");
foreach (string error in errors)
    Console.WriteLine($"- {error}");
```

---

## Step 49: Generic Methods

```csharp
// ============================================
// Generic Methods
// ============================================

// Generic method - ทำงานได้กับหลายชนิดข้อมูล
T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

Console.WriteLine(Max(3, 7));         // 7
Console.WriteLine(Max(3.14, 2.71));   // 3.14
Console.WriteLine(Max("apple", "mango")); // mango

// Swap generic
void Swap<T>(ref T a, ref T b) => (a, b) = (b, a);

int x = 5, y = 10;
Swap(ref x, ref y);
Console.WriteLine($"x={x}, y={y}"); // x=10, y=5

// ============================================
// Generic Constraints
// ============================================

// where T : class - T ต้องเป็น reference type
T? FindFirst<T>(List<T> list, Func<T, bool> predicate) where T : class
    => list.FirstOrDefault(predicate);

// where T : struct - T ต้องเป็น value type
T? TryParse<T>(string input) where T : struct
{
    try
    {
        return (T)Convert.ChangeType(input, typeof(T));
    }
    catch
    {
        return null;
    }
}

// where T : new() - T ต้องมี parameterless constructor
T CreateDefault<T>() where T : new() => new T();

// where T : IComparable<T> - T ต้องมี CompareTo
T Clamp2<T>(T value, T min, T max) where T : IComparable<T>
{
    if (value.CompareTo(min) < 0) return min;
    if (value.CompareTo(max) > 0) return max;
    return value;
}

Console.WriteLine(Clamp2(15, 0, 10));   // 10
Console.WriteLine(Clamp2(-5, 0, 10));   // 0
Console.WriteLine(Clamp2(5, 0, 10));    // 5

// ============================================
// Generic Utility Methods
// ============================================

static class Utils
{
    // Safe cast
    public static T? As<T>(object? obj) where T : class
        => obj as T;
    
    // Default if null
    public static T DefaultIfNull<T>(T? value, T defaultValue) where T : class
        => value ?? defaultValue;
    
    // Memoize function
    public static Func<T, TResult> Memoize<T, TResult>(Func<T, TResult> func) 
        where T : notnull
    {
        var cache = new Dictionary<T, TResult>();
        return input =>
        {
            if (!cache.TryGetValue(input, out var result))
            {
                result = func(input);
                cache[input] = result;
            }
            return result;
        };
    }
    
    // Retry
    public static T Retry<T>(Func<T> action, int maxAttempts = 3)
    {
        Exception? lastException = null;
        for (int i = 0; i < maxAttempts; i++)
        {
            try
            {
                return action();
            }
            catch (Exception ex)
            {
                lastException = ex;
                if (i < maxAttempts - 1)
                    Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))).Wait();
            }
        }
        throw new Exception($"Failed after {maxAttempts} attempts", lastException);
    }
}

// Memoize Fibonacci
var memoFib = Utils.Memoize<int, long>(n =>
{
    if (n <= 1) return n;
    // Note: This won't memoize recursive calls
    return n; // simplified
});
```

---

## Step 50: Method สำหรับ MAUI (Preview)

```csharp
// ============================================
// Methods ที่ใช้บ่อยใน MAUI Development
// ============================================

// Utility methods สำหรับ UI
static class UiHelper
{
    // แปลง hex color เป็น Color
    public static Color? ParseColor(string hexColor)
    {
        try
        {
            return Color.FromArgb(hexColor);
        }
        catch
        {
            return null;
        }
    }
    
    // Format file size
    public static string FormatFileSize(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB", "TB" };
        double len = bytes;
        int order = 0;
        while (len >= 1024 && order < sizes.Length - 1)
        {
            order++;
            len /= 1024;
        }
        return $"{len:0.##} {sizes[order]}";
    }
    
    // Format ระยะเวลา
    public static string FormatDuration(TimeSpan duration)
    {
        return duration switch
        {
            { TotalSeconds: < 60 } => $"{(int)duration.TotalSeconds} วินาที",
            { TotalMinutes: < 60 } => $"{(int)duration.TotalMinutes} นาที",
            { TotalHours: < 24 }   => $"{(int)duration.TotalHours} ชั่วโมง",
            _                      => $"{(int)duration.TotalDays} วัน"
        };
    }
    
    // Truncate text สำหรับ UI
    public static string TruncateForDisplay(string text, int maxLength = 50)
    {
        if (string.IsNullOrEmpty(text)) return string.Empty;
        return text.Length <= maxLength ? text : $"{text[..(maxLength - 3)]}...";
    }
    
    // Format relative time (เช่น "2 ชั่วโมงที่แล้ว")
    public static string ToRelativeTime(DateTime dateTime)
    {
        TimeSpan diff = DateTime.Now - dateTime;
        return diff switch
        {
            { TotalSeconds: < 60 }  => "เมื่อกี้นี้",
            { TotalMinutes: < 60 }  => $"{(int)diff.TotalMinutes} นาทีที่แล้ว",
            { TotalHours: < 24 }    => $"{(int)diff.TotalHours} ชั่วโมงที่แล้ว",
            { TotalDays: < 7 }      => $"{(int)diff.TotalDays} วันที่แล้ว",
            { TotalDays: < 30 }     => $"{(int)(diff.TotalDays / 7)} สัปดาห์ที่แล้ว",
            { TotalDays: < 365 }    => $"{(int)(diff.TotalDays / 30)} เดือนที่แล้ว",
            _                       => $"{(int)(diff.TotalDays / 365)} ปีที่แล้ว"
        };
    }
}

// ทดสอบ
Console.WriteLine(UiHelper.FormatFileSize(1024));           // 1 KB
Console.WriteLine(UiHelper.FormatFileSize(1_048_576));      // 1 MB
Console.WriteLine(UiHelper.FormatFileSize(1_073_741_824));  // 1 GB

Console.WriteLine(UiHelper.FormatDuration(TimeSpan.FromSeconds(45)));   // 45 วินาที
Console.WriteLine(UiHelper.FormatDuration(TimeSpan.FromMinutes(90)));   // 1 ชั่วโมง
Console.WriteLine(UiHelper.FormatDuration(TimeSpan.FromDays(3)));       // 3 วัน

Console.WriteLine(UiHelper.ToRelativeTime(DateTime.Now.AddMinutes(-30)));  // 30 นาทีที่แล้ว
Console.WriteLine(UiHelper.ToRelativeTime(DateTime.Now.AddDays(-2)));      // 2 วันที่แล้ว
```

---

## แบบฝึกหัด Part 05

### แบบฝึกหัดที่ 5.1: String Processing Library
สร้าง static class ที่มี Extension Methods สำหรับ string:
- `IsNumeric()` - ตรวจสอบว่าเป็นตัวเลขทั้งหมด
- `IsPalindrome()` - ตรวจสอบ palindrome
- `WordCount()` - นับจำนวนคำ
- `CountOccurrences(char c)` - นับจำนวนครั้งที่ตัวอักษรปรากฏ

### แบบฝึกหัดที่ 5.2: Generic Stack
สร้าง Generic Stack class ที่มี:
- `Push(T item)` - เพิ่มไปด้านบน
- `Pop()` - ดึงออกจากด้านบน
- `Peek()` - ดูค่าด้านบน ไม่ลบออก
- `IsEmpty` property

### แบบฝึกหัดที่ 5.3: Math Library
สร้าง static class MathLib ที่มี:
- `GCD(a, b)` - หา ห.ร.ม.
- `LCM(a, b)` - หา ค.ร.น.
- `IsPrime(n)` - ตรวจสอบจำนวนเฉพาะ
- `PrimeFactors(n)` - หาตัวประกอบที่เป็นจำนวนเฉพาะ

---

## เฉลยแบบฝึกหัด Part 05

### เฉลย 5.1: String Processing Library

```csharp
static class StringLib
{
    public static bool IsNumeric(this string s)
        => s.All(char.IsDigit);
    
    public static bool IsPalindrome(this string s)
    {
        string cleaned = new string(s.ToLower().Where(char.IsLetterOrDigit).ToArray());
        return cleaned == new string(cleaned.Reverse().ToArray());
    }
    
    public static int WordCount(this string s)
        => s.Split(new[] { ' ', '\t', '\n', '\r' }, 
            StringSplitOptions.RemoveEmptyEntries).Length;
    
    public static int CountOccurrences(this string s, char c)
        => s.Count(ch => ch == c);
}

Console.WriteLine("12345".IsNumeric());       // True
Console.WriteLine("12a45".IsNumeric());       // False
Console.WriteLine("racecar".IsPalindrome());  // True
Console.WriteLine("A man a plan a canal Panama".IsPalindrome()); // True
Console.WriteLine("Hello World C#".WordCount()); // 3
Console.WriteLine("Hello World".CountOccurrences('l')); // 3
```

### เฉลย 5.2: Generic Stack

```csharp
class GenericStack<T>
{
    private readonly List<T> _items = new();
    
    public int Count => _items.Count;
    public bool IsEmpty => _items.Count == 0;
    
    public void Push(T item) => _items.Add(item);
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        T item = _items[^1]; // last element
        _items.RemoveAt(_items.Count - 1);
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _items[^1];
    }
    
    public void Clear() => _items.Clear();
    
    public override string ToString()
        => $"[{string.Join(", ", _items)}]";
}

var stack = new GenericStack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);
Console.WriteLine(stack);         // [1, 2, 3]
Console.WriteLine(stack.Pop());   // 3
Console.WriteLine(stack.Peek());  // 2
Console.WriteLine(stack);         // [1, 2]
```

### เฉลย 5.3: Math Library

```csharp
static class MathLib
{
    public static long GCD(long a, long b)
    {
        a = Math.Abs(a);
        b = Math.Abs(b);
        while (b != 0)
        {
            long t = b;
            b = a % b;
            a = t;
        }
        return a;
    }
    
    public static long LCM(long a, long b)
    {
        if (a == 0 || b == 0) return 0;
        return Math.Abs(a / GCD(a, b) * b);
    }
    
    public static bool IsPrime(long n)
    {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (long i = 3; i * i <= n; i += 2)
            if (n % i == 0) return false;
        return true;
    }
    
    public static List<long> PrimeFactors(long n)
    {
        var factors = new List<long>();
        for (long i = 2; i * i <= n; i++)
        {
            while (n % i == 0)
            {
                factors.Add(i);
                n /= i;
            }
        }
        if (n > 1) factors.Add(n);
        return factors;
    }
}

Console.WriteLine(MathLib.GCD(48, 18));    // 6
Console.WriteLine(MathLib.LCM(4, 6));     // 12
Console.WriteLine(MathLib.IsPrime(17));    // True
Console.WriteLine(MathLib.IsPrime(15));    // False
var factors = MathLib.PrimeFactors(360);
Console.WriteLine(string.Join(" × ", factors)); // 2 × 2 × 2 × 3 × 3 × 5
```

---

## สรุป Part 05

ในส่วนนี้เราได้เรียนรู้:

1. **Methods พื้นฐาน** - Declaration, Expression-bodied, Static/Instance
2. **Parameters** - Value, ref, out, in, params
3. **Optional Parameters** - Default values
4. **Named Parameters** - Readability
5. **Method Overloading** - Same name, different params
6. **Local Functions** - Nested functions
7. **Extension Methods** - Adding methods to existing types
8. **Delegates** - Func, Action, Predicate
9. **Recursion** - Base case, Memoization
10. **Generic Methods** - Type-safe reusable methods

## ขั้นต่อไป

ใน Part 06 เราจะเรียนรู้เกี่ยวกับ:
- Arrays
- List<T>
- Dictionary<K, V>
- HashSet<T>
- Queue<T> and Stack<T>
- Collections best practices

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 05 | Steps 41-50*
