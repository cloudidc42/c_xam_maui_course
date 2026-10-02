# Part 76: Multi-Tenant Architecture
## Steps 751-760: Tenant Isolation, White-Label Apps, Data Partitioning

---

## Step 751: Tenant Context

```csharp
// ============================================
// Multi-Tenant Context Management
// ============================================

public interface ITenantContext
{
    string TenantId { get; }
    string TenantName { get; }
    TenantConfig Config { get; }
    bool IsInitialized { get; }
}

public record TenantConfig(
    string PrimaryColor,
    string SecondaryColor,
    string LogoUrl,
    string AppName,
    string SupportPhone,
    string[] EnabledFeatures,
    string CurrencyCode,
    string CountryCode);

public class TenantContext : ITenantContext
{
    private TenantConfig? _config;
    
    public string TenantId { get; private set; } = "";
    public string TenantName { get; private set; } = "";
    public TenantConfig Config => _config ?? DefaultConfig;
    public bool IsInitialized => _config != null;
    
    private static readonly TenantConfig DefaultConfig = new(
        "#007AFF", "#34C759", "", "MyApp", "",
        Array.Empty<string>(), "THB", "TH");
    
    public void Initialize(string tenantId, string tenantName, TenantConfig config)
    {
        TenantId = tenantId;
        TenantName = tenantName;
        _config = config;
    }
}

// Tenant-aware base repository
public abstract class TenantAwareRepository<T> where T : ITenantEntity
{
    private readonly SQLiteAsyncConnection _db;
    protected readonly ITenantContext _tenant;
    
    protected TenantAwareRepository(SQLiteAsyncConnection db, ITenantContext tenant)
    {
        _db = db;
        _tenant = tenant;
    }
    
    protected Task<List<T>> GetAllAsync()
        => _db.Table<T>().Where(x => x.TenantId == _tenant.TenantId).ToListAsync();
    
    protected async Task SaveAsync(T entity)
    {
        entity.TenantId = _tenant.TenantId;
        if (entity.Id == 0) await _db.InsertAsync(entity);
        else await _db.UpdateAsync(entity);
    }
    
    protected Task<List<T>> QueryAsync(Expression<Func<T, bool>> predicate)
        => _db.Table<T>()
              .Where(x => x.TenantId == _tenant.TenantId)
              .Where(predicate)
              .ToListAsync();
}

public interface ITenantEntity
{
    int Id { get; set; }
    string TenantId { get; set; }
}
```

---

## Step 752: White-Label Theme Engine

```csharp
// ============================================
// Runtime Theme Switching for White-Label
// ============================================

public class WhiteLabelThemeService
{
    private readonly ITenantContext _tenant;
    
    public WhiteLabelThemeService(ITenantContext tenant)
    {
        _tenant = tenant;
    }
    
    public void ApplyTheme()
    {
        if (!_tenant.IsInitialized) return;
        
        var config = _tenant.Config;
        
        if (Application.Current?.Resources == null) return;
        
        Application.Current.Resources["PrimaryColor"] =
            Color.FromArgb(config.PrimaryColor);
        Application.Current.Resources["SecondaryColor"] =
            Color.FromArgb(config.SecondaryColor);
        Application.Current.Resources["AppName"] =
            config.AppName;
    }
    
    public async Task LoadTenantLogoAsync()
    {
        var logoUrl = _tenant.Config.LogoUrl;
        if (string.IsNullOrEmpty(logoUrl)) return;
        
        var path = Path.Combine(FileSystem.CacheDirectory, "logo.png");
        
        if (!File.Exists(path))
        {
            using var http = new HttpClient();
            var bytes = await http.GetByteArrayAsync(logoUrl);
            await File.WriteAllBytesAsync(path, bytes);
        }
        
        WeakReferenceMessenger.Default.Send(new TenantLogoLoadedMessage(path));
    }
}

public record TenantLogoLoadedMessage(string LocalPath);
```

---

## Step 753: Tenant-Scoped Feature Flags

```csharp
// ============================================
// Tenant Feature Flags
// ============================================

public class TenantFeatureFlagService
{
    private readonly ITenantContext _tenant;
    private readonly IRemoteFeatureFlagService _remote;
    
    public TenantFeatureFlagService(ITenantContext tenant, IRemoteFeatureFlagService remote)
    {
        _tenant = tenant;
        _remote = remote;
    }
    
    public async Task<bool> IsEnabledAsync(string feature)
    {
        // Check tenant-level override first
        if (_tenant.Config.EnabledFeatures.Contains(feature)) return true;
        
        // Then check remote flag with tenant scope
        return await _remote.IsEnabledAsync($"{_tenant.TenantId}:{feature}");
    }
    
    public bool IsEnabledLocally(string feature)
        => _tenant.Config.EnabledFeatures.Contains(feature);
}
```

---

## Step 754: API Gateway with Tenant Routing

```csharp
// ============================================
// Tenant-Aware HTTP Client
// ============================================

public class TenantApiClient
{
    private readonly HttpClient _client;
    private readonly ITenantContext _tenant;
    private readonly ITokenService _tokens;
    
    public TenantApiClient(HttpClient client, ITenantContext tenant, ITokenService tokens)
    {
        _client = client;
        _tenant = tenant;
        _tokens = tokens;
    }
    
    public async Task<T?> GetAsync<T>(string path, CancellationToken ct = default)
    {
        var request = await BuildRequestAsync(HttpMethod.Get, path);
        var response = await _client.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
    
    public async Task<TResponse?> PostAsync<TBody, TResponse>(
        string path, TBody body, CancellationToken ct = default)
    {
        var request = await BuildRequestAsync(HttpMethod.Post, path);
        request.Content = JsonContent.Create(body);
        var response = await _client.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<TResponse>(cancellationToken: ct);
    }
    
    private async Task<HttpRequestMessage> BuildRequestAsync(HttpMethod method, string path)
    {
        var request = new HttpRequestMessage(method, path);
        
        // Tenant identification headers
        request.Headers.Add("X-Tenant-Id", _tenant.TenantId);
        request.Headers.Add("X-Tenant-Name", _tenant.TenantName);
        
        // Auth token
        var token = await _tokens.GetAccessTokenAsync();
        if (token != null)
            request.Headers.Authorization = new("Bearer", token);
        
        return request;
    }
}
```

---

## Step 755: Tenant Onboarding Flow

```csharp
// ============================================
// Tenant Discovery and Initialization
// ============================================

public class TenantDiscoveryService
{
    private readonly HttpClient _http;
    private readonly TenantContext _context;
    private readonly WhiteLabelThemeService _theme;
    
    public TenantDiscoveryService(HttpClient http, TenantContext context,
        WhiteLabelThemeService theme)
    {
        _http = http;
        _context = context;
        _theme = theme;
    }
    
    // Discover tenant from app invite link or QR code
    public async Task<bool> DiscoverFromCodeAsync(string tenantCode)
    {
        try
        {
            var response = await _http.GetFromJsonAsync<TenantDiscoveryResponse>(
                $"/api/tenants/{tenantCode}");
            
            if (response == null) return false;
            
            var config = new TenantConfig(
                response.PrimaryColor,
                response.SecondaryColor,
                response.LogoUrl,
                response.AppName,
                response.SupportPhone,
                response.Features,
                response.CurrencyCode,
                response.CountryCode);
            
            _context.Initialize(response.TenantId, response.TenantName, config);
            
            // Persist for next launch
            await PersistTenantAsync(response);
            
            // Apply branding
            _theme.ApplyTheme();
            await _theme.LoadTenantLogoAsync();
            
            return true;
        }
        catch { return false; }
    }
    
    public async Task<bool> RestoreFromStorageAsync()
    {
        var json = Preferences.Get("tenant_config", null as string);
        if (json == null) return false;
        
        try
        {
            var stored = System.Text.Json.JsonSerializer.Deserialize<TenantDiscoveryResponse>(json);
            if (stored == null) return false;
            
            var config = new TenantConfig(
                stored.PrimaryColor, stored.SecondaryColor,
                stored.LogoUrl, stored.AppName, stored.SupportPhone,
                stored.Features, stored.CurrencyCode, stored.CountryCode);
            
            _context.Initialize(stored.TenantId, stored.TenantName, config);
            _theme.ApplyTheme();
            return true;
        }
        catch { return false; }
    }
    
    private Task PersistTenantAsync(TenantDiscoveryResponse response)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(response);
        Preferences.Set("tenant_config", json);
        return Task.CompletedTask;
    }
}

public class TenantDiscoveryResponse
{
    public string TenantId { get; set; } = "";
    public string TenantName { get; set; } = "";
    public string PrimaryColor { get; set; } = "#007AFF";
    public string SecondaryColor { get; set; } = "#34C759";
    public string LogoUrl { get; set; } = "";
    public string AppName { get; set; } = "MyApp";
    public string SupportPhone { get; set; } = "";
    public string[] Features { get; set; } = Array.Empty<string>();
    public string CurrencyCode { get; set; } = "THB";
    public string CountryCode { get; set; } = "TH";
}
```

---

## Step 756: Data Isolation Testing

```csharp
// ============================================
// Tenant Data Isolation Tests
// ============================================

[TestFixture]
public class TenantIsolationTests
{
    private SQLiteAsyncConnection _db = null!;
    
    [SetUp]
    public async Task SetUp()
    {
        _db = new SQLiteAsyncConnection(":memory:");
        await _db.CreateTableAsync<TenantOrderEntity>();
    }
    
    [Test]
    public async Task TenantA_CannotSee_TenantBData()
    {
        var tenantA = CreateContext("tenant-a");
        var tenantB = CreateContext("tenant-b");
        
        var repoA = new TenantOrderRepository(_db, tenantA);
        var repoB = new TenantOrderRepository(_db, tenantB);
        
        // Tenant A creates order
        await repoA.SaveAsync(new TenantOrderEntity
        {
            TenantId = "tenant-a",
            OrderNumber = "A-001",
            Total = 200
        });
        
        // Tenant B queries – should see nothing
        var ordersB = await repoB.GetAllAsync();
        Assert.That(ordersB, Is.Empty,
            "Tenant B should not see Tenant A orders");
    }
    
    [Test]
    public async Task TenantA_CanSee_OwnData()
    {
        var tenantA = CreateContext("tenant-a");
        var repo = new TenantOrderRepository(_db, tenantA);
        
        await repo.SaveAsync(new TenantOrderEntity
        {
            TenantId = "tenant-a",
            OrderNumber = "A-002",
            Total = 350
        });
        
        var orders = await repo.GetAllAsync();
        Assert.That(orders.Count, Is.EqualTo(1));
        Assert.That(orders[0].OrderNumber, Is.EqualTo("A-002"));
    }
    
    private static TenantContext CreateContext(string tenantId)
    {
        var ctx = new TenantContext();
        ctx.Initialize(tenantId, tenantId,
            new TenantConfig("#000", "#FFF", "", "Test", "",
                Array.Empty<string>(), "THB", "TH"));
        return ctx;
    }
}

[Table("tenant_orders")]
public class TenantOrderEntity : ITenantEntity
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string TenantId { get; set; } = "";
    public string OrderNumber { get; set; } = "";
    public decimal Total { get; set; }
}

public class TenantOrderRepository : TenantAwareRepository<TenantOrderEntity>
{
    public TenantOrderRepository(SQLiteAsyncConnection db, ITenantContext tenant)
        : base(db, tenant) { }
    
    public new Task<List<TenantOrderEntity>> GetAllAsync() => base.GetAllAsync();
    public new Task SaveAsync(TenantOrderEntity entity) => base.SaveAsync(entity);
}
```

---

## Step 757: Dynamic Branding XAML

```xml
<!-- Dynamic branded UI using tenant resources -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyApp.BrandedPage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             BackgroundColor="{AppThemeBinding 
                 Light={StaticResource BackgroundLight}, 
                 Dark={StaticResource BackgroundDark}}">
    
    <VerticalStackLayout Padding="16">
        
        <!-- Logo: uses tenant's loaded image -->
        <Image Source="{Binding TenantLogoSource}"
               HeightRequest="60" Aspect="AspectFit"
               HorizontalOptions="Center" />
        
        <!-- Primary color button from tenant config -->
        <Button Text="{Binding CheckoutText}"
                BackgroundColor="{DynamicResource PrimaryColor}"
                TextColor="White"
                CornerRadius="12" />
        
        <!-- App name label -->
        <Label Text="{DynamicResource AppName}"
               FontSize="24" FontAttributes="Bold"
               HorizontalOptions="Center" />
        
    </VerticalStackLayout>
</ContentPage>
```

```csharp
public partial class BrandedPageViewModel : ObservableObject
{
    private readonly ITenantContext _tenant;
    
    [ObservableProperty] private ImageSource? _tenantLogoSource;
    
    public string CheckoutText => _tenant.Config.AppName == "RestaurantApp"
        ? "สั่งอาหาร" : "ดำเนินการต่อ";
    
    public BrandedPageViewModel(ITenantContext tenant)
    {
        _tenant = tenant;
        
        WeakReferenceMessenger.Default.Register<TenantLogoLoadedMessage>(this,
            (_, msg) => TenantLogoSource = ImageSource.FromFile(msg.LocalPath));
    }
}
```

---

## Step 758: Multi-Tenant CI/CD

```yaml
# .github/workflows/multi-tenant-build.yml
name: Multi-Tenant Build

on:
  push:
    branches: [main]

jobs:
  build-tenants:
    strategy:
      matrix:
        tenant: [restaurant-a, restaurant-b, restaurant-c]
    
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Load tenant config
        run: |
          echo "Loading config for ${{ matrix.tenant }}"
          cp tenants/${{ matrix.tenant }}/config.json ./TenantConfig.json
          cp tenants/${{ matrix.tenant }}/colors.xaml ./Resources/TenantColors.xaml
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Build Android APK
        run: |
          dotnet publish -f net9.0-android \
            -c Release \
            -p:ApplicationId=com.mycompany.${{ matrix.tenant }} \
            -p:ApplicationTitle="${{ matrix.tenant }}" \
            --output ./artifacts/${{ matrix.tenant }}
      
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: apk-${{ matrix.tenant }}
          path: ./artifacts/${{ matrix.tenant }}/*.apk
```

---

## Step 759: Tenant Analytics Dashboard

```csharp
// ============================================
// Per-Tenant Analytics
// ============================================

public class TenantAnalyticsService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ITenantContext _tenant;
    
    public TenantAnalyticsService(SQLiteAsyncConnection db, ITenantContext tenant)
    {
        _db = db;
        _tenant = tenant;
    }
    
    public async Task<TenantDashboardData> GetDashboardAsync(DateTime since)
    {
        var orders = await _db.QueryAsync<OrderSummary>(
            @"SELECT COUNT(*) as OrderCount, SUM(Total) as Revenue
              FROM Orders 
              WHERE TenantId = ? AND CreatedAt >= ?",
            _tenant.TenantId, since);
        
        var topItems = await _db.QueryAsync<TopItemStat>(
            @"SELECT MenuItemName, COUNT(*) as OrderCount 
              FROM OrderItems oi
              JOIN Orders o ON oi.OrderId = o.Id
              WHERE o.TenantId = ? AND o.CreatedAt >= ?
              GROUP BY MenuItemName
              ORDER BY OrderCount DESC
              LIMIT 5",
            _tenant.TenantId, since);
        
        return new TenantDashboardData
        {
            TenantId = _tenant.TenantId,
            TenantName = _tenant.TenantName,
            TotalOrders = orders.FirstOrDefault()?.OrderCount ?? 0,
            TotalRevenue = orders.FirstOrDefault()?.Revenue ?? 0,
            TopItems = topItems,
            Period = since
        };
    }
}

public class TenantDashboardData
{
    public string TenantId { get; set; } = "";
    public string TenantName { get; set; } = "";
    public int TotalOrders { get; set; }
    public decimal TotalRevenue { get; set; }
    public List<TopItemStat> TopItems { get; set; } = new();
    public DateTime Period { get; set; }
}

public class TopItemStat
{
    public string MenuItemName { get; set; } = "";
    public int OrderCount { get; set; }
}
```

---

## Step 760: Tenant Migration

```csharp
// ============================================
// Cross-Tenant Data Migration
// ============================================

public class TenantDataMigrationService
{
    private readonly SQLiteAsyncConnection _db;
    
    public TenantDataMigrationService(SQLiteAsyncConnection db)
    {
        _db = db;
    }
    
    // Export all data for a tenant (GDPR right to portability)
    public async Task<string> ExportTenantDataAsync(string tenantId)
    {
        var orders = await _db.Table<TenantOrderEntity>()
            .Where(o => o.TenantId == tenantId).ToListAsync();
        
        var export = new TenantDataExport
        {
            TenantId = tenantId,
            ExportedAt = DateTime.UtcNow,
            Orders = orders
        };
        
        return System.Text.Json.JsonSerializer.Serialize(export);
    }
    
    // Transfer tenant to new app instance
    public async Task<int> TransferTenantAsync(string fromTenantId, string toTenantId)
    {
        var orders = await _db.Table<TenantOrderEntity>()
            .Where(o => o.TenantId == fromTenantId).ToListAsync();
        
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var order in orders)
            {
                order.Id = 0; // New ID
                order.TenantId = toTenantId;
                conn.Insert(order);
            }
        });
        
        return orders.Count;
    }
    
    // Delete all tenant data
    public async Task DeleteTenantDataAsync(string tenantId)
    {
        await _db.ExecuteAsync(
            "DELETE FROM tenant_orders WHERE TenantId = ?", tenantId);
        // Delete other tables...
    }
}

public class TenantDataExport
{
    public string TenantId { get; set; } = "";
    public DateTime ExportedAt { get; set; }
    public List<TenantOrderEntity> Orders { get; set; } = new();
}
```

---

## สรุป Part 76

ใน Part 76 เราได้เรียนรู้:

1. **Tenant Context** - ITenantContext, TenantConfig record, base repository
2. **White-Label Theme** - Runtime color/logo injection via App.Resources
3. **Tenant Feature Flags** - Feature array in config + remote flag scoping
4. **Tenant API Client** - X-Tenant-Id header injection, bearer token
5. **Tenant Discovery** - Code-based discovery, persisted tenant config
6. **Isolation Tests** - Verified cross-tenant data cannot leak
7. **Dynamic Branding XAML** - DynamicResource binding for tenant colors
8. **Multi-Tenant CI/CD** - Matrix build strategy, per-tenant APK
9. **Tenant Analytics** - Per-tenant order/revenue/top-items dashboard
10. **Tenant Migration** - Export, transfer, GDPR deletion

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 76 | Steps 751-760*
