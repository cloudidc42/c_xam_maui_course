# Part 49: Performance Optimization
## Steps 481-490: Profiling, Memory, Rendering

---

## Step 481: Memory Management

```csharp
// ============================================
// Memory Management & Avoiding Leaks
// ============================================

// ❌ Memory leak: event subscription not disposed
public class LeakyViewModel : ObservableObject
{
    public LeakyViewModel(IEventBus eventBus)
    {
        // Never unsubscribed → ViewModel never collected
        eventBus.Subscribe<ProductUpdatedEvent>(OnProductUpdated);
    }
    
    private void OnProductUpdated(ProductUpdatedEvent e) { }
}

// ✅ Proper disposal
public class CleanViewModel : ObservableObject, IDisposable
{
    private readonly CompositeDisposable _disposables = new();
    
    public CleanViewModel(IEventBus eventBus)
    {
        var sub = eventBus.Subscribe<ProductUpdatedEvent>(OnProductUpdated);
        _disposables.Add(sub);
    }
    
    private void OnProductUpdated(ProductUpdatedEvent e) { }
    
    public void Dispose() => _disposables.Dispose();
}

// ❌ Closure capturing large object
public class ClosureLeak
{
    private byte[] _largeData = new byte[1024 * 1024]; // 1MB
    
    public Func<string> CreateLeakyFunc()
    {
        // Captures 'this' → _largeData never collected
        return () => _largeData.Length.ToString();
    }
    
    public Func<string> CreateCleanFunc()
    {
        var length = _largeData.Length; // capture only what's needed
        return () => length.ToString();
    }
}

// WeakReference for callbacks
public class EventAggregator
{
    private readonly List<WeakReference<Func<object, Task>>> _handlers = new();
    
    public void Subscribe(Func<object, Task> handler)
    {
        _handlers.Add(new WeakReference<Func<object, Task>>(handler));
    }
    
    public async Task PublishAsync(object message)
    {
        var alive = new List<WeakReference<Func<object, Task>>>();
        
        foreach (var weakRef in _handlers)
        {
            if (weakRef.TryGetTarget(out var handler))
            {
                alive.Add(weakRef);
                await handler(message);
            }
        }
        
        // Replace list with only alive references
        _handlers.Clear();
        _handlers.AddRange(alive);
    }
}

// Memory pool for frequent allocations
public class DataProcessor
{
    private static readonly System.Buffers.ArrayPool<byte> _pool = 
        System.Buffers.ArrayPool<byte>.Shared;
    
    public async Task ProcessAsync(Stream stream)
    {
        var buffer = _pool.Rent(4096);
        try
        {
            int bytesRead;
            while ((bytesRead = await stream.ReadAsync(buffer.AsMemory(0, buffer.Length))) > 0)
            {
                ProcessChunk(buffer.AsSpan(0, bytesRead));
            }
        }
        finally
        {
            _pool.Return(buffer);
        }
    }
    
    private static void ProcessChunk(Span<byte> data) { /* process */ }
}
```

---

## Step 482: Span<T> and stackalloc

```csharp
// ============================================
// High-Performance C# with Span<T>
// ============================================

public class HighPerfParsing
{
    // ❌ Allocates strings
    public static IEnumerable<string> ParseCsvLineSlow(string line)
    {
        return line.Split(',').Select(s => s.Trim());
    }
    
    // ✅ Zero allocation with Span
    public static void ParseCsvLineFast(ReadOnlySpan<char> line,
        Span<ReadOnlyMemory<char>> fields, out int count)
    {
        count = 0;
        int start = 0;
        
        for (int i = 0; i <= line.Length; i++)
        {
            if (i == line.Length || line[i] == ',')
            {
                var field = line.Slice(start, i - start);
                fields[count++] = line.Slice(start, i - start).ToString().AsMemory();
                start = i + 1;
            }
        }
    }
    
    // stackalloc for small buffers (avoids heap allocation)
    public static string FormatOrderId(int id)
    {
        Span<char> buffer = stackalloc char[20];
        if (!id.TryFormat(buffer, out int written))
            return id.ToString();
        
        return new string(buffer[..written]);
    }
    
    // ❌ Boxing with generic constraint
    public static string FormatAmount(decimal amount)
    {
        return amount.ToString("N2"); // internal boxing
    }
    
    // ✅ No boxing
    public static bool TryFormatAmount(decimal amount, Span<char> destination, out int written)
    {
        return amount.TryFormat(destination, out written, "N2");
    }
}

// StringBuilder pooling
public class TextFormatter
{
    private static readonly System.Text.StringBuilder _pool = new();
    
    public static string BuildOrderSummary(Order order)
    {
        var sb = System.Text.StringBuilder_Pool.Rent();
        try
        {
            sb.AppendLine($"คำสั่งซื้อ #{order.Id}");
            sb.AppendLine($"ลูกค้า: {order.CustomerId}");
            foreach (var line in order.Lines)
                sb.AppendLine($"  - {line.Name}: {line.Quantity} x {line.UnitPrice}");
            sb.AppendLine($"รวม: {order.TotalAmount}");
            
            return sb.ToString();
        }
        finally
        {
            System.Text.StringBuilder_Pool.Return(sb);
        }
    }
}
```

---

## Step 483: CollectionView Performance

```xml
<!-- ============================================
     CollectionView Performance Optimization
     ============================================ -->
<CollectionView
    ItemsSource="{Binding Products}"
    RemainingItemsThreshold="5"
    RemainingItemsThresholdReachedCommand="{Binding LoadMoreCommand}"
    SelectionMode="None">
    
    <!-- Virtualization is on by default in CollectionView -->
    <!-- Use LinearItemsLayout for best perf on large lists -->
    <CollectionView.ItemsLayout>
        <LinearItemsLayout Orientation="Vertical" ItemSpacing="8" />
    </CollectionView.ItemsLayout>
    
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="models:Product">
            <!-- Keep items simple - no nested layouts -->
            <Grid Padding="16,8" ColumnDefinitions="60,*,Auto">
                <!-- Use compiled bindings -->
                <Image Grid.Column="0"
                       Source="{Binding ImageUrl}"
                       Aspect="AspectFill"
                       WidthRequest="50"
                       HeightRequest="50" />
                
                <VerticalStackLayout Grid.Column="1" Spacing="2">
                    <Label Text="{Binding Name}"
                           FontAttributes="Bold"
                           LineBreakMode="TailTruncation" />
                    <Label Text="{Binding Category}"
                           TextColor="{AppThemeBinding Light=Gray, Dark=LightGray}"
                           FontSize="12" />
                </VerticalStackLayout>
                
                <Label Grid.Column="2"
                       Text="{Binding Price, StringFormat='{0:N0} ฿'}"
                       FontAttributes="Bold"
                       TextColor="{StaticResource PrimaryColor}" />
            </Grid>
        </DataTemplate>
    </CollectionView.ItemTemplate>
    
    <CollectionView.EmptyView>
        <Label Text="ไม่พบสินค้า" HorizontalOptions="Center" />
    </CollectionView.EmptyView>
    
    <CollectionView.Footer>
        <ActivityIndicator IsRunning="{Binding IsLoadingMore}"
                           IsVisible="{Binding IsLoadingMore}" />
    </CollectionView.Footer>
</CollectionView>
```

```csharp
// ViewModel for infinite scroll
public partial class ProductListViewModel : ObservableObject
{
    private readonly IProductService _service;
    private int _page = 1;
    private const int PageSize = 20;
    
    [ObservableProperty] private ObservableCollection<Product> _products = new();
    [ObservableProperty] private bool _isLoading;
    [ObservableProperty] private bool _isLoadingMore;
    [ObservableProperty] private bool _hasMore = true;
    
    public ProductListViewModel(IProductService service) => _service = service;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        _page = 1;
        Products.Clear();
        HasMore = true;
        
        await LoadPageAsync();
        IsLoading = false;
    }
    
    [RelayCommand(CanExecute = nameof(CanLoadMore))]
    private async Task LoadMoreAsync()
    {
        if (!HasMore) return;
        IsLoadingMore = true;
        _page++;
        await LoadPageAsync();
        IsLoadingMore = false;
    }
    
    private bool CanLoadMore => HasMore && !IsLoadingMore;
    
    private async Task LoadPageAsync()
    {
        var items = await _service.GetPageAsync(_page, PageSize);
        
        if (items.Count < PageSize) HasMore = false;
        
        foreach (var item in items)
            Products.Add(item);
    }
}
```

---

## Step 484: Image Optimization

```csharp
// ============================================
// Image Loading & Caching
// ============================================

// Use FFImageLoading or MAUI's built-in caching
// For custom caching:
public class ImageCacheService
{
    private readonly string _cacheDir;
    private readonly Dictionary<string, SemaphoreSlim> _locks = new();
    private readonly SemaphoreSlim _lockLock = new(1, 1);
    
    public ImageCacheService()
    {
        _cacheDir = Path.Combine(FileSystem.CacheDirectory, "images");
        Directory.CreateDirectory(_cacheDir);
    }
    
    public async Task<string> GetCachedPathAsync(string url, CancellationToken ct = default)
    {
        var hash = Convert.ToHexString(
            System.Security.Cryptography.MD5.HashData(
                System.Text.Encoding.UTF8.GetBytes(url)));
        
        var ext = Path.GetExtension(new Uri(url).LocalPath);
        var cachePath = Path.Combine(_cacheDir, $"{hash}{ext}");
        
        if (File.Exists(cachePath)) return cachePath;
        
        // Per-URL locking to avoid duplicate downloads
        var sem = await GetLockAsync(url);
        await sem.WaitAsync(ct);
        try
        {
            if (File.Exists(cachePath)) return cachePath; // double-check
            
            using var http = new HttpClient();
            var data = await http.GetByteArrayAsync(url, ct);
            
            await File.WriteAllBytesAsync(cachePath, data, ct);
            return cachePath;
        }
        finally { sem.Release(); }
    }
    
    public async Task CleanOldFilesAsync(TimeSpan maxAge)
    {
        var cutoff = DateTime.UtcNow - maxAge;
        var files = Directory.GetFiles(_cacheDir)
            .Where(f => File.GetLastAccessTimeUtc(f) < cutoff);
        
        foreach (var file in files)
            File.Delete(file);
    }
    
    private async Task<SemaphoreSlim> GetLockAsync(string url)
    {
        await _lockLock.WaitAsync();
        try
        {
            if (!_locks.TryGetValue(url, out var sem))
                _locks[url] = sem = new SemaphoreSlim(1, 1);
            return sem;
        }
        finally { _lockLock.Release(); }
    }
}
```

---

## Step 485: Startup Performance

```csharp
// ============================================
// App Startup Optimization
// ============================================

// MauiProgram.cs - lazy initialization
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
            });
        
        // Register services lazily
        builder.Services.AddSingleton<IProductService, ProductService>();
        builder.Services.AddSingleton<IOrderService, OrderService>();
        
        // Defer heavy init
        builder.Services.AddSingleton<IDatabaseService>(sp =>
            new Lazy<IDatabaseService>(() => new DatabaseService()).Value);
        
        return builder.Build();
    }
}

// Splash screen optimized App.cs
public partial class App : Application
{
    private static Task? _warmupTask;
    
    public App(IServiceProvider services)
    {
        InitializeComponent();
        
        // Start warmup in background
        _warmupTask = WarmupAsync(services);
        
        MainPage = new AppShell();
    }
    
    private static async Task WarmupAsync(IServiceProvider services)
    {
        // Pre-warm DB connection
        var db = services.GetRequiredService<IDatabaseService>();
        await db.InitializeAsync();
        
        // Pre-load critical data
        var products = services.GetRequiredService<IProductService>();
        await products.PrefetchFeaturedAsync();
    }
}

// Lazy page creation
[QueryProperty("ProductId", "id")]
public partial class ProductDetailPage : ContentPage
{
    private ProductDetailViewModel? _viewModel;
    
    protected override void OnAppearing()
    {
        base.OnAppearing();
        
        // Create VM only when page first appears
        if (_viewModel == null)
        {
            _viewModel = IPlatformApplication.Current!.Services
                .GetRequiredService<ProductDetailViewModel>();
            BindingContext = _viewModel;
        }
        
        _viewModel.LoadCommand.Execute(null);
    }
}
```

---

## Step 486: Rendering Performance

```csharp
// ============================================
// Custom Handler Optimization
// ============================================

// Avoid expensive layouts
// ❌ Nested StackLayouts
/*
<StackLayout>
    <StackLayout Orientation="Horizontal">
        <StackLayout>
            <Label />
        </StackLayout>
    </StackLayout>
</StackLayout>
*/

// ✅ Grid is faster
/*
<Grid RowDefinitions="Auto,Auto" ColumnDefinitions="*,Auto">
    <Label Grid.Row="0" Grid.Column="0" />
    <Label Grid.Row="0" Grid.Column="1" />
</Grid>
*/

// Custom DrawingView for complex UI (SkiaSharp)
public class PerformantChartView : SKCanvasView
{
    private SKPicture? _cachedPicture;
    private List<float>? _lastData;
    
    public static readonly BindableProperty DataProperty =
        BindableProperty.Create(nameof(Data), typeof(List<float>), typeof(PerformantChartView),
            propertyChanged: (b, _, n) =>
            {
                var view = (PerformantChartView)b;
                view._lastData = (List<float>)n;
                view._cachedPicture = null; // invalidate cache
                view.InvalidateSurface();
            });
    
    public List<float>? Data
    {
        get => (List<float>?)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear();
        
        if (_lastData == null || _lastData.Count == 0) return;
        
        // Cache the picture recording
        if (_cachedPicture == null)
        {
            var recorder = new SKPictureRecorder();
            var bounds = new SKRect(0, 0, e.Info.Width, e.Info.Height);
            var recordCanvas = recorder.BeginRecording(bounds);
            
            DrawChart(recordCanvas, _lastData, e.Info);
            _cachedPicture = recorder.EndRecording();
        }
        
        canvas.DrawPicture(_cachedPicture);
    }
    
    private static void DrawChart(SKCanvas canvas, List<float> data, SKImageInfo info)
    {
        using var paint = new SKPaint
        {
            Color = SKColors.Blue,
            StrokeWidth = 2,
            IsAntialias = true,
            Style = SKPaintStyle.Stroke
        };
        
        var max = data.Max();
        var step = info.Width / (float)(data.Count - 1);
        
        var path = new SKPath();
        for (int i = 0; i < data.Count; i++)
        {
            var x = i * step;
            var y = info.Height - (data[i] / max * info.Height);
            
            if (i == 0) path.MoveTo(x, y);
            else path.LineTo(x, y);
        }
        
        canvas.DrawPath(path, paint);
    }
}
```

---

## Step 487: Database Query Optimization

```csharp
// ============================================
// SQLite Query Optimization
// ============================================

public class OptimizedProductRepository
{
    private readonly SQLiteConnection _db;
    
    // ❌ N+1 query problem
    public List<ProductWithCategory> GetAllSlow()
    {
        var products = _db.Table<Product>().ToList();
        return products.Select(p => new ProductWithCategory(
            p, _db.Get<Category>(p.CategoryId))).ToList(); // N queries!
    }
    
    // ✅ JOIN query
    public List<ProductWithCategory> GetAllFast()
    {
        return _db.Query<ProductWithCategory>("""
            SELECT p.*, c.Name as CategoryName, c.Color as CategoryColor
            FROM Product p
            JOIN Category c ON c.Id = p.CategoryId
            WHERE p.IsActive = 1
            ORDER BY p.Name
            """);
    }
    
    // ❌ Loading all then filtering in memory
    public List<Product> GetCheapSlow(decimal maxPrice)
        => _db.Table<Product>().ToList().Where(p => p.Price <= maxPrice).ToList();
    
    // ✅ Filter in SQL
    public List<Product> GetCheapFast(decimal maxPrice)
        => _db.Query<Product>("SELECT * FROM Product WHERE Price <= ?", maxPrice);
    
    // ✅ Count without loading all records
    public int GetTotalCount()
        => _db.ExecuteScalar<int>("SELECT COUNT(*) FROM Product WHERE IsActive = 1");
    
    // ✅ Keyset pagination (better than OFFSET for large tables)
    public List<Product> GetPageKeySet(int lastId, int size)
        => _db.Query<Product>(
            "SELECT * FROM Product WHERE Id > ? ORDER BY Id LIMIT ?",
            lastId, size);
    
    // ✅ Index usage - add index for frequently filtered columns
    public void EnsureIndexes()
    {
        _db.Execute("CREATE INDEX IF NOT EXISTS IX_Product_CategoryId ON Product(CategoryId)");
        _db.Execute("CREATE INDEX IF NOT EXISTS IX_Product_Price ON Product(Price)");
        _db.Execute("CREATE INDEX IF NOT EXISTS IX_Product_Name ON Product(Name)");
    }
    
    // ✅ Batch insert with transaction
    public void BulkInsert(IEnumerable<Product> products)
    {
        _db.BeginTransaction();
        try
        {
            foreach (var chunk in products.Chunk(500))
            {
                foreach (var p in chunk)
                    _db.Insert(p);
            }
            _db.Commit();
        }
        catch { _db.Rollback(); throw; }
    }
}
```

---

## Step 488: HTTP Performance

```csharp
// ============================================
// HTTP Client Optimization
// ============================================

// ❌ Creating new HttpClient per request (socket exhaustion)
public class SlowApiClient
{
    public async Task<Product?> GetProductAsync(int id)
    {
        using var client = new HttpClient(); // BAD!
        return await client.GetFromJsonAsync<Product>($"api/products/{id}");
    }
}

// ✅ IHttpClientFactory
public class FastApiClient
{
    private readonly HttpClient _http;
    
    public FastApiClient(HttpClient http) => _http = http;
    
    public async Task<Product?> GetProductAsync(int id)
        => await _http.GetFromJsonAsync<Product>($"api/products/{id}");
}

// Register in DI
// builder.Services.AddHttpClient<FastApiClient>(c =>
// {
//     c.BaseAddress = new Uri("https://api.myapp.com/");
//     c.Timeout = TimeSpan.FromSeconds(30);
//     c.DefaultRequestHeaders.Add("Accept", "application/json");
// }).AddStandardResilienceHandler(); // .NET 9 Polly integration

// Response compression
public class CompressedApiClient
{
    private static readonly HttpClient _http = new(
        new HttpClientHandler
        {
            AutomaticDecompression = System.Net.DecompressionMethods.All
        });
    
    // Cache GET responses
    private readonly MemoryCache<string, string> _cache = new();
    
    public async Task<T?> GetCachedAsync<T>(string url, TimeSpan ttl)
    {
        var cacheKey = url;
        if (_cache.TryGet(cacheKey, out string? json))
            return System.Text.Json.JsonSerializer.Deserialize<T>(json!);
        
        json = await _http.GetStringAsync(url);
        _cache.Set(cacheKey, json, ttl);
        
        return System.Text.Json.JsonSerializer.Deserialize<T>(json);
    }
}

// Parallel requests
public class DashboardService
{
    private readonly HttpClient _http;
    
    public DashboardService(HttpClient http) => _http = http;
    
    public async Task<DashboardData> LoadAsync()
    {
        // ❌ Sequential (slow)
        // var products = await GetProductsAsync();
        // var orders = await GetOrdersAsync();
        // var stats = await GetStatsAsync();
        
        // ✅ Parallel
        var (products, orders, stats) = await (
            GetProductsAsync(),
            GetOrdersAsync(),
            GetStatsAsync()
        ).WhenAll();
        
        return new DashboardData(products, orders, stats);
    }
    
    private Task<List<Product>> GetProductsAsync() =>
        _http.GetFromJsonAsync<List<Product>>("api/products")!;
    private Task<List<Order>> GetOrdersAsync() =>
        _http.GetFromJsonAsync<List<Order>>("api/orders")!;
    private Task<DashboardStats> GetStatsAsync() =>
        _http.GetFromJsonAsync<DashboardStats>("api/stats")!;
}

// Extension method for parallel tuple
public static class TaskExtensions
{
    public static async Task<(T1, T2, T3)> WhenAll<T1, T2, T3>(
        this (Task<T1>, Task<T2>, Task<T3>) tasks)
    {
        await Task.WhenAll(tasks.Item1, tasks.Item2, tasks.Item3);
        return (tasks.Item1.Result, tasks.Item2.Result, tasks.Item3.Result);
    }
}
```

---

## Step 489: Profiling Tools

```bash
# ============================================
# Performance Profiling
# ============================================

# dotnet-trace - CPU profiling
dotnet tool install -g dotnet-trace
dotnet-trace collect --process-id <PID> --duration 00:00:30

# dotnet-counters - real-time monitoring
dotnet tool install -g dotnet-counters
dotnet-counters monitor --process-id <PID> \
    System.Runtime \
    Microsoft.AspNetCore.Hosting

# dotnet-dump - memory analysis
dotnet tool install -g dotnet-dump
dotnet-dump collect --process-id <PID>
dotnet-dump analyze ./core_<PID>_<date>.dmp

# Key performance counters to monitor:
# - cpu-usage: CPU usage (%)
# - working-set: Memory in use (MB)
# - gc-heap-size: GC heap size
# - gen-0-gc-count: Gen0 GC count per minute
# - gen-1-gc-count: Gen1 GC count per minute
# - gen-2-gc-count: Gen2 GC count (most expensive!)
# - threadpool-queue-length: Thread starvation indicator
# - active-timer-count: Timer leaks
```

```csharp
// Custom performance monitoring
public class PerformanceMonitor
{
    private static readonly System.Diagnostics.Metrics.Meter _meter = 
        new("MyApp.Performance");
    
    private static readonly System.Diagnostics.Metrics.Histogram<double> _dbQueryTime =
        _meter.CreateHistogram<double>("db.query.duration", "ms");
    
    private static readonly System.Diagnostics.Metrics.Counter<long> _apiCalls =
        _meter.CreateCounter<long>("api.calls");
    
    public async Task<T> TrackQueryAsync<T>(string queryName, Func<Task<T>> query)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var result = await query();
            sw.Stop();
            _dbQueryTime.Record(sw.Elapsed.TotalMilliseconds,
                new System.Diagnostics.TagList { { "query", queryName } });
            return result;
        }
        catch
        {
            sw.Stop();
            _dbQueryTime.Record(sw.Elapsed.TotalMilliseconds,
                new System.Diagnostics.TagList { { "query", queryName }, { "error", true } });
            throw;
        }
    }
}
```

---

## Step 490: Benchmark-Driven Optimization

```csharp
// ============================================
// BenchmarkDotNet
// ============================================

// dotnet add package BenchmarkDotNet

using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(iterationCount: 10)]
public class StringParsingBenchmarks
{
    private const string CsvLine = "สินค้า A,299.99,50,Electronics";
    
    [Benchmark(Baseline = true)]
    public string[] ParseWithSplit() => CsvLine.Split(',');
    
    [Benchmark]
    public List<string> ParseWithSpan()
    {
        var result = new List<string>(4);
        var span = CsvLine.AsSpan();
        int start = 0;
        
        for (int i = 0; i <= span.Length; i++)
        {
            if (i == span.Length || span[i] == ',')
            {
                result.Add(new string(span[start..i]));
                start = i + 1;
            }
        }
        
        return result;
    }
    
    [Benchmark]
    public int CountFields()
    {
        var span = CsvLine.AsSpan();
        int count = 1;
        foreach (var c in span)
            if (c == ',') count++;
        return count;
    }
}

// Run benchmarks
class Program
{
    static void Main(string[] args)
    {
        var summary = BenchmarkRunner.Run<StringParsingBenchmarks>();
    }
}

/*
 * Example output:
 * | Method         | Mean      | Alloc  |
 * |--------------- |----------:|-------:|
 * | ParseWithSplit | 450.3 ns  | 288 B  |
 * | ParseWithSpan  | 120.1 ns  | 176 B  |
 * | CountFields    |   8.2 ns  |   0 B  |
 */
```

---

## สรุป Part 49

ใน Part 49 เราได้เรียนรู้:

1. **Memory Management** - WeakReference, ArrayPool, CompositeDisposable
2. **Span<T>** - Zero-allocation parsing, stackalloc
3. **CollectionView** - Virtualization, infinite scroll
4. **Image Caching** - Per-URL locking, hash-based cache
5. **Startup Performance** - Lazy init, background warmup
6. **Rendering** - SkiaSharp picture caching, Grid over StackLayout
7. **Database** - N+1 fix, JOIN, indexes, keyset pagination
8. **HTTP** - IHttpClientFactory, compression, parallel requests
9. **Profiling Tools** - dotnet-trace, dotnet-counters, metrics
10. **BenchmarkDotNet** - Measure before optimizing

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 49 | Steps 481-490*
