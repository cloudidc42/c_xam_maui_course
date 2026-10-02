# Part 98: Career Guide & Portfolio Development
## Steps 971-980: Portfolio App, Code Review Skills, Open Source, Interview Prep, Career Path

---

## Step 971: Portfolio App Architecture

```csharp
// ============================================
// Showcase App: Food Delivery Full Architecture
// ============================================

// This is the capstone architecture summary of everything learned

// SOLUTION STRUCTURE:
// FoodDelivery.sln
// ├── FoodDelivery.App          (MAUI UI project)
// ├── FoodDelivery.Core         (Domain models, interfaces)
// ├── FoodDelivery.Infrastructure (Repositories, API clients)
// ├── FoodDelivery.Tests.Unit   (NUnit, NSubstitute)
// ├── FoodDelivery.Tests.Integration (SQLite, real HTTP)
// └── FoodDelivery.Tests.UI     (Appium, Playwright)

// Core/Domain/OrderAggregate.cs
namespace FoodDelivery.Core.Domain;

public sealed class Order
{
    public OrderId Id { get; private set; }
    public UserId CustomerId { get; private set; }
    public RestaurantId RestaurantId { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; }
    public DeliveryAddress DeliveryAddress { get; private set; }
    public DateTime CreatedAt { get; private set; }
    
    private readonly List<OrderItem> _items = new();
    private readonly List<IDomainEvent> _events = new();
    
    private Order() { } // EF/SQLite constructor
    
    public static Order Create(
        UserId customerId,
        RestaurantId restaurantId,
        DeliveryAddress address,
        IEnumerable<OrderItem> items)
    {
        var order = new Order
        {
            Id = new OrderId(Guid.NewGuid()),
            CustomerId = customerId,
            RestaurantId = restaurantId,
            DeliveryAddress = address,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        
        foreach (var item in items)
            order._items.Add(item);
        
        order.Total = order.CalculateTotal();
        order._events.Add(new OrderCreatedEvent(order.Id, order.Total));
        
        return order;
    }
    
    public Result Confirm()
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure($"Cannot confirm order in {Status} status");
        
        Status = OrderStatus.Confirmed;
        _events.Add(new OrderConfirmedEvent(Id, DateTime.UtcNow));
        return Result.Ok();
    }
    
    private Money CalculateTotal()
        => new Money(_items.Sum(i => i.UnitPrice.Amount * i.Quantity));
    
    public IReadOnlyList<IDomainEvent> GetUncommittedEvents() => _events.AsReadOnly();
    public void ClearEvents() => _events.Clear();
}

// Value objects
public record OrderId(Guid Value);
public record UserId(Guid Value);
public record RestaurantId(Guid Value);
public record Money(decimal Amount);
public record OrderItem(string Name, Money UnitPrice, int Quantity);
public record DeliveryAddress(string Line1, string District, string Province, string PostalCode);

public enum OrderStatus { Pending, Confirmed, Preparing, PickedUp, Delivered, Cancelled }

public record Result(bool IsOk, string? Error)
{
    public static Result Ok() => new(true, null);
    public static Result Failure(string error) => new(false, error);
}
```

---

## Step 972: Clean Architecture Layer Boundaries

```csharp
// ============================================
// Dependency Rule: outer layers depend on inner
// ============================================

// ┌─────────────────────────────────────────────┐
// │  UI (MAUI Pages, ViewModels)                │
// │  ┌───────────────────────────────────────┐  │
// │  │  Application (Use Cases, Commands)    │  │
// │  │  ┌───────────────────────────────┐    │  │
// │  │  │  Domain (Entities, Events)    │    │  │
// │  │  └───────────────────────────────┘    │  │
// │  │  ┌───────────────────────────────┐    │  │
// │  │  │  Infrastructure (DB, API)     │    │  │
// │  └──┴───────────────────────────────┴────┘  │
// └─────────────────────────────────────────────┘

// Application/UseCases/PlaceOrderUseCase.cs
public class PlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    private readonly IMenuRepository _menu;
    private readonly IPaymentGateway _payment;
    private readonly IEventPublisher _events;
    
    public PlaceOrderUseCase(
        IOrderRepository orders,
        IMenuRepository menu,
        IPaymentGateway payment,
        IEventPublisher events)
    {
        _orders = orders;
        _menu = menu;
        _payment = payment;
        _events = events;
    }
    
    public async Task<PlaceOrderResult> ExecuteAsync(PlaceOrderCommand cmd)
    {
        // 1. Validate menu items are still available
        var menuItems = await _menu.GetByIdsAsync(cmd.ItemIds);
        var unavailable = menuItems.Where(m => !m.IsAvailable).ToList();
        if (unavailable.Any())
            return PlaceOrderResult.Failure($"Items unavailable: {string.Join(", ", unavailable.Select(i => i.Name))}");
        
        // 2. Create order
        var items = menuItems.Select(m => new OrderItem(
            m.Name, new Money(m.Price),
            cmd.Quantities.GetValueOrDefault(m.Id, 1)));
        
        var order = Order.Create(
            new UserId(cmd.UserId),
            new RestaurantId(cmd.RestaurantId),
            cmd.DeliveryAddress,
            items);
        
        // 3. Charge payment
        var payment = await _payment.ChargeAsync(order.Total.Amount, cmd.PaymentToken);
        if (!payment.Succeeded)
            return PlaceOrderResult.Failure("Payment failed: " + payment.Error);
        
        // 4. Confirm and save
        order.Confirm();
        await _orders.SaveAsync(order);
        
        // 5. Publish events
        foreach (var evt in order.GetUncommittedEvents())
            await _events.PublishAsync(evt);
        
        order.ClearEvents();
        
        return PlaceOrderResult.Success(order.Id);
    }
}

public record PlaceOrderCommand(
    Guid UserId, Guid RestaurantId, List<Guid> ItemIds,
    Dictionary<Guid, int> Quantities, DeliveryAddress DeliveryAddress, string PaymentToken);

public record PlaceOrderResult(bool IsSuccess, OrderId? OrderId, string? Error)
{
    public static PlaceOrderResult Success(OrderId id) => new(true, id, null);
    public static PlaceOrderResult Failure(string error) => new(false, null, error);
}
```

---

## Step 973: Code Review Checklist

```csharp
// ============================================
// Self-Review Checklist for PRs
// ============================================

/*
Before submitting a PR, review each of these:

## Correctness
☐ Does the code do what the ticket describes?
☐ Edge cases handled (null, empty list, zero, max int)?
☐ No off-by-one errors in loops?
☐ Async methods use ConfigureAwait(false) where appropriate?
☐ Cancellation tokens propagated throughout?

## Security
☐ User input validated and sanitized?
☐ No sensitive data logged?
☐ No SQL injection possibility?
☐ HTTPS enforced for all API calls?
☐ Sensitive data in SecureStorage, not Preferences?

## Performance
☐ No N+1 queries in loops?
☐ Large collections paginated?
☐ Images compressed before upload?
☐ UI work on main thread, IO on background thread?
☐ No blocking .Result or .Wait() calls?

## Testability
☐ Dependencies injected (not new'd inline)?
☐ Static methods avoided for testable code?
☐ Pure functions where possible?
☐ Tests added for the new code?
☐ Edge cases covered in tests?

## Readability
☐ Method names describe what they do?
☐ No magic numbers (use named constants)?
☐ Complex logic commented (WHY, not WHAT)?
☐ No dead code or commented-out code?
☐ Consistent with existing code style?

## MAUI-Specific
☐ Platform-specific code in #if blocks?
☐ Images in correct DPI variants?
☐ Theme works in both light and dark mode?
☐ Accessibility properties set?
☐ Memory leaks: event handlers unsubscribed?
*/

// Example of a well-reviewed piece of code:
public sealed class CartViewModel : ObservableObject, IDisposable
{
    private readonly ICartService _cart;
    
    // Constructor injection — testable
    public CartViewModel(ICartService cart)
    {
        _cart = cart;
        // Subscribe with WeakReference to avoid memory leak
        WeakReferenceMessenger.Default.Register<CartUpdatedMessage>(this, OnCartUpdated);
    }
    
    // Named constant, no magic number
    private const int MaxItemQuantity = 99;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(IsEmpty))]
    private List<CartItem> _items = new();
    
    public bool IsEmpty => Items.Count == 0;
    
    // Async command with cancellation
    [RelayCommand]
    private async Task AddItemAsync(MenuItem item, CancellationToken ct)
    {
        // Validate at boundary
        ArgumentNullException.ThrowIfNull(item);
        
        if (item.Price < 0)
            throw new ArgumentException("Price cannot be negative", nameof(item));
        
        await _cart.AddAsync(item, 1, ct);
    }
    
    private void OnCartUpdated(object r, CartUpdatedMessage m)
        => Items = new List<CartItem>(m.Items);
    
    // Clean up subscription
    public void Dispose()
        => WeakReferenceMessenger.Default.UnregisterAll(this);
}
```

---

## Step 974: Open Source Contribution Guide

```markdown
## Contributing to .NET MAUI OSS Projects

### Finding Issues to Work On
- https://github.com/dotnet/maui/issues?q=is:open+label:"help wanted"
- https://github.com/CommunityToolkit/Maui/issues?q=is:open+label:"good first issue"
- https://github.com/CommunityToolkit/dotnet/issues

### Your First PR — Checklist
1. Fork the repository
2. Clone: `git clone https://github.com/YOUR_USERNAME/maui.git`
3. Create branch: `git checkout -b fix/issue-1234-null-ref-scrollview`
4. Make change with test
5. Run: `dotnet test`
6. Push and open PR against `main`

### Commit Message Format (Conventional Commits)
```
fix(scrollview): prevent NullReferenceException when binding null source

Closes #1234
```

### Code Quality for OSS
- Match the project's existing style exactly
- One focused change per PR (avoid scope creep)
- Add XML doc comments for public APIs
- Include unit test that reproduces the bug
- Respond to review comments within 48 hours
```

---

## Step 975: Technical Interview Preparation

```csharp
// ============================================
// Common Interview Questions & Model Answers
// ============================================

/*
Q1: What is the difference between async/await and Task.Run?
A: async/await is for I/O-bound operations (HTTP, DB) — it doesn't create new threads,
   it yields the current thread back to the pool. Task.Run is for CPU-bound work 
   that should run on a thread pool thread. In MAUI, never use Task.Run for UI updates;
   use MainThread.InvokeOnMainThreadAsync instead.

Q2: Explain MVVM in MAUI
A: Model = data/domain classes. ViewModel = state + commands, inherits ObservableObject,
   no direct View reference. View = XAML page, binds to ViewModel via BindingContext.
   CommunityToolkit.Mvvm generates [ObservableProperty] and [RelayCommand] boilerplate.

Q3: How do you prevent memory leaks with event handlers?
A: Three approaches:
   1. WeakReferenceMessenger — holds weak reference, auto-GC safe
   2. Unsubscribe in Dispose/OnDisappearing
   3. Use lambdas stored as fields so they can be unsubscribed:
      _handler = (s, e) => DoWork();
      button.Clicked += _handler;
      // later:
      button.Clicked -= _handler;

Q4: What is Dependency Injection and why use it?
A: DI = provide dependencies from outside rather than creating inside.
   Benefits: testable (swap real for fake), configurable (change implementation),
   single instance (singleton services), clear dependencies.
   MAUI registers in MauiProgram.cs builder.Services.

Q5: Explain SQLite transactions and when to use them
A: RunInTransactionAsync groups multiple writes atomically — either all succeed or all roll back.
   Use for: bulk inserts (10x+ faster), multi-table operations that must stay consistent.
   Don't nest transactions; SQLite doesn't support nested transactions.
*/

// Common live-coding question: LRU Cache
public class LruCache<TKey, TValue>
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _dict = new();
    private readonly LinkedList<(TKey Key, TValue Value)> _list = new();
    
    public LruCache(int capacity) => _capacity = capacity;
    
    public TValue? Get(TKey key)
    {
        if (!_dict.TryGetValue(key, out var node)) return default;
        _list.Remove(node);
        _list.AddFirst(node);
        return node.Value.Value;
    }
    
    public void Put(TKey key, TValue value)
    {
        if (_dict.TryGetValue(key, out var existing))
        {
            _list.Remove(existing);
            _dict.Remove(key);
        }
        
        var node = _list.AddFirst((key, value));
        _dict[key] = node;
        
        if (_dict.Count > _capacity)
        {
            var last = _list.Last!;
            _list.RemoveLast();
            _dict.Remove(last.Value.Key);
        }
    }
}
```

---

## Step 976: Performance Optimization Profile

```csharp
// ============================================
// App Startup Time Optimization
// ============================================

public class StartupOptimizer
{
    // Measure startup phases
    private static readonly Stopwatch _appStartSw = Stopwatch.StartNew();
    
    public static void RecordPhase(string phase)
    {
        System.Diagnostics.Debug.WriteLine($"[STARTUP] {phase}: {_appStartSw.ElapsedMilliseconds}ms");
    }
    
    // MauiProgram.cs optimizations:
    // 1. Defer non-critical service initialization
    // 2. Use lazy initialization for heavy services
    // 3. Register SQLite connection as singleton (not transient)
    // 4. Pre-warm HttpClient connection pool
    
    public static async Task PreWarmAsync(IServiceProvider services)
    {
        // Background pre-warm after app visible
        await Task.Delay(1000); // Let first frame render
        
        // Pre-warm: establish DB connection
        var db = services.GetRequiredService<SQLiteAsyncConnection>();
        await db.ExecuteAsync("SELECT 1");
        
        // Pre-warm: DNS resolution for primary API
        using var http = new HttpClient();
        _ = http.GetAsync("https://api.fooddelivery.th/health").ConfigureAwait(false);
        
        RecordPhase("pre-warm complete");
    }
}

// XAML Performance Tips:
// 1. Use x:DataType for compiled bindings (10-30% faster)
//    <ContentPage x:DataType="vm:HomeViewModel">
// 2. CollectionView > ListView (virtualized, recycled views)
// 3. Set Shell.NavBarIsVisible="False" when not needed
// 4. Use CachedImage (FFImageLoading) for network images
// 5. Avoid nesting too many Grid/StackLayout (flatten hierarchy)
// 6. Use Shell.PresentationMode="ModalAnimated" sparingly
```

---

## Step 977: GitHub Portfolio Tips

```csharp
// ============================================
// Making Your GitHub Portfolio Stand Out
// ============================================

/*
README.md best practices for your MAUI portfolio app:

## 📱 FoodDelivery App

[![CI](https://github.com/username/fooddelivery-maui/actions/workflows/ci.yml/badge.svg)]()
[![Coverage](https://img.shields.io/badge/coverage-85%25-brightgreen)]()
[![Platform](https://img.shields.io/badge/platform-iOS%20%7C%20Android-blue)]()

A production-ready food delivery app built with .NET MAUI.

### Screenshots
[Grid of 4-5 screenshots]

### Architecture
- MVVM with CommunityToolkit.Mvvm
- Clean Architecture (Domain / Application / Infrastructure / UI)
- SQLite + Multi-layer cache (Memory → Disk → Remote)
- SAGA pattern for order flow
- Certificate pinning + AES-256-GCM encryption

### Key Features
- Real-time rider tracking with SignalR
- PromPay QR code generation (EMVCo format)
- GraphQL with WebSocket subscriptions
- Biometric authentication
- 85% test coverage

### Running Locally
```bash
dotnet workload install maui
git clone https://github.com/username/fooddelivery-maui
cd fooddelivery-maui
dotnet build FoodDelivery.sln
```

Showcase measurable achievements:
- "Reduced startup time from 2.8s to 1.1s by deferring non-critical init"
- "Achieved 99.7% crash-free rate across 50,000 DAU"
- "Cut API payload size 60% with Brotli compression"
*/

// Skills to list on resume for Senior MAUI Developer:
var seniorMauiSkills = new[]
{
    // Frameworks & Platforms
    ".NET MAUI", "Xamarin.Forms", "C# 12", "MVVM", "CommunityToolkit.Mvvm",
    
    // Architecture
    "Clean Architecture", "CQRS", "Event Sourcing", "SAGA", "Outbox Pattern",
    
    // UI
    "Custom Controls", "Animations", "Dark Mode", "Accessibility (WCAG 2.1 AA)",
    
    // Data
    "SQLite", "SQLite-net-pcl", "Full-Text Search (FTS5)", "Entity Framework Core",
    
    // Networking
    "REST API", "GraphQL", "SignalR", "WebSocket", "Polly Resilience",
    
    // Security
    "Certificate Pinning", "AES-256-GCM", "OAuth2/OIDC", "Biometric Auth",
    
    // Testing
    "NUnit", "NSubstitute", "Integration Tests", "UI Tests", "Code Coverage",
    
    // DevOps
    "GitHub Actions", "Firebase App Distribution", "TestFlight", "Crash Reporting",
    
    // Payments
    "Omise", "PromPay QR", "In-App Purchase", "Subscription Management"
};
```

---

## Step 978: Senior Developer Mindset

```csharp
// ============================================
// Engineering Principles at Senior Level
// ============================================

/*
1. YAGNI (You Aren't Gonna Need It)
   - Don't add abstraction until you have 3 real uses
   - Premature generalization is a form of waste
   
2. Make it work → Make it right → Make it fast
   - Get correct behavior first (with tests)
   - Then refactor for clarity
   - Then profile and optimize only proven bottlenecks
   
3. Boring technology for core, interesting for edges
   - Use SQLite/JSON/REST for core data flow
   - Experiment at the edge (new protocol, new pattern)
   
4. Observe before optimizing
   - Profile with dotnet-trace / Instruments before guessing
   - "I think this is slow" is not data
   
5. Write code for the 3am on-call developer (probably you)
   - Log at boundaries, not inside logic
   - Errors should say WHAT failed and WHY
   - Recovery path must be clear
   
6. Design for delete
   - The best code is code you don't have to maintain
   - Ask: can this feature be removed without surgery?
   
7. Thai market specifics (competitive edge)
   - Line payment integration
   - PromPay QR (high mobile penetration)
   - Thai Buddhist calendar support
   - Thai language search (FTS5 unicode61)
   - Bangkok traffic patterns in delivery ETA
*/

// Example of senior-level code review comment:
/*
Instead of:
  var result = new List<Restaurant>();
  foreach (var r in allRestaurants)
  {
    if (r.IsOpen && r.Distance < maxDistance)
      result.Add(r);
  }
  result.Sort((a, b) => a.Rating.CompareTo(b.Rating));
  return result;

Prefer:
  return allRestaurants
    .Where(r => r.IsOpen && r.Distance < maxDistance)
    .OrderByDescending(r => r.Rating)
    .ToList();

The LINQ version reads like the business rule, not like an algorithm.
*/
```

---

## Step 979: Teaching & Knowledge Sharing

```csharp
// ============================================
// Becoming a Technical Leader
// ============================================

/*
MAUI Internal Workshop Agenda (90 minutes)

Part 1: Why MAUI? (15 min)
- Single codebase for iOS + Android + macOS + Windows
- .NET ecosystem: dependency injection, testing, NuGet
- Live demo: platform APIs in 5 lines (camera, location, storage)

Part 2: MVVM Deep Dive (30 min)
- Draw the message flow: View → ViewModel → Service → API → back
- Live-code a CartViewModel with unit test
- Show generated code from [ObservableProperty]

Part 3: Architecture Decisions (30 min)
- Why Clean Architecture over MVC?
- SQLite vs shared preferences — decision table
- When to use caching vs always-fetch-fresh

Part 4: Q&A + Practice Task (15 min)
- Build a "Today's Specials" feature from scratch
- Must include: ObservableProperty, RelayCommand, Unit Test

Code Review Mentoring Tips:
- Ask questions instead of demanding changes: "What happens if X is null here?"
- Explain WHY, not just WHAT to fix
- Praise good patterns explicitly: "Nice use of sealed class — prevents unintended inheritance"
- Share relevant docs or examples, not just criticism
- Batch small nits; only block on real issues
*/
```

---

## Step 980: Career Roadmap

```
// ============================================
// MAUI Developer Career Path
// ============================================

Junior MAUI Developer (0-2 years)
├── XAML layouts, data binding
├── REST API integration
├── Basic MVVM
├── SQLite basic CRUD
└── App Store submission

Mid-Level MAUI Developer (2-4 years)
├── Custom controls & animations
├── Platform-specific code (#if, dependency injection)
├── Unit testing + CI/CD
├── Performance optimization
├── Complex navigation patterns
└── Multi-environment config

Senior MAUI Developer (4+ years)
├── Architecture decision-making (Clean Arch, DDD)
├── Security hardening (pinning, encryption, RBAC)
├── Enterprise patterns (SAGA, Event Sourcing, CQRS)
├── Cross-team technical leadership
├── Mentoring junior developers
└── OSS contribution

Specialization Paths:
1. Mobile Platform Expert → deep iOS/Android native, write platform bindings
2. Enterprise Architect → multi-tenant, SSO, compliance (PDPA/GDPR)
3. Performance Engineer → startup time, memory profiling, battery
4. DevOps/Mobile → CI/CD pipelines, automated testing at scale
5. Product Engineering Lead → combine technical + product thinking

Thailand Market Opportunities (2025+):
- FinTech: payment apps, digital banking (KBANK, SCB, KTB have MAUI teams)
- E-Commerce: Lazada, Shopee, Central Group
- Health Tech: hospital apps, telemedicine
- Government: ภาษี, e-ID, government services digitization
- Logistics: Kerry, Thailand Post, Flash Express
```

---

## สรุป Part 98

ใน Part 98 เราได้เรียนรู้:

1. **Portfolio Architecture** - Solution structure, domain model, Clean Architecture
2. **Layer Boundaries** - Dependency rule, PlaceOrderUseCase, Result pattern
3. **Code Review Checklist** - Correctness, Security, Performance, Testability, MAUI-specific
4. **OSS Contribution** - Finding issues, PR etiquette, conventional commits
5. **Interview Prep** - async/await, MVVM, memory leaks, DI, LRU Cache coding
6. **Startup Optimization** - Phase timing, deferred init, compiled bindings
7. **GitHub Portfolio** - README badges, architecture summary, measurable achievements
8. **Senior Mindset** - YAGNI, observe before optimizing, 3am developer
9. **Teaching** - Workshop agenda, code review mentoring, praise good patterns
10. **Career Roadmap** - Junior → Senior path, Thailand market opportunities

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 98 | Steps 971-980*
