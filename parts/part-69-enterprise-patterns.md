# Part 69: Enterprise Integration Patterns
## Steps 681-690: CQRS+ES, Saga Choreography, Inbox, API Gateway

---

## Step 681: Full CQRS with Event Sourcing

```csharp
// ============================================
// Event Sourcing Core
// ============================================

// Event store
public interface IEventStore
{
    Task AppendAsync(string streamId, long expectedVersion, IEnumerable<IDomainEvent> events);
    Task<(IReadOnlyList<IDomainEvent> Events, long Version)> LoadAsync(string streamId);
    Task<IReadOnlyList<IDomainEvent>> LoadAllAfterAsync(long position, int maxCount = 500);
}

public class SQLiteEventStore : IEventStore
{
    private readonly SQLiteAsyncConnection _db;
    private static long _globalPosition = 0;
    
    public SQLiteEventStore(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<StoredEvent>().Wait();
    }
    
    public async Task AppendAsync(string streamId, long expectedVersion,
        IEnumerable<IDomainEvent> events)
    {
        var current = await GetVersionAsync(streamId);
        if (current != expectedVersion)
            throw new ConcurrencyException(streamId, expectedVersion, current);
        
        var version = expectedVersion;
        foreach (var evt in events)
        {
            version++;
            var position = Interlocked.Increment(ref _globalPosition);
            
            await _db.InsertAsync(new StoredEvent
            {
                StreamId = streamId,
                EventType = evt.GetType().AssemblyQualifiedName!,
                Payload = JsonSerializer.Serialize(evt, evt.GetType()),
                Version = version,
                GlobalPosition = position,
                OccurredAt = DateTime.UtcNow
            });
        }
    }
    
    public async Task<(IReadOnlyList<IDomainEvent>, long)> LoadAsync(string streamId)
    {
        var stored = await _db.Table<StoredEvent>()
            .Where(e => e.StreamId == streamId)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        var events = stored.Select(Deserialize).ToList();
        var version = stored.LastOrDefault()?.Version ?? -1;
        
        return (events, version);
    }
    
    public async Task<IReadOnlyList<IDomainEvent>> LoadAllAfterAsync(long position, int maxCount)
        => (await _db.Table<StoredEvent>()
            .Where(e => e.GlobalPosition > position)
            .OrderBy(e => e.GlobalPosition)
            .Take(maxCount)
            .ToListAsync())
            .Select(Deserialize)
            .ToList();
    
    private IDomainEvent Deserialize(StoredEvent stored)
    {
        var type = Type.GetType(stored.EventType)
            ?? throw new InvalidOperationException($"Unknown event type: {stored.EventType}");
        return (IDomainEvent)JsonSerializer.Deserialize(stored.Payload, type)!;
    }
    
    private async Task<long> GetVersionAsync(string streamId)
    {
        var last = await _db.Table<StoredEvent>()
            .Where(e => e.StreamId == streamId)
            .OrderByDescending(e => e.Version)
            .FirstOrDefaultAsync();
        return last?.Version ?? -1;
    }
}

public class StoredEvent
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string StreamId { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public long Version { get; set; }
    public long GlobalPosition { get; set; }
    public DateTime OccurredAt { get; set; }
}

public class ConcurrencyException : Exception
{
    public ConcurrencyException(string streamId, long expected, long actual)
        : base($"Stream '{streamId}': expected v{expected}, got v{actual}") { }
}
```

---

## Step 682: Event-Sourced Aggregate

```csharp
// ============================================
// Event-Sourced Aggregate Base
// ============================================

public abstract class EventSourcedAggregate
{
    private readonly List<IDomainEvent> _pendingEvents = new();
    
    public string StreamId => $"{GetType().Name}-{Id}";
    public int Id { get; protected set; }
    public long Version { get; private set; } = -1;
    
    public IReadOnlyList<IDomainEvent> PendingEvents => _pendingEvents;
    
    protected void Apply(IDomainEvent @event)
    {
        Handle(@event);
        _pendingEvents.Add(@event);
    }
    
    protected abstract void Handle(IDomainEvent @event);
    
    public void LoadFromHistory(IEnumerable<IDomainEvent> events, long version)
    {
        foreach (var evt in events)
            Handle(evt);
        Version = version;
    }
    
    public void ClearPendingEvents() => _pendingEvents.Clear();
}

// Event-sourced FoodOrder
public class EventSourcedFoodOrder : EventSourcedAggregate
{
    public int CustomerId { get; private set; }
    public int RestaurantId { get; private set; }
    public OrderStatus Status { get; private set; }
    public List<OrderLine> Lines { get; private set; } = new();
    public Money Total { get; private set; } = Money.Zero;
    
    // Factory from events
    public static EventSourcedFoodOrder Reconstitute(IEnumerable<IDomainEvent> events, long version)
    {
        var order = new EventSourcedFoodOrder();
        order.LoadFromHistory(events, version);
        return order;
    }
    
    // Command handlers
    public static EventSourcedFoodOrder Place(int customerId, int restaurantId, List<OrderLine> lines)
    {
        var order = new EventSourcedFoodOrder();
        if (lines.Count == 0) throw new DomainException("Order must have at least one item");
        
        var subtotal = lines.Sum(l => l.Subtotal.Amount);
        order.Apply(new FoodOrderPlaced(
            Guid.NewGuid().GetHashCode(), customerId, restaurantId,
            lines, Money.Of(subtotal, "THB"),
            DateTime.UtcNow));
        
        return order;
    }
    
    public void Confirm(string paymentTransactionId)
    {
        if (Status != OrderStatus.Placed)
            throw new DomainException("Can only confirm a placed order");
        
        Apply(new OrderPaymentConfirmed(Id, paymentTransactionId, DateTime.UtcNow));
    }
    
    public void Accept(int cookingMinutes)
    {
        if (Status != OrderStatus.PaymentConfirmed)
            throw new DomainException("Can only accept a payment-confirmed order");
        
        Apply(new OrderAcceptedByRestaurant(Id, cookingMinutes,
            DateTime.UtcNow.AddMinutes(cookingMinutes)));
    }
    
    public void Cancel(string reason)
    {
        if (Status is OrderStatus.PickedUp or OrderStatus.Delivered)
            throw new DomainException("Cannot cancel a delivered order");
        
        Apply(new OrderCancelled(Id, reason, DateTime.UtcNow));
    }
    
    // Event handlers (state mutations)
    protected override void Handle(IDomainEvent @event)
    {
        switch (@event)
        {
            case FoodOrderPlaced e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                RestaurantId = e.RestaurantId;
                Lines = e.Lines.ToList();
                Total = e.Total;
                Status = OrderStatus.Placed;
                break;
            
            case OrderPaymentConfirmed:
                Status = OrderStatus.PaymentConfirmed;
                break;
            
            case OrderAcceptedByRestaurant:
                Status = OrderStatus.Accepted;
                break;
            
            case OrderCancelled:
                Status = OrderStatus.Cancelled;
                break;
        }
    }
}

// Event-sourced repository
public class EventSourcedOrderRepository
{
    private readonly IEventStore _eventStore;
    
    public EventSourcedOrderRepository(IEventStore eventStore) => _eventStore = eventStore;
    
    public async Task SaveAsync(EventSourcedFoodOrder order)
    {
        await _eventStore.AppendAsync(
            order.StreamId, order.Version, order.PendingEvents);
        order.ClearPendingEvents();
    }
    
    public async Task<EventSourcedFoodOrder?> GetByIdAsync(int id)
    {
        var streamId = $"EventSourcedFoodOrder-{id}";
        var (events, version) = await _eventStore.LoadAsync(streamId);
        
        if (!events.Any()) return null;
        return EventSourcedFoodOrder.Reconstitute(events, version);
    }
}
```

---

## Step 683: CQRS Query Side (Read Model)

```csharp
// ============================================
// CQRS Read Model Projection
// ============================================

// Query
public record GetOrderSummaryQuery(int CustomerId, int Page = 1, int PageSize = 20) : IQuery<PagedResult<OrderSummaryDto>>;
public record GetOrderDetailQuery(int OrderId) : IQuery<OrderDetailDto?>;

// DTOs for read side
public record OrderSummaryDto(
    int Id, string RestaurantName, string RestaurantImage,
    decimal Total, string Status, DateTime PlacedAt, int ItemCount);

public record OrderDetailDto(
    int Id, string RestaurantName, DateTime PlacedAt,
    string Status, List<OrderLineDto> Lines,
    decimal Subtotal, decimal DeliveryFee, decimal Total,
    string? RiderName, string DeliveryAddress);

// Read model table (denormalized for fast reads)
public class OrderReadModel
{
    [PrimaryKey] public int OrderId { get; set; }
    public int CustomerId { get; set; }
    public string RestaurantName { get; set; } = string.Empty;
    public string RestaurantImage { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public int ItemCount { get; set; }
    public string LinesJson { get; set; } = string.Empty;
    public string? RiderName { get; set; }
    public string DeliveryAddress { get; set; } = string.Empty;
    public DateTime PlacedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

// Projector: listens to events, maintains read model
public class OrderReadModelProjector :
    IEventHandler<FoodOrderPlaced>,
    IEventHandler<OrderAcceptedByRestaurant>,
    IEventHandler<OrderCancelled>
{
    private readonly SQLiteConnection _db;
    
    public OrderReadModelProjector(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<OrderReadModel>();
    }
    
    public Task HandleAsync(FoodOrderPlaced evt, CancellationToken ct = default)
    {
        _db.InsertOrReplace(new OrderReadModel
        {
            OrderId = evt.OrderId,
            CustomerId = evt.CustomerId,
            Status = "placed",
            Total = evt.Total.Amount,
            ItemCount = evt.Lines.Count,
            LinesJson = JsonSerializer.Serialize(evt.Lines),
            PlacedAt = evt.OccurredAt,
            UpdatedAt = DateTime.UtcNow
        });
        return Task.CompletedTask;
    }
    
    public Task HandleAsync(OrderAcceptedByRestaurant evt, CancellationToken ct = default)
    {
        _db.Execute(
            "UPDATE OrderReadModel SET Status='accepted', UpdatedAt=? WHERE OrderId=?",
            DateTime.UtcNow, evt.OrderId);
        return Task.CompletedTask;
    }
    
    public Task HandleAsync(OrderCancelled evt, CancellationToken ct = default)
    {
        _db.Execute(
            "UPDATE OrderReadModel SET Status='cancelled', UpdatedAt=? WHERE OrderId=?",
            DateTime.UtcNow, evt.OrderId);
        return Task.CompletedTask;
    }
}

// Query handler
public class GetOrderSummaryQueryHandler : IQueryHandler<GetOrderSummaryQuery, PagedResult<OrderSummaryDto>>
{
    private readonly SQLiteConnection _db;
    
    public GetOrderSummaryQueryHandler(SQLiteConnection db) => _db = db;
    
    public Task<PagedResult<OrderSummaryDto>> HandleAsync(
        GetOrderSummaryQuery query, CancellationToken ct = default)
    {
        var total = _db.ExecuteScalar<int>(
            "SELECT COUNT(*) FROM OrderReadModel WHERE CustomerId=?", query.CustomerId);
        
        var items = _db.Table<OrderReadModel>()
            .Where(o => o.CustomerId == query.CustomerId)
            .OrderByDescending(o => o.PlacedAt)
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(o => new OrderSummaryDto(
                o.OrderId, o.RestaurantName, o.RestaurantImage,
                o.Total, o.Status, o.PlacedAt, o.ItemCount))
            .ToList();
        
        return Task.FromResult(new PagedResult<OrderSummaryDto>(
            items, total, query.Page, query.PageSize));
    }
}
```

---

## Step 684: Saga Choreography (Event-Driven)

```csharp
// ============================================
// Saga Choreography (vs Orchestration)
// ============================================

// Each service reacts to events independently (no central orchestrator)

// Payment Service: listens to OrderPlaced, emits PaymentCompleted/Failed
public class PaymentSagaParticipant :
    IEventHandler<FoodOrderPlaced>
{
    private readonly IPaymentService _payments;
    private readonly IEventBus _bus;
    
    public PaymentSagaParticipant(IPaymentService payments, IEventBus bus)
    {
        _payments = payments;
        _bus = bus;
    }
    
    public async Task HandleAsync(FoodOrderPlaced evt, CancellationToken ct = default)
    {
        try
        {
            var result = await _payments.ChargeAsync(evt.CustomerId, evt.Total.Amount);
            
            if (result.Success)
                await _bus.PublishAsync(new PaymentCompleted(
                    evt.OrderId, result.TransactionId!, evt.Total.Amount));
            else
                await _bus.PublishAsync(new PaymentFailed(0, evt.OrderId, result.Error!));
        }
        catch (Exception ex)
        {
            await _bus.PublishAsync(new PaymentFailed(0, evt.OrderId, ex.Message));
        }
    }
}

// Restaurant Service: listens to PaymentCompleted, notifies restaurant
public class RestaurantNotificationParticipant :
    IEventHandler<PaymentCompleted>
{
    private readonly IRestaurantNotifier _notifier;
    private readonly IEventBus _bus;
    
    public RestaurantNotificationParticipant(IRestaurantNotifier notifier, IEventBus bus)
    {
        _notifier = notifier;
        _bus = bus;
    }
    
    public async Task HandleAsync(PaymentCompleted evt, CancellationToken ct = default)
    {
        await _notifier.NotifyNewOrderAsync(evt.OrderId);
        await _bus.PublishAsync(new RestaurantNotified(evt.OrderId));
    }
}

// Compensation: PaymentFailed → Cancel Order
public class CompensatingTransactionHandler :
    IEventHandler<PaymentFailed>
{
    private readonly EventSourcedOrderRepository _orders;
    private readonly IEventBus _bus;
    
    public CompensatingTransactionHandler(EventSourcedOrderRepository orders, IEventBus bus)
    {
        _orders = orders;
        _bus = bus;
    }
    
    public async Task HandleAsync(PaymentFailed evt, CancellationToken ct = default)
    {
        var order = await _orders.GetByIdAsync(evt.OrderId);
        if (order == null) return;
        
        order.Cancel($"Payment failed: {evt.Reason}");
        await _orders.SaveAsync(order);
        
        await _bus.PublishAsync(new OrderCancelled(evt.OrderId, evt.Reason, DateTime.UtcNow));
    }
}
```

---

## Step 685: Inbox Pattern

```csharp
// ============================================
// Transactional Inbox Pattern
// ============================================

// Prevents duplicate event processing
public class InboxProcessor
{
    private readonly SQLiteConnection _db;
    private readonly IServiceProvider _services;
    
    public InboxProcessor(SQLiteConnection db, IServiceProvider services)
    {
        _db = db;
        _services = services;
        _db.CreateTable<InboxMessage>();
    }
    
    public void Record(IDomainEvent evt)
    {
        var msgId = ComputeMessageId(evt);
        
        var existing = _db.Table<InboxMessage>()
            .FirstOrDefault(m => m.MessageId == msgId);
        
        if (existing != null) return; // Already seen
        
        _db.Insert(new InboxMessage
        {
            MessageId = msgId,
            EventType = evt.GetType().AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(evt, evt.GetType()),
            Status = InboxStatus.Pending,
            ReceivedAt = DateTime.UtcNow
        });
    }
    
    public async Task ProcessPendingAsync(CancellationToken ct = default)
    {
        var pending = _db.Table<InboxMessage>()
            .Where(m => m.Status == InboxStatus.Pending)
            .Take(20)
            .ToList();
        
        foreach (var msg in pending)
        {
            try
            {
                await DispatchAsync(msg, ct);
                
                msg.Status = InboxStatus.Processed;
                msg.ProcessedAt = DateTime.UtcNow;
                _db.Update(msg);
            }
            catch (Exception ex)
            {
                msg.RetryCount++;
                msg.LastError = ex.Message;
                
                if (msg.RetryCount >= 3)
                    msg.Status = InboxStatus.Failed;
                
                _db.Update(msg);
            }
        }
    }
    
    private async Task DispatchAsync(InboxMessage msg, CancellationToken ct)
    {
        var type = Type.GetType(msg.EventType)!;
        var evt = (IDomainEvent)JsonSerializer.Deserialize(msg.Payload, type)!;
        
        var handlerType = typeof(IEventHandler<>).MakeGenericType(type);
        var handlers = _services.GetServices(handlerType);
        
        foreach (var handler in handlers)
        {
            await (Task)handlerType.GetMethod("HandleAsync")!
                .Invoke(handler, new object[] { evt, ct })!;
        }
    }
    
    private string ComputeMessageId(IDomainEvent evt)
    {
        var key = $"{evt.GetType().Name}:{JsonSerializer.Serialize(evt)}";
        var bytes = System.Security.Cryptography.SHA256.HashData(
            System.Text.Encoding.UTF8.GetBytes(key));
        return Convert.ToHexString(bytes)[..16];
    }
}

public class InboxMessage
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string MessageId { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public InboxStatus Status { get; set; }
    public int RetryCount { get; set; }
    public string? LastError { get; set; }
    public DateTime ReceivedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
}

public enum InboxStatus { Pending, Processed, Failed }
```

---

## Step 686: API Gateway Pattern

```csharp
// ============================================
// Client-Side API Gateway
// ============================================

public interface IApiGateway
{
    Task<T> GetAsync<T>(string path, Dictionary<string, string>? query = null);
    Task<TResult> PostAsync<TRequest, TResult>(string path, TRequest body);
    Task<TResult> PutAsync<TRequest, TResult>(string path, TRequest body);
    Task DeleteAsync(string path);
}

public class ApiGateway : IApiGateway
{
    private readonly HttpClient _http;
    private readonly IStructuredLogger _logger;
    private readonly AppPerformanceMonitor _perf;
    
    public ApiGateway(HttpClient http, IStructuredLogger logger, AppPerformanceMonitor perf)
    {
        _http = http;
        _logger = logger;
        _perf = perf;
    }
    
    public async Task<T> GetAsync<T>(string path, Dictionary<string, string>? query = null)
    {
        var url = BuildUrl(path, query);
        var sw = Stopwatch.StartNew();
        
        try
        {
            var response = await _http.GetAsync(url);
            sw.Stop();
            
            await _perf.TrackApiCallAsync(path, (int)response.StatusCode, sw.ElapsedMilliseconds);
            
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<T>()
                ?? throw new InvalidOperationException("Empty response");
        }
        catch (HttpRequestException ex)
        {
            _logger.Error($"GET {path} failed", ex);
            throw new ApiException(path, ex);
        }
    }
    
    public async Task<TResult> PostAsync<TRequest, TResult>(string path, TRequest body)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var response = await _http.PostAsJsonAsync(path, body);
            sw.Stop();
            
            await _perf.TrackApiCallAsync(path, (int)response.StatusCode, sw.ElapsedMilliseconds);
            
            if (!response.IsSuccessStatusCode)
            {
                var error = await response.Content.ReadFromJsonAsync<ErrorResponse>();
                throw new ApiException(path, (int)response.StatusCode, error?.Message);
            }
            
            return await response.Content.ReadFromJsonAsync<TResult>()
                ?? throw new InvalidOperationException("Empty response");
        }
        catch (ApiException) { throw; }
        catch (Exception ex)
        {
            _logger.Error($"POST {path} failed", ex);
            throw new ApiException(path, ex);
        }
    }
    
    public Task<TResult> PutAsync<TRequest, TResult>(string path, TRequest body)
        => PostAsync<TRequest, TResult>(path, body); // simplified
    
    public async Task DeleteAsync(string path)
    {
        var response = await _http.DeleteAsync(path);
        response.EnsureSuccessStatusCode();
    }
    
    private string BuildUrl(string path, Dictionary<string, string>? query)
    {
        if (query == null || !query.Any()) return path;
        var qs = string.Join("&", query.Select(p => $"{Uri.EscapeDataString(p.Key)}={Uri.EscapeDataString(p.Value)}"));
        return $"{path}?{qs}";
    }
}

public class ApiException : Exception
{
    public int? StatusCode { get; }
    public string Path { get; }
    
    public ApiException(string path, Exception inner)
        : base($"API call to {path} failed", inner)
    { Path = path; }
    
    public ApiException(string path, int statusCode, string? message)
        : base(message ?? $"API call to {path} returned {statusCode}")
    { Path = path; StatusCode = statusCode; }
}
```

---

## Step 687: Mediator Pattern

```csharp
// ============================================
// Mediator (like MediatR)
// ============================================

public interface IMediator
{
    Task<TResult> SendAsync<TResult>(IQuery<TResult> query, CancellationToken ct = default);
    Task SendAsync(ICommand command, CancellationToken ct = default);
    Task PublishAsync(IDomainEvent notification, CancellationToken ct = default);
}

public interface ICommand { }
public interface IQuery<TResult> { }
public interface ICommandHandler<TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command, CancellationToken ct = default);
}
public interface IQueryHandler<TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct = default);
}

public class Mediator : IMediator
{
    private readonly IServiceProvider _services;
    
    public Mediator(IServiceProvider services) => _services = services;
    
    public Task<TResult> SendAsync<TResult>(IQuery<TResult> query, CancellationToken ct = default)
    {
        var handlerType = typeof(IQueryHandler<,>)
            .MakeGenericType(query.GetType(), typeof(TResult));
        
        var handler = _services.GetRequiredService(handlerType);
        var method = handlerType.GetMethod("HandleAsync")!;
        
        return (Task<TResult>)method.Invoke(handler, new object[] { query, ct })!;
    }
    
    public Task SendAsync(ICommand command, CancellationToken ct = default)
    {
        var handlerType = typeof(ICommandHandler<>).MakeGenericType(command.GetType());
        var handler = _services.GetRequiredService(handlerType);
        var method = handlerType.GetMethod("HandleAsync")!;
        return (Task)method.Invoke(handler, new object[] { command, ct })!;
    }
    
    public async Task PublishAsync(IDomainEvent notification, CancellationToken ct = default)
    {
        var handlerType = typeof(IEventHandler<>).MakeGenericType(notification.GetType());
        var handlers = _services.GetServices(handlerType);
        
        foreach (var handler in handlers)
        {
            var method = handlerType.GetMethod("HandleAsync")!;
            await (Task)method.Invoke(handler, new object[] { notification, ct })!;
        }
    }
}

// Pipeline behaviors (cross-cutting concerns)
public interface IPipelineBehavior<TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command, Func<Task> next, CancellationToken ct);
}

public class LoggingBehavior<TCommand> : IPipelineBehavior<TCommand>
    where TCommand : ICommand
{
    private readonly IStructuredLogger _logger;
    
    public LoggingBehavior(IStructuredLogger logger) => _logger = logger;
    
    public async Task HandleAsync(TCommand command, Func<Task> next, CancellationToken ct)
    {
        _logger.Info($"Executing {typeof(TCommand).Name}");
        var sw = Stopwatch.StartNew();
        
        try
        {
            await next();
            _logger.Info($"{typeof(TCommand).Name} completed in {sw.ElapsedMilliseconds}ms");
        }
        catch (Exception ex)
        {
            _logger.Error($"{typeof(TCommand).Name} failed", ex);
            throw;
        }
    }
}

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
```

---

## Step 688: Process Manager

```csharp
// ============================================
// Process Manager (Long-Running Business Process)
// ============================================

// Tracks state of a multi-step business process
public class OrderFulfillmentProcess
{
    public int OrderId { get; set; }
    public FulfillmentStep CurrentStep { get; set; }
    public string? FailureReason { get; set; }
    public DateTime StartedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public int RetryCount { get; set; }
    public Dictionary<string, string> State { get; set; } = new();
}

public enum FulfillmentStep
{
    WaitingForPayment,
    PaymentReceived,
    SentToRestaurant,
    AcceptedByRestaurant,
    ReadyForPickup,
    RiderAssigned,
    PickedUp,
    Delivered,
    Failed,
    Cancelled
}

public class OrderFulfillmentProcessManager :
    IEventHandler<FoodOrderPlaced>,
    IEventHandler<PaymentCompleted>,
    IEventHandler<PaymentFailed>,
    IEventHandler<OrderAcceptedByRestaurant>
{
    private readonly SQLiteConnection _db;
    private readonly IEventBus _bus;
    private readonly IStructuredLogger _logger;
    
    public OrderFulfillmentProcessManager(
        SQLiteConnection db, IEventBus bus, IStructuredLogger logger)
    {
        _db = db;
        _bus = bus;
        _logger = logger;
        _db.CreateTable<OrderFulfillmentProcess>();
    }
    
    public Task HandleAsync(FoodOrderPlaced evt, CancellationToken ct = default)
    {
        _db.Insert(new OrderFulfillmentProcess
        {
            OrderId = evt.OrderId,
            CurrentStep = FulfillmentStep.WaitingForPayment,
            StartedAt = DateTime.UtcNow
        });
        return Task.CompletedTask;
    }
    
    public async Task HandleAsync(PaymentCompleted evt, CancellationToken ct = default)
    {
        var process = GetProcess(evt.OrderId);
        if (process == null) return;
        
        process.CurrentStep = FulfillmentStep.PaymentReceived;
        _db.Update(process);
        
        // Trigger next step: notify restaurant
        await _bus.PublishAsync(new NotifyRestaurantCommand(evt.OrderId));
    }
    
    public async Task HandleAsync(PaymentFailed evt, CancellationToken ct = default)
    {
        var process = GetProcess(evt.OrderId);
        if (process == null) return;
        
        if (process.RetryCount < 2)
        {
            process.RetryCount++;
            _db.Update(process);
            await _bus.PublishAsync(new RetryPaymentCommand(evt.OrderId));
        }
        else
        {
            process.CurrentStep = FulfillmentStep.Failed;
            process.FailureReason = evt.Reason;
            _db.Update(process);
        }
    }
    
    public Task HandleAsync(OrderAcceptedByRestaurant evt, CancellationToken ct = default)
    {
        var process = GetProcess(evt.OrderId);
        if (process == null) return Task.CompletedTask;
        
        process.CurrentStep = FulfillmentStep.AcceptedByRestaurant;
        _db.Update(process);
        return Task.CompletedTask;
    }
    
    private OrderFulfillmentProcess? GetProcess(int orderId)
        => _db.Table<OrderFulfillmentProcess>().FirstOrDefault(p => p.OrderId == orderId);
}

public record NotifyRestaurantCommand(int OrderId) : ICommand;
public record RetryPaymentCommand(int OrderId) : ICommand;
```

---

## Step 689: Anti-Corruption Layer

```csharp
// ============================================
// Anti-Corruption Layer (ACL)
// ============================================

// Translate external API models to our domain

// External: payment provider API model
public class PaymentProviderResponse
{
    [JsonPropertyName("txn_id")] public string TransactionId { get; set; } = string.Empty;
    [JsonPropertyName("status_code")] public string StatusCode { get; set; } = string.Empty;
    [JsonPropertyName("amount_satangs")] public int AmountSatangs { get; set; }
    [JsonPropertyName("timestamp_unix")] public long TimestampUnix { get; set; }
}

// ACL: translate to our domain
public class PaymentProviderAcl
{
    public PaymentResult Translate(PaymentProviderResponse response)
    {
        var success = response.StatusCode is "00" or "SUCCESS";
        var amount = response.AmountSatangs / 100m; // Satangs to Baht
        
        return success
            ? new PaymentResult(true, response.TransactionId, null)
            : new PaymentResult(false, null, TranslateError(response.StatusCode));
    }
    
    private string TranslateError(string code) => code switch
    {
        "01" => "บัตรหมดอายุ",
        "05" => "ยอดเงินในบัตรไม่เพียงพอ",
        "51" => "ยอดชำระเกินวงเงิน",
        "14" => "หมายเลขบัตรไม่ถูกต้อง",
        _ => $"รหัสข้อผิดพลาด: {code}"
    };
}

// External: delivery partner API
public class DeliveryPartnerApi
{
    public string rider_id { get; set; } = string.Empty;
    public string rider_name_th { get; set; } = string.Empty;
    public string phone_no { get; set; } = string.Empty;
    public double current_lat { get; set; }
    public double current_lng { get; set; }
    public string vehicle_plate { get; set; } = string.Empty;
}

// ACL for delivery partner
public class DeliveryPartnerAcl
{
    public RiderInfo Translate(DeliveryPartnerApi api) => new(
        api.rider_id,
        api.rider_name_th,
        InputSanitizer.NormalizePhone(api.phone_no),
        new Location(api.current_lat, api.current_lng),
        api.vehicle_plate);
}

public record RiderInfo(string Id, string Name, string Phone, Location Location, string Plate);
```

---

## Step 690: Event Schema Evolution

```csharp
// ============================================
// Event Schema Evolution & Upcasting
// ============================================

// V1 event (original)
public record OrderPlacedV1(int OrderId, int CustomerId, decimal Total) : IDomainEvent;

// V2 event (added restaurant info)
public record OrderPlacedV2(int OrderId, int CustomerId, int RestaurantId,
    decimal Total, string Currency) : IDomainEvent;

// V3 event (added delivery address)
public record OrderPlacedV3(int OrderId, int CustomerId, int RestaurantId,
    decimal Total, string Currency, string DeliveryAddressJson) : IDomainEvent;

// Upcaster: converts old events to current version
public class OrderPlacedUpcaster
{
    public OrderPlacedV3 Upcast(IDomainEvent evt) => evt switch
    {
        OrderPlacedV3 v3 => v3,
        OrderPlacedV2 v2 => new OrderPlacedV3(
            v2.OrderId, v2.CustomerId, v2.RestaurantId,
            v2.Total, v2.Currency, "{}"),
        OrderPlacedV1 v1 => new OrderPlacedV3(
            v1.OrderId, v1.CustomerId, 0,
            v1.Total, "THB", "{}"),
        _ => throw new InvalidOperationException($"Unknown version: {evt.GetType()}")
    };
}

// Event store with upcasting
public class UpcastingEventStore : IEventStore
{
    private readonly SQLiteEventStore _inner;
    private readonly Dictionary<Type, Func<IDomainEvent, IDomainEvent>> _upcasters;
    
    public UpcastingEventStore(SQLiteEventStore inner)
    {
        _inner = inner;
        
        var upcaster = new OrderPlacedUpcaster();
        _upcasters = new()
        {
            [typeof(OrderPlacedV1)] = e => upcaster.Upcast(e),
            [typeof(OrderPlacedV2)] = e => upcaster.Upcast(e),
        };
    }
    
    public async Task<(IReadOnlyList<IDomainEvent>, long)> LoadAsync(string streamId)
    {
        var (events, version) = await _inner.LoadAsync(streamId);
        var upcasted = events.Select(ApplyUpcasters).ToList();
        return (upcasted, version);
    }
    
    private IDomainEvent ApplyUpcasters(IDomainEvent evt)
    {
        if (_upcasters.TryGetValue(evt.GetType(), out var upcast))
            return upcast(evt);
        return evt;
    }
    
    public Task AppendAsync(string streamId, long expectedVersion, IEnumerable<IDomainEvent> events)
        => _inner.AppendAsync(streamId, expectedVersion, events);
    
    public Task<IReadOnlyList<IDomainEvent>> LoadAllAfterAsync(long position, int maxCount)
        => _inner.LoadAllAfterAsync(position, maxCount);
}
```

---

## สรุป Part 69

ใน Part 69 เราได้เรียนรู้:

1. **Event Sourcing** - Event store, stored events, concurrency check
2. **Event-Sourced Aggregate** - Apply/Handle, pending events, reconstitute
3. **CQRS Read Model** - Projector, denormalized read tables, query handlers
4. **Saga Choreography** - Event-driven compensation (vs orchestration)
5. **Inbox Pattern** - Idempotent event processing, retry logic
6. **API Gateway** - Unified HTTP client with logging/metrics
7. **Mediator Pattern** - Command/Query dispatch, pipeline behaviors
8. **Process Manager** - Long-running multi-step business process state
9. **Anti-Corruption Layer** - External API translation
10. **Event Schema Evolution** - Upcasters for backward compatibility

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 69 | Steps 681-690*
