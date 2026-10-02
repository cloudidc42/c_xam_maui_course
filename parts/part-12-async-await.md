# Part 12: Async/Await และ Task Programming
## Steps 111-120: การเขียนโปรแกรมแบบ Asynchronous

---

## Step 111: Async/Await พื้นฐาน

```csharp
// ============================================
// ทำไมต้องใช้ Async
// ============================================

/*
 * Synchronous: thread ถูกบล็อกขณะรอ I/O
 * Asynchronous: thread ถูกปล่อยออกขณะรอ I/O
 * 
 * Use cases:
 * - HTTP requests
 * - File I/O
 * - Database queries
 * - UI operations (MAUI/WPF)
 */

// ❌ Synchronous (บล็อก thread)
string BadDownload(string url)
{
    using var client = new HttpClient();
    return client.GetStringAsync(url).Result; // บล็อก! อาจ deadlock
}

// ✅ Asynchronous
async Task<string> GoodDownloadAsync(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url); // ไม่บล็อก
}

// ============================================
// async/await syntax
// ============================================

// Method ที่ return Task
async Task DoSomethingAsync()
{
    Console.WriteLine("Start");
    await Task.Delay(1000); // รอ 1 วินาที โดยไม่บล็อก thread
    Console.WriteLine("Done after 1 second");
}

// Method ที่ return Task<T>
async Task<int> ComputeAsync()
{
    await Task.Delay(500);
    return 42;
}

// Method ที่ return ValueTask<T> (เบากว่า Task สำหรับ hot path)
async ValueTask<string> GetCachedAsync(string key)
{
    if (_cache.TryGetValue(key, out string? cached))
        return cached; // ไม่ต้อง await - return ทันที
    
    string value = await LoadFromDbAsync(key);
    _cache[key] = value;
    return value;
}

private Dictionary<string, string> _cache = new();
private Task<string> LoadFromDbAsync(string key)
    => Task.FromResult($"value_{key}");

// ============================================
// Task States
// ============================================

async Task DemoTaskStates()
{
    // Created (not started)
    var task = new Task<int>(() => 42);
    Console.WriteLine($"State: {task.Status}"); // Created
    
    // Running
    task.Start();
    Console.WriteLine($"State: {task.Status}"); // Running/WaitingToRun
    
    // Completed
    int result = await task;
    Console.WriteLine($"State: {task.Status}"); // RanToCompletion
    Console.WriteLine($"Result: {result}");
    
    // Faulted
    var faulted = Task.Run(() => throw new Exception("Error"));
    try { await faulted; }
    catch { }
    Console.WriteLine($"Faulted state: {faulted.Status}"); // Faulted
    
    // Cancelled
    var cts = new CancellationTokenSource();
    var cancelled = Task.Run(async () => {
        await Task.Delay(5000, cts.Token);
    }, cts.Token);
    cts.Cancel();
    try { await cancelled; }
    catch (OperationCanceledException) { }
    Console.WriteLine($"Cancelled state: {cancelled.Status}"); // Canceled
}
```

---

## Step 112: Task Combinators

```csharp
// ============================================
// Task.WhenAll - รอทุก tasks
// ============================================

async Task WhenAllExample()
{
    // รันพร้อมกัน
    var task1 = Task.Delay(1000).ContinueWith(_ => "Result 1");
    var task2 = Task.Delay(500).ContinueWith(_ => "Result 2");
    var task3 = Task.Delay(1500).ContinueWith(_ => "Result 3");
    
    var start = DateTime.Now;
    var results = await Task.WhenAll(task1, task2, task3);
    var elapsed = (DateTime.Now - start).TotalMilliseconds;
    
    Console.WriteLine($"All done in {elapsed:F0}ms"); // ~1500ms (not 3000ms)
    foreach (var r in results)
        Console.WriteLine($"  {r}");
}

// ============================================
// Task.WhenAny - รอ task แรกที่เสร็จ
// ============================================

async Task WhenAnyExample()
{
    // Race: ใช้ผลจาก server ที่เร็วที่สุด
    var server1 = FetchFromServerAsync("server1", 800);
    var server2 = FetchFromServerAsync("server2", 1200);
    var server3 = FetchFromServerAsync("server3", 600);
    
    var fastest = await Task.WhenAny(server1, server2, server3);
    Console.WriteLine($"Fastest result: {await fastest}");
}

async Task<string> FetchFromServerAsync(string server, int delay)
{
    await Task.Delay(delay);
    return $"Data from {server}";
}

// ============================================
// Task.WhenAll with Error Handling
// ============================================

async Task HandleMultipleErrors()
{
    var tasks = new[]
    {
        Task.Run(() => "OK 1"),
        Task.Run<string>(() => throw new Exception("Error 2")),
        Task.Run(() => "OK 3"),
        Task.Run<string>(() => throw new Exception("Error 4")),
    };
    
    var results = await Task.WhenAll(tasks.Select(t => t.ContinueWith(t =>
        t.IsFaulted 
            ? Result<string>.Failure(t.Exception!.InnerException!.Message)
            : Result<string>.Success(t.Result)
    )));
    
    foreach (var r in results)
        r.Match(
            val => Console.WriteLine($"✓ {val}"),
            err => Console.WriteLine($"✗ {err}")
        );
}

// ============================================
// Parallel Work with Throttling
// ============================================

async Task ProcessWithThrottleAsync<T>(
    IEnumerable<T> items, 
    Func<T, Task> processor,
    int maxConcurrency = 5)
{
    using var semaphore = new SemaphoreSlim(maxConcurrency);
    
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try
        {
            await processor(item);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    await Task.WhenAll(tasks);
}

// ใช้งาน
var emails = Enumerable.Range(1, 20).Select(i => $"user{i}@example.com");
await ProcessWithThrottleAsync(emails, async email =>
{
    await Task.Delay(100); // simulate sending email
    Console.WriteLine($"Sent to: {email}");
}, maxConcurrency: 5);
```

---

## Step 113: CancellationToken

```csharp
// ============================================
// CancellationToken
// ============================================

// ส่ง CancellationToken ตลอด call chain
async Task<List<string>> SearchAsync(
    string query, 
    CancellationToken cancellationToken = default)
{
    Console.WriteLine($"Searching for: {query}");
    
    // Check cancellation
    cancellationToken.ThrowIfCancellationRequested();
    
    // Pass token to inner operations
    await Task.Delay(2000, cancellationToken);
    
    var results = await FetchResultsAsync(query, cancellationToken);
    
    return results;
}

async Task<List<string>> FetchResultsAsync(string query, CancellationToken ct)
{
    await Task.Delay(500, ct);
    return new List<string> { "Result 1", "Result 2" };
}

// ============================================
// CancellationTokenSource
// ============================================

async Task DemoCancellation()
{
    // 1. Manual cancellation
    var cts = new CancellationTokenSource();
    
    var searchTask = SearchAsync("programming", cts.Token);
    
    // Cancel after 500ms
    await Task.Delay(500);
    cts.Cancel();
    
    try
    {
        await searchTask;
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Search was cancelled");
    }
    
    // 2. Timeout cancellation
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(1));
    
    try
    {
        await SearchAsync("slow query", timeoutCts.Token);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Search timed out");
    }
    
    // 3. Linked cancellation
    var userCts = new CancellationTokenSource();
    var timeoutCts2 = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(
        userCts.Token, timeoutCts2.Token);
    
    // Cancel when either user cancels OR timeout occurs
    await SearchAsync("query", linked.Token);
}

// ============================================
// Cooperative Cancellation
// ============================================

async Task<int[]> ProcessItemsAsync(
    int[] items, 
    CancellationToken ct)
{
    var results = new List<int>();
    
    foreach (var item in items)
    {
        // Check before each expensive operation
        ct.ThrowIfCancellationRequested();
        
        // Pass token to async operations
        await Task.Delay(10, ct);
        results.Add(item * 2);
    }
    
    return results.ToArray();
}
```

---

## Step 114: Progress Reporting

```csharp
// ============================================
// IProgress<T>
// ============================================

public record ProgressInfo(int Current, int Total, string Message)
{
    public double Percentage => Total == 0 ? 0 : (double)Current / Total * 100;
    public override string ToString() => $"[{Percentage:F0}%] {Message} ({Current}/{Total})";
}

async Task ProcessFilesAsync(
    IEnumerable<string> files,
    IProgress<ProgressInfo>? progress = null,
    CancellationToken ct = default)
{
    var fileList = files.ToList();
    int total = fileList.Count;
    
    for (int i = 0; i < total; i++)
    {
        ct.ThrowIfCancellationRequested();
        
        // Process file
        await Task.Delay(100, ct); // simulate work
        
        progress?.Report(new ProgressInfo(i + 1, total, $"Processing: {fileList[i]}"));
    }
}

// ใน Console App
var progress = new Progress<ProgressInfo>(info =>
{
    Console.Write($"\r{info}  "); // Overwrite same line
});

var files = Enumerable.Range(1, 10).Select(i => $"file{i:D2}.csv");
await ProcessFilesAsync(files, progress);
Console.WriteLine("\nDone!");

// ============================================
// Progress ใน MAUI (UI Thread safe)
// ============================================

// Progress<T> automatically posts to the synchronization context
// ดังนั้น callback จะรันบน UI thread เสมอใน MAUI/WPF

/*
// ViewModel
private int _progressValue;
public int ProgressValue
{
    get => _progressValue;
    set { _progressValue = value; OnPropertyChanged(); }
}

private async Task StartProcessingAsync()
{
    var progress = new Progress<int>(value => ProgressValue = value);
    await ProcessDataAsync(progress);
}
*/
```

---

## Step 115: Async Patterns

```csharp
// ============================================
// Async Stream (IAsyncEnumerable)
// ============================================

async IAsyncEnumerable<int> GenerateNumbersAsync(
    int count,
    [System.Runtime.CompilerServices.EnumeratorCancellation] 
    CancellationToken ct = default)
{
    for (int i = 0; i < count; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct);
        yield return i;
    }
}

// consume async stream
await foreach (var num in GenerateNumbersAsync(5))
{
    Console.WriteLine($"Received: {num}");
}

// ============================================
// Async Stream ด้วย Real Data
// ============================================

async IAsyncEnumerable<string> ReadLinesAsync(
    string filePath,
    [System.Runtime.CompilerServices.EnumeratorCancellation]
    CancellationToken ct = default)
{
    using var reader = new StreamReader(filePath);
    
    while (!reader.EndOfStream)
    {
        ct.ThrowIfCancellationRequested();
        string? line = await reader.ReadLineAsync(ct);
        if (line != null)
            yield return line;
    }
}

// Process large CSV without loading all into memory
async Task ProcessLargeCsvAsync(string csvPath)
{
    int lineCount = 0;
    await foreach (var line in ReadLinesAsync(csvPath))
    {
        var fields = line.Split(',');
        // Process each line
        lineCount++;
    }
    Console.WriteLine($"Processed {lineCount} lines");
}

// ============================================
// Async with Timeout Pattern
// ============================================

async Task<T> WithTimeoutAsync<T>(
    Task<T> task, 
    TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    var timeoutTask = Task.Delay(Timeout.Infinite, cts.Token);
    
    var completed = await Task.WhenAny(task, timeoutTask);
    
    if (completed == timeoutTask)
        throw new TimeoutException($"Operation timed out after {timeout}");
    
    return await task;
}

// ============================================
// Fire-and-Forget Pattern
// ============================================

// ❌ Bad: swallows exceptions
void BadFireAndForget()
{
    _ = DoSomethingAsync(); // หาก throw, exception หาย
}

// ✅ Good: handle exceptions
void GoodFireAndForget(ILogger? logger = null)
{
    _ = DoSomethingAsync().ContinueWith(t =>
    {
        if (t.IsFaulted)
            logger?.LogError($"Background task failed: {t.Exception?.GetBaseException()?.Message}");
    }, TaskContinuationOptions.OnlyOnFaulted);
}

async Task DoSomethingAsync()
{
    await Task.Delay(1000);
    Console.WriteLine("Background task done");
}
```

---

## Step 116: ConfigureAwait

```csharp
// ============================================
// ConfigureAwait(false)
// ============================================

/*
 * await ปกติ: resume บน original context (UI thread ใน MAUI)
 * ConfigureAwait(false): resume บน thread pool thread
 * 
 * ใช้ ConfigureAwait(false) ใน library code
 * อย่าใช้ใน UI code (ViewModel, code-behind)
 */

// Library code
public class DataService2
{
    public async Task<string> GetDataAsync()
    {
        // I/O operations - ไม่ต้องการ UI thread
        string data = await File.ReadAllTextAsync("data.txt")
            .ConfigureAwait(false);
        
        // CPU processing - ไม่ต้องการ UI thread
        string processed = await Task.Run(() => ProcessData(data))
            .ConfigureAwait(false);
        
        return processed;
    }
    
    private string ProcessData(string data) => data.ToUpper();
}

// ❌ Bad ใน library: 
/*
public async Task<string> GetDataAsyncBad()
{
    // ถ้าเรียกจาก UI thread, จะ resume กลับ UI thread
    // ทำให้ UI อาจ hang ถ้า library code รอบน UI thread
    string data = await File.ReadAllTextAsync("data.txt");
    return data;
}
*/

// ============================================
// Async ใน MAUI ViewModel
// ============================================

/*
public class MainViewModel : INotifyPropertyChanged
{
    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set { _isLoading = value; OnPropertyChanged(); }
    }
    
    private string _data = string.Empty;
    public string Data
    {
        get => _data;
        set { _data = value; OnPropertyChanged(); }
    }
    
    // ✅ ไม่ต้อง ConfigureAwait(false) ใน ViewModel
    // เพราะต้องการ update UI properties บน UI thread
    public async Task LoadDataAsync()
    {
        IsLoading = true;
        try
        {
            // Library ทำ ConfigureAwait(false) เอง
            Data = await _dataService.GetDataAsync();
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
*/
```

---

## Step 117: Channel - Producer/Consumer

```csharp
using System.Threading.Channels;

// ============================================
// Channel<T> - High-performance message passing
// ============================================

async Task DemoChannelAsync()
{
    // Bounded channel (buffer = 10)
    var channel = Channel.CreateBounded<string>(10);
    
    // Unbounded channel
    var unbounded = Channel.CreateUnbounded<string>();
    
    // Producer
    async Task ProduceAsync(ChannelWriter<string> writer)
    {
        for (int i = 0; i < 20; i++)
        {
            await writer.WriteAsync($"Message {i}");
            Console.WriteLine($"Produced: Message {i}");
            await Task.Delay(50);
        }
        writer.Complete(); // signal done
    }
    
    // Consumer
    async Task ConsumeAsync(ChannelReader<string> reader)
    {
        await foreach (var item in reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumed: {item}");
            await Task.Delay(100); // slower than producer
        }
    }
    
    // Run producer and consumer concurrently
    await Task.WhenAll(
        ProduceAsync(channel.Writer),
        ConsumeAsync(channel.Reader)
    );
}

// ============================================
// Multiple Producers, Multiple Consumers
// ============================================

async Task MultiProducerConsumerAsync()
{
    var channel = Channel.CreateUnbounded<int>();
    
    // 3 Producers
    var producers = Enumerable.Range(0, 3).Select(pid => Task.Run(async () =>
    {
        for (int i = 0; i < 5; i++)
        {
            int value = pid * 100 + i;
            await channel.Writer.WriteAsync(value);
            Console.WriteLine($"Producer {pid}: wrote {value}");
            await Task.Delay(new Random().Next(50, 150));
        }
    }));
    
    // 2 Consumers  
    var consumers = Enumerable.Range(0, 2).Select(cid => Task.Run(async () =>
    {
        await foreach (var value in channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumer {cid}: processing {value}");
            await Task.Delay(200);
        }
    }));
    
    // Wait for all producers
    await Task.WhenAll(producers);
    channel.Writer.Complete();
    
    // Wait for consumers to drain
    await Task.WhenAll(consumers);
}
```

---

## Step 118: Async Data Access Pattern

```csharp
// ============================================
// Repository Pattern ด้วย Async
// ============================================

public interface IAsyncRepository<T, TId>
{
    Task<T?> GetByIdAsync(TId id, CancellationToken ct = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default);
    Task<IEnumerable<T>> FindAsync(Func<T, bool> predicate, CancellationToken ct = default);
    Task<T> AddAsync(T entity, CancellationToken ct = default);
    Task<T> UpdateAsync(T entity, CancellationToken ct = default);
    Task<bool> DeleteAsync(TId id, CancellationToken ct = default);
}

public record Product6(int Id, string Name, decimal Price, int Stock);

public class InMemoryProductRepository : IAsyncRepository<Product6, int>
{
    private readonly List<Product6> _products = new()
    {
        new(1, "iPhone 15", 35000, 100),
        new(2, "Samsung S24", 30000, 150),
    };
    private int _nextId = 3;
    
    public Task<Product6?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        return Task.FromResult(_products.FirstOrDefault(p => p.Id == id));
    }
    
    public Task<IEnumerable<Product6>> GetAllAsync(CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        return Task.FromResult<IEnumerable<Product6>>(_products.ToList());
    }
    
    public Task<IEnumerable<Product6>> FindAsync(
        Func<Product6, bool> predicate, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        return Task.FromResult<IEnumerable<Product6>>(_products.Where(predicate).ToList());
    }
    
    public async Task<Product6> AddAsync(Product6 entity, CancellationToken ct = default)
    {
        await Task.Delay(10, ct); // simulate DB latency
        var product = entity with { Id = _nextId++ };
        _products.Add(product);
        return product;
    }
    
    public async Task<Product6> UpdateAsync(Product6 entity, CancellationToken ct = default)
    {
        await Task.Delay(10, ct);
        int idx = _products.FindIndex(p => p.Id == entity.Id);
        if (idx < 0) throw new NotFoundException2("Product", entity.Id);
        _products[idx] = entity;
        return entity;
    }
    
    public async Task<bool> DeleteAsync(int id, CancellationToken ct = default)
    {
        await Task.Delay(10, ct);
        int removed = _products.RemoveAll(p => p.Id == id);
        return removed > 0;
    }
}

// Service layer
public class ProductService5
{
    private readonly IAsyncRepository<Product6, int> _repo;
    
    public ProductService5(IAsyncRepository<Product6, int> repo) => _repo = repo;
    
    public async Task<IEnumerable<Product6>> GetInStockAsync(CancellationToken ct = default)
        => await _repo.FindAsync(p => p.Stock > 0, ct);
    
    public async Task<Result<Product6>> RestockAsync(
        int productId, int quantity, CancellationToken ct = default)
    {
        var product = await _repo.GetByIdAsync(productId, ct);
        if (product == null) return Result<Product6>.Failure($"Product {productId} not found");
        
        var updated = await _repo.UpdateAsync(product with { Stock = product.Stock + quantity }, ct);
        return Result<Product6>.Success(updated);
    }
}
```

---

## Step 119: Async Anti-Patterns

```csharp
// ============================================
// Anti-Patterns ที่ควรหลีกเลี่ยง
// ============================================

// ❌ 1. .Result หรือ .Wait() - อาจ deadlock
string BadSync()
{
    return GetDataAsync().Result; // DEADLOCK risk in UI contexts!
}

async Task<string> GetDataAsync()
{
    await Task.Delay(100);
    return "data";
}

// ✅ 1. ใช้ async ตลอด
async Task<string> GoodAsync() => await GetDataAsync();

// ❌ 2. async void (ยกเว้น event handlers)
async void BadAsyncVoid()
{
    await Task.Delay(100);
    throw new Exception("This exception will crash the process!"); // unhandled!
}

// ✅ 2. async Task
async Task GoodAsyncTask()
{
    await Task.Delay(100);
    throw new Exception("This is catchable");
}

// ✅ 2b. async void สำหรับ event handlers เท่านั้น
void OnButtonClicked(object sender, EventArgs e)
{
    // Wrap in Task to handle exceptions
    _ = HandleClickAsync().ContinueWith(t =>
    {
        if (t.IsFaulted)
            Console.WriteLine($"Click handler error: {t.Exception}");
    });
}

async Task HandleClickAsync()
{
    await LoadDataAsync2();
}

async Task LoadDataAsync2() => await Task.Delay(100);

// ❌ 3. ไม่ส่ง CancellationToken
async Task BadNoCancel()
{
    await Task.Delay(10000); // ไม่สามารถยกเลิกได้
}

// ✅ 3. ส่ง CancellationToken เสมอ
async Task GoodWithCancel(CancellationToken ct = default)
{
    await Task.Delay(10000, ct);
}

// ❌ 4. ไม่จำเป็นต้อง async
async Task<string> UnnecessaryAsync() // warning: CS1998
{
    return "hello"; // ไม่มี await!
}

// ✅ 4. ถ้าไม่มี await ให้ return Task.FromResult
Task<string> NecessarySync()
{
    return Task.FromResult("hello"); // ประสิทธิภาพดีกว่า
}

// ❌ 5. ทำงานแบบ sequential เมื่อควรจะ parallel
async Task SequentialBad()
{
    var r1 = await FetchAsync("url1"); // รอ
    var r2 = await FetchAsync("url2"); // รอ
    var r3 = await FetchAsync("url3"); // รอ
    // Total: ~3000ms
}

async Task<string> FetchAsync(string url)
{
    await Task.Delay(1000);
    return $"data from {url}";
}

// ✅ 5. รันพร้อมกัน
async Task ParallelGood()
{
    var t1 = FetchAsync("url1");
    var t2 = FetchAsync("url2");
    var t3 = FetchAsync("url3");
    
    var results = await Task.WhenAll(t1, t2, t3); // ~1000ms
}
```

---

## Step 120: Complete Async Application

```csharp
// ============================================
// Complete Async Application: News Aggregator
// ============================================

public record Article(string Title, string Source, string Content, DateTime Published);
public record NewsSource(string Name, string Url, int Priority);

public class NewsAggregator
{
    private readonly IEnumerable<NewsSource> _sources;
    private readonly int _maxConcurrency;
    
    public NewsAggregator(IEnumerable<NewsSource> sources, int maxConcurrency = 3)
    {
        _sources = sources;
        _maxConcurrency = maxConcurrency;
    }
    
    public async Task<List<Article>> FetchAllArticlesAsync(
        IProgress<(string Source, int ArticleCount)>? progress = null,
        CancellationToken ct = default)
    {
        using var semaphore = new SemaphoreSlim(_maxConcurrency);
        var allArticles = new System.Collections.Concurrent.ConcurrentBag<Article>();
        
        var tasks = _sources.Select(async source =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                var articles = await FetchFromSourceAsync(source, ct);
                foreach (var article in articles)
                    allArticles.Add(article);
                
                progress?.Report((source.Name, articles.Count));
            }
            finally
            {
                semaphore.Release();
            }
        });
        
        await Task.WhenAll(tasks);
        
        return allArticles
            .OrderByDescending(a => a.Published)
            .ToList();
    }
    
    private async Task<List<Article>> FetchFromSourceAsync(
        NewsSource source, CancellationToken ct)
    {
        // Simulate fetching with priority-based delay
        await Task.Delay(source.Priority * 100, ct);
        
        return new List<Article>
        {
            new Article($"ข่าว 1 จาก {source.Name}", source.Name, 
                        "เนื้อหาข่าว...", DateTime.Now.AddMinutes(-30)),
            new Article($"ข่าว 2 จาก {source.Name}", source.Name, 
                        "เนื้อหาข่าว...", DateTime.Now.AddMinutes(-60)),
        };
    }
    
    public async IAsyncEnumerable<Article> StreamArticlesAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken ct = default)
    {
        foreach (var source in _sources.OrderBy(s => s.Priority))
        {
            ct.ThrowIfCancellationRequested();
            
            var articles = await FetchFromSourceAsync(source, ct);
            foreach (var article in articles)
            {
                yield return article;
                await Task.Delay(10, ct); // slight delay for streaming effect
            }
        }
    }
}

// ใช้งาน
var sources = new[]
{
    new NewsSource("Thai PBS", "https://thaipbs.com", 1),
    new NewsSource("BBC Thai", "https://bbc.com/thai", 2),
    new NewsSource("Matichon", "https://matichon.co.th", 3),
};

var aggregator = new NewsAggregator(sources, maxConcurrency: 2);

// Progress tracking
var progress = new Progress<(string Source, int ArticleCount)>(info =>
    Console.WriteLine($"Fetched {info.ArticleCount} articles from {info.Source}"));

using var cts2 = new CancellationTokenSource(TimeSpan.FromSeconds(10));

Console.WriteLine("Fetching all articles...");
var articles = await aggregator.FetchAllArticlesAsync(progress, cts2.Token);
Console.WriteLine($"\nTotal articles: {articles.Count}");
Console.WriteLine("\nTop 5 latest:");
articles.Take(5).ToList().ForEach(a => 
    Console.WriteLine($"  [{a.Source}] {a.Title} ({a.Published:HH:mm})"));

// Streaming
Console.WriteLine("\n=== Streaming articles ===");
await foreach (var article in aggregator.StreamArticlesAsync(cts2.Token))
{
    Console.WriteLine($"  {article.Title}");
}
```

---

## สรุป Part 12

ใน Part 12 เราได้เรียนรู้:

1. **Async/Await Basics** - Task, Task<T>, ValueTask
2. **Task Combinators** - WhenAll, WhenAny
3. **CancellationToken** - Cancellation, Timeout, Linked
4. **Progress Reporting** - IProgress<T>
5. **Async Stream** - IAsyncEnumerable
6. **Channel** - Producer/Consumer pattern
7. **ConfigureAwait** - Library vs UI code
8. **Async Repository** - Data access patterns
9. **Anti-Patterns** - .Result, async void, sequential
10. **Complete Example** - News Aggregator

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 12 | Steps 111-120*
