# Part 11: LINQ - Language Integrated Query
## Steps 101-110: การค้นหาและแปลงข้อมูล

---

## Step 101: LINQ Basics

```csharp
// ============================================
// LINQ คืออะไร
// ============================================

/*
 * LINQ = Language Integrated Query
 * ช่วยให้ query ข้อมูลได้เหมือน SQL แต่อยู่ใน C#
 * ทำงานได้กับ Collections, Database, XML, JSON
 */

using System.Linq;

// ข้อมูลตัวอย่าง
var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
var names = new[] { "สมชาย", "สมหญิง", "สมศรี", "วิชัย", "วิภา" };

var students = new List<Student5>
{
    new Student5(1, "อลิส", 22, "CS", 3.8),
    new Student5(2, "บ็อบ", 20, "IT", 3.2),
    new Student5(3, "แครอล", 23, "CS", 3.9),
    new Student5(4, "เดวิด", 21, "IT", 2.8),
    new Student5(5, "อีฟ", 22, "CS", 3.5),
    new Student5(6, "แฟรงก์", 24, "Math", 3.7),
    new Student5(7, "เกรซ", 20, "CS", 3.1),
};

public record Student5(int Id, string Name, int Age, string Major, double GPA);

// ============================================
// Query Syntax vs Method Syntax
// ============================================

// Query Syntax (คล้าย SQL)
var csStudents_query = 
    from s in students
    where s.Major == "CS"
    orderby s.GPA descending
    select s;

// Method Syntax (Lambda)
var csStudents_method = students
    .Where(s => s.Major == "CS")
    .OrderByDescending(s => s.GPA);

// ทั้งสองให้ผลเหมือนกัน
foreach (var s in csStudents_method)
    Console.WriteLine($"{s.Name}: GPA={s.GPA}");

// ============================================
// Basic Operations
// ============================================

// Where - กรอง
var adults = students.Where(s => s.Age >= 22);

// Select - แปลง
var names2 = students.Select(s => s.Name);
var nameGpa = students.Select(s => new { s.Name, s.GPA });

// OrderBy, OrderByDescending
var byAge = students.OrderBy(s => s.Age);
var byGpaDesc = students.OrderByDescending(s => s.GPA);

// ThenBy (secondary sort)
var sorted = students
    .OrderBy(s => s.Major)
    .ThenByDescending(s => s.GPA);

// Take, Skip (Pagination)
int pageSize = 3, page = 1;
var pageData = students
    .Skip((page - 1) * pageSize)
    .Take(pageSize);

// First, FirstOrDefault
var topStudent = students.First(s => s.GPA >= 3.9);
var itStudent = students.FirstOrDefault(s => s.Major == "IT");
var noneFound = students.FirstOrDefault(s => s.GPA > 4.0); // null

// Single, SingleOrDefault (expect exactly 1)
var student1 = students.Single(s => s.Id == 1);

// Last, LastOrDefault
var lastStudent = students.Last();

// Any, All, Count
bool hasHighGpa = students.Any(s => s.GPA >= 3.8);
bool allAdults = students.All(s => s.Age >= 18);
int csCount = students.Count(s => s.Major == "CS");

// Sum, Average, Min, Max
double avgGpa = students.Average(s => s.GPA);
double minGpa = students.Min(s => s.GPA);
double maxGpa = students.Max(s => s.GPA);
int totalAge = students.Sum(s => s.Age);

Console.WriteLine($"Count CS: {csCount}, Avg GPA: {avgGpa:F2}");
Console.WriteLine($"Has high GPA: {hasHighGpa}, All adults: {allAdults}");
```

---

## Step 102: Grouping และ Joining

```csharp
// ============================================
// GroupBy
// ============================================

// Group students by major
var byMajor = students.GroupBy(s => s.Major);

foreach (var group in byMajor)
{
    Console.WriteLine($"\n{group.Key}:");
    foreach (var s in group)
        Console.WriteLine($"  {s.Name}: GPA={s.GPA}");
}

// GroupBy with projection
var majorStats = students
    .GroupBy(s => s.Major)
    .Select(g => new
    {
        Major = g.Key,
        Count = g.Count(),
        AvgGpa = g.Average(s => s.GPA),
        TopStudent = g.OrderByDescending(s => s.GPA).First().Name
    })
    .OrderByDescending(x => x.AvgGpa);

foreach (var stat in majorStats)
    Console.WriteLine($"{stat.Major}: {stat.Count} students, avg GPA={stat.AvgGpa:F2}, top={stat.TopStudent}");

// ============================================
// Join
// ============================================

var courses = new[]
{
    new { CourseId = 1, CourseName = "Programming", MajorReq = "CS" },
    new { CourseId = 2, CourseName = "Database", MajorReq = "CS" },
    new { CourseId = 3, CourseName = "Networks", MajorReq = "IT" },
    new { CourseId = 4, CourseName = "Calculus", MajorReq = "Math" },
};

var enrollments = new[]
{
    new { StudentId = 1, CourseId = 1 },
    new { StudentId = 1, CourseId = 2 },
    new { StudentId = 2, CourseId = 3 },
    new { StudentId = 3, CourseId = 1 },
    new { StudentId = 3, CourseId = 2 },
};

// Inner Join
var studentCourses = students
    .Join(enrollments,
        s => s.Id,
        e => e.StudentId,
        (s, e) => new { s.Name, e.CourseId })
    .Join(courses,
        se => se.CourseId,
        c => c.CourseId,
        (se, c) => new { se.Name, c.CourseName });

foreach (var sc in studentCourses)
    Console.WriteLine($"{sc.Name} enrolled in {sc.CourseName}");

// GroupJoin (Left Join equivalent)
var studentWithCourses = students
    .GroupJoin(
        enrollments,
        s => s.Id,
        e => e.StudentId,
        (s, enrList) => new
        {
            Student = s.Name,
            Courses = enrList.Count(),
            CourseNames = enrList
                .Join(courses, e => e.CourseId, c => c.CourseId, (e, c) => c.CourseName)
                .ToList()
        });

foreach (var swc in studentWithCourses)
    Console.WriteLine($"{swc.Student}: {swc.Courses} courses - {string.Join(", ", swc.CourseNames)}");
```

---

## Step 103: Advanced LINQ Operations

```csharp
// ============================================
// SelectMany - Flatten
// ============================================

var departments = new[]
{
    new { Name = "Engineering", Employees = new[] { "Alice", "Bob", "Carol" } },
    new { Name = "Marketing", Employees = new[] { "Dave", "Eve" } },
    new { Name = "Finance", Employees = new[] { "Frank" } }
};

// Flatten all employees
var allEmployees = departments.SelectMany(d => d.Employees);
// ["Alice", "Bob", "Carol", "Dave", "Eve", "Frank"]

// SelectMany with result selector
var deptEmployee = departments.SelectMany(
    d => d.Employees,
    (d, emp) => new { Department = d.Name, Employee = emp }
);

foreach (var de in deptEmployee)
    Console.WriteLine($"{de.Department}: {de.Employee}");

// ============================================
// Zip
// ============================================

var nums1 = new[] { 1, 2, 3, 4, 5 };
var nums2 = new[] { 10, 20, 30, 40, 50 };

var zipped = nums1.Zip(nums2, (a, b) => a + b); // [11, 22, 33, 44, 55]
var pairs = nums1.Zip(nums2); // (1,10), (2,20)...

// ============================================
// Aggregate (Reduce)
// ============================================

// Sum using Aggregate
int sum = numbers.Aggregate(0, (acc, n) => acc + n);

// String join
string joined = names.Aggregate((a, b) => $"{a}, {b}");
Console.WriteLine(joined); // สมชาย, สมหญิง, สมศรี, วิชัย, วิภา

// Running total
var runningTotals = numbers.Aggregate(
    new List<int>(),
    (list, n) =>
    {
        list.Add((list.LastOrDefault()) + n);
        return list;
    }
);
// [1, 3, 6, 10, 15, 21, 28, 36, 45, 55]

// ============================================
// Distinct, Except, Intersect, Union
// ============================================

var a = new[] { 1, 2, 3, 4, 5 };
var b = new[] { 3, 4, 5, 6, 7 };

var union = a.Union(b);       // 1,2,3,4,5,6,7
var intersect = a.Intersect(b); // 3,4,5
var except = a.Except(b);     // 1,2
var distinct = new[] { 1, 1, 2, 2, 3 }.Distinct(); // 1,2,3

// DistinctBy (C# 6+)
var distinctByMajor = students.DistinctBy(s => s.Major);

// ============================================
// Chunk (C# 6+)
// ============================================

var items = Enumerable.Range(1, 10);
var chunks = items.Chunk(3); // [[1,2,3], [4,5,6], [7,8,9], [10]]

foreach (var chunk in chunks)
    Console.WriteLine($"Chunk: {string.Join(",", chunk)}");
```

---

## Step 104: LINQ Performance

```csharp
// ============================================
// Deferred vs Immediate Execution
// ============================================

var data = new List<int> { 1, 2, 3, 4, 5 };

// Deferred: query ยังไม่ทำงาน
var query = data.Where(x => x > 2); // ยังไม่ scan

data.Add(6); // เพิ่มหลัง query นิยาม

// query จะรวม 6 ด้วย เพราะ execute ตอนนี้
foreach (var x in query)
    Console.Write($"{x} "); // 3 4 5 6

// Immediate: execute ทันที
var snapshot = data.Where(x => x > 2).ToList(); // execute now
data.Add(7); // ไม่กระทบ snapshot
Console.WriteLine(snapshot.Count); // 3, ไม่มี 7

// ============================================
// Materialization
// ============================================

// ToList() - เก็บเป็น List
var list = students.Where(s => s.GPA > 3.5).ToList();

// ToArray() - เก็บเป็น Array
var arr = students.Select(s => s.Name).ToArray();

// ToDictionary() - เก็บเป็น Dictionary
var dict = students.ToDictionary(s => s.Id, s => s.Name);

// ToHashSet() - เก็บเป็น HashSet
var set = students.Select(s => s.Major).ToHashSet();

// ToLookup() - หนึ่ง key หลาย values (immutable GroupBy)
var lookup = students.ToLookup(s => s.Major);
var csStudents2 = lookup["CS"]; // ทุก CS students

// ============================================
// AsParallel (PLINQ)
// ============================================

var largeData = Enumerable.Range(1, 1_000_000).ToArray();

// Sequential
var result1 = largeData
    .Where(x => x % 2 == 0)
    .Select(x => x * x)
    .Sum();

// Parallel - เร็วกว่าสำหรับข้อมูลใหญ่และ CPU-intensive
var result2 = largeData
    .AsParallel()
    .Where(x => x % 2 == 0)
    .Select(x => x * x)
    .Sum();

Console.WriteLine(result1 == result2); // true

// ============================================
// เลี่ยง multiple enumeration
// ============================================

// ❌ Bad: query ถูก enumerate สองครั้ง
var expensive = students.Where(s => { 
    Console.WriteLine($"Checking {s.Name}..."); // รันหลายครั้ง!
    return s.GPA > 3.5; 
});

var count = expensive.Count(); // enumerate ครั้งที่ 1
var first = expensive.First(); // enumerate ครั้งที่ 2

// ✅ Good: materialize เพียงครั้งเดียว
var materialized = students
    .Where(s => s.GPA > 3.5)
    .ToList(); // enumerate ครั้งเดียว

var count2 = materialized.Count;
var first2 = materialized.First();
```

---

## Step 105: Custom LINQ Extensions

```csharp
// ============================================
// Extension Methods สำหรับ LINQ
// ============================================

public static class LinqExtensions
{
    // WhereNot
    public static IEnumerable<T> WhereNot<T>(
        this IEnumerable<T> source, Func<T, bool> predicate)
        => source.Where(x => !predicate(x));
    
    // ForEach
    public static void ForEach<T>(
        this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
            action(item);
    }
    
    // Batch (before C# 6 Chunk)
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int size)
    {
        var batch = new List<T>(size);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == size)
            {
                yield return batch;
                batch = new List<T>(size);
            }
        }
        if (batch.Count > 0)
            yield return batch;
    }
    
    // DistinctBy with comparer (before C# 6)
    public static IEnumerable<T> DistinctBy2<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
        {
            if (seen.Add(keySelector(item)))
                yield return item;
        }
    }
    
    // MinBy, MaxBy 
    public static T MinBy2<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector) 
        where TKey : IComparable<TKey>
    {
        return source.Aggregate((min, x) => 
            keySelector(x).CompareTo(keySelector(min)) < 0 ? x : min);
    }
    
    // Shuffle
    public static IEnumerable<T> Shuffle<T>(this IEnumerable<T> source)
    {
        var list = source.ToList();
        var rng = new Random();
        for (int i = list.Count - 1; i > 0; i--)
        {
            int j = rng.Next(0, i + 1);
            (list[i], list[j]) = (list[j], list[i]);
        }
        return list;
    }
    
    // Paginate
    public static (IEnumerable<T> Items, int TotalCount, int TotalPages) Paginate<T>(
        this IEnumerable<T> source, int page, int pageSize)
    {
        var list = source.ToList();
        int total = list.Count;
        int totalPages = (int)Math.Ceiling((double)total / pageSize);
        var items = list.Skip((page - 1) * pageSize).Take(pageSize);
        return (items, total, totalPages);
    }
    
    // IsNullOrEmpty
    public static bool IsNullOrEmpty<T>(this IEnumerable<T>? source)
        => source == null || !source.Any();
}

// ใช้งาน
students.WhereNot(s => s.Major == "CS")
    .ForEach(s => Console.WriteLine($"{s.Name} is not in CS"));

var (pageStudents, total, totalPages) = students.Paginate(1, 3);
Console.WriteLine($"Page 1/{totalPages}: {total} total students");

var shuffled = students.Shuffle();
```

---

## Step 106: LINQ กับ Complex Data

```csharp
// ============================================
// Real-world LINQ Example: E-Commerce
// ============================================

public record Product5(int Id, string Name, string Category, decimal Price, int Stock);
public record OrderItem(int ProductId, int Quantity);
public record Order5(int Id, int CustomerId, DateTime Date, List<OrderItem> Items);
public record Customer2(int Id, string Name, string City, string Tier);

var products2 = new List<Product5>
{
    new(1, "iPhone 15", "Electronics", 35000, 100),
    new(2, "Samsung S24", "Electronics", 30000, 150),
    new(3, "iPad Pro", "Electronics", 42000, 50),
    new(4, "Nike Air Max", "Shoes", 4500, 200),
    new(5, "Adidas Ultra", "Shoes", 3800, 180),
    new(6, "Levi's Jeans", "Clothing", 1800, 300),
};

var customers2 = new List<Customer2>
{
    new(1, "สมชาย", "Bangkok", "Gold"),
    new(2, "สมหญิง", "Chiang Mai", "Silver"),
    new(3, "วิชัย", "Bangkok", "Gold"),
    new(4, "วิภา", "Phuket", "Bronze"),
};

var orders2 = new List<Order5>
{
    new(1, 1, DateTime.Today.AddDays(-5), new List<OrderItem> { new(1, 2), new(4, 1) }),
    new(2, 2, DateTime.Today.AddDays(-3), new List<OrderItem> { new(2, 1), new(6, 2) }),
    new(3, 1, DateTime.Today.AddDays(-1), new List<OrderItem> { new(3, 1) }),
    new(4, 3, DateTime.Today, new List<OrderItem> { new(1, 1), new(5, 3) }),
};

// ============================================
// Complex Queries
// ============================================

// 1. Revenue per category
var revenueByCategory = orders2
    .SelectMany(o => o.Items, (o, item) => new { o.Id, item.ProductId, item.Quantity })
    .Join(products2, x => x.ProductId, p => p.Id, 
          (x, p) => new { p.Category, Revenue = p.Price * x.Quantity })
    .GroupBy(x => x.Category)
    .Select(g => new { Category = g.Key, TotalRevenue = g.Sum(x => x.Revenue) })
    .OrderByDescending(x => x.TotalRevenue);

Console.WriteLine("Revenue by Category:");
foreach (var r in revenueByCategory)
    Console.WriteLine($"  {r.Category}: {r.TotalRevenue:C}");

// 2. Top customers by spending
var topCustomers = orders2
    .SelectMany(o => o.Items, (o, item) => new { o.CustomerId, item.ProductId, item.Quantity })
    .Join(products2, x => x.ProductId, p => p.Id,
          (x, p) => new { x.CustomerId, Spend = p.Price * x.Quantity })
    .GroupBy(x => x.CustomerId)
    .Select(g => new { CustomerId = g.Key, TotalSpend = g.Sum(x => x.Spend) })
    .Join(customers2, s => s.CustomerId, c => c.Id,
          (s, c) => new { c.Name, c.Tier, s.TotalSpend })
    .OrderByDescending(x => x.TotalSpend)
    .Take(3);

Console.WriteLine("\nTop 3 Customers:");
foreach (var c in topCustomers)
    Console.WriteLine($"  {c.Name} ({c.Tier}): {c.TotalSpend:C}");

// 3. Low stock products
var lowStock = products2
    .Where(p => p.Stock < 100)
    .Select(p => new { p.Name, p.Category, p.Stock })
    .OrderBy(p => p.Stock);

Console.WriteLine("\nLow Stock Products:");
foreach (var p in lowStock)
    Console.WriteLine($"  {p.Name}: {p.Stock} units");

// 4. Orders within date range
var recentOrders = orders2
    .Where(o => o.Date >= DateTime.Today.AddDays(-7))
    .OrderByDescending(o => o.Date)
    .Select(o => new {
        o.Id,
        Date = o.Date.ToString("yyyy-MM-dd"),
        Items = o.Items.Count,
        Total = o.Items.Sum(i => 
            products2.First(p => p.Id == i.ProductId).Price * i.Quantity)
    });

Console.WriteLine("\nRecent Orders (7 days):");
foreach (var o in recentOrders)
    Console.WriteLine($"  Order #{o.Id}: {o.Date}, {o.Items} items, {o.Total:C}");
```

---

## Step 107: Functional LINQ Patterns

```csharp
// ============================================
// Functional Programming ด้วย LINQ
// ============================================

// Pipeline pattern
public static class DataPipeline
{
    public static IEnumerable<T> Filter<T>(
        this IEnumerable<T> source, Func<T, bool> predicate)
        => source.Where(predicate);
    
    public static IEnumerable<TResult> Transform<T, TResult>(
        this IEnumerable<T> source, Func<T, TResult> selector)
        => source.Select(selector);
    
    public static T2 Reduce<T, T2>(
        this IEnumerable<T> source, T2 seed, Func<T2, T, T2> accumulator)
        => source.Aggregate(seed, accumulator);
}

// Immutable transformations
var pipeline = students
    .Filter(s => s.GPA >= 3.0)
    .Filter(s => s.Age >= 21)
    .Transform(s => new { s.Name, s.Major, Honor = s.GPA >= 3.8 ? "Honors" : "" })
    .Reduce(new Dictionary<string, int>(),
            (dict, s) =>
            {
                var major = s.Major;
                dict[major] = dict.GetValueOrDefault(major, 0) + 1;
                return dict;
            });

foreach (var (major, count) in pipeline)
    Console.WriteLine($"{major}: {count} eligible students");

// ============================================
// Memoization + LINQ
// ============================================

public static class MemoizedLinq
{
    public static Func<T, TResult> Memoize<T, TResult>(Func<T, TResult> fn) 
        where T : notnull
    {
        var cache = new Dictionary<T, TResult>();
        return x =>
        {
            if (!cache.TryGetValue(x, out var result))
                cache[x] = result = fn(x);
            return result;
        };
    }
}

// เร่งความเร็ว expensive selector
var expensiveCalc = MemoizedLinq.Memoize<double, double>(gpa => 
{
    Thread.Sleep(10); // simulate slow calculation
    return Math.Exp(gpa);
});

var scores = students
    .Select(s => new { s.Name, Score = expensiveCalc(s.GPA) });
```

---

## Step 108: IQueryable vs IEnumerable

```csharp
// ============================================
// IEnumerable vs IQueryable
// ============================================

/*
 * IEnumerable<T>:
 * - ทำงานใน memory (LINQ to Objects)
 * - Expression ถูก compile เป็น delegates
 * - ดึงข้อมูลทั้งหมดมาก่อน แล้วค่อย filter
 * 
 * IQueryable<T>:
 * - ทำงานบน data source (DB, etc.)
 * - Expression ถูก translate เป็น query (SQL)
 * - Filter ที่ data source ก่อน ดึงมาน้อยลง
 */

// Simulated IQueryable (Entity Framework style)
public class InMemoryQueryable<T> : IQueryable<T>
{
    private readonly IEnumerable<T> _source;
    private readonly Expression _expression;
    
    public InMemoryQueryable(IEnumerable<T> source)
    {
        _source = source;
        _expression = Expression.Constant(this);
    }
    
    public Type ElementType => typeof(T);
    public Expression Expression => _expression;
    public IQueryProvider Provider => new InMemoryQueryProvider(_source);
    
    public IEnumerator<T> GetEnumerator() => _source.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public class InMemoryQueryProvider : IQueryProvider
{
    private readonly IEnumerable _source;
    
    public InMemoryQueryProvider(IEnumerable source) => _source = source;
    
    public IQueryable CreateQuery(Expression expression)
        => throw new NotImplementedException();
    
    public IQueryable<TElement> CreateQuery<TElement>(Expression expression)
        => throw new NotImplementedException();
    
    public object Execute(Expression expression) 
        => throw new NotImplementedException();
    
    public TResult Execute<TResult>(Expression expression)
        => throw new NotImplementedException();
}

// In Real EF Core:
/*
// IQueryable: SQL WHERE, translated
var dbQuery = dbContext.Students
    .Where(s => s.GPA > 3.5) // SQL: WHERE GPA > 3.5
    .OrderBy(s => s.Name);   // SQL: ORDER BY Name

// ❌ Bad: ดึงทั้งหมดมาแล้ว filter ใน memory
var bad = dbContext.Students.AsEnumerable() // ดึงทุก record!
    .Where(s => s.GPA > 3.5);

// ✅ Good: filter ที่ DB
var good = dbContext.Students.Where(s => s.GPA > 3.5).ToList();
*/
```

---

## Step 109: LINQ สำหรับ XML

```csharp
// ============================================
// LINQ to XML
// ============================================

using System.Xml.Linq;

// สร้าง XML
var xml = new XDocument(
    new XDeclaration("1.0", "utf-8", "yes"),
    new XElement("Students",
        new XElement("Student",
            new XAttribute("id", "1"),
            new XElement("Name", "อลิส"),
            new XElement("GPA", "3.8"),
            new XElement("Major", "CS")),
        new XElement("Student",
            new XAttribute("id", "2"),
            new XElement("Name", "บ็อบ"),
            new XElement("GPA", "3.2"),
            new XElement("Major", "IT")),
        new XElement("Student",
            new XAttribute("id", "3"),
            new XElement("Name", "แครอล"),
            new XElement("GPA", "3.9"),
            new XElement("Major", "CS"))
    )
);

// Query XML
var highGpaStudents = 
    from s in xml.Root!.Elements("Student")
    let gpa = double.Parse(s.Element("GPA")!.Value)
    where gpa >= 3.8
    select new
    {
        Id = (int)s.Attribute("id")!,
        Name = (string)s.Element("Name")!,
        GPA = gpa,
        Major = (string)s.Element("Major")!
    };

foreach (var s in highGpaStudents)
    Console.WriteLine($"ID={s.Id}, {s.Name}: GPA={s.GPA} ({s.Major})");

// แก้ไข XML
var targetStudent = xml.Root.Elements("Student")
    .FirstOrDefault(s => (string)s.Element("Name")! == "บ็อบ");

if (targetStudent != null)
    targetStudent.Element("GPA")!.Value = "3.5";

// บันทึก
xml.Save("students.xml");
Console.WriteLine(xml.ToString());
```

---

## Step 110: LINQ Best Practices

```csharp
// ============================================
// LINQ Best Practices
// ============================================

// 1. ใช้ method syntax เป็นหลัก (อ่านง่ายกว่า)
// ❌ Query syntax สำหรับ simple queries
var bad2 = from s in students
           where s.GPA > 3.5
           select s.Name;

// ✅ Method syntax  
var good2 = students.Where(s => s.GPA > 3.5).Select(s => s.Name);

// 2. Materialize เมื่อใช้หลายครั้ง
var filtered = students.Where(s => s.GPA > 3.5).ToList(); // materialize once
var count3 = filtered.Count;       // no re-evaluation
var first3 = filtered.FirstOrDefault(); // no re-evaluation

// 3. ใช้ FirstOrDefault แทน First เมื่อ result อาจ empty
var maybe = students.FirstOrDefault(s => s.GPA > 4.0); // null-safe
// var risky = students.First(s => s.GPA > 4.0); // throws if empty!

// 4. ใช้ Any() แทน Count() > 0
// ❌ Count ต้อง enumerate ทั้งหมด
bool hasBad = students.Count(s => s.Major == "CS") > 0;

// ✅ Any หยุดทันทีที่เจอครั้งแรก
bool hasGood = students.Any(s => s.Major == "CS");

// 5. หลีกเลี่ยง nested LINQ ที่ซับซ้อน (ใช้ join แทน)
// ❌ Bad: O(n²)
var bad3 = students.SelectMany(s => 
    enrollments.Where(e => e.StudentId == s.Id)
               .Select(e => new { s.Name, e.CourseId }));

// ✅ Good: O(n) with lookup
var lookup2 = enrollments.ToLookup(e => e.StudentId);
var good3 = students.SelectMany(s => 
    lookup2[s.Id].Select(e => new { s.Name, e.CourseId }));

// 6. ใช้ AsEnumerable() เมื่อต้อง switch จาก IQueryable
// (ใน EF Core: ดึงข้อมูลมาแล้ว filter ที่ memory)
// var data = dbContext.Products
//     .Where(p => p.Price > 100) // SQL WHERE
//     .AsEnumerable()             // switch to memory
//     .Where(p => MyComplexCSharpFunction(p)); // memory filter
```

---

## แบบฝึกหัด Part 11

### แบบฝึกหัด: Sales Analysis System

```csharp
// ข้อมูล
public record Sale(DateTime Date, string Product, string Region, int Quantity, decimal UnitPrice)
{
    public decimal Total => Quantity * UnitPrice;
}

var sales = new List<Sale>
{
    new(new DateTime(2024, 1, 15), "iPhone", "North", 5, 35000),
    new(new DateTime(2024, 1, 20), "Samsung", "South", 3, 30000),
    new(new DateTime(2024, 2, 10), "iPhone", "South", 8, 35000),
    new(new DateTime(2024, 2, 25), "iPad", "North", 2, 42000),
    new(new DateTime(2024, 3, 5), "Samsung", "North", 6, 30000),
    new(new DateTime(2024, 3, 18), "iPhone", "East", 10, 35000),
    new(new DateTime(2024, 3, 22), "iPad", "East", 4, 42000),
};

// TODO: เขียน LINQ queries สำหรับ:
// 1. ยอดขายรวมต่อ product
// 2. ยอดขายสูงสุดต่อ region
// 3. เดือนที่ทำยอดได้ดีที่สุด
// 4. Top 2 products by quantity sold
// 5. เดือนที่ขาย iPhone ได้มากกว่า 7 ชิ้น
```

### เฉลย:

```csharp
// 1. ยอดขายรวมต่อ product
var productRevenue = sales
    .GroupBy(s => s.Product)
    .Select(g => new { Product = g.Key, Revenue = g.Sum(s => s.Total) })
    .OrderByDescending(x => x.Revenue);

Console.WriteLine("Revenue by Product:");
productRevenue.ForEach(x => Console.WriteLine($"  {x.Product}: {x.Revenue:C}"));

// 2. ยอดขายสูงสุดต่อ region
var topByRegion = sales
    .GroupBy(s => s.Region)
    .Select(g => new {
        Region = g.Key,
        TotalRevenue = g.Sum(s => s.Total),
        TopProduct = g.GroupBy(s => s.Product)
                       .OrderByDescending(pg => pg.Sum(s => s.Total))
                       .First().Key
    });

Console.WriteLine("\nTop Product by Region:");
topByRegion.ForEach(x => Console.WriteLine($"  {x.Region}: {x.TopProduct} ({x.TotalRevenue:C})"));

// 3. เดือนที่ทำยอดดีที่สุด
var bestMonth = sales
    .GroupBy(s => new { s.Date.Year, s.Date.Month })
    .Select(g => new {
        Month = $"{g.Key.Year}-{g.Key.Month:D2}",
        Revenue = g.Sum(s => s.Total)
    })
    .OrderByDescending(x => x.Revenue)
    .First();

Console.WriteLine($"\nBest Month: {bestMonth.Month} with {bestMonth.Revenue:C}");

// 4. Top 2 products by quantity
var top2 = sales
    .GroupBy(s => s.Product)
    .Select(g => new { Product = g.Key, TotalQty = g.Sum(s => s.Quantity) })
    .OrderByDescending(x => x.TotalQty)
    .Take(2);

Console.WriteLine("\nTop 2 Products by Quantity:");
top2.ForEach(x => Console.WriteLine($"  {x.Product}: {x.TotalQty} units"));

// 5. เดือนที่ขาย iPhone > 7
var iphoneMonths = sales
    .Where(s => s.Product == "iPhone")
    .GroupBy(s => new { s.Date.Year, s.Date.Month })
    .Where(g => g.Sum(s => s.Quantity) > 7)
    .Select(g => $"{g.Key.Year}-{g.Key.Month:D2}");

Console.WriteLine("\nMonths with iPhone > 7 units:");
iphoneMonths.ForEach(m => Console.WriteLine($"  {m}"));
```

---

## สรุป Part 11

ใน Part 11 เราได้เรียนรู้:

1. **LINQ Basics** - Where, Select, OrderBy, Take, Skip
2. **Aggregation** - Count, Sum, Average, Min, Max
3. **Grouping** - GroupBy, ToLookup
4. **Joining** - Join, GroupJoin
5. **Advanced** - SelectMany, Zip, Aggregate
6. **Set Operations** - Union, Intersect, Except, Distinct
7. **Performance** - Deferred execution, Materialization, PLINQ
8. **Custom Extensions** - ForEach, Batch, Paginate
9. **Complex Data** - E-Commerce queries
10. **Best Practices** - Any vs Count, avoid nested LINQ

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 11 | Steps 101-110*
