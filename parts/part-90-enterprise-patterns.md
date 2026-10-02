# Part 90: Enterprise Patterns
## Steps 891-900: Multi-Layer Caching, Event Sourcing, SAGA Pattern, Distributed Transactions

---

## Step 891: Multi-Layer Caching

```csharp
// ============================================
// L1 (Memory) → L2 (Disk) → L3 (Redis/API) Cache
// ============================================

public interface IMultiLayerCache
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, CacheOptions? options = null);
    Task InvalidateAsync(string key);
    Task InvalidatePatternAsync(string pattern);
}

public record CacheOptions(
    TimeSpan? L1Ttl = null,
    TimeSpan? L2Ttl = null,
    string? Tag = null);

public class MultiLayerCache : IMultiLayerCache
{
    private readonly IMemoryCache _l1;
    private readonly DiskCache _l2;
    private readonly RemoteCache _l3;
    private static readonly TimeSpan DefaultL1 = TimeSpan.FromMinutes(5);
    private static readonly TimeSpan DefaultL2 = TimeSpan.FromHours(1);
    
    public MultiLayerCache(IMemoryCache l1, DiskCache l2, RemoteCache l3)
    {
        _l1 = l1; _l2 = l2; _l3 = l3;
    }
    
    public async Task<T?> GetAsync<T>(string key)
    {
        // L1 hit
        if (_l1.TryGetValue(key, out T? l1Value)) return l1Value;
        
        // L2 hit → promote to L1
        var l2Value = await _l2.GetAsync<T>(key);
        if (l2Value != null)
        {
            _l1.Set(key, l2Value, DefaultL1);
            return l2Value;
        }
        
        // L3 hit → promote to L1 + L2
        var l3Value = await _l3.GetAsync<T>(key);
        if (l3Value != null)
        {
            _l1.Set(key, l3Value, DefaultL1);
            await _l2.SetAsync(key, l3Value, DefaultL2);
        }
        
        return l3Value;
    }
    
    public async Task SetAsync<T>(string key, T value, CacheOptions? options = null)
    {
        var l1Ttl = options?.L1Ttl ?? DefaultL1;
        var l2Ttl = options?.L2Ttl ?? DefaultL2;
        
        _l1.Set(key, value, l1Ttl);
        await _l2.SetAsync(key, value, l2Ttl);
        await _l3.SetAsync(key, value);
    }
    
    public Task InvalidateAsync(string key)
    {
        _l1.Remove(key);
        return Task.WhenAll(_l2.RemoveAsync(key), _l3.RemoveAsync(key));
    }
    
    public Task InvalidatePatternAsync(string pattern)
    {
        // Invalidate all keys matching pattern (e.g., "restaurant:*")
        return Task.WhenAll(_l2.RemoveByPatternAsync(pattern), _l3.RemoveByPatternAsync(pattern));
    }
}

// Disk cache layer
public class DiskCache
{
    private readonly string _dir;
    
    public DiskCache(string cacheDir)
    {
        _dir = cacheDir;
        Directory.CreateDirectory(_dir);
    }
    
    public async Task<T?> GetAsync<T>(string key)
    {
        var path = GetPath(key);
        if (!File.Exists(path)) return default;
        
        var entry = await System.Text.Json.JsonSerializer
            .DeserializeAsync<CacheEntry<T>>(File.OpenRead(path));
        
        if (entry == null || entry.ExpiresAt < DateTime.UtcNow)
        {
            File.Delete(path);
            return default;
        }
        
        return entry.Value;
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan ttl)
    {
        var entry = new CacheEntry<T>(value, DateTime.UtcNow.Add(ttl));
        await using var file = File.Create(GetPath(key));
        await System.Text.Json.JsonSerializer.SerializeAsync(file, entry);
    }
    
    public Task RemoveAsync(string key) { try { File.Delete(GetPath(key)); } catch { } return Task.CompletedTask; }
    
    public Task RemoveByPatternAsync(string pattern)
    {
        var prefix = pattern.TrimEnd('*');
        foreach (var file in Directory.GetFiles(_dir, $"{prefix}*.json"))
            try { File.Delete(file); } catch { }
        return Task.CompletedTask;
    }
    
    private string GetPath(string key)
    {
        var safe = Uri.EscapeDataString(key);
        return Path.Combine(_dir, $"{safe}.json");
    }
    
    private record CacheEntry<T>(T? Value, DateTime ExpiresAt);
}

// Remote cache stub
public class RemoteCache
{
    private readonly HttpClient _http;
    public RemoteCache(HttpClient http) => _http = http;
    public async Task<T?> GetAsync<T>(string key) => await Task.FromResult<T?>(default);
    public Task SetAsync<T>(string key, T value) => Task.CompletedTask;
    public Task RemoveAsync(string key) => Task.CompletedTask;
    public Task RemoveByPatternAsync(string pattern) => Task.CompletedTask;
}
```

---

## Step 892: Event Sourcing

```csharp
// ============================================
// Event Sourcing for Order Aggregate
// ============================================

public interface IDomainEvent
{
    string AggregateId { get; }
    DateTime OccurredAt { get; }
    int Version { get; }
}

public abstract class EventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommittedEvents = new();
    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommittedEvents;
    public int CurrentVersion { get; protected set; }
    
    protected void Apply(IDomainEvent domainEvent)
    {
        _uncommittedEvents.Add(domainEvent);
        ApplyEvent(domainEvent);
        CurrentVersion++;
    }
    
    protected abstract void ApplyEvent(IDomainEvent domainEvent);
    
    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
    
    public void Rehydrate(IEnumerable<IDomainEvent> events)
    {
        foreach (var e in events)
        {
            ApplyEvent(e);
            CurrentVersion++;
        }
    }
}

// Order domain events
public record OrderCreatedEvent(
    string AggregateId, string CustomerId, List<OrderLine> Lines,
    DateTime OccurredAt, int Version) : IDomainEvent;

public record OrderConfirmedEvent(
    string AggregateId, DateTime OccurredAt, int Version) : IDomainEvent;

public record OrderCancelledEvent(
    string AggregateId, string Reason, DateTime OccurredAt, int Version) : IDomainEvent;

public record PromoAppliedEvent(
    string AggregateId, string Code, decimal DiscountAmount,
    DateTime OccurredAt, int Version) : IDomainEvent;

// Event-sourced Order
public class EventSourcedOrder : EventSourcedAggregate
{
    public string Id { get; private set; } = "";
    public string CustomerId { get; private set; } = "";
    public List<OrderLine> Lines { get; private set; } = new();
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }
    
    // Factory
    public static EventSourcedOrder Create(string customerId, List<OrderLine> lines)
    {
        var order = new EventSourcedOrder();
        order.Apply(new OrderCreatedEvent(
            Guid.NewGuid().ToString(), customerId, lines, DateTime.UtcNow, 0));
        return order;
    }
    
    public void Confirm()
    {
        if (Status != OrderStatus.Pending) throw new DomainException("Cannot confirm non-pending order");
        Apply(new OrderConfirmedEvent(Id, DateTime.UtcNow, CurrentVersion));
    }
    
    protected override void ApplyEvent(IDomainEvent domainEvent)
    {
        switch (domainEvent)
        {
            case OrderCreatedEvent e:
                Id = e.AggregateId;
                CustomerId = e.CustomerId;
                Lines = e.Lines;
                Total = Lines.Sum(l => l.UnitPrice.Amount * l.Quantity);
                Status = OrderStatus.Pending;
                break;
            
            case OrderConfirmedEvent:
                Status = OrderStatus.Confirmed;
                break;
            
            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;
            
            case PromoAppliedEvent e:
                Total -= e.DiscountAmount;
                break;
        }
    }
}
```

---

## Step 893: Event Store

```csharp
// ============================================
// SQLite Event Store
// ============================================

public class EventStore
{
    private readonly SQLiteAsyncConnection _db;
    private readonly IEventBus _eventBus;
    
    public EventStore(SQLiteAsyncConnection db, IEventBus eventBus)
    {
        _db = db;
        _eventBus = eventBus;
        _db.CreateTableAsync<StoredEvent>().Wait();
    }
    
    public async Task AppendAsync(string aggregateId, IEnumerable<IDomainEvent> events)
    {
        var stored = events.Select(e => new StoredEvent
        {
            Id = Guid.NewGuid().ToString(),
            AggregateId = aggregateId,
            EventType = e.GetType().Name,
            Payload = System.Text.Json.JsonSerializer.Serialize(e, e.GetType()),
            Version = e.Version,
            OccurredAt = e.OccurredAt
        }).ToList();
        
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var s in stored)
                conn.Insert(s);
        });
        
        // Publish to event bus (projections, notifications)
        foreach (var e in events)
            _eventBus.Publish(e);
    }
    
    public async Task<List<IDomainEvent>> GetEventsAsync(string aggregateId)
    {
        var stored = await _db.Table<StoredEvent>()
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        return stored.Select(s => Deserialize(s)).OfType<IDomainEvent>().ToList();
    }
    
    private IDomainEvent? Deserialize(StoredEvent stored)
    {
        var type = Type.GetType($"FoodDelivery.Domain.Events.{stored.EventType}");
        if (type == null) return null;
        return System.Text.Json.JsonSerializer.Deserialize(stored.Payload, type) as IDomainEvent;
    }
}

public class StoredEvent
{
    [PrimaryKey] public string Id { get; set; } = "";
    [Indexed] public string AggregateId { get; set; } = "";
    public string EventType { get; set; } = "";
    public string Payload { get; set; } = "";
    public int Version { get; set; }
    public DateTime OccurredAt { get; set; }
}

// Repository using event store
public class EventSourcedOrderRepository
{
    private readonly EventStore _store;
    
    public EventSourcedOrderRepository(EventStore store) => _store = store;
    
    public async Task<EventSourcedOrder?> GetAsync(string orderId)
    {
        var events = await _store.GetEventsAsync(orderId);
        if (!events.Any()) return null;
        
        var order = new EventSourcedOrder();
        order.Rehydrate(events);
        return order;
    }
    
    public async Task SaveAsync(EventSourcedOrder order)
    {
        await _store.AppendAsync(order.Id, order.UncommittedEvents);
        order.MarkEventsAsCommitted();
    }
}
```

---

## Step 894: SAGA Pattern

```csharp
// ============================================
// Order Placement SAGA (Distributed Transaction)
// ============================================

public enum SagaStatus { Started, Completed, Failed, Compensating, Compensated }

public abstract class Saga<TState> where TState : new()
{
    public string SagaId { get; } = Guid.NewGuid().ToString();
    public SagaStatus Status { get; protected set; } = SagaStatus.Started;
    public TState State { get; protected set; } = new();
    
    protected abstract Task ExecuteAsync();
    protected abstract Task CompensateAsync();
    
    public async Task RunAsync()
    {
        try
        {
            await ExecuteAsync();
            Status = SagaStatus.Completed;
        }
        catch
        {
            Status = SagaStatus.Compensating;
            try
            {
                await CompensateAsync();
                Status = SagaStatus.Compensated;
            }
            catch
            {
                Status = SagaStatus.Failed;
            }
            throw;
        }
    }
}

public class OrderSagaState
{
    public string? OrderId { get; set; }
    public string? PaymentTransactionId { get; set; }
    public bool InventoryReserved { get; set; }
    public bool PaymentCharged { get; set; }
    public bool NotificationSent { get; set; }
}

public class PlaceOrderSaga : Saga<OrderSagaState>
{
    private readonly PlaceOrderCommand _command;
    private readonly IInventoryService _inventory;
    private readonly IPaymentGateway _payment;
    private readonly IOrderRepository _orders;
    private readonly ILocalNotificationService _notifications;
    
    public PlaceOrderSaga(
        PlaceOrderCommand command,
        IInventoryService inventory,
        IPaymentGateway payment,
        IOrderRepository orders,
        ILocalNotificationService notifications)
    {
        _command = command;
        _inventory = inventory;
        _payment = payment;
        _orders = orders;
        _notifications = notifications;
    }
    
    protected override async Task ExecuteAsync()
    {
        // Step 1: Reserve inventory
        await _inventory.ReserveAsync(_command.Items);
        State.InventoryReserved = true;
        
        // Step 2: Create order
        var order = Order.Create(new CustomerId(_command.CustomerId), _command.Items.Select(i =>
            new OrderLine(i.MenuItemId, i.Name, new Money(i.Price, "THB"), i.Quantity)).ToList());
        State.OrderId = order.Id.ToString();
        
        // Step 3: Charge payment
        var payment = await _payment.ChargeAsync(new PaymentRequest(
            _command.CustomerId, order.Total.Amount, "THB",
            _command.PaymentMethodId, $"Order {order.Id}", order.Id.ToString()));
        
        if (!payment.Success) throw new PaymentException(payment.ErrorMessage);
        State.PaymentTransactionId = payment.TransactionId;
        State.PaymentCharged = true;
        
        // Step 4: Save order
        await _orders.SaveAsync(order);
        
        // Step 5: Send confirmation
        await _notifications.SendAsync("ออเดอร์ได้รับการยืนยัน ✅", $"ออเดอร์ #{order.Id}");
        State.NotificationSent = true;
    }
    
    protected override async Task CompensateAsync()
    {
        // Reverse in opposite order
        
        if (State.PaymentCharged && State.PaymentTransactionId != null)
            await _payment.RefundAsync(State.PaymentTransactionId, 0, "Order saga failed");
        
        if (State.InventoryReserved)
            await _inventory.ReleaseAsync(_command.Items);
    }
}
```

---

## Step 895: CQRS Projection

```csharp
// ============================================
// Read Model Projections from Events
// ============================================

public class OrderProjection : IEventHandler<OrderCreatedEvent>,
    IEventHandler<OrderConfirmedEvent>, IEventHandler<OrderCancelledEvent>
{
    private readonly SQLiteAsyncConnection _db;
    
    public OrderProjection(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<OrderReadModel>().Wait();
    }
    
    public async Task HandleAsync(OrderCreatedEvent evt)
    {
        await _db.InsertOrReplaceAsync(new OrderReadModel
        {
            Id = evt.AggregateId,
            CustomerId = evt.CustomerId,
            Total = evt.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity),
            Status = "Pending",
            ItemCount = evt.Lines.Sum(l => l.Quantity),
            CreatedAt = evt.OccurredAt,
            UpdatedAt = evt.OccurredAt
        });
    }
    
    public async Task HandleAsync(OrderConfirmedEvent evt)
    {
        var model = await _db.GetAsync<OrderReadModel>(evt.AggregateId);
        if (model == null) return;
        model.Status = "Confirmed";
        model.UpdatedAt = evt.OccurredAt;
        await _db.UpdateAsync(model);
    }
    
    public async Task HandleAsync(OrderCancelledEvent evt)
    {
        var model = await _db.GetAsync<OrderReadModel>(evt.AggregateId);
        if (model == null) return;
        model.Status = "Cancelled";
        model.UpdatedAt = evt.OccurredAt;
        await _db.UpdateAsync(model);
    }
}

public class OrderReadModel
{
    [PrimaryKey] public string Id { get; set; } = "";
    [Indexed] public string CustomerId { get; set; } = "";
    public decimal Total { get; set; }
    public string Status { get; set; } = "";
    public int ItemCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

---

## Step 896: Outbox Pattern

```csharp
// ============================================
// Transactional Outbox (reliable event publishing)
// ============================================

public class OutboxMessage
{
    [PrimaryKey] public string Id { get; set; } = "";
    public string EventType { get; set; } = "";
    public string Payload { get; set; } = "";
    public bool IsProcessed { get; set; }
    public int RetryCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
}

public class OutboxRepository
{
    private readonly SQLiteAsyncConnection _db;
    
    public OutboxRepository(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<OutboxMessage>().Wait();
    }
    
    // Called within same transaction as business operation
    public Task AddAsync(IDomainEvent evt)
        => _db.InsertAsync(new OutboxMessage
        {
            Id = Guid.NewGuid().ToString(),
            EventType = evt.GetType().Name,
            Payload = System.Text.Json.JsonSerializer.Serialize(evt, evt.GetType()),
            IsProcessed = false,
            CreatedAt = DateTime.UtcNow
        });
    
    public Task<List<OutboxMessage>> GetUnprocessedAsync(int limit = 50)
        => _db.Table<OutboxMessage>()
            .Where(m => !m.IsProcessed && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(limit)
            .ToListAsync();
    
    public Task MarkProcessedAsync(string id)
        => _db.ExecuteAsync(
            "UPDATE OutboxMessages SET IsProcessed=1, ProcessedAt=? WHERE Id=?",
            DateTime.UtcNow, id);
    
    public Task IncrementRetryAsync(string id)
        => _db.ExecuteAsync(
            "UPDATE OutboxMessages SET RetryCount=RetryCount+1 WHERE Id=?", id);
}

public class OutboxProcessor
{
    private readonly OutboxRepository _outbox;
    private readonly IEventBus _eventBus;
    private readonly ILogger<OutboxProcessor> _logger;
    private readonly Timer _timer;
    
    public OutboxProcessor(
        OutboxRepository outbox, IEventBus eventBus, ILogger<OutboxProcessor> logger)
    {
        _outbox = outbox;
        _eventBus = eventBus;
        _logger = logger;
        _timer = new Timer(async _ => await ProcessAsync(), null,
            TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(5));
    }
    
    private async Task ProcessAsync()
    {
        var messages = await _outbox.GetUnprocessedAsync();
        
        foreach (var msg in messages)
        {
            try
            {
                var type = Type.GetType($"FoodDelivery.Domain.Events.{msg.EventType}");
                if (type == null) continue;
                
                var evt = System.Text.Json.JsonSerializer.Deserialize(msg.Payload, type) as IDomainEvent;
                if (evt != null) _eventBus.Publish(evt);
                
                await _outbox.MarkProcessedAsync(msg.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Outbox processing failed for {Id}", msg.Id);
                await _outbox.IncrementRetryAsync(msg.Id);
            }
        }
    }
}
```

---

## Step 897: Domain-Driven Events Read Model

```csharp
// ============================================
// Analytics Read Model from Event Stream
// ============================================

public class OrderAnalyticsProjection : IEventHandler<OrderCreatedEvent>
{
    private readonly SQLiteAsyncConnection _db;
    
    public OrderAnalyticsProjection(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<DailyOrderStats>().Wait();
    }
    
    public async Task HandleAsync(OrderCreatedEvent evt)
    {
        var date = evt.OccurredAt.Date;
        var total = evt.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity);
        
        var existing = await _db.Table<DailyOrderStats>()
            .FirstOrDefaultAsync(s => s.Date == date);
        
        if (existing == null)
        {
            await _db.InsertAsync(new DailyOrderStats
            {
                Date = date,
                OrderCount = 1,
                Revenue = total,
                UniqueCustomers = 1
            });
        }
        else
        {
            existing.OrderCount++;
            existing.Revenue += total;
            await _db.UpdateAsync(existing);
        }
    }
}

public class DailyOrderStats
{
    [PrimaryKey] public DateTime Date { get; set; }
    public int OrderCount { get; set; }
    public decimal Revenue { get; set; }
    public int UniqueCustomers { get; set; }
}
```

---

## Step 898: Snapshot Strategy

```csharp
// ============================================
// Snapshots for Fast Aggregate Loading
// ============================================

public class SnapshotStore
{
    private readonly SQLiteAsyncConnection _db;
    private const int SnapshotInterval = 50; // snapshot every 50 events
    
    public SnapshotStore(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<AggregateSnapshot>().Wait();
    }
    
    public async Task<(EventSourcedOrder? Aggregate, int FromVersion)> LoadAsync(string aggregateId)
    {
        var snapshot = await _db.Table<AggregateSnapshot>()
            .Where(s => s.AggregateId == aggregateId)
            .OrderByDescending(s => s.Version)
            .FirstOrDefaultAsync();
        
        if (snapshot == null) return (null, 0);
        
        var order = System.Text.Json.JsonSerializer
            .Deserialize<EventSourcedOrder>(snapshot.Payload);
        
        return (order, snapshot.Version);
    }
    
    public async Task SaveIfNeededAsync(EventSourcedOrder order)
    {
        if (order.CurrentVersion % SnapshotInterval != 0) return;
        
        await _db.InsertOrReplaceAsync(new AggregateSnapshot
        {
            Id = $"{order.Id}-{order.CurrentVersion}",
            AggregateId = order.Id,
            Version = order.CurrentVersion,
            Payload = System.Text.Json.JsonSerializer.Serialize(order),
            CreatedAt = DateTime.UtcNow
        });
    }
}

public class AggregateSnapshot
{
    [PrimaryKey] public string Id { get; set; } = "";
    [Indexed] public string AggregateId { get; set; } = "";
    public int Version { get; set; }
    public string Payload { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}
```

---

## Step 899: Integration Event Pipeline

```csharp
// ============================================
// Integration Event Pipeline (cross-service)
// ============================================

public interface IIntegrationEvent
{
    string EventId { get; }
    string EventType { get; }
    DateTime OccurredAt { get; }
}

public class IntegrationEventPipeline
{
    private readonly List<Func<IIntegrationEvent, Task>> _middlewares = new();
    private Func<IIntegrationEvent, Task>? _handler;
    
    public IntegrationEventPipeline Use(Func<IIntegrationEvent, Func<Task>, Task> middleware)
    {
        _middlewares.Add(evt =>
        {
            Func<Task> next = _handler != null ? () => _handler(evt) : () => Task.CompletedTask;
            return middleware(evt, next);
        });
        return this;
    }
    
    public IntegrationEventPipeline Handle(Func<IIntegrationEvent, Task> handler)
    {
        _handler = handler;
        return this;
    }
    
    public Task ExecuteAsync(IIntegrationEvent evt)
    {
        if (!_middlewares.Any()) return _handler?.Invoke(evt) ?? Task.CompletedTask;
        
        // Build chain
        Func<IIntegrationEvent, Task>? chain = _handler;
        for (int i = _middlewares.Count - 1; i >= 0; i--)
        {
            var current = _middlewares[i];
            var innerChain = chain;
            chain = e =>
            {
                Func<Task> next = innerChain != null ? () => innerChain(e) : () => Task.CompletedTask;
                return current(e);
            };
        }
        
        return chain?.Invoke(evt) ?? Task.CompletedTask;
    }
}
```

---

## Step 900: Enterprise Test Suite

```csharp
// ============================================
// Event Sourcing Tests
// ============================================

[TestFixture]
public class EventSourcingTests
{
    [Test]
    public void EventSourcedOrder_Create_ProducesCreatedEvent()
    {
        var lines = new List<OrderLine>
        {
            new("item-1", "ข้าวผัด", new Money(80, "THB"), 2)
        };
        
        var order = EventSourcedOrder.Create("cust-1", lines);
        
        Assert.That(order.UncommittedEvents.Count, Is.EqualTo(1));
        Assert.That(order.UncommittedEvents[0], Is.InstanceOf<OrderCreatedEvent>());
        Assert.That(order.Total, Is.EqualTo(160));
        Assert.That(order.Status, Is.EqualTo(OrderStatus.Pending));
    }
    
    [Test]
    public void EventSourcedOrder_Confirm_ProducesConfirmedEvent()
    {
        var order = CreateTestOrder();
        order.Confirm();
        
        var events = order.UncommittedEvents;
        Assert.That(events.Any(e => e is OrderConfirmedEvent), Is.True);
        Assert.That(order.Status, Is.EqualTo(OrderStatus.Confirmed));
    }
    
    [Test]
    public void EventSourcedOrder_Rehydrate_RebuildsState()
    {
        var original = CreateTestOrder();
        original.Confirm();
        
        var events = original.UncommittedEvents.ToList();
        
        var rehydrated = new EventSourcedOrder();
        rehydrated.Rehydrate(events);
        
        Assert.That(rehydrated.Id, Is.EqualTo(original.Id));
        Assert.That(rehydrated.Status, Is.EqualTo(OrderStatus.Confirmed));
        Assert.That(rehydrated.Total, Is.EqualTo(original.Total));
    }
    
    [Test]
    public async Task MultiLayerCache_L2Hit_PromotesToL1()
    {
        var l1 = new MemoryCache(new MemoryCacheOptions());
        var l2 = new DiskCache(Path.Combine(Path.GetTempPath(), "test-cache"));
        var l3 = new RemoteCache(new HttpClient());
        
        var cache = new MultiLayerCache(l1, l2, l3);
        
        // Seed L2
        await l2.SetAsync("restaurant:1", "test-data", TimeSpan.FromHours(1));
        
        // Get should promote to L1
        var result = await cache.GetAsync<string>("restaurant:1");
        Assert.That(result, Is.EqualTo("test-data"));
        
        // L1 should now have it
        Assert.That(l1.TryGetValue<string>("restaurant:1", out _), Is.True);
    }
    
    private static EventSourcedOrder CreateTestOrder() =>
        EventSourcedOrder.Create("cust-1", new List<OrderLine>
        {
            new("item-1", "ข้าวผัด", new Money(80, "THB"), 1)
        });
}
```

---

## สรุป Part 90

ใน Part 90 เราได้เรียนรู้:

1. **Multi-Layer Cache** - L1 (Memory) → L2 (Disk) → L3 (Remote), promotion on hit
2. **Event Sourcing** - IDomainEvent, EventSourcedAggregate, Apply/Rehydrate
3. **Event Store** - StoredEvent SQLite table, serialize/deserialize by type name
4. **SAGA Pattern** - Distributed transaction with compensating transactions
5. **CQRS Projections** - Read model updated by event handlers
6. **Outbox Pattern** - Transactional event publishing, retry logic
7. **Analytics Projection** - DailyOrderStats from event stream
8. **Snapshots** - Periodic snapshots every 50 events for fast loading
9. **Integration Events** - Cross-service event pipeline with middleware
10. **Enterprise Tests** - Event create, confirm, rehydrate, cache promotion

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 90 | Steps 891-900*
