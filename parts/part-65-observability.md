# Part 65: Observability & Monitoring
## Steps 641-650: OpenTelemetry, Metrics, Tracing, Health Checks

---

## Step 641: OpenTelemetry Setup

```csharp
// ============================================
// OpenTelemetry Instrumentation
// ============================================

using System.Diagnostics;
using System.Diagnostics.Metrics;

public static class Telemetry
{
    public static readonly ActivitySource ActivitySource =
        new("MyApp", "1.0.0");
    
    public static readonly Meter Meter =
        new("MyApp", "1.0.0");
    
    // Counters
    public static readonly Counter<int> OrdersPlaced =
        Meter.CreateCounter<int>("orders.placed", "count", "Total orders placed");
    
    public static readonly Counter<int> LoginAttempts =
        Meter.CreateCounter<int>("auth.login.attempts", "count");
    
    public static readonly Counter<int> ApiErrors =
        Meter.CreateCounter<int>("api.errors", "count");
    
    // Histograms
    public static readonly Histogram<double> ApiLatency =
        Meter.CreateHistogram<double>("api.latency", "ms", "API call latency");
    
    public static readonly Histogram<double> DatabaseQueryTime =
        Meter.CreateHistogram<double>("db.query.time", "ms");
    
    // Gauges
    public static ObservableGauge<int> CartItemsGauge =
        Meter.CreateObservableGauge("cart.items", () => GetCurrentCartItemCount());
    
    private static int GetCurrentCartItemCount() => 0; // hook to actual value
}

// Tracing service
public class TraceService
{
    private readonly ActivitySource _source;
    
    public TraceService(ActivitySource source) => _source = source;
    
    public Activity? StartActivity(string name, ActivityKind kind = ActivityKind.Internal)
        => _source.StartActivity(name, kind);
    
    public Activity? StartSpan(string spanName, Dictionary<string, object>? tags = null)
    {
        var activity = _source.StartActivity(spanName);
        if (activity != null && tags != null)
            foreach (var (key, value) in tags)
                activity.SetTag(key, value);
        return activity;
    }
}

// Instrumented order service
public class InstrumentedOrderService
{
    private readonly IOrderService _inner;
    private readonly TraceService _tracer;
    
    public InstrumentedOrderService(IOrderService inner, TraceService tracer)
    {
        _inner = inner;
        _tracer = tracer;
    }
    
    public async Task<int> PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        using var activity = _tracer.StartActivity("order.place", ActivityKind.Internal);
        activity?.SetTag("customer.id", cmd.CustomerId);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var orderId = await _inner.PlaceOrderAsync(cmd);
            
            Telemetry.OrdersPlaced.Add(1,
                new KeyValuePair<string, object?>("payment_method", cmd.PaymentMethod));
            
            activity?.SetTag("order.id", orderId);
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            return orderId;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            
            Telemetry.ApiErrors.Add(1,
                new KeyValuePair<string, object?>("operation", "place_order"),
                new KeyValuePair<string, object?>("error_type", ex.GetType().Name));
            
            throw;
        }
        finally
        {
            stopwatch.Stop();
            Telemetry.ApiLatency.Record(stopwatch.ElapsedMilliseconds,
                new KeyValuePair<string, object?>("operation", "place_order"));
        }
    }
}
```

---

## Step 642: App Performance Monitor

```csharp
// ============================================
// Performance Monitoring Service
// ============================================

public class AppPerformanceMonitor
{
    private readonly SQLiteConnection _db;
    private static readonly List<PerformanceRecord> _buffer = new();
    private static readonly SemaphoreSlim _lock = new(1, 1);
    
    public AppPerformanceMonitor(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<PerformanceRecord>();
    }
    
    public async Task TrackAsync(string metric, double value,
        Dictionary<string, string>? tags = null)
    {
        await _lock.WaitAsync();
        try
        {
            _buffer.Add(new PerformanceRecord
            {
                Metric = metric,
                Value = value,
                Tags = tags != null
                    ? System.Text.Json.JsonSerializer.Serialize(tags)
                    : null,
                Timestamp = DateTime.UtcNow
            });
            
            if (_buffer.Count >= 50)
                await FlushBufferAsync();
        }
        finally { _lock.Release(); }
    }
    
    private async Task FlushBufferAsync()
    {
        if (_buffer.Count == 0) return;
        
        var toWrite = _buffer.ToList();
        _buffer.Clear();
        
        _db.InsertAll(toWrite);
        
        // Upload to server asynchronously
        _ = UploadMetricsAsync(toWrite);
    }
    
    private async Task UploadMetricsAsync(List<PerformanceRecord> records)
    {
        try
        {
            // batch upload to analytics server
            await Task.Delay(100); // placeholder
        }
        catch { /* silent fail */ }
    }
    
    // Convenience methods
    public Task TrackScreenLoadAsync(string screen, double loadTimeMs)
        => TrackAsync("screen_load", loadTimeMs, new() { ["screen"] = screen });
    
    public Task TrackApiCallAsync(string endpoint, int statusCode, double latencyMs)
        => TrackAsync("api_call", latencyMs, new()
        {
            ["endpoint"] = endpoint,
            ["status"] = statusCode.ToString()
        });
    
    public Task TrackCrashAsync(string exceptionType, string? screen)
        => TrackAsync("crash", 1, new()
        {
            ["type"] = exceptionType,
            ["screen"] = screen ?? "unknown"
        });
}

public class PerformanceRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Metric { get; set; } = string.Empty;
    public double Value { get; set; }
    public string? Tags { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## Step 643: Structured Logging

```csharp
// ============================================
// Structured Logging
// ============================================

public interface IStructuredLogger
{
    void Log(LogLevel level, string message, Dictionary<string, object>? props = null, Exception? ex = null);
    void Info(string message, Dictionary<string, object>? props = null);
    void Warning(string message, Dictionary<string, object>? props = null);
    void Error(string message, Exception? ex = null, Dictionary<string, object>? props = null);
}

public class StructuredLogger : IStructuredLogger
{
    private readonly SQLiteConnection _db;
    private readonly string _context;
    
    public StructuredLogger(SQLiteConnection db, string context)
    {
        _db = db;
        _context = context;
        _db.CreateTable<LogEntry>();
    }
    
    public void Log(LogLevel level, string message,
        Dictionary<string, object>? props = null, Exception? ex = null)
    {
        var entry = new LogEntry
        {
            Level = level.ToString(),
            Message = message,
            Context = _context,
            Properties = props != null
                ? System.Text.Json.JsonSerializer.Serialize(props) : null,
            ExceptionType = ex?.GetType().Name,
            ExceptionMessage = ex?.Message,
            StackTrace = ex?.StackTrace?.Truncate(2000),
            DeviceId = DeviceInfo.Current.Name,
            AppVersion = AppInfo.Current.VersionString,
            Platform = DeviceInfo.Current.Platform.ToString(),
            OsVersion = DeviceInfo.Current.VersionString,
            Timestamp = DateTime.UtcNow
        };
        
        _db.Insert(entry);
        
        // Console output in debug
#if DEBUG
        System.Diagnostics.Debug.WriteLine(
            $"[{entry.Level}] [{_context}] {message}" +
            (ex != null ? $"\n{ex}" : ""));
#endif
    }
    
    public void Info(string message, Dictionary<string, object>? props = null)
        => Log(LogLevel.Information, message, props);
    
    public void Warning(string message, Dictionary<string, object>? props = null)
        => Log(LogLevel.Warning, message, props);
    
    public void Error(string message, Exception? ex = null, Dictionary<string, object>? props = null)
        => Log(LogLevel.Error, message, props, ex);
}

public class LogEntry
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Level { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public string Context { get; set; } = string.Empty;
    public string? Properties { get; set; }
    public string? ExceptionType { get; set; }
    public string? ExceptionMessage { get; set; }
    public string? StackTrace { get; set; }
    public string DeviceId { get; set; } = string.Empty;
    public string AppVersion { get; set; } = string.Empty;
    public string Platform { get; set; } = string.Empty;
    public string OsVersion { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
}

public static class StringExtensions
{
    public static string Truncate(this string s, int maxLength)
        => s.Length <= maxLength ? s : s[..maxLength] + "...";
}
```

---

## Step 644: Health Check Dashboard

```csharp
// ============================================
// Health Check System
// ============================================

public interface IHealthCheck
{
    string Name { get; }
    Task<HealthCheckResult> CheckAsync(CancellationToken ct = default);
}

public record HealthCheckResult(
    string Name, HealthStatus Status,
    string? Description = null, TimeSpan? Duration = null, Exception? Exception = null);

public enum HealthStatus { Healthy, Degraded, Unhealthy }

// Database health check
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly SQLiteConnection _db;
    public string Name => "Database";
    
    public DatabaseHealthCheck(SQLiteConnection db) => _db = db;
    
    public Task<HealthCheckResult> CheckAsync(CancellationToken ct = default)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            // Simple ping query
            var count = _db.ExecuteScalar<int>("SELECT COUNT(*) FROM sqlite_master");
            sw.Stop();
            
            var status = sw.ElapsedMilliseconds switch
            {
                < 100 => HealthStatus.Healthy,
                < 500 => HealthStatus.Degraded,
                _ => HealthStatus.Unhealthy
            };
            
            return Task.FromResult(new HealthCheckResult(
                Name, status, $"Query took {sw.ElapsedMilliseconds}ms", sw.Elapsed));
        }
        catch (Exception ex)
        {
            return Task.FromResult(new HealthCheckResult(
                Name, HealthStatus.Unhealthy, ex.Message, sw.Elapsed, ex));
        }
    }
}

// API health check
public class ApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _http;
    public string Name => "API";
    
    public ApiHealthCheck(HttpClient http) => _http = http;
    
    public async Task<HealthCheckResult> CheckAsync(CancellationToken ct = default)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
            cts.CancelAfter(5000);
            
            var response = await _http.GetAsync("/health", cts.Token);
            sw.Stop();
            
            var status = response.IsSuccessStatusCode
                ? sw.ElapsedMilliseconds < 2000
                    ? HealthStatus.Healthy : HealthStatus.Degraded
                : HealthStatus.Unhealthy;
            
            return new HealthCheckResult(
                Name, status,
                $"HTTP {(int)response.StatusCode} in {sw.ElapsedMilliseconds}ms",
                sw.Elapsed);
        }
        catch (Exception ex)
        {
            return new HealthCheckResult(
                Name, HealthStatus.Unhealthy, ex.Message, sw.Elapsed, ex);
        }
    }
}

// Aggregated health service
public class HealthCheckService
{
    private readonly List<IHealthCheck> _checks;
    
    public HealthCheckService(IEnumerable<IHealthCheck> checks)
        => _checks = checks.ToList();
    
    public async Task<AppHealthReport> CheckAllAsync(CancellationToken ct = default)
    {
        var results = await Task.WhenAll(_checks.Select(c => c.CheckAsync(ct)));
        
        var overall = results.All(r => r.Status == HealthStatus.Healthy)
            ? HealthStatus.Healthy
            : results.Any(r => r.Status == HealthStatus.Unhealthy)
                ? HealthStatus.Unhealthy
                : HealthStatus.Degraded;
        
        return new AppHealthReport(overall, results.ToList(), DateTime.UtcNow);
    }
}

public record AppHealthReport(
    HealthStatus Overall, List<HealthCheckResult> Checks, DateTime CheckedAt)
{
    public bool IsHealthy => Overall == HealthStatus.Healthy;
}
```

---

## Step 645: Error Reporting

```csharp
// ============================================
// Error Reporting Service
// ============================================

public class ErrorReportingService
{
    private readonly SQLiteConnection _db;
    private readonly IStructuredLogger _logger;
    private static bool _initialized;
    
    public ErrorReportingService(SQLiteConnection db, IStructuredLogger logger)
    {
        _db = db;
        _logger = logger;
        _db.CreateTable<ErrorReport>();
    }
    
    public void Initialize()
    {
        if (_initialized) return;
        _initialized = true;
        
        // Catch unhandled exceptions
        AppDomain.CurrentDomain.UnhandledException += OnUnhandledException;
        TaskScheduler.UnobservedTaskException += OnUnobservedTaskException;
        
        // MAUI specific
        MauiExceptions.UnhandledException += OnMauiUnhandledException;
    }
    
    private void OnUnhandledException(object sender, UnhandledExceptionEventArgs e)
    {
        var ex = e.ExceptionObject as Exception;
        CaptureException(ex, "UnhandledException", e.IsTerminating);
    }
    
    private void OnUnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
    {
        CaptureException(e.Exception, "UnobservedTaskException", false);
        e.SetObserved();
    }
    
    private void OnMauiUnhandledException(object? sender, UnhandledExceptionEventArgs e)
    {
        var ex = e.ExceptionObject as Exception;
        CaptureException(ex, "MauiUnhandledException", false);
    }
    
    private void CaptureException(Exception? ex, string source, bool isFatal)
    {
        if (ex == null) return;
        
        try
        {
            var report = new ErrorReport
            {
                ExceptionType = ex.GetType().FullName ?? "Unknown",
                Message = ex.Message,
                StackTrace = ex.StackTrace?.Truncate(5000),
                Source = source,
                IsFatal = isFatal,
                AppVersion = AppInfo.Current.VersionString,
                Platform = DeviceInfo.Current.Platform.ToString(),
                OsVersion = DeviceInfo.Current.VersionString,
                DeviceModel = DeviceInfo.Current.Model,
                Timestamp = DateTime.UtcNow,
                Uploaded = false
            };
            
            _db.Insert(report);
            _logger.Error($"Captured {source}: {ex.Message}", ex);
        }
        catch { /* never throw in error handler */ }
    }
    
    public async Task UploadPendingAsync()
    {
        var pending = _db.Table<ErrorReport>()
            .Where(r => !r.Uploaded)
            .Take(20)
            .ToList();
        
        foreach (var report in pending)
        {
            try
            {
                // POST to crash reporting endpoint
                report.Uploaded = true;
                report.UploadedAt = DateTime.UtcNow;
                _db.Update(report);
            }
            catch { break; }
        }
    }
}

public class ErrorReport
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string ExceptionType { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public string? StackTrace { get; set; }
    public string Source { get; set; } = string.Empty;
    public bool IsFatal { get; set; }
    public string AppVersion { get; set; } = string.Empty;
    public string Platform { get; set; } = string.Empty;
    public string OsVersion { get; set; } = string.Empty;
    public string DeviceModel { get; set; } = string.Empty;
    public bool Uploaded { get; set; }
    public DateTime Timestamp { get; set; }
    public DateTime? UploadedAt { get; set; }
}
```

---

## Step 646: User Analytics

```csharp
// ============================================
// User Behavior Analytics
// ============================================

public class UserAnalyticsService
{
    private readonly SQLiteConnection _db;
    private readonly string _sessionId = Guid.NewGuid().ToString("N");
    private DateTime _sessionStart = DateTime.UtcNow;
    private string? _currentScreen;
    
    public UserAnalyticsService(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<AnalyticsEvent>();
    }
    
    public void TrackScreenView(string screenName)
    {
        // End previous screen session
        if (_currentScreen != null)
        {
            Track("screen_exit", new() { ["screen"] = _currentScreen });
        }
        
        _currentScreen = screenName;
        Track("screen_view", new() { ["screen"] = screenName });
    }
    
    public void TrackButtonTap(string buttonName, string? screen = null)
        => Track("button_tap", new()
        {
            ["button"] = buttonName,
            ["screen"] = screen ?? _currentScreen ?? "unknown"
        });
    
    public void TrackSearch(string query, int resultCount)
        => Track("search", new()
        {
            ["query"] = query,
            ["result_count"] = resultCount.ToString()
        });
    
    public void TrackAddToCart(int productId, string productName, decimal price)
        => Track("add_to_cart", new()
        {
            ["product_id"] = productId.ToString(),
            ["product_name"] = productName,
            ["price"] = price.ToString("F2")
        });
    
    public void TrackPurchase(int orderId, decimal total, string paymentMethod)
        => Track("purchase", new()
        {
            ["order_id"] = orderId.ToString(),
            ["total"] = total.ToString("F2"),
            ["payment_method"] = paymentMethod
        });
    
    private void Track(string eventName, Dictionary<string, string>? properties = null)
    {
        _db.Insert(new AnalyticsEvent
        {
            SessionId = _sessionId,
            EventName = eventName,
            Properties = properties != null
                ? System.Text.Json.JsonSerializer.Serialize(properties) : null,
            UserId = GetCurrentUserId(),
            AppVersion = AppInfo.Current.VersionString,
            Timestamp = DateTime.UtcNow
        });
    }
    
    private int? GetCurrentUserId() => null; // hook to auth
}

public class AnalyticsEvent
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string SessionId { get; set; } = string.Empty;
    public string EventName { get; set; } = string.Empty;
    public string? Properties { get; set; }
    public int? UserId { get; set; }
    public string AppVersion { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
    public bool Sent { get; set; }
}
```

---

## Step 647: Network Monitor

```csharp
// ============================================
// Network Monitor
// ============================================

public class NetworkMonitor : IDisposable
{
    private readonly IConnectivity _connectivity;
    private readonly IStructuredLogger _logger;
    private readonly UserAnalyticsService _analytics;
    
    public event Action<bool>? IsOnlineChanged;
    public bool IsOnline { get; private set; }
    public ConnectionType ConnectionType { get; private set; }
    
    public NetworkMonitor(
        IConnectivity connectivity, IStructuredLogger logger,
        UserAnalyticsService analytics)
    {
        _connectivity = connectivity;
        _logger = logger;
        _analytics = analytics;
        
        IsOnline = connectivity.NetworkAccess == NetworkAccess.Internet;
        ConnectionType = connectivity.ConnectionProfiles.FirstOrDefault();
        
        connectivity.ConnectivityChanged += OnConnectivityChanged;
    }
    
    private void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
    {
        var wasOnline = IsOnline;
        IsOnline = e.NetworkAccess == NetworkAccess.Internet;
        ConnectionType = e.ConnectionProfiles.FirstOrDefault();
        
        if (wasOnline != IsOnline)
        {
            _logger.Info(IsOnline ? "Network connected" : "Network disconnected",
                new() { ["connection_type"] = ConnectionType.ToString() });
            
            IsOnlineChanged?.Invoke(IsOnline);
        }
    }
    
    public void Dispose()
        => _connectivity.ConnectivityChanged -= OnConnectivityChanged;
}

// Network-aware ViewModel
public partial class NetworkAwareViewModel : ObservableObject, IDisposable
{
    private readonly NetworkMonitor _network;
    
    [ObservableProperty] private bool _isOnline;
    [ObservableProperty] private string _networkMessage = string.Empty;
    
    public NetworkAwareViewModel(NetworkMonitor network)
    {
        _network = network;
        IsOnline = network.IsOnline;
        
        network.IsOnlineChanged += isOnline =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                IsOnline = isOnline;
                NetworkMessage = isOnline
                    ? string.Empty
                    : "ไม่มีการเชื่อมต่ออินเทอร์เน็ต";
            });
        };
    }
    
    public void Dispose() { }
}
```

---

## Step 648: Diagnostic Tools

```csharp
// ============================================
// In-App Diagnostic Tools
// ============================================

public class DiagnosticService
{
    private readonly SQLiteConnection _db;
    private readonly HealthCheckService _health;
    private readonly IStructuredLogger _logger;
    
    public DiagnosticService(
        SQLiteConnection db, HealthCheckService health, IStructuredLogger logger)
    {
        _db = db;
        _health = health;
        _logger = logger;
    }
    
    public async Task<DiagnosticReport> GenerateReportAsync()
    {
        var health = await _health.CheckAllAsync();
        var logCount = _db.ExecuteScalar<int>("SELECT COUNT(*) FROM LogEntry");
        var errorCount = _db.ExecuteScalar<int>(
            "SELECT COUNT(*) FROM ErrorReport WHERE Uploaded = 0");
        
        return new DiagnosticReport(
            AppVersion: AppInfo.Current.VersionString,
            Platform: $"{DeviceInfo.Current.Platform} {DeviceInfo.Current.VersionString}",
            DeviceModel: $"{DeviceInfo.Current.Manufacturer} {DeviceInfo.Current.Model}",
            MemoryUsage: GC.GetTotalMemory(false),
            StorageUsed: GetDbSize(),
            Health: health,
            PendingLogs: logCount,
            PendingErrors: errorCount,
            Timestamp: DateTime.UtcNow);
    }
    
    private long GetDbSize()
    {
        try
        {
            var dbPath = Path.Combine(FileSystem.AppDataDirectory, "app.db");
            return new FileInfo(dbPath).Length;
        }
        catch { return 0; }
    }
    
    public async Task ExportLogsAsync(string filePath)
    {
        var logs = _db.Table<LogEntry>()
            .OrderByDescending(l => l.Timestamp)
            .Take(1000)
            .ToList();
        
        var content = System.Text.Json.JsonSerializer.Serialize(logs,
            new System.Text.Json.JsonSerializerOptions { WriteIndented = true });
        
        await File.WriteAllTextAsync(filePath, content);
    }
    
    public void ClearLogs()
    {
        _db.Execute("DELETE FROM LogEntry WHERE Timestamp < ?",
            DateTime.UtcNow.AddDays(-7));
    }
}

public record DiagnosticReport(
    string AppVersion, string Platform, string DeviceModel,
    long MemoryUsage, long StorageUsed, AppHealthReport Health,
    int PendingLogs, int PendingErrors, DateTime Timestamp);
```

---

## Step 649: Performance Baseline

```csharp
// ============================================
// Performance Baseline Tests
// ============================================

[MemoryDiagnoser]
[RankColumn]
public class AppPerformanceBaseline
{
    private SQLiteConnection _db = null!;
    private List<ProductData> _testData = null!;
    
    [GlobalSetup]
    public void Setup()
    {
        _db = new SQLiteConnection(":memory:");
        _db.CreateTable<ProductData>();
        
        // Seed 10,000 products
        _testData = Enumerable.Range(1, 10000).Select(i => new ProductData
        {
            Id = i, Name = $"Product {i}",
            Category = $"Category {i % 20}",
            Price = i * 10m, StockQuantity = i % 100
        }).ToList();
        _db.InsertAll(_testData);
    }
    
    [Benchmark(Baseline = true)]
    public List<ProductData> ListAll()
        => _db.Table<ProductData>().ToList();
    
    [Benchmark]
    public List<ProductData> FilteredQuery()
        => _db.Table<ProductData>()
            .Where(p => p.Category == "Category 5" && p.StockQuantity > 0)
            .OrderBy(p => p.Price)
            .Take(20)
            .ToList();
    
    [Benchmark]
    public List<ProductData> RawSqlPaged()
        => _db.Query<ProductData>(
            "SELECT * FROM ProductData WHERE StockQuantity > 0 ORDER BY Price LIMIT 20 OFFSET 0");
    
    [Benchmark]
    public string SerializeProducts()
        => System.Text.Json.JsonSerializer.Serialize(_testData.Take(100).ToList());
    
    [Benchmark]
    public List<ProductDto> MapAndProject()
        => _testData.Take(100)
            .Select(p => new ProductDto(p.Id, p.Name, p.Category, p.Price))
            .ToList();
}
```

---

## Step 650: Debug Console

```csharp
// ============================================
// Debug Console (Dev builds only)
// ============================================

#if DEBUG
public partial class DebugConsolePage : ContentPage
{
    private readonly DiagnosticService _diag;
    private readonly HealthCheckService _health;
    
    public DebugConsolePage(DiagnosticService diag, HealthCheckService health)
    {
        _diag = diag;
        _health = health;
        
        BuildUI();
    }
    
    private void BuildUI()
    {
        Title = "Debug Console";
        
        Content = new ScrollView
        {
            Content = new VerticalStackLayout
            {
                Padding = new Thickness(16),
                Spacing = 12,
                Children =
                {
                    CreateSection("System Info", new[]
                    {
                        $"App: {AppInfo.Current.VersionString}",
                        $"Platform: {DeviceInfo.Current.Platform}",
                        $"OS: {DeviceInfo.Current.VersionString}",
                        $"Model: {DeviceInfo.Current.Model}",
                        $"Memory: {GC.GetTotalMemory(false) / 1024 / 1024}MB"
                    }),
                    
                    CreateButton("Run Health Checks", async () =>
                    {
                        var report = await _health.CheckAllAsync();
                        await DisplayAlert("Health",
                            string.Join("\n", report.Checks.Select(c => $"{c.Name}: {c.Status}")),
                            "OK");
                    }),
                    
                    CreateButton("Export Logs", async () =>
                    {
                        var path = Path.Combine(FileSystem.CacheDirectory, "logs.json");
                        await _diag.ExportLogsAsync(path);
                        await Share.Default.RequestAsync(new ShareFileRequest(path));
                    }),
                    
                    CreateButton("Clear Logs", async () =>
                    {
                        _diag.ClearLogs();
                        await DisplayAlert("Done", "Logs cleared", "OK");
                    }),
                    
                    CreateButton("Generate Crash", () =>
                    {
                        throw new Exception("Manual test crash");
                    }),
                }
            }
        };
    }
    
    private static VerticalStackLayout CreateSection(string title, string[] items)
    {
        var layout = new VerticalStackLayout { Spacing = 4 };
        layout.Children.Add(new Label { Text = title, FontAttributes = FontAttributes.Bold, FontSize = 16 });
        foreach (var item in items)
            layout.Children.Add(new Label { Text = item, FontSize = 13, TextColor = Colors.Gray });
        return layout;
    }
    
    private static Button CreateButton(string text, Func<Task> action)
        => new() { Text = text, Clicked = async (_, _) => await action() };
    
    private static Button CreateButton(string text, Action action)
        => new() { Text = text, Clicked = (_, _) => action() };
}
#endif
```

---

## สรุป Part 65

ใน Part 65 เราได้เรียนรู้:

1. **OpenTelemetry** - ActivitySource, Meter, counters, histograms
2. **Performance Monitor** - Buffered metrics, upload
3. **Structured Logging** - Log entries with context
4. **Health Check Dashboard** - DB, API health checks
5. **Error Reporting** - Unhandled exception capture
6. **User Analytics** - Screen views, events, purchases
7. **Network Monitor** - Connectivity change events
8. **Diagnostic Tools** - Report generation, log export
9. **Performance Baseline** - BenchmarkDotNet tests
10. **Debug Console** - Dev-only diagnostics page

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 65 | Steps 641-650*
