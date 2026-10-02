# Part 34: Enterprise Patterns
## Steps 331-340: Professional Enterprise Development

---

## Step 331: Multi-Tenant Architecture

```csharp
// ============================================
// Multi-Tenant Support
// ============================================

public interface ITenantProvider
{
    string TenantId { get; }
    string TenantName { get; }
    TenantConfig Config { get; }
}

public class TenantProvider : ITenantProvider
{
    private TenantInfo? _tenant;
    
    public string TenantId => _tenant?.Id ?? "default";
    public string TenantName => _tenant?.Name ?? "Default";
    public TenantConfig Config => _tenant?.Config ?? TenantConfig.Default;
    
    public void SetTenant(TenantInfo tenant) => _tenant = tenant;
    public void ClearTenant() => _tenant = null;
}

public record TenantInfo(string Id, string Name, TenantConfig Config);

public class TenantConfig
{
    public string ApiBaseUrl { get; init; } = string.Empty;
    public string LogoUrl { get; init; } = string.Empty;
    public string PrimaryColor { get; init; } = "#2196F3";
    public bool EnableFeatureX { get; init; }
    public int MaxUsersPerGroup { get; init; } = 50;
    
    public static TenantConfig Default => new()
    {
        ApiBaseUrl = AppConfiguration.ApiBaseUrl,
        PrimaryColor = "#2196F3"
    };
}

// Tenant-aware HTTP client
public class TenantHttpHandler : DelegatingHandler
{
    private readonly ITenantProvider _tenant;
    
    public TenantHttpHandler(ITenantProvider tenant) => _tenant = tenant;
    
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        request.Headers.Add("X-Tenant-Id", _tenant.TenantId);
        return base.SendAsync(request, ct);
    }
}

// Tenant-aware repository
public class TenantProductRepository : IProductRepository
{
    private readonly SqliteProductRepository _inner;
    private readonly ITenantProvider _tenant;
    
    public TenantProductRepository(SqliteProductRepository inner, ITenantProvider tenant)
    {
        _inner = inner;
        _tenant = tenant;
    }
    
    public async Task<List<Product>> GetAllAsync()
    {
        // Filter by tenant
        return await _inner.GetByTenantAsync(_tenant.TenantId);
    }
    
    public async Task<Product> CreateAsync(Product product)
    {
        product.TenantId = _tenant.TenantId;
        return await _inner.CreateAsync(product);
    }
}
```

---

## Step 332: Audit Trail

```csharp
// ============================================
// Audit Logging
// ============================================

public enum AuditAction { Create, Read, Update, Delete, Login, Logout, Export }

public class AuditEntry
{
    [SQLite.PrimaryKey, SQLite.AutoIncrement] public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public string UserEmail { get; set; } = string.Empty;
    public AuditAction Action { get; set; }
    public string EntityType { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string? OldValues { get; set; }
    public string? NewValues { get; set; }
    public string? IpAddress { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

public class AuditService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ICurrentUserService _currentUser;
    
    public AuditService(SQLiteAsyncConnection db, ICurrentUserService currentUser)
    {
        _db = db;
        _currentUser = currentUser;
    }
    
    public async Task LogAsync<T>(
        AuditAction action,
        string entityId,
        T? oldValue = default,
        T? newValue = default)
    {
        var entry = new AuditEntry
        {
            UserId = _currentUser.UserId?.ToString() ?? "anonymous",
            UserEmail = _currentUser.Email ?? string.Empty,
            Action = action,
            EntityType = typeof(T).Name,
            EntityId = entityId,
            OldValues = oldValue != null ? System.Text.Json.JsonSerializer.Serialize(oldValue) : null,
            NewValues = newValue != null ? System.Text.Json.JsonSerializer.Serialize(newValue) : null,
            Timestamp = DateTime.UtcNow
        };
        
        await _db.InsertAsync(entry);
    }
    
    public async Task<List<AuditEntry>> GetRecentAsync(int count = 100)
        => await _db.Table<AuditEntry>()
            .OrderByDescending(e => e.Timestamp)
            .Take(count)
            .ToListAsync();
    
    public async Task<List<AuditEntry>> GetByEntityAsync(string entityType, string entityId)
        => await _db.Table<AuditEntry>()
            .Where(e => e.EntityType == entityType && e.EntityId == entityId)
            .OrderByDescending(e => e.Timestamp)
            .ToListAsync();
}

// Decorator
public class AuditedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly AuditService _audit;
    
    public AuditedProductRepository(IProductRepository inner, AuditService audit)
    {
        _inner = inner;
        _audit = audit;
    }
    
    public async Task<Product> CreateAsync(Product product)
    {
        var result = await _inner.CreateAsync(product);
        await _audit.LogAsync(AuditAction.Create, result.Id.ToString(), newValue: result);
        return result;
    }
    
    public async Task UpdateAsync(Product product)
    {
        var old = await _inner.GetByIdAsync(product.Id);
        await _inner.UpdateAsync(product);
        await _audit.LogAsync(AuditAction.Update, product.Id.ToString(), old, product);
    }
    
    public async Task DeleteAsync(int id)
    {
        var old = await _inner.GetByIdAsync(id);
        await _inner.DeleteAsync(id);
        await _audit.LogAsync(AuditAction.Delete, id.ToString(), old);
    }
}
```

---

## Step 333: Data Export

```csharp
// ============================================
// Export to CSV / Excel
// ============================================

public class DataExportService
{
    // Export to CSV
    public async Task<string> ExportToCsvAsync<T>(
        IEnumerable<T> data, string fileName)
    {
        var filePath = Path.Combine(FileSystem.CacheDirectory, fileName + ".csv");
        
        var properties = typeof(T).GetProperties()
            .Where(p => p.CanRead)
            .ToArray();
        
        using var writer = new StreamWriter(filePath, false, System.Text.Encoding.UTF8);
        
        // Write BOM for Excel compatibility
        await writer.WriteAsync('﻿');
        
        // Header row
        var header = string.Join(",", properties.Select(p => 
            $"\"{p.Name}\""));
        await writer.WriteLineAsync(header);
        
        // Data rows
        foreach (var item in data)
        {
            var values = properties.Select(p =>
            {
                var value = p.GetValue(item)?.ToString() ?? string.Empty;
                value = value.Replace("\"", "\"\""); // Escape quotes
                return $"\"{value}\"";
            });
            
            await writer.WriteLineAsync(string.Join(",", values));
        }
        
        return filePath;
    }
    
    // Export to JSON
    public async Task<string> ExportToJsonAsync<T>(
        IEnumerable<T> data, string fileName)
    {
        var filePath = Path.Combine(FileSystem.CacheDirectory, fileName + ".json");
        
        var json = System.Text.Json.JsonSerializer.Serialize(data, new System.Text.Json.JsonSerializerOptions
        {
            WriteIndented = true,
            PropertyNamingPolicy = System.Text.Json.JsonNamingPolicy.CamelCase
        });
        
        await File.WriteAllTextAsync(filePath, json);
        return filePath;
    }
    
    // Share exported file
    public async Task ShareExportAsync(string filePath, string title)
    {
        await Share.Default.RequestAsync(new ShareFileRequest
        {
            Title = title,
            File = new ShareFile(filePath)
        });
    }
    
    // Export products example
    public async Task ExportProductsAsync(IEnumerable<Product> products)
    {
        var data = products.Select(p => new
        {
            ID = p.Id,
            Name = p.Name,
            Price = p.Price.ToString("N2"),
            Category = p.Category,
            Stock = p.StockQuantity,
            CreatedAt = p.CreatedAt.ToString("dd/MM/yyyy")
        });
        
        var filePath = await ExportToCsvAsync(data, "products_export");
        await ShareExportAsync(filePath, "ส่งออกข้อมูลสินค้า");
    }
}
```

---

## Step 334: Microservices Client

```csharp
// ============================================
// Microservices API Client
// ============================================

public class ApiGatewayClient
{
    private readonly IHttpClientFactory _factory;
    
    public ApiGatewayClient(IHttpClientFactory factory) => _factory = factory;
    
    public HttpClient GetProductsClient() => _factory.CreateClient("products");
    public HttpClient GetOrdersClient() => _factory.CreateClient("orders");
    public HttpClient GetUserClient() => _factory.CreateClient("users");
    public HttpClient GetInventoryClient() => _factory.CreateClient("inventory");
}

// Registration
public static class HttpClientRegistration
{
    public static IServiceCollection AddMicroserviceClients(
        this IServiceCollection services, string gatewayUrl)
    {
        void ConfigureClient(HttpClient client, string service)
        {
            client.BaseAddress = new Uri($"{gatewayUrl}/{service}/");
            client.DefaultRequestHeaders.Add("Accept", "application/json");
            client.Timeout = TimeSpan.FromSeconds(30);
        }
        
        services.AddHttpClient("products", c => ConfigureClient(c, "products"))
            .AddHttpMessageHandler<AuthHeaderHandler>()
            .AddHttpMessageHandler<RetryHandler>();
        
        services.AddHttpClient("orders", c => ConfigureClient(c, "orders"))
            .AddHttpMessageHandler<AuthHeaderHandler>()
            .AddHttpMessageHandler<RetryHandler>();
        
        services.AddHttpClient("users", c => ConfigureClient(c, "users"))
            .AddHttpMessageHandler<AuthHeaderHandler>();
        
        services.AddSingleton<ApiGatewayClient>();
        return services;
    }
}

// Retry handler with exponential backoff
public class RetryHandler : DelegatingHandler
{
    private const int MaxRetries = 3;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        for (var attempt = 0; attempt <= MaxRetries; attempt++)
        {
            try
            {
                var response = await base.SendAsync(request, ct);
                
                if (response.IsSuccessStatusCode || 
                    response.StatusCode == System.Net.HttpStatusCode.NotFound ||
                    response.StatusCode == System.Net.HttpStatusCode.BadRequest)
                    return response;
                
                if (attempt == MaxRetries) return response;
            }
            catch when (attempt < MaxRetries) { }
            
            var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt));
            await Task.Delay(delay, ct);
        }
        
        throw new HttpRequestException("Max retries exceeded");
    }
}
```

---

## Step 335: Data Synchronization

```csharp
// ============================================
// Conflict Resolution for Sync
// ============================================

public enum ConflictResolution { LocalWins, ServerWins, Latest, Manual }

public class SyncConflict
{
    public string EntityType { get; init; } = string.Empty;
    public string EntityId { get; init; } = string.Empty;
    public object LocalVersion { get; init; } = null!;
    public object ServerVersion { get; init; } = null!;
    public DateTime LocalModifiedAt { get; init; }
    public DateTime ServerModifiedAt { get; init; }
}

public class SyncEngine<T> where T : class, ISyncable
{
    private readonly IRepository<T> _localRepo;
    private readonly IApiService<T> _api;
    private readonly ConflictResolution _conflictStrategy;
    
    public SyncEngine(
        IRepository<T> localRepo,
        IApiService<T> api,
        ConflictResolution conflictStrategy = ConflictResolution.Latest)
    {
        _localRepo = localRepo;
        _api = api;
        _conflictStrategy = conflictStrategy;
    }
    
    public event Action<SyncConflict>? ConflictDetected;
    
    public async Task<SyncResult> SyncAsync(CancellationToken ct = default)
    {
        var result = new SyncResult();
        
        // 1. Push local changes
        var localChanges = await _localRepo.GetModifiedSinceAsync(await GetLastSyncTimeAsync());
        
        foreach (var item in localChanges)
        {
            ct.ThrowIfCancellationRequested();
            
            try
            {
                var serverVersion = await _api.GetByIdAsync(item.Id);
                
                if (serverVersion != null && serverVersion.ModifiedAt > item.ModifiedAt)
                {
                    // Conflict!
                    var resolution = await ResolveConflictAsync(item, serverVersion);
                    if (resolution == item)
                    {
                        await _api.UpdateAsync(item.Id, item);
                        result.PushedCount++;
                    }
                    else
                    {
                        await _localRepo.UpsertAsync(resolution);
                        result.PulledCount++;
                    }
                }
                else
                {
                    if (item.IsDeleted) await _api.DeleteAsync(item.Id);
                    else await _api.UpdateAsync(item.Id, item);
                    result.PushedCount++;
                }
            }
            catch (Exception ex)
            {
                result.Errors.Add($"{item.Id}: {ex.Message}");
            }
        }
        
        // 2. Pull server changes
        var lastSync = await GetLastSyncTimeAsync();
        var serverChanges = await _api.GetUpdatedSinceAsync(lastSync, ct);
        
        foreach (var item in serverChanges)
        {
            await _localRepo.UpsertAsync(item);
            result.PulledCount++;
        }
        
        await SetLastSyncTimeAsync(DateTime.UtcNow);
        return result;
    }
    
    private async Task<T> ResolveConflictAsync(T local, T server)
    {
        return _conflictStrategy switch
        {
            ConflictResolution.LocalWins => local,
            ConflictResolution.ServerWins => server,
            ConflictResolution.Latest => local.ModifiedAt > server.ModifiedAt ? local : server,
            ConflictResolution.Manual => await AskUserAsync(local, server),
            _ => server
        };
    }
    
    private async Task<T> AskUserAsync(T local, T server)
    {
        var conflict = new SyncConflict
        {
            LocalVersion = local,
            ServerVersion = server,
            LocalModifiedAt = local.ModifiedAt,
            ServerModifiedAt = server.ModifiedAt
        };
        
        ConflictDetected?.Invoke(conflict);
        
        // Show dialog
        var choice = await Shell.Current.DisplayActionSheet(
            "พบข้อขัดแย้ง", "ยกเลิก", null,
            $"ใช้ข้อมูลของฉัน ({local.ModifiedAt:HH:mm})",
            $"ใช้ข้อมูลเซิร์ฟเวอร์ ({server.ModifiedAt:HH:mm})");
        
        return choice?.Contains("ของฉัน") == true ? local : server;
    }
    
    private Task<DateTime> GetLastSyncTimeAsync()
    {
        var str = Preferences.Default.Get($"last_sync_{typeof(T).Name}", DateTime.MinValue.ToString("O"));
        return Task.FromResult(DateTime.Parse(str));
    }
    
    private Task SetLastSyncTimeAsync(DateTime time)
    {
        Preferences.Default.Set($"last_sync_{typeof(T).Name}", time.ToString("O"));
        return Task.CompletedTask;
    }
}

public interface ISyncable
{
    string Id { get; }
    DateTime ModifiedAt { get; }
    bool IsDeleted { get; }
}

public record SyncResult
{
    public int PushedCount { get; set; }
    public int PulledCount { get; set; }
    public List<string> Errors { get; } = new();
    public bool HasErrors => Errors.Count > 0;
}
```

---

## Step 336: Notification System

```csharp
// ============================================
// In-App Notification Center
// ============================================

public class NotificationCenter
{
    private readonly SQLiteAsyncConnection _db;
    private readonly IEventBus _bus;
    
    [ObservableProperty] private int _unreadCount;
    
    public NotificationCenter(SQLiteAsyncConnection db, IEventBus bus)
    {
        _db = db;
        _bus = bus;
    }
    
    public async Task AddAsync(AppNotification notification)
    {
        await _db.InsertAsync(notification);
        
        var count = await _db.Table<AppNotification>()
            .Where(n => !n.IsRead)
            .CountAsync();
        
        _bus.Publish(new NotificationAddedEvent(notification, count));
    }
    
    public async Task<List<AppNotification>> GetAllAsync(int page = 1, int size = 20)
    {
        return await _db.Table<AppNotification>()
            .OrderByDescending(n => n.CreatedAt)
            .Skip((page - 1) * size)
            .Take(size)
            .ToListAsync();
    }
    
    public async Task MarkReadAsync(string notificationId)
    {
        var notif = await _db.Table<AppNotification>()
            .FirstOrDefaultAsync(n => n.Id == notificationId);
        
        if (notif != null && !notif.IsRead)
        {
            notif.IsRead = true;
            await _db.UpdateAsync(notif);
            
            var count = await _db.Table<AppNotification>()
                .Where(n => !n.IsRead).CountAsync();
            
            _bus.Publish(new UnreadCountChangedEvent(count));
        }
    }
    
    public async Task MarkAllReadAsync()
    {
        await _db.ExecuteAsync(
            "UPDATE app_notifications SET is_read = 1 WHERE is_read = 0");
        _bus.Publish(new UnreadCountChangedEvent(0));
    }
    
    public async Task DeleteOldAsync(int daysToKeep = 30)
    {
        var cutoff = DateTime.UtcNow.AddDays(-daysToKeep).ToString("O");
        await _db.ExecuteAsync(
            "DELETE FROM app_notifications WHERE created_at < ? AND is_read = 1",
            cutoff);
    }
}

public record NotificationAddedEvent(AppNotification Notification, int TotalUnread);
public record UnreadCountChangedEvent(int Count);
```

---

## Step 337: Plugin Architecture

```csharp
// ============================================
// Plugin/Extension System
// ============================================

public interface IAppPlugin
{
    string Id { get; }
    string Name { get; }
    string Version { get; }
    void Initialize(IServiceProvider services);
    void RegisterRoutes(IShellRouter router);
}

public interface IShellRouter
{
    void RegisterRoute(string route, Type pageType);
    void AddTabBarItem(string route, string title, string icon);
}

public class PluginManager
{
    private readonly List<IAppPlugin> _plugins = new();
    private readonly IServiceProvider _services;
    
    public PluginManager(IServiceProvider services) => _services = services;
    
    public void Register(IAppPlugin plugin)
    {
        _plugins.Add(plugin);
        plugin.Initialize(_services);
    }
    
    public void ConfigureShell(IShellRouter router)
    {
        foreach (var plugin in _plugins)
            plugin.RegisterRoutes(router);
    }
    
    public IAppPlugin? Get(string id)
        => _plugins.FirstOrDefault(p => p.Id == id);
    
    public IReadOnlyList<IAppPlugin> GetAll() => _plugins.AsReadOnly();
}

// Example plugin
public class LoyaltyPlugin : IAppPlugin
{
    public string Id => "loyalty";
    public string Name => "ระบบสะสมแต้ม";
    public string Version => "1.0.0";
    
    public void Initialize(IServiceProvider services)
    {
        // Register loyalty-specific services
    }
    
    public void RegisterRoutes(IShellRouter router)
    {
        router.RegisterRoute("loyalty/points", typeof(LoyaltyPointsPage));
        router.RegisterRoute("loyalty/rewards", typeof(LoyaltyRewardsPage));
        router.AddTabBarItem("loyalty/points", "แต้มสะสม", "star");
    }
}

// Placeholder pages for plugin
public class LoyaltyPointsPage : ContentPage { }
public class LoyaltyRewardsPage : ContentPage { }
```

---

## Step 338: Logging

```csharp
// ============================================
// Structured Logging
// ============================================

// Install: Microsoft.Extensions.Logging
// For production: Serilog.Extensions.Hosting, Serilog.Sinks.File

public static class LoggingConfiguration
{
    public static MauiAppBuilder ConfigureLogging(this MauiAppBuilder builder)
    {
        builder.Logging
            .AddDebug()
            .SetMinimumLevel(LogLevel.Information)
            .AddFilter("Microsoft", LogLevel.Warning)
            .AddFilter("System", LogLevel.Warning)
            .AddFilter("MyApp", LogLevel.Debug);
        
        return builder;
    }
}

// Structured logging with scopes
public class ProductService2
{
    private readonly ILogger<ProductService2> _logger;
    private readonly IProductRepository _repo;
    
    public ProductService2(ILogger<ProductService2> logger, IProductRepository repo)
    {
        _logger = logger;
        _repo = repo;
    }
    
    public async Task<Product?> GetProductAsync(int id)
    {
        using var scope = _logger.BeginScope("GetProduct {ProductId}", id);
        
        _logger.LogDebug("Fetching product from database");
        
        var product = await _repo.GetByIdAsync(id);
        
        if (product == null)
            _logger.LogWarning("Product {ProductId} not found", id);
        else
            _logger.LogDebug("Found product {ProductName} (${Price})", product.Name, product.Price);
        
        return product;
    }
    
    public async Task<Product> CreateProductAsync(Product product)
    {
        _logger.LogInformation(
            "Creating product {Name} at price {Price}", 
            product.Name, product.Price);
        
        try
        {
            var result = await _repo.CreateAsync(product);
            _logger.LogInformation("Product {Id} created successfully", result.Id);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create product {Name}", product.Name);
            throw;
        }
    }
}

// Log to file
public class FileLoggerProvider : ILoggerProvider
{
    private readonly string _logPath;
    
    public FileLoggerProvider()
    {
        _logPath = Path.Combine(FileSystem.AppDataDirectory, "app.log");
    }
    
    public ILogger CreateLogger(string categoryName) 
        => new FileLogger(categoryName, _logPath);
    
    public void Dispose() { }
}

public class FileLogger : ILogger
{
    private readonly string _category;
    private readonly string _logPath;
    private static readonly object _lock = new();
    
    public FileLogger(string category, string logPath)
    {
        _category = category;
        _logPath = logPath;
    }
    
    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => null;
    
    public bool IsEnabled(LogLevel logLevel) => logLevel >= LogLevel.Information;
    
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, 
        Exception? exception, Func<TState, Exception?, string> formatter)
    {
        if (!IsEnabled(logLevel)) return;
        
        var message = $"[{DateTime.Now:HH:mm:ss}] [{logLevel}] {_category}: {formatter(state, exception)}";
        if (exception != null) message += $"\n{exception}";
        
        lock (_lock)
        {
            File.AppendAllText(_logPath, message + "\n");
        }
    }
}
```

---

## Step 339: Performance Profiling

```csharp
// ============================================
// Performance Profiling
// ============================================

public class PerformanceTracker
{
    private readonly Dictionary<string, List<double>> _metrics = new();
    private readonly ILogger<PerformanceTracker> _logger;
    
    public PerformanceTracker(ILogger<PerformanceTracker> logger) => _logger = logger;
    
    public IDisposable Track(string operation)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        return new TrackingScope(operation, () =>
        {
            sw.Stop();
            RecordMetric(operation, sw.Elapsed.TotalMilliseconds);
        });
    }
    
    private void RecordMetric(string operation, double ms)
    {
        if (!_metrics.ContainsKey(operation))
            _metrics[operation] = new List<double>();
        
        _metrics[operation].Add(ms);
        
        if (ms > 500)
            _logger.LogWarning("Slow operation: {Operation} took {Ms}ms", operation, ms);
    }
    
    public OperationStats GetStats(string operation)
    {
        if (!_metrics.TryGetValue(operation, out var values) || values.Count == 0)
            return new OperationStats(operation, 0, 0, 0, 0, 0);
        
        var sorted = values.OrderBy(v => v).ToArray();
        return new OperationStats(
            operation,
            sorted.Length,
            sorted.Average(),
            sorted[0],
            sorted[^1],
            sorted[(int)(sorted.Length * 0.95)]
        );
    }
    
    public Dictionary<string, OperationStats> GetAllStats()
        => _metrics.Keys.ToDictionary(k => k, k => GetStats(k));
    
    private class TrackingScope : IDisposable
    {
        private readonly Action _onDispose;
        public TrackingScope(string op, Action onDispose) => _onDispose = onDispose;
        public void Dispose() => _onDispose();
    }
}

public record OperationStats(
    string Operation,
    int Count,
    double AverageMs,
    double MinMs,
    double MaxMs,
    double P95Ms
);

// Usage
public class ProductService3
{
    private readonly PerformanceTracker _tracker;
    
    public async Task<List<Product>> GetProductsAsync()
    {
        using var _ = _tracker.Track("GetProducts");
        return await _repo.GetAllAsync();
    }
}
```

---

## Step 340: Integration Testing

```csharp
// ============================================
// Integration Tests for Enterprise Features
// ============================================

public class EnterpriseSyncTests : IAsyncLifetime
{
    private SQLiteAsyncConnection _db = null!;
    private SyncEngine<Product> _syncEngine = null!;
    private Mock<IApiService<Product>> _mockApi = null!;
    
    public async Task InitializeAsync()
    {
        _db = new SQLiteAsyncConnection(":memory:");
        await _db.CreateTableAsync<Product>();
        
        _mockApi = new Mock<IApiService<Product>>();
        // Create sync engine with test setup
    }
    
    public async Task DisposeAsync() => await _db.CloseAsync();
    
    [Fact]
    public async Task Sync_LocalNew_PushesToServer()
    {
        // Arrange
        var newProduct = new Product 
        { 
            Name = "New Product", Price = 100, IsNew = true,
            ModifiedAt = DateTime.UtcNow
        };
        await _db.InsertAsync(newProduct);
        
        _mockApi.Setup(a => a.GetByIdAsync(It.IsAny<string>()))
            .ReturnsAsync((Product?)null);
        _mockApi.Setup(a => a.UpdateAsync(It.IsAny<string>(), It.IsAny<Product>()))
            .Returns(Task.CompletedTask);
        
        // Act
        var result = await _syncEngine.SyncAsync();
        
        // Assert
        result.PushedCount.Should().Be(1);
        _mockApi.Verify(a => a.UpdateAsync(It.IsAny<string>(), It.IsAny<Product>()), Times.Once);
    }
    
    [Fact]
    public async Task Sync_ServerNewer_UpdatesLocal()
    {
        // Arrange
        var localProduct = new Product { Name = "Local", ModifiedAt = DateTime.UtcNow.AddHours(-1) };
        var serverProduct = new Product { Name = "Server", ModifiedAt = DateTime.UtcNow };
        
        _mockApi.Setup(a => a.GetUpdatedSinceAsync(It.IsAny<DateTime>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(new List<Product> { serverProduct });
        
        // Act
        var result = await _syncEngine.SyncAsync();
        
        // Assert
        result.PulledCount.Should().Be(1);
    }
}
```

---

## สรุป Part 34

ใน Part 34 เราได้เรียนรู้:

1. **Multi-Tenant** - Tenant isolation and config
2. **Audit Trail** - Log all data changes
3. **Data Export** - CSV/JSON export
4. **Microservices Client** - Gateway pattern
5. **Sync Engine** - Conflict resolution
6. **Notification Center** - In-app notifications
7. **Plugin Architecture** - Extensible app
8. **Structured Logging** - ILogger, file logging
9. **Performance Profiling** - Track slow operations
10. **Integration Tests** - Test enterprise features

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 34 | Steps 331-340*
