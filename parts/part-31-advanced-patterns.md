# Part 31: Advanced MVVM & Architecture Patterns
## Steps 301-310: Enterprise Architecture

---

## Step 301: ViewModel Composition

```csharp
// ============================================
// Composable ViewModels
// ============================================

// Base functionality as separate "aspects"
public abstract class PageViewModel : ObservableObject
{
    [ObservableProperty] private bool _isBusy;
    [ObservableProperty] private string? _errorMessage;
    [ObservableProperty] private string? _successMessage;
    
    protected async Task ExecuteAsync(Func<Task> operation, string? busyMessage = null)
    {
        if (IsBusy) return;
        
        IsBusy = true;
        ErrorMessage = null;
        SuccessMessage = null;
        
        try
        {
            await operation();
        }
        catch (UnauthorizedAccessException)
        {
            ErrorMessage = "ไม่มีสิทธิ์เข้าถึง";
            await Shell.Current.GoToAsync("//login");
        }
        catch (HttpRequestException ex)
        {
            ErrorMessage = $"ข้อผิดพลาดเครือข่าย: {ex.Message}";
        }
        catch (Exception ex)
        {
            ErrorMessage = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }
}

// Composable search aspect
public partial class SearchAspect<T> : ObservableObject
{
    private readonly Func<string, CancellationToken, Task<IList<T>>> _searchFunc;
    private CancellationTokenSource? _cts;
    
    [ObservableProperty] private string _query = string.Empty;
    [ObservableProperty] private ObservableCollection<T> _results = new();
    [ObservableProperty] private bool _isSearching;
    
    public SearchAspect(Func<string, CancellationToken, Task<IList<T>>> searchFunc)
        => _searchFunc = searchFunc;
    
    partial void OnQueryChanged(string value)
        => _ = SearchDebounced(value);
    
    private async Task SearchDebounced(string query)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        
        try
        {
            await Task.Delay(300, _cts.Token);
            
            if (string.IsNullOrWhiteSpace(query)) { Results.Clear(); return; }
            
            IsSearching = true;
            var results = await _searchFunc(query, _cts.Token);
            Results = new ObservableCollection<T>(results);
        }
        catch (OperationCanceledException) { }
        finally { IsSearching = false; }
    }
}

// Compose them:
public partial class ProductListViewModel : PageViewModel
{
    private readonly IProductApiService _api;
    
    public SearchAspect<Product> Search { get; }
    public PaginationAspect<Product> Pagination { get; }
    
    public ProductListViewModel(IProductApiService api)
    {
        _api = api;
        Search = new SearchAspect<Product>(async (q, ct) => await _api.SearchAsync(q));
        Pagination = new PaginationAspect<Product>(async (page, size) => 
            await _api.GetPagedAsync(page, size));
    }
}
```

---

## Step 302: Repository with Caching

```csharp
// ============================================
// Cached Repository Pattern
// ============================================

public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly MemoryCache<int, Product> _cache;
    private readonly MemoryCache<string, List<Product>> _listCache;
    
    public CachedProductRepository(IProductRepository inner)
    {
        _inner = inner;
        _cache = new MemoryCache<int, Product>();
        _listCache = new MemoryCache<string, List<Product>>();
    }
    
    public async Task<Product?> GetByIdAsync(int id)
    {
        return await _cache.GetOrSetAsync(id,
            () => _inner.GetByIdAsync(id)!,
            TimeSpan.FromMinutes(5));
    }
    
    public async Task<List<Product>> GetAllAsync()
    {
        return await _listCache.GetOrSetAsync("all",
            () => _inner.GetAllAsync(),
            TimeSpan.FromMinutes(1));
    }
    
    public async Task<Product> CreateAsync(Product product)
    {
        var result = await _inner.CreateAsync(product);
        _listCache.Invalidate("all");
        return result;
    }
    
    public async Task UpdateAsync(Product product)
    {
        await _inner.UpdateAsync(product);
        _cache.Invalidate(product.Id);
        _listCache.Invalidate("all");
    }
    
    public async Task DeleteAsync(int id)
    {
        await _inner.DeleteAsync(id);
        _cache.Invalidate(id);
        _listCache.Invalidate("all");
    }
    
    public async Task<List<Product>> SearchAsync(string query)
    {
        var cacheKey = $"search:{query.ToLower()}";
        return await _listCache.GetOrSetAsync(cacheKey,
            () => _inner.SearchAsync(query),
            TimeSpan.FromSeconds(30));
    }
}
```

---

## Step 303: Event-Driven Architecture

```csharp
// ============================================
// Domain Events via MessagingCenter
// ============================================

// Domain Events
public record ProductCreatedEvent(Product Product);
public record ProductUpdatedEvent(int ProductId, Product Product);
public record ProductDeletedEvent(int ProductId);
public record CartUpdatedEvent(int ItemCount, decimal Total);
public record UserLoggedInEvent(string UserId, string Email);
public record UserLoggedOutEvent(string UserId);

// Event bus
public interface IEventBus
{
    void Publish<TEvent>(TEvent @event) where TEvent : class;
    void Subscribe<TEvent>(object subscriber, Action<TEvent> handler) where TEvent : class;
    void Unsubscribe<TEvent>(object subscriber) where TEvent : class;
}

public class WeakEventBus : IEventBus
{
    public void Publish<TEvent>(TEvent @event) where TEvent : class
        => WeakReferenceMessenger.Default.Send(new EventMessage<TEvent>(@event));
    
    public void Subscribe<TEvent>(object subscriber, Action<TEvent> handler) where TEvent : class
        => WeakReferenceMessenger.Default.Register<EventMessage<TEvent>>(
            subscriber, (_, msg) => handler(msg.Event));
    
    public void Unsubscribe<TEvent>(object subscriber) where TEvent : class
        => WeakReferenceMessenger.Default.Unregister<EventMessage<TEvent>>(subscriber);
}

public record EventMessage<T>(T Event);

// Usage:
public partial class CartViewModel : ObservableObject
{
    private readonly IEventBus _bus;
    
    [ObservableProperty] private int _cartItemCount;
    [ObservableProperty] private decimal _cartTotal;
    
    public CartViewModel(IEventBus bus)
    {
        _bus = bus;
        _bus.Subscribe<CartUpdatedEvent>(this, OnCartUpdated);
        _bus.Subscribe<UserLoggedOutEvent>(this, _ => ClearCart());
    }
    
    private void OnCartUpdated(CartUpdatedEvent @event)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            CartItemCount = @event.ItemCount;
            CartTotal = @event.Total;
        });
    }
    
    private void ClearCart()
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            CartItemCount = 0;
            CartTotal = 0;
        });
    }
    
    public void Dispose()
    {
        _bus.Unsubscribe<CartUpdatedEvent>(this);
        _bus.Unsubscribe<UserLoggedOutEvent>(this);
    }
}
```

---

## Step 304: Mediator Pattern

```csharp
// ============================================
// Mediator (CQRS Commands/Queries)
// ============================================

public interface IRequest<TResponse> { }
public interface ICommand : IRequest<Unit> { }
public interface IQuery<TResponse> : IRequest<TResponse> { }

public record Unit;

public interface IRequestHandler<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct);
}

public class Mediator
{
    private readonly IServiceProvider _services;
    
    public Mediator(IServiceProvider services) => _services = services;
    
    public async Task<TResponse> SendAsync<TResponse>(
        IRequest<TResponse> request, CancellationToken ct = default)
    {
        var handlerType = typeof(IRequestHandler<,>)
            .MakeGenericType(request.GetType(), typeof(TResponse));
        
        var handler = _services.GetRequiredService(handlerType);
        
        var method = handlerType.GetMethod("HandleAsync")!;
        var result = await (Task<TResponse>)method.Invoke(handler, [request, ct])!;
        
        return result;
    }
}

// Commands
public record CreateProductCommand(
    string Name, 
    decimal Price, 
    string Category) : ICommand;

public record UpdateProductCommand(
    int Id, 
    string Name, 
    decimal Price) : ICommand;

public record DeleteProductCommand(int Id) : ICommand;

// Queries
public record GetProductQuery(int Id) : IQuery<Product?>;
public record GetProductsQuery(string? Search = null, int Page = 1, int PageSize = 20) 
    : IQuery<PagedResult<Product>>;

// Handlers
public class CreateProductHandler : IRequestHandler<CreateProductCommand, Unit>
{
    private readonly IProductRepository _repo;
    private readonly IEventBus _bus;
    
    public CreateProductHandler(IProductRepository repo, IEventBus bus)
    {
        _repo = repo;
        _bus = bus;
    }
    
    public async Task<Unit> HandleAsync(CreateProductCommand cmd, CancellationToken ct)
    {
        var product = new Product
        {
            Name = cmd.Name,
            Price = cmd.Price,
            Category = cmd.Category
        };
        
        var created = await _repo.CreateAsync(product);
        _bus.Publish(new ProductCreatedEvent(created));
        
        return new Unit();
    }
}

// ViewModel using Mediator
public partial class CreateProductViewModel : PageViewModel
{
    private readonly Mediator _mediator;
    
    [ObservableProperty] private string _name = string.Empty;
    [ObservableProperty] private string _price = string.Empty;
    [ObservableProperty] private string _category = string.Empty;
    
    public CreateProductViewModel(Mediator mediator) => _mediator = mediator;
    
    [RelayCommand]
    private async Task SaveAsync()
    {
        if (!decimal.TryParse(Price, out var price)) { ErrorMessage = "ราคาไม่ถูกต้อง"; return; }
        
        await ExecuteAsync(async () =>
        {
            await _mediator.SendAsync(new CreateProductCommand(Name, price, Category));
            SuccessMessage = "บันทึกสำเร็จ";
            await Shell.Current.GoToAsync("..");
        });
    }
}
```

---

## Step 305: Validation Framework

```csharp
// ============================================
// Fluent Validation
// ============================================

public interface IValidator<T>
{
    ValidationResult Validate(T value);
}

public class ValidationResult
{
    private readonly List<string> _errors = new();
    
    public bool IsValid => _errors.Count == 0;
    public IReadOnlyList<string> Errors => _errors.AsReadOnly();
    public string FirstError => _errors.FirstOrDefault() ?? string.Empty;
    
    public void AddError(string error) => _errors.Add(error);
}

public class ProductValidator : IValidator<ProductFormModel>
{
    public ValidationResult Validate(ProductFormModel model)
    {
        var result = new ValidationResult();
        
        if (string.IsNullOrWhiteSpace(model.Name))
            result.AddError("กรุณากรอกชื่อสินค้า");
        else if (model.Name.Length < 3)
            result.AddError("ชื่อสินค้าต้องมีอย่างน้อย 3 ตัวอักษร");
        else if (model.Name.Length > 100)
            result.AddError("ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร");
        
        if (model.Price <= 0)
            result.AddError("ราคาต้องมากกว่า 0");
        else if (model.Price > 1_000_000)
            result.AddError("ราคาต้องไม่เกิน 1,000,000 บาท");
        
        if (string.IsNullOrWhiteSpace(model.Category))
            result.AddError("กรุณาเลือกหมวดหมู่");
        
        if (model.StockQuantity < 0)
            result.AddError("จำนวนสต็อกต้องไม่น้อยกว่า 0");
        
        return result;
    }
}

public class ProductFormModel : ObservableValidator
{
    private string _name = string.Empty;
    private decimal _price;
    private string _category = string.Empty;
    private int _stockQuantity;
    
    [Required(ErrorMessage = "กรุณากรอกชื่อสินค้า")]
    [MinLength(3, ErrorMessage = "ชื่อสินค้าต้องมีอย่างน้อย 3 ตัวอักษร")]
    [MaxLength(100, ErrorMessage = "ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร")]
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value, true);
    }
    
    [Range(0.01, 1_000_000, ErrorMessage = "ราคาต้องอยู่ระหว่าง 0.01 - 1,000,000")]
    public decimal Price
    {
        get => _price;
        set => SetProperty(ref _price, value, true);
    }
    
    [Required(ErrorMessage = "กรุณาเลือกหมวดหมู่")]
    public string Category
    {
        get => _category;
        set => SetProperty(ref _category, value, true);
    }
    
    [Range(0, int.MaxValue, ErrorMessage = "จำนวนต้องไม่น้อยกว่า 0")]
    public int StockQuantity
    {
        get => _stockQuantity;
        set => SetProperty(ref _stockQuantity, value, true);
    }
    
    public bool TryValidate(out string[] errors)
    {
        ValidateAllProperties();
        errors = GetErrors().Select(e => e.ErrorMessage ?? string.Empty).ToArray();
        return !HasErrors;
    }
}
```

---

## Step 306: State Machine

```csharp
// ============================================
// Order State Machine
// ============================================

public enum OrderState
{
    Draft,
    Submitted,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

public enum OrderTrigger
{
    Submit,
    Confirm,
    StartProcessing,
    Ship,
    Deliver,
    Cancel,
    Refund
}

public class OrderStateMachine
{
    private readonly Dictionary<(OrderState, OrderTrigger), OrderState> _transitions = new()
    {
        [(OrderState.Draft, OrderTrigger.Submit)] = OrderState.Submitted,
        [(OrderState.Draft, OrderTrigger.Cancel)] = OrderState.Cancelled,
        [(OrderState.Submitted, OrderTrigger.Confirm)] = OrderState.Confirmed,
        [(OrderState.Submitted, OrderTrigger.Cancel)] = OrderState.Cancelled,
        [(OrderState.Confirmed, OrderTrigger.StartProcessing)] = OrderState.Processing,
        [(OrderState.Confirmed, OrderTrigger.Cancel)] = OrderState.Cancelled,
        [(OrderState.Processing, OrderTrigger.Ship)] = OrderState.Shipped,
        [(OrderState.Shipped, OrderTrigger.Deliver)] = OrderState.Delivered,
        [(OrderState.Delivered, OrderTrigger.Refund)] = OrderState.Refunded,
    };
    
    private readonly Dictionary<OrderTrigger, Func<Order, Task>> _actions = new();
    
    public OrderState CurrentState { get; private set; }
    
    public OrderStateMachine(OrderState initialState) => CurrentState = initialState;
    
    public void Configure(OrderTrigger trigger, Func<Order, Task> action)
        => _actions[trigger] = action;
    
    public async Task<bool> FireAsync(Order order, OrderTrigger trigger)
    {
        if (!_transitions.TryGetValue((CurrentState, trigger), out var nextState))
            return false;
        
        if (_actions.TryGetValue(trigger, out var action))
            await action(order);
        
        CurrentState = nextState;
        return true;
    }
    
    public IReadOnlyList<OrderTrigger> GetAvailableTriggers()
        => _transitions.Keys
            .Where(k => k.Item1 == CurrentState)
            .Select(k => k.Item2)
            .ToList()
            .AsReadOnly();
    
    public bool CanFire(OrderTrigger trigger)
        => _transitions.ContainsKey((CurrentState, trigger));
}

// Usage
public class OrderService
{
    public OrderStateMachine CreateStateMachine(OrderState state)
    {
        var machine = new OrderStateMachine(state);
        
        machine.Configure(OrderTrigger.Submit, async order =>
        {
            order.SubmittedAt = DateTime.UtcNow;
            await SendOrderConfirmationEmailAsync(order);
        });
        
        machine.Configure(OrderTrigger.Ship, async order =>
        {
            order.ShippedAt = DateTime.UtcNow;
            await SendShippingNotificationAsync(order);
        });
        
        machine.Configure(OrderTrigger.Deliver, async order =>
        {
            order.DeliveredAt = DateTime.UtcNow;
            await SendDeliveryConfirmationAsync(order);
        });
        
        return machine;
    }
    
    private Task SendOrderConfirmationEmailAsync(Order order) => Task.CompletedTask;
    private Task SendShippingNotificationAsync(Order order) => Task.CompletedTask;
    private Task SendDeliveryConfirmationAsync(Order order) => Task.CompletedTask;
}
```

---

## Step 307: Pipeline Pattern

```csharp
// ============================================
// Request Pipeline (Middleware)
// ============================================

public interface IPipelineBehavior<TRequest, TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, Func<Task<TResponse>> next, CancellationToken ct);
}

// Logging behavior
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;
    
    public async Task<TResponse> HandleAsync(
        TRequest request, Func<Task<TResponse>> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {Request}", requestName);
        
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var response = await next();
            _logger.LogInformation("{Request} handled in {Ms}ms", requestName, sw.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "{Request} failed after {Ms}ms", requestName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Validation behavior
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;
    
    public async Task<TResponse> HandleAsync(
        TRequest request, Func<Task<TResponse>> next, CancellationToken ct)
    {
        foreach (var validator in _validators)
        {
            var result = validator.Validate(request);
            if (!result.IsValid)
                throw new ValidationException(result.Errors);
        }
        
        return await next();
    }
}

public class ValidationException : Exception
{
    public IReadOnlyList<string> Errors { get; }
    
    public ValidationException(IReadOnlyList<string> errors) 
        : base(string.Join(", ", errors))
        => Errors = errors;
}
```

---

## Step 308: Result Pattern

```csharp
// ============================================
// Comprehensive Result<T>
// ============================================

public abstract record Result<T>
{
    public bool IsSuccess => this is Success<T>;
    public bool IsFailure => !IsSuccess;
    
    public static Result<T> Ok(T value) => new Success<T>(value);
    public static Result<T> Fail(string error, ErrorCode code = ErrorCode.General) 
        => new Failure<T>(error, code);
    public static Result<T> NotFound(string message = "ไม่พบข้อมูล")
        => new Failure<T>(message, ErrorCode.NotFound);
    
    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<string, TOut> onFailure) 
        => this switch
        {
            Success<T> s => onSuccess(s.Value),
            Failure<T> f => onFailure(f.Error),
            _ => throw new InvalidOperationException()
        };
    
    public Result<TOut> Map<TOut>(Func<T, TOut> mapper) 
        => this switch
        {
            Success<T> s => Result<TOut>.Ok(mapper(s.Value)),
            Failure<T> f => Result<TOut>.Fail(f.Error, f.Code),
            _ => throw new InvalidOperationException()
        };
    
    public async Task<Result<TOut>> BindAsync<TOut>(Func<T, Task<Result<TOut>>> binder)
        => this switch
        {
            Success<T> s => await binder(s.Value),
            Failure<T> f => Result<TOut>.Fail(f.Error, f.Code),
            _ => throw new InvalidOperationException()
        };
    
    public T GetValueOrDefault(T defaultValue = default!)
        => this is Success<T> s ? s.Value : defaultValue;
}

public sealed record Success<T>(T Value) : Result<T>;
public sealed record Failure<T>(string Error, ErrorCode Code = ErrorCode.General) : Result<T>;

public enum ErrorCode
{
    General,
    NotFound,
    Unauthorized,
    Forbidden,
    Conflict,
    Validation,
    Network
}

// Service using Result<T>
public class ProductService
{
    private readonly IProductRepository _repo;
    
    public ProductService(IProductRepository repo) => _repo = repo;
    
    public async Task<Result<Product>> GetProductAsync(int id)
    {
        var product = await _repo.GetByIdAsync(id);
        return product != null
            ? Result<Product>.Ok(product)
            : Result<Product>.NotFound($"ไม่พบสินค้า ID: {id}");
    }
    
    public async Task<Result<Product>> UpdatePriceAsync(int id, decimal newPrice)
    {
        if (newPrice <= 0)
            return Result<Product>.Fail("ราคาต้องมากกว่า 0", ErrorCode.Validation);
        
        return await GetProductAsync(id)
            .BindAsync(async product =>
            {
                product.Price = newPrice;
                await _repo.UpdateAsync(product);
                return Result<Product>.Ok(product);
            });
    }
}

// ViewModel using Result<T>
public partial class ProductDetailViewModel : ObservableObject
{
    private readonly ProductService _service;
    
    [ObservableProperty] private Product? _product;
    [ObservableProperty] private string? _errorMessage;
    
    public ProductDetailViewModel(ProductService service) => _service = service;
    
    [RelayCommand]
    private async Task LoadAsync(int id)
    {
        var result = await _service.GetProductAsync(id);
        
        result.Match(
            onSuccess: p => { Product = p; ErrorMessage = null; },
            onFailure: err => { ErrorMessage = err; Product = null; });
    }
}
```

---

## Step 309: Feature Flags

```csharp
// ============================================
// Feature Flag System
// ============================================

public interface IFeatureFlags
{
    bool IsEnabled(string feature);
    Task<bool> IsEnabledAsync(string feature);
}

public class LocalFeatureFlags : IFeatureFlags
{
    private readonly Dictionary<string, bool> _flags;
    
    public LocalFeatureFlags()
    {
        _flags = new Dictionary<string, bool>
        {
            ["new-checkout-flow"] = false,
            ["ai-recommendations"] = true,
            ["dark-mode-v2"] = true,
            ["push-notifications"] = true,
            ["biometric-login"] = false,
            ["offline-mode"] = true
        };
        
        // Override from app settings
        foreach (var flag in _flags.Keys.ToList())
        {
            if (Preferences.Default.ContainsKey($"ff_{flag}"))
                _flags[flag] = Preferences.Default.Get($"ff_{flag}", _flags[flag]);
        }
    }
    
    public bool IsEnabled(string feature)
        => _flags.TryGetValue(feature, out var enabled) && enabled;
    
    public Task<bool> IsEnabledAsync(string feature)
        => Task.FromResult(IsEnabled(feature));
}

// Usage
public partial class CheckoutViewModel : ObservableObject
{
    private readonly IFeatureFlags _flags;
    
    [ObservableProperty] private bool _useNewCheckoutFlow;
    
    public CheckoutViewModel(IFeatureFlags flags)
    {
        _flags = flags;
        UseNewCheckoutFlow = _flags.IsEnabled("new-checkout-flow");
    }
}
```

---

## Step 310: Dependency Injection Advanced

```csharp
// ============================================
// Advanced DI Configuration
// ============================================

public static class ServiceConfiguration
{
    public static MauiAppBuilder ConfigureServices(this MauiAppBuilder builder)
    {
        var services = builder.Services;
        
        // Feature flags
        services.AddSingleton<IFeatureFlags, LocalFeatureFlags>();
        
        // Event bus
        services.AddSingleton<IEventBus, WeakEventBus>();
        
        // Repositories with caching decorator
        services.AddScoped<SqliteProductRepository>();
        services.AddScoped<IProductRepository>(sp =>
        {
            var inner = sp.GetRequiredService<SqliteProductRepository>();
            return new CachedProductRepository(inner);
        });
        
        // Mediator
        services.AddSingleton<Mediator>();
        services.AddScoped<IRequestHandler<CreateProductCommand, Unit>, CreateProductHandler>();
        services.AddScoped<IRequestHandler<GetProductQuery, Product?>, GetProductQueryHandler>();
        
        // ViewModels (scoped - new per navigation)
        services.AddTransient<ProductListViewModel>();
        services.AddTransient<ProductDetailViewModel>();
        services.AddTransient<CreateProductViewModel>();
        services.AddTransient<CartViewModel>();
        
        // Services
        services.AddSingleton<ProductService>();
        services.AddSingleton<AuthorizationService>();
        services.AddSingleton<AppTaskScheduler>();
        
        return builder;
    }
}

// Lazy resolution to avoid circular dependencies
public class LazyDependency<T>
{
    private readonly Lazy<T> _lazy;
    public T Value => _lazy.Value;
    
    public LazyDependency(IServiceProvider sp) 
        => _lazy = new Lazy<T>(() => sp.GetRequiredService<T>());
}
```

---

## สรุป Part 31

ใน Part 31 เราได้เรียนรู้:

1. **ViewModel Composition** - Composable aspects
2. **Cached Repository** - Decorator pattern
3. **Event-Driven** - Domain events + WeakEventBus
4. **Mediator/CQRS** - Commands/Queries separation
5. **Validation Framework** - FluentValidation approach
6. **State Machine** - Order state transitions
7. **Pipeline Pattern** - Request middleware
8. **Result<T>** - Functional error handling
9. **Feature Flags** - Toggle features
10. **Advanced DI** - Decorators, Mediator wiring

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 31 | Steps 301-310*
