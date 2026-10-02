# Part 80: Capstone Project – Full Food Delivery App
## Steps 791-800: Putting It All Together

---

## Step 791: Project Structure

```
FoodDelivery.sln
├── src/
│   ├── FoodDelivery.Domain/           # Entities, Value Objects, Events
│   │   ├── Aggregates/
│   │   │   ├── Order.cs
│   │   │   ├── Restaurant.cs
│   │   │   └── Customer.cs
│   │   ├── Events/
│   │   │   ├── OrderCreated.cs
│   │   │   ├── OrderConfirmed.cs
│   │   │   └── ...
│   │   ├── Repositories/              # Interfaces only
│   │   │   ├── IOrderRepository.cs
│   │   │   └── ICustomerRepository.cs
│   │   └── Services/
│   │       └── IPricingService.cs
│   │
│   ├── FoodDelivery.Application/      # Use cases
│   │   ├── Commands/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderHandler.cs
│   │   │   └── CancelOrderCommand.cs
│   │   ├── Queries/
│   │   │   ├── GetOrderQuery.cs
│   │   │   └── GetRestaurantsQuery.cs
│   │   └── DTOs/
│   │       └── OrderDto.cs
│   │
│   ├── FoodDelivery.Infrastructure/   # SQLite, HTTP, etc.
│   │   ├── Persistence/
│   │   │   ├── OrderRepository.cs
│   │   │   └── DatabaseFactory.cs
│   │   ├── Api/
│   │   │   ├── RestaurantApiClient.cs
│   │   │   └── PaymentApiClient.cs
│   │   └── SignalR/
│   │       └── LiveOrderHub.cs
│   │
│   └── FoodDelivery.Maui/             # UI Layer
│       ├── Pages/
│       │   ├── Home/
│       │   ├── Restaurant/
│       │   ├── Cart/
│       │   ├── Order/
│       │   └── Profile/
│       ├── ViewModels/
│       ├── Controls/
│       └── MauiProgram.cs
│
└── tests/
    ├── FoodDelivery.Domain.Tests/
    ├── FoodDelivery.Application.Tests/
    └── FoodDelivery.UI.Tests/
```

---

## Step 792: MauiProgram – Full DI Setup

```csharp
// ============================================
// Complete Dependency Injection Configuration
// ============================================

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("Inter-Regular.ttf", "Regular");
                fonts.AddFont("Inter-SemiBold.ttf", "SemiBold");
                fonts.AddFont("Inter-Bold.ttf", "Bold");
            })
            .UseMauiMaps();
        
        RegisterCoreServices(builder.Services);
        RegisterInfrastructure(builder.Services);
        RegisterApplication(builder.Services);
        RegisterUI(builder.Services);
        
        return builder.Build();
    }
    
    private static void RegisterCoreServices(IServiceCollection s)
    {
        s.AddSingleton<IConnectivity>(_ => Connectivity.Current);
        s.AddSingleton<IPreferences>(_ => Preferences.Default);
        s.AddSingleton<ISecureStorage>(_ => SecureStorage.Default);
        s.AddSingleton<HapticService>();
        s.AddSingleton<ThaiFormatter>();
    }
    
    private static void RegisterInfrastructure(IServiceCollection s)
    {
        // Database
        s.AddSingleton<SQLiteAsyncConnection>(_ =>
        {
            var path = Path.Combine(FileSystem.AppDataDirectory, "app.db");
            return new SQLiteAsyncConnection(path);
        });
        s.AddSingleton<DatabaseMigrator>();
        
        // Repositories
        s.AddScoped<IOrderRepository, SqliteOrderRepository>();
        s.AddScoped<ICustomerRepository, SqliteCustomerRepository>();
        s.AddScoped<IMenuRepository, CachedMenuRepository>();
        
        // HTTP
        s.AddHttpClient<IRestaurantApiClient, RestaurantApiClient>(c =>
        {
            c.BaseAddress = new Uri(AppConfiguration.Current.ApiBaseUrl);
            c.Timeout = TimeSpan.FromSeconds(30);
        }).AddTransientHttpErrorPolicy(p =>
            p.WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i))));
        
        // SignalR
        s.AddSingleton<SignalRConnectionService>();
        s.AddSingleton<LiveOrderTracker>();
        s.AddSingleton<ChatService>();
        
        // Auth
        s.AddSingleton<ITokenService, JwtTokenService>();
        s.AddSingleton<IAuthorizationService, RoleBasedAuthorizationService>();
        
        // Features
        s.AddSingleton<IFeatureFlagService, RemoteFeatureFlagService>();
        s.AddSingleton<CanaryReleaseService>();
        
        // Analytics
        s.AddSingleton<UserAnalyticsService>();
        s.AddSingleton<RecommendationService>();
        s.AddSingleton<PersonalizationEngine>();
        
        // Sync
        s.AddSingleton<SyncEngine>();
        s.AddSingleton<BackgroundSyncService>();
        s.AddSingleton<OfflineQueue>();
    }
    
    private static void RegisterApplication(IServiceCollection s)
    {
        s.AddScoped<ICommandBus, CommandBus>();
        s.AddScoped<IQueryBus, QueryBus>();
        s.AddScoped<IUnitOfWork, SqliteUnitOfWork>();
        
        // Command handlers
        s.AddScoped<ICommandHandler<PlaceOrderCommand>, PlaceOrderHandler>();
        s.AddScoped<ICommandHandler<CancelOrderCommand>, CancelOrderHandler>();
        
        // Pipeline behaviors
        s.AddScoped(typeof(IPipelineBehavior<>), typeof(ValidationBehavior<>));
        s.AddScoped(typeof(IPipelineBehavior<>), typeof(TransactionBehavior<>));
        s.AddScoped(typeof(IPipelineBehavior<>), typeof(LoggingBehavior<>));
        
        // Domain event handlers
        s.AddScoped<IDomainEventHandler<OrderConfirmed>, OrderConfirmedNotificationHandler>();
        s.AddScoped<IDomainEventHandler<OrderCreated>, OrderCreatedAnalyticsHandler>();
    }
    
    private static void RegisterUI(IServiceCollection s)
    {
        // Pages
        s.AddTransient<HomePage>();
        s.AddTransient<RestaurantPage>();
        s.AddTransient<CartPage>();
        s.AddTransient<CheckoutPage>();
        s.AddTransient<OrderTrackingPage>();
        s.AddTransient<ProfilePage>();
        s.AddTransient<SearchPage>();
        
        // ViewModels
        s.AddTransient<HomeViewModel>();
        s.AddTransient<RestaurantViewModel>();
        s.AddTransient<CartViewModel>();
        s.AddTransient<CheckoutViewModel>();
        s.AddTransient<OrderTrackingViewModel>();
        s.AddTransient<ProfileViewModel>();
        s.AddTransient<SearchViewModel>();
    }
}
```

---

## Step 793: App Shell Navigation

```xml
<!-- AppShell.xaml -->
<?xml version="1.0" encoding="UTF-8" ?>
<Shell x:Class="FoodDelivery.AppShell"
       xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:pages="clr-namespace:FoodDelivery.Pages"
       Shell.NavBarIsVisible="False">

    <!-- Auth section (no tab bar) -->
    <ShellContent Route="login" ContentTemplate="{DataTemplate pages:LoginPage}" />
    <ShellContent Route="register" ContentTemplate="{DataTemplate pages:RegisterPage}" />

    <!-- Main section (tab bar) -->
    <TabBar>
        <Tab Title="หน้าหลัก" Icon="home.png">
            <ShellContent Route="home" ContentTemplate="{DataTemplate pages:HomePage}" />
        </Tab>
        
        <Tab Title="ค้นหา" Icon="search.png">
            <ShellContent Route="search" ContentTemplate="{DataTemplate pages:SearchPage}" />
        </Tab>
        
        <Tab Title="ออเดอร์" Icon="orders.png">
            <ShellContent Route="orders" ContentTemplate="{DataTemplate pages:OrdersPage}" />
        </Tab>
        
        <Tab Title="โปรไฟล์" Icon="profile.png">
            <ShellContent Route="profile" ContentTemplate="{DataTemplate pages:ProfilePage}" />
        </Tab>
    </TabBar>

</Shell>
```

```csharp
// AppShell.xaml.cs – Route registration
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        
        Routing.RegisterRoute("restaurant", typeof(RestaurantPage));
        Routing.RegisterRoute("cart", typeof(CartPage));
        Routing.RegisterRoute("checkout", typeof(CheckoutPage));
        Routing.RegisterRoute("tracking", typeof(OrderTrackingPage));
        Routing.RegisterRoute("chat", typeof(ChatPage));
        Routing.RegisterRoute("payment/promptpay", typeof(PromptPayPage));
        Routing.RegisterRoute("payment/card", typeof(CardInputPage));
        Routing.RegisterRoute("rating", typeof(RatingPage));
    }
}

// Navigation helper
public static class Navigation
{
    public static Task GoToRestaurantAsync(string restaurantId)
        => Shell.Current.GoToAsync($"restaurant?id={restaurantId}");
    
    public static Task GoToCartAsync()
        => Shell.Current.GoToAsync("cart");
    
    public static Task GoToTrackingAsync(string orderId)
        => Shell.Current.GoToAsync($"tracking?orderId={orderId}");
    
    public static Task GoBackAsync()
        => Shell.Current.GoToAsync("..");
}
```

---

## Step 794: Home Page Complete

```csharp
// ============================================
// Home Page ViewModel – Complete Implementation
// ============================================

public partial class HomeViewModel : ObservableObject
{
    private readonly IRestaurantApiClient _api;
    private readonly RecommendationService _recs;
    private readonly IConnectivity _connectivity;
    private readonly UserAnalyticsService _analytics;
    
    [ObservableProperty] private bool _isRefreshing;
    [ObservableProperty] private bool _isLoading = true;
    [ObservableProperty] private string? _errorMessage;
    [ObservableProperty] private ObservableCollection<Restaurant> _nearbyRestaurants = new();
    [ObservableProperty] private ObservableCollection<Restaurant> _popularRestaurants = new();
    [ObservableProperty] private ObservableCollection<RecommendedItem> _forYou = new();
    [ObservableProperty] private ObservableCollection<Category> _categories = new();
    [ObservableProperty] private string _greeting = "";
    
    public HomeViewModel(IRestaurantApiClient api, RecommendationService recs,
        IConnectivity connectivity, UserAnalyticsService analytics)
    {
        _api = api; _recs = recs;
        _connectivity = connectivity; _analytics = analytics;
        
        Greeting = BuildGreeting();
    }
    
    public async Task InitializeAsync()
    {
        _analytics.TrackScreenView("Home");
        await LoadDataAsync();
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        IsRefreshing = true;
        await LoadDataAsync();
        IsRefreshing = false;
    }
    
    private async Task LoadDataAsync()
    {
        IsLoading = true;
        ErrorMessage = null;
        
        try
        {
            var tasks = new List<Task>
            {
                LoadRestaurantsAsync(),
                LoadCategoriesAsync(),
                LoadRecommendationsAsync()
            };
            
            await Task.WhenAll(tasks);
        }
        catch (HttpRequestException) when
            (_connectivity.NetworkAccess != NetworkAccess.Internet)
        {
            ErrorMessage = "คุณอยู่ในโหมดออฟไลน์";
        }
        catch (Exception ex)
        {
            ErrorMessage = $"เกิดข้อผิดพลาด: {ex.Message}";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private async Task LoadRestaurantsAsync()
    {
        var (nearby, popular) = await _api.GetRestaurantsAsync();
        NearbyRestaurants = new ObservableCollection<Restaurant>(nearby);
        PopularRestaurants = new ObservableCollection<Restaurant>(popular);
    }
    
    private async Task LoadCategoriesAsync()
    {
        var cats = await _api.GetCategoriesAsync();
        Categories = new ObservableCollection<Category>(cats);
    }
    
    private async Task LoadRecommendationsAsync()
    {
        var recs = await _recs.GetPersonalizedAsync("current-user", 10);
        ForYou = new ObservableCollection<RecommendedItem>(recs);
    }
    
    [RelayCommand]
    private async Task SelectRestaurantAsync(Restaurant restaurant)
    {
        _analytics.TrackButtonTap("select_restaurant", restaurant.Id);
        await Navigation.GoToRestaurantAsync(restaurant.Id);
    }
    
    [RelayCommand]
    private async Task SelectCategoryAsync(Category category)
        => await Shell.Current.GoToAsync($"search?category={category.Id}");
    
    private static string BuildGreeting()
    {
        var hour = DateTime.Now.Hour;
        return hour switch
        {
            < 12 => "อรุณสวัสดิ์ 👋",
            < 18 => "สวัสดีตอนบ่าย 👋",
            _ => "สวัสดีตอนเย็น 👋"
        };
    }
}
```

---

## Step 795: Complete Cart Flow

```csharp
// ============================================
// Cart ViewModel – Full Implementation
// ============================================

public partial class CartViewModel : ObservableObject
{
    private readonly ICartRepository _cartRepo;
    private readonly ICommandBus _bus;
    private readonly UserAnalyticsService _analytics;
    private readonly HapticService _haptic;
    
    [ObservableProperty] private ObservableCollection<CartItem> _items = new();
    [ObservableProperty] private decimal _subtotal;
    [ObservableProperty] private decimal _deliveryFee;
    [ObservableProperty] private decimal _discount;
    [ObservableProperty] private decimal _total;
    [ObservableProperty] private string? _promoCode;
    [ObservableProperty] private bool _isPromoApplied;
    [ObservableProperty] private bool _isCheckingOut;
    
    public bool HasItems => Items.Count > 0;
    
    public CartViewModel(ICartRepository cartRepo, ICommandBus bus,
        UserAnalyticsService analytics, HapticService haptic)
    {
        _cartRepo = cartRepo; _bus = bus;
        _analytics = analytics; _haptic = haptic;
        
        Items.CollectionChanged += (_, _) =>
        {
            OnPropertyChanged(nameof(HasItems));
            RecalculateTotals();
        };
    }
    
    public async Task LoadAsync()
    {
        var items = await _cartRepo.GetItemsAsync();
        Items = new ObservableCollection<CartItem>(items);
        RecalculateTotals();
    }
    
    [RelayCommand]
    private async Task IncrementAsync(CartItem item)
    {
        item.Quantity++;
        _haptic.LightTap();
        await _cartRepo.UpdateAsync(item);
        RecalculateTotals();
    }
    
    [RelayCommand]
    private async Task DecrementAsync(CartItem item)
    {
        if (item.Quantity <= 1)
        {
            await RemoveAsync(item);
            return;
        }
        item.Quantity--;
        _haptic.LightTap();
        await _cartRepo.UpdateAsync(item);
        RecalculateTotals();
    }
    
    [RelayCommand]
    private async Task RemoveAsync(CartItem item)
    {
        Items.Remove(item);
        await _cartRepo.RemoveAsync(item.Id);
        AccessibilityHelper.Announce($"ลบ {item.Name} ออกจากตะกร้าแล้ว");
    }
    
    [RelayCommand]
    private async Task ApplyPromoAsync()
    {
        if (string.IsNullOrWhiteSpace(PromoCode)) return;
        
        // Validate promo code
        var discount = await _cartRepo.ValidatePromoAsync(PromoCode, Subtotal);
        if (discount > 0)
        {
            Discount = discount;
            IsPromoApplied = true;
            _haptic.Success();
            AccessibilityHelper.Announce($"ใช้โค้ดส่วนลด ฿{discount:F0} สำเร็จ");
        }
        else
        {
            _haptic.Error();
            await Shell.Current.DisplayAlert("ข้อผิดพลาด",
                "โค้ดส่วนลดไม่ถูกต้องหรือหมดอายุ", "ตกลง");
        }
        
        RecalculateTotals();
    }
    
    [RelayCommand]
    private async Task CheckoutAsync()
    {
        if (!HasItems) return;
        
        IsCheckingOut = true;
        
        try
        {
            _analytics.TrackButtonTap("checkout", Items.Count.ToString());
            await Navigation.GoToCartAsync();
        }
        finally
        {
            IsCheckingOut = false;
        }
    }
    
    private void RecalculateTotals()
    {
        Subtotal = Items.Sum(i => i.Price * i.Quantity);
        DeliveryFee = Subtotal >= 300 ? 0 : 40;
        Total = Subtotal + DeliveryFee - Discount;
    }
}
```

---

## Step 796: End-to-End Order Flow

```csharp
// ============================================
// Complete Order Placement Flow
// ============================================

public partial class CheckoutViewModel : ObservableObject
{
    private readonly ICommandBus _bus;
    private readonly IPaymentService _payment;
    private readonly CartViewModel _cart;
    private readonly UserAnalyticsService _analytics;
    
    [ObservableProperty] private string _selectedPaymentMethod = "wallet";
    [ObservableProperty] private string _deliveryAddress = "";
    [ObservableProperty] private string _deliveryNotes = "";
    [ObservableProperty] private bool _isPlacingOrder;
    [ObservableProperty] private string? _error;
    
    public CheckoutViewModel(ICommandBus bus, IPaymentService payment,
        CartViewModel cart, UserAnalyticsService analytics)
    {
        _bus = bus; _payment = payment;
        _cart = cart; _analytics = analytics;
    }
    
    [RelayCommand]
    private async Task PlaceOrderAsync()
    {
        if (!Validate()) return;
        
        IsPlacingOrder = true;
        Error = null;
        
        try
        {
            // 1. Process payment
            var paymentResult = await _payment.ChargeAsync(
                _cart.Total, SelectedPaymentMethod);
            
            if (!paymentResult.Success)
            {
                Error = paymentResult.Error ?? "การชำระเงินล้มเหลว";
                return;
            }
            
            // 2. Place order
            var command = new PlaceOrderCommand(
                CustomerId: Guid.NewGuid(), // from auth
                Items: _cart.Items.Select(i => (i.MenuItemId, i.Quantity)).ToList()
            );
            
            await _bus.SendAsync(command);
            
            // 3. Track
            _analytics.TrackPurchase(_cart.Total, _cart.Items.Count);
            
            // 4. Navigate to tracking
            var orderId = paymentResult.OrderId ?? Guid.NewGuid().ToString();
            await Shell.Current.GoToAsync($"//tracking?orderId={orderId}",
                new Dictionary<string, object> { ["orderId"] = orderId });
        }
        catch (Exception ex)
        {
            Error = $"เกิดข้อผิดพลาด: {ex.Message}";
        }
        finally
        {
            IsPlacingOrder = false;
        }
    }
    
    private bool Validate()
    {
        if (string.IsNullOrWhiteSpace(DeliveryAddress))
        {
            Error = "กรุณากรอกที่อยู่จัดส่ง";
            return false;
        }
        return true;
    }
}
```

---

## Step 797: Integration Tests

```csharp
// ============================================
// End-to-End Integration Tests
// ============================================

[TestFixture]
public class OrderFlowIntegrationTests
{
    private IServiceProvider _sp = null!;
    
    [SetUp]
    public void SetUp()
    {
        var services = new ServiceCollection();
        
        // Register in-memory implementations
        services.AddSingleton<SQLiteAsyncConnection>(
            _ => new SQLiteAsyncConnection(":memory:"));
        services.AddScoped<IOrderRepository, SqliteOrderRepository>();
        services.AddScoped<IMenuRepository, InMemoryMenuRepository>();
        services.AddScoped<ICommandHandler<PlaceOrderCommand>, PlaceOrderHandler>();
        services.AddScoped<ICommandBus, CommandBus>();
        services.AddScoped<IUnitOfWork, SqliteUnitOfWork>();
        services.AddSingleton<SpyEventBus>();
        services.AddScoped<IEventBus>(sp => sp.GetRequiredService<SpyEventBus>());
        
        _sp = services.BuildServiceProvider();
        
        // Run migrations
        _sp.GetRequiredService<SQLiteAsyncConnection>()
            .CreateTableAsync<OrderEntity>().Wait();
    }
    
    [Test]
    public async Task PlaceOrder_WithValidItems_SavesOrderAndEmitsEvent()
    {
        var bus = _sp.GetRequiredService<ICommandBus>();
        var events = _sp.GetRequiredService<SpyEventBus>();
        var repo = _sp.GetRequiredService<IOrderRepository>();
        
        var customerId = Guid.NewGuid();
        var cmd = new PlaceOrderCommand(customerId,
            new List<(string, int)> { ("item-1", 2), ("item-2", 1) });
        
        await bus.SendAsync(cmd);
        
        // Verify order saved
        var orders = await repo.GetByCustomerAsync(customerId);
        Assert.That(orders.Count, Is.EqualTo(1));
        Assert.That(orders[0].Lines.Count, Is.EqualTo(2));
        
        // Verify domain event emitted
        Assert.That(events.HasPublished<OrderCreated>(), Is.True);
        var evt = events.GetLastPublished<OrderCreated>()!;
        Assert.That(evt.CustomerId.Value, Is.EqualTo(customerId));
    }
    
    [Test]
    public async Task PlaceOrder_WithEmptyItems_ThrowsValidationException()
    {
        var bus = _sp.GetRequiredService<ICommandBus>();
        var cmd = new PlaceOrderCommand(Guid.NewGuid(), new List<(string, int)>());
        
        Assert.ThrowsAsync<DomainException>(() => bus.SendAsync(cmd));
    }
}
```

---

## Step 798: Performance Benchmarks

```csharp
// ============================================
// Capstone Performance Benchmarks
// ============================================

[TestFixture]
public class CapstonePerformanceTests
{
    private HomeViewModel _homeVm = null!;
    private SearchViewModel _searchVm = null!;
    
    [SetUp]
    public void SetUp()
    {
        // Wire up with in-memory fakes
        var db = new SQLiteAsyncConnection(":memory:");
        _searchVm = new SearchViewModel(new InMemorySearchTrie(), new ThaiTextSearchEngine(),
            new SynchronousDebouncer());
    }
    
    [Test]
    public async Task HomePageLoad_Under2Seconds()
    {
        var sw = Stopwatch.StartNew();
        await _homeVm.InitializeAsync();
        sw.Stop();
        
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(2000),
            "Home page must load in under 2 seconds");
    }
    
    [Test]
    public void SearchIndex_1000Items_Under50Ms()
    {
        var engine = new ThaiTextSearchEngine();
        var items = Enumerable.Range(0, 1000).Select(i => new MenuItem
        {
            Id = i.ToString(),
            Name = $"เมนู {i}",
            Price = (i % 10 + 1) * 50
        }).ToList();
        
        var sw = Stopwatch.StartNew();
        engine.IndexMenuItems(items);
        sw.Stop();
        
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(50),
            "Indexing 1000 items must be under 50ms");
    }
    
    [Test]
    public void Search_Returns_Under10Ms()
    {
        var engine = new ThaiTextSearchEngine();
        var items = Enumerable.Range(0, 500).Select(i =>
            new MenuItem { Id = i.ToString(), Name = $"ข้าวผัด {i}", Price = 80 }).ToList();
        engine.IndexMenuItems(items);
        
        var sw = Stopwatch.StartNew();
        var results = engine.Search("ข้าวผัด");
        sw.Stop();
        
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(10),
            "Search must complete under 10ms");
        Assert.That(results, Is.Not.Empty);
    }
}
```

---

## Step 799: Release Checklist

```csharp
// ============================================
// Production Release Checklist
// ============================================

public class ReleaseChecklist
{
    private readonly IList<ChecklistItem> _items = new List<ChecklistItem>
    {
        // Security
        new("SEC-01", "ปิดโหมด Debug ทั้งหมด"),
        new("SEC-02", "เปิดใช้งาน certificate pinning"),
        new("SEC-03", "ตรวจสอบ API keys ไม่อยู่ใน source code"),
        new("SEC-04", "เปิดใช้งาน SQLCipher สำหรับ production"),
        new("SEC-05", "ตรวจสอบ PDPA consent flow ทำงานถูกต้อง"),
        
        // Performance
        new("PERF-01", "ทดสอบเวลา cold start < 3 วินาที"),
        new("PERF-02", "ทดสอบ memory leak ด้วย profiler"),
        new("PERF-03", "ตรวจสอบ image cache ไม่เกิน 100MB"),
        new("PERF-04", "ทดสอบ offline mode ทุก scenario"),
        
        // Functionality
        new("FUNC-01", "ทดสอบ payment flow ทุก payment method"),
        new("FUNC-02", "ทดสอบ push notification iOS และ Android"),
        new("FUNC-03", "ทดสอบ deep link ทุก route"),
        new("FUNC-04", "ทดสอบ force update dialog"),
        new("FUNC-05", "ทดสอบ biometric login ทั้ง 2 platform"),
        
        // Accessibility
        new("A11Y-01", "ทดสอบ VoiceOver (iOS) และ TalkBack (Android)"),
        new("A11Y-02", "ทดสอบ large text size"),
        new("A11Y-03", "ตรวจสอบ color contrast ratio >= 4.5:1"),
        
        // Deployment
        new("DEP-01", "เพิ่ม version number และ build number"),
        new("DEP-02", "อัปเดต changelog"),
        new("DEP-03", "สร้าง signed APK/AAB"),
        new("DEP-04", "อัปโหลด dSYM สำหรับ crash symbolication"),
        new("DEP-05", "ทดสอบบน TestFlight และ Internal Testing"),
    };
    
    public async Task RunAutomatedChecksAsync()
    {
        var launchChecklist = new LaunchChecklist();
        var result = await launchChecklist.RunAsync();
        
        foreach (var (check, passed) in result)
        {
            var item = _items.FirstOrDefault(i => i.Code == check);
            if (item != null) item.Passed = passed;
        }
        
        var failed = _items.Where(i => !i.Passed).ToList();
        if (failed.Any())
        {
            throw new ReleaseBlockedException(
                $"รายการที่ยังไม่ผ่าน: {string.Join(", ", failed.Select(f => f.Code))}");
        }
    }
    
    public string GenerateReport()
    {
        var passed = _items.Count(i => i.Passed);
        var total = _items.Count;
        
        return $"Release Checklist: {passed}/{total} รายการผ่าน\n" +
               string.Join("\n", _items.Select(i =>
                   $"{(i.Passed ? "✅" : "❌")} [{i.Code}] {i.Description}"));
    }
}

public class ChecklistItem
{
    public string Code { get; }
    public string Description { get; }
    public bool Passed { get; set; }
    
    public ChecklistItem(string code, string description)
    { Code = code; Description = description; }
}

public class ReleaseBlockedException : Exception
{
    public ReleaseBlockedException(string message) : base(message) { }
}
```

---

## Step 800: World-Class Quality Standards

```csharp
// ============================================
// Quality Metrics & Standards
// ============================================

public static class QualityStandards
{
    // Performance Budgets
    public static class Performance
    {
        public const int ColdStartMaxMs = 3000;      // < 3 sec cold start
        public const int HotStartMaxMs = 1000;       // < 1 sec hot start
        public const int ApiCallMaxMs = 2000;        // < 2 sec API calls (p95)
        public const int FrameDropBudget = 2;        // < 2 dropped frames per interaction
        public const int MemoryMaxMb = 200;          // < 200MB memory usage
        public const int CacheMaxMb = 100;           // < 100MB disk cache
    }
    
    // Test Coverage Targets
    public static class Testing
    {
        public const double DomainCoverageMin = 0.95;       // 95% domain coverage
        public const double ApplicationCoverageMin = 0.85;  // 85% application coverage
        public const double UiTestCoverageMin = 0.70;       // 70% UI test coverage
        public const double MutationScoreMin = 0.80;        // 80% mutation score
    }
    
    // Accessibility
    public static class Accessibility
    {
        public const double MinContrastRatioNormal = 4.5;  // WCAG AA
        public const double MinContrastRatioLarge = 3.0;   // WCAG AA large text
        public const double MinTouchTargetDp = 44;         // iOS HIG minimum
        public const bool RequireScreenReaderLabels = true;
    }
    
    // Security
    public static class Security
    {
        public const bool RequireCertificatePinning = true;
        public const bool RequireEncryptedStorage = true;
        public const bool RequireJailbreakDetection = true;
        public const int SessionTimeoutMinutes = 30;
        public const int MaxLoginAttempts = 5;
    }
    
    // Reliability
    public static class Reliability
    {
        public const double CrashFreeRateMin = 0.999;  // 99.9% crash-free
        public const double UptimeMin = 0.9999;        // 99.99% uptime
        public const double SyncSuccessRateMin = 0.99; // 99% sync success
    }
}
```

---

## สรุป Part 80 – Capstone Project

ใน Part 80 เราได้นำทุกสิ่งที่เรียนรู้มาประกอบเข้าด้วยกัน:

1. **Project Structure** - Clean Architecture folder layout ที่สมบูรณ์
2. **MauiProgram** - Full DI setup: Core, Infrastructure, Application, UI
3. **Shell Navigation** - Route registration, TabBar, auth flow
4. **Home ViewModel** - Parallel load, error handling, analytics
5. **Cart Flow** - Increment/decrement, promo codes, totals
6. **Checkout Flow** - Payment, command bus, tracking navigation
7. **Integration Tests** - Full order flow with SpyEventBus
8. **Performance Tests** - Load time, search, index benchmarks
9. **Release Checklist** - 20 automated+manual checks across 4 areas
10. **World-Class Standards** - Performance budgets, test targets, accessibility, security

---

**ยินดีด้วย! คุณได้เรียนรู้ครบ 800 steps แห่งการพัฒนาแอป .NET MAUI ระดับโลก!**

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 80 | Steps 791-800*
