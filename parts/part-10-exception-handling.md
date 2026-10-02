# Part 10: Exception Handling
## Steps 91-100: การจัดการข้อผิดพลาด

---

## Step 91: Exception Basics

```csharp
// ============================================
// Exception Hierarchy
// ============================================

/*
 * System.Object
 *   └── System.Exception
 *         ├── System.SystemException
 *         │     ├── NullReferenceException
 *         │     ├── ArgumentException
 *         │     │     ├── ArgumentNullException
 *         │     │     └── ArgumentOutOfRangeException
 *         │     ├── InvalidOperationException
 *         │     ├── IndexOutOfRangeException
 *         │     ├── OverflowException
 *         │     ├── DivideByZeroException
 *         │     ├── NotImplementedException
 *         │     ├── NotSupportedException
 *         │     ├── IOException
 *         │     │     ├── FileNotFoundException
 *         │     │     └── DirectoryNotFoundException
 *         │     └── OutOfMemoryException
 *         └── System.ApplicationException (ใช้สำหรับ custom)
 */

// ============================================
// try/catch/finally
// ============================================

void DemoBasicExceptions()
{
    // 1. Basic try/catch
    try
    {
        int result = 10 / 0;
    }
    catch (DivideByZeroException ex)
    {
        Console.WriteLine($"Division error: {ex.Message}");
    }
    
    // 2. Multiple catch blocks
    string? input = null;
    try
    {
        int value = int.Parse(input!); // ArgumentNullException
        int[] arr = new int[5];
        arr[10] = value; // IndexOutOfRangeException
    }
    catch (ArgumentNullException ex)
    {
        Console.WriteLine($"Null argument: {ex.ParamName}");
    }
    catch (FormatException ex)
    {
        Console.WriteLine($"Format error: {ex.Message}");
    }
    catch (IndexOutOfRangeException ex)
    {
        Console.WriteLine($"Index error: {ex.Message}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Unexpected error: {ex.GetType().Name} - {ex.Message}");
    }
    
    // 3. finally block
    FileStream? file = null;
    try
    {
        file = File.OpenRead("data.txt");
        // process file...
    }
    catch (FileNotFoundException)
    {
        Console.WriteLine("File not found");
    }
    finally
    {
        file?.Dispose();
        Console.WriteLine("Cleanup complete");
    }
}

// ============================================
// Exception Properties
// ============================================

void InspectException()
{
    try
    {
        throw new InvalidOperationException("Outer exception",
            new ArgumentException("Inner exception", "param1"));
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Type: {ex.GetType().Name}");
        Console.WriteLine($"Message: {ex.Message}");
        Console.WriteLine($"Source: {ex.Source}");
        Console.WriteLine($"StackTrace: {ex.StackTrace?[..100]}...");
        
        if (ex.InnerException != null)
        {
            Console.WriteLine($"Inner: {ex.InnerException.Message}");
        }
    }
}
```

---

## Step 92: Custom Exceptions

```csharp
// ============================================
// Custom Exception Classes
// ============================================

// Base custom exception
public class AppException : Exception
{
    public string ErrorCode { get; }
    public object? Context { get; }
    
    public AppException(string errorCode, string message) 
        : base(message)
    {
        ErrorCode = errorCode;
    }
    
    public AppException(string errorCode, string message, Exception innerException) 
        : base(message, innerException)
    {
        ErrorCode = errorCode;
    }
    
    public AppException(string errorCode, string message, object context) 
        : base(message)
    {
        ErrorCode = errorCode;
        Context = context;
    }
}

// Domain-specific exceptions
public class ValidationException2 : AppException
{
    public IEnumerable<string> Errors { get; }
    
    public ValidationException2(IEnumerable<string> errors) 
        : base("VALIDATION_ERROR", "Validation failed")
    {
        Errors = errors.ToList();
    }
    
    public override string ToString()
        => $"{base.ToString()}\nErrors:\n{string.Join("\n", Errors.Select(e => $"  - {e}"))}";
}

public class NotFoundException2 : AppException
{
    public string ResourceType { get; }
    public object ResourceId { get; }
    
    public NotFoundException2(string resourceType, object resourceId) 
        : base("NOT_FOUND", $"{resourceType} with ID '{resourceId}' not found")
    {
        ResourceType = resourceType;
        ResourceId = resourceId;
    }
}

public class UnauthorizedException : AppException
{
    public UnauthorizedException(string action) 
        : base("UNAUTHORIZED", $"Not authorized to perform: {action}")
    {
    }
}

public class ConflictException : AppException
{
    public ConflictException(string message) 
        : base("CONFLICT", message)
    {
    }
}

// ============================================
// Domain Exceptions
// ============================================

public class InsufficientFundsException : AppException
{
    public decimal CurrentBalance { get; }
    public decimal RequiredAmount { get; }
    
    public InsufficientFundsException(decimal current, decimal required) 
        : base("INSUFFICIENT_FUNDS", 
              $"Insufficient funds: have {current:C}, need {required:C}")
    {
        CurrentBalance = current;
        RequiredAmount = required;
    }
}

public class OrderAlreadyPaidException : AppException
{
    public string OrderId { get; }
    
    public OrderAlreadyPaidException(string orderId) 
        : base("ORDER_ALREADY_PAID", $"Order {orderId} has already been paid")
    {
        OrderId = orderId;
    }
}

// ============================================
// Using Custom Exceptions
// ============================================

public class BankAccount
{
    public string AccountNumber { get; }
    public decimal Balance { get; private set; }
    
    public BankAccount(string accountNumber, decimal initialBalance)
    {
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("Account number required", nameof(accountNumber));
        if (initialBalance < 0)
            throw new ArgumentOutOfRangeException(nameof(initialBalance), "Balance cannot be negative");
        
        AccountNumber = accountNumber;
        Balance = initialBalance;
    }
    
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ValidationException2(new[] { "Deposit amount must be positive" });
        
        Balance += amount;
    }
    
    public void Withdraw(decimal amount)
    {
        var errors = new List<string>();
        
        if (amount <= 0)
            errors.Add("Withdrawal amount must be positive");
        
        if (errors.Any())
            throw new ValidationException2(errors);
        
        if (amount > Balance)
            throw new InsufficientFundsException(Balance, amount);
        
        Balance -= amount;
    }
}
```

---

## Step 93: Exception Filters

```csharp
// ============================================
// Exception Filters (C# 6+)
// ============================================

// when clause
void DemoExceptionFilters()
{
    try
    {
        ProcessRequest(-1);
    }
    catch (ArgumentException ex) when (ex.ParamName == "id")
    {
        Console.WriteLine("Invalid ID parameter");
    }
    catch (ArgumentException ex) when (ex.Message.Contains("required"))
    {
        Console.WriteLine("Required field missing");
    }
    catch (ArgumentException ex)
    {
        Console.WriteLine($"Other argument error: {ex.Message}");
    }
}

void ProcessRequest(int id)
{
    if (id < 0)
        throw new ArgumentException("ID must be positive", nameof(id));
}

// ============================================
// Exception Filters สำหรับ HTTP Status
// ============================================

public class HttpException : Exception
{
    public int StatusCode { get; }
    
    public HttpException(int statusCode, string message) : base(message)
        => StatusCode = statusCode;
}

void HandleHttpExceptions()
{
    try
    {
        throw new HttpException(404, "Resource not found");
    }
    catch (HttpException ex) when (ex.StatusCode == 401)
    {
        Console.WriteLine("Authentication required");
    }
    catch (HttpException ex) when (ex.StatusCode == 403)
    {
        Console.WriteLine("Access forbidden");
    }
    catch (HttpException ex) when (ex.StatusCode == 404)
    {
        Console.WriteLine($"Not found: {ex.Message}");
    }
    catch (HttpException ex) when (ex.StatusCode >= 500)
    {
        Console.WriteLine($"Server error: {ex.StatusCode}");
    }
}

// ============================================
// Re-throwing Exceptions
// ============================================

async Task<string> LoadDataAsync(string url)
{
    try
    {
        // Simulate loading
        await Task.Delay(10);
        throw new HttpException(503, "Service unavailable");
    }
    catch (HttpException ex) when (ex.StatusCode >= 500)
    {
        // Log and re-throw with more context
        Console.WriteLine($"Server error from {url}: {ex.StatusCode}");
        throw; // Preserve original stack trace
    }
    catch (Exception ex)
    {
        // Wrap with more context
        throw new AppException("LOAD_FAILED", 
            $"Failed to load data from {url}: {ex.Message}", ex);
    }
}
```

---

## Step 94: Result Pattern

```csharp
// ============================================
// Result Pattern - Alternative to Exceptions
// ============================================

public class Result<T>
{
    public bool IsSuccess { get; private set; }
    public T? Value { get; private set; }
    public string? Error { get; private set; }
    public string? ErrorCode { get; private set; }
    
    private Result() { }
    
    public static Result<T> Success(T value) => new() 
    { 
        IsSuccess = true, Value = value 
    };
    
    public static Result<T> Failure(string error, string? errorCode = null) => new()
    {
        IsSuccess = false, Error = error, ErrorCode = errorCode
    };
    
    // Monadic operations
    public Result<TNext> Map<TNext>(Func<T, TNext> mapper)
    {
        if (!IsSuccess) return Result<TNext>.Failure(Error!, ErrorCode);
        
        try
        {
            return Result<TNext>.Success(mapper(Value!));
        }
        catch (Exception ex)
        {
            return Result<TNext>.Failure(ex.Message);
        }
    }
    
    public async Task<Result<TNext>> MapAsync<TNext>(Func<T, Task<TNext>> mapper)
    {
        if (!IsSuccess) return Result<TNext>.Failure(Error!, ErrorCode);
        
        try
        {
            var result = await mapper(Value!);
            return Result<TNext>.Success(result);
        }
        catch (Exception ex)
        {
            return Result<TNext>.Failure(ex.Message);
        }
    }
    
    public Result<TNext> Bind<TNext>(Func<T, Result<TNext>> binder)
    {
        if (!IsSuccess) return Result<TNext>.Failure(Error!, ErrorCode);
        return binder(Value!);
    }
    
    public T GetValueOrDefault(T defaultValue) 
        => IsSuccess ? Value! : defaultValue;
    
    public void Match(Action<T> onSuccess, Action<string> onFailure)
    {
        if (IsSuccess) onSuccess(Value!);
        else onFailure(Error!);
    }
    
    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
    
    public override string ToString() 
        => IsSuccess ? $"Success({Value})" : $"Failure({ErrorCode}: {Error})";
}

// ============================================
// ใช้งาน Result Pattern
// ============================================

public class UserRegistrationService
{
    private readonly List<string> _emails = new();
    
    public Result<string> Register(string name, string email, string password)
    {
        // Validate
        var validationErrors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(name))
            validationErrors.Add("Name is required");
        if (!email.Contains('@'))
            validationErrors.Add("Invalid email");
        if (password.Length < 8)
            validationErrors.Add("Password must be at least 8 characters");
        
        if (validationErrors.Any())
            return Result<string>.Failure(
                string.Join(", ", validationErrors), "VALIDATION_ERROR");
        
        // Check duplicate
        if (_emails.Contains(email.ToLower()))
            return Result<string>.Failure($"Email {email} already exists", "DUPLICATE_EMAIL");
        
        // Register
        _emails.Add(email.ToLower());
        return Result<string>.Success($"User {name} registered successfully");
    }
}

// ใช้งาน
var service = new UserRegistrationService();

var result1 = service.Register("สมชาย", "somchai@gmail.com", "password123");
result1.Match(
    msg => Console.WriteLine($"✓ {msg}"),
    err => Console.WriteLine($"✗ {err}")
);

var result2 = service.Register("", "invalid-email", "short");
result2.Match(
    msg => Console.WriteLine($"✓ {msg}"),
    err => Console.WriteLine($"✗ {err}")
);

// Chaining results
var finalResult = service.Register("สมหญิง", "somying@gmail.com", "securePass99")
    .Map(msg => msg.ToUpper())
    .Map(msg => $"[{DateTime.Now:HH:mm:ss}] {msg}");

Console.WriteLine(finalResult);
```

---

## Step 95: Global Exception Handling

```csharp
// ============================================
// Global Exception Handler
// ============================================

// Console Application
AppDomain.CurrentDomain.UnhandledException += (sender, e) =>
{
    var ex = (Exception)e.ExceptionObject;
    Console.Error.WriteLine($"FATAL: {ex.Message}");
    Console.Error.WriteLine(ex.StackTrace);
    // Log to file/service
    Environment.Exit(1);
};

// ============================================
// Exception Middleware สำหรับ ASP.NET Core (reference)
// ============================================

/*
// Program.cs
app.UseExceptionHandler(appError =>
{
    appError.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";
        
        var errorFeature = context.Features.Get<IExceptionHandlerFeature>();
        if (errorFeature != null)
        {
            var ex = errorFeature.Error;
            var response = ex switch
            {
                ValidationException2 ve => new { Status = 400, Errors = ve.Errors },
                NotFoundException2 ne => new { Status = 404, Error = ne.Message },
                UnauthorizedException ue => new { Status = 401, Error = ue.Message },
                _ => new { Status = 500, Error = "Internal server error" }
            };
            await context.Response.WriteAsJsonAsync(response);
        }
    });
});
*/

// ============================================
// Logging Exceptions
// ============================================

public class ExceptionLogger
{
    private readonly string _logPath;
    
    public ExceptionLogger(string logPath) => _logPath = logPath;
    
    public async Task LogAsync(Exception ex, string context = "")
    {
        var entry = new
        {
            Timestamp = DateTime.UtcNow,
            Context = context,
            ExceptionType = ex.GetType().Name,
            Message = ex.Message,
            StackTrace = ex.StackTrace,
            InnerException = ex.InnerException?.Message
        };
        
        string json = System.Text.Json.JsonSerializer.Serialize(entry, 
            new System.Text.Json.JsonSerializerOptions { WriteIndented = true });
        
        await File.AppendAllTextAsync(_logPath, json + "\n---\n");
    }
    
    public void Log(Exception ex, string context = "")
        => LogAsync(ex, context).GetAwaiter().GetResult();
}
```

---

## Step 96: Async Exception Handling

```csharp
// ============================================
// Async/Await Exception Handling
// ============================================

// AggregateException จาก Task
async Task HandleAggregateExceptions()
{
    var tasks = new[]
    {
        Task.Run(() => { throw new InvalidOperationException("Task 1 failed"); }),
        Task.Run(() => { throw new ArgumentException("Task 2 failed"); }),
        Task.Run(async () => { await Task.Delay(10); return "Task 3 OK"; })
    };
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch (Exception ex)
    {
        // เมื่อ await Task.WhenAll, จะ throw exception แรก
        Console.WriteLine($"First exception: {ex.Message}");
        
        // ดู exceptions ทั้งหมด
        var allExceptions = tasks
            .Where(t => t.IsFaulted)
            .SelectMany(t => t.Exception!.InnerExceptions);
        
        foreach (var e in allExceptions)
            Console.WriteLine($"  - {e.GetType().Name}: {e.Message}");
    }
}

// ============================================
// Cancellation + Exception
// ============================================

async Task HandleCancellation(CancellationToken cancellationToken)
{
    try
    {
        for (int i = 0; i < 100; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Task.Delay(100, cancellationToken);
            Console.WriteLine($"Processing item {i}");
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Operation was cancelled");
    }
}

// ============================================
// Retry Logic
// ============================================

public static class RetryHelper
{
    public static async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        int maxAttempts = 3,
        TimeSpan? delay = null,
        Func<Exception, bool>? shouldRetry = null)
    {
        delay ??= TimeSpan.FromSeconds(1);
        shouldRetry ??= ex => ex is HttpException he && he.StatusCode >= 500;
        
        Exception lastException = null!;
        
        for (int attempt = 1; attempt <= maxAttempts; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (shouldRetry(ex) && attempt < maxAttempts)
            {
                lastException = ex;
                Console.WriteLine($"Attempt {attempt} failed: {ex.Message}. Retrying...");
                await Task.Delay(delay.Value * attempt); // Exponential backoff
            }
        }
        
        throw new AggregateException($"All {maxAttempts} attempts failed", lastException);
    }
}

// ใช้งาน Retry
int callCount = 0;
var result = await RetryHelper.ExecuteWithRetryAsync(async () =>
{
    callCount++;
    if (callCount < 3)
        throw new HttpException(503, "Service unavailable");
    
    await Task.Delay(10);
    return "Success!";
});

Console.WriteLine($"Result after {callCount} attempts: {result}");
```

---

## Step 97: Using Pattern + Exception Safety

```csharp
// ============================================
// IDisposable และ Resource Safety
// ============================================

public class DatabaseConnection : IDisposable
{
    private bool _disposed = false;
    private bool _isOpen = false;
    
    public DatabaseConnection(string connectionString)
    {
        Console.WriteLine($"Opening connection to: {connectionString}");
        _isOpen = true;
    }
    
    public void Execute(string sql)
    {
        if (_disposed) throw new ObjectDisposedException(nameof(DatabaseConnection));
        if (!_isOpen) throw new InvalidOperationException("Connection is closed");
        
        Console.WriteLine($"Executing: {sql}");
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                Console.WriteLine("Closing database connection");
                _isOpen = false;
            }
            _disposed = true;
        }
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    ~DatabaseConnection() => Dispose(false);
}

// ใช้ using statement - guarantee disposal
void SafeDbOperation()
{
    using var conn = new DatabaseConnection("Server=localhost;DB=mydb");
    conn.Execute("SELECT * FROM users");
    conn.Execute("UPDATE users SET active=1");
} // Dispose() เรียกโดยอัตโนมัติ แม้มี exception

// ============================================
// Safe File Operations
// ============================================

public static class FileHelper
{
    public static async Task<Result<string>> ReadTextAsync(string path)
    {
        try
        {
            if (!File.Exists(path))
                return Result<string>.Failure($"File not found: {path}", "FILE_NOT_FOUND");
            
            string content = await File.ReadAllTextAsync(path);
            return Result<string>.Success(content);
        }
        catch (UnauthorizedAccessException)
        {
            return Result<string>.Failure($"Access denied: {path}", "ACCESS_DENIED");
        }
        catch (IOException ex)
        {
            return Result<string>.Failure($"IO error: {ex.Message}", "IO_ERROR");
        }
    }
    
    public static async Task<Result<bool>> WriteTextAsync(string path, string content)
    {
        try
        {
            string? dir = Path.GetDirectoryName(path);
            if (!string.IsNullOrEmpty(dir) && !Directory.Exists(dir))
                Directory.CreateDirectory(dir);
            
            await File.WriteAllTextAsync(path, content);
            return Result<bool>.Success(true);
        }
        catch (Exception ex)
        {
            return Result<bool>.Failure($"Write failed: {ex.Message}");
        }
    }
}
```

---

## Step 98: Polly - Resilience Library

```csharp
// ============================================
// Polly Patterns (for reference - in real projects use Polly NuGet)
// ============================================

// Simulating Polly-like retry
public class ResiliencePolicy
{
    private int _retries = 3;
    private TimeSpan _retryDelay = TimeSpan.FromSeconds(1);
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);
    private Func<Exception, bool>? _retryCondition;
    
    public ResiliencePolicy WithRetry(int count, TimeSpan? delay = null)
    {
        _retries = count;
        if (delay.HasValue) _retryDelay = delay.Value;
        return this;
    }
    
    public ResiliencePolicy WithRetryOn<TException>(int count) where TException : Exception
    {
        _retries = count;
        _retryCondition = ex => ex is TException;
        return this;
    }
    
    public ResiliencePolicy WithTimeout(TimeSpan timeout)
    {
        _timeout = timeout;
        return this;
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        using var cts = new CancellationTokenSource(_timeout);
        Exception? lastEx = null;
        
        for (int i = 0; i <= _retries; i++)
        {
            try
            {
                return await action().WaitAsync(cts.Token);
            }
            catch (OperationCanceledException) when (cts.IsCancellationRequested)
            {
                throw new TimeoutException($"Operation timed out after {_timeout}");
            }
            catch (Exception ex) when (i < _retries && 
                (_retryCondition == null || _retryCondition(ex)))
            {
                lastEx = ex;
                Console.WriteLine($"Retry {i + 1}/{_retries}: {ex.Message}");
                await Task.Delay(_retryDelay * (i + 1));
            }
        }
        
        throw new Exception($"All retries exhausted", lastEx);
    }
}

// ใช้งาน
var policy = new ResiliencePolicy()
    .WithRetry(3, TimeSpan.FromSeconds(0.5))
    .WithTimeout(TimeSpan.FromSeconds(10));

int attempts = 0;
var data = await policy.ExecuteAsync(async () =>
{
    attempts++;
    if (attempts < 3)
        throw new HttpException(503, "Service overloaded");
    
    await Task.Delay(50);
    return new[] { "item1", "item2", "item3" };
});

Console.WriteLine($"Got {data.Length} items after {attempts} attempts");
```

---

## Step 99: Exception Best Practices

```csharp
// ============================================
// Exception Best Practices
// ============================================

// ✅ 1. Throw exceptions, don't return null for errors
public class ProductService4
{
    private readonly List<Product4> _products = new();
    
    // ❌ Bad: returning null silently
    public Product4? GetProductBad(int id) 
        => _products.FirstOrDefault(p => p.Id == id);
    
    // ✅ Good: throw meaningful exception
    public Product4 GetProduct(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null)
            throw new NotFoundException2("Product", id);
        return product;
    }
    
    // ✅ Good: result pattern when not-found is expected
    public Result<Product4> TryGetProduct(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        return product != null 
            ? Result<Product4>.Success(product)
            : Result<Product4>.Failure($"Product {id} not found", "NOT_FOUND");
    }
}

// ✅ 2. Validate early
public class OrderProcessor
{
    public void PlaceOrder(string? customerId, List<string>? items, decimal total)
    {
        // Guard clauses
        ArgumentException.ThrowIfNullOrEmpty(customerId, nameof(customerId));
        ArgumentNullException.ThrowIfNull(items, nameof(items));
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(total, nameof(total));
        
        if (!items.Any())
            throw new ArgumentException("Order must have at least one item", nameof(items));
        
        // Process order...
    }
}

// ✅ 3. ไม่ catch exception ที่จะ catch ใหม่ทันที
void DontSwallowExceptions()
{
    // ❌ Bad: catch และ ignore
    try { SomeOperation(); }
    catch (Exception) { } // ปิดบัง error!
    
    // ❌ Bad: catch และ re-throw ใหม่ (เสีย stack trace)
    try { SomeOperation(); }
    catch (Exception ex) { throw ex; } // เสีย original stack trace
    
    // ✅ Good: rethrow โดย preserve stack trace
    try { SomeOperation(); }
    catch (Exception ex)
    {
        Console.Error.WriteLine($"Error: {ex}");
        throw; // preserve stack trace
    }
    
    // ✅ Good: wrap เมื่อต้องการ context
    try { SomeOperation(); }
    catch (Exception ex)
    {
        throw new AppException("OPERATION_FAILED", 
            "Failed to complete operation", ex); // InnerException preserved
    }
}

void SomeOperation() { }

// ✅ 4. Exception messages ที่ชัดเจน
// ❌ Bad
throw new Exception("Error");
throw new Exception("Failed");

// ✅ Good  
throw new InvalidOperationException(
    $"Cannot cancel order {orderId} because it's in state {orderState}. " +
    "Orders can only be cancelled when in 'Pending' state.");

// ✅ 5. ใช้ specific exception types
// ❌ Bad
throw new Exception("Null argument");
throw new Exception("Out of range");

// ✅ Good
throw new ArgumentNullException(nameof(customerId), "Customer ID is required");
throw new ArgumentOutOfRangeException(nameof(age), age, "Age must be between 0 and 150");
```

---

## Step 100: Complete Error Handling System

```csharp
// ============================================
// Complete Error Handling System
// ============================================

// Error codes enum
public enum ErrorCode
{
    // General
    Unknown = 0,
    ValidationError = 1000,
    NotFound = 1001,
    Conflict = 1002,
    Unauthorized = 1003,
    Forbidden = 1004,
    
    // Business
    InsufficientFunds = 2000,
    OrderAlreadyPaid = 2001,
    ProductOutOfStock = 2002,
    
    // Technical
    DatabaseError = 3000,
    NetworkError = 3001,
    TimeoutError = 3002
}

// Structured error
public record Error(ErrorCode Code, string Message, string? Details = null)
{
    public static Error NotFound(string resource, object id) 
        => new(ErrorCode.NotFound, $"{resource} not found", $"ID: {id}");
    
    public static Error Validation(string message, string? details = null) 
        => new(ErrorCode.ValidationError, message, details);
    
    public static Error Unauthorized(string action) 
        => new(ErrorCode.Unauthorized, $"Not authorized to {action}");
    
    public static Error Unknown(string details) 
        => new(ErrorCode.Unknown, "An unexpected error occurred", details);
}

// Result with structured error
public class Result2<T>
{
    private Result2() { }
    
    public bool IsSuccess { get; private init; }
    public T? Value { get; private init; }
    public Error? Error { get; private init; }
    
    public static Result2<T> Ok(T value) => new() { IsSuccess = true, Value = value };
    public static Result2<T> Fail(Error error) => new() { IsSuccess = false, Error = error };
    
    public Result2<TNext> Map<TNext>(Func<T, TNext> mapper)
        => IsSuccess 
            ? Result2<TNext>.Ok(mapper(Value!)) 
            : Result2<TNext>.Fail(Error!);
    
    public void Deconstruct(out bool isSuccess, out T? value, out Error? error)
    {
        isSuccess = IsSuccess;
        value = Value;
        error = Error;
    }
}

// Service using structured results
public class ProductCatalogService
{
    private readonly List<Product4> _products = new()
    {
        new Product4 { Id = 1, Name = "iPhone 15", Price = 35000 },
        new Product4 { Id = 2, Name = "Samsung S24", Price = 30000 }
    };
    
    public Result2<Product4> GetProduct(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        
        return product != null 
            ? Result2<Product4>.Ok(product)
            : Result2<Product4>.Fail(Error.NotFound("Product", id));
    }
    
    public Result2<Product4> UpdatePrice(int id, decimal newPrice)
    {
        if (newPrice <= 0)
            return Result2<Product4>.Fail(
                Error.Validation("Price must be positive", $"Provided: {newPrice}"));
        
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null)
            return Result2<Product4>.Fail(Error.NotFound("Product", id));
        
        product.Price = newPrice;
        return Result2<Product4>.Ok(product);
    }
}

// ใช้ deconstruct
var catalogService = new ProductCatalogService();

var (isSuccess, product, error) = catalogService.GetProduct(1);
if (isSuccess)
    Console.WriteLine($"Found: {product!.Name} = {product.Price:C}");
else
    Console.WriteLine($"Error [{error!.Code}]: {error.Message}");

// Chain operations
var priceResult = catalogService.GetProduct(2)
    .Map(p => $"Product: {p.Name}, Price: {p.Price:C}");

if (priceResult.IsSuccess)
    Console.WriteLine(priceResult.Value);
```

---

## แบบฝึกหัด Part 10

### แบบฝึกหัด 1: Banking System with Exceptions

```csharp
// สร้าง Banking System ที่มี exception handling ที่สมบูรณ์

public class Bank
{
    private Dictionary<string, BankAccount> _accounts = new();
    
    public BankAccount CreateAccount(string ownerName, decimal initialDeposit)
    {
        // TODO: validate inputs, generate account number, create account
        throw new NotImplementedException();
    }
    
    public void Transfer(string fromAccount, string toAccount, decimal amount)
    {
        // TODO: validate accounts exist, check funds, transfer atomically
        throw new NotImplementedException();
    }
    
    public IEnumerable<string> GetTransactionHistory(string accountNumber)
    {
        // TODO: return history, throw if account not found
        throw new NotImplementedException();
    }
}
```

### เฉลย:

```csharp
public class BankException : AppException
{
    public BankException(string code, string message) : base(code, message) { }
}

public class AccountNotFoundException : BankException
{
    public AccountNotFoundException(string number) 
        : base("ACCOUNT_NOT_FOUND", $"Account {number} not found") { }
}

public class Bank2
{
    private readonly Dictionary<string, BankAccount2> _accounts = new();
    private int _nextId = 1001;
    
    public BankAccount2 CreateAccount(string ownerName, decimal initialDeposit)
    {
        ArgumentException.ThrowIfNullOrEmpty(ownerName, nameof(ownerName));
        if (initialDeposit < 0)
            throw new ArgumentOutOfRangeException(nameof(initialDeposit), 
                "Initial deposit cannot be negative");
        
        string number = $"TH{_nextId++:D6}";
        var account = new BankAccount2(number, ownerName, initialDeposit);
        _accounts[number] = account;
        return account;
    }
    
    public void Transfer(string fromNumber, string toNumber, decimal amount)
    {
        if (!_accounts.TryGetValue(fromNumber, out var from))
            throw new AccountNotFoundException(fromNumber);
        if (!_accounts.TryGetValue(toNumber, out var to))
            throw new AccountNotFoundException(toNumber);
        
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount), "Transfer amount must be positive");
        
        if (from.Balance < amount)
            throw new InsufficientFundsException(from.Balance, amount);
        
        from.Withdraw(amount);
        to.Deposit(amount);
        
        Console.WriteLine($"Transferred {amount:C} from {fromNumber} to {toNumber}");
    }
}

public class BankAccount2
{
    private readonly List<string> _history = new();
    
    public string AccountNumber { get; }
    public string OwnerName { get; }
    public decimal Balance { get; private set; }
    
    public BankAccount2(string number, string ownerName, decimal initialBalance)
    {
        AccountNumber = number;
        OwnerName = ownerName;
        Balance = initialBalance;
        _history.Add($"Account created with balance {initialBalance:C}");
    }
    
    public void Deposit(decimal amount)
    {
        Balance += amount;
        _history.Add($"{DateTime.Now:yyyy-MM-dd HH:mm} Deposit: +{amount:C} | Balance: {Balance:C}");
    }
    
    public void Withdraw(decimal amount)
    {
        if (amount > Balance)
            throw new InsufficientFundsException(Balance, amount);
        Balance -= amount;
        _history.Add($"{DateTime.Now:yyyy-MM-dd HH:mm} Withdrawal: -{amount:C} | Balance: {Balance:C}");
    }
    
    public IEnumerable<string> GetHistory() => _history.ToList();
}
```

---

## สรุป Part 10

ใน Part 10 เราได้เรียนรู้:

1. **Exception Hierarchy** - โครงสร้าง .NET exceptions
2. **try/catch/finally** - พื้นฐาน exception handling
3. **Custom Exceptions** - สร้าง domain exceptions
4. **Exception Filters** - `when` clause
5. **Result Pattern** - Alternative to exceptions
6. **Global Exception Handling** - Unhandled exceptions
7. **Async Exceptions** - AggregateException, Cancellation
8. **Retry Logic** - Resilience patterns
9. **IDisposable** - Resource safety
10. **Best Practices** - สิ่งที่ควรและไม่ควรทำ

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 10 | Steps 91-100*
