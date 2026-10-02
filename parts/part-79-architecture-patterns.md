# Part 79: Advanced Architecture Patterns
## Steps 781-790: Clean Architecture, DDD Aggregates, Functional Patterns, Reactive

---

## Step 781: Clean Architecture Layers

```csharp
// ============================================
// Clean Architecture Structure
// ============================================

// Domain Layer (no dependencies)
namespace MyApp.Domain.Entities
{
    public class Order : IAggregateRoot
    {
        private readonly List<OrderLine> _lines = new();
        private readonly List<IDomainEvent> _events = new();
        
        public OrderId Id { get; private init; }
        public CustomerId CustomerId { get; private init; }
        public OrderStatus Status { get; private set; }
        public Money Total { get; private set; }
        public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
        
        private Order() { }  // For ORM
        
        public static Order Create(CustomerId customerId, IReadOnlyList<OrderLine> lines)
        {
            if (!lines.Any())
                throw new DomainException("Order must have at least one line");
            
            var order = new Order
            {
                Id = new OrderId(Guid.NewGuid()),
                CustomerId = customerId,
                Status = OrderStatus.Pending,
            };
            
            foreach (var line in lines) order._lines.Add(line);
            order.Total = order.CalculateTotal();
            
            order._events.Add(new OrderCreated(order.Id, customerId, order.Total));
            return order;
        }
        
        public void Confirm()
        {
            if (Status != OrderStatus.Pending)
                throw new DomainException($"Cannot confirm order in {Status} state");
            
            Status = OrderStatus.Confirmed;
            _events.Add(new OrderConfirmed(Id));
        }
        
        public IReadOnlyList<IDomainEvent> GetDomainEvents() => _events.AsReadOnly();
        public void ClearDomainEvents() => _events.Clear();
        
        private Money CalculateTotal()
            => _lines.Aggregate(Money.Zero, (acc, line) => acc + line.Subtotal);
    }
    
    // Value objects
    public record OrderId(Guid Value);
    public record CustomerId(Guid Value);
    public record Money(decimal Amount, string Currency)
    {
        public static readonly Money Zero = new(0, "THB");
        public static Money operator +(Money a, Money b)
            => a with { Amount = a.Amount + b.Amount };
    }
    
    public record OrderLine(string MenuItemId, string Name, Money Price, int Qty)
    {
        public Money Subtotal => Price with { Amount = Price.Amount * Qty };
    }
    
    public enum OrderStatus { Pending, Confirmed, Preparing, ReadyForPickup, Delivered, Cancelled }
}

// Application Layer (depends on Domain only)
namespace MyApp.Application.Commands
{
    public record PlaceOrderCommand(
        Guid CustomerId,
        List<(string MenuItemId, int Qty)> Items) : ICommand;
    
    public class PlaceOrderHandler : ICommandHandler<PlaceOrderCommand>
    {
        private readonly IOrderRepository _orders;
        private readonly IMenuRepository _menu;
        private readonly IEventBus _bus;
        
        public PlaceOrderHandler(IOrderRepository orders,
            IMenuRepository menu, IEventBus bus)
        {
            _orders = orders; _menu = menu; _bus = bus;
        }
        
        public async Task HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
        {
            var menuItems = await _menu.GetByIdsAsync(
                cmd.Items.Select(i => i.MenuItemId).ToList(), ct);
            
            var lines = cmd.Items.Select(item =>
            {
                var mi = menuItems.First(m => m.Id == item.MenuItemId);
                return new OrderLine(mi.Id, mi.Name,
                    new Money(mi.Price, "THB"), item.Qty);
            }).ToList();
            
            var order = Order.Create(new CustomerId(cmd.CustomerId), lines);
            
            await _orders.SaveAsync(order, ct);
            
            foreach (var evt in order.GetDomainEvents())
                await _bus.PublishAsync(evt, ct);
            
            order.ClearDomainEvents();
        }
    }
}
```

---

## Step 782: Repository with Unit of Work

```csharp
// ============================================
// Unit of Work Pattern
// ============================================

public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    Task<int> CommitAsync(CancellationToken ct = default);
}

public class SqliteUnitOfWork : IUnitOfWork
{
    private readonly SQLiteAsyncConnection _db;
    private readonly List<object> _pending = new();
    
    public IOrderRepository Orders { get; }
    public ICustomerRepository Customers { get; }
    
    public SqliteUnitOfWork(SQLiteAsyncConnection db)
    {
        _db = db;
        Orders = new SqliteOrderRepository(db, _pending);
        Customers = new SqliteCustomerRepository(db, _pending);
    }
    
    public async Task<int> CommitAsync(CancellationToken ct = default)
    {
        if (_pending.Count == 0) return 0;
        
        var count = 0;
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var entity in _pending)
            {
                conn.InsertOrReplace(entity);
                count++;
            }
        });
        
        _pending.Clear();
        return count;
    }
    
    public void Dispose() => _pending.Clear();
}
```

---

## Step 783: Specification Pattern

```csharp
// ============================================
// Specification Pattern for Business Rules
// ============================================

public abstract class Specification<T>
{
    public abstract bool IsSatisfiedBy(T candidate);
    
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
    { _left = left; _right = right; }
    public override bool IsSatisfiedBy(T c) => _left.IsSatisfiedBy(c) && _right.IsSatisfiedBy(c);
}

public class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public OrSpecification(Specification<T> left, Specification<T> right)
    { _left = left; _right = right; }
    public override bool IsSatisfiedBy(T c) => _left.IsSatisfiedBy(c) || _right.IsSatisfiedBy(c);
}

public class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpecification(Specification<T> inner) { _inner = inner; }
    public override bool IsSatisfiedBy(T c) => !_inner.IsSatisfiedBy(c);
}

// Domain specifications
public class MinimumOrderAmountSpec : Specification<Order>
{
    private readonly decimal _minimum;
    public MinimumOrderAmountSpec(decimal minimum) { _minimum = minimum; }
    
    public override bool IsSatisfiedBy(Order order)
        => order.Total.Amount >= _minimum;
}

public class DeliveryRadiusSpec : Specification<Order>
{
    private readonly double _maxKm;
    private readonly Location _restaurantLocation;
    
    public DeliveryRadiusSpec(Location restaurant, double maxKm)
    { _restaurantLocation = restaurant; _maxKm = maxKm; }
    
    public override bool IsSatisfiedBy(Order order)
        => true; // Simplified; would check delivery address distance
}

// Usage
var canDeliver = new MinimumOrderAmountSpec(100)
    .And(new DeliveryRadiusSpec(restaurantLoc, 5.0));

if (!canDeliver.IsSatisfiedBy(order))
    throw new BusinessRuleException("ออเดอร์ไม่ตรงตามเงื่อนไขการจัดส่ง");
```

---

## Step 784: Result Type (Railway-Oriented)

```csharp
// ============================================
// Result<T, E> for Railway-Oriented Programming
// ============================================

public readonly struct Result<T, TError>
{
    private readonly T? _value;
    private readonly TError? _error;
    private readonly bool _isSuccess;
    
    private Result(T value) { _value = value; _isSuccess = true; _error = default; }
    private Result(TError error) { _error = error; _isSuccess = false; _value = default; }
    
    public static Result<T, TError> Ok(T value) => new(value);
    public static Result<T, TError> Fail(TError error) => new(error);
    
    public bool IsSuccess => _isSuccess;
    public T Value => _isSuccess ? _value! : throw new InvalidOperationException("Result is failure");
    public TError Error => !_isSuccess ? _error! : throw new InvalidOperationException("Result is success");
    
    public Result<TNew, TError> Map<TNew>(Func<T, TNew> mapper)
        => _isSuccess ? Result<TNew, TError>.Ok(mapper(_value!))
                      : Result<TNew, TError>.Fail(_error!);
    
    public Result<T, TNewError> MapError<TNewError>(Func<TError, TNewError> mapper)
        => _isSuccess ? Result<T, TNewError>.Ok(_value!)
                      : Result<T, TNewError>.Fail(mapper(_error!));
    
    public Result<TNew, TError> Bind<TNew>(Func<T, Result<TNew, TError>> binder)
        => _isSuccess ? binder(_value!) : Result<TNew, TError>.Fail(_error!);
    
    public T Unwrap(T defaultValue = default!) => _isSuccess ? _value! : defaultValue;
    
    public void Match(Action<T> onSuccess, Action<TError> onFailure)
    {
        if (_isSuccess) onSuccess(_value!);
        else onFailure(_error!);
    }
    
    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<TError, TResult> onFailure)
        => _isSuccess ? onSuccess(_value!) : onFailure(_error!);
}

// Domain error types
public abstract record DomainError(string Message);
public record ValidationError(string Field, string Message) : DomainError(Message);
public record NotFoundError(string Entity, string Id) : DomainError($"{Entity} {Id} not found");
public record BusinessRuleError(string Rule) : DomainError(Rule);

// Usage in service
public class OrderService
{
    public Result<Order, DomainError> CreateOrder(
        Guid customerId, List<(string ItemId, int Qty)> items)
    {
        if (!items.Any())
            return Result<Order, DomainError>.Fail(
                new ValidationError("Items", "Order must have items"));
        
        if (items.Any(i => i.Qty <= 0))
            return Result<Order, DomainError>.Fail(
                new ValidationError("Quantity", "Quantity must be positive"));
        
        var order = Order.Create(new CustomerId(customerId),
            items.Select(i => new OrderLine(i.ItemId, "", Money.Zero, i.Qty)).ToList());
        
        return Result<Order, DomainError>.Ok(order);
    }
}

// Usage in ViewModel
var result = _orderService.CreateOrder(userId, cartItems);
result.Match(
    onSuccess: order => NavigateToConfirmation(order),
    onFailure: error => ShowError(error.Message));
```

---

## Step 785: Reactive Extensions (Rx.NET)

```csharp
// ============================================
// Reactive Programming with System.Reactive
// ============================================

// Install: System.Reactive

public class ReactiveOrderService : IDisposable
{
    private readonly Subject<Order> _orderStream = new();
    private readonly CompositeDisposable _disposables = new();
    
    public IObservable<Order> OrderStream => _orderStream.AsObservable();
    
    public void Start(IOrderRepository repo, IConnectivity connectivity)
    {
        // Poll for new orders every 30s when online
        var polling = Observable
            .Interval(TimeSpan.FromSeconds(30))
            .Where(_ => connectivity.NetworkAccess == NetworkAccess.Internet)
            .SelectMany(_ => Observable.FromAsync(repo.GetPendingOrdersAsync))
            .SelectMany(orders => orders)
            .Subscribe(order => _orderStream.OnNext(order));
        
        _disposables.Add(polling);
        
        // Alert on high-value orders
        var highValueAlert = _orderStream
            .Where(o => o.Total.Amount > 1000)
            .Throttle(TimeSpan.FromSeconds(1))
            .ObserveOn(SynchronizationContext.Current!)
            .Subscribe(order =>
                WeakReferenceMessenger.Default.Send(
                    new HighValueOrderAlert(order)));
        
        _disposables.Add(highValueAlert);
    }
    
    public void Dispose()
    {
        _disposables.Dispose();
        _orderStream.Dispose();
    }
}

// Search with debounce using Rx
public class ReactiveSearchService
{
    private readonly Subject<string> _searchQuery = new();
    private IDisposable? _sub;
    
    public void Init(IMenuRepository repo, Action<IReadOnlyList<MenuItem>> onResults)
    {
        _sub = _searchQuery
            .Where(q => q.Length >= 2)
            .Throttle(TimeSpan.FromMilliseconds(300))
            .DistinctUntilChanged()
            .SelectMany(q => Observable.FromAsync(() => repo.SearchAsync(q)))
            .ObserveOn(SynchronizationContext.Current!)
            .Subscribe(onResults);
    }
    
    public void Search(string query) => _searchQuery.OnNext(query);
    public void Dispose() => _sub?.Dispose();
}

public record HighValueOrderAlert(Order Order);
```

---

## Step 786: CQRS Command Bus

```csharp
// ============================================
// CQRS Command/Query Bus with Pipeline
// ============================================

public interface ICommandBus
{
    Task SendAsync<TCommand>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand;
}

public interface IQueryBus
{
    Task<TResult> QueryAsync<TQuery, TResult>(TQuery query, CancellationToken ct = default)
        where TQuery : IQuery<TResult>;
}

public class CommandBus : ICommandBus
{
    private readonly IServiceProvider _sp;
    
    public CommandBus(IServiceProvider sp) => _sp = sp;
    
    public async Task SendAsync<TCommand>(TCommand command, CancellationToken ct = default)
        where TCommand : ICommand
    {
        var pipeline = _sp.GetServices<IPipelineBehavior<TCommand>>().ToList();
        var handler = _sp.GetRequiredService<ICommandHandler<TCommand>>();
        
        Func<Task> handle = () => handler.HandleAsync(command, ct);
        
        // Wrap with pipeline behaviors (inside-out)
        foreach (var behavior in pipeline.AsEnumerable().Reverse())
        {
            var next = handle;
            handle = () => behavior.HandleAsync(command, next, ct);
        }
        
        await handle();
    }
}

public interface IPipelineBehavior<TCommand>
{
    Task HandleAsync(TCommand command, Func<Task> next, CancellationToken ct);
}

// Validation behavior
public class ValidationBehavior<TCommand> : IPipelineBehavior<TCommand>
    where TCommand : ICommand
{
    private readonly IEnumerable<IValidator<TCommand>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TCommand>> validators)
        => _validators = validators;
    
    public async Task HandleAsync(TCommand command, Func<Task> next, CancellationToken ct)
    {
        foreach (var validator in _validators)
        {
            var result = await validator.ValidateAsync(command, ct);
            if (!result.IsValid)
                throw new ValidationException(result.Errors);
        }
        await next();
    }
}

// Transaction behavior
public class TransactionBehavior<TCommand> : IPipelineBehavior<TCommand>
    where TCommand : ICommand
{
    private readonly IUnitOfWork _uow;
    
    public TransactionBehavior(IUnitOfWork uow) => _uow = uow;
    
    public async Task HandleAsync(TCommand command, Func<Task> next, CancellationToken ct)
    {
        await next();
        await _uow.CommitAsync(ct);
    }
}
```

---

## Step 787: Domain Events Dispatcher

```csharp
// ============================================
// Domain Events Dispatch
// ============================================

public class DomainEventDispatcher
{
    private readonly IServiceProvider _sp;
    
    public DomainEventDispatcher(IServiceProvider sp) => _sp = sp;
    
    public async Task DispatchAsync<TEvent>(TEvent domainEvent, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        var handlers = _sp.GetServices<IDomainEventHandler<TEvent>>();
        
        await Task.WhenAll(handlers.Select(h => h.HandleAsync(domainEvent, ct)));
    }
    
    public async Task DispatchAllAsync(
        IReadOnlyList<IDomainEvent> events, CancellationToken ct = default)
    {
        foreach (var evt in events)
            await DispatchAsync((dynamic)evt, ct);
    }
}

// Handler: send notification when order confirmed
public class OrderConfirmedNotificationHandler : IDomainEventHandler<OrderConfirmed>
{
    private readonly IPushNotificationService _push;
    
    public OrderConfirmedNotificationHandler(IPushNotificationService push) => _push = push;
    
    public async Task HandleAsync(OrderConfirmed evt, CancellationToken ct)
        => await _push.SendAsync(evt.CustomerId.Value.ToString(),
            "ออเดอร์ได้รับการยืนยัน",
            "ร้านอาหารกำลังเตรียมอาหารของคุณ", ct);
}
```

---

## Step 788: Functional Programming Utilities

```csharp
// ============================================
// Functional Helpers
// ============================================

public static class FunctionalExtensions
{
    // Pipe operator
    public static TOut Pipe<TIn, TOut>(this TIn value, Func<TIn, TOut> func)
        => func(value);
    
    public static T Tap<T>(this T value, Action<T> action)
    {
        action(value);
        return value;
    }
    
    // Maybe/Option type
    public static TOut? IfNotNull<TIn, TOut>(this TIn? value, Func<TIn, TOut> func)
        where TIn : class
        => value != null ? func(value) : default;
    
    public static T OrDefault<T>(this T? value, T defaultValue)
        where T : class
        => value ?? defaultValue;
    
    // Safe navigation
    public static TOut Safe<TIn, TOut>(this TIn? value, Func<TIn, TOut> func, TOut fallback)
        where TIn : class
    {
        try { return value != null ? func(value) : fallback; }
        catch { return fallback; }
    }
}

// Currying
public static class Curry
{
    public static Func<A, Func<B, C>> Of<A, B, C>(Func<A, B, C> f)
        => a => b => f(a, b);
    
    public static Func<A, Func<B, Func<C, D>>> Of<A, B, C, D>(Func<A, B, C, D> f)
        => a => b => c => f(a, b, c);
}

// Memoization
public static class Memoize
{
    public static Func<TIn, TOut> Of<TIn, TOut>(Func<TIn, TOut> func)
        where TIn : notnull
    {
        var cache = new Dictionary<TIn, TOut>();
        return input =>
        {
            if (cache.TryGetValue(input, out var cached)) return cached;
            return cache[input] = func(input);
        };
    }
}

// Usage
var memoizedDistance = Memoize.Of<(double, double, double, double), double>(
    coords => HaversineDistance(coords.Item1, coords.Item2, coords.Item3, coords.Item4));

var calculateTax = Curry.Of<decimal, decimal, decimal>((amount, rate) => amount * rate);
var calculateVat7 = calculateTax(0.07m);
var tax = calculateVat7(500m); // 35m

static double HaversineDistance(double lat1, double lon1, double lat2, double lon2)
    => 0; // implementation
```

---

## Step 789: Decorator Chain Pattern

```csharp
// ============================================
// Composable Decorator Chain
// ============================================

public interface IOrderProcessor
{
    Task<OrderResult> ProcessAsync(Order order, CancellationToken ct);
}

// Base processor
public class StandardOrderProcessor : IOrderProcessor
{
    private readonly IOrderRepository _repo;
    
    public StandardOrderProcessor(IOrderRepository repo) => _repo = repo;
    
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        await _repo.SaveAsync(order, ct);
        return new OrderResult(order.Id, true, null);
    }
}

// Audit decorator
public class AuditingOrderProcessor : IOrderProcessor
{
    private readonly IOrderProcessor _inner;
    private readonly IAuditLogger _audit;
    
    public AuditingOrderProcessor(IOrderProcessor inner, IAuditLogger audit)
    { _inner = inner; _audit = audit; }
    
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        await _audit.LogAsync("PlaceOrder", order.Id.Value.ToString(), null,
            $"Amount={order.Total.Amount}");
        var result = await _inner.ProcessAsync(order, ct);
        await _audit.LogAsync("PlaceOrderResult", order.Id.Value.ToString(), null,
            result.Success ? "Success" : result.Error ?? "Failed");
        return result;
    }
}

// Retry decorator
public class RetryingOrderProcessor : IOrderProcessor
{
    private readonly IOrderProcessor _inner;
    private readonly int _maxRetries;
    
    public RetryingOrderProcessor(IOrderProcessor inner, int maxRetries = 3)
    { _inner = inner; _maxRetries = maxRetries; }
    
    public async Task<OrderResult> ProcessAsync(Order order, CancellationToken ct)
    {
        for (int attempt = 0; attempt <= _maxRetries; attempt++)
        {
            try { return await _inner.ProcessAsync(order, ct); }
            catch when (attempt < _maxRetries)
            {
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
            }
        }
        return new OrderResult(order.Id, false, "Max retries exceeded");
    }
}

// Chain: Audit → Retry → Standard
var processor = new AuditingOrderProcessor(
    new RetryingOrderProcessor(
        new StandardOrderProcessor(repo)),
    audit);

public record OrderResult(OrderId Id, bool Success, string? Error);
```

---

## Step 790: Architecture Tests

```csharp
// ============================================
// Architecture Tests with NetArchTest
// ============================================

// Install: NetArchTest.Rules

[TestFixture]
public class ArchitectureTests
{
    [Test]
    public void Domain_ShouldNotDependOn_Application()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .That().ResideInNamespace("MyApp.Domain")
            .ShouldNot().HaveDependencyOn("MyApp.Application")
            .GetResult();
        
        Assert.That(result.IsSuccessful, Is.True,
            "Domain must not depend on Application");
    }
    
    [Test]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .That().ResideInNamespace("MyApp.Domain")
            .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
            .GetResult();
        
        Assert.That(result.IsSuccessful, Is.True,
            "Domain must not depend on Infrastructure");
    }
    
    [Test]
    public void Application_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(PlaceOrderCommand).Assembly)
            .That().ResideInNamespace("MyApp.Application")
            .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
            .GetResult();
        
        Assert.That(result.IsSuccessful, Is.True,
            "Application must not depend on Infrastructure");
    }
    
    [Test]
    public void Handlers_ShouldHaveSuffix_Handler()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .That().ImplementInterface(typeof(ICommandHandler<>))
            .Should().HaveNameEndingWith("Handler")
            .GetResult();
        
        Assert.That(result.IsSuccessful, Is.True,
            "All command handlers should end with 'Handler'");
    }
    
    [Test]
    public void Aggregates_ShouldBeSealed_Or_Abstract()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .That().ImplementInterface(typeof(IAggregateRoot))
            .Should().BeSealed()
            .GetResult();
        
        Assert.That(result.IsSuccessful, Is.True,
            "Aggregates should be sealed");
    }
}
```

---

## สรุป Part 79

ใน Part 79 เราได้เรียนรู้:

1. **Clean Architecture** - Domain/Application/Infrastructure layers, strict dependency rule
2. **Unit of Work** - Transaction scope with multiple repositories
3. **Specification Pattern** - Composable And/Or/Not specs for business rules
4. **Result Type** - Railway-oriented programming, Map/Bind/Match
5. **Reactive Extensions** - Subject, Observable.Interval, Throttle, DistinctUntilChanged
6. **CQRS Command Bus** - Pipeline behaviors, validation, transaction
7. **Domain Events** - Dispatcher, fan-out to multiple handlers
8. **Functional Utilities** - Pipe, Tap, Curry, Memoize
9. **Decorator Chain** - Composable processors (Audit → Retry → Standard)
10. **Architecture Tests** - NetArchTest layer dependency enforcement

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 79 | Steps 781-790*
