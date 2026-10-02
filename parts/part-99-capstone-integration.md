# Part 99: Final Capstone Integration
## Steps 981-990: Integrating All Patterns Into Production-Ready App

---

## Step 981: Complete App Bootstrap

```csharp
// ============================================
// MauiProgram.cs — Production Configuration
// ============================================

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            .UseMauiCommunityToolkit()
            .UseMauiMaps()
            .ConfigureFonts(ConfigureFonts)
            .ConfigureEssentials(e => e.UseVersionTracking());
        
        // Logging
        builder.Logging.AddDebug();
        
        // Configuration
        RegisterConfiguration(builder.Services);
        
        // Infrastructure
        RegisterInfrastructure(builder.Services);
        
        // Domain & Application
        RegisterApplication(builder.Services);
        
        // Platform services
        RegisterPlatformServices(builder.Services);
        
        // HTTP Clients
        RegisterHttpClients(builder);
        
        // ViewModels
        RegisterViewModels(builder.Services);
        
        var app = builder.Build();
        
        // Post-build initialization
        _ = InitializeAsync(app.Services);
        
        return app;
    }
    
    private static void RegisterConfiguration(IServiceCollection services)
    {
        var env = AppConfig.DetermineEnvironment();
        var config = AppConfig.FromEnvironment(env);
        services.AddSingleton(config);
    }
    
    private static void RegisterInfrastructure(IServiceCollection services)
    {
        // Database
        services.AddSingleton<SQLiteAsyncConnection>(sp =>
        {
            var dbPath = Path.Combine(FileSystem.AppDataDirectory, "fooddelivery.db");
            return new SQLiteAsyncConnection(dbPath);
        });
        
        // Cache
        services.AddSingleton<IMemoryCache, MemoryCache>();
        services.AddSingleton<DiskCache>();
        services.AddSingleton<MultiLayerCache>();
        
        // Repositories
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IMenuRepository, MenuRepository>();
        services.AddScoped<IRestaurantRepository, RestaurantRepository>();
        
        // Auth
        services.AddSingleton<OAuthService>();
        services.AddSingleton<AuthorizationService>();
        services.AddSingleton<SessionTimeoutService>();
        services.AddSingleton<TenantContext>();
    }
    
    private static void RegisterApplication(IServiceCollection services)
    {
        services.AddScoped<PlaceOrderUseCase>();
        services.AddScoped<CartService>();
        services.AddScoped<LoyaltyService>();
        services.AddScoped<PromPayService>();
        services.AddScoped<LocationService>();
        services.AddScoped<NotificationPreferences>();
        services.AddScoped<LocalizationService>();
    }
    
    private static void RegisterHttpClients(MauiAppBuilder builder)
    {
        builder.Services.AddHttpClient<IFoodApiClient, FoodApiClient>(c =>
        {
            var config = builder.Services.BuildServiceProvider().GetRequiredService<AppConfig>();
            c.BaseAddress = new Uri(config.ApiBaseUrl);
        })
        .AddResilienceHandler("food-api", ConfigureResilience)
        .AddHttpMessageHandler<CachingHttpHandler>()
        .AddHttpMessageHandler<TenantHttpHandler>();
    }
    
    private static void RegisterViewModels(IServiceCollection services)
    {
        services.AddTransient<HomeViewModel>();
        services.AddTransient<RestaurantViewModel>();
        services.AddTransient<CartViewModel>();
        services.AddTransient<CheckoutViewModel>();
        services.AddTransient<OrderTrackingViewModel>();
        services.AddTransient<ProfileViewModel>();
        services.AddTransient<NotificationCenterViewModel>();
        services.AddTransient<LoyaltyViewModel>();
    }
    
    private static void ConfigureResilience(ResiliencePipelineBuilder<HttpResponseMessage> pipeline)
    {
        pipeline.AddRetry(new Polly.Retry.HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = Polly.DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(300)
        });
        pipeline.AddCircuitBreaker(new Polly.CircuitBreaker.HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(15)
        });
        pipeline.AddTimeout(TimeSpan.FromSeconds(10));
    }
    
    private static void RegisterPlatformServices(IServiceCollection services)
    {
        services.AddSingleton<IPermissionService, PermissionService>();
        services.AddSingleton<ILocalNotificationService, LocalNotificationService>();
        services.AddSingleton<NetworkQualityMonitor>();
        services.AddSingleton<CrashReportingService>();
        services.AddSingleton<AccessibilityAnnouncementService>();
    }
    
    private static void ConfigureFonts(IFontCollection fonts)
    {
        fonts.AddFont("Sarabun-Regular.ttf", "Sarabun");
        fonts.AddFont("Sarabun-Bold.ttf", "SarabunBold");
        fonts.AddFont("Sarabun-Medium.ttf", "SarabunMedium");
        fonts.AddFont("fa-solid-900.ttf", "FontAwesome");
    }
    
    private static async Task InitializeAsync(IServiceProvider services)
    {
        await Task.Delay(500); // Let first frame render
        
        var db = services.GetRequiredService<SQLiteAsyncConnection>();
        await new MigrationRunner(db, GetAllMigrations()).MigrateAsync();
        
        var auth = services.GetRequiredService<OAuthService>();
        var session = services.GetRequiredService<SessionTimeoutService>();
        session.Start();
        
        var privacy = services.GetRequiredService<PrivacyConsentService>();
        if (!privacy.HasConsented())
            await Shell.Current.GoToAsync("//privacy");
        
        StartupOptimizer.RecordPhase("app ready");
    }
    
    private static IEnumerable<IMigration> GetAllMigrations()
        => new IMigration[]
        {
            new Migration001CreateOrders(),
            new Migration002AddDeliveryAddress()
        };
}
```

---

## Step 982: Complete Order Flow

```csharp
// ============================================
// End-to-End: Browse → Cart → Checkout → Track
// ============================================

public class CheckoutViewModel : ObservableObject
{
    private readonly CartService _cart;
    private readonly PlaceOrderUseCase _placeOrder;
    private readonly OmisePaymentGateway _payment;
    private readonly AccessibilityAnnouncementService _a11y;
    
    [ObservableProperty] private CartSummary? _cartSummary;
    [ObservableProperty] private string _deliveryAddress = "";
    [ObservableProperty] private string _selectedPayment = "card";
    [ObservableProperty] private bool _isProcessing;
    [ObservableProperty] private string? _errorMessage;
    
    public CheckoutViewModel(
        CartService cart,
        PlaceOrderUseCase placeOrder,
        OmisePaymentGateway payment,
        AccessibilityAnnouncementService a11y)
    {
        _cart = cart;
        _placeOrder = placeOrder;
        _payment = payment;
        _a11y = a11y;
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        CartSummary = await _cart.GetSummaryAsync();
    }
    
    [RelayCommand(CanExecute = nameof(CanCheckout))]
    private async Task PlaceOrderAsync(CancellationToken ct)
    {
        if (CartSummary == null) return;
        
        IsProcessing = true;
        ErrorMessage = null;
        
        try
        {
            // 1. Tokenize card via Omise.js (simplified)
            var paymentToken = await _payment.TokenizeCardAsync();
            
            // 2. Build command
            var cmd = new PlaceOrderCommand(
                UserId: Guid.NewGuid(), // From auth service
                RestaurantId: CartSummary.RestaurantId,
                ItemIds: CartSummary.Items.Select(i => i.MenuItemId).ToList(),
                Quantities: CartSummary.Items.ToDictionary(i => i.MenuItemId, i => i.Quantity),
                DeliveryAddress: ParseAddress(DeliveryAddress),
                PaymentToken: paymentToken);
            
            // 3. Execute use case (payment + save + events)
            var result = await _placeOrder.ExecuteAsync(cmd);
            
            if (!result.IsSuccess)
            {
                ErrorMessage = result.Error;
                _a11y.AnnounceError(result.Error ?? "เกิดข้อผิดพลาด");
                return;
            }
            
            // 4. Navigate to tracking
            _a11y.Announce("สั่งอาหารสำเร็จ กำลังไปที่หน้าติดตามคำสั่งซื้อ");
            await Shell.Current.GoToAsync(
                $"tracking?orderId={result.OrderId!.Value}",
                animate: true);
        }
        catch (Exception ex)
        {
            ErrorMessage = "เกิดข้อผิดพลาดในการสั่งซื้อ กรุณาลองใหม่อีกครั้ง";
            _a11y.AnnounceError(ErrorMessage);
        }
        finally
        {
            IsProcessing = false;
        }
    }
    
    private bool CanCheckout() =>
        !IsProcessing &&
        CartSummary?.Items.Any() == true &&
        !string.IsNullOrWhiteSpace(DeliveryAddress);
    
    private static DeliveryAddress ParseAddress(string raw)
        => new DeliveryAddress(raw, "", "", "");
}
```

---

## Step 983: Complete Notification System

```csharp
// ============================================
// Orchestrate: FCM → Local Notification → Deep Link → In-App Center
// ============================================

public class NotificationOrchestrator
{
    private readonly FcmPushNotificationService _fcm;
    private readonly LocalNotificationService _local;
    private readonly DeepLinkService _deepLink;
    private readonly InAppNotificationRepository _inApp;
    private readonly NotificationPreferences _prefs;
    private readonly AccessibilityAnnouncementService _a11y;
    
    public NotificationOrchestrator(
        FcmPushNotificationService fcm,
        LocalNotificationService local,
        DeepLinkService deepLink,
        InAppNotificationRepository inApp,
        NotificationPreferences prefs,
        AccessibilityAnnouncementService a11y)
    {
        _fcm = fcm;
        _local = local;
        _deepLink = deepLink;
        _inApp = inApp;
        _prefs = prefs;
        _a11y = a11y;
        
        _fcm.NotificationReceived += OnPushReceived;
        _fcm.NotificationTapped += OnPushTapped;
    }
    
    private async void OnPushReceived(object? sender, PushNotificationPayload payload)
    {
        // 1. Save to in-app center
        await _inApp.SaveAsync(new InAppNotification
        {
            Id = Guid.NewGuid().ToString(),
            Title = payload.Title,
            Body = payload.Body,
            Type = payload.Type,
            DeepLink = payload.DeepLink,
            IsRead = false,
            ReceivedAt = DateTime.UtcNow
        });
        
        // 2. Respect quiet hours
        if (_prefs.IsQuietHour(DateTime.Now))
        {
            WeakReferenceMessenger.Default.Send(new UnreadCountChangedMessage(
                await _inApp.GetUnreadCountAsync()));
            return;
        }
        
        // 3. Check category preference
        if (!_prefs.IsEnabled(payload.Type)) return;
        
        // 4. Show local notification
        await _local.SendAsync(new LocalNotification(
            Id: Math.Abs(payload.NotificationId),
            Title: payload.Title,
            Message: payload.Body,
            Data: payload.DeepLink));
        
        // 5. Announce for screen readers
        if (payload.Type is "order_status")
            _a11y.Announce(payload.Body);
        
        // 6. Update badge
        AppInfo.RequestedTheme = AppTheme.System;
        WeakReferenceMessenger.Default.Send(new UnreadCountChangedMessage(
            await _inApp.GetUnreadCountAsync()));
    }
    
    private async void OnPushTapped(object? sender, PushNotificationPayload payload)
    {
        if (!string.IsNullOrEmpty(payload.DeepLink))
            await _deepLink.HandleDeepLinkAsync(payload.DeepLink);
    }
}

public record UnreadCountChangedMessage(int Count);
```

---

## Step 984: Complete Analytics Pipeline

```csharp
// ============================================
// Unified Analytics: Events → Buffer → Batch → Server
// ============================================

public class UnifiedAnalyticsService
{
    private readonly Queue<AnalyticsEvent> _buffer = new();
    private readonly Timer _flushTimer;
    private readonly IApiClient _api;
    private readonly PrivacyConsentService _privacy;
    private readonly AbTestService _abTest;
    
    public UnifiedAnalyticsService(IApiClient api, PrivacyConsentService privacy, AbTestService abTest)
    {
        _api = api;
        _privacy = privacy;
        _abTest = abTest;
        _flushTimer = new Timer(FlushAsync, null, TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
    }
    
    public void Track(string name, Dictionary<string, object>? properties = null)
    {
        if (!_privacy.CanCollectAnalytics) return;
        
        var evt = new AnalyticsEvent(
            name,
            DateTime.UtcNow,
            properties ?? new Dictionary<string, object>(),
            SessionId: GetSessionId(),
            Platform: DeviceInfo.Platform.ToString(),
            AppVersion: AppInfo.VersionString);
        
        // Add A/B experiment context
        evt.Properties["home_variant"] = _abTest.GetHomeLayoutVariant();
        
        lock (_buffer)
            _buffer.Enqueue(evt);
    }
    
    private async void FlushAsync(object? _)
    {
        List<AnalyticsEvent> batch;
        lock (_buffer)
        {
            if (_buffer.Count == 0) return;
            batch = _buffer.TakeWhile(_ => true).ToList();
            _buffer.Clear();
        }
        
        try
        {
            await _api.PostAsync<object>("/analytics/batch", new { events = batch });
        }
        catch
        {
            // Re-queue on failure
            lock (_buffer)
                foreach (var evt in batch.Take(100)) // cap re-queue
                    _buffer.Enqueue(evt);
        }
    }
    
    private string GetSessionId()
        => Preferences.Get("analytics_session_id", Guid.NewGuid().ToString("N"));
}
```

---

## Step 985: Complete Security Stack

```csharp
// ============================================
// Production Security: All Layers Active
// ============================================

public class SecurityOrchestrator
{
    public async Task<SecurityStatus> CheckAndEnforceAsync(IServiceProvider services)
    {
        var issues = new List<string>();
        
        // 1. Device integrity
        var deviceCheck = services.GetRequiredService<DeviceIntegrityService>();
        var report = await deviceCheck.GetReportAsync();
        if (!report.IsIntact)
        {
            issues.Add("Device integrity compromised");
            if (report.IsRooted || report.IsEmulator)
                return SecurityStatus.BlockApp("เครื่องนี้ไม่สามารถใช้งานแอปได้");
        }
        
        // 2. Certificate pinning active (via PinnedHttpClientHandler in DI)
        // 3. App lock after 5 minutes idle (via AppLockService)
        var appLock = services.GetRequiredService<AppLockService>();
        if (appLock.IsLocked)
        {
            var unlocked = await appLock.UnlockAsync();
            if (!unlocked)
                return SecurityStatus.RequireAuth;
        }
        
        // 4. Jailbreak warning (soft-block)
        if (report.IsJailbroken)
            issues.Add("Device is jailbroken — security reduced");
        
        // 5. Version check (force update if security patch)
        var versionMgr = services.GetRequiredService<AppVersionManager>();
        var versionResult = await versionMgr.CheckForUpdateAsync();
        if (versionResult.Status == VersionStatus.ForceUpdate)
            return SecurityStatus.RequireUpdate;
        
        return issues.Any()
            ? SecurityStatus.Warning(issues)
            : SecurityStatus.Ok;
    }
}

public record SecurityStatus(SecurityStatusCode Code, string? Message, List<string>? Warnings)
{
    public static SecurityStatus Ok => new(SecurityStatusCode.Ok, null, null);
    public static SecurityStatus BlockApp(string msg) => new(SecurityStatusCode.Blocked, msg, null);
    public static SecurityStatus RequireAuth => new(SecurityStatusCode.RequireAuth, null, null);
    public static SecurityStatus RequireUpdate => new(SecurityStatusCode.RequireUpdate, null, null);
    public static SecurityStatus Warning(List<string> w) => new(SecurityStatusCode.Warning, null, w);
}

public enum SecurityStatusCode { Ok, Warning, RequireAuth, RequireUpdate, Blocked }
```

---

## Step 986: Complete Testing Strategy

```csharp
// ============================================
// Test Pyramid: 70% Unit / 20% Integration / 10% UI
// ============================================

// Unit: PlaceOrderUseCaseTests
[TestFixture]
public class PlaceOrderUseCaseTests
{
    private IOrderRepository _orderRepo = null!;
    private IMenuRepository _menuRepo = null!;
    private IPaymentGateway _payment = null!;
    private IEventPublisher _events = null!;
    private PlaceOrderUseCase _sut = null!;
    
    [SetUp]
    public void SetUp()
    {
        _orderRepo = Substitute.For<IOrderRepository>();
        _menuRepo = Substitute.For<IMenuRepository>();
        _payment = Substitute.For<IPaymentGateway>();
        _events = Substitute.For<IEventPublisher>();
        _sut = new PlaceOrderUseCase(_orderRepo, _menuRepo, _payment, _events);
    }
    
    [Test]
    public async Task PlaceOrder_Success_SavesOrderAndPublishesEvent()
    {
        var menuItem = new MenuItemEntity
        {
            Id = Guid.NewGuid(), Name = "ข้าวผัดกระเพรา", Price = 80, IsAvailable = true
        };
        
        _menuRepo.GetByIdsAsync(Arg.Any<List<Guid>>())
            .Returns(new List<MenuItemEntity> { menuItem });
        
        _payment.ChargeAsync(Arg.Any<decimal>(), Arg.Any<string>())
            .Returns(new PaymentResult(true, null, "ch_test_123"));
        
        var cmd = new PlaceOrderCommand(
            Guid.NewGuid(), Guid.NewGuid(),
            new List<Guid> { menuItem.Id },
            new Dictionary<Guid, int> { [menuItem.Id] = 2 },
            new DeliveryAddress("123 สีลม", "บางรัก", "กรุงเทพฯ", "10500"),
            "tok_test_123");
        
        var result = await _sut.ExecuteAsync(cmd);
        
        Assert.That(result.IsSuccess, Is.True);
        Assert.That(result.OrderId, Is.Not.Null);
        await _orderRepo.Received(1).SaveAsync(Arg.Any<Order>());
        await _events.Received().PublishAsync(Arg.Is<IDomainEvent>(e => e is OrderCreatedEvent));
    }
    
    [Test]
    public async Task PlaceOrder_UnavailableItem_ReturnsFailure()
    {
        var menuItem = new MenuItemEntity { Id = Guid.NewGuid(), Name = "พิเศษ", Price = 100, IsAvailable = false };
        _menuRepo.GetByIdsAsync(Arg.Any<List<Guid>>()).Returns(new List<MenuItemEntity> { menuItem });
        
        var cmd = new PlaceOrderCommand(
            Guid.NewGuid(), Guid.NewGuid(),
            new List<Guid> { menuItem.Id }, new Dictionary<Guid, int>(),
            new DeliveryAddress("1", "2", "3", "4"), "tok");
        
        var result = await _sut.ExecuteAsync(cmd);
        
        Assert.That(result.IsSuccess, Is.False);
        Assert.That(result.Error, Does.Contain("unavailable"));
        await _payment.DidNotReceive().ChargeAsync(Arg.Any<decimal>(), Arg.Any<string>());
    }
}
```

---

## Step 987: Production Monitoring Dashboard

```csharp
// ============================================
// Health Endpoint for Ops Monitoring
// ============================================

// Backend-side health endpoint (for reference)
// GET /health → checks DB, cache, payment gateway

// Mobile-side: report health on startup
public class AppHealthReporter
{
    private readonly IApiClient _api;
    
    public AppHealthReporter(IApiClient api) => _api = api;
    
    public async Task ReportAsync()
    {
        var report = new AppHealthReport(
            AppVersion: AppInfo.VersionString,
            BuildNumber: AppInfo.BuildString,
            Platform: DeviceInfo.Platform.ToString(),
            OsVersion: DeviceInfo.VersionString,
            DeviceModel: DeviceInfo.Model,
            IsEmulator: DeviceInfo.DeviceType == DeviceType.Virtual,
            Locale: System.Globalization.CultureInfo.CurrentCulture.Name,
            Timezone: TimeZoneInfo.Local.Id,
            MemoryMB: (int)(GC.GetTotalMemory(false) / 1_000_000),
            ReportedAt: DateTime.UtcNow
        );
        
        await _api.PostAsync<object>("/app/health", report);
    }
}

public record AppHealthReport(
    string AppVersion, string BuildNumber, string Platform, string OsVersion,
    string DeviceModel, bool IsEmulator, string Locale, string Timezone,
    int MemoryMB, DateTime ReportedAt);
```

---

## Step 988: Complete CI/CD Pipeline Summary

```yaml
# ============================================
# Complete GitHub Actions Workflow
# ============================================

# .github/workflows/production.yml
name: Production Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - name: Test with coverage
        run: |
          dotnet test --collect:"XPlat Code Coverage" \
            --results-directory ./coverage
          dotnet tool install -g dotnet-reportgenerator-globaltool
          reportgenerator -reports:./coverage/**/coverage.cobertura.xml \
            -targetdir:./coverage/report -reporttypes:Html
      - name: Enforce 80% coverage
        run: |
          dotnet tool install -g dotnet-coverage
          dotnet-coverage merge ./coverage/**/*.xml -f cobertura -o merged.xml
          # Fail if < 80%
          python3 -c "
          import xml.etree.ElementTree as ET
          tree = ET.parse('merged.xml')
          rate = float(tree.getroot().attrib.get('line-rate', 0))
          print(f'Coverage: {rate*100:.1f}%')
          assert rate >= 0.80, f'Coverage {rate*100:.1f}% below 80% threshold'
          "
  
  android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin' }
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      - name: Install MAUI workload
        run: dotnet workload install maui-android
      - name: Build AAB
        run: |
          dotnet publish FoodDelivery.App/FoodDelivery.App.csproj \
            -f net9.0-android \
            -c Release \
            /p:AndroidSigningKeyStore=fooddelivery.keystore \
            /p:AndroidSigningKeyAlias=${{ secrets.KEYSTORE_ALIAS }} \
            /p:AndroidSigningStorePass=${{ secrets.KEYSTORE_PASSWORD }} \
            /p:AndroidSigningKeyPass=${{ secrets.KEY_PASSWORD }}
      - name: Upload to Firebase
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_APP_ID_ANDROID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_CREDENTIALS }}
          groups: internal-testers
          releaseNotesFile: RELEASE_NOTES.md
          file: FoodDelivery.App/bin/Release/net9.0-android/fooddelivery.aab
  
  ios:
    needs: test
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      - name: Install MAUI workload
        run: dotnet workload install maui-ios
      - name: Import cert & profile
        uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.P12_BASE64 }}
          p12-password: ${{ secrets.P12_PASSWORD }}
      - name: Build IPA
        run: |
          dotnet publish FoodDelivery.App/FoodDelivery.App.csproj \
            -f net9.0-ios -c Release \
            /p:ArchiveOnBuild=true \
            /p:RuntimeIdentifier=ios-arm64 \
            /p:CodesignKey="iPhone Distribution: FoodDelivery Co. Ltd"
      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: FoodDelivery.App/bin/Release/net9.0-ios/fooddelivery.ipa
          issuer-id: ${{ secrets.APPLE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPLE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPLE_API_PRIVATE_KEY }}
```

---

## Step 989: Load Testing & Performance Benchmarks

```csharp
// ============================================
// BenchmarkDotNet for Critical Paths
// ============================================

// NuGet: BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class OrderBenchmarks
{
    private Order _order = null!;
    private List<ConventionalCommit> _commits = null!;
    
    [GlobalSetup]
    public void Setup()
    {
        var items = Enumerable.Range(1, 5).Select(i =>
            new OrderItem($"Item {i}", new Money(i * 50), i)).ToList();
        
        _order = Order.Create(
            new UserId(Guid.NewGuid()),
            new RestaurantId(Guid.NewGuid()),
            new DeliveryAddress("123", "Bang Rak", "Bangkok", "10500"),
            items);
        
        _commits = Enumerable.Range(1, 100).Select(i =>
            new ConventionalCommit("feat", "order", $"add feature {i}", false)).ToList();
    }
    
    [Benchmark]
    public decimal CalculateTotal()
        => _order.Total.Amount;
    
    [Benchmark]
    public string GenerateReleaseNotes()
        => new ReleaseNotesBuilder().ToMarkdown(
            new ReleaseNotesBuilder().Build("2.5.0", _commits));
    
    [Benchmark]
    public string FormatThaiCurrency()
        => ThaiCurrencyFormatter.ToThaiWords(12345.50m);
    
    [Benchmark]
    public double GetContrastRatio()
        => ContrastChecker.GetContrastRatio(Colors.White, Color.FromArgb("#E63946"));
}

/*
Benchmark results target:
| Method                | Mean      | Allocated |
|----------------------|-----------|-----------|
| CalculateTotal        | < 100 ns  | < 64 B    |
| GenerateReleaseNotes  | < 1 ms    | < 10 KB   |
| FormatThaiCurrency    | < 500 µs  | < 1 KB    |
| GetContrastRatio      | < 100 ns  | 0 B       |
*/
```

---

## Step 990: Integration Test — Full Order Flow

```csharp
// ============================================
// Integration Test: Complete Order Workflow
// ============================================

[TestFixture]
[Category("Integration")]
public class FullOrderFlowTests
{
    private IServiceProvider _services = null!;
    
    [SetUp]
    public async Task SetUp()
    {
        var services = new ServiceCollection();
        
        // In-memory SQLite
        var db = new SQLiteAsyncConnection(":memory:");
        await new MigrationRunner(db, new IMigration[]
        {
            new Migration001CreateOrders(),
            new Migration002AddDeliveryAddress()
        }).MigrateAsync();
        
        services.AddSingleton(db);
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IMenuRepository, FakeMenuRepository>();
        services.AddScoped<IPaymentGateway, FakePaymentGateway>();
        services.AddScoped<IEventPublisher, InMemoryEventPublisher>();
        services.AddScoped<PlaceOrderUseCase>();
        
        _services = services.BuildServiceProvider();
    }
    
    [Test]
    public async Task FullFlow_PlaceAndConfirmOrder_PersistsCorrectly()
    {
        var useCase = _services.GetRequiredService<PlaceOrderUseCase>();
        var menuRepo = _services.GetRequiredService<IMenuRepository>() as FakeMenuRepository;
        var menuItem = menuRepo!.AddItem("ข้าวผัดกระเพราไก่", 80);
        
        // Place order
        var result = await useCase.ExecuteAsync(new PlaceOrderCommand(
            Guid.NewGuid(), Guid.NewGuid(),
            new List<Guid> { menuItem.Id },
            new Dictionary<Guid, int> { [menuItem.Id] = 2 },
            new DeliveryAddress("1 สีลม", "บางรัก", "กรุงเทพฯ", "10500"),
            "tok_test"));
        
        Assert.That(result.IsSuccess, Is.True, result.Error);
        
        // Verify persisted
        var orderRepo = _services.GetRequiredService<IOrderRepository>();
        var saved = await orderRepo.GetByIdAsync(result.OrderId!);
        
        Assert.That(saved, Is.Not.Null);
        Assert.That(saved!.Status, Is.EqualTo(OrderStatus.Confirmed));
        Assert.That(saved.Total.Amount, Is.EqualTo(160m)); // 80 × 2
        
        // Verify event published
        var publisher = _services.GetRequiredService<IEventPublisher>() as InMemoryEventPublisher;
        Assert.That(publisher!.Events, Has.Some.TypeOf<OrderCreatedEvent>());
        Assert.That(publisher.Events, Has.Some.TypeOf<OrderConfirmedEvent>());
    }
}

// Fakes
public class FakeMenuRepository : IMenuRepository
{
    private readonly List<MenuItemEntity> _items = new();
    
    public MenuItemEntity AddItem(string name, decimal price)
    {
        var item = new MenuItemEntity
        {
            Id = Guid.NewGuid(), Name = name, Price = price, IsAvailable = true
        };
        _items.Add(item);
        return item;
    }
    
    public Task<List<MenuItemEntity>> GetByIdsAsync(List<Guid> ids)
        => Task.FromResult(_items.Where(i => ids.Contains(i.Id)).ToList());
}

public class FakePaymentGateway : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(decimal amount, string token)
        => Task.FromResult(new PaymentResult(true, null, "ch_fake_" + Guid.NewGuid().ToString("N")[..8]));
}

public class InMemoryEventPublisher : IEventPublisher
{
    public List<IDomainEvent> Events { get; } = new();
    public Task PublishAsync(IDomainEvent evt) { Events.Add(evt); return Task.CompletedTask; }
}
```

---

## สรุป Part 99

ใน Part 99 เราได้เรียนรู้:

1. **Complete Bootstrap** - MauiProgram.cs registering all layers (infra, domain, HTTP, VMs)
2. **End-to-End Order Flow** - CheckoutViewModel → PlaceOrderUseCase → Payment → Navigate
3. **Notification Orchestrator** - FCM → quiet hours → local notify → screen reader
4. **Unified Analytics** - Buffer → consent check → A/B context → batch flush
5. **Security Stack** - Device integrity → app lock → version check → orchestration
6. **Complete Test Strategy** - Unit test with NSubstitute, failure case, no payment call
7. **Health Reporting** - App version, device model, locale, memory on startup
8. **Full CI/CD YAML** - Test+coverage, Android AAB, iOS IPA, Firebase+TestFlight
9. **Benchmarks** - BenchmarkDotNet for critical paths, memory allocator checks
10. **Integration Test** - Full order flow: place → persist → event published

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 99 | Steps 981-990*
