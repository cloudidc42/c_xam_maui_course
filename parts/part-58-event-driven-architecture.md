# Part 58: Event-Driven Architecture & Message Bus
## Steps 571-580: Event Bus, Message Broker, Saga, Outbox Pattern

---

## Step 571: In-Process Event Bus

```csharp
// ============================================
// In-Process Event Bus (Pub/Sub)
// ============================================

public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

public interface IEventHandler<TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct = default);
}

public class EventBus
{
    private readonly Dictionary<Type, List<object>> _handlers = new();
    private readonly IServiceProvider _services;
    
    public EventBus(IServiceProvider services) => _services = services;
    
    public void Register<TEvent, THandler>()
        where TEvent : IDomainEvent
        where THandler : IEventHandler<TEvent>
    {
        var key = typeof(TEvent);
        if (!_handlers.ContainsKey(key))
            _handlers[key] = new();
        _handlers[key].Add(typeof(THandler));
    }
    
    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        if (!_handlers.TryGetValue(typeof(TEvent), out var handlerTypes))
            return;
        
        var tasks = handlerTypes.Select(async ht =>
        {
            var handler = (IEventHandler<TEvent>)_services.GetRequiredService(ht);
            await handler.HandleAsync(@event, ct);
        });
        
        await Task.WhenAll(tasks);
    }
}

// Domain events
public record OrderCreatedEvent(
    Guid EventId, DateTime OccurredAt,
    int OrderId, int CustomerId, decimal Amount) : IDomainEvent;

public record OrderShippedEvent(
    Guid EventId, DateTime OccurredAt,
    int OrderId, string TrackingNumber) : IDomainEvent;

public record PaymentCompletedEvent(
    Guid EventId, DateTime OccurredAt,
    int OrderId, string PaymentRef, decimal Amount) : IDomainEvent;

// Handlers
public class SendOrderConfirmationEmailHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IEmailService _email;
    public SendOrderConfirmationEmailHandler(IEmailService email) => _email = email;
    
    public async Task HandleAsync(OrderCreatedEvent @event, CancellationToken ct = default)
    {
        await _email.SendOrderConfirmationAsync(@event.CustomerId, @event.OrderId);
    }
}

public class UpdateInventoryOnOrderHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IInventoryService _inventory;
    public UpdateInventoryOnOrderHandler(IInventoryService inventory) => _inventory = inventory;
    
    public async Task HandleAsync(OrderCreatedEvent @event, CancellationToken ct = default)
    {
        await _inventory.ReserveForOrderAsync(@event.OrderId, ct);
    }
}
```

---

## Step 572: Outbox Pattern

```csharp
// ============================================
// Transactional Outbox Pattern
// ============================================

public class OutboxMessage
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public bool Processed { get; set; }
    public int Attempts { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
}

public interface IOutbox
{
    void Add<TEvent>(TEvent @event) where TEvent : IDomainEvent;
    Task FlushAsync(CancellationToken ct = default);
}

public class SQLiteOutbox : IOutbox
{
    private readonly SQLiteConnection _db;
    private readonly EventBus _bus;
    private static readonly System.Text.Json.JsonSerializerOptions _opts = new()
    {
        WriteIndented = false
    };
    
    public SQLiteOutbox(SQLiteConnection db, EventBus bus)
    {
        _db = db;
        _bus = bus;
        _db.CreateTable<OutboxMessage>();
    }
    
    public void Add<TEvent>(TEvent @event) where TEvent : IDomainEvent
    {
        _db.Insert(new OutboxMessage
        {
            EventType = typeof(TEvent).AssemblyQualifiedName!,
            Payload = System.Text.Json.JsonSerializer.Serialize(@event, _opts),
            CreatedAt = DateTime.UtcNow
        });
    }
    
    public async Task FlushAsync(CancellationToken ct = default)
    {
        var pending = _db.Table<OutboxMessage>()
            .Where(m => !m.Processed && m.Attempts < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(50)
            .ToList();
        
        foreach (var msg in pending)
        {
            if (ct.IsCancellationRequested) break;
            
            try
            {
                var eventType = Type.GetType(msg.EventType)
                    ?? throw new InvalidOperationException($"Unknown type: {msg.EventType}");
                
                var @event = (IDomainEvent)System.Text.Json.JsonSerializer.Deserialize(
                    msg.Payload, eventType, _opts)!;
                
                await PublishAsync((dynamic)@event, ct);
                
                msg.Processed = true;
                msg.ProcessedAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                msg.Attempts++;
                msg.Error = ex.Message;
            }
            
            _db.Update(msg);
        }
    }
    
    private Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct)
        where TEvent : IDomainEvent
        => _bus.PublishAsync(@event, ct);
}

// Use outbox in aggregate service
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IOutbox _outbox;
    private readonly SQLiteConnection _db;
    
    public OrderService(IOrderRepository repo, IOutbox outbox, SQLiteConnection db)
    {
        _repo = repo;
        _outbox = outbox;
        _db = db;
    }
    
    public async Task<int> CreateOrderAsync(CreateOrderRequest req)
    {
        _db.BeginTransaction();
        
        try
        {
            var orderId = await _repo.CreateAsync(req);
            
            _outbox.Add(new OrderCreatedEvent(
                Guid.NewGuid(), DateTime.UtcNow,
                orderId, req.CustomerId, req.Amount));
            
            _db.Commit();
            return orderId;
        }
        catch
        {
            _db.Rollback();
            throw;
        }
    }
}
```

---

## Step 573: Saga Pattern

```csharp
// ============================================
// Saga Pattern (Orchestration)
// ============================================

public enum OrderSagaStep
{
    Started, PaymentRequested, PaymentCompleted,
    InventoryReserved, ShipmentRequested, Completed,
    Compensating, Compensated, Failed
}

public class OrderSaga
{
    [PrimaryKey] public Guid SagaId { get; set; }
    public int OrderId { get; set; }
    public OrderSagaStep Step { get; set; }
    public string? Error { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    
    // Compensation data
    public string? PaymentRef { get; set; }
    public bool InventoryReserved { get; set; }
}

public class OrderSagaOrchestrator
{
    private readonly SQLiteConnection _db;
    private readonly IPaymentService _payment;
    private readonly IInventoryService _inventory;
    private readonly IShipmentService _shipment;
    private readonly IOrderRepository _orders;
    
    public OrderSagaOrchestrator(
        SQLiteConnection db,
        IPaymentService payment,
        IInventoryService inventory,
        IShipmentService shipment,
        IOrderRepository orders)
    {
        _db = db;
        _payment = payment;
        _inventory = inventory;
        _shipment = shipment;
        _orders = orders;
        _db.CreateTable<OrderSaga>();
    }
    
    public async Task<Guid> StartAsync(int orderId)
    {
        var saga = new OrderSaga
        {
            SagaId = Guid.NewGuid(),
            OrderId = orderId,
            Step = OrderSagaStep.Started,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };
        _db.Insert(saga);
        
        await ExecuteAsync(saga);
        return saga.SagaId;
    }
    
    private async Task ExecuteAsync(OrderSaga saga)
    {
        try
        {
            // Step 1: Request payment
            UpdateStep(saga, OrderSagaStep.PaymentRequested);
            var paymentRef = await _payment.ChargeAsync(saga.OrderId);
            saga.PaymentRef = paymentRef;
            
            // Step 2: Reserve inventory
            UpdateStep(saga, OrderSagaStep.InventoryReserved);
            await _inventory.ReserveAsync(saga.OrderId);
            saga.InventoryReserved = true;
            
            // Step 3: Request shipment
            UpdateStep(saga, OrderSagaStep.ShipmentRequested);
            await _shipment.ScheduleAsync(saga.OrderId);
            
            // Complete
            UpdateStep(saga, OrderSagaStep.Completed);
            await _orders.MarkAsConfirmedAsync(saga.OrderId);
        }
        catch (Exception ex)
        {
            saga.Error = ex.Message;
            await CompensateAsync(saga);
        }
    }
    
    private async Task CompensateAsync(OrderSaga saga)
    {
        UpdateStep(saga, OrderSagaStep.Compensating);
        
        var errors = new List<string>();
        
        // Reverse: inventory → payment
        if (saga.InventoryReserved)
        {
            try { await _inventory.ReleaseReservationAsync(saga.OrderId); }
            catch (Exception ex) { errors.Add($"Inventory: {ex.Message}"); }
        }
        
        if (saga.PaymentRef != null)
        {
            try { await _payment.RefundAsync(saga.OrderId, saga.PaymentRef); }
            catch (Exception ex) { errors.Add($"Payment: {ex.Message}"); }
        }
        
        UpdateStep(saga, errors.Count == 0
            ? OrderSagaStep.Compensated
            : OrderSagaStep.Failed);
        
        if (errors.Count > 0)
            saga.Error = string.Join("; ", errors);
        
        _db.Update(saga);
    }
    
    private void UpdateStep(OrderSaga saga, OrderSagaStep step)
    {
        saga.Step = step;
        saga.UpdatedAt = DateTime.UtcNow;
        _db.Update(saga);
    }
}
```

---

## Step 574: Event Sourcing Read Model

```csharp
// ============================================
// CQRS + Event Sourcing Read Model
// ============================================

public class OrderReadModel
{
    [PrimaryKey] public int OrderId { get; set; }
    public int CustomerId { get; set; }
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public int ItemCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public string? TrackingNumber { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? DeliveredAt { get; set; }
}

// Projector rebuilds read model from events
public class OrderProjector :
    IEventHandler<OrderCreatedEvent>,
    IEventHandler<OrderShippedEvent>,
    IEventHandler<PaymentCompletedEvent>
{
    private readonly SQLiteConnection _db;
    
    public OrderProjector(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<OrderReadModel>();
    }
    
    public Task HandleAsync(OrderCreatedEvent e, CancellationToken ct = default)
    {
        _db.InsertOrReplace(new OrderReadModel
        {
            OrderId = e.OrderId,
            CustomerId = e.CustomerId,
            Status = "Pending",
            TotalAmount = e.Amount,
            ItemCount = 0,
            CreatedAt = e.OccurredAt,
            UpdatedAt = e.OccurredAt
        });
        return Task.CompletedTask;
    }
    
    public Task HandleAsync(PaymentCompletedEvent e, CancellationToken ct = default)
    {
        var model = _db.Find<OrderReadModel>(e.OrderId);
        if (model == null) return Task.CompletedTask;
        
        model.Status = "Confirmed";
        model.UpdatedAt = e.OccurredAt;
        _db.Update(model);
        return Task.CompletedTask;
    }
    
    public Task HandleAsync(OrderShippedEvent e, CancellationToken ct = default)
    {
        var model = _db.Find<OrderReadModel>(e.OrderId);
        if (model == null) return Task.CompletedTask;
        
        model.Status = "Shipped";
        model.TrackingNumber = e.TrackingNumber;
        model.ShippedAt = e.OccurredAt;
        model.UpdatedAt = e.OccurredAt;
        _db.Update(model);
        return Task.CompletedTask;
    }
}
```

---

## Step 575: Message Queue Abstraction

```csharp
// ============================================
// Message Queue Abstraction
// ============================================

public interface IMessageQueue
{
    Task EnqueueAsync<T>(string queue, T message, CancellationToken ct = default);
    Task<T?> DequeueAsync<T>(string queue, CancellationToken ct = default);
    Task AckAsync(string queue, string messageId, CancellationToken ct = default);
    Task NackAsync(string queue, string messageId, CancellationToken ct = default);
}

// SQLite-backed implementation
public class SQLiteMessageQueue : IMessageQueue
{
    private readonly SQLiteConnection _db;
    
    public SQLiteMessageQueue(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<QueuedMessage>();
    }
    
    public Task EnqueueAsync<T>(string queue, T message, CancellationToken ct = default)
    {
        _db.Insert(new QueuedMessage
        {
            Queue = queue,
            MessageType = typeof(T).AssemblyQualifiedName!,
            Payload = System.Text.Json.JsonSerializer.Serialize(message),
            Status = MessageStatus.Pending,
            EnqueuedAt = DateTime.UtcNow
        });
        return Task.CompletedTask;
    }
    
    public Task<T?> DequeueAsync<T>(string queue, CancellationToken ct = default)
    {
        var msg = _db.Table<QueuedMessage>()
            .Where(m => m.Queue == queue && m.Status == MessageStatus.Pending)
            .OrderBy(m => m.EnqueuedAt)
            .FirstOrDefault();
        
        if (msg == null) return Task.FromResult<T?>(default);
        
        msg.Status = MessageStatus.Processing;
        msg.ProcessingStartedAt = DateTime.UtcNow;
        _db.Update(msg);
        
        var value = (T)System.Text.Json.JsonSerializer.Deserialize(msg.Payload, typeof(T))!;
        return Task.FromResult<T?>(value);
    }
    
    public Task AckAsync(string queue, string messageId, CancellationToken ct = default)
    {
        _db.Execute(
            "UPDATE QueuedMessage SET Status = ?, AckedAt = ? WHERE Id = ? AND Queue = ?",
            MessageStatus.Processed, DateTime.UtcNow, messageId, queue);
        return Task.CompletedTask;
    }
    
    public Task NackAsync(string queue, string messageId, CancellationToken ct = default)
    {
        _db.Execute(
            "UPDATE QueuedMessage SET Status = ?, Attempts = Attempts + 1 WHERE Id = ? AND Queue = ?",
            MessageStatus.Pending, messageId, queue);
        return Task.CompletedTask;
    }
}

public class QueuedMessage
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Queue { get; set; } = string.Empty;
    public string MessageType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public MessageStatus Status { get; set; }
    public int Attempts { get; set; }
    public DateTime EnqueuedAt { get; set; }
    public DateTime? ProcessingStartedAt { get; set; }
    public DateTime? AckedAt { get; set; }
}

public enum MessageStatus { Pending, Processing, Processed, DeadLetter }
```

---

## Step 576: Domain Event Dispatcher

```csharp
// ============================================
// Domain Event Dispatcher with Retry
// ============================================

public class DomainEventDispatcher
{
    private readonly EventBus _bus;
    private readonly IOutbox _outbox;
    private bool _useOutbox;
    
    public DomainEventDispatcher(EventBus bus, IOutbox outbox)
    {
        _bus = bus;
        _outbox = outbox;
    }
    
    public void UseOutboxPattern(bool value = true) => _useOutbox = value;
    
    public async Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct = default)
    {
        foreach (var @event in events)
        {
            if (_useOutbox)
            {
                AddToOutbox((dynamic)@event);
            }
            else
            {
                await PublishAsync((dynamic)@event, ct);
            }
        }
    }
    
    private void AddToOutbox<TEvent>(TEvent @event) where TEvent : IDomainEvent
        => _outbox.Add(@event);
    
    private Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct)
        where TEvent : IDomainEvent
        => _bus.PublishAsync(@event, ct);
}

// Use in Application Service
public class OrderApplicationServiceV2
{
    private readonly IOrderRepository _repo;
    private readonly DomainEventDispatcher _dispatcher;
    
    public OrderApplicationServiceV2(
        IOrderRepository repo, DomainEventDispatcher dispatcher)
    {
        _repo = repo;
        _dispatcher = dispatcher;
    }
    
    public async Task<int> PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);
        await _repo.SaveAsync(order);
        
        // Dispatch all domain events collected on aggregate
        await _dispatcher.DispatchAsync(order.DomainEvents);
        order.ClearDomainEvents();
        
        return order.Id;
    }
}
```

---

## Step 577: Event Replay

```csharp
// ============================================
// Event Replay for Read Model Rebuild
// ============================================

public class EventReplayService
{
    private readonly IEventStore _eventStore;
    private readonly EventBus _bus;
    
    public EventReplayService(IEventStore eventStore, EventBus bus)
    {
        _eventStore = eventStore;
        _bus = bus;
    }
    
    public async Task ReplayAllAsync(
        string aggregateType,
        CancellationToken ct = default,
        IProgress<int>? progress = null)
    {
        var events = await _eventStore.GetAllEventsAsync(aggregateType);
        var total = events.Count;
        
        for (int i = 0; i < total; i++)
        {
            if (ct.IsCancellationRequested) break;
            
            var serialized = events[i];
            var eventType = Type.GetType(serialized.EventType)
                ?? throw new InvalidOperationException($"Unknown: {serialized.EventType}");
            
            var @event = (IDomainEvent)System.Text.Json.JsonSerializer.Deserialize(
                serialized.Payload, eventType)!;
            
            await PublishAsync((dynamic)@event, ct);
            progress?.Report((i + 1) * 100 / total);
        }
    }
    
    private Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct)
        where TEvent : IDomainEvent
        => _bus.PublishAsync(@event, ct);
}

// Event replay ViewModel
public partial class EventReplayViewModel : ObservableObject
{
    private readonly EventReplayService _replay;
    
    [ObservableProperty] private int _progress;
    [ObservableProperty] private bool _isReplaying;
    [ObservableProperty] private string _status = "พร้อม";
    
    public EventReplayViewModel(EventReplayService replay) => _replay = replay;
    
    [RelayCommand]
    private async Task ReplayAsync(CancellationToken ct)
    {
        IsReplaying = true;
        Status = "กำลัง Replay Event...";
        
        try
        {
            var progressReporter = new Progress<int>(p =>
            {
                Progress = p;
                Status = $"Replay: {p}%";
            });
            
            await _replay.ReplayAllAsync("Order", ct, progressReporter);
            Status = "Replay เสร็จแล้ว";
        }
        catch (OperationCanceledException)
        {
            Status = "ยกเลิกแล้ว";
        }
        finally { IsReplaying = false; }
    }
}
```

---

## Step 578: Dead Letter Queue

```csharp
// ============================================
// Dead Letter Queue (DLQ)
// ============================================

public class DeadLetterQueue
{
    private readonly SQLiteConnection _db;
    
    public DeadLetterQueue(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<DeadLetterMessage>();
    }
    
    public void Add(string originalQueue, string payload, string error, string eventType)
    {
        _db.Insert(new DeadLetterMessage
        {
            OriginalQueue = originalQueue,
            EventType = eventType,
            Payload = payload,
            Error = error,
            FailedAt = DateTime.UtcNow
        });
    }
    
    public List<DeadLetterMessage> GetAll()
        => _db.Table<DeadLetterMessage>().OrderByDescending(m => m.FailedAt).ToList();
    
    public void Requeue(int id, SQLiteMessageQueue queue)
    {
        var msg = _db.Find<DeadLetterMessage>(id);
        if (msg == null) return;
        
        _db.Execute(
            "INSERT INTO QueuedMessage (Queue, MessageType, Payload, Status, EnqueuedAt) VALUES (?,?,?,?,?)",
            msg.OriginalQueue, msg.EventType, msg.Payload, MessageStatus.Pending, DateTime.UtcNow);
        
        _db.Delete(msg);
    }
}

public class DeadLetterMessage
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string OriginalQueue { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public string Error { get; set; } = string.Empty;
    public DateTime FailedAt { get; set; }
}

// DLQ monitor ViewModel
public partial class DlqMonitorViewModel : ObservableObject
{
    private readonly DeadLetterQueue _dlq;
    
    [ObservableProperty] private List<DeadLetterMessage> _messages = new();
    [ObservableProperty] private bool _hasMessages;
    
    public DlqMonitorViewModel(DeadLetterQueue dlq) => _dlq = dlq;
    
    public void Refresh()
    {
        Messages = _dlq.GetAll();
        HasMessages = Messages.Count > 0;
    }
}
```

---

## Step 579: Event Schema Registry

```csharp
// ============================================
// Event Schema Versioning
// ============================================

public interface IEventUpgrader<TFrom, TTo>
    where TFrom : IDomainEvent
    where TTo : IDomainEvent
{
    TTo Upgrade(TFrom old);
}

// V1 → V2 migration
public record OrderCreatedEventV1(
    Guid EventId, DateTime OccurredAt,
    int OrderId, int CustomerId, decimal Amount) : IDomainEvent;

public record OrderCreatedEventV2(
    Guid EventId, DateTime OccurredAt,
    int OrderId, int CustomerId, decimal Amount,
    string Currency, string Channel) : IDomainEvent;

public class OrderCreatedV1ToV2Upgrader :
    IEventUpgrader<OrderCreatedEventV1, OrderCreatedEventV2>
{
    public OrderCreatedEventV2 Upgrade(OrderCreatedEventV1 old)
        => new(old.EventId, old.OccurredAt,
               old.OrderId, old.CustomerId, old.Amount,
               "THB",       // default
               "App");      // default
}

public class EventSchemaRegistry
{
    private readonly Dictionary<string, Func<string, IDomainEvent>> _deserializers = new();
    private readonly Dictionary<string, Func<IDomainEvent, IDomainEvent>> _upgraders = new();
    
    public void Register<TEvent>(string version) where TEvent : IDomainEvent
    {
        var key = $"{typeof(TEvent).Name}:{version}";
        _deserializers[key] = payload =>
            (IDomainEvent)System.Text.Json.JsonSerializer.Deserialize<TEvent>(payload)!;
    }
    
    public void RegisterUpgrade<TFrom, TTo>(
        IEventUpgrader<TFrom, TTo> upgrader)
        where TFrom : IDomainEvent
        where TTo : IDomainEvent
    {
        var key = typeof(TFrom).Name;
        _upgraders[key] = e => upgrader.Upgrade((TFrom)e);
    }
    
    public IDomainEvent Deserialize(string eventType, string version, string payload)
    {
        var key = $"{eventType}:{version}";
        if (!_deserializers.TryGetValue(key, out var deserializer))
            throw new InvalidOperationException($"No deserializer for {key}");
        
        var @event = deserializer(payload);
        
        // Upgrade if needed
        while (_upgraders.TryGetValue(@event.GetType().Name, out var upgrader))
            @event = upgrader(@event);
        
        return @event;
    }
}
```

---

## Step 580: Event-Driven Testing

```csharp
// ============================================
// Testing Event-Driven Code
// ============================================

public class EventBusTests
{
    [Fact]
    public async Task Publish_CallsAllRegisteredHandlers()
    {
        var services = new ServiceCollection();
        var handler1Called = false;
        var handler2Called = false;
        
        services.AddSingleton<IEventHandler<OrderCreatedEvent>>(
            new LambdaEventHandler<OrderCreatedEvent>(_ => { handler1Called = true; return Task.CompletedTask; }));
        
        var sp = services.BuildServiceProvider();
        var bus = new EventBus(sp);
        
        var @event = new OrderCreatedEvent(Guid.NewGuid(), DateTime.UtcNow, 1, 1, 100m);
        await bus.PublishAsync(@event);
        
        handler1Called.Should().BeTrue();
    }
    
    [Fact]
    public async Task OrderSaga_WhenPaymentFails_Compensates()
    {
        var payment = new Mock<IPaymentService>();
        payment.Setup(p => p.ChargeAsync(It.IsAny<int>()))
            .ThrowsAsync(new Exception("Payment declined"));
        
        var inventory = new Mock<IInventoryService>();
        var shipment = new Mock<IShipmentService>();
        var orders = new Mock<IOrderRepository>();
        
        var db = new SQLiteConnection(":memory:");
        var saga = new OrderSagaOrchestrator(db, payment.Object,
            inventory.Object, shipment.Object, orders.Object);
        
        await saga.StartAsync(orderId: 1);
        
        var savedSaga = db.Table<OrderSaga>().First();
        savedSaga.Step.Should().Be(OrderSagaStep.Compensated);
        
        // Inventory and shipment should NOT have been called
        inventory.Verify(i => i.ReserveAsync(It.IsAny<int>(), default), Times.Never);
        shipment.Verify(s => s.ScheduleAsync(It.IsAny<int>()), Times.Never);
    }
    
    [Fact]
    public async Task Outbox_FlushedAfterTransaction()
    {
        var db = new SQLiteConnection(":memory:");
        var bus = new Mock<EventBus>(db); // simplified
        var outbox = new SQLiteOutbox(db, bus.Object);
        
        outbox.Add(new OrderCreatedEvent(Guid.NewGuid(), DateTime.UtcNow, 1, 1, 100m));
        
        var pending = db.Table<OutboxMessage>().Where(m => !m.Processed).Count();
        pending.Should().Be(1);
    }
}

public class LambdaEventHandler<TEvent> : IEventHandler<TEvent>
    where TEvent : IDomainEvent
{
    private readonly Func<TEvent, Task> _handler;
    public LambdaEventHandler(Func<TEvent, Task> handler) => _handler = handler;
    public Task HandleAsync(TEvent @event, CancellationToken ct = default) => _handler(@event);
}
```

---

## สรุป Part 58

ใน Part 58 เราได้เรียนรู้:

1. **Event Bus** - In-process Pub/Sub
2. **Outbox Pattern** - Transactional consistency
3. **Saga Pattern** - Long-running distributed process
4. **Read Model Projection** - CQRS read side
5. **Message Queue** - SQLite-backed
6. **Event Dispatcher** - Domain event routing
7. **Event Replay** - Read model rebuild
8. **Dead Letter Queue** - Failed message handling
9. **Schema Registry** - Event versioning
10. **Event-Driven Testing** - Saga/handler tests

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 58 | Steps 571-580*
