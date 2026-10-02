# Part 03: Control Flow - if/else, switch, Pattern Matching
## Steps 21-30: การควบคุมการทำงานของโปรแกรม

---

## Step 21: if Statement พื้นฐาน

```csharp
// ============================================
// if Statement
// ============================================

// รูปแบบพื้นฐาน
int age = 20;

if (age >= 18)
{
    Console.WriteLine("คุณเป็นผู้ใหญ่");
}

// if-else
if (age >= 18)
{
    Console.WriteLine("ผู้ใหญ่");
}
else
{
    Console.WriteLine("เยาวชน");
}

// if-else if-else
int score = 75;

if (score >= 90)
{
    Console.WriteLine("เกรด A");
}
else if (score >= 80)
{
    Console.WriteLine("เกรด B");
}
else if (score >= 70)
{
    Console.WriteLine("เกรด C");
}
else if (score >= 60)
{
    Console.WriteLine("เกรด D");
}
else
{
    Console.WriteLine("เกรด F");
}

// ============================================
// if แบบไม่มีวงเล็บ (single statement)
// ============================================

// ใช้ได้ แต่ไม่แนะนำ เพราะอาจเข้าใจผิด
if (age >= 18)
    Console.WriteLine("ผู้ใหญ่");
else
    Console.WriteLine("เยาวชน");

// ============================================
// Nested if
// ============================================

bool isLoggedIn = true;
bool isAdmin = false;
bool hasPermission = true;

if (isLoggedIn)
{
    if (isAdmin)
    {
        Console.WriteLine("Admin: เข้าถึงได้ทุก feature");
    }
    else if (hasPermission)
    {
        Console.WriteLine("User: เข้าถึงได้บาง feature");
    }
    else
    {
        Console.WriteLine("User: เข้าถึงได้ feature พื้นฐาน");
    }
}
else
{
    Console.WriteLine("กรุณา Login ก่อน");
}

// ============================================
// Compound Conditions
// ============================================

int temperature = 28;
bool isSunny = true;
bool isWeekend = true;

if (temperature > 25 && isSunny && isWeekend)
{
    Console.WriteLine("วันนี้เหมาะสำหรับไปทะเล!");
}

if (temperature < 10 || temperature > 40)
{
    Console.WriteLine("อากาศอันตราย!");
}

// Complex conditions - ใช้ parentheses ให้ชัดเจน
if ((temperature > 20 && temperature < 35) && (isSunny || isWeekend))
{
    Console.WriteLine("สภาพอากาศดี");
}
```

---

## Step 22: switch Statement

```csharp
// ============================================
// switch Statement แบบดั้งเดิม
// ============================================

int dayNumber = 3;

switch (dayNumber)
{
    case 1:
        Console.WriteLine("วันจันทร์");
        break;
    case 2:
        Console.WriteLine("วันอังคาร");
        break;
    case 3:
        Console.WriteLine("วันพุธ");
        break;
    case 4:
        Console.WriteLine("วันพฤหัสบดี");
        break;
    case 5:
        Console.WriteLine("วันศุกร์");
        break;
    case 6:
    case 7:
        Console.WriteLine("วันหยุดสุดสัปดาห์"); // Fall-through
        break;
    default:
        Console.WriteLine("ไม่ถูกต้อง");
        break;
}

// ============================================
// switch ด้วย String
// ============================================

string command = "start";

switch (command.ToLower())
{
    case "start":
    case "begin":
        Console.WriteLine("เริ่มต้น");
        break;
    case "stop":
    case "end":
        Console.WriteLine("หยุด");
        break;
    case "pause":
        Console.WriteLine("หยุดชั่วคราว");
        break;
    default:
        Console.WriteLine($"คำสั่งไม่รู้จัก: {command}");
        break;
}

// ============================================
// switch ด้วย Enum
// ============================================

enum Season { Spring, Summer, Autumn, Winter }

Season currentSeason = Season.Summer;

switch (currentSeason)
{
    case Season.Spring:
        Console.WriteLine("ฤดูใบไม้ผลิ - อากาศอบอุ่น");
        break;
    case Season.Summer:
        Console.WriteLine("ฤดูร้อน - อากาศร้อน");
        break;
    case Season.Autumn:
        Console.WriteLine("ฤดูใบไม้ร่วง - ใบไม้เปลี่ยนสี");
        break;
    case Season.Winter:
        Console.WriteLine("ฤดูหนาว - อากาศหนาว");
        break;
}

// ============================================
// switch Expression (C# 8+) - แนะนำ
// ============================================

// แบบเก่า
string dayName;
switch (dayNumber)
{
    case 1: dayName = "จันทร์"; break;
    case 2: dayName = "อังคาร"; break;
    case 3: dayName = "พุธ"; break;
    default: dayName = "ไม่ทราบ"; break;
}

// แบบใหม่ - กระชับกว่า
string dayName2 = dayNumber switch
{
    1 => "จันทร์",
    2 => "อังคาร",
    3 => "พุธ",
    4 => "พฤหัสบดี",
    5 => "ศุกร์",
    6 or 7 => "วันหยุด",
    _ => "ไม่ทราบ"
};

Console.WriteLine(dayName2); // พุธ

// switch expression ด้วย tuple
(int x, int y) point = (1, 1);
string quadrant = point switch
{
    (> 0, > 0) => "Quadrant I",
    (< 0, > 0) => "Quadrant II",
    (< 0, < 0) => "Quadrant III",
    (> 0, < 0) => "Quadrant IV",
    (0, 0)     => "Origin",
    (0, _)     => "Y-axis",
    (_, 0)     => "X-axis",
    _          => "Unknown"
};
Console.WriteLine(quadrant); // Quadrant I
```

---

## Step 23: Pattern Matching

```csharp
// ============================================
// is Pattern Matching (C# 7+)
// ============================================

object value = 42;

// แบบเก่า
if (value is int)
{
    int intValue = (int)value;
    Console.WriteLine($"int: {intValue}");
}

// แบบใหม่ - กระชับกว่า
if (value is int intVal)
{
    Console.WriteLine($"int: {intVal}");
}

// ============================================
// Type Pattern
// ============================================

object[] objects = { 42, "hello", 3.14, true, null };

foreach (object obj in objects)
{
    string description = obj switch
    {
        int i    => $"Integer: {i}",
        string s => $"String: '{s}'",
        double d => $"Double: {d}",
        bool b   => $"Boolean: {b}",
        null     => "null",
        _        => $"Other: {obj.GetType().Name}"
    };
    Console.WriteLine(description);
}

// ============================================
// Property Pattern (C# 8+)
// ============================================

record Point(int X, int Y);
record Rectangle(Point TopLeft, Point BottomRight);

Point p = new(5, -3);

string location = p switch
{
    { X: 0, Y: 0 } => "Origin",
    { X: > 0, Y: > 0 } => "Quadrant I",
    { X: < 0, Y: > 0 } => "Quadrant II",
    { X: < 0, Y: < 0 } => "Quadrant III",
    { X: > 0, Y: < 0 } => "Quadrant IV",
    { X: 0 } => "Y-axis",
    { Y: 0 } => "X-axis",
    _ => "Unknown"
};
Console.WriteLine($"Point({p.X}, {p.Y}) is in {location}"); // Quadrant IV

// Nested property pattern
var rect = new Rectangle(new Point(0, 5), new Point(5, 0));
bool isInOrigin = rect is { TopLeft: { X: 0, Y: 0 } };
Console.WriteLine($"Starts at origin: {isInOrigin}"); // False

// ============================================
// Positional Pattern (C# 8+)
// ============================================

// ใช้กับ record ที่มี Deconstruct
Point point2 = new(3, 4);

if (point2 is (> 0, > 0))
{
    Console.WriteLine($"({point2.X}, {point2.Y}) is in Quadrant I");
}

// ============================================
// Relational Pattern (C# 9+)
// ============================================

int temperature = 28;

string weather = temperature switch
{
    < 0       => "หนาวจัด",
    >= 0 and < 10  => "หนาวมาก",
    >= 10 and < 20 => "หนาว",
    >= 20 and < 30 => "อบอุ่น",
    >= 30 and < 40 => "ร้อน",
    >= 40     => "ร้อนจัด",
    _         => "ไม่ทราบ"
};
Console.WriteLine($"{temperature}°C: {weather}"); // อบอุ่น

// ============================================
// Logical Pattern (C# 9+)
// ============================================

int num = 15;

bool isDivisibleBy3And5 = num is (> 0) and (int n when n % 3 == 0 and n % 5 == 0);

// ใช้ and, or, not
string category = num switch
{
    0 => "ศูนย์",
    > 0 and < 10 => "1-9",
    >= 10 and <= 20 => "10-20",
    not (> 0) => "ลบ",
    _ => "มากกว่า 20"
};
Console.WriteLine(category); // 10-20

// ============================================
// Guard Clause ด้วย when
// ============================================

int score2 = 85;
string grade = score2 switch
{
    int s when s >= 90 => "A",
    int s when s >= 80 => "B",
    int s when s >= 70 => "C",
    int s when s >= 60 => "D",
    _ => "F"
};
Console.WriteLine($"Score {score2}: Grade {grade}"); // B
```

---

## Step 24: Guard Clauses และ Early Return

```csharp
// ============================================
// Guard Clauses - ออกจาก method เร็วๆ ถ้า invalid
// ============================================

// แบบ Bad - Nested conditions
string ProcessOrderBad(string customerId, int quantity, decimal price)
{
    if (!string.IsNullOrEmpty(customerId))
    {
        if (quantity > 0)
        {
            if (price > 0)
            {
                decimal total = quantity * price;
                return $"Order processed: Customer {customerId}, Total: {total:C}";
            }
            else
            {
                return "Error: Price must be positive";
            }
        }
        else
        {
            return "Error: Quantity must be positive";
        }
    }
    else
    {
        return "Error: Customer ID required";
    }
}

// แบบ Good - Guard Clauses
string ProcessOrderGood(string customerId, int quantity, decimal price)
{
    // Guard clauses - ตรวจสอบเงื่อนไข invalid ก่อน
    if (string.IsNullOrEmpty(customerId))
        return "Error: Customer ID required";
    
    if (quantity <= 0)
        return "Error: Quantity must be positive";
    
    if (price <= 0)
        return "Error: Price must be positive";
    
    // Main logic - ตรงไปตรงมา
    decimal total = quantity * price;
    return $"Order processed: Customer {customerId}, Total: {total:C}";
}

// ============================================
// Early Return Pattern
// ============================================

User? FindUserByEmail(List<User> users, string email)
{
    // Guard: ตรวจสอบ input ก่อน
    if (users == null || users.Count == 0)
        return null;
    
    if (string.IsNullOrWhiteSpace(email))
        return null;
    
    // Main logic
    return users.FirstOrDefault(u => 
        u.Email.Equals(email, StringComparison.OrdinalIgnoreCase));
}

record User(string Name, string Email);

// ============================================
// Validation Pattern
// ============================================

class OrderValidator
{
    public (bool IsValid, List<string> Errors) Validate(Order order)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrEmpty(order.CustomerName))
            errors.Add("Customer name is required");
        
        if (order.Items == null || order.Items.Count == 0)
            errors.Add("Order must have at least one item");
        
        if (order.TotalAmount <= 0)
            errors.Add("Total amount must be positive");
        
        if (order.DeliveryDate < DateTime.Today)
            errors.Add("Delivery date cannot be in the past");
        
        return (!errors.Any(), errors);
    }
}

record Order(
    string CustomerName, 
    List<string>? Items, 
    decimal TotalAmount, 
    DateTime DeliveryDate
);
```

---

## Step 25: Complex Conditional Logic

```csharp
// ============================================
// ตัวอย่างจริง: ระบบ Discount
// ============================================

class DiscountCalculator
{
    // ประเภทลูกค้า
    enum CustomerType { Regular, Silver, Gold, Platinum }
    
    // ระดับ discount
    record DiscountRule(CustomerType Type, decimal MinPurchase, decimal DiscountPercent, string Label);
    
    static readonly List<DiscountRule> DiscountRules = new()
    {
        new(CustomerType.Platinum, 0, 20, "Platinum 20%"),
        new(CustomerType.Gold, 5000, 15, "Gold 15% (purchase ≥ 5000)"),
        new(CustomerType.Gold, 0, 10, "Gold 10%"),
        new(CustomerType.Silver, 3000, 8, "Silver 8% (purchase ≥ 3000)"),
        new(CustomerType.Silver, 0, 5, "Silver 5%"),
        new(CustomerType.Regular, 10000, 5, "Regular 5% (purchase ≥ 10000)"),
    };
    
    public static (decimal Discount, string Label) Calculate(CustomerType customerType, decimal purchaseAmount)
    {
        // หา rule ที่ match
        var rule = DiscountRules
            .FirstOrDefault(r => r.Type == customerType && purchaseAmount >= r.MinPurchase);
        
        if (rule == null)
            return (0, "ไม่มี discount");
        
        return (purchaseAmount * rule.DiscountPercent / 100, rule.Label);
    }
}

// การใช้งาน
var (discount, label) = DiscountCalculator.Calculate(
    DiscountCalculator.CustomerType.Gold, 
    6000
);
Console.WriteLine($"Discount: {discount:C} ({label})");

// ============================================
// State Machine Pattern
// ============================================

enum OrderStatus
{
    Pending,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

class OrderStateMachine
{
    public static bool CanTransition(OrderStatus from, OrderStatus to)
    {
        return (from, to) switch
        {
            (OrderStatus.Pending, OrderStatus.Confirmed) => true,
            (OrderStatus.Pending, OrderStatus.Cancelled) => true,
            (OrderStatus.Confirmed, OrderStatus.Processing) => true,
            (OrderStatus.Confirmed, OrderStatus.Cancelled) => true,
            (OrderStatus.Processing, OrderStatus.Shipped) => true,
            (OrderStatus.Shipped, OrderStatus.Delivered) => true,
            (OrderStatus.Delivered, OrderStatus.Refunded) => true,
            _ => false
        };
    }
    
    public static string GetNextActions(OrderStatus status)
    {
        return status switch
        {
            OrderStatus.Pending    => "สามารถ: Confirm, Cancel",
            OrderStatus.Confirmed  => "สามารถ: Process, Cancel",
            OrderStatus.Processing => "สามารถ: Ship",
            OrderStatus.Shipped    => "สามารถ: Deliver",
            OrderStatus.Delivered  => "สามารถ: Refund",
            OrderStatus.Cancelled  => "ไม่มีการกระทำเพิ่มเติม",
            OrderStatus.Refunded   => "ไม่มีการกระทำเพิ่มเติม",
            _ => "Unknown status"
        };
    }
}

// ทดสอบ State Machine
var currentStatus = OrderStatus.Pending;
Console.WriteLine($"Current: {currentStatus}");
Console.WriteLine(OrderStateMachine.GetNextActions(currentStatus));

// ลอง transition
var targetStatus = OrderStatus.Confirmed;
bool canTransition = OrderStateMachine.CanTransition(currentStatus, targetStatus);
Console.WriteLine($"Can go to {targetStatus}: {canTransition}"); // True

targetStatus = OrderStatus.Shipped;
canTransition = OrderStateMachine.CanTransition(currentStatus, targetStatus);
Console.WriteLine($"Can go to {targetStatus}: {canTransition}"); // False
```

---

## Step 26: Exception ด้วย if/switch

```csharp
// ============================================
// การจัดการ Errors ด้วย if
// ============================================

// Pattern: Result type
record Result<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }
    
    public static Result<T> Success(T value) => new() { IsSuccess = true, Value = value };
    public static Result<T> Failure(string error) => new() { IsSuccess = false, Error = error };
}

Result<int> Divide(int a, int b)
{
    if (b == 0)
        return Result<int>.Failure("Cannot divide by zero");
    
    return Result<int>.Success(a / b);
}

// การใช้งาน
var result = Divide(10, 2);
if (result.IsSuccess)
    Console.WriteLine($"Result: {result.Value}");  // Result: 5
else
    Console.WriteLine($"Error: {result.Error}");

var result2 = Divide(10, 0);
if (result2.IsSuccess)
    Console.WriteLine($"Result: {result2.Value}");
else
    Console.WriteLine($"Error: {result2.Error}");  // Error: Cannot divide by zero

// ============================================
// Input Validation
// ============================================

class RegistrationValidator
{
    public record ValidationResult(bool IsValid, Dictionary<string, string> Errors);
    
    public static ValidationResult ValidateRegistration(
        string username, string email, string password, int age)
    {
        var errors = new Dictionary<string, string>();
        
        // Username validation
        if (string.IsNullOrWhiteSpace(username))
            errors["username"] = "Username is required";
        else if (username.Length < 3)
            errors["username"] = "Username must be at least 3 characters";
        else if (username.Length > 20)
            errors["username"] = "Username must be at most 20 characters";
        else if (!System.Text.RegularExpressions.Regex.IsMatch(username, @"^[a-zA-Z0-9_]+$"))
            errors["username"] = "Username can only contain letters, numbers, and underscores";
        
        // Email validation
        if (string.IsNullOrWhiteSpace(email))
            errors["email"] = "Email is required";
        else if (!email.Contains('@') || !email.Contains('.'))
            errors["email"] = "Invalid email format";
        
        // Password validation
        if (string.IsNullOrWhiteSpace(password))
            errors["password"] = "Password is required";
        else if (password.Length < 8)
            errors["password"] = "Password must be at least 8 characters";
        else
        {
            bool hasUpper = password.Any(char.IsUpper);
            bool hasLower = password.Any(char.IsLower);
            bool hasDigit = password.Any(char.IsDigit);
            
            if (!hasUpper)
                errors["password"] = "Password must contain at least one uppercase letter";
            else if (!hasLower)
                errors["password"] = "Password must contain at least one lowercase letter";
            else if (!hasDigit)
                errors["password"] = "Password must contain at least one digit";
        }
        
        // Age validation
        if (age < 0 || age > 150)
            errors["age"] = "Age must be between 0 and 150";
        else if (age < 13)
            errors["age"] = "Must be at least 13 years old";
        
        return new ValidationResult(!errors.Any(), errors);
    }
}

// ทดสอบ Validation
var validation = RegistrationValidator.ValidateRegistration(
    username: "john_doe",
    email: "john@example.com",
    password: "SecurePass1",
    age: 25
);

if (validation.IsValid)
    Console.WriteLine("Registration valid!");
else
{
    Console.WriteLine("Validation errors:");
    foreach (var error in validation.Errors)
        Console.WriteLine($"  {error.Key}: {error.Value}");
}
```

---

## Step 27: Conditional Expressions ขั้นสูง

```csharp
// ============================================
// Complex Business Rules
// ============================================

record LoanApplication(
    decimal Income,
    decimal LoanAmount,
    int CreditScore,
    int YearsEmployed,
    bool HasCollateral
);

enum LoanDecision { Approved, ConditionalApproval, Rejected }

class LoanEvaluator
{
    public static (LoanDecision Decision, string Reason, decimal? ApprovedAmount) Evaluate(LoanApplication app)
    {
        // คำนวณ debt-to-income ratio
        decimal monthlyIncome = app.Income / 12;
        decimal monthlyPayment = app.LoanAmount / 60; // 5-year loan
        decimal debtToIncomeRatio = monthlyPayment / monthlyIncome;
        
        // ตรวจสอบเงื่อนไขพื้นฐาน
        if (app.Income < 15000)
            return (LoanDecision.Rejected, "รายได้ต่ำกว่าเกณฑ์ขั้นต่ำ (15,000 บาท/เดือน)", null);
        
        if (app.CreditScore < 500)
            return (LoanDecision.Rejected, $"Credit score ต่ำเกินไป ({app.CreditScore})", null);
        
        if (debtToIncomeRatio > 0.5m)
            return (LoanDecision.Rejected, $"Debt-to-income ratio สูงเกินไป ({debtToIncomeRatio:P1})", null);
        
        // ประเมิน Credit Score
        if (app.CreditScore >= 750)
        {
            // Credit ดีมาก
            if (debtToIncomeRatio <= 0.35m)
                return (LoanDecision.Approved, "Credit score ดีเยี่ยม และ DTI ต่ำ", app.LoanAmount);
            else
                return (LoanDecision.Approved, "Credit score ดีเยี่ยม", app.LoanAmount * 0.9m);
        }
        else if (app.CreditScore >= 650)
        {
            // Credit ดี
            if (app.YearsEmployed >= 2)
                return (LoanDecision.Approved, "Credit score ดีและมีความมั่นคงในอาชีพ", app.LoanAmount * 0.8m);
            else
                return (LoanDecision.ConditionalApproval, "ต้องมีหลักประกันเพิ่มเติม", app.LoanAmount * 0.7m);
        }
        else if (app.CreditScore >= 550)
        {
            // Credit พอใช้
            if (app.HasCollateral && app.YearsEmployed >= 3)
                return (LoanDecision.ConditionalApproval, "อนุมัติตามเงื่อนไข - ต้องมีหลักประกัน", app.LoanAmount * 0.6m);
            else
                return (LoanDecision.Rejected, "ต้องมีหลักประกันและประสบการณ์ทำงานอย่างน้อย 3 ปี", null);
        }
        
        return (LoanDecision.Rejected, "ไม่ผ่านเกณฑ์การพิจารณา", null);
    }
}

// ทดสอบ
var application = new LoanApplication(
    Income: 50000,
    LoanAmount: 500000,
    CreditScore: 700,
    YearsEmployed: 3,
    HasCollateral: true
);

var (decision, reason, amount) = LoanEvaluator.Evaluate(application);
Console.WriteLine($"\nผลการพิจารณาสินเชื่อ:");
Console.WriteLine($"การตัดสิน: {decision}");
Console.WriteLine($"เหตุผล: {reason}");
if (amount.HasValue)
    Console.WriteLine($"วงเงินที่อนุมัติ: {amount:C}");
```

---

## Step 28: Pattern Matching ขั้นสูง

```csharp
// ============================================
// List Pattern (C# 11+)
// ============================================

int[] numbers = { 1, 2, 3, 4, 5 };

// ตรวจสอบรูปแบบ list
string description2 = numbers switch
{
    [] => "empty",
    [var single] => $"single: {single}",
    [var first, var second] => $"two: {first}, {second}",
    [var first2, .., var last] => $"starts: {first2}, ends: {last}",
};
Console.WriteLine(description2); // starts: 1, ends: 5

// ตรวจสอบ list ที่มีรูปแบบเฉพาะ
int[] sequence = { 1, 2, 3 };
bool isIncreasing = sequence is [var a, var b, var c] && a < b && b < c;
Console.WriteLine($"Is increasing: {isIncreasing}"); // True

// ============================================
// Deconstruction
// ============================================

// Tuple deconstruction
var (x, y) = (10, 20);
Console.WriteLine($"x={x}, y={y}");

// Record deconstruction
record PersonInfo(string Name, int Age, string City);
var person = new PersonInfo("สมชาย", 30, "กรุงเทพ");
var (name, age2, city) = person;
Console.WriteLine($"{name}, {age2} ปี, {city}");

// Custom deconstruction
class Temperature2
{
    public double Celsius { get; }
    public Temperature2(double celsius) => Celsius = celsius;
    public double Fahrenheit => Celsius * 9 / 5 + 32;
    public double Kelvin => Celsius + 273.15;
    
    public void Deconstruct(out double celsius, out double fahrenheit, out double kelvin)
    {
        celsius = Celsius;
        fahrenheit = Fahrenheit;
        kelvin = Kelvin;
    }
}

var temp = new Temperature2(100);
var (c, f, k) = temp;
Console.WriteLine($"Celsius: {c}, Fahrenheit: {f}, Kelvin: {k}");

// Deconstruction ใน switch
object shape = new Circle2(5.0);

string area = shape switch
{
    Circle2(var r) => $"Circle area: {Math.PI * r * r:F2}",
    Rectangle2(var w, var h) => $"Rectangle area: {w * h}",
    _ => "Unknown shape"
};
Console.WriteLine(area);

record Circle2(double Radius)
{
    public void Deconstruct(out double radius) => radius = Radius;
}

record Rectangle2(double Width, double Height)
{
    public void Deconstruct(out double width, out double height)
    {
        width = Width;
        height = Height;
    }
}
```

---

## Step 29: Conditional Compilation

```csharp
// ============================================
// Preprocessor Directives
// ============================================

#define DEBUG_MODE

class ConditionalExample
{
    public void Log(string message)
    {
        #if DEBUG_MODE
        Console.WriteLine($"[DEBUG] {message}");
        #elif VERBOSE
        Console.WriteLine($"[VERBOSE] {message}");
        #else
        // Production: ไม่แสดง debug messages
        #endif
    }
    
    #if DEBUG_MODE
    public void DebugInfo()
    {
        Console.WriteLine("Debug information...");
    }
    #endif
}

// ============================================
// Conditional Attribute
// ============================================

using System.Diagnostics;

class Logger
{
    [Conditional("DEBUG")]
    public static void Debug(string message)
    {
        Console.WriteLine($"[DEBUG] {message}");
        // method นี้จะถูก compile เฉพาะเมื่อ DEBUG symbol ถูก define
    }
    
    [Conditional("TRACE")]
    public static void Trace(string message)
    {
        Console.WriteLine($"[TRACE] {message}");
    }
}

// ============================================
// Environment-based Conditions
// ============================================

class AppConfig
{
    public static string GetApiUrl()
    {
        string environment = Environment.GetEnvironmentVariable("ENVIRONMENT") ?? "Development";
        
        return environment switch
        {
            "Production"  => "https://api.production.com",
            "Staging"     => "https://api.staging.com",
            "Testing"     => "https://api.test.com",
            "Development" => "https://localhost:5001",
            _ => "https://localhost:5001"
        };
    }
    
    public static bool IsDebugMode()
    {
        string env = Environment.GetEnvironmentVariable("ENVIRONMENT") ?? "Development";
        return env is "Development" or "Testing";
    }
}
```

---

## Step 30: Best Practices สำหรับ Control Flow

```csharp
// ============================================
// Anti-patterns และ วิธีแก้ไข
// ============================================

// ❌ Anti-pattern 1: Magic numbers
if (score >= 90)
    Console.WriteLine("Excellent");

// ✅ Best practice: Named constants
const int EXCELLENT_THRESHOLD = 90;
const int GOOD_THRESHOLD = 75;
const int PASS_THRESHOLD = 60;

if (score >= EXCELLENT_THRESHOLD)
    Console.WriteLine("Excellent");
else if (score >= GOOD_THRESHOLD)
    Console.WriteLine("Good");
else if (score >= PASS_THRESHOLD)
    Console.WriteLine("Pass");

// ❌ Anti-pattern 2: Complex nested conditions
void ProcessOrder2(Order2 order)
{
    if (order != null)
    {
        if (order.Items != null)
        {
            if (order.Items.Count > 0)
            {
                if (order.TotalAmount > 0)
                {
                    // main logic
                }
            }
        }
    }
}

// ✅ Best practice: Guard clauses
void ProcessOrderBetter(Order2? order)
{
    if (order is null) return;
    if (order.Items is null or { Count: 0 }) return;
    if (order.TotalAmount <= 0) return;
    
    // main logic - clear and readable
}

// ❌ Anti-pattern 3: Boolean flag abuse
bool shouldSendEmail = false;
bool shouldLogError = false;
bool shouldNotifyAdmin = false;

if (error != null)
{
    shouldLogError = true;
    if (error.IsSerious)
    {
        shouldNotifyAdmin = true;
        shouldSendEmail = true;
    }
}

if (shouldLogError) LogError(error);
if (shouldNotifyAdmin) NotifyAdmin(error);
if (shouldSendEmail) SendEmail(error);

// ✅ Best practice: Direct logic
if (error != null)
{
    LogError(error);
    if (error.IsSerious)
    {
        NotifyAdmin(error);
        SendEmail(error);
    }
}

// ❌ Anti-pattern 4: Condition ไม่ชัดเจน
if (!user.Locked && user.Active && DateTime.Now < user.ExpiryDate && user.Role != "Banned")
{
    // ...
}

// ✅ Best practice: Extract method ที่มีชื่อชัดเจน
if (IsUserAllowedToAccess(user))
{
    // ...
}

bool IsUserAllowedToAccess(User2 user)
{
    return !user.IsLocked 
        && user.IsActive 
        && !user.IsExpired() 
        && user.Role != "Banned";
}

// Placeholder types
record Order2(List<string>? Items, decimal TotalAmount);
record User2(bool IsLocked, bool IsActive, string Role)
{
    public bool IsExpired() => false;
}
class Error2
{
    public bool IsSerious { get; set; }
}
Error2? error = null;
User2 user = new(false, true, "User");
void LogError(Error2? e) { }
void NotifyAdmin(Error2? e) { }
void SendEmail(Error2? e) { }
```

---

## แบบฝึกหัด Part 03

### แบบฝึกหัดที่ 3.1: เกมทายตัวเลข
สร้างเกมทายตัวเลขโดย:
- สุ่มตัวเลข 1-100
- ผู้ใช้ทาย
- บอก "มากไป", "น้อยไป", หรือ "ถูกต้อง"
- นับจำนวนครั้งที่ทาย

### แบบฝึกหัดที่ 3.2: ระบบคำนวณคะแนน
สร้างระบบที่รับคะแนน 5 วิชาแล้วคำนวณ:
- ค่าเฉลี่ย
- เกรด (A-F)
- สถิติ min/max
- รายงานผล

### แบบฝึกหัดที่ 3.3: เครื่องคิดเลขขั้นสูง
สร้างเครื่องคิดเลขที่รองรับ:
- บวก ลบ คูณ หาร
- ยกกำลัง (^)
- หาค่าสมบูรณ์ (abs)
- ตรวจสอบ input ที่ไม่ถูกต้อง

---

## เฉลยแบบฝึกหัด Part 03

### เฉลย 3.1: เกมทายตัวเลข

```csharp
using System;

Console.WriteLine("=== เกมทายตัวเลข ===");
Console.WriteLine("ทายตัวเลข 1-100\n");

Random random = new();
int secret = random.Next(1, 101);
int attempts = 0;
bool won = false;
const int MAX_ATTEMPTS = 10;

while (attempts < MAX_ATTEMPTS && !won)
{
    Console.Write($"ครั้งที่ {attempts + 1}/{MAX_ATTEMPTS} - ทาย: ");
    string input = Console.ReadLine() ?? "";
    
    if (!int.TryParse(input, out int guess))
    {
        Console.WriteLine("กรุณาใส่ตัวเลขเท่านั้น");
        continue;
    }
    
    if (guess < 1 || guess > 100)
    {
        Console.WriteLine("ตัวเลขต้องอยู่ระหว่าง 1-100");
        continue;
    }
    
    attempts++;
    
    if (guess == secret)
    {
        won = true;
        Console.WriteLine($"\n🎉 ถูกต้อง! ตัวเลขคือ {secret}");
        Console.WriteLine($"คุณใช้ {attempts} ครั้งในการทาย");
        
        string rating = attempts switch
        {
            1 => "เก่งมาก! โชคดีเป็นพิเศษ!",
            <= 3 => "ยอดเยี่ยม!",
            <= 6 => "ดีมาก!",
            <= 8 => "ดี",
            _ => "ผ่านได้"
        };
        Console.WriteLine($"คะแนน: {rating}");
    }
    else if (guess < secret)
    {
        int diff = secret - guess;
        string hint = diff switch
        {
            <= 5 => "ใกล้มาก! มากกว่านี้นิดเดียว",
            <= 15 => "มากกว่านี้",
            _ => "มากกว่านี้เยอะ"
        };
        Console.WriteLine(hint);
    }
    else
    {
        int diff = guess - secret;
        string hint = diff switch
        {
            <= 5 => "ใกล้มาก! น้อยกว่านี้นิดเดียว",
            <= 15 => "น้อยกว่านี้",
            _ => "น้อยกว่านี้เยอะ"
        };
        Console.WriteLine(hint);
    }
}

if (!won)
{
    Console.WriteLine($"\n😔 หมดครั้งแล้ว! ตัวเลขคือ {secret}");
}
```

### เฉลย 3.2: ระบบคำนวณคะแนน

```csharp
using System;

Console.WriteLine("=== ระบบคำนวณคะแนน ===\n");

string[] subjects = { "คณิตศาสตร์", "ภาษาไทย", "ภาษาอังกฤษ", "วิทยาศาสตร์", "สังคมศึกษา" };
double[] scores = new double[subjects.Length];

// รับคะแนนแต่ละวิชา
for (int i = 0; i < subjects.Length; i++)
{
    while (true)
    {
        Console.Write($"คะแนน {subjects[i]} (0-100): ");
        if (double.TryParse(Console.ReadLine(), out double score) && score >= 0 && score <= 100)
        {
            scores[i] = score;
            break;
        }
        Console.WriteLine("คะแนนต้องอยู่ระหว่าง 0-100");
    }
}

// คำนวณสถิติ
double sum = 0, min = scores[0], max = scores[0];
for (int i = 0; i < scores.Length; i++)
{
    sum += scores[i];
    if (scores[i] < min) min = scores[i];
    if (scores[i] > max) max = scores[i];
}

double average = sum / scores.Length;

// กำหนดเกรด
string grade = average switch
{
    >= 90 => "A",
    >= 80 => "B+",
    >= 75 => "B",
    >= 70 => "C+",
    >= 65 => "C",
    >= 60 => "D+",
    >= 55 => "D",
    _ => "F"
};

// แสดงรายงาน
Console.WriteLine("\n=== รายงานผลการเรียน ===");
Console.WriteLine(new string('-', 35));
for (int i = 0; i < subjects.Length; i++)
{
    string subjectGrade = scores[i] switch
    {
        >= 80 => "ดีมาก",
        >= 70 => "ดี",
        >= 60 => "พอใช้",
        >= 50 => "ผ่าน",
        _ => "ไม่ผ่าน"
    };
    Console.WriteLine($"{subjects[i],-15}: {scores[i],6:F1} ({subjectGrade})");
}
Console.WriteLine(new string('-', 35));
Console.WriteLine($"{"คะแนนเฉลี่ย",-15}: {average,6:F2}");
Console.WriteLine($"{"คะแนนสูงสุด",-15}: {max,6:F1}");
Console.WriteLine($"{"คะแนนต่ำสุด",-15}: {min,6:F1}");
Console.WriteLine($"{"เกรดรวม",-15}: {grade,6}");

string result = grade == "F" ? "ไม่ผ่าน" : "ผ่าน";
Console.WriteLine($"\nผลการเรียน: {result}");
```

---

## สรุป Part 03

ในส่วนนี้เราได้เรียนรู้:

1. **if Statement** - พื้นฐาน, if-else, nested
2. **switch Statement** - แบบเก่าและ switch expression
3. **Pattern Matching** - type, property, positional, relational, logical
4. **Guard Clauses** - Early return pattern
5. **Complex Business Logic** - State machines, validation
6. **List Pattern** - C# 11 features
7. **Deconstruction** - Tuple, Record, Custom
8. **Conditional Compilation** - #if, #define
9. **Best Practices** - Named constants, guard clauses, readable code

## ขั้นต่อไป

ใน Part 04 เราจะเรียนรู้เกี่ยวกับ:
- for loop
- while loop
- foreach loop
- do-while loop
- Loop control (break, continue)
- Nested loops

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 03 | Steps 21-30*
