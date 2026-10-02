# Part 09: Interfaces และ Abstract Classes ขั้นสูง
## Steps 81-90: SOLID Principles และ Design Patterns

---

## Step 81: Abstract Class vs Interface

```csharp
// ============================================
// เมื่อไหร่ควรใช้ Abstract Class
// ============================================

// ใช้ Abstract Class เมื่อ:
// 1. มี shared state หรือ implementation
// 2. Classes มีความสัมพันธ์แบบ "is-a"
// 3. ต้องการ Template Method Pattern
// 4. ต้องการ Constructor logic

public abstract class Repository<T, TId> where T : class
{
    // Shared state
    protected readonly List<T> _items = new();
    
    // Shared implementation
    public IEnumerable<T> GetAll() => _items.ToList();
    
    public int Count() => _items.Count;
    
    // Abstract: แต่ละ repository ต้องนิยามเอง
    public abstract T? GetById(TId id);
    public abstract T Add(T item);
    public abstract bool Delete(TId id);
    
    // Template method
    protected virtual void ValidateBeforeAdd(T item) { }
    
    protected virtual void OnAfterAdd(T item) 
        => Console.WriteLine($"Added: {item}");
    
    // Template method pattern
    public T AddWithValidation(T item)
    {
        ValidateBeforeAdd(item); // hook
        var added = Add(item);
        OnAfterAdd(added); // hook
        return added;
    }
}

// ============================================
// เมื่อไหร่ควรใช้ Interface
// ============================================

// ใช้ Interface เมื่อ:
// 1. Define contract สำหรับ unrelated classes
// 2. ต้องการ multiple inheritance
// 3. Classes มีความสัมพันธ์แบบ "can-do"
// 4. ต้องการ decouple implementation

public interface ILogger
{
    void Log(string message, LogLevel level = LogLevel.Info);
    void LogError(string message, Exception? ex = null);
    void LogWarning(string message);
}

public enum LogLevel { Debug, Info, Warning, Error, Critical }

public interface ICacheable
{
    string CacheKey { get; }
    TimeSpan CacheDuration { get; }
}

public interface IValidatable
{
    bool IsValid();
    IEnumerable<string> GetValidationErrors();
}

// Class ที่ implement หลาย interface
public class Product4 : ICacheable, IValidatable
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    
    // ICacheable
    public string CacheKey => $"product:{Id}";
    public TimeSpan CacheDuration => TimeSpan.FromMinutes(30);
    
    // IValidatable
    public bool IsValid() => !GetValidationErrors().Any();
    
    public IEnumerable<string> GetValidationErrors()
    {
        if (string.IsNullOrWhiteSpace(Name))
            yield return "Name is required";
        if (Price <= 0)
            yield return "Price must be positive";
    }
}

// ============================================
// เปรียบเทียบ
// ============================================

/*
 * Feature          | Abstract Class | Interface
 * -----------------|---------------|----------
 * State            | ✅ มีได้       | ❌ ไม่มี
 * Constructor      | ✅ มีได้       | ❌ ไม่มี
 * Method impl      | ✅ มีได้       | ✅ C#8+ (default)
 * Multiple inherit | ❌ ไม่ได้      | ✅ ได้
 * Access modifiers | ✅ มีได้       | ❌ public only
 * Fields           | ✅ มีได้       | ❌ ไม่มี
 * Use when         | "is-a"         | "can-do"
 */
```

---

## Step 82: Default Interface Implementation (C# 8+)

```csharp
// ============================================
// Default Interface Methods
// ============================================

public interface IShape
{
    double GetArea();
    double GetPerimeter();
    
    // Default implementation
    double GetDiameter() => Math.Sqrt(GetArea() / Math.PI) * 2;
    
    string Describe() => 
        $"Shape: Area={GetArea():F2}, Perimeter={GetPerimeter():F2}";
    
    bool IsLargerThan(IShape other) => GetArea() > other.GetArea();
}

public class Circle3 : IShape
{
    public double Radius { get; }
    public Circle3(double radius) => Radius = radius;
    
    public double GetArea() => Math.PI * Radius * Radius;
    public double GetPerimeter() => 2 * Math.PI * Radius;
    // GetDiameter และ Describe จะใช้ default implementation
}

public class Square : IShape
{
    public double Side { get; }
    public Square(double side) => Side = side;
    
    public double GetArea() => Side * Side;
    public double GetPerimeter() => 4 * Side;
    
    // Override default implementation
    public string Describe() => $"Square: Side={Side}, Area={GetArea():F2}";
}

// ============================================
// Interface Evolution
// ============================================

// Version 1: ออก interface แรก
public interface INotification_V1
{
    void Send(string message, string recipient);
}

// Version 2: เพิ่ม method โดยไม่ทำลาย existing code
public interface INotification_V2 : INotification_V1
{
    // เพิ่ม default implementation ให้ backward compatible
    void SendBatch(IEnumerable<string> messages, string recipient)
    {
        foreach (var msg in messages)
            Send(msg, recipient);
    }
    
    Task SendAsync(string message, string recipient)
        => Task.Run(() => Send(message, recipient));
}

// ============================================
// Interface Segregation Principle (I in SOLID)
// ============================================

// ❌ Bad: Fat interface
public interface IBadWorker
{
    void Work();
    void Eat();
    void Sleep();
    void Drive();
    void Program();
}

// ✅ Good: Segregated interfaces
public interface IWorker { void Work(); }
public interface IEater { void Eat(); }
public interface ISleeper { void Sleep(); }
public interface IDriver { void Drive(); }
public interface IProgrammer { void Program(); }

// Classes ใช้เฉพาะ interface ที่จำเป็น
public class Programmer3 : IWorker, IEater, ISleeper, IProgrammer
{
    public void Work() => Console.WriteLine("Working...");
    public void Eat() => Console.WriteLine("Eating lunch...");
    public void Sleep() => Console.WriteLine("Sleeping...");
    public void Program() => Console.WriteLine("Coding...");
}

public class Driver3 : IWorker, IEater, ISleeper, IDriver
{
    public void Work() => Console.WriteLine("Working...");
    public void Eat() => Console.WriteLine("Eating on the road...");
    public void Sleep() => Console.WriteLine("Sleeping at truck stop...");
    public void Drive() => Console.WriteLine("Driving...");
}
```

---

## Step 83: SOLID Principles

```csharp
// ============================================
// S - Single Responsibility Principle
// ============================================

// ❌ Bad: Class มีหน้าที่หลายอย่าง
public class BadUserService
{
    public User3 Register(string name, string email, string password)
    {
        // Validate
        if (string.IsNullOrEmpty(email)) throw new Exception("Invalid email");
        
        // Hash password
        string hash = BCryptSimple(password);
        
        // Save to database
        var user = new User3 { Name = name, Email = email, PasswordHash = hash };
        // db.Save(user);
        
        // Send welcome email
        // emailService.Send(email, "Welcome!", "...");
        
        // Log
        Console.WriteLine($"User registered: {email}");
        
        return user;
    }
    
    private string BCryptSimple(string password) => $"hashed_{password}";
}

// ✅ Good: แยก responsibility
public class UserValidator
{
    public bool ValidateRegistration(string name, string email, string password)
    {
        return !string.IsNullOrWhiteSpace(name) &&
               email.Contains('@') &&
               password.Length >= 8;
    }
}

public class PasswordHasher
{
    public string Hash(string password) => $"bcrypt_{password}"; // simplified
    public bool Verify(string password, string hash) => hash == Hash(password);
}

public class UserRepository3
{
    private readonly List<User3> _users = new();
    
    public User3 Save(User3 user)
    {
        _users.Add(user);
        return user;
    }
    
    public User3? FindByEmail(string email)
        => _users.FirstOrDefault(u => u.Email == email);
}

public class EmailService
{
    public void SendWelcomeEmail(string email, string name)
        => Console.WriteLine($"Sending welcome email to {email}");
}

public class UserService
{
    private readonly UserValidator _validator;
    private readonly PasswordHasher _hasher;
    private readonly UserRepository3 _repository;
    private readonly EmailService _emailService;
    
    public UserService(
        UserValidator validator,
        PasswordHasher hasher,
        UserRepository3 repository,
        EmailService emailService)
    {
        _validator = validator;
        _hasher = hasher;
        _repository = repository;
        _emailService = emailService;
    }
    
    public User3 Register(string name, string email, string password)
    {
        if (!_validator.ValidateRegistration(name, email, password))
            throw new ArgumentException("Invalid registration data");
        
        string hash = _hasher.Hash(password);
        var user = _repository.Save(new User3 { Name = name, Email = email, PasswordHash = hash });
        _emailService.SendWelcomeEmail(email, name);
        
        return user;
    }
}

public class User3
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
}

// ============================================
// O - Open/Closed Principle
// ============================================

// ❌ Bad: ต้องแก้ไข class ทุกครั้งที่เพิ่ม payment method
public class BadPaymentProcessor
{
    public void Process(string paymentType, decimal amount)
    {
        if (paymentType == "CreditCard")
            ProcessCreditCard(amount);
        else if (paymentType == "PayPal")
            ProcessPayPal(amount);
        // ต้องเพิ่ม else if ทุกครั้ง
    }
    
    private void ProcessCreditCard(decimal amount) { }
    private void ProcessPayPal(decimal amount) { }
}

// ✅ Good: Open for extension, Closed for modification
public interface IPaymentMethod
{
    string Name { get; }
    Task<bool> ProcessAsync(decimal amount, string currency);
    bool Supports(string currency);
}

public class CreditCardPayment : IPaymentMethod
{
    public string Name => "Credit Card";
    
    public Task<bool> ProcessAsync(decimal amount, string currency)
    {
        Console.WriteLine($"Processing {amount:C} {currency} via Credit Card");
        return Task.FromResult(true);
    }
    
    public bool Supports(string currency) => true; // รองรับทุก currency
}

public class PromptPayPayment : IPaymentMethod
{
    public string Name => "PromptPay";
    
    public Task<bool> ProcessAsync(decimal amount, string currency)
    {
        Console.WriteLine($"Processing {amount:C} THB via PromptPay");
        return Task.FromResult(true);
    }
    
    public bool Supports(string currency) => currency == "THB";
}

public class PaymentProcessor2
{
    private readonly List<IPaymentMethod> _methods = new();
    
    public void Register(IPaymentMethod method) => _methods.Add(method);
    
    public async Task<bool> ProcessAsync(string methodName, decimal amount, string currency = "THB")
    {
        var method = _methods.FirstOrDefault(m => 
            m.Name.Equals(methodName, StringComparison.OrdinalIgnoreCase));
        
        if (method == null)
            throw new InvalidOperationException($"Payment method '{methodName}' not found");
        
        if (!method.Supports(currency))
            throw new InvalidOperationException($"{methodName} doesn't support {currency}");
        
        return await method.ProcessAsync(amount, currency);
    }
}

// ============================================
// L - Liskov Substitution Principle
// ============================================

// ❌ Bad: subclass ทำงานแตกต่างจาก parent โดยไม่เหมาะสม
public class BadRectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class BadSquare : BadRectangle
{
    // Square บังคับให้ Width = Height เสมอ
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // ทำลาย parent contract!
    }
}

// Code ที่ใช้ Rectangle จะแตกเมื่อส่ง Square
void ResizeRectangle(BadRectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    // คาดว่า Area = 50, แต่ Square จะเป็น 100!
}

// ✅ Good: Design hierarchy ให้ถูกต้อง
public abstract class Shape2
{
    public abstract int Area { get; }
}

public class Rectangle3 : Shape2
{
    public int Width { get; }
    public int Height { get; }
    
    public Rectangle3(int width, int height)
    {
        Width = width;
        Height = height;
    }
    
    public override int Area => Width * Height;
}

public class Square2 : Shape2
{
    public int Side { get; }
    
    public Square2(int side) => Side = side;
    
    public override int Area => Side * Side;
}

// ============================================
// D - Dependency Inversion Principle
// ============================================

// ❌ Bad: High-level depends on Low-level
public class BadEmailNotificationService
{
    private GmailSender _sender = new GmailSender(); // concrete dependency!
    
    public void NotifyUser(string message, string email)
    {
        _sender.Send(email, "Notification", message);
    }
}

public class GmailSender
{
    public void Send(string to, string subject, string body)
        => Console.WriteLine($"Gmail: {to} - {subject}");
}

// ✅ Good: Depend on abstractions
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body);
}

public class GoodEmailNotificationService
{
    private readonly IEmailSender _sender; // abstraction dependency
    
    public GoodEmailNotificationService(IEmailSender sender)
    {
        _sender = sender; // injected from outside
    }
    
    public Task NotifyUserAsync(string message, string email)
        => _sender.SendAsync(email, "Notification", message);
}

// Implementations can change without touching GoodEmailNotificationService
public class GmailEmailSender : IEmailSender
{
    public Task SendAsync(string to, string subject, string body)
    {
        Console.WriteLine($"[Gmail] To: {to}, Subject: {subject}");
        return Task.CompletedTask;
    }
}

public class SendGridEmailSender : IEmailSender
{
    public Task SendAsync(string to, string subject, string body)
    {
        Console.WriteLine($"[SendGrid] To: {to}, Subject: {subject}");
        return Task.CompletedTask;
    }
}
```

---

## Step 84: Dependency Injection Container

```csharp
// ============================================
// Simple DI Container
// ============================================

public class Container
{
    private readonly Dictionary<Type, Func<object>> _registrations = new();
    private readonly Dictionary<Type, object> _singletons = new();
    
    // Register Transient (สร้าง instance ใหม่ทุกครั้ง)
    public void RegisterTransient<TInterface, TImplementation>()
        where TImplementation : TInterface, new()
    {
        _registrations[typeof(TInterface)] = () => new TImplementation();
    }
    
    // Register Singleton (สร้าง instance ครั้งเดียว)
    public void RegisterSingleton<TInterface, TImplementation>()
        where TImplementation : TInterface, new()
    {
        _registrations[typeof(TInterface)] = () =>
        {
            var type = typeof(TInterface);
            if (!_singletons.ContainsKey(type))
                _singletons[type] = new TImplementation();
            return _singletons[type];
        };
    }
    
    // Register Instance
    public void RegisterInstance<TInterface>(TInterface instance) where TInterface : notnull
    {
        _registrations[typeof(TInterface)] = () => instance;
    }
    
    // Resolve
    public T Resolve<T>()
    {
        var type = typeof(T);
        if (_registrations.TryGetValue(type, out var factory))
            return (T)factory();
        
        throw new InvalidOperationException($"No registration for {type.Name}");
    }
}

// ============================================
// Microsoft DI (สำหรับ MAUI)
// ============================================

// ใน MauiProgram.cs (จะเรียนใน Part ต่อๆ ไป)
/*
var builder = MauiApp.CreateBuilder();
builder.Services
    .AddSingleton<ILogger, ConsoleLogger>()
    .AddTransient<IUserRepository, UserRepository>()
    .AddScoped<UserService>()
    .AddSingleton<IEmailSender, GmailEmailSender>();
*/

// ============================================
// Service Locator Pattern (ไม่แนะนำ แต่รู้ไว้)
// ============================================

public class ServiceLocator
{
    private static readonly Lazy<ServiceLocator> _instance 
        = new(() => new ServiceLocator());
    
    public static ServiceLocator Current => _instance.Value;
    
    private readonly Dictionary<Type, object> _services = new();
    
    private ServiceLocator() { }
    
    public void Register<T>(T service) where T : notnull
        => _services[typeof(T)] = service;
    
    public T Get<T>()
    {
        if (_services.TryGetValue(typeof(T), out var service))
            return (T)service;
        throw new InvalidOperationException($"Service {typeof(T).Name} not registered");
    }
}
```

---

## Step 85: Generic Constraints และ Advanced Generics

```csharp
// ============================================
// Advanced Generic Constraints
// ============================================

// Multiple constraints
T CreateAndInitialize<T>() 
    where T : class, IInitializable, new()
{
    var obj = new T();
    obj.Initialize();
    return obj;
}

public interface IInitializable
{
    void Initialize();
}

// Generic class with constraints
public class DataStore<T> where T : class, IEquatable<T>
{
    private readonly HashSet<T> _data = new();
    
    public bool Add(T item) => _data.Add(item);
    
    public bool Contains(T item) => _data.Contains(item);
    
    public IEnumerable<T> GetAll() => _data.ToList();
}

// ============================================
// Generic Interfaces
// ============================================

public interface IConverter<TSource, TDestination>
{
    TDestination Convert(TSource source);
    IEnumerable<TDestination> ConvertAll(IEnumerable<TSource> sources)
        => sources.Select(Convert);
}

public record PersonDto(string Name, int Age);
public record PersonEntity(int Id, string FullName, int Age, DateTime CreatedAt);

public class PersonDtoToEntityConverter : IConverter<PersonDto, PersonEntity>
{
    private static int _nextId = 1;
    
    public PersonEntity Convert(PersonDto dto)
        => new PersonEntity(_nextId++, dto.Name, dto.Age, DateTime.Now);
}

// ============================================
// Generic Builder Pattern
// ============================================

public class Builder<T> where T : class
{
    private readonly T _instance;
    
    public Builder(T instance) => _instance = instance;
    
    public Builder<T> Set<TValue>(
        System.Linq.Expressions.Expression<Func<T, TValue>> propertyExpr, 
        TValue value)
    {
        if (propertyExpr.Body is System.Linq.Expressions.MemberExpression memberExpr)
        {
            var property = (System.Reflection.PropertyInfo)memberExpr.Member;
            property.SetValue(_instance, value);
        }
        return this;
    }
    
    public T Build() => _instance;
}

// ใช้งาน
class Config
{
    public string Host { get; set; } = "localhost";
    public int Port { get; set; } = 8080;
    public bool UseSsl { get; set; }
    public string ApiKey { get; set; } = string.Empty;
}

var config = new Builder<Config>(new Config())
    .Set(c => c.Host, "api.example.com")
    .Set(c => c.Port, 443)
    .Set(c => c.UseSsl, true)
    .Set(c => c.ApiKey, "secret-key")
    .Build();
```

---

## Step 86: Fluent Interface ขั้นสูง

```csharp
// ============================================
// Fluent Builder สำหรับ HTTP Request
// ============================================

public class HttpRequestBuilder
{
    private string _url = string.Empty;
    private string _method = "GET";
    private readonly Dictionary<string, string> _headers = new();
    private string? _body;
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);
    
    public HttpRequestBuilder Url(string url)
    {
        _url = url;
        return this;
    }
    
    public HttpRequestBuilder Method(string method)
    {
        _method = method.ToUpper();
        return this;
    }
    
    public HttpRequestBuilder Get() => Method("GET");
    public HttpRequestBuilder Post() => Method("POST");
    public HttpRequestBuilder Put() => Method("PUT");
    public HttpRequestBuilder Delete() => Method("DELETE");
    
    public HttpRequestBuilder Header(string key, string value)
    {
        _headers[key] = value;
        return this;
    }
    
    public HttpRequestBuilder Authorization(string token)
        => Header("Authorization", $"Bearer {token}");
    
    public HttpRequestBuilder ContentType(string type)
        => Header("Content-Type", type);
    
    public HttpRequestBuilder JsonBody(object body)
    {
        _body = System.Text.Json.JsonSerializer.Serialize(body);
        return ContentType("application/json");
    }
    
    public HttpRequestBuilder Timeout(TimeSpan timeout)
    {
        _timeout = timeout;
        return this;
    }
    
    public HttpRequest Build()
    {
        if (string.IsNullOrWhiteSpace(_url))
            throw new InvalidOperationException("URL is required");
        
        return new HttpRequest(_url, _method, _headers, _body, _timeout);
    }
    
    public async Task<string> ExecuteAsync()
    {
        var request = Build();
        // In real implementation: use HttpClient
        Console.WriteLine($"{request.Method} {request.Url}");
        foreach (var (key, value) in request.Headers)
            Console.WriteLine($"  {key}: {value}");
        if (request.Body != null)
            Console.WriteLine($"  Body: {request.Body}");
        return "{ \"success\": true }";
    }
}

public record HttpRequest(
    string Url, 
    string Method, 
    Dictionary<string, string> Headers,
    string? Body,
    TimeSpan Timeout
);

// ใช้งาน
var response = await new HttpRequestBuilder()
    .Url("https://api.example.com/users")
    .Post()
    .Authorization("my-token-123")
    .JsonBody(new { name = "สมชาย", email = "somchai@example.com" })
    .Timeout(TimeSpan.FromSeconds(10))
    .ExecuteAsync();

Console.WriteLine($"Response: {response}");
```

---

## Step 87: Decorator Pattern

```csharp
// ============================================
// Decorator Pattern
// ============================================

// เพิ่ม behavior โดยไม่แก้ไข original class

public interface ICoffee
{
    string GetDescription();
    decimal GetCost();
}

public class SimpleCoffee : ICoffee
{
    public string GetDescription() => "กาแฟ";
    public decimal GetCost() => 25m;
}

// Decorators
public abstract class CoffeeDecorator : ICoffee
{
    protected readonly ICoffee _coffee;
    
    protected CoffeeDecorator(ICoffee coffee) => _coffee = coffee;
    
    public virtual string GetDescription() => _coffee.GetDescription();
    public virtual decimal GetCost() => _coffee.GetCost();
}

public class MilkDecorator : CoffeeDecorator
{
    public MilkDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() => $"{base.GetDescription()}, นม";
    public override decimal GetCost() => base.GetCost() + 10m;
}

public class SugarDecorator : CoffeeDecorator
{
    private readonly int _teaspoons;
    
    public SugarDecorator(ICoffee coffee, int teaspoons = 1) : base(coffee)
        => _teaspoons = teaspoons;
    
    public override string GetDescription() 
        => $"{base.GetDescription()}, น้ำตาล {_teaspoons} ช้อน";
    public override decimal GetCost() => base.GetCost() + _teaspoons * 2m;
}

public class WhipDecorator : CoffeeDecorator
{
    public WhipDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() => $"{base.GetDescription()}, วิปครีม";
    public override decimal GetCost() => base.GetCost() + 15m;
}

public class VanillaDecorator : CoffeeDecorator
{
    public VanillaDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() => $"{base.GetDescription()}, วานิลลา";
    public override decimal GetCost() => base.GetCost() + 20m;
}

// ใช้งาน
ICoffee order = new SimpleCoffee();
Console.WriteLine($"{order.GetDescription()} = {order.GetCost():C}");

order = new MilkDecorator(order);
order = new SugarDecorator(order, 2);
Console.WriteLine($"{order.GetDescription()} = {order.GetCost():C}");

// ชงกาแฟพิเศษ
ICoffee specialOrder = new VanillaDecorator(
    new WhipDecorator(
        new MilkDecorator(
            new SugarDecorator(
                new SimpleCoffee(), 1))));

Console.WriteLine($"{specialOrder.GetDescription()} = {specialOrder.GetCost():C}");

// ============================================
// Decorator ใน .NET (Logging)
// ============================================

public interface IOrderService2
{
    Task<Order4> CreateOrderAsync(string customerId, List<string> items);
    Task<bool> CancelOrderAsync(string orderId);
}

public class OrderService2 : IOrderService2
{
    public Task<Order4> CreateOrderAsync(string customerId, List<string> items)
    {
        var order = new Order4(Guid.NewGuid().ToString(), customerId, items);
        return Task.FromResult(order);
    }
    
    public Task<bool> CancelOrderAsync(string orderId) => Task.FromResult(true);
}

public record Order4(string Id, string CustomerId, List<string> Items);

// Logging Decorator
public class LoggingOrderService : IOrderService2
{
    private readonly IOrderService2 _inner;
    private readonly ILogger _logger;
    
    public LoggingOrderService(IOrderService2 inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }
    
    public async Task<Order4> CreateOrderAsync(string customerId, List<string> items)
    {
        _logger.Log($"Creating order for {customerId}", LogLevel.Info);
        var order = await _inner.CreateOrderAsync(customerId, items);
        _logger.Log($"Order {order.Id} created", LogLevel.Info);
        return order;
    }
    
    public async Task<bool> CancelOrderAsync(string orderId)
    {
        _logger.Log($"Cancelling order {orderId}", LogLevel.Warning);
        var result = await _inner.CancelOrderAsync(orderId);
        _logger.Log($"Order {orderId} {(result ? "cancelled" : "cancel failed")}", LogLevel.Info);
        return result;
    }
}

// Caching Decorator
public class CachingOrderService : IOrderService2
{
    private readonly IOrderService2 _inner;
    private readonly Dictionary<string, Order4> _cache = new();
    
    public CachingOrderService(IOrderService2 inner) => _inner = inner;
    
    public async Task<Order4> CreateOrderAsync(string customerId, List<string> items)
    {
        var order = await _inner.CreateOrderAsync(customerId, items);
        _cache[order.Id] = order;
        return order;
    }
    
    public async Task<bool> CancelOrderAsync(string orderId)
    {
        var result = await _inner.CancelOrderAsync(orderId);
        if (result) _cache.Remove(orderId);
        return result;
    }
}

public class ConsoleLogger : ILogger
{
    public void Log(string message, LogLevel level = LogLevel.Info)
        => Console.WriteLine($"[{level}] {DateTime.Now:HH:mm:ss} {message}");
    public void LogError(string message, Exception? ex = null)
        => Log($"ERROR: {message}" + (ex != null ? $" - {ex.Message}" : ""), LogLevel.Error);
    public void LogWarning(string message) => Log(message, LogLevel.Warning);
}
```

---

## Step 88: Chain of Responsibility

```csharp
// ============================================
// Chain of Responsibility Pattern
// ============================================

// Request Processing Pipeline

public abstract class RequestHandler<T, TResult>
{
    private RequestHandler<T, TResult>? _next;
    
    public RequestHandler<T, TResult> SetNext(RequestHandler<T, TResult> next)
    {
        _next = next;
        return next; // สำหรับ chaining
    }
    
    public abstract Task<TResult?> HandleAsync(T request);
    
    protected Task<TResult?> PassToNextAsync(T request)
    {
        return _next?.HandleAsync(request) ?? Task.FromResult<TResult?>(default);
    }
}

// Request/Response
public record ApiRequest(string Method, string Path, Dictionary<string, string> Headers, object? Body);
public record ApiResponse(int StatusCode, string Message, object? Data = null);

// Handlers
public class AuthenticationHandler : RequestHandler<ApiRequest, ApiResponse>
{
    public override Task<ApiResponse?> HandleAsync(ApiRequest request)
    {
        Console.WriteLine("[Auth] Checking authentication...");
        
        if (!request.Headers.TryGetValue("Authorization", out string? authHeader) ||
            !authHeader.StartsWith("Bearer "))
        {
            return Task.FromResult<ApiResponse?>(new ApiResponse(401, "Unauthorized"));
        }
        
        Console.WriteLine("[Auth] Authenticated ✓");
        return PassToNextAsync(request);
    }
}

public class RateLimitHandler : RequestHandler<ApiRequest, ApiResponse>
{
    private readonly Dictionary<string, int> _requestCounts = new();
    private const int MAX_REQUESTS_PER_MINUTE = 60;
    
    public override Task<ApiResponse?> HandleAsync(ApiRequest request)
    {
        Console.WriteLine("[RateLimit] Checking rate limit...");
        
        string key = request.Headers.GetValueOrDefault("X-Client-Id", "anonymous");
        _requestCounts[key] = _requestCounts.GetValueOrDefault(key, 0) + 1;
        
        if (_requestCounts[key] > MAX_REQUESTS_PER_MINUTE)
        {
            return Task.FromResult<ApiResponse?>(new ApiResponse(429, "Too Many Requests"));
        }
        
        Console.WriteLine($"[RateLimit] Rate: {_requestCounts[key]}/{MAX_REQUESTS_PER_MINUTE} ✓");
        return PassToNextAsync(request);
    }
}

public class ValidationHandler : RequestHandler<ApiRequest, ApiResponse>
{
    public override Task<ApiResponse?> HandleAsync(ApiRequest request)
    {
        Console.WriteLine("[Validation] Validating request...");
        
        if (string.IsNullOrEmpty(request.Path))
        {
            return Task.FromResult<ApiResponse?>(new ApiResponse(400, "Path is required"));
        }
        
        if (request.Method is "POST" or "PUT" && request.Body == null)
        {
            return Task.FromResult<ApiResponse?>(new ApiResponse(400, "Body required for POST/PUT"));
        }
        
        Console.WriteLine("[Validation] Valid ✓");
        return PassToNextAsync(request);
    }
}

public class BusinessLogicHandler : RequestHandler<ApiRequest, ApiResponse>
{
    public override Task<ApiResponse?> HandleAsync(ApiRequest request)
    {
        Console.WriteLine("[Business] Processing request...");
        
        var response = request.Path switch
        {
            "/users" => new ApiResponse(200, "OK", new[] { "Alice", "Bob", "Carol" }),
            "/products" => new ApiResponse(200, "OK", new[] { "iPhone", "Samsung" }),
            _ => new ApiResponse(404, "Not Found")
        };
        
        Console.WriteLine($"[Business] Processed: {response.StatusCode} ✓");
        return Task.FromResult<ApiResponse?>(response);
    }
}

// สร้าง pipeline
var pipeline = new AuthenticationHandler();
pipeline
    .SetNext(new RateLimitHandler())
    .SetNext(new ValidationHandler())
    .SetNext(new BusinessLogicHandler());

// ทดสอบ
var request = new ApiRequest(
    "GET",
    "/users",
    new Dictionary<string, string>
    {
        { "Authorization", "Bearer valid-token-123" },
        { "X-Client-Id", "client-001" }
    },
    null
);

Console.WriteLine("=== Processing Request ===");
var response2 = await pipeline.HandleAsync(request);
Console.WriteLine($"\nResponse: {response2?.StatusCode} - {response2?.Message}");
```

---

## Step 89: Command Pattern

```csharp
// ============================================
// Command Pattern - Undo/Redo
// ============================================

public interface ICommand
{
    void Execute();
    void Undo();
    string Description { get; }
}

public class TextEditor
{
    private string _content = string.Empty;
    private int _cursorPosition = 0;
    
    public string Content => _content;
    public int CursorPosition => _cursorPosition;
    
    public void InsertText(string text, int position)
    {
        _content = _content[..position] + text + _content[position..];
        _cursorPosition = position + text.Length;
    }
    
    public void DeleteText(int startPosition, int length)
    {
        if (startPosition + length > _content.Length)
            length = _content.Length - startPosition;
        _content = _content[..startPosition] + _content[(startPosition + length)..];
        _cursorPosition = startPosition;
    }
    
    public void Print()
    {
        Console.WriteLine($"Content: \"{_content}\"");
        Console.WriteLine($"Cursor: {_cursorPosition}");
    }
}

public class InsertTextCommand : ICommand
{
    private readonly TextEditor _editor;
    private readonly string _text;
    private readonly int _position;
    
    public InsertTextCommand(TextEditor editor, string text, int position)
    {
        _editor = editor;
        _text = text;
        _position = position;
        Description = $"Insert \"{text}\" at {position}";
    }
    
    public string Description { get; }
    
    public void Execute() => _editor.InsertText(_text, _position);
    public void Undo() => _editor.DeleteText(_position, _text.Length);
}

public class DeleteTextCommand : ICommand
{
    private readonly TextEditor _editor;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText = string.Empty;
    
    public DeleteTextCommand(TextEditor editor, int position, int length)
    {
        _editor = editor;
        _position = position;
        _length = length;
        Description = $"Delete {length} chars at {position}";
    }
    
    public string Description { get; }
    
    public void Execute()
    {
        _deletedText = _editor.Content.Substring(_position, 
            Math.Min(_length, _editor.Content.Length - _position));
        _editor.DeleteText(_position, _length);
    }
    
    public void Undo() => _editor.InsertText(_deletedText, _position);
}

public class CommandHistory
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();
    
    public void Execute(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear(); // Clear redo หลัง execute ใหม่
        Console.WriteLine($"Executed: {command.Description}");
    }
    
    public void Undo()
    {
        if (!_undoStack.Any())
        {
            Console.WriteLine("Nothing to undo");
            return;
        }
        
        var command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
        Console.WriteLine($"Undone: {command.Description}");
    }
    
    public void Redo()
    {
        if (!_redoStack.Any())
        {
            Console.WriteLine("Nothing to redo");
            return;
        }
        
        var command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
        Console.WriteLine($"Redone: {command.Description}");
    }
}

// ทดสอบ
var editor = new TextEditor();
var history = new CommandHistory();

history.Execute(new InsertTextCommand(editor, "Hello", 0));
editor.Print();

history.Execute(new InsertTextCommand(editor, ", World", 5));
editor.Print();

history.Execute(new DeleteTextCommand(editor, 5, 7));
editor.Print();

history.Undo();
editor.Print();

history.Undo();
editor.Print();

history.Redo();
editor.Print();
```

---

## Step 90: Proxy Pattern

```csharp
// ============================================
// Proxy Pattern
// ============================================

public interface IDataService
{
    Task<string> GetDataAsync(string key);
    Task SetDataAsync(string key, string value);
}

// Real implementation
public class DatabaseService : IDataService
{
    public async Task<string> GetDataAsync(string key)
    {
        await Task.Delay(100); // Simulate DB query
        return $"DB:{key}:value";
    }
    
    public async Task SetDataAsync(string key, string value)
    {
        await Task.Delay(50); // Simulate DB write
        Console.WriteLine($"DB: saved {key}={value}");
    }
}

// Cache Proxy
public class CachedDataService : IDataService
{
    private readonly IDataService _inner;
    private readonly Dictionary<string, (string Value, DateTime Expiry)> _cache = new();
    private readonly TimeSpan _cacheDuration;
    
    public CachedDataService(IDataService inner, TimeSpan? cacheDuration = null)
    {
        _inner = inner;
        _cacheDuration = cacheDuration ?? TimeSpan.FromMinutes(5);
    }
    
    public async Task<string> GetDataAsync(string key)
    {
        if (_cache.TryGetValue(key, out var cached) && cached.Expiry > DateTime.Now)
        {
            Console.WriteLine($"Cache HIT: {key}");
            return cached.Value;
        }
        
        Console.WriteLine($"Cache MISS: {key}");
        string value = await _inner.GetDataAsync(key);
        _cache[key] = (value, DateTime.Now.Add(_cacheDuration));
        return value;
    }
    
    public async Task SetDataAsync(string key, string value)
    {
        await _inner.SetDataAsync(key, value);
        _cache[key] = (value, DateTime.Now.Add(_cacheDuration));
    }
}

// Virtual Proxy (Lazy loading)
public class LazyDataService : IDataService
{
    private IDataService? _realService;
    
    private IDataService RealService 
        => _realService ??= new DatabaseService();
    
    public Task<string> GetDataAsync(string key) => RealService.GetDataAsync(key);
    public Task SetDataAsync(string key, string value) => RealService.SetDataAsync(key, value);
}

// ใช้งาน
IDataService service = new CachedDataService(new DatabaseService());

Console.WriteLine(await service.GetDataAsync("user:1")); // MISS - hit DB
Console.WriteLine(await service.GetDataAsync("user:1")); // HIT - from cache
Console.WriteLine(await service.GetDataAsync("user:2")); // MISS
await service.SetDataAsync("setting:theme", "dark");
Console.WriteLine(await service.GetDataAsync("setting:theme")); // HIT - just set
```

---

## สรุป Part 09

ในส่วนนี้เราได้เรียนรู้:

1. **Abstract vs Interface** - เมื่อไหร่ใช้อะไร
2. **Default Interface Methods** - C# 8+ evolution
3. **Interface Segregation** - ISP principle
4. **SOLID Principles** - S, O, L, D รายละเอียด
5. **Dependency Injection** - Container, DI patterns
6. **Generic Constraints** - Advanced generics
7. **Decorator Pattern** - Adding behavior dynamically
8. **Chain of Responsibility** - Request pipeline
9. **Command Pattern** - Undo/Redo
10. **Proxy Pattern** - Cache, Lazy loading

## ขั้นต่อไป

ใน Part 10 เราจะเรียนรู้เกี่ยวกับ:
- Exception Handling
- Custom Exceptions
- try/catch/finally
- Exception hierarchies
- Global exception handling
- Logging errors

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 09 | Steps 81-90*
