# Part 47: Advanced Design Patterns
## Steps 461-470: DDD, Event Sourcing, Hexagonal Architecture

---

## Step 461: Domain-Driven Design (DDD)

```csharp
// ============================================
// Domain-Driven Design Fundamentals
// ============================================

/*
 * DDD Building Blocks:
 * 
 * Value Object - Immutable, defined by values
 * Entity - Has identity (Id)
 * Aggregate Root - Consistency boundary
 * Domain Event - Something that happened
 * Repository - Collection-like persistence
 * Domain Service - Logic that doesn't belong to entity
 * Factory - Complex object creation
 */

// Value Objects
public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
    public static Money Thai(decimal amount) => new(amount, "THB");
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Multiply(int quantity) => new(Amount * quantity, Currency);
    
    public bool IsZero => Amount == 0;
    
    public override string ToString() => $"{Amount:N2} {Currency}";
}

public record Address(
    string Street,
    string City,
    string Province,
    string PostalCode,
    string Country = "TH")
{
    public string FullAddress => $"{Street}, {City}, {Province} {PostalCode}";
    
    public bool IsThailand => Country == "TH";
}

public record ProductName
{
    public string Value { get; }
    
    public ProductName(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("ชื่อสินค้าต้องไม่ว่าง");
        if (value.Length > 200)
            throw new ArgumentException("ชื่อสินค้าต้องไม่เกิน 200 ตัวอักษร");
        Value = value.Trim();
    }
    
    public static implicit operator string(ProductName name) => name.Value;
}
```

---

## Step 462: Aggregate Root

```csharp
// ============================================
// Aggregate Root
// ============================================

public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    
    public int Id { get; protected set; }
    public int Version { get; private set; }
    
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
        Version++;
    }
    
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Order Aggregate
public class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = new();
    
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; } = null!;
    public Money TotalAmount { get; private set; } = Money.Zero("THB");
    public DateTime CreatedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public DateTime? DeliveredAt { get; private set; }
    
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    
    // Factory method
    public static Order Create(int customerId, Address shippingAddress)
    {
        var order = new Order
        {
            CustomerId = customerId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Draft,
            CreatedAt = DateTime.UtcNow
        };
        
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }
    
    public void AddLine(int productId, ProductName name, Money unitPrice, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new OrderDomainException("สามารถแก้ไขคำสั่งซื้อ Draft เท่านั้น");
        
        var existing = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (existing != null)
        {
            existing.IncreaseQuantity(quantity);
        }
        else
        {
            _lines.Add(new OrderLine(productId, name, unitPrice, quantity));
        }
        
        RecalculateTotal();
    }
    
    public void RemoveLine(int productId)
    {
        if (Status != OrderStatus.Draft)
            throw new OrderDomainException("สามารถแก้ไขคำสั่งซื้อ Draft เท่านั้น");
        
        _lines.RemoveAll(l => l.ProductId == productId);
        RecalculateTotal();
    }
    
    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new OrderDomainException("ยืนยันได้เฉพาะ Draft");
        
        if (!_lines.Any())
            throw new OrderDomainException("ต้องมีสินค้าอย่างน้อย 1 รายการ");
        
        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount.Amount));
    }
    
    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Confirmed)
            throw new OrderDomainException("จัดส่งได้เฉพาะ Confirmed");
        
        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;
        AddDomainEvent(new OrderShippedEvent(Id, trackingNumber));
    }
    
    public void Deliver()
    {
        if (Status != OrderStatus.Shipped)
            throw new OrderDomainException("รับได้เฉพาะ Shipped");
        
        Status = OrderStatus.Delivered;
        DeliveredAt = DateTime.UtcNow;
        AddDomainEvent(new OrderDeliveredEvent(Id));
    }
    
    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Delivered || Status == OrderStatus.Cancelled)
            throw new OrderDomainException("ยกเลิกไม่ได้");
        
        Status = OrderStatus.Cancelled;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }
    
    private void RecalculateTotal()
    {
        TotalAmount = _lines.Aggregate(
            Money.Zero("THB"),
            (sum, line) => sum.Add(line.UnitPrice.Multiply(line.Quantity)));
    }
}

public class OrderLine
{
    public int ProductId { get; private set; }
    public ProductName Name { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }
    
    public Money LineTotal => UnitPrice.Multiply(Quantity);
    
    public OrderLine(int productId, ProductName name, Money unitPrice, int quantity)
    {
        ProductId = productId;
        Name = name;
        UnitPrice = unitPrice;
        Quantity = quantity > 0 ? quantity : throw new ArgumentException("จำนวนต้องมากกว่า 0");
    }
    
    public void IncreaseQuantity(int by) => Quantity += by;
}

public enum OrderStatus { Draft, Confirmed, Shipped, Delivered, Cancelled }

public class OrderDomainException : Exception
{
    public OrderDomainException(string message) : base(message) { }
}
```

---

## Step 463: Domain Events

```csharp
// ============================================
// Domain Events
// ============================================

public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
    string EventType { get; }
}

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

// Order domain events
public record OrderCreatedEvent(int OrderId, int CustomerId) : DomainEvent;
public record OrderConfirmedEvent(int OrderId, int CustomerId, decimal TotalAmount) : DomainEvent;
public record OrderShippedEvent(int OrderId, string TrackingNumber) : DomainEvent;
public record OrderDeliveredEvent(int OrderId) : DomainEvent;
public record OrderCancelledEvent(int OrderId, string Reason) : DomainEvent;

// Domain Event Dispatcher
public interface IDomainEventDispatcher
{
    Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct = default);
}

public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IServiceProvider _services;
    
    public DomainEventDispatcher(IServiceProvider services) => _services = services;
    
    public async Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct = default)
    {
        foreach (var @event in events)
        {
            var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(@event.GetType());
            var handlers = _services.GetServices(handlerType);
            
            foreach (var handler in handlers)
            {
                await ((dynamic)handler).HandleAsync((dynamic)@event, ct);
            }
        }
    }
}

public interface IDomainEventHandler<T> where T : IDomainEvent
{
    Task HandleAsync(T @event, CancellationToken ct = default);
}

// Handlers
public class SendOrderConfirmationEmailHandler : IDomainEventHandler<OrderConfirmedEvent>
{
    private readonly IEmailService _email;
    
    public SendOrderConfirmationEmailHandler(IEmailService email) => _email = email;
    
    public async Task HandleAsync(OrderConfirmedEvent @event, CancellationToken ct = default)
    {
        await _email.SendOrderConfirmationAsync(@event.OrderId, @event.CustomerId);
    }
}

public class UpdateInventoryOnOrderConfirmedHandler : IDomainEventHandler<OrderConfirmedEvent>
{
    private readonly IInventoryService _inventory;
    
    public UpdateInventoryOnOrderConfirmedHandler(IInventoryService inventory)
        => _inventory = inventory;
    
    public async Task HandleAsync(OrderConfirmedEvent @event, CancellationToken ct = default)
    {
        await _inventory.ReserveForOrderAsync(@event.OrderId);
    }
}
```

---

## Step 464: Event Sourcing

```csharp
// ============================================
// Event Sourcing
// ============================================

public interface IEventStore
{
    Task AppendAsync(string aggregateId, IEnumerable<IDomainEvent> events, int expectedVersion);
    Task<IEnumerable<IDomainEvent>> GetEventsAsync(string aggregateId, int fromVersion = 0);
}

public class SQLiteEventStore : IEventStore
{
    private readonly SQLiteConnection _db;
    
    public SQLiteEventStore(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<StoredEvent>();
    }
    
    public async Task AppendAsync(
        string aggregateId, IEnumerable<IDomainEvent> events, int expectedVersion)
    {
        await Task.Run(() =>
        {
            _db.BeginTransaction();
            try
            {
                // Optimistic concurrency check
                var currentVersion = _db.Table<StoredEvent>()
                    .Where(e => e.AggregateId == aggregateId)
                    .Count();
                
                if (currentVersion != expectedVersion)
                    throw new ConcurrencyException(
                        $"Expected version {expectedVersion}, got {currentVersion}");
                
                int version = expectedVersion;
                foreach (var @event in events)
                {
                    _db.Insert(new StoredEvent
                    {
                        AggregateId = aggregateId,
                        EventType = @event.EventType,
                        Payload = System.Text.Json.JsonSerializer.Serialize(@event,
                            @event.GetType()),
                        Version = ++version,
                        OccurredAt = @event.OccurredAt
                    });
                }
                
                _db.Commit();
            }
            catch
            {
                _db.Rollback();
                throw;
            }
        });
    }
    
    public async Task<IEnumerable<IDomainEvent>> GetEventsAsync(
        string aggregateId, int fromVersion = 0)
    {
        return await Task.Run(() =>
        {
            var stored = _db.Table<StoredEvent>()
                .Where(e => e.AggregateId == aggregateId && e.Version > fromVersion)
                .OrderBy(e => e.Version)
                .ToList();
            
            return stored.Select(Deserialize).ToList().AsEnumerable();
        });
    }
    
    private static IDomainEvent Deserialize(StoredEvent stored)
    {
        var eventType = Type.GetType(stored.EventType)
            ?? throw new InvalidOperationException($"Unknown event type: {stored.EventType}");
        
        return (IDomainEvent)System.Text.Json.JsonSerializer.Deserialize(
            stored.Payload, eventType)!;
    }
}

public class StoredEvent
{
    [PrimaryKey, AutoIncrement] public long Id { get; set; }
    public string AggregateId { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public int Version { get; set; }
    public DateTime OccurredAt { get; set; }
}

public class ConcurrencyException : Exception
{
    public ConcurrencyException(string message) : base(message) { }
}

// Event-sourced Order Repository
public class EventSourcedOrderRepository
{
    private readonly IEventStore _eventStore;
    
    public EventSourcedOrderRepository(IEventStore eventStore) => _eventStore = eventStore;
    
    public async Task<Order> GetByIdAsync(int id)
    {
        var events = await _eventStore.GetEventsAsync(id.ToString());
        var order = new Order();
        
        foreach (var @event in events)
            Apply(order, @event);
        
        return order;
    }
    
    public async Task SaveAsync(Order order)
    {
        var domainEvents = order.DomainEvents;
        await _eventStore.AppendAsync(
            order.Id.ToString(),
            domainEvents,
            order.Version - domainEvents.Count);
        
        order.ClearDomainEvents();
    }
    
    private static void Apply(Order order, IDomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                // Rebuild state from event
                break;
            case OrderConfirmedEvent e:
                // Apply to in-memory state
                break;
        }
    }
}
```

---

## Step 465: Hexagonal Architecture (Ports & Adapters)

```csharp
// ============================================
// Hexagonal Architecture
// ============================================

/*
 *         [UI/API] ──── [Driving Port] ──── [Application Core]
 *                                                 │
 *                                          [Driven Port]
 *                                                 │
 *              [Database/Email/SMS/API] ──── [Adapters]
 */

// Port (interface in domain)
public interface IOrderPort
{
    Task<Order?> GetOrderAsync(int id);
    Task SaveOrderAsync(Order order);
    Task<IEnumerable<Order>> GetCustomerOrdersAsync(int customerId);
}

public interface IPaymentPort
{
    Task<PaymentResult> ChargeAsync(int orderId, decimal amount, PaymentMethod method);
    Task<bool> RefundAsync(string paymentId);
}

public interface INotificationPort
{
    Task SendOrderConfirmationAsync(int orderId, string email);
    Task SendShippingUpdateAsync(int orderId, string trackingNumber, string phone);
}

// Application service (uses ports)
public class OrderApplicationService
{
    private readonly IOrderPort _orders;
    private readonly IPaymentPort _payment;
    private readonly INotificationPort _notifications;
    private readonly IDomainEventDispatcher _events;
    
    public OrderApplicationService(
        IOrderPort orders, IPaymentPort payment,
        INotificationPort notifications, IDomainEventDispatcher events)
    {
        _orders = orders;
        _payment = payment;
        _notifications = notifications;
        _events = events;
    }
    
    public async Task<Result<int>> PlaceOrderAsync(PlaceOrderCommand command)
    {
        // Validate
        if (!command.Items.Any())
            return Result<int>.Failure("ต้องมีสินค้าอย่างน้อย 1 รายการ");
        
        // Create order aggregate
        var order = Order.Create(command.CustomerId, command.ShippingAddress);
        
        foreach (var item in command.Items)
            order.AddLine(item.ProductId, new ProductName(item.Name),
                Money.Thai(item.UnitPrice), item.Quantity);
        
        order.Confirm();
        
        // Charge payment
        var paymentResult = await _payment.ChargeAsync(
            order.Id, order.TotalAmount.Amount, command.PaymentMethod);
        
        if (!paymentResult.IsSuccess)
            return Result<int>.Failure($"ชำระเงินไม่สำเร็จ: {paymentResult.Error}");
        
        // Save
        await _orders.SaveOrderAsync(order);
        
        // Dispatch domain events
        await _events.DispatchAsync(order.DomainEvents);
        
        return Result<int>.Success(order.Id);
    }
}

// Adapter (implements port)
public class SQLiteOrderAdapter : IOrderPort
{
    private readonly SQLiteConnection _db;
    
    public SQLiteOrderAdapter(SQLiteConnection db) => _db = db;
    
    public async Task<Order?> GetOrderAsync(int id)
    {
        return await Task.Run(() =>
        {
            var data = _db.Table<OrderData>().FirstOrDefault(o => o.Id == id);
            return data != null ? MapToDomain(data) : null;
        });
    }
    
    public async Task SaveOrderAsync(Order order)
    {
        await Task.Run(() =>
        {
            _db.InsertOrReplace(MapToData(order));
        });
    }
    
    public async Task<IEnumerable<Order>> GetCustomerOrdersAsync(int customerId)
    {
        return await Task.Run(() =>
        {
            return _db.Table<OrderData>()
                .Where(o => o.CustomerId == customerId)
                .Select(MapToDomain)
                .ToList();
        });
    }
    
    private static Order MapToDomain(OrderData data) => new(); // simplified
    private static OrderData MapToData(Order order) => new(); // simplified
}
```

---

## Step 466: CQRS with MediatR

```csharp
// ============================================
// CQRS with MediatR-style Mediator
// ============================================

// Commands
public record PlaceOrderCommand(
    int CustomerId,
    Address ShippingAddress,
    List<OrderItemDto> Items,
    PaymentMethod PaymentMethod
) : ICommand<Result<int>>;

public record CancelOrderCommand(int OrderId, string Reason) : ICommand<Result>;

// Queries
public record GetOrderQuery(int OrderId) : IQuery<OrderDto?>;
public record GetCustomerOrdersQuery(int CustomerId, int Page, int Size) 
    : IQuery<PagedResult<OrderDto>>;

// Interfaces
public interface ICommand<TResult> { }
public interface IQuery<TResult> { }

public interface ICommandHandler<TCommand, TResult> 
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

public interface IQueryHandler<TQuery, TResult>
    where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct = default);
}

// Mediator
public class AppMediator
{
    private readonly IServiceProvider _services;
    
    public AppMediator(IServiceProvider services) => _services = services;
    
    public async Task<TResult> SendAsync<TResult>(
        ICommand<TResult> command, CancellationToken ct = default)
    {
        var handlerType = typeof(ICommandHandler<,>)
            .MakeGenericType(command.GetType(), typeof(TResult));
        var handler = _services.GetRequiredService(handlerType);
        return await (Task<TResult>)((dynamic)handler).HandleAsync((dynamic)command, ct);
    }
    
    public async Task<TResult> QueryAsync<TResult>(
        IQuery<TResult> query, CancellationToken ct = default)
    {
        var handlerType = typeof(IQueryHandler<,>)
            .MakeGenericType(query.GetType(), typeof(TResult));
        var handler = _services.GetRequiredService(handlerType);
        return await (Task<TResult>)((dynamic)handler).HandleAsync((dynamic)query, ct);
    }
}

// Handler
public class PlaceOrderCommandHandler : ICommandHandler<PlaceOrderCommand, Result<int>>
{
    private readonly OrderApplicationService _orderService;
    
    public PlaceOrderCommandHandler(OrderApplicationService service) => _orderService = service;
    
    public async Task<Result<int>> HandleAsync(
        PlaceOrderCommand command, CancellationToken ct = default)
        => await _orderService.PlaceOrderAsync(command);
}

// ViewModel using Mediator
public partial class CheckoutViewModel : ObservableObject
{
    private readonly AppMediator _mediator;
    
    [ObservableProperty] private bool _isPlacingOrder;
    [ObservableProperty] private string? _errorMessage;
    
    public CheckoutViewModel(AppMediator mediator) => _mediator = mediator;
    
    [RelayCommand]
    private async Task PlaceOrderAsync()
    {
        IsPlacingOrder = true;
        ErrorMessage = null;
        
        var command = BuildCommand();
        var result = await _mediator.SendAsync(command);
        
        IsPlacingOrder = false;
        
        if (result.IsSuccess)
            await Shell.Current.GoToAsync($"//order-confirmation?id={result.Value}");
        else
            ErrorMessage = result.Error;
    }
    
    private PlaceOrderCommand BuildCommand() => throw new NotImplementedException();
}
```

---

## Step 467: Specification Builder

```csharp
// ============================================
// Rich Specification Pattern
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
    public AndSpecification(Specification<T> l, Specification<T> r) { _left = l; _right = r; }
    public override bool IsSatisfiedBy(T entity) => _left.IsSatisfiedBy(entity) && _right.IsSatisfiedBy(entity);
}

public class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public OrSpecification(Specification<T> l, Specification<T> r) { _left = l; _right = r; }
    public override bool IsSatisfiedBy(T entity) => _left.IsSatisfiedBy(entity) || _right.IsSatisfiedBy(entity);
}

public class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpecification(Specification<T> inner) => _inner = inner;
    public override bool IsSatisfiedBy(T entity) => !_inner.IsSatisfiedBy(entity);
}

// Domain specifications
public class InStockSpec : Specification<Product>
{
    public override bool IsSatisfiedBy(Product product) => product.Stock > 0;
}

public class AffordableSpec : Specification<Product>
{
    private readonly decimal _maxPrice;
    public AffordableSpec(decimal maxPrice) => _maxPrice = maxPrice;
    public override bool IsSatisfiedBy(Product product) => product.Price <= _maxPrice;
}

public class InCategorySpec : Specification<Product>
{
    private readonly string _category;
    public InCategorySpec(string category) => _category = category;
    public override bool IsSatisfiedBy(Product product) => product.Category == _category;
}

// Usage
var spec = new InStockSpec()
    .And(new AffordableSpec(1000))
    .And(new InCategorySpec("Electronics"));

var products = allProducts.Where(spec.IsSatisfiedBy).ToList();
```

---

## Step 468: Template Method

```csharp
// ============================================
// Template Method Pattern
// ============================================

public abstract class DataImporter<T>
{
    // Template method
    public async Task<ImportResult> ImportAsync(Stream data)
    {
        var result = new ImportResult();
        
        var raw = await ReadDataAsync(data);
        var items = await ParseAsync(raw);
        var valid = Validate(items, result);
        
        if (result.HasErrors && !ContinueOnError) return result;
        
        await TransformAsync(valid);
        await SaveAsync(valid, result);
        await PostProcessAsync(result);
        
        return result;
    }
    
    protected abstract Task<string> ReadDataAsync(Stream data);
    protected abstract Task<List<T>> ParseAsync(string raw);
    protected virtual bool ContinueOnError => false;
    
    protected virtual List<T> Validate(List<T> items, ImportResult result)
    {
        return items; // default: accept all
    }
    
    protected virtual Task TransformAsync(List<T> items) => Task.CompletedTask;
    
    protected abstract Task SaveAsync(List<T> items, ImportResult result);
    
    protected virtual Task PostProcessAsync(ImportResult result) => Task.CompletedTask;
}

// Concrete implementation
public class CsvProductImporter : DataImporter<Product>
{
    private readonly IProductRepository _products;
    
    public CsvProductImporter(IProductRepository products) => _products = products;
    
    protected override async Task<string> ReadDataAsync(Stream data)
    {
        using var reader = new StreamReader(data);
        return await reader.ReadToEndAsync();
    }
    
    protected override Task<List<Product>> ParseAsync(string raw)
    {
        var lines = raw.Split('\n').Skip(1); // skip header
        var products = lines
            .Where(l => !string.IsNullOrWhiteSpace(l))
            .Select(ParseLine)
            .ToList();
        return Task.FromResult(products);
    }
    
    private static Product ParseLine(string line)
    {
        var parts = line.Split(',');
        return new Product
        {
            Name = parts[0].Trim(),
            Price = decimal.Parse(parts[1].Trim()),
            Stock = int.Parse(parts[2].Trim()),
            Category = parts.ElementAtOrDefault(3)?.Trim() ?? "Other"
        };
    }
    
    protected override async Task SaveAsync(List<Product> items, ImportResult result)
    {
        foreach (var product in items)
        {
            await _products.UpsertAsync(product);
            result.Imported++;
        }
    }
}

public class ImportResult
{
    public int Imported { get; set; }
    public int Skipped { get; set; }
    public List<string> Errors { get; } = new();
    public bool HasErrors => Errors.Count > 0;
}
```

---

## Step 469: Visitor Pattern

```csharp
// ============================================
// Visitor Pattern for Discount Calculation
// ============================================

public interface IDiscountVisitor
{
    decimal Visit(RegularProduct product);
    decimal Visit(PremiumProduct product);
    decimal Visit(ClearanceProduct product);
}

public interface IVisitableProduct
{
    decimal Accept(IDiscountVisitor visitor);
    string Name { get; }
    decimal Price { get; }
}

public class RegularProduct : IVisitableProduct
{
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public decimal Accept(IDiscountVisitor visitor) => visitor.Visit(this);
}

public class PremiumProduct : IVisitableProduct
{
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public int MemberTier { get; init; }
    public decimal Accept(IDiscountVisitor visitor) => visitor.Visit(this);
}

public class ClearanceProduct : IVisitableProduct
{
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public double ClearanceRate { get; init; }
    public decimal Accept(IDiscountVisitor visitor) => visitor.Visit(this);
}

// Discount strategies as visitors
public class MemberDiscountVisitor : IDiscountVisitor
{
    private readonly MemberLevel _level;
    
    public MemberDiscountVisitor(MemberLevel level) => _level = level;
    
    public decimal Visit(RegularProduct product)
    {
        var discount = _level switch
        {
            MemberLevel.Gold => 0.10m,
            MemberLevel.Platinum => 0.15m,
            _ => 0.05m
        };
        return product.Price * (1 - discount);
    }
    
    public decimal Visit(PremiumProduct product)
    {
        // Members get extra 5% on premium
        var baseDiscount = 0.05m + (_level == MemberLevel.Platinum ? 0.10m : 0.05m);
        return product.Price * (1 - baseDiscount);
    }
    
    public decimal Visit(ClearanceProduct product)
    {
        // Clearance already discounted, no member discount
        return product.Price * (1 - (decimal)product.ClearanceRate);
    }
}

public enum MemberLevel { Regular, Gold, Platinum }
```

---

## Step 470: Chain of Responsibility

```csharp
// ============================================
// Chain of Responsibility (Request Pipeline)
// ============================================

public abstract class RequestHandler<TRequest, TResponse>
{
    private RequestHandler<TRequest, TResponse>? _next;
    
    public RequestHandler<TRequest, TResponse> SetNext(
        RequestHandler<TRequest, TResponse> next)
    {
        _next = next;
        return next;
    }
    
    public abstract Task<TResponse> HandleAsync(TRequest request);
    
    protected async Task<TResponse> NextAsync(TRequest request)
    {
        if (_next != null) return await _next.HandleAsync(request);
        throw new InvalidOperationException("No handler processed the request");
    }
}

// Order processing pipeline
public record OrderRequest(Order Order);
public record OrderResponse(bool Success, string? Error = null);

public class ValidationHandler : RequestHandler<OrderRequest, OrderResponse>
{
    public override async Task<OrderResponse> HandleAsync(OrderRequest request)
    {
        if (!request.Order.Lines.Any())
            return new OrderResponse(false, "ไม่มีสินค้าในคำสั่งซื้อ");
        
        if (request.Order.TotalAmount.Amount <= 0)
            return new OrderResponse(false, "ยอดรวมต้องมากกว่า 0");
        
        return await NextAsync(request);
    }
}

public class FraudCheckHandler : RequestHandler<OrderRequest, OrderResponse>
{
    private readonly IFraudDetectionService _fraud;
    
    public FraudCheckHandler(IFraudDetectionService fraud) => _fraud = fraud;
    
    public override async Task<OrderResponse> HandleAsync(OrderRequest request)
    {
        var isRisk = await _fraud.CheckAsync(request.Order.CustomerId,
            request.Order.TotalAmount.Amount);
        
        if (isRisk)
            return new OrderResponse(false, "คำสั่งซื้อถูกระงับเพื่อตรวจสอบความปลอดภัย");
        
        return await NextAsync(request);
    }
}

public class InventoryCheckHandler : RequestHandler<OrderRequest, OrderResponse>
{
    private readonly IInventoryService _inventory;
    
    public InventoryCheckHandler(IInventoryService inventory) => _inventory = inventory;
    
    public override async Task<OrderResponse> HandleAsync(OrderRequest request)
    {
        foreach (var line in request.Order.Lines)
        {
            var available = await _inventory.GetStockAsync(line.ProductId);
            if (available < line.Quantity)
                return new OrderResponse(false, $"สินค้า {line.Name} ไม่เพียงพอ");
        }
        
        return await NextAsync(request);
    }
}

// Usage
public class OrderPipelineFactory
{
    public static RequestHandler<OrderRequest, OrderResponse> Create(
        IFraudDetectionService fraud, IInventoryService inventory)
    {
        var handler = new ValidationHandler();
        handler
            .SetNext(new FraudCheckHandler(fraud))
            .SetNext(new InventoryCheckHandler(inventory));
        
        return handler;
    }
}
```

---

## สรุป Part 47

ใน Part 47 เราได้เรียนรู้:

1. **DDD Basics** - Value Objects, strong typing
2. **Aggregate Root** - Consistency boundary, domain events
3. **Domain Events** - Event dispatcher, handlers
4. **Event Sourcing** - Event store, rebuild from events
5. **Hexagonal Architecture** - Ports & Adapters
6. **CQRS with Mediator** - Commands, Queries, Handlers
7. **Specification Builder** - And/Or/Not composition
8. **Template Method** - CSV importer with hooks
9. **Visitor Pattern** - Type-safe polymorphic operations
10. **Chain of Responsibility** - Request pipeline

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 47 | Steps 461-470*
