# Part 07: Object-Oriented Programming - Classes
## Steps 61-70: หลักการ OOP และการสร้าง Classes

---

## Step 61: Classes และ Objects พื้นฐาน

```csharp
// ============================================
// Class Declaration
// ============================================

// Class คือ blueprint สำหรับสร้าง objects
public class Person
{
    // Fields (ตัวแปรของ class)
    private string _name;
    private int _age;
    
    // Properties (expose fields อย่างปลอดภัย)
    public string Name
    {
        get => _name;
        set => _name = value ?? throw new ArgumentNullException(nameof(value));
    }
    
    public int Age
    {
        get => _age;
        set
        {
            if (value < 0 || value > 150)
                throw new ArgumentOutOfRangeException(nameof(value), "Age must be 0-150");
            _age = value;
        }
    }
    
    // Auto-implemented property
    public string Email { get; set; } = string.Empty;
    
    // Read-only property
    public bool IsAdult => _age >= 18;
    
    // Computed property
    public string DisplayName => $"{_name} (อายุ {_age} ปี)";
    
    // Constructor
    public Person(string name, int age)
    {
        _name = name;
        _age = age;
    }
    
    // Method
    public void Greet()
    {
        Console.WriteLine($"สวัสดี ฉันชื่อ {_name}");
    }
    
    // Override ToString
    public override string ToString()
        => $"Person {{ Name={_name}, Age={_age} }}";
}

// สร้าง Object จาก Class
var person1 = new Person("สมชาย", 30);
person1.Email = "somchai@email.com";

Console.WriteLine(person1.Name);         // สมชาย
Console.WriteLine(person1.IsAdult);      // True
Console.WriteLine(person1.DisplayName);  // สมชาย (อายุ 30 ปี)
person1.Greet();                         // สวัสดี ฉันชื่อ สมชาย
Console.WriteLine(person1);             // Person { Name=สมชาย, Age=30 }

// ============================================
// Object Initializer
// ============================================

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string Province { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    
    public override string ToString()
        => $"{Street}, {City}, {Province} {PostalCode}";
}

// Object initializer - กำหนดค่าหลัง constructor
var address = new Address
{
    Street = "123 ถนนสุขุมวิท",
    City = "กรุงเทพ",
    Province = "กรุงเทพมหานคร",
    PostalCode = "10110"
};

Console.WriteLine(address);
// 123 ถนนสุขุมวิท, กรุงเทพ, กรุงเทพมหานคร 10110
```

---

## Step 62: Constructors

```csharp
// ============================================
// Constructor Types
// ============================================

public class BankAccount
{
    public string AccountNumber { get; }
    public string OwnerName { get; }
    public decimal Balance { get; private set; }
    public DateTime CreatedAt { get; }
    
    // Primary Constructor (C# 12+)
    // public BankAccount(string accountNumber, string ownerName, decimal initialBalance = 0)
    
    // Default Constructor
    public BankAccount()
    {
        AccountNumber = GenerateAccountNumber();
        OwnerName = "Unknown";
        Balance = 0;
        CreatedAt = DateTime.Now;
    }
    
    // Parameterized Constructor
    public BankAccount(string ownerName) : this()
    {
        OwnerName = ownerName;
    }
    
    // Full Constructor
    public BankAccount(string accountNumber, string ownerName, decimal initialBalance = 0)
    {
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("Account number cannot be empty");
        if (string.IsNullOrWhiteSpace(ownerName))
            throw new ArgumentException("Owner name cannot be empty");
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        
        AccountNumber = accountNumber;
        OwnerName = ownerName;
        Balance = initialBalance;
        CreatedAt = DateTime.Now;
    }
    
    // Copy Constructor
    public BankAccount(BankAccount other)
    {
        AccountNumber = GenerateAccountNumber(); // สร้าง account ใหม่
        OwnerName = other.OwnerName;
        Balance = other.Balance;
        CreatedAt = DateTime.Now;
    }
    
    // Static Factory Methods (แทน Constructor)
    public static BankAccount CreateSavingsAccount(string ownerName)
        => new BankAccount(GenerateAccountNumber(), ownerName, 0);
    
    public static BankAccount CreateFixedDeposit(string ownerName, decimal amount)
    {
        if (amount < 50000)
            throw new ArgumentException("Fixed deposit minimum is 50,000 baht");
        return new BankAccount(GenerateAccountNumber(), ownerName, amount);
    }
    
    private static string GenerateAccountNumber()
        => $"{DateTime.Now:yyyyMMdd}{new Random().Next(1000, 9999)}";
    
    public bool Deposit(decimal amount)
    {
        if (amount <= 0) return false;
        Balance += amount;
        return true;
    }
    
    public bool Withdraw(decimal amount)
    {
        if (amount <= 0 || amount > Balance) return false;
        Balance -= amount;
        return true;
    }
    
    public override string ToString()
        => $"Account {AccountNumber}: {OwnerName} = {Balance:C}";
}

// ใช้งาน
var acc1 = new BankAccount("ACC001", "สมชาย", 10000);
var acc2 = BankAccount.CreateSavingsAccount("สมหญิง");
var acc3 = BankAccount.CreateFixedDeposit("สมศรี", 100000);

acc1.Deposit(5000);
acc1.Withdraw(3000);
Console.WriteLine(acc1); // Account ACC001: สมชาย = ฿12,000.00
```

---

## Step 63: Properties ขั้นสูง

```csharp
// ============================================
// Property Patterns
// ============================================

public class Temperature
{
    private double _celsius;
    
    // Backing field + validation
    public double Celsius
    {
        get => _celsius;
        set
        {
            if (value < -273.15)
                throw new ArgumentOutOfRangeException(
                    nameof(value), "Temperature cannot be below absolute zero");
            _celsius = value;
        }
    }
    
    // Computed properties
    public double Fahrenheit
    {
        get => _celsius * 9 / 5 + 32;
        set => Celsius = (value - 32) * 5 / 9;
    }
    
    public double Kelvin
    {
        get => _celsius + 273.15;
        set => Celsius = value - 273.15;
    }
    
    // Init-only property (C# 9+) - กำหนดได้แค่ตอน initialization
    public DateTime MeasuredAt { get; init; } = DateTime.Now;
    
    public Temperature(double celsius) => Celsius = celsius;
    
    public override string ToString()
        => $"{Celsius:F1}°C / {Fahrenheit:F1}°F / {Kelvin:F1}K";
}

var temp = new Temperature(100);
Console.WriteLine(temp); // 100.0°C / 212.0°F / 373.2K

temp.Fahrenheit = 32;
Console.WriteLine(temp); // 0.0°C / 32.0°F / 273.2K

// Init-only
var frozenTemp = new Temperature(0) { MeasuredAt = DateTime.Today };
// frozenTemp.MeasuredAt = DateTime.Now; // ERROR: init-only property

// ============================================
// Required Properties (C# 11+)
// ============================================

public class Employee
{
    // Required = ต้องกำหนดค่าตอน init
    public required string Name { get; init; }
    public required string Department { get; init; }
    public required string Email { get; init; }
    
    // Optional
    public string? PhoneNumber { get; set; }
    public DateTime StartDate { get; init; } = DateTime.Today;
    public decimal Salary { get; set; }
}

// ต้องกำหนดทุก required property
var emp = new Employee
{
    Name = "สมชาย ใจดี",
    Department = "Engineering",
    Email = "somchai@company.com"
};

// var emp2 = new Employee { Name = "Bob" }; // ERROR: Department, Email required

// ============================================
// Property Interceptor Pattern
// ============================================

public class ValidatedModel
{
    private readonly Dictionary<string, string> _errors = new();
    
    public IReadOnlyDictionary<string, string> Errors => _errors.AsReadOnly();
    public bool IsValid => !_errors.Any();
    
    private string _username = string.Empty;
    public string Username
    {
        get => _username;
        set
        {
            _username = value;
            ValidateUsername(value);
        }
    }
    
    private string _email = string.Empty;
    public string Email
    {
        get => _email;
        set
        {
            _email = value;
            ValidateEmail(value);
        }
    }
    
    private void ValidateUsername(string username)
    {
        _errors.Remove("username");
        if (string.IsNullOrWhiteSpace(username))
            _errors["username"] = "Username is required";
        else if (username.Length < 3)
            _errors["username"] = "Username must be at least 3 characters";
    }
    
    private void ValidateEmail(string email)
    {
        _errors.Remove("email");
        if (string.IsNullOrWhiteSpace(email))
            _errors["email"] = "Email is required";
        else if (!email.Contains('@'))
            _errors["email"] = "Invalid email format";
    }
}

var model = new ValidatedModel();
model.Username = "ab"; // Short username
model.Email = "invalid-email"; // Invalid email

Console.WriteLine($"Valid: {model.IsValid}"); // False
foreach (var (field, error) in model.Errors)
    Console.WriteLine($"{field}: {error}");
```

---

## Step 64: Access Modifiers

```csharp
// ============================================
// Access Modifiers
// ============================================

public class AccessExample
{
    // public: เข้าถึงได้จากทุกที่
    public string PublicField = "Everyone can access";
    
    // private: เข้าถึงได้เฉพาะใน class นี้ (default)
    private string _privateField = "Only this class";
    
    // protected: เข้าถึงได้จาก class นี้และ subclass
    protected string ProtectedField = "This class and subclasses";
    
    // internal: เข้าถึงได้ใน assembly เดียวกัน
    internal string InternalField = "Same assembly";
    
    // protected internal: protected OR internal
    protected internal string ProtectedInternal = "Same assembly OR subclass";
    
    // private protected: protected AND internal (C# 7.2+)
    private protected string PrivateProtected = "Same assembly AND subclass";
    
    // file (C# 11+): เข้าถึงได้แค่ในไฟล์เดียวกัน
    // file class FileScoped { } // ใช้แค่ใน class declaration
}

// ============================================
// Encapsulation Best Practices
// ============================================

public class BankAccountV2
{
    // Private fields - ซ่อน implementation
    private decimal _balance;
    private readonly List<string> _transactionHistory = new();
    
    // Public read-only - external can read but not write
    public decimal Balance => _balance;
    public IReadOnlyList<string> TransactionHistory => _transactionHistory.AsReadOnly();
    
    // Controlled mutation through methods
    public void Deposit(decimal amount)
    {
        ValidateAmount(amount, "Deposit");
        _balance += amount;
        _transactionHistory.Add($"{DateTime.Now:yyyy-MM-dd HH:mm}: +{amount:C}");
    }
    
    public void Withdraw(decimal amount)
    {
        ValidateAmount(amount, "Withdrawal");
        if (amount > _balance)
            throw new InvalidOperationException("Insufficient funds");
        _balance -= amount;
        _transactionHistory.Add($"{DateTime.Now:yyyy-MM-dd HH:mm}: -{amount:C}");
    }
    
    private static void ValidateAmount(decimal amount, string operation)
    {
        if (amount <= 0)
            throw new ArgumentException($"{operation} amount must be positive");
    }
}
```

---

## Step 65: Static Members

```csharp
// ============================================
// Static Fields, Properties, Methods
// ============================================

public class Counter
{
    // Static field - shared across all instances
    private static int _totalCount = 0;
    private static readonly object _lock = new();
    
    // Instance field
    private readonly int _id;
    
    // Static property
    public static int TotalCount => _totalCount;
    
    // Instance property
    public int Id => _id;
    
    public Counter()
    {
        lock (_lock) // Thread-safe increment
        {
            _totalCount++;
            _id = _totalCount;
        }
    }
    
    public static void Reset()
    {
        lock (_lock)
        {
            _totalCount = 0;
        }
    }
}

var c1 = new Counter();
var c2 = new Counter();
var c3 = new Counter();

Console.WriteLine(Counter.TotalCount); // 3
Console.WriteLine(c1.Id); // 1
Console.WriteLine(c2.Id); // 2

// ============================================
// Static Class
// ============================================

public static class MathUtils
{
    public static double Pi => Math.PI;
    
    public static double CircleArea(double radius) => Pi * radius * radius;
    public static double CirclePerimeter(double radius) => 2 * Pi * radius;
    
    public static (double Area, double Perimeter) CircleMetrics(double radius)
        => (CircleArea(radius), CirclePerimeter(radius));
    
    public static int Fibonacci(int n)
    {
        if (n <= 1) return n;
        int a = 0, b = 1;
        for (int i = 2; i <= n; i++)
            (a, b) = (b, a + b);
        return b;
    }
}

Console.WriteLine(MathUtils.CircleArea(5));
var (area, perimeter) = MathUtils.CircleMetrics(5);

// ============================================
// Static Constructor
// ============================================

public class Configuration
{
    public static readonly string AppName;
    public static readonly string Version;
    public static readonly Dictionary<string, string> Settings;
    
    // Static constructor - รันครั้งเดียวก่อน instance ใดๆ
    static Configuration()
    {
        AppName = "MyMauiApp";
        Version = "1.0.0";
        Settings = LoadSettings();
        Console.WriteLine($"Configuration initialized for {AppName} v{Version}");
    }
    
    private static Dictionary<string, string> LoadSettings()
    {
        // In real app: load from file/environment
        return new Dictionary<string, string>
        {
            { "ApiUrl", "https://api.example.com" },
            { "Timeout", "30" },
            { "MaxRetries", "3" }
        };
    }
}

// Static constructor รันเมื่อเข้าถึง class ครั้งแรก
Console.WriteLine(Configuration.AppName); // "Configuration initialized..." then "MyMauiApp"
Console.WriteLine(Configuration.Version); // "1.0.0"
```

---

## Step 66: Records

```csharp
// ============================================
// Record Types (C# 9+)
// ============================================

// Record: immutable value-based class
// - Value equality (เปรียบเทียบค่า ไม่ใช่ reference)
// - Non-destructive mutation ด้วย with expression
// - Auto-generated ToString, GetHashCode, Equals

// ============================================
// Record class
// ============================================

public record PersonRecord(string FirstName, string LastName, int Age)
{
    // Computed property
    public string FullName => $"{FirstName} {LastName}";
    
    // Validation
    public PersonRecord
    {
        if (string.IsNullOrWhiteSpace(FirstName))
            throw new ArgumentException("FirstName cannot be empty");
        if (Age < 0 || Age > 150)
            throw new ArgumentOutOfRangeException(nameof(Age));
    }
    
    // Additional methods
    public bool IsAdult => Age >= 18;
}

var p1 = new PersonRecord("สมชาย", "ใจดี", 30);
var p2 = new PersonRecord("สมชาย", "ใจดี", 30);
var p3 = new PersonRecord("สมหญิง", "ใจงาม", 25);

// Value equality
Console.WriteLine(p1 == p2);  // True (same values)
Console.WriteLine(p1 == p3);  // False

// Non-destructive mutation (with expression)
var olderP1 = p1 with { Age = 31 }; // สร้าง record ใหม่
Console.WriteLine(p1.Age);    // 30 (ไม่เปลี่ยน)
Console.WriteLine(olderP1.Age); // 31

// Auto-generated ToString
Console.WriteLine(p1); // PersonRecord { FirstName = สมชาย, LastName = ใจดี, Age = 30 }

// Deconstruction
var (firstName, lastName, age) = p1;
Console.WriteLine($"{firstName} {lastName}");

// ============================================
// Record struct (C# 10+)
// ============================================

// เหมือน record class แต่เป็น value type
public record struct Point3D(double X, double Y, double Z)
{
    public double DistanceTo(Point3D other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        double dz = Z - other.Z;
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}

var origin = new Point3D(0, 0, 0);
var point = new Point3D(1, 2, 3);
Console.WriteLine(origin.DistanceTo(point)); // 3.742...

// ============================================
// Inheritance ของ Records
// ============================================

public record Animal(string Name, string Species);
public record Dog(string Name, string Breed) : Animal(Name, "Canis familiaris")
{
    public string Bark() => "Woof!";
}

var dog = new Dog("บุญมา", "Thai Dog");
Console.WriteLine(dog.Name);    // บุญมา
Console.WriteLine(dog.Species); // Canis familiaris
Console.WriteLine(dog.Breed);   // Thai Dog
Console.WriteLine(dog.Bark());  // Woof!

// ============================================
// Record ใน MAUI (DTO/Model)
// ============================================

// Records เหมาะสำหรับ DTOs และ API responses
public record UserDto(int Id, string Username, string Email, DateTime CreatedAt);
public record ProductDto(int Id, string Name, decimal Price, int Stock);
public record OrderDto(int Id, UserDto User, List<ProductDto> Items, decimal Total);

// API response
var user = new UserDto(1, "john_doe", "john@example.com", DateTime.Now);
var product1 = new ProductDto(1, "iPhone 16", 39900m, 50);
var product2 = new ProductDto(2, "Case", 499m, 200);
var order = new OrderDto(1001, user, new() { product1, product2 }, 40399m);

Console.WriteLine($"Order {order.Id} by {order.User.Username}: {order.Total:C}");
```

---

## Step 67: Object Equality และ Comparison

```csharp
// ============================================
// Equality
// ============================================

// Reference types: default เปรียบเทียบ reference
class MyClass
{
    public int Value { get; }
    public MyClass(int value) => Value = value;
}

var a = new MyClass(5);
var b = new MyClass(5);
var c = a;

Console.WriteLine(a == b);       // False (reference equality)
Console.WriteLine(a == c);       // True (same reference)
Console.WriteLine(a.Equals(b));  // False

// ============================================
// Override Equals และ GetHashCode
// ============================================

public class Point : IEquatable<Point>
{
    public double X { get; }
    public double Y { get; }
    
    public Point(double x, double y) => (X, Y) = (x, y);
    
    // Override Equals
    public override bool Equals(object? obj)
    {
        if (obj is Point other)
            return Equals(other);
        return false;
    }
    
    // IEquatable<T> implementation (ดีกว่า boxing)
    public bool Equals(Point? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return X == other.X && Y == other.Y;
    }
    
    // Override GetHashCode - ต้อง override คู่กับ Equals เสมอ!
    public override int GetHashCode() => HashCode.Combine(X, Y);
    
    // Operator overloading
    public static bool operator ==(Point? left, Point? right)
    {
        if (left is null) return right is null;
        return left.Equals(right);
    }
    
    public static bool operator !=(Point? left, Point? right) => !(left == right);
    
    public override string ToString() => $"({X}, {Y})";
}

var p1 = new Point(3, 4);
var p2 = new Point(3, 4);
var p3 = new Point(5, 6);

Console.WriteLine(p1 == p2);  // True (value equality)
Console.WriteLine(p1 == p3);  // False
Console.WriteLine(p1.Equals(p2)); // True

// HashSet ใช้ GetHashCode
var pointSet = new HashSet<Point> { p1, p2, p3 };
Console.WriteLine(pointSet.Count); // 2 (p1 และ p2 เหมือนกัน)

// ============================================
// IComparable สำหรับ Sorting
// ============================================

public class Product3 : IComparable<Product3>
{
    public string Name { get; }
    public decimal Price { get; }
    public int Stock { get; }
    
    public Product3(string name, decimal price, int stock)
    {
        Name = name;
        Price = price;
        Stock = stock;
    }
    
    // เรียงตาม Price (ascending)
    public int CompareTo(Product3? other)
    {
        if (other is null) return 1;
        return Price.CompareTo(other.Price);
    }
    
    public override string ToString() => $"{Name}: {Price:C} (stock: {Stock})";
}

var products = new List<Product3>
{
    new("iPhone", 39900m, 10),
    new("Samsung", 35900m, 15),
    new("Pixel", 29900m, 8)
};

products.Sort(); // ใช้ IComparable
Console.WriteLine("Sorted by price:");
products.ForEach(Console.WriteLine);

// ============================================
// IComparer - Custom Sort
// ============================================

// Sort by name
products.Sort(Comparer<Product3>.Create((a, b) => 
    string.Compare(a.Name, b.Name, StringComparison.Ordinal)));

// Sort by stock descending
products.Sort((a, b) => b.Stock.CompareTo(a.Stock));
```

---

## Step 68: Object Cloning

```csharp
// ============================================
// Shallow Copy vs Deep Copy
// ============================================

public class Address2
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
}

public class Customer : ICloneable
{
    public string Name { get; set; } = string.Empty;
    public Address2 Address { get; set; } = new();
    public List<string> Orders { get; set; } = new();
    
    // Shallow Clone - ไม่ copy reference types
    public object Clone() => MemberwiseClone(); // built-in shallow copy
    
    // Deep Clone - copy ทุกอย่าง
    public Customer DeepClone()
    {
        return new Customer
        {
            Name = Name,
            Address = new Address2 { Street = Address.Street, City = Address.City },
            Orders = new List<string>(Orders) // copy list
        };
    }
}

var original = new Customer
{
    Name = "สมชาย",
    Address = new Address2 { Street = "123 สุขุมวิท", City = "กรุงเทพ" },
    Orders = new List<string> { "ORD001", "ORD002" }
};

// Shallow copy
var shallow = (Customer)original.Clone();
shallow.Name = "สมหญิง"; // ไม่กระทบ original
shallow.Address.City = "เชียงใหม่"; // กระทบ original! (same reference)

Console.WriteLine(original.Address.City); // เชียงใหม่ (ถูกแก้ไข!)

// Deep copy
original.Address.City = "กรุงเทพ"; // reset
var deep = original.DeepClone();
deep.Name = "สมศรี";
deep.Address.City = "ขอนแก่น"; // ไม่กระทบ original

Console.WriteLine(original.Address.City); // กรุงเทพ (ไม่เปลี่ยน)

// ============================================
// Deep Clone ด้วย System.Text.Json
// ============================================

using System.Text.Json;

T? DeepCloneJson<T>(T obj)
{
    string json = JsonSerializer.Serialize(obj);
    return JsonSerializer.Deserialize<T>(json);
}

// ใช้งาน
var customer2 = new Customer { Name = "Test" };
var cloned = DeepCloneJson(customer2);
```

---

## Step 69: Object Lifecycle

```csharp
// ============================================
// Constructors และ Finalizers
// ============================================

public class ResourceManager : IDisposable
{
    private bool _disposed = false;
    private readonly string _name;
    private IntPtr _nativeHandle; // simulate native resource
    
    public ResourceManager(string name)
    {
        _name = name;
        _nativeHandle = IntPtr.Zero; // acquire native resource
        Console.WriteLine($"ResourceManager '{_name}' created");
    }
    
    // Finalizer (Destructor) - เรียกโดย GC
    ~ResourceManager()
    {
        Console.WriteLine($"ResourceManager '{_name}' finalized");
        Dispose(false);
    }
    
    // IDisposable implementation
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // บอก GC ไม่ต้อง finalize
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            // Release managed resources
            Console.WriteLine($"Releasing managed resources for '{_name}'");
        }
        
        // Release unmanaged resources (ทำเสมอ)
        if (_nativeHandle != IntPtr.Zero)
        {
            // Free native resource
            _nativeHandle = IntPtr.Zero;
        }
        
        _disposed = true;
        Console.WriteLine($"ResourceManager '{_name}' disposed");
    }
    
    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        Console.WriteLine($"'{_name}' is working...");
    }
}

// using statement - dispose อัตโนมัติ
using (var rm = new ResourceManager("DatabaseConnection"))
{
    rm.DoWork();
} // Dispose เรียกที่นี่

// using declaration (C# 8+) - กระชับกว่า
using var rm2 = new ResourceManager("FileHandler");
rm2.DoWork();
// Dispose เรียกเมื่อออกจาก scope

// ============================================
// Object Pool Pattern
// ============================================

class ObjectPool<T> where T : class, new()
{
    private readonly Stack<T> _pool = new();
    private readonly int _maxSize;
    
    public ObjectPool(int maxSize = 10)
    {
        _maxSize = maxSize;
        // Pre-populate pool
        for (int i = 0; i < maxSize / 2; i++)
            _pool.Push(new T());
    }
    
    public T Rent()
    {
        return _pool.Count > 0 ? _pool.Pop() : new T();
    }
    
    public void Return(T obj)
    {
        if (_pool.Count < _maxSize)
            _pool.Push(obj);
    }
    
    public int AvailableCount => _pool.Count;
}
```

---

## Step 70: ตัวอย่างโปรเจกต์จริง - Contact Manager

```csharp
// ============================================
// ระบบ Contact Manager
// ============================================

public record PhoneNumber(string Number, string Type = "Mobile");
public record EmailAddress(string Address, string Type = "Personal");

public class Contact
{
    private static int _nextId = 1;
    
    public int Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public List<PhoneNumber> PhoneNumbers { get; } = new();
    public List<EmailAddress> Emails { get; } = new();
    public Address3? HomeAddress { get; set; }
    public string? Company { get; set; }
    public string? Notes { get; set; }
    public DateTime CreatedAt { get; }
    public DateTime UpdatedAt { get; private set; }
    
    public string FullName => $"{FirstName} {LastName}".Trim();
    public PhoneNumber? PrimaryPhone => PhoneNumbers.FirstOrDefault();
    public EmailAddress? PrimaryEmail => Emails.FirstOrDefault();
    
    public Contact(string firstName, string lastName)
    {
        Id = _nextId++;
        FirstName = firstName;
        LastName = lastName;
        CreatedAt = DateTime.Now;
        UpdatedAt = DateTime.Now;
    }
    
    public void AddPhone(string number, string type = "Mobile")
    {
        PhoneNumbers.Add(new PhoneNumber(number, type));
        Touch();
    }
    
    public void AddEmail(string email, string type = "Personal")
    {
        Emails.Add(new EmailAddress(email, type));
        Touch();
    }
    
    private void Touch() => UpdatedAt = DateTime.Now;
    
    public override string ToString()
        => $"[{Id}] {FullName}" + (Company != null ? $" @ {Company}" : "");
}

public record Address3(string Street, string City, string Province, string PostalCode);

public class ContactBook
{
    private readonly Dictionary<int, Contact> _contacts = new();
    private readonly Dictionary<string, HashSet<int>> _phoneIndex = new();
    private readonly Dictionary<string, HashSet<int>> _emailIndex = new();
    
    public int Count => _contacts.Count;
    
    public Contact Add(Contact contact)
    {
        _contacts[contact.Id] = contact;
        IndexContact(contact);
        return contact;
    }
    
    private void IndexContact(Contact contact)
    {
        foreach (var phone in contact.PhoneNumbers)
        {
            if (!_phoneIndex.ContainsKey(phone.Number))
                _phoneIndex[phone.Number] = new HashSet<int>();
            _phoneIndex[phone.Number].Add(contact.Id);
        }
        
        foreach (var email in contact.Emails)
        {
            string key = email.Address.ToLower();
            if (!_emailIndex.ContainsKey(key))
                _emailIndex[key] = new HashSet<int>();
            _emailIndex[key].Add(contact.Id);
        }
    }
    
    public Contact? GetById(int id)
        => _contacts.TryGetValue(id, out var c) ? c : null;
    
    public List<Contact> Search(string query)
    {
        if (string.IsNullOrWhiteSpace(query)) return _contacts.Values.ToList();
        
        query = query.ToLower();
        return _contacts.Values.Where(c =>
            c.FullName.ToLower().Contains(query) ||
            c.Company?.ToLower().Contains(query) == true ||
            c.PhoneNumbers.Any(p => p.Number.Contains(query)) ||
            c.Emails.Any(e => e.Address.ToLower().Contains(query))
        ).ToList();
    }
    
    public bool Delete(int id) => _contacts.Remove(id);
    
    public IEnumerable<IGrouping<string, Contact>> GetGroupedAlphabetically()
        => _contacts.Values
            .OrderBy(c => c.LastName)
            .ThenBy(c => c.FirstName)
            .GroupBy(c => c.LastName.Length > 0 
                ? c.LastName[0].ToString().ToUpper() 
                : "#");
    
    public void Print()
    {
        foreach (var group in GetGroupedAlphabetically())
        {
            Console.WriteLine($"\n--- {group.Key} ---");
            foreach (var contact in group)
            {
                Console.WriteLine($"  {contact}");
                if (contact.PrimaryPhone != null)
                    Console.WriteLine($"    📞 {contact.PrimaryPhone.Number}");
                if (contact.PrimaryEmail != null)
                    Console.WriteLine($"    ✉️ {contact.PrimaryEmail.Address}");
            }
        }
    }
}

// ทดสอบ
var book = new ContactBook();

var alice = new Contact("Alice", "Johnson");
alice.AddPhone("081-234-5678");
alice.AddEmail("alice@example.com");
alice.Company = "TechCorp";
book.Add(alice);

var bob = new Contact("Bob", "Smith");
bob.AddPhone("082-345-6789", "Work");
bob.AddPhone("083-456-7890", "Home");
bob.AddEmail("bob@example.com");
book.Add(bob);

var carol = new Contact("Carol", "Anderson");
carol.AddPhone("084-567-8901");
book.Add(carol);

Console.WriteLine($"Contacts: {book.Count}");
book.Print();

var results = book.Search("alice");
Console.WriteLine($"\nSearch 'alice': {results.Count} found");
```

---

## แบบฝึกหัด Part 07

### แบบฝึกหัดที่ 7.1: Library System
สร้างระบบห้องสมุดที่มี:
- Book class (ISBN, Title, Author, Copies)
- Library class (Add/Remove books, Borrow/Return)
- Member class

### แบบฝึกหัดที่ 7.2: Shopping Cart
สร้างระบบตะกร้าสินค้าที่มี:
- Product record (Id, Name, Price)
- CartItem (Product, Quantity)
- ShoppingCart (Add/Remove/UpdateQty/Total/Checkout)

### แบบฝึกหัดที่ 7.3: Temperature Converter Class
สร้าง Temperature class ที่:
- เก็บค่าใน Celsius
- Convert ระหว่าง C, F, K
- เปรียบเทียบได้ (IComparable)
- มี static factory methods (FromCelsius, FromFahrenheit, FromKelvin)

---

## สรุป Part 07

ในส่วนนี้เราได้เรียนรู้:

1. **Classes** - Declaration, Fields, Properties, Methods
2. **Constructors** - Default, Parameterized, Copy, Factory
3. **Properties** - Backing fields, Computed, Init-only, Required
4. **Access Modifiers** - public, private, protected, internal
5. **Static Members** - Fields, Methods, Classes, Constructors
6. **Records** - Value equality, Non-destructive mutation
7. **Object Equality** - Override Equals/GetHashCode, IComparable
8. **Object Cloning** - Shallow vs Deep copy
9. **Object Lifecycle** - IDisposable, Finalizer, Object Pool
10. **Real Project** - Contact Manager

## ขั้นต่อไป

ใน Part 08 เราจะเรียนรู้เกี่ยวกับ:
- Inheritance
- Method Override
- Abstract Classes
- Sealed Classes
- Base keyword
- Polymorphism

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 07 | Steps 61-70*
