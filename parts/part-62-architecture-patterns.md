# Part 62: Advanced Architecture Patterns
## Steps 611-620: Vertical Slice, Feature Flags, Multi-tenant, Plugin System

---

## Step 611: Vertical Slice Architecture

```csharp
// ============================================
// Vertical Slice Architecture
// ============================================

/*
 * Vertical Slice: แต่ละ Feature อยู่ในที่เดียว
 * 
 * Features/
 *   Products/
 *     GetProducts/
 *       GetProductsQuery.cs
 *       GetProductsHandler.cs
 *       GetProductsViewModel.cs
 *       GetProductsPage.xaml
 *     CreateProduct/
 *       CreateProductCommand.cs
 *       CreateProductHandler.cs
 *       CreateProductPage.xaml
 *   Orders/
 *     PlaceOrder/...
 */

// Self-contained feature slice
namespace Features.Products.GetProducts
{
    // Query
    public record GetProductsQuery(
        string? Search, string? Category,
        int Page = 1, int PageSize = 20) : IRequest<PagedResult<ProductSummary>>;
    
    // Response type
    public record ProductSummary(
        int Id, string Name, string Category,
        decimal Price, bool InStock, string? ThumbnailUrl);
    
    // Handler (all logic in one place)
    public class GetProductsHandler : IRequestHandler<GetProductsQuery, PagedResult<ProductSummary>>
    {
        private readonly SQLiteConnection _db;
        
        public GetProductsHandler(SQLiteConnection db) => _db = db;
        
        public Task<PagedResult<ProductSummary>> Handle(
            GetProductsQuery query, CancellationToken ct)
        {
            var q = _db.Table<ProductData>().AsQueryable();
            
            if (!string.IsNullOrEmpty(query.Search))
                q = q.Where(p => p.Name.Contains(query.Search));
            
            if (!string.IsNullOrEmpty(query.Category))
                q = q.Where(p => p.Category == query.Category);
            
            var total = q.Count();
            var items = q
                .OrderBy(p => p.Name)
                .Skip((query.Page - 1) * query.PageSize)
                .Take(query.PageSize)
                .Select(p => new ProductSummary(
                    p.Id, p.Name, p.Category, p.Price,
                    p.StockQuantity > 0, p.ThumbnailUrl))
                .ToList();
            
            return Task.FromResult(new PagedResult<ProductSummary>(
                items, total, query.Page, query.PageSize));
        }
    }
    
    // ViewModel (self-contained)
    public partial class GetProductsViewModel : ObservableObject
    {
        private readonly IMediator _mediator;
        
        [ObservableProperty] private List<ProductSummary> _products = new();
        [ObservableProperty] private bool _isLoading;
        [ObservableProperty] private string _searchText = string.Empty;
        
        public GetProductsViewModel(IMediator mediator) => _mediator = mediator;
        
        [RelayCommand]
        private async Task LoadAsync()
        {
            IsLoading = true;
            var result = await _mediator.Send(new GetProductsQuery(SearchText));
            Products = result.Items;
            IsLoading = false;
        }
    }
}
```

---

## Step 612: Feature Flag System

```csharp
// ============================================
// Feature Flag System
// ============================================

public enum FeatureFlag
{
    NewCheckoutFlow,
    AiProductRecommendations,
    LiveChat,
    DarkModeToggle,
    BiometricLogin,
    SplitPayment,
    LoyaltyPoints,
    AugmentedReality
}

public interface IFeatureFlagService
{
    bool IsEnabled(FeatureFlag flag);
    Task<bool> IsEnabledAsync(FeatureFlag flag, int? userId = null);
    void Override(FeatureFlag flag, bool value);
    void ClearOverride(FeatureFlag flag);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IRemoteConfigService _remote;
    private readonly IPreferences _prefs;
    private readonly Dictionary<FeatureFlag, bool> _overrides = new();
    
    // Local defaults (used when offline)
    private static readonly Dictionary<FeatureFlag, bool> _defaults = new()
    {
        [FeatureFlag.NewCheckoutFlow] = false,
        [FeatureFlag.AiProductRecommendations] = false,
        [FeatureFlag.LiveChat] = true,
        [FeatureFlag.DarkModeToggle] = true,
        [FeatureFlag.BiometricLogin] = true,
        [FeatureFlag.SplitPayment] = false,
        [FeatureFlag.LoyaltyPoints] = false,
        [FeatureFlag.AugmentedReality] = false,
    };
    
    public FeatureFlagService(IRemoteConfigService remote, IPreferences prefs)
    {
        _remote = remote;
        _prefs = prefs;
    }
    
    public bool IsEnabled(FeatureFlag flag)
    {
        // Manual override (for testing)
        if (_overrides.TryGetValue(flag, out var overrideVal))
            return overrideVal;
        
        // Cached remote value
        var key = $"feature_{flag}";
        if (_prefs.ContainsKey(key))
            return _prefs.Get(key, false);
        
        // Default
        return _defaults.TryGetValue(flag, out var def) && def;
    }
    
    public async Task<bool> IsEnabledAsync(FeatureFlag flag, int? userId = null)
    {
        if (_overrides.TryGetValue(flag, out var overrideVal))
            return overrideVal;
        
        // User-specific override from server
        var flagName = flag.ToString();
        var remoteValue = await _remote.GetBoolAsync(flagName, userId);
        
        if (remoteValue.HasValue)
        {
            _prefs.Set($"feature_{flag}", remoteValue.Value);
            return remoteValue.Value;
        }
        
        return _defaults.TryGetValue(flag, out var def) && def;
    }
    
    public void Override(FeatureFlag flag, bool value)
        => _overrides[flag] = value;
    
    public void ClearOverride(FeatureFlag flag)
        => _overrides.Remove(flag);
}

// Usage in ViewModel
public partial class CheckoutViewModel : ObservableObject
{
    private readonly IFeatureFlagService _flags;
    
    [ObservableProperty] private bool _showNewCheckout;
    [ObservableProperty] private bool _showSplitPayment;
    
    public CheckoutViewModel(IFeatureFlagService flags)
    {
        _flags = flags;
        ShowNewCheckout = flags.IsEnabled(FeatureFlag.NewCheckoutFlow);
        ShowSplitPayment = flags.IsEnabled(FeatureFlag.SplitPayment);
    }
}
```

---

## Step 613: A/B Testing

```csharp
// ============================================
// A/B Testing Framework
// ============================================

public record Experiment(
    string Name, string[] Variants,
    Dictionary<string, int> Weights);

public interface IExperimentService
{
    string GetVariant(string experimentName, int userId);
    Task TrackConversionAsync(string experimentName, string variant, string eventName);
}

public class ExperimentService : IExperimentService
{
    private readonly IPreferences _prefs;
    private readonly IAnalyticsService _analytics;
    
    private static readonly Dictionary<string, Experiment> _experiments = new()
    {
        ["checkout_button_color"] = new(
            "checkout_button_color",
            new[] { "blue", "green", "red" },
            new() { ["blue"] = 50, ["green"] = 30, ["red"] = 20 }),
        
        ["product_list_layout"] = new(
            "product_list_layout",
            new[] { "grid", "list" },
            new() { ["grid"] = 60, ["list"] = 40 }),
        
        ["recommendation_algorithm"] = new(
            "recommendation_algorithm",
            new[] { "collaborative", "content_based", "hybrid" },
            new() { ["collaborative"] = 40, ["content_based"] = 30, ["hybrid"] = 30 }),
    };
    
    public ExperimentService(IPreferences prefs, IAnalyticsService analytics)
    {
        _prefs = prefs;
        _analytics = analytics;
    }
    
    public string GetVariant(string experimentName, int userId)
    {
        var cacheKey = $"exp_{experimentName}_{userId}";
        
        // Sticky assignment (same user always gets same variant)
        if (_prefs.ContainsKey(cacheKey))
            return _prefs.Get(cacheKey, "control");
        
        if (!_experiments.TryGetValue(experimentName, out var experiment))
            return "control";
        
        var variant = AssignVariant(experiment, userId);
        _prefs.Set(cacheKey, variant);
        
        // Track assignment
        _ = _analytics.TrackAsync("experiment_assigned", new Dictionary<string, string>
        {
            ["experiment"] = experimentName,
            ["variant"] = variant,
            ["userId"] = userId.ToString()
        });
        
        return variant;
    }
    
    private static string AssignVariant(Experiment experiment, int userId)
    {
        var hash = Math.Abs(HashCode.Combine(experiment.Name, userId) % 100);
        var cumulative = 0;
        
        foreach (var (variant, weight) in experiment.Weights)
        {
            cumulative += weight;
            if (hash < cumulative) return variant;
        }
        
        return experiment.Variants[0];
    }
    
    public Task TrackConversionAsync(string experiment, string variant, string eventName)
        => _analytics.TrackAsync($"experiment_conversion", new Dictionary<string, string>
        {
            ["experiment"] = experiment,
            ["variant"] = variant,
            ["event"] = eventName
        });
}
```

---

## Step 614: Multi-tenant Architecture

```csharp
// ============================================
// Multi-tenant Support
// ============================================

public interface ITenantContext
{
    string TenantId { get; }
    string TenantName { get; }
    TenantConfig Config { get; }
}

public record TenantConfig(
    string PrimaryColor, string LogoUrl,
    bool AllowGuestCheckout, int MaxCartItems,
    List<string> EnabledPaymentMethods);

public class TenantContext : ITenantContext
{
    public string TenantId { get; private set; } = "default";
    public string TenantName { get; private set; } = "MyShop";
    public TenantConfig Config { get; private set; } = DefaultConfig;
    
    private static readonly TenantConfig DefaultConfig = new(
        "#007AFF", "logo.png", true, 100,
        new List<string> { "credit_card", "promptpay", "cod" });
    
    public void SetTenant(string tenantId, string tenantName, TenantConfig config)
    {
        TenantId = tenantId;
        TenantName = tenantName;
        Config = config;
    }
}

// Repository with tenant isolation
public class TenantAwareProductRepository
{
    private readonly SQLiteConnection _db;
    private readonly ITenantContext _tenant;
    
    public TenantAwareProductRepository(SQLiteConnection db, ITenantContext tenant)
    {
        _db = db;
        _tenant = tenant;
    }
    
    public List<ProductData> GetProducts()
        => _db.Table<ProductData>()
            .Where(p => p.TenantId == _tenant.TenantId)
            .ToList();
    
    public ProductData? GetById(int id)
        => _db.Table<ProductData>()
            .FirstOrDefault(p => p.Id == id && p.TenantId == _tenant.TenantId);
    
    public void Save(ProductData product)
    {
        product.TenantId = _tenant.TenantId;
        if (product.Id == 0) _db.Insert(product);
        else _db.Update(product);
    }
    
    public void Delete(int id)
        => _db.Execute(
            "DELETE FROM ProductData WHERE Id = ? AND TenantId = ?",
            id, _tenant.TenantId);
}
```

---

## Step 615: Plugin System

```csharp
// ============================================
// Plugin/Extension System
// ============================================

public interface IPlugin
{
    string Name { get; }
    string Version { get; }
    void Initialize(IServiceCollection services);
    void Configure(IServiceProvider provider);
}

public interface IPaymentPlugin : IPlugin
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
    bool Supports(string method);
}

// Example payment plugins
public class PromptPayPlugin : IPaymentPlugin
{
    public string Name => "PromptPay";
    public string Version => "1.0.0";
    
    public void Initialize(IServiceCollection services)
        => services.AddSingleton<PromptPayPlugin>();
    
    public void Configure(IServiceProvider provider) { }
    
    public bool Supports(string method) 
        => method == "promptpay";
    
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        // Generate QR code, wait for confirmation
        var qrCode = GenerateQrCode(request.Amount, request.Reference);
        await WaitForConfirmationAsync(request.Reference);
        return new PaymentResult(true, request.Reference, "PromptPay");
    }
    
    private string GenerateQrCode(decimal amount, string reference) => $"QR:{amount}:{reference}";
    private Task WaitForConfirmationAsync(string reference) => Task.Delay(1000);
}

public class PluginManager
{
    private readonly List<IPlugin> _plugins = new();
    
    public void Register(IPlugin plugin) => _plugins.Add(plugin);
    
    public void InitializeAll(IServiceCollection services)
    {
        foreach (var plugin in _plugins)
            plugin.Initialize(services);
    }
    
    public void ConfigureAll(IServiceProvider provider)
    {
        foreach (var plugin in _plugins)
            plugin.Configure(provider);
    }
    
    public T? GetPlugin<T>(string name) where T : IPlugin
        => _plugins.OfType<T>().FirstOrDefault(p => p.Name == name);
    
    public IEnumerable<T> GetPlugins<T>() where T : IPlugin
        => _plugins.OfType<T>();
}

// Payment service using plugins
public class PluginPaymentService
{
    private readonly PluginManager _plugins;
    
    public PluginPaymentService(PluginManager plugins) => _plugins = plugins;
    
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        var plugin = _plugins.GetPlugins<IPaymentPlugin>()
            .FirstOrDefault(p => p.Supports(request.Method))
            ?? throw new NotSupportedException($"Payment method not supported: {request.Method}");
        
        return await plugin.ProcessAsync(request);
    }
}

public record PaymentRequest(decimal Amount, string Method, string Reference);
public record PaymentResult(bool Success, string TransactionId, string Method);
```

---

## Step 616: Domain Service Pattern

```csharp
// ============================================
// Domain Services for Complex Business Logic
// ============================================

// Pricing domain service
public class PricingService
{
    private readonly IDiscountRepository _discounts;
    private readonly IUserRepository _users;
    
    public PricingService(IDiscountRepository discounts, IUserRepository users)
    {
        _discounts = discounts;
        _users = users;
    }
    
    public async Task<PricingResult> CalculateAsync(
        List<CartItem> items, int userId, string? couponCode)
    {
        var subtotal = items.Sum(i => i.UnitPrice * i.Quantity);
        var user = await _users.GetByIdAsync(userId);
        
        var discounts = new List<DiscountApplied>();
        
        // Tier discount
        if (user?.TierLevel == "Gold" && subtotal >= 1000)
        {
            var tierDiscount = subtotal * 0.10m;
            discounts.Add(new("สมาชิก Gold 10%", tierDiscount));
            subtotal -= tierDiscount;
        }
        
        // Volume discount
        var totalItems = items.Sum(i => i.Quantity);
        if (totalItems >= 5)
        {
            var volumeDiscount = subtotal * 0.05m;
            discounts.Add(new("ส่วนลดปริมาณ 5%", volumeDiscount));
            subtotal -= volumeDiscount;
        }
        
        // Coupon
        if (couponCode != null)
        {
            var coupon = await _discounts.FindCouponAsync(couponCode);
            if (coupon != null && coupon.IsValid && subtotal >= coupon.MinimumOrder)
            {
                var couponDiscount = coupon.Type == DiscountType.Percentage
                    ? subtotal * coupon.Value / 100
                    : Math.Min(coupon.Value, subtotal);
                
                discounts.Add(new($"คูปอง {couponCode}", couponDiscount));
                subtotal -= couponDiscount;
            }
        }
        
        // Shipping
        var shipping = subtotal >= 999 ? 0m : 50m;
        
        return new PricingResult(
            Subtotal: items.Sum(i => i.UnitPrice * i.Quantity),
            Discounts: discounts,
            ShippingFee: shipping,
            Total: subtotal + shipping);
    }
}

public record PricingResult(
    decimal Subtotal, List<DiscountApplied> Discounts,
    decimal ShippingFee, decimal Total);

public record DiscountApplied(string Description, decimal Amount);
```

---

## Step 617: Specification Pattern Advanced

```csharp
// ============================================
// Advanced Specification Pattern
// ============================================

public abstract class Specification<T>
{
    public abstract bool IsSatisfiedBy(T entity);
    
    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other) => new OrSpec<T>(this, other);
    public Specification<T> Not() => new NotSpec<T>(this);
    
    public IEnumerable<T> Filter(IEnumerable<T> items)
        => items.Where(IsSatisfiedBy);
}

class AndSpec<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public AndSpec(Specification<T> l, Specification<T> r) { _left = l; _right = r; }
    public override bool IsSatisfiedBy(T e) => _left.IsSatisfiedBy(e) && _right.IsSatisfiedBy(e);
}

class OrSpec<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public OrSpec(Specification<T> l, Specification<T> r) { _left = l; _right = r; }
    public override bool IsSatisfiedBy(T e) => _left.IsSatisfiedBy(e) || _right.IsSatisfiedBy(e);
}

class NotSpec<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpec(Specification<T> inner) => _inner = inner;
    public override bool IsSatisfiedBy(T e) => !_inner.IsSatisfiedBy(e);
}

// Product specifications
public class InStockSpec : Specification<ProductData>
{
    public override bool IsSatisfiedBy(ProductData p) => p.StockQuantity > 0;
}

public class PriceRangeSpec : Specification<ProductData>
{
    private readonly decimal _min, _max;
    public PriceRangeSpec(decimal min, decimal max) { _min = min; _max = max; }
    public override bool IsSatisfiedBy(ProductData p) => p.Price >= _min && p.Price <= _max;
}

public class CategorySpec : Specification<ProductData>
{
    private readonly string _category;
    public CategorySpec(string category) => _category = category;
    public override bool IsSatisfiedBy(ProductData p) => p.Category == _category;
}

public class OnSaleSpec : Specification<ProductData>
{
    public override bool IsSatisfiedBy(ProductData p)
        => p.OriginalPrice.HasValue && p.Price < p.OriginalPrice;
}

// Usage
public class ProductSearchService
{
    public List<ProductData> Search(List<ProductData> all, SearchCriteria criteria)
    {
        Specification<ProductData> spec = new InStockSpec();
        
        if (criteria.Category != null)
            spec = spec.And(new CategorySpec(criteria.Category));
        
        if (criteria.MinPrice.HasValue || criteria.MaxPrice.HasValue)
            spec = spec.And(new PriceRangeSpec(
                criteria.MinPrice ?? 0,
                criteria.MaxPrice ?? decimal.MaxValue));
        
        if (criteria.OnSaleOnly)
            spec = spec.And(new OnSaleSpec());
        
        return spec.Filter(all).ToList();
    }
}

public record SearchCriteria(
    string? Category = null,
    decimal? MinPrice = null, decimal? MaxPrice = null,
    bool OnSaleOnly = false);
```

---

## Step 618: Aggregate Root Advanced

```csharp
// ============================================
// Aggregate Root with Invariants
// ============================================

public class Cart : AggregateRoot
{
    private readonly List<CartLineItem> _lines = new();
    private const int MaxItems = 100;
    private const decimal MaxTotal = 100_000m;
    
    public int CustomerId { get; private set; }
    public CartStatus Status { get; private set; }
    public string? CouponCode { get; private set; }
    
    public IReadOnlyList<CartLineItem> Lines => _lines.AsReadOnly();
    public decimal Subtotal => _lines.Sum(l => l.LineTotal);
    public int TotalQuantity => _lines.Sum(l => l.Quantity);
    
    private Cart() { } // EF/SQLite reconstruction
    
    public static Cart Create(int customerId)
    {
        var cart = new Cart
        {
            CustomerId = customerId,
            Status = CartStatus.Active
        };
        cart.AddDomainEvent(new CartCreatedEvent(Guid.NewGuid(), DateTime.UtcNow, customerId));
        return cart;
    }
    
    public void AddItem(int productId, string name, decimal price, int quantity)
    {
        if (Status != CartStatus.Active)
            throw new DomainException("CART_NOT_ACTIVE", "ตะกร้าสินค้าไม่ใช้งาน");
        
        if (TotalQuantity + quantity > MaxItems)
            throw new DomainException("CART_MAX_ITEMS", $"ไม่สามารถเพิ่มสินค้าเกิน {MaxItems} ชิ้น");
        
        var existing = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (existing != null)
        {
            existing.IncreaseQuantity(quantity);
        }
        else
        {
            _lines.Add(new CartLineItem(productId, name, price, quantity));
        }
        
        if (Subtotal > MaxTotal)
            throw new DomainException("CART_MAX_TOTAL", $"ยอดรวมไม่สามารถเกิน ฿{MaxTotal:N0}");
    }
    
    public void RemoveItem(int productId)
    {
        var line = _lines.FirstOrDefault(l => l.ProductId == productId)
            ?? throw new DomainException("ITEM_NOT_FOUND", "ไม่พบสินค้าในตะกร้า");
        _lines.Remove(line);
    }
    
    public void ApplyCoupon(string code)
    {
        if (_lines.Count == 0)
            throw new DomainException("CART_EMPTY", "ตะกร้าสินค้าว่างเปล่า");
        CouponCode = code;
    }
    
    public void Checkout()
    {
        if (_lines.Count == 0)
            throw new DomainException("CART_EMPTY", "กรุณาเพิ่มสินค้าก่อนชำระเงิน");
        
        Status = CartStatus.CheckedOut;
        AddDomainEvent(new CartCheckedOutEvent(
            Guid.NewGuid(), DateTime.UtcNow, CustomerId, Subtotal));
    }
}

public class CartLineItem
{
    public int ProductId { get; }
    public string Name { get; }
    public decimal UnitPrice { get; }
    public int Quantity { get; private set; }
    public decimal LineTotal => UnitPrice * Quantity;
    
    public CartLineItem(int id, string name, decimal price, int qty)
    {
        ProductId = id; Name = name; UnitPrice = price; Quantity = qty;
    }
    
    public void IncreaseQuantity(int qty)
    {
        if (qty <= 0) throw new ArgumentException("จำนวนต้องมากกว่า 0");
        Quantity += qty;
    }
}

public enum CartStatus { Active, CheckedOut, Abandoned }
```

---

## Step 619: Repository Pattern Advanced

```csharp
// ============================================
// Generic Repository + Unit of Work
// ============================================

public interface IRepository<T, TId>
{
    Task<T?> GetByIdAsync(TId id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    Task<int> SaveAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(TId id, CancellationToken ct = default);
}

public interface IUnitOfWork : IDisposable
{
    ICartRepository Carts { get; }
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    
    Task BeginAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

public class SQLiteUnitOfWork : IUnitOfWork
{
    private readonly SQLiteConnection _db;
    private bool _inTransaction;
    
    public ICartRepository Carts { get; }
    public IOrderRepository Orders { get; }
    public IProductRepository Products { get; }
    
    public SQLiteUnitOfWork(
        SQLiteConnection db,
        ICartRepository carts,
        IOrderRepository orders,
        IProductRepository products)
    {
        _db = db;
        Carts = carts;
        Orders = orders;
        Products = products;
    }
    
    public Task BeginAsync()
    {
        _db.BeginTransaction();
        _inTransaction = true;
        return Task.CompletedTask;
    }
    
    public Task CommitAsync()
    {
        if (_inTransaction) _db.Commit();
        _inTransaction = false;
        return Task.CompletedTask;
    }
    
    public Task RollbackAsync()
    {
        if (_inTransaction) _db.Rollback();
        _inTransaction = false;
        return Task.CompletedTask;
    }
    
    public void Dispose()
    {
        if (_inTransaction) _db.Rollback();
    }
}

// Usage
public class PlaceOrderService
{
    private readonly IUnitOfWork _uow;
    private readonly PricingService _pricing;
    
    public PlaceOrderService(IUnitOfWork uow, PricingService pricing)
    {
        _uow = uow;
        _pricing = pricing;
    }
    
    public async Task<int> PlaceOrderAsync(int cartId, int userId, string address)
    {
        await _uow.BeginAsync();
        
        try
        {
            var cart = await _uow.Carts.GetByIdAsync(cartId)
                ?? throw new DomainException("CART_NOT_FOUND", "ไม่พบตะกร้าสินค้า");
            
            var pricing = await _pricing.CalculateAsync(
                cart.Lines.Select(l => new CartItem(l.ProductId, l.Name, l.UnitPrice, l.Quantity)).ToList(),
                userId, cart.CouponCode);
            
            var order = Order.Create(userId, cart.Lines
                .Select(l => new OrderLine(l.ProductId, l.Name, l.UnitPrice, l.Quantity))
                .ToList(), address, pricing.Total);
            
            await _uow.Orders.SaveAsync(order);
            cart.Checkout();
            await _uow.Carts.SaveAsync(cart);
            
            await _uow.CommitAsync();
            return order.Id;
        }
        catch
        {
            await _uow.RollbackAsync();
            throw;
        }
    }
}
```

---

## Step 620: Architecture Tests

```csharp
// ============================================
// Architecture Tests (NetArchTest)
// ============================================

// dotnet add package NetArchTest.Rules

public class ArchitectureTests
{
    private const string DomainNamespace = "MyApp.Domain";
    private const string ApplicationNamespace = "MyApp.Application";
    private const string InfraNamespace = "MyApp.Infrastructure";
    private const string UiNamespace = "MyApp.UI.MAUI";
    
    [Fact]
    public void Domain_ShouldNotReferenceOtherLayers()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .That().ResideInNamespace(DomainNamespace)
            .ShouldNot().HaveDependencyOnAny(
                ApplicationNamespace, InfraNamespace, UiNamespace)
            .GetResult();
        
        result.IsSuccessful.Should().BeTrue(
            string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>()));
    }
    
    [Fact]
    public void Application_ShouldNotReferenceInfrastructure()
    {
        var result = Types.InAssembly(typeof(CreateProductHandler).Assembly)
            .That().ResideInNamespace(ApplicationNamespace)
            .ShouldNot().HaveDependencyOn(InfraNamespace)
            .GetResult();
        
        result.IsSuccessful.Should().BeTrue();
    }
    
    [Fact]
    public void ViewModels_ShouldEndWithViewModel()
    {
        var result = Types.InAssembly(typeof(ProductListViewModel).Assembly)
            .That().Inherit(typeof(ObservableObject))
            .And().AreNotAbstract()
            .Should().HaveNameEndingWith("ViewModel")
            .GetResult();
        
        result.IsSuccessful.Should().BeTrue();
    }
    
    [Fact]
    public void Handlers_ShouldImplementHandlerInterface()
    {
        var result = Types.InAssembly(typeof(CreateProductHandler).Assembly)
            .That().HaveNameEndingWith("Handler")
            .Should().ImplementInterface(typeof(IRequestHandler<,>))
            .GetResult();
        
        result.IsSuccessful.Should().BeTrue();
    }
}
```

---

## สรุป Part 62

ใน Part 62 เราได้เรียนรู้:

1. **Vertical Slice** - Feature-first architecture
2. **Feature Flags** - Remote config, override
3. **A/B Testing** - Sticky variant assignment
4. **Multi-tenant** - Tenant context, data isolation
5. **Plugin System** - IPlugin, PluginManager
6. **Domain Services** - Complex pricing logic
7. **Specification Advanced** - And/Or/Not composition
8. **Aggregate Root Advanced** - Cart with invariants
9. **Repository + UoW** - Generic + Unit of Work
10. **Architecture Tests** - NetArchTest.Rules

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 62 | Steps 611-620*
