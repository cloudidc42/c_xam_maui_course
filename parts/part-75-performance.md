# Part 75: Advanced Performance Optimization
## Steps 741-750: Memory Management, Rendering, Profiling, Startup

---

## Step 741: Memory Profiling & Leak Detection

```csharp
// ============================================
// Memory Management
// ============================================

// WeakReference to avoid retaining ViewModels
public class WeakViewModelCache
{
    private readonly Dictionary<string, WeakReference<IViewModel>> _cache = new();
    
    public void Store(string key, IViewModel vm)
        => _cache[key] = new WeakReference<IViewModel>(vm);
    
    public IViewModel? Get(string key)
    {
        if (_cache.TryGetValue(key, out var weak)
            && weak.TryGetTarget(out var vm))
            return vm;
        _cache.Remove(key);
        return null;
    }
    
    public void Purge()
    {
        var dead = _cache.Where(kv => !kv.Value.TryGetTarget(out _))
                         .Select(kv => kv.Key).ToList();
        foreach (var key in dead) _cache.Remove(key);
    }
}

// Leak-safe ViewModel: unsubscribe events in IDisposable
public class LeakSafeViewModel : ObservableObject, IDisposable
{
    private readonly IConnectivity _connectivity;
    private bool _disposed;
    
    public LeakSafeViewModel(IConnectivity connectivity)
    {
        _connectivity = connectivity;
        _connectivity.ConnectivityChanged += OnConnectivityChanged;
    }
    
    private void OnConnectivityChanged(object? s, ConnectivityChangedEventArgs e) { }
    
    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _connectivity.ConnectivityChanged -= OnConnectivityChanged;
        WeakReferenceMessenger.Default.UnregisterAll(this);
    }
}

// Object pooling for frequently created objects
public class MenuItemPool
{
    private readonly ConcurrentQueue<MenuItemViewModel> _pool = new();
    private const int MaxPoolSize = 50;
    
    public MenuItemViewModel Rent()
    {
        if (_pool.TryDequeue(out var vm))
        {
            vm.Reset();
            return vm;
        }
        return new MenuItemViewModel();
    }
    
    public void Return(MenuItemViewModel vm)
    {
        if (_pool.Count < MaxPoolSize)
            _pool.Enqueue(vm);
    }
}
```

---

## Step 742: Image Optimization

```csharp
// ============================================
// Smart Image Loading
// ============================================

public class OptimizedImageCache
{
    private readonly LruCache<string, byte[]> _memoryCache;
    private readonly string _diskCachePath;
    
    public OptimizedImageCache(int maxMemoryMb = 50)
    {
        var maxBytes = maxMemoryMb * 1024 * 1024;
        _memoryCache = new LruCache<string, byte[]>(maxBytes,
            sizeOf: (_, v) => v.Length);
        _diskCachePath = Path.Combine(FileSystem.CacheDirectory, "images");
        Directory.CreateDirectory(_diskCachePath);
    }
    
    public async Task<byte[]?> GetAsync(string url)
    {
        // L1: memory
        if (_memoryCache.TryGet(url, out var mem)) return mem;
        
        // L2: disk
        var diskPath = GetDiskPath(url);
        if (File.Exists(diskPath))
        {
            var bytes = await File.ReadAllBytesAsync(diskPath);
            _memoryCache.Put(url, bytes);
            return bytes;
        }
        
        return null;
    }
    
    public async Task PutAsync(string url, byte[] data)
    {
        _memoryCache.Put(url, data);
        var diskPath = GetDiskPath(url);
        await File.WriteAllBytesAsync(diskPath, data);
    }
    
    private string GetDiskPath(string url)
    {
        var hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(url)));
        return Path.Combine(_diskCachePath, hash + ".jpg");
    }
    
    public long GetDiskCacheSizeBytes()
        => new DirectoryInfo(_diskCachePath)
            .GetFiles().Sum(f => f.Length);
    
    public void ClearDiskCache()
    {
        foreach (var file in new DirectoryInfo(_diskCachePath).GetFiles())
            file.Delete();
    }
}

// Simple LRU Cache
public class LruCache<TKey, TValue> where TKey : notnull
{
    private readonly int _maxSize;
    private readonly Func<TKey, TValue, long> _sizeOf;
    private readonly LinkedList<(TKey Key, TValue Value, long Size)> _list = new();
    private readonly Dictionary<TKey, LinkedListNode<(TKey, TValue, long)>> _map = new();
    private long _currentSize;
    
    public LruCache(int maxSize, Func<TKey, TValue, long> sizeOf)
    {
        _maxSize = maxSize;
        _sizeOf = sizeOf;
    }
    
    public bool TryGet(TKey key, out TValue value)
    {
        if (_map.TryGetValue(key, out var node))
        {
            _list.Remove(node);
            _list.AddFirst(node);
            value = node.Value.Value;
            return true;
        }
        value = default!;
        return false;
    }
    
    public void Put(TKey key, TValue value)
    {
        var size = _sizeOf(key, value);
        if (_map.TryGetValue(key, out var existing))
        {
            _currentSize -= existing.Value.Size;
            _list.Remove(existing);
        }
        
        var node = _list.AddFirst((key, value, size));
        _map[key] = node;
        _currentSize += size;
        
        while (_currentSize > _maxSize && _list.Count > 0)
        {
            var last = _list.Last!;
            _list.RemoveLast();
            _map.Remove(last.Value.Key);
            _currentSize -= last.Value.Size;
        }
    }
}
```

---

## Step 743: CollectionView Performance

```csharp
// ============================================
// High-Performance CollectionView
// ============================================

// RecycleElement is default; use this for very large lists
// <CollectionView ItemsUpdatingScrollMode="KeepItemsInView" />

public class VirtualizedMenuViewModel : ObservableObject
{
    [ObservableProperty] private ObservableRangeCollection<MenuItemVm> _items = new();
    
    private const int PageSize = 20;
    private int _currentPage;
    private bool _isLoadingMore;
    
    public async Task LoadInitialAsync()
    {
        _currentPage = 0;
        var page = await LoadPageAsync(0);
        Items = new ObservableRangeCollection<MenuItemVm>(page);
        _currentPage = 1;
    }
    
    public async Task LoadMoreAsync()
    {
        if (_isLoadingMore) return;
        _isLoadingMore = true;
        
        try
        {
            var page = await LoadPageAsync(_currentPage);
            if (page.Count > 0)
            {
                Items.AddRange(page); // Single notification
                _currentPage++;
            }
        }
        finally
        {
            _isLoadingMore = false;
        }
    }
    
    private Task<List<MenuItemVm>> LoadPageAsync(int page)
        => Task.FromResult(new List<MenuItemVm>()); // from repo
}

// ObservableRangeCollection: batches notifications
public class ObservableRangeCollection<T> : ObservableCollection<T>
{
    public ObservableRangeCollection() { }
    public ObservableRangeCollection(IEnumerable<T> items) : base(items) { }
    
    public void AddRange(IEnumerable<T> items)
    {
        var list = items.ToList();
        foreach (var item in list)
            Items.Add(item);
        
        OnPropertyChanged(new PropertyChangedEventArgs(nameof(Count)));
        OnCollectionChanged(new NotifyCollectionChangedEventArgs(
            NotifyCollectionChangedAction.Add, list, Items.Count - list.Count));
    }
    
    public void ReplaceAll(IEnumerable<T> items)
    {
        Items.Clear();
        foreach (var item in items) Items.Add(item);
        OnCollectionChanged(new NotifyCollectionChangedEventArgs(
            NotifyCollectionChangedAction.Reset));
    }
}
```

---

## Step 744: Startup Performance

```csharp
// ============================================
// App Startup Optimization
// ============================================

public class OptimizedAppStartup
{
    private readonly IServiceProvider _services;
    
    public OptimizedAppStartup(IServiceProvider services)
    {
        _services = services;
    }
    
    public async Task<StartupResult> RunAsync()
    {
        using var _ = StartupTimer.Measure("total_startup");
        
        // Critical path: must complete before showing UI
        await RunCriticalAsync();
        
        // Deferred: can run after UI shows
        _ = Task.Run(RunDeferredAsync);
        
        return StartupResult.Success;
    }
    
    private async Task RunCriticalAsync()
    {
        using var _ = StartupTimer.Measure("critical_path");
        
        // 1. DB migration (~20ms)
        await _services.GetRequiredService<DatabaseMigrator>().MigrateAsync();
        
        // 2. Load auth token (~5ms)
        await _services.GetRequiredService<ITokenService>().LoadAsync();
        
        // 3. Feature flags from cache (~2ms)
        await _services.GetRequiredService<IFeatureFlagService>().LoadCachedAsync();
    }
    
    private async Task RunDeferredAsync()
    {
        // These run after UI is showing
        await Task.WhenAll(
            _services.GetRequiredService<SyncEngine>().SyncAsync(),
            _services.GetRequiredService<BackgroundSyncService>().StartLazyAsync(),
            _services.GetRequiredService<RecommendationService>().WarmUpCacheAsync()
        );
    }
}

public static class StartupTimer
{
    private static readonly List<(string Name, long Ms)> _records = new();
    
    public static IDisposable Measure(string name)
    {
        var sw = Stopwatch.StartNew();
        return new Disposable(() =>
        {
            sw.Stop();
            _records.Add((name, sw.ElapsedMilliseconds));
        });
    }
    
    public static string Report()
        => string.Join("\n", _records.Select(r => $"{r.Name}: {r.Ms}ms"));
}

public class Disposable : IDisposable
{
    private readonly Action _action;
    public Disposable(Action action) => _action = action;
    public void Dispose() => _action();
}
```

---

## Step 745: Rendering Performance

```csharp
// ============================================
// Reducing Layout Passes
// ============================================

// GOOD: Single-pass Grid
// BAD: Nested StackLayouts causing multiple measure passes

// Measure cache to avoid redundant layout calculations
public static class LayoutOptimizer
{
    // Use cached size for items that don't change dimensions
    public static Size GetCachedSize(string key, Func<Size> compute)
    {
        if (SizeCache.TryGetValue(key, out var cached)) return cached;
        var size = compute();
        SizeCache[key] = size;
        return size;
    }
    
    private static readonly Dictionary<string, Size> SizeCache = new();
    
    // Profile layout inflation
    public static async Task<T> MeasureInflationAsync<T>(
        string name, Func<Task<T>> factory)
    {
        var sw = Stopwatch.StartNew();
        var result = await factory();
        sw.Stop();
        
        if (sw.ElapsedMilliseconds > 16)
            Debug.WriteLine($"[PERF] {name} took {sw.ElapsedMilliseconds}ms (>16ms threshold)");
        
        return result;
    }
}

// Avoid overdraw: transparent backgrounds draw on top of each other
// XAML: don't set BackgroundColor="Transparent" unless needed
// Use ClipsToBounds wisely (clips children but slows rendering)

public class PerformanceMonitor
{
    private static readonly Dictionary<string, List<long>> _samples = new();
    
    public static IDisposable Track(string operation)
    {
        var sw = Stopwatch.StartNew();
        return new Disposable(() =>
        {
            sw.Stop();
            if (!_samples.ContainsKey(operation))
                _samples[operation] = new List<long>();
            _samples[operation].Add(sw.ElapsedMilliseconds);
        });
    }
    
    public static PerformanceReport GenerateReport()
    {
        var stats = _samples.ToDictionary(
            kv => kv.Key,
            kv => new OperationStats(
                Count: kv.Value.Count,
                Min: kv.Value.Min(),
                Max: kv.Value.Max(),
                Avg: kv.Value.Average(),
                P95: Percentile(kv.Value, 0.95)));
        
        return new PerformanceReport(stats);
    }
    
    private static long Percentile(List<long> sorted, double p)
    {
        var list = sorted.OrderBy(x => x).ToList();
        var idx = (int)Math.Ceiling(p * list.Count) - 1;
        return list[Math.Clamp(idx, 0, list.Count - 1)];
    }
}

public record OperationStats(int Count, long Min, long Max, double Avg, long P95);
public record PerformanceReport(Dictionary<string, OperationStats> Stats);
```

---

## Step 746: Async Best Practices

```csharp
// ============================================
// Async Performance Patterns
// ============================================

public class AsyncBestPractices
{
    // ValueTask for hot paths (avoids heap allocation when result is synchronous)
    public async ValueTask<MenuItem?> GetCachedItemAsync(string id)
    {
        if (_cache.TryGetValue(id, out var item)) return item; // synchronous path
        return await _repository.GetByIdAsync(id);             // async path
    }
    private readonly Dictionary<string, MenuItem> _cache = new();
    private readonly IMenuRepository _repository = null!;
    
    // ConfigureAwait(false) in library code (not UI)
    public async Task<IReadOnlyList<MenuItem>> LoadMenuAsync(string restaurantId)
    {
        var data = await _repository.GetMenuAsync(restaurantId).ConfigureAwait(false);
        return data.Where(x => x.IsAvailable).ToList().AsReadOnly();
    }
    
    // Parallel fetches with WhenAll
    public async Task<(Restaurant, IReadOnlyList<MenuItem>)> LoadRestaurantPageAsync(
        string restaurantId)
    {
        var restaurantTask = _repository.GetRestaurantAsync(restaurantId);
        var menuTask = LoadMenuAsync(restaurantId);
        
        await Task.WhenAll(restaurantTask, menuTask);
        
        return (await restaurantTask, await menuTask);
    }
    
    // CancellationToken propagation
    public async Task<IReadOnlyList<MenuItem>> SearchAsync(
        string query, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        var results = await _repository.SearchAsync(query, ct).ConfigureAwait(false);
        ct.ThrowIfCancellationRequested();
        return results;
    }
    
    // Semaphore for controlled concurrency
    private readonly SemaphoreSlim _uploadSemaphore = new(3); // max 3 concurrent uploads
    
    public async Task UploadImagesAsync(IEnumerable<string> paths)
    {
        await Task.WhenAll(paths.Select(async path =>
        {
            await _uploadSemaphore.WaitAsync();
            try { await UploadSingleAsync(path); }
            finally { _uploadSemaphore.Release(); }
        }));
    }
    
    private Task UploadSingleAsync(string path) => Task.CompletedTask;
}
```

---

## Step 747: Database Query Optimization

```csharp
// ============================================
// SQLite Query Optimization
// ============================================

public class OptimizedOrderRepository
{
    private readonly SQLiteAsyncConnection _db;
    
    public OptimizedOrderRepository(SQLiteAsyncConnection db)
    {
        _db = db;
    }
    
    // Use indices for frequent queries
    public async Task CreateIndicesAsync()
    {
        await _db.ExecuteAsync(
            "CREATE INDEX IF NOT EXISTS idx_orders_user ON Orders(UserId, CreatedAt DESC)");
        await _db.ExecuteAsync(
            "CREATE INDEX IF NOT EXISTS idx_orders_status ON Orders(Status, CreatedAt)");
        await _db.ExecuteAsync(
            "CREATE INDEX IF NOT EXISTS idx_items_order ON OrderItems(OrderId)");
    }
    
    // Paged query: avoid loading everything
    public async Task<PagedResult<Order>> GetOrdersPagedAsync(
        string userId, int page, int size = 20)
    {
        var offset = page * size;
        var countQuery = "SELECT COUNT(*) FROM Orders WHERE UserId = ?";
        var dataQuery = @"
            SELECT * FROM Orders 
            WHERE UserId = ? 
            ORDER BY CreatedAt DESC 
            LIMIT ? OFFSET ?";
        
        var total = await _db.ExecuteScalarAsync<int>(countQuery, userId);
        var items = await _db.QueryAsync<Order>(dataQuery, userId, size, offset);
        
        return new PagedResult<Order>(items, total, page, size);
    }
    
    // Batch inserts in transaction
    public async Task InsertOrderItemsBatchAsync(IReadOnlyList<OrderItem> items)
    {
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var item in items)
                conn.Insert(item);
        });
    }
    
    // Projection: only select needed columns
    public async Task<IReadOnlyList<OrderSummary>> GetOrderSummariesAsync(string userId)
    {
        return await _db.QueryAsync<OrderSummary>(
            "SELECT Id, Total, Status, CreatedAt FROM Orders WHERE UserId = ? ORDER BY CreatedAt DESC LIMIT 50",
            userId);
    }
    
    // Compiled query pattern (cache query objects)
    private static readonly TableQuery<Order>? _compiledQuery;
    
    public async Task<IReadOnlyList<Order>> GetActiveOrdersAsync(string userId)
    {
        return await _db.Table<Order>()
            .Where(o => o.UserId == userId && o.Status == "active")
            .ToListAsync();
    }
}

public record PagedResult<T>(
    IReadOnlyList<T> Items, int Total, int Page, int Size)
{
    public bool HasMore => (Page + 1) * Size < Total;
    public int TotalPages => (int)Math.Ceiling((double)Total / Size);
}

public class OrderSummary
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}
```

---

## Step 748: Network Performance

```csharp
// ============================================
// HTTP Performance Optimizations
// ============================================

public class PerformantHttpClient
{
    private readonly HttpClient _client;
    
    public PerformantHttpClient()
    {
        var handler = new SocketsHttpHandler
        {
            PooledConnectionLifetime = TimeSpan.FromMinutes(5),
            PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
            MaxConnectionsPerServer = 10,
            EnableMultipleHttp2Connections = true,
        };
        
        _client = new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(30),
            DefaultRequestHeaders =
            {
                { "Accept-Encoding", "gzip, br" },
                { "Accept", "application/json" }
            }
        };
    }
    
    // With compression and caching headers
    public async Task<T?> GetWithCacheAsync<T>(
        string url, string? etag = null, CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, url);
        
        if (etag != null)
            request.Headers.IfNoneMatch.ParseAdd(etag);
        
        var response = await _client.SendAsync(request, ct);
        
        if (response.StatusCode == System.Net.HttpStatusCode.NotModified)
            return default; // Use cached version
        
        response.EnsureSuccessStatusCode();
        
        // Auto-decompress gzip
        var content = await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
        return content;
    }
}

// Request batching: combine multiple API calls
public class RequestBatcher
{
    private readonly HttpClient _client;
    private readonly List<BatchRequest> _queue = new();
    private readonly SemaphoreSlim _lock = new(1, 1);
    private Timer? _timer;
    
    public RequestBatcher(HttpClient client)
    {
        _client = client;
    }
    
    public async Task<T?> EnqueueAsync<T>(string endpoint, CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<object?>();
        
        await _lock.WaitAsync(ct);
        try
        {
            _queue.Add(new BatchRequest(endpoint, tcs));
            if (_queue.Count == 1)
                _timer = new Timer(_ => _ = FlushAsync(), null, 50, Timeout.Infinite);
        }
        finally
        {
            _lock.Release();
        }
        
        var result = await tcs.Task;
        return result is T typed ? typed : default;
    }
    
    private async Task FlushAsync()
    {
        List<BatchRequest> batch;
        await _lock.WaitAsync();
        try
        {
            batch = new List<BatchRequest>(_queue);
            _queue.Clear();
        }
        finally { _lock.Release(); }
        
        // Send as single batch request
        var endpoints = batch.Select(b => b.Endpoint).ToArray();
        // var results = await _client.PostAsJsonAsync("/batch", endpoints);
        // Distribute results back to each TaskCompletionSource
        foreach (var req in batch)
            req.Completion.SetResult(null);
    }
}

public record BatchRequest(string Endpoint, TaskCompletionSource<object?> Completion);
```

---

## Step 749: Profiling Tools Setup

```csharp
// ============================================
// In-App Performance Dashboard
// ============================================

#if DEBUG
public class PerformanceDashboard : ContentPage
{
    public PerformanceDashboard()
    {
        Title = "Performance";
        
        var report = PerformanceMonitor.GenerateReport();
        
        Content = new ScrollView
        {
            Content = new VerticalStackLayout
            {
                Padding = 16, Spacing = 8,
                Children =
                {
                    new Label { Text = "Performance Report", FontSize = 20, FontAttributes = Bold },
                    new Label { Text = $"Generated: {DateTime.Now:HH:mm:ss}", FontSize = 12 },
                    BuildStatsView(report)
                }
            }
        };
    }
    
    private View BuildStatsView(PerformanceReport report)
    {
        var grid = new Grid
        {
            ColumnDefinitions = Columns.Define(Star, Star, Star, Star),
            RowDefinitions = Rows.Define(Auto),
            Padding = new Thickness(0, 8)
        };
        
        // Headers
        grid.Add(new Label { Text = "Operation", FontAttributes = Bold }, 0, 0);
        grid.Add(new Label { Text = "Avg", FontAttributes = Bold }, 1, 0);
        grid.Add(new Label { Text = "P95", FontAttributes = Bold }, 2, 0);
        grid.Add(new Label { Text = "Count", FontAttributes = Bold }, 3, 0);
        
        int row = 1;
        foreach (var (name, stats) in report.Stats.OrderByDescending(s => s.Value.P95))
        {
            var rowDef = new RowDefinition { Height = GridLength.Auto };
            grid.RowDefinitions.Add(rowDef);
            
            var color = stats.P95 > 200 ? Colors.Red :
                        stats.P95 > 50 ? Colors.Orange : Colors.Green;
            
            grid.Add(new Label { Text = name, FontSize = 11 }, 0, row);
            grid.Add(new Label { Text = $"{stats.Avg:F0}ms", TextColor = color, FontSize = 11 }, 1, row);
            grid.Add(new Label { Text = $"{stats.P95}ms", FontSize = 11 }, 2, row);
            grid.Add(new Label { Text = stats.Count.ToString(), FontSize = 11 }, 3, row);
            row++;
        }
        
        return grid;
    }
}
#endif
```

---

## Step 750: Performance Test Benchmarks

```csharp
// ============================================
// Micro-benchmarks with BenchmarkDotNet
// ============================================

// Install: BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class JsonSerializationBenchmark
{
    private readonly List<MenuItem> _items = Enumerable.Range(1, 100)
        .Select(i => new MenuItem { Id = i.ToString(), Name = $"Item {i}", Price = i * 10 })
        .ToList();
    
    [Benchmark(Baseline = true)]
    public string Newtonsoft()
        => Newtonsoft.Json.JsonConvert.SerializeObject(_items);
    
    [Benchmark]
    public string SystemTextJson()
        => System.Text.Json.JsonSerializer.Serialize(_items);
    
    [Benchmark]
    public string SystemTextJsonSource()
        => System.Text.Json.JsonSerializer.Serialize(_items, MenuItemContext.Default.ListMenuItem);
}

[System.Text.Json.Serialization.JsonSerializable(typeof(List<MenuItem>))]
public partial class MenuItemContext : System.Text.Json.Serialization.JsonSerializerContext { }

[MemoryDiagnoser]
public class CollectionBenchmark
{
    private readonly List<int> _list = Enumerable.Range(0, 10_000).ToList();
    
    [Benchmark]
    public int LinqSum() => _list.Sum();
    
    [Benchmark]
    public int ForSum()
    {
        var sum = 0;
        foreach (var x in _list) sum += x;
        return sum;
    }
    
    [Benchmark]
    public List<int> LinqFilter()
        => _list.Where(x => x % 2 == 0).ToList();
    
    [Benchmark]
    public List<int> ForFilter()
    {
        var result = new List<int>(_list.Count / 2);
        foreach (var x in _list)
            if (x % 2 == 0) result.Add(x);
        return result;
    }
}

// Performance acceptance tests
[TestFixture]
public class PerformanceAcceptanceTests
{
    [Test]
    public async Task HomePageLoad_ShouldBeFasterThan2Seconds()
    {
        var vm = new HomeViewModel(/* inject mocks */);
        var sw = Stopwatch.StartNew();
        
        await vm.InitializeAsync();
        
        sw.Stop();
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(2000),
            "Home page load exceeded 2 second budget");
    }
    
    [Test]
    public async Task Search_HundredItems_ShouldBeFasterThan100Ms()
    {
        var engine = new ThaiTextSearchEngine();
        var items = Enumerable.Range(0, 100)
            .Select(i => new MenuItem { Id = i.ToString(), Name = $"ข้าวผัด {i}", Price = 80 })
            .ToList();
        engine.IndexMenuItems(items);
        
        var sw = Stopwatch.StartNew();
        var results = engine.Search("ข้าวผัด");
        sw.Stop();
        
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(100));
        Assert.That(results, Is.Not.Empty);
    }
}
```

---

## สรุป Part 75

ใน Part 75 เราได้เรียนรู้:

1. **Memory Management** - WeakReference cache, dispose pattern, object pooling
2. **Image Optimization** - LRU memory+disk cache, SHA-256 based filenames
3. **CollectionView Performance** - ObservableRangeCollection batch notifications, pagination
4. **Startup Optimization** - Critical vs deferred path, StartupTimer instrumentation
5. **Rendering Performance** - Measure cache, overdraw avoidance, layout profiling
6. **Async Best Practices** - ValueTask, ConfigureAwait, WhenAll, CancellationToken
7. **Database Optimization** - Indices, paged queries, batch inserts, projections
8. **Network Performance** - Connection pooling, ETags, HTTP/2, request batching
9. **Profiling Dashboard** - In-app debug performance viewer with P95 highlighting
10. **Performance Benchmarks** - BenchmarkDotNet, acceptance test budget checks

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 75 | Steps 741-750*
