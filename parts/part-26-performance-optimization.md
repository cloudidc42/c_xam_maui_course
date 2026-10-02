# Part 26: Performance Optimization
## Steps 251-260: เพิ่มประสิทธิภาพแอพ

---

## Step 251: Memory Management

```csharp
// ============================================
// Memory Management ใน .NET
// ============================================

/*
 * .NET Memory Structure:
 * 
 * Stack - local variables, value types, references
 * Heap  - objects (class instances)
 * 
 * GC Generations:
 * Gen 0 - short-lived objects
 * Gen 1 - medium-lived objects  
 * Gen 2 - long-lived objects (singletons, static)
 * LOH   - Large Object Heap (>85KB)
 */

// ============================================
// Span<T> - Zero-allocation operations
// ============================================

public static class HighPerformanceParsing
{
    // Without Span - allocates substrings
    public static string[] SplitSlowWay(string input, char delimiter)
        => input.Split(delimiter);
    
    // With Span - zero allocation
    public static List<string> SplitFastWay(ReadOnlySpan<char> input, char delimiter)
    {
        var result = new List<string>();
        int start = 0;
        
        for (int i = 0; i < input.Length; i++)
        {
            if (input[i] == delimiter)
            {
                result.Add(input[start..i].ToString());
                start = i + 1;
            }
        }
        
        if (start < input.Length)
            result.Add(input[start..].ToString());
        
        return result;
    }
    
    // Parse without allocation
    public static bool TryParseIntFromSpan(ReadOnlySpan<char> span, out int value)
        => int.TryParse(span, out value);
    
    // String slicing with Span
    public static ReadOnlySpan<char> ExtractBetween(
        ReadOnlySpan<char> source, char open, char close)
    {
        int start = source.IndexOf(open) + 1;
        int end = source.LastIndexOf(close);
        return start < end ? source[start..end] : ReadOnlySpan<char>.Empty;
    }
}

// ============================================
// ArrayPool - Reuse arrays
// ============================================

using System.Buffers;

public class BufferExample
{
    public static async Task ProcessLargeFileAsync(string path)
    {
        // Rent buffer instead of allocating
        var buffer = ArrayPool<byte>.Shared.Rent(8192);
        
        try
        {
            using var stream = File.OpenRead(path);
            int bytesRead;
            
            while ((bytesRead = await stream.ReadAsync(buffer)) > 0)
            {
                ProcessChunk(buffer.AsSpan(0, bytesRead));
            }
        }
        finally
        {
            // Must return to pool
            ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
        }
    }
    
    private static void ProcessChunk(Span<byte> chunk) { /* process */ }
}
```

---

## Step 252: String Performance

```csharp
// ============================================
// String Performance
// ============================================

using System.Text;

public class StringPerformance
{
    // BAD: String concatenation in loop
    public static string BuildBadWay(IEnumerable<string> items)
    {
        string result = "";
        foreach (var item in items)
            result += item + ", "; // Creates new string each time
        return result;
    }
    
    // GOOD: StringBuilder
    public static string BuildGoodWay(IEnumerable<string> items)
    {
        var sb = new StringBuilder();
        foreach (var item in items)
            sb.Append(item).Append(", ");
        
        if (sb.Length > 2)
            sb.Length -= 2; // Remove trailing ", "
        
        return sb.ToString();
    }
    
    // BEST: string.Join
    public static string BuildBestWay(IEnumerable<string> items)
        => string.Join(", ", items);
    
    // String interning
    public static void DemonstrateInterning()
    {
        string s1 = "hello"; // Interned by compiler
        string s2 = "hello"; // Same reference
        string s3 = new string(new[] { 'h', 'e', 'l', 'l', 'o' }); // Not interned
        string s4 = string.Intern(s3); // Force intern
        
        Console.WriteLine(ReferenceEquals(s1, s2)); // True
        Console.WriteLine(ReferenceEquals(s1, s3)); // False
        Console.WriteLine(ReferenceEquals(s1, s4)); // True
    }
    
    // String.Create - zero copy from chars
    public static string FormatProductCodeFast(int categoryId, int productId)
    {
        return string.Create(10, (categoryId, productId), (chars, state) =>
        {
            chars[0] = 'P';
            chars[1] = 'R';
            chars[2] = 'D';
            state.categoryId.TryFormat(chars[3..6], out _);
            chars[6] = '-';
            state.productId.TryFormat(chars[7..], out _);
        });
    }
}
```

---

## Step 253: LINQ Performance

```csharp
// ============================================
// LINQ Performance Tips
// ============================================

public class LinqPerformance
{
    private static readonly List<ProductEntity> _products = 
        Enumerable.Range(1, 10000)
            .Select(i => new ProductEntity 
            { 
                Id = i, 
                Name = $"Product {i}", 
                Price = i * 10m, 
                Stock = i % 100,
                IsActive = i % 3 != 0
            })
            .ToList();
    
    // BAD: Count > 0 is slower than Any()
    public static bool HasActiveBad()
        => _products.Count(p => p.IsActive) > 0;
    
    // GOOD: Any() stops on first match
    public static bool HasActiveGood()
        => _products.Any(p => p.IsActive);
    
    // BAD: First() throws, FirstOrDefault is safer
    // BAD: ToList() before Where()
    public static List<ProductEntity> GetExpensiveBad(decimal minPrice)
        => _products.ToList().Where(p => p.Price >= minPrice).ToList();
    
    // GOOD: Deferred execution, no intermediate list
    public static List<ProductEntity> GetExpensiveGood(decimal minPrice)
        => _products.Where(p => p.Price >= minPrice).ToList();
    
    // BAD: Multiple enumeration
    public static void MultipleEnumerationBad(IEnumerable<ProductEntity> source)
    {
        int count = source.Count(); // First enumeration
        var first = source.First(); // Second enumeration
    }
    
    // GOOD: Materialize once
    public static void SingleEnumerationGood(IEnumerable<ProductEntity> source)
    {
        var list = source.ToList(); // Single enumeration
        int count = list.Count;
        var first = list[0];
    }
    
    // Use Dictionary for O(1) lookups
    public static void LookupPerformance()
    {
        var productList = _products;
        
        // BAD: O(n) lookup
        // var found = productList.FirstOrDefault(p => p.Id == 5000);
        
        // GOOD: O(1) lookup
        var productDict = productList.ToDictionary(p => p.Id);
        var found = productDict.GetValueOrDefault(5000);
    }
    
    // PLINQ for CPU-bound
    public static decimal CalculateTotalParallel()
        => _products.AsParallel()
            .Where(p => p.IsActive)
            .Sum(p => p.Price * p.Stock);
}
```

---

## Step 254: UI Performance

```csharp
// ============================================
// UI Performance in MAUI
// ============================================

// ============================================
// CollectionView Virtualization
// ============================================

/*
<CollectionView ItemsSource="{Binding Products}"
                ItemsUpdatingScrollMode="MaintainScrollOffset"
                RemainingItemsThreshold="5"
                RemainingItemsThresholdReachedCommand="{Binding LoadMoreCommand}">
    
    <!-- DataTemplate with x:DataType for compiled binding -->
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="vm:ProductItem">
            <Grid Padding="10">
                <!-- Compiled binding - faster than reflection-based -->
                <Label Text="{Binding Name}" />
                <Label Text="{Binding Price, StringFormat='฿{0:N0}'}" />
            </Grid>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
*/

// ============================================
// Compiled Bindings
// ============================================

// In XAML: x:DataType="vm:ProductItem" enables compiled bindings
// This is faster because it generates code at compile time
// instead of using reflection at runtime.

// ============================================
// Image Optimization
// ============================================

public class ImageOptimizationService
{
    public async Task<byte[]> OptimizeImageAsync(Stream imageStream, 
        int maxWidth = 800, int quality = 80)
    {
        // Using SkiaSharp for image processing
        // NuGet: SkiaSharp, SkiaSharp.Views.Maui.Controls
        
        /*
        var bitmap = SKBitmap.Decode(imageStream);
        
        // Resize if too large
        if (bitmap.Width > maxWidth)
        {
            double ratio = (double)maxWidth / bitmap.Width;
            int newHeight = (int)(bitmap.Height * ratio);
            bitmap = bitmap.Resize(new SKImageInfo(maxWidth, newHeight), SKFilterQuality.High);
        }
        
        // Encode with compression
        using var image = SKImage.FromBitmap(bitmap);
        var data = image.Encode(SKEncodedImageFormat.Jpeg, quality);
        return data.ToArray();
        */
        
        // Placeholder
        await Task.Delay(100);
        return Array.Empty<byte>();
    }
}

// ============================================
// Throttle/Debounce
// ============================================

public class Throttler
{
    private DateTime _lastExecution = DateTime.MinValue;
    private readonly TimeSpan _interval;
    
    public Throttler(TimeSpan interval) => _interval = interval;
    
    public bool CanExecute()
    {
        if (DateTime.UtcNow - _lastExecution >= _interval)
        {
            _lastExecution = DateTime.UtcNow;
            return true;
        }
        return false;
    }
}

public class Debouncer
{
    private CancellationTokenSource? _cts;
    private readonly TimeSpan _delay;
    
    public Debouncer(TimeSpan delay) => _delay = delay;
    
    public async Task DebounceAsync(Func<Task> action)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        
        try
        {
            await Task.Delay(_delay, _cts.Token);
            await action();
        }
        catch (OperationCanceledException)
        {
            // Debounced - another call came in
        }
    }
}
```

---

## Step 255: Caching

```csharp
// ============================================
// In-Memory Cache
// ============================================

public class MemoryCache<TKey, TValue> where TKey : notnull
{
    private record CacheEntry(TValue Value, DateTime ExpiresAt);
    
    private readonly Dictionary<TKey, CacheEntry> _cache = new();
    private readonly TimeSpan _defaultTtl;
    private readonly int _maxSize;
    
    public MemoryCache(TimeSpan? defaultTtl = null, int maxSize = 1000)
    {
        _defaultTtl = defaultTtl ?? TimeSpan.FromMinutes(5);
        _maxSize = maxSize;
    }
    
    public bool TryGet(TKey key, out TValue? value)
    {
        if (_cache.TryGetValue(key, out var entry))
        {
            if (DateTime.UtcNow < entry.ExpiresAt)
            {
                value = entry.Value;
                return true;
            }
            _cache.Remove(key);
        }
        value = default;
        return false;
    }
    
    public void Set(TKey key, TValue value, TimeSpan? ttl = null)
    {
        if (_cache.Count >= _maxSize)
            EvictOldest();
        
        _cache[key] = new CacheEntry(value, DateTime.UtcNow.Add(ttl ?? _defaultTtl));
    }
    
    public async Task<TValue> GetOrSetAsync(TKey key, Func<Task<TValue>> factory, TimeSpan? ttl = null)
    {
        if (TryGet(key, out var cached))
            return cached!;
        
        var value = await factory();
        Set(key, value, ttl);
        return value;
    }
    
    public void Remove(TKey key) => _cache.Remove(key);
    
    public void Clear() => _cache.Clear();
    
    private void EvictOldest()
    {
        var oldest = _cache.OrderBy(kv => kv.Value.ExpiresAt).First().Key;
        _cache.Remove(oldest);
    }
    
    public void RemoveExpired()
    {
        var expired = _cache.Where(kv => DateTime.UtcNow >= kv.Value.ExpiresAt)
                            .Select(kv => kv.Key)
                            .ToList();
        foreach (var key in expired)
            _cache.Remove(key);
    }
}

// Usage
public class CachedProductService
{
    private readonly IProductApiService _api;
    private readonly MemoryCache<string, List<ProductDto>> _cache;
    
    public CachedProductService(IProductApiService api)
    {
        _api = api;
        _cache = new MemoryCache<string, List<ProductDto>>(TimeSpan.FromMinutes(5));
    }
    
    public async Task<List<ProductDto>> GetProductsAsync(string? category = null)
    {
        var cacheKey = $"products:{category ?? "all"}";
        
        return await _cache.GetOrSetAsync(cacheKey, async () =>
        {
            var result = await _api.GetProductsAsync(category: category);
            return result.IsSuccess ? result.Data!.Items : new List<ProductDto>();
        });
    }
    
    public void InvalidateProducts()
        => _cache.Clear();
}
```

---

## Step 256: Background Tasks

```csharp
// ============================================
// Background Task Management
// ============================================

public interface IBackgroundTaskService
{
    Task RunAsync(string taskId, Func<CancellationToken, Task> work, 
        TimeSpan? timeout = null);
    void Cancel(string taskId);
    void CancelAll();
}

public class BackgroundTaskService : IBackgroundTaskService, IDisposable
{
    private readonly Dictionary<string, CancellationTokenSource> _tasks = new();
    
    public async Task RunAsync(string taskId, Func<CancellationToken, Task> work, 
        TimeSpan? timeout = null)
    {
        Cancel(taskId); // Cancel existing task with same ID
        
        var cts = timeout.HasValue
            ? new CancellationTokenSource(timeout.Value)
            : new CancellationTokenSource();
        
        _tasks[taskId] = cts;
        
        try
        {
            await Task.Run(() => work(cts.Token), cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine($"Task '{taskId}' was cancelled");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Task '{taskId}' failed: {ex.Message}");
        }
        finally
        {
            _tasks.Remove(taskId);
            cts.Dispose();
        }
    }
    
    public void Cancel(string taskId)
    {
        if (_tasks.TryGetValue(taskId, out var cts))
        {
            cts.Cancel();
            _tasks.Remove(taskId);
        }
    }
    
    public void CancelAll()
    {
        foreach (var cts in _tasks.Values)
            cts.Cancel();
        _tasks.Clear();
    }
    
    public void Dispose() => CancelAll();
}

// ============================================
// Startup Performance
// ============================================

public class LazyInitializationExample
{
    // Lazy<T> - initialize only when needed
    private readonly Lazy<DatabaseService> _db;
    private readonly Lazy<Task<List<ProductEntity>>> _products;
    
    public LazyInitializationExample()
    {
        _db = new Lazy<DatabaseService>(() => new DatabaseService());
        _products = new Lazy<Task<List<ProductEntity>>>(() => 
            _db.Value.GetAllProductsAsync());
    }
    
    public DatabaseService Database => _db.Value;
    public Task<List<ProductEntity>> GetProductsAsync() => _products.Value;
}

// ============================================
// Profiling with Stopwatch
// ============================================

public class PerformanceProfiler
{
    private readonly Dictionary<string, List<long>> _measurements = new();
    
    public IDisposable Measure(string name)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        return new MeasurementScope(() =>
        {
            sw.Stop();
            if (!_measurements.ContainsKey(name))
                _measurements[name] = new List<long>();
            _measurements[name].Add(sw.ElapsedMilliseconds);
        });
    }
    
    public void PrintReport()
    {
        foreach (var (name, times) in _measurements)
        {
            Console.WriteLine($"{name}:");
            Console.WriteLine($"  Count: {times.Count}");
            Console.WriteLine($"  Avg: {times.Average():F2}ms");
            Console.WriteLine($"  Min: {times.Min()}ms");
            Console.WriteLine($"  Max: {times.Max()}ms");
        }
    }
    
    private class MeasurementScope : IDisposable
    {
        private readonly Action _onDispose;
        public MeasurementScope(Action onDispose) => _onDispose = onDispose;
        public void Dispose() => _onDispose();
    }
}
```

---

## Step 257: Startup Time Optimization

```csharp
// ============================================
// App Startup Optimization
// ============================================

// 1. Lazy Service Registration
public static class StartupOptimization
{
    // Don't initialize everything at startup
    public static IServiceCollection AddOptimizedServices(
        this IServiceCollection services)
    {
        // Use factory for expensive services
        services.AddSingleton<IExpensiveService>(sp =>
        {
            // Only created when first requested
            return new ExpensiveService();
        });
        
        // Background initialization
        services.AddHostedService<BackgroundInitService>();
        
        return services;
    }
}

// 2. Splash Screen + Background init
public class BackgroundInitService : IHostedService
{
    private readonly IServiceProvider _sp;
    
    public BackgroundInitService(IServiceProvider sp) => _sp = sp;
    
    public async Task StartAsync(CancellationToken ct)
    {
        // Run in background, don't block startup
        _ = Task.Run(async () =>
        {
            await Task.Delay(100, ct); // Let UI render first
            
            using var scope = _sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<DatabaseService>();
            
            // Pre-warm cache
            await db.GetAllProductsAsync();
        }, ct);
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

public class IExpensiveService { }
public class ExpensiveService : IExpensiveService { }
```

---

## Step 258: Network Optimization

```csharp
// ============================================
// Network Optimization
// ============================================

public class OptimizedApiClient
{
    private readonly HttpClient _http;
    private readonly MemoryCache<string, string> _responseCache;
    
    public OptimizedApiClient(HttpClient http)
    {
        _http = http;
        _responseCache = new MemoryCache<string, string>(TimeSpan.FromMinutes(2));
    }
    
    // Cache GET responses
    public async Task<T?> GetCachedAsync<T>(string endpoint, 
        TimeSpan? cacheDuration = null, CancellationToken ct = default)
    {
        var cacheKey = endpoint;
        
        // Return cached if available
        if (_responseCache.TryGet(cacheKey, out var cachedJson) && cachedJson != null)
            return System.Text.Json.JsonSerializer.Deserialize<T>(cachedJson);
        
        // Fetch fresh
        var response = await _http.GetAsync(endpoint, ct);
        response.EnsureSuccessStatusCode();
        
        var json = await response.Content.ReadAsStringAsync(ct);
        _responseCache.Set(cacheKey, json, cacheDuration);
        
        return System.Text.Json.JsonSerializer.Deserialize<T>(json);
    }
    
    // Batch requests
    public async Task<List<T>> GetBatchAsync<T>(
        IEnumerable<string> endpoints, int maxConcurrency = 5,
        CancellationToken ct = default)
    {
        using var semaphore = new SemaphoreSlim(maxConcurrency);
        
        var tasks = endpoints.Select(async endpoint =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                return await GetCachedAsync<T>(endpoint, ct: ct);
            }
            finally
            {
                semaphore.Release();
            }
        });
        
        var results = await Task.WhenAll(tasks);
        return results.Where(r => r != null).Select(r => r!).ToList();
    }
    
    // Compression
    public static HttpClient CreateCompressedClient()
    {
        var handler = new HttpClientHandler
        {
            AutomaticDecompression = 
                System.Net.DecompressionMethods.GZip | 
                System.Net.DecompressionMethods.Deflate
        };
        return new HttpClient(handler);
    }
}
```

---

## Step 259: Database Optimization

```csharp
// ============================================
// SQLite Optimization
// ============================================

public class OptimizedDatabaseService
{
    private readonly SQLiteAsyncConnection _db;
    
    public OptimizedDatabaseService(string dbPath)
    {
        _db = new SQLiteAsyncConnection(dbPath, 
            SQLiteOpenFlags.ReadWrite | SQLiteOpenFlags.Create | 
            SQLiteOpenFlags.SharedCache);
        
        Initialize();
    }
    
    private void Initialize()
    {
        // Performance pragmas
        _db.ExecuteAsync("PRAGMA journal_mode=WAL;").GetAwaiter().GetResult();
        _db.ExecuteAsync("PRAGMA synchronous=NORMAL;").GetAwaiter().GetResult();
        _db.ExecuteAsync("PRAGMA cache_size=10000;").GetAwaiter().GetResult();
        _db.ExecuteAsync("PRAGMA temp_store=MEMORY;").GetAwaiter().GetResult();
    }
    
    // Batch insert (much faster than individual inserts)
    public async Task BulkInsertAsync<T>(IEnumerable<T> items)
    {
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var item in items)
                conn.Insert(item);
        });
    }
    
    // Indexed query
    public Task<List<ProductEntity>> GetByCategoryFastAsync(string category)
        => _db.QueryAsync<ProductEntity>(
            "SELECT * FROM products WHERE Category = ? AND IsActive = 1 ORDER BY Name",
            category);
    
    // Paginated with total count (single query)
    public async Task<(List<ProductEntity> Items, int Total)> GetPagedFastAsync(
        int page, int size)
    {
        // Use COUNT(*) OVER() window function for single-query pagination
        var query = @"
            SELECT *, COUNT(*) OVER() as TotalCount 
            FROM products 
            WHERE IsActive = 1
            ORDER BY CreatedAt DESC
            LIMIT ? OFFSET ?";
        
        var rows = await _db.QueryAsync<ProductWithCount>(query, size, (page - 1) * size);
        
        var total = rows.FirstOrDefault()?.TotalCount ?? 0;
        var items = rows.Select(r => (ProductEntity)r).ToList();
        
        return (items, total);
    }
    
    // Analyze queries
    public async Task AnalyzeQueryAsync(string query, params object[] args)
    {
        var plan = await _db.QueryAsync<QueryPlanRow>(
            $"EXPLAIN QUERY PLAN {query}", args);
        
        foreach (var row in plan)
            Console.WriteLine($"[Query Plan] {row.Detail}");
    }
}

public class ProductWithCount : ProductEntity
{
    public int TotalCount { get; set; }
    public static explicit operator ProductEntity(ProductWithCount p)
        => new() { Id = p.Id, Name = p.Name, Category = p.Category, 
                   Price = p.Price, Stock = p.Stock, IsActive = p.IsActive, 
                   CreatedAt = p.CreatedAt };
}

public class QueryPlanRow
{
    public int Id { get; set; }
    public int Parent { get; set; }
    public int Notused { get; set; }
    public string Detail { get; set; } = string.Empty;
}
```

---

## Step 260: Profiling and Monitoring

```csharp
// ============================================
// App Performance Monitoring
// ============================================

public class AppPerformanceMonitor
{
    private readonly Dictionary<string, PerformanceMetric> _metrics = new();
    
    public void RecordMetric(string name, double value, string unit = "ms")
    {
        if (!_metrics.TryGetValue(name, out var metric))
        {
            metric = new PerformanceMetric(name, unit);
            _metrics[name] = metric;
        }
        
        metric.AddSample(value);
    }
    
    public IDisposable MeasureScope(string name)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        return new DisposableAction(() =>
        {
            sw.Stop();
            RecordMetric(name, sw.ElapsedMilliseconds);
        });
    }
    
    public PerformanceSummary GetSummary()
    {
        return new PerformanceSummary(
            _metrics.ToDictionary(
                kv => kv.Key,
                kv => kv.Value.GetStats()));
    }
    
    public void Reset() => _metrics.Clear();
    
    private class DisposableAction : IDisposable
    {
        private readonly Action _action;
        public DisposableAction(Action action) => _action = action;
        public void Dispose() => _action();
    }
}

public class PerformanceMetric
{
    private readonly List<double> _samples = new();
    
    public string Name { get; }
    public string Unit { get; }
    
    public PerformanceMetric(string name, string unit)
    {
        Name = name;
        Unit = unit;
    }
    
    public void AddSample(double value) => _samples.Add(value);
    
    public MetricStats GetStats() => new()
    {
        Count = _samples.Count,
        Average = _samples.Count > 0 ? _samples.Average() : 0,
        Min = _samples.Count > 0 ? _samples.Min() : 0,
        Max = _samples.Count > 0 ? _samples.Max() : 0,
        P95 = GetPercentile(95),
        P99 = GetPercentile(99)
    };
    
    private double GetPercentile(int percentile)
    {
        if (_samples.Count == 0) return 0;
        var sorted = _samples.OrderBy(x => x).ToList();
        int idx = (int)Math.Ceiling(percentile / 100.0 * sorted.Count) - 1;
        return sorted[Math.Clamp(idx, 0, sorted.Count - 1)];
    }
}

public record MetricStats
{
    public int Count { get; init; }
    public double Average { get; init; }
    public double Min { get; init; }
    public double Max { get; init; }
    public double P95 { get; init; }
    public double P99 { get; init; }
}

public record PerformanceSummary(Dictionary<string, MetricStats> Metrics);
```

---

## สรุป Part 26

ใน Part 26 เราได้เรียนรู้:

1. **Memory Management** - Span<T>, ArrayPool, GC generations
2. **String Performance** - StringBuilder vs concat, interning, String.Create
3. **LINQ Performance** - Any() vs Count(), deferred execution, Dictionary lookup
4. **UI Performance** - CollectionView virtualization, compiled bindings
5. **Caching** - MemoryCache<TKey,TValue>, TTL, eviction
6. **Background Tasks** - Task management, cancellation
7. **Startup Optimization** - Lazy init, background pre-warm
8. **Network Optimization** - Response caching, batch requests, compression
9. **Database Optimization** - WAL mode, batch insert, window functions
10. **Performance Monitoring** - Metrics, Stopwatch, percentiles

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 26 | Steps 251-260*
