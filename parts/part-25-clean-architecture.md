# Part 25: Clean Architecture
## Steps 241-250: Enterprise Application Architecture

---

## Step 241: Clean Architecture Overview

```csharp
// ============================================
// Clean Architecture Layers
// ============================================

/*
 * ┌─────────────────────────────────────────────────────┐
 * │  Presentation Layer (MAUI, XAML, ViewModels)        │
 * │  - UI Pages                                         │
 * │  - ViewModels (MVVM)                                │
 * │  - Converters, Behaviors                            │
 * ├─────────────────────────────────────────────────────┤
 * │  Application Layer                                  │
 * │  - Use Cases / Application Services                 │
 * │  - DTOs / Request / Response models                 │
 * │  - Interfaces (IRepository, IService)               │
 * ├─────────────────────────────────────────────────────┤
 * │  Domain Layer (Core)                                │
 * │  - Entities                                         │
 * │  - Value Objects                                    │
 * │  - Domain Events                                    │
 * │  - Business Rules                                   │
 * ├─────────────────────────────────────────────────────┤
 * │  Infrastructure Layer                               │
 * │  - Repositories (SQLite, HTTP)                      │
 * │  - External Services                                │
 * │  - File System                                      │
 * └─────────────────────────────────────────────────────┘
 *
 * Dependency Rule: Outer layers depend on inner layers.
 * Domain layer has NO dependencies.
 */

// Project structure:
// MyApp.Domain/          - Domain entities, interfaces
// MyApp.Application/     - Use cases, DTOs, services
// MyApp.Infrastructure/  - Repository, API implementations
// MyApp.Presentation/    - MAUI project (Pages, ViewModels)
```

---

## Step 242: Domain Layer

```csharp
// ============================================
// Domain Entities
// ============================================

namespace MyApp.Domain.Entities;

// Base Entity
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;
    
    protected Entity(TId id) => Id = id;
    protected Entity() { }
    
    public override bool Equals(object? obj)
        => obj is Entity<TId> other && Id!.Equals(other.Id);
    
    public override int GetHashCode() => Id!.GetHashCode();
}

// Value Object
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();
    
    public override bool Equals(object? obj)
        => obj is ValueObject other &&
           GetType() == other.GetType() &&
           GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    
    public override int GetHashCode()
        => GetEqualityComponents()
            .Where(c => c != null)
            .Aggregate(0, HashCode.Combine);
}

// ============================================
// Domain Entities
// ============================================

public class Product : Entity<int>
{
    public ProductName Name { get; private set; }
    public Money Price { get; private set; }
    public Quantity Stock { get; private set; }
    public string? Category { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }
    
    private Product() { Name = null!; Price = null!; Stock = null!; }
    
    public static Product Create(string name, decimal price, int stock, string? category = null)
    {
        var product = new Product
        {
            Name = ProductName.Create(name),
            Price = Money.Create(price),
            Stock = Quantity.Create(stock),
            Category = category,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
        return product;
    }
    
    public void UpdatePrice(decimal newPrice)
    {
        Price = Money.Create(newPrice);
    }
    
    public void AddStock(int quantity)
    {
        Stock = Quantity.Create(Stock.Value + quantity);
    }
    
    public bool CanFulfillOrder(int requestedQuantity)
        => IsActive && Stock.Value >= requestedQuantity;
    
    public void Deactivate() => IsActive = false;
}

// Value Objects
public class ProductName : ValueObject
{
    public string Value { get; }
    
    private ProductName(string value) => Value = value;
    
    public static ProductName Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Product name cannot be empty");
        if (value.Length > 200)
            throw new DomainException("Product name too long (max 200 chars)");
        return new ProductName(value.Trim());
    }
    
    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
    
    public override string ToString() => Value;
}

public class Money : ValueObject
{
    public decimal Value { get; }
    public string Currency { get; }
    
    private Money(decimal value, string currency)
    {
        Value = value;
        Currency = currency;
    }
    
    public static Money Create(decimal amount, string currency = "THB")
    {
        if (amount < 0)
            throw new DomainException("Price cannot be negative");
        return new Money(Math.Round(amount, 2), currency);
    }
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Cannot add different currencies");
        return new Money(Value + other.Value, Currency);
    }
    
    public Money Multiply(int quantity)
        => new Money(Value * quantity, Currency);
    
    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
        yield return Currency;
    }
    
    public override string ToString() => $"{Value:N2} {Currency}";
}

public class Quantity : ValueObject
{
    public int Value { get; }
    
    private Quantity(int value) => Value = value;
    
    public static Quantity Create(int value)
    {
        if (value < 0)
            throw new DomainException("Quantity cannot be negative");
        return new Quantity(value);
    }
    
    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}

public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

---

## Step 243: Domain Events

```csharp
// ============================================
// Domain Events
// ============================================

namespace MyApp.Domain.Events;

public abstract class DomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
    public Guid EventId { get; } = Guid.NewGuid();
}

public class ProductCreatedEvent : DomainEvent
{
    public int ProductId { get; }
    public string ProductName { get; }
    
    public ProductCreatedEvent(int productId, string name)
    {
        ProductId = productId;
        ProductName = name;
    }
}

public class StockDepletedEvent : DomainEvent
{
    public int ProductId { get; }
    public string ProductName { get; }
    
    public StockDepletedEvent(int productId, string name)
    {
        ProductId = productId;
        ProductName = name;
    }
}

public class OrderPlacedEvent : DomainEvent
{
    public int OrderId { get; }
    public string CustomerEmail { get; }
    public decimal Total { get; }
    
    public OrderPlacedEvent(int orderId, string email, decimal total)
    {
        OrderId = orderId;
        CustomerEmail = email;
        Total = total;
    }
}

// ============================================
// Domain Event Handler
// ============================================

public interface IDomainEventHandler<TEvent> where TEvent : DomainEvent
{
    Task HandleAsync(TEvent domainEvent, CancellationToken ct = default);
}

public class StockDepletedHandler : IDomainEventHandler<StockDepletedEvent>
{
    private readonly INotificationService _notifications;
    
    public StockDepletedHandler(INotificationService notifications)
        => _notifications = notifications;
    
    public async Task HandleAsync(StockDepletedEvent @event, CancellationToken ct = default)
    {
        await _notifications.SendAsync(
            "สต็อกหมดแล้ว",
            $"สินค้า '{@event.ProductName}' หมดสต็อกแล้ว กรุณาเติมสินค้า",
            ct);
    }
}

public interface INotificationService
{
    Task SendAsync(string title, string message, CancellationToken ct = default);
}
```

---

## Step 244: Application Layer - Use Cases

```csharp
// ============================================
// Application Layer - Use Cases
// ============================================

namespace MyApp.Application.Products;

// Request / Response
public record GetProductsQuery(string? Category = null, int Page = 1, int PageSize = 20);
public record GetProductByIdQuery(int Id);
public record CreateProductCommand(string Name, decimal Price, int Stock, string? Category);
public record UpdatePriceCommand(int ProductId, decimal NewPrice);
public record AddStockCommand(int ProductId, int Quantity);
public record DeleteProductCommand(int Id);

// Use Case Interfaces
public interface IGetProductsUseCase
{
    Task<PaginatedResult<ProductSummaryDto>> ExecuteAsync(
        GetProductsQuery query, CancellationToken ct = default);
}

public interface ICreateProductUseCase
{
    Task<ProductDetailDto> ExecuteAsync(
        CreateProductCommand command, CancellationToken ct = default);
}

// DTOs (Application layer)
public record ProductSummaryDto(
    int Id, string Name, string? Category, 
    decimal Price, int Stock, bool IsActive);

public record ProductDetailDto(
    int Id, string Name, string? Category,
    decimal Price, int Stock, bool IsActive, DateTime CreatedAt);

// ============================================
// Use Case Implementation
// ============================================

public class GetProductsUseCase : IGetProductsUseCase
{
    private readonly IProductRepository2 _productRepository;
    
    public GetProductsUseCase(IProductRepository2 productRepository)
        => _productRepository = productRepository;
    
    public async Task<PaginatedResult<ProductSummaryDto>> ExecuteAsync(
        GetProductsQuery query, CancellationToken ct = default)
    {
        var (products, total) = await _productRepository.GetPagedAsync(
            query.Category, query.Page, query.PageSize, ct);
        
        var dtos = products.Select(p => new ProductSummaryDto(
            p.Id, p.Name.Value, p.Category, p.Price.Value, p.Stock.Value, p.IsActive))
            .ToList();
        
        return new PaginatedResult<ProductSummaryDto>(dtos, total, query.Page, query.PageSize);
    }
}

public class CreateProductUseCase : ICreateProductUseCase
{
    private readonly IProductRepository2 _productRepository;
    private readonly IDomainEventDispatcher _eventDispatcher;
    
    public CreateProductUseCase(IProductRepository2 productRepository,
        IDomainEventDispatcher eventDispatcher)
    {
        _productRepository = productRepository;
        _eventDispatcher = eventDispatcher;
    }
    
    public async Task<ProductDetailDto> ExecuteAsync(
        CreateProductCommand command, CancellationToken ct = default)
    {
        // Validate business rules
        var existing = await _productRepository.FindByNameAsync(command.Name, ct);
        if (existing != null)
            throw new BusinessRuleException($"Product with name '{command.Name}' already exists");
        
        // Create domain entity (with value objects validation)
        var product = Product.Create(command.Name, command.Price, command.Stock, command.Category);
        
        // Persist
        await _productRepository.AddAsync(product, ct);
        
        // Dispatch domain events
        await _eventDispatcher.DispatchAsync(
            new ProductCreatedEvent(product.Id, product.Name.Value), ct);
        
        return new ProductDetailDto(
            product.Id, product.Name.Value, product.Category,
            product.Price.Value, product.Stock.Value, product.IsActive, product.CreatedAt);
    }
}

// Repository Interface (defined in Application layer)
public interface IProductRepository2
{
    Task<(List<Product> Products, int Total)> GetPagedAsync(
        string? category, int page, int size, CancellationToken ct);
    Task<Product?> FindByNameAsync(string name, CancellationToken ct);
    Task AddAsync(Product product, CancellationToken ct);
    Task UpdateAsync(Product product, CancellationToken ct);
    Task DeleteAsync(int id, CancellationToken ct);
}

public interface IDomainEventDispatcher
{
    Task DispatchAsync<TEvent>(TEvent domainEvent, CancellationToken ct = default)
        where TEvent : DomainEvent;
}

public class BusinessRuleException : Exception
{
    public BusinessRuleException(string message) : base(message) { }
}
```

---

## Step 245: Infrastructure Layer

```csharp
// ============================================
// Infrastructure - Repository Implementation
// ============================================

namespace MyApp.Infrastructure.Repositories;

public class SqliteProductRepository : IProductRepository2
{
    private readonly SQLiteAsyncConnection _db;
    
    public SqliteProductRepository(SQLiteAsyncConnection db) => _db = db;
    
    public async Task<(List<Product> Products, int Total)> GetPagedAsync(
        string? category, int page, int size, CancellationToken ct)
    {
        var query = _db.Table<ProductEntity>();
        
        if (!string.IsNullOrEmpty(category))
            query = query.Where(p => p.Category == category);
        
        var total = await query.CountAsync();
        var entities = await query
            .Skip((page - 1) * size)
            .Take(size)
            .ToListAsync();
        
        return (entities.Select(MapToDomain).ToList(), total);
    }
    
    public async Task<Product?> FindByNameAsync(string name, CancellationToken ct)
    {
        var entity = await _db.Table<ProductEntity>()
            .FirstOrDefaultAsync(p => p.Name == name);
        
        return entity == null ? null : MapToDomain(entity);
    }
    
    public async Task AddAsync(Product product, CancellationToken ct)
    {
        var entity = MapToEntity(product);
        await _db.InsertAsync(entity);
        // Set ID back to domain entity using reflection (or separate mechanism)
    }
    
    public async Task UpdateAsync(Product product, CancellationToken ct)
    {
        var entity = MapToEntity(product);
        await _db.UpdateAsync(entity);
    }
    
    public async Task DeleteAsync(int id, CancellationToken ct)
        => await _db.DeleteAsync<ProductEntity>(id);
    
    private static Product MapToDomain(ProductEntity entity)
    {
        var product = Product.Create(
            entity.Name, entity.Price, entity.Stock, entity.Category);
        return product;
    }
    
    private static ProductEntity MapToEntity(Product domain)
        => new()
        {
            Id = domain.Id,
            Name = domain.Name.Value,
            Category = domain.Category,
            Price = domain.Price.Value,
            Stock = domain.Stock.Value,
            IsActive = domain.IsActive,
            CreatedAt = domain.CreatedAt
        };
}
```

---

## Step 246: Dependency Injection Setup

```csharp
// ============================================
// DI Container Setup
// ============================================

public static class DependencyInjection
{
    public static IServiceCollection AddDomain(this IServiceCollection services)
    {
        // Domain services
        return services;
    }
    
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // Use Cases
        services.AddTransient<IGetProductsUseCase, GetProductsUseCase>();
        services.AddTransient<ICreateProductUseCase, CreateProductUseCase>();
        
        // Event Dispatcher
        services.AddSingleton<IDomainEventDispatcher, DomainEventDispatcher>();
        
        return services;
    }
    
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, string dbPath)
    {
        // Database
        services.AddSingleton<SQLiteAsyncConnection>(_ =>
        {
            var db = new SQLiteAsyncConnection(dbPath);
            db.CreateTablesAsync<ProductEntity, OrderEntity, OrderItemEntity>()
              .GetAwaiter().GetResult();
            return db;
        });
        
        // Repositories
        services.AddScoped<IProductRepository2, SqliteProductRepository>();
        
        // External Services
        services.AddHttpClient<IProductApiService, ProductApiService>(client =>
        {
            client.BaseAddress = new Uri("https://api.myapp.com/");
        });
        
        return services;
    }
    
    public static IServiceCollection AddPresentation(this IServiceCollection services)
    {
        // ViewModels
        services.AddTransient<ProductsApiViewModel>();
        services.AddTransient<ProductDetailViewModel2>();
        services.AddTransient<LoginViewModel>();
        services.AddTransient<SettingsViewModel>();
        
        // Navigation
        services.AddSingleton<INavService, NavService>();
        
        // Theme & Localization
        services.AddSingleton<ThemeService>();
        services.AddSingleton<LocalizationService>();
        
        return services;
    }
}

// MauiProgram.cs
/*
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();
        
        var dbPath = Path.Combine(FileSystem.AppDataDirectory, "myapp.db3");
        
        builder.Services
            .AddDomain()
            .AddApplication()
            .AddInfrastructure(dbPath)
            .AddPresentation();
        
        return builder.Build();
    }
}
*/

public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IServiceProvider _sp;
    
    public DomainEventDispatcher(IServiceProvider sp) => _sp = sp;
    
    public async Task DispatchAsync<TEvent>(TEvent domainEvent, CancellationToken ct = default)
        where TEvent : DomainEvent
    {
        var handlers = _sp.GetServices<IDomainEventHandler<TEvent>>();
        
        foreach (var handler in handlers)
            await handler.HandleAsync(domainEvent, ct);
    }
}
```

---

## Step 247: CQRS Pattern

```csharp
// ============================================
// CQRS - Command Query Responsibility Segregation
// ============================================

// Base interfaces
public interface IQuery<TResult> { }
public interface ICommand { }
public interface ICommand<TResult> { }

public interface IQueryHandler<TQuery, TResult>
    where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct = default);
}

public interface ICommandHandler<TCommand>
    where TCommand : ICommand
{
    Task HandleAsync(TCommand command, CancellationToken ct = default);
}

public interface ICommandHandler<TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

// Mediator
public interface IMediator
{
    Task<TResult> QueryAsync<TQuery, TResult>(TQuery query, CancellationToken ct = default)
        where TQuery : IQuery<TResult>;
    Task CommandAsync<TCommand>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand;
    Task<TResult> CommandAsync<TCommand, TResult>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand<TResult>;
}

public class Mediator : IMediator
{
    private readonly IServiceProvider _sp;
    
    public Mediator(IServiceProvider sp) => _sp = sp;
    
    public Task<TResult> QueryAsync<TQuery, TResult>(TQuery query, CancellationToken ct = default)
        where TQuery : IQuery<TResult>
    {
        var handler = _sp.GetRequiredService<IQueryHandler<TQuery, TResult>>();
        return handler.HandleAsync(query, ct);
    }
    
    public Task CommandAsync<TCommand>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand
    {
        var handler = _sp.GetRequiredService<ICommandHandler<TCommand>>();
        return handler.HandleAsync(command, ct);
    }
    
    public Task<TResult> CommandAsync<TCommand, TResult>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand<TResult>
    {
        var handler = _sp.GetRequiredService<ICommandHandler<TCommand, TResult>>();
        return handler.HandleAsync(command, ct);
    }
}

// ============================================
// CQRS Example: Get Products
// ============================================

public record GetProductsQuery2(string? Category, int Page, int PageSize) 
    : IQuery<PaginatedResult<ProductSummaryDto>>;

public class GetProductsQueryHandler 
    : IQueryHandler<GetProductsQuery2, PaginatedResult<ProductSummaryDto>>
{
    private readonly IProductRepository2 _repo;
    
    public GetProductsQueryHandler(IProductRepository2 repo) => _repo = repo;
    
    public async Task<PaginatedResult<ProductSummaryDto>> HandleAsync(
        GetProductsQuery2 query, CancellationToken ct = default)
    {
        var (products, total) = await _repo.GetPagedAsync(
            query.Category, query.Page, query.PageSize, ct);
        
        var dtos = products.Select(p => new ProductSummaryDto(
            p.Id, p.Name.Value, p.Category, p.Price.Value, p.Stock.Value, p.IsActive))
            .ToList();
        
        return new PaginatedResult<ProductSummaryDto>(dtos, total, query.Page, query.PageSize);
    }
}

// In ViewModel:
public partial class ProductsCqrsViewModel : ObservableObject
{
    private readonly IMediator _mediator;
    
    [ObservableProperty]
    private List<ProductSummaryDto> _products = new();
    
    public ProductsCqrsViewModel(IMediator mediator) => _mediator = mediator;
    
    [RelayCommand]
    private async Task LoadProductsAsync()
    {
        var result = await _mediator.QueryAsync<GetProductsQuery2, PaginatedResult<ProductSummaryDto>>(
            new GetProductsQuery2(null, 1, 20));
        Products = result.Items;
    }
}
```

---

## Step 248: Repository Pattern Advanced

```csharp
// ============================================
// Specification Pattern
// ============================================

public abstract class Specification<T>
{
    public abstract bool IsSatisfiedBy(T entity);
    
    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);
    
    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);
    
    public Specification<T> Not()
        => new NotSpecification<T>(this);
}

public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    
    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }
    
    public override bool IsSatisfiedBy(T entity)
        => _left.IsSatisfiedBy(entity) && _right.IsSatisfiedBy(entity);
}

public class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    
    public OrSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }
    
    public override bool IsSatisfiedBy(T entity)
        => _left.IsSatisfiedBy(entity) || _right.IsSatisfiedBy(entity);
}

public class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    
    public NotSpecification(Specification<T> inner) => _inner = inner;
    
    public override bool IsSatisfiedBy(T entity)
        => !_inner.IsSatisfiedBy(entity);
}

// Domain Specifications
public class ActiveProductSpec : Specification<Product>
{
    public override bool IsSatisfiedBy(Product entity)
        => entity.IsActive;
}

public class InStockProductSpec : Specification<Product>
{
    public override bool IsSatisfiedBy(Product entity)
        => entity.Stock.Value > 0;
}

public class AffordableProductSpec : Specification<Product>
{
    private readonly decimal _maxPrice;
    
    public AffordableProductSpec(decimal maxPrice) => _maxPrice = maxPrice;
    
    public override bool IsSatisfiedBy(Product entity)
        => entity.Price.Value <= _maxPrice;
}

// Usage:
// var spec = new ActiveProductSpec().And(new InStockProductSpec()).And(new AffordableProductSpec(5000));
// var products = allProducts.Where(spec.IsSatisfiedBy);
```

---

## Step 249: Result Pattern

```csharp
// ============================================
// Result Pattern - Functional Error Handling
// ============================================

public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }
    public List<string> ValidationErrors { get; } = new();
    
    private Result(bool success, T? value, string? error)
    {
        IsSuccess = success;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => new(true, value, null);
    
    public static Result<T> Failure(string error) => new(false, default, error);
    
    public static Result<T> ValidationFailure(params string[] errors)
    {
        var result = new Result<T>(false, default, "Validation failed");
        result.ValidationErrors.AddRange(errors);
        return result;
    }
    
    public Result<TNew> Map<TNew>(Func<T, TNew> mapper)
        => IsSuccess
            ? Result<TNew>.Success(mapper(Value!))
            : Result<TNew>.Failure(Error!);
    
    public async Task<Result<TNew>> MapAsync<TNew>(Func<T, Task<TNew>> mapper)
        => IsSuccess
            ? Result<TNew>.Success(await mapper(Value!))
            : Result<TNew>.Failure(Error!);
    
    public Result<TNew> Bind<TNew>(Func<T, Result<TNew>> binder)
        => IsSuccess ? binder(Value!) : Result<TNew>.Failure(Error!);
    
    public T GetValueOrDefault(T defaultValue = default!)
        => IsSuccess ? Value! : defaultValue;
    
    public void Match(Action<T> onSuccess, Action<string> onFailure)
    {
        if (IsSuccess) onSuccess(Value!);
        else onFailure(Error!);
    }
    
    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Usage in Application layer
public class ProductUseCase2
{
    private readonly IProductRepository2 _repo;
    
    public ProductUseCase2(IProductRepository2 repo) => _repo = repo;
    
    public async Task<Result<ProductDetailDto>> CreateAsync(CreateProductCommand cmd)
    {
        // Validate
        if (string.IsNullOrWhiteSpace(cmd.Name))
            return Result<ProductDetailDto>.ValidationFailure("Name is required");
        if (cmd.Price < 0)
            return Result<ProductDetailDto>.ValidationFailure("Price must be positive");
        
        try
        {
            var product = Product.Create(cmd.Name, cmd.Price, cmd.Stock, cmd.Category);
            await _repo.AddAsync(product, CancellationToken.None);
            
            return Result<ProductDetailDto>.Success(new ProductDetailDto(
                product.Id, product.Name.Value, product.Category,
                product.Price.Value, product.Stock.Value, product.IsActive, product.CreatedAt));
        }
        catch (DomainException ex)
        {
            return Result<ProductDetailDto>.Failure(ex.Message);
        }
    }
}

// In ViewModel
public partial class AddProductViewModel2 : ObservableObject
{
    private readonly ProductUseCase2 _useCase;
    
    [ObservableProperty]
    private string _name = string.Empty;
    
    [ObservableProperty]
    private decimal _price;
    
    [ObservableProperty]
    private string _errorMessage = string.Empty;
    
    public AddProductViewModel2(ProductUseCase2 useCase) => _useCase = useCase;
    
    [RelayCommand]
    private async Task SaveAsync()
    {
        var result = await _useCase.CreateAsync(
            new CreateProductCommand(Name, Price, 0, null));
        
        result.Match(
            onSuccess: product =>
            {
                // Navigate to product detail
                Shell.Current.GoToAsync($"product-detail?id={product.Id}").FireAndForget();
            },
            onFailure: error =>
            {
                ErrorMessage = error;
            }
        );
    }
}
```

---

## Step 250: Error Handling Strategy

```csharp
// ============================================
// Global Error Handling Strategy
// ============================================

public class ErrorHandlingService
{
    private readonly INavService _nav;
    private readonly ILogger<ErrorHandlingService> _logger;
    
    public ErrorHandlingService(INavService nav, ILogger<ErrorHandlingService> logger)
    {
        _nav = nav;
        _logger = logger;
    }
    
    public async Task HandleAsync(Exception exception)
    {
        _logger.LogError(exception, "Unhandled exception");
        
        var (title, message) = exception switch
        {
            ValidationException ve => ("ข้อมูลไม่ถูกต้อง", ve.Message),
            NotFoundException nfe => ("ไม่พบข้อมูล", nfe.Message),
            BusinessRuleException bre => ("ขัดต่อกฎธุรกิจ", bre.Message),
            HttpRequestException hre => ("เชื่อมต่อไม่ได้", "ไม่สามารถเชื่อมต่อ internet"),
            UnauthorizedAccessException => ("ไม่มีสิทธิ์", "กรุณาเข้าสู่ระบบ"),
            TimeoutException => ("หมดเวลา", "การทำงานใช้เวลานานเกินไป"),
            _ => ("เกิดข้อผิดพลาด", "กรุณาลองใหม่อีกครั้ง")
        };
        
        await _nav.ShowAlertAsync(title, message);
        
        if (exception is UnauthorizedAccessException)
            await _nav.GoToRootAsync("login");
    }
}

// ============================================
// Exception Middleware for ViewModels
// ============================================

public static class ViewModelExtensions
{
    public static async Task<bool> TryExecuteAsync(
        this INotifyPropertyChanged vm,
        Func<Task> operation,
        Action<string> onError,
        Action? onFinally = null)
    {
        try
        {
            await operation();
            return true;
        }
        catch (Exception ex)
        {
            onError(ex.Message);
            return false;
        }
        finally
        {
            onFinally?.Invoke();
        }
    }
}
```

---

## สรุป Part 25

ใน Part 25 เราได้เรียนรู้:

1. **Clean Architecture** - 4 layers, dependency rule
2. **Domain Layer** - Entities, Value Objects, Domain Events
3. **Application Layer** - Use Cases, DTOs, interfaces
4. **Infrastructure Layer** - Repository implementations
5. **Domain Events** - Event-driven architecture
6. **CQRS Pattern** - Commands/Queries separation, Mediator
7. **Specification Pattern** - Composable business rules
8. **Result Pattern** - Functional error handling
9. **DI Setup** - Extension methods สำหรับแต่ละ layer
10. **Error Handling Strategy** - Centralized exception handling

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 25 | Steps 241-250*
