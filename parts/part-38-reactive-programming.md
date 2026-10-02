# Part 38: Reactive Programming กับ Rx.NET
## Steps 371-380: ReactiveUI และ Observable Streams

---

## Step 371: Rx.NET Fundamentals

```csharp
// ============================================
// Reactive Extensions (Rx.NET)
// ============================================

/*
 * Rx.NET คืออะไร?
 * - Library สำหรับ asynchronous event-based programming
 * - ใช้ LINQ operators กับ event streams
 * - ทำให้โค้ด async ซับซ้อนอ่านง่ายขึ้น
 *
 * NuGet:
 * System.Reactive
 * ReactiveUI
 * ReactiveUI.Maui
 */

using System.Reactive;
using System.Reactive.Linq;
using System.Reactive.Subjects;
using System.Reactive.Disposables;

// Observable basics
public class RxBasicsDemo
{
    public void Demo()
    {
        // Create observable from values
        var numbers = Observable.Range(1, 10);
        
        // Subscribe
        var sub = numbers
            .Where(n => n % 2 == 0)
            .Select(n => n * n)
            .Subscribe(
                n => Console.WriteLine($"Value: {n}"),
                ex => Console.WriteLine($"Error: {ex.Message}"),
                () => Console.WriteLine("Completed"));
        
        // Dispose when done
        sub.Dispose();
        
        // Create from events
        var clickObs = Observable.FromEventPattern<EventArgs>(
            h => Button.Clicked += h,
            h => Button.Clicked -= h);
        
        // Throttle rapid events (debounce)
        clickObs
            .Throttle(TimeSpan.FromMilliseconds(300))
            .Subscribe(_ => HandleClick());
        
        // Timer
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Take(10)
            .Subscribe(i => Console.WriteLine($"Tick {i}"));
        
        // From task
        var asyncObs = Observable.FromAsync(async ct =>
        {
            await Task.Delay(1000, ct);
            return "Done";
        });
    }
    
    private static void HandleClick() { }
    private static Button Button => null!;
}
```

---

## Step 372: Subject Types

```csharp
// ============================================
// Subjects - hot observables
// ============================================

public class SubjectExamples
{
    // Subject<T> - basic pub/sub
    public void BasicSubject()
    {
        var subject = new Subject<string>();
        
        subject.Subscribe(msg => Console.WriteLine($"Sub1: {msg}"));
        
        subject.OnNext("Hello");
        subject.OnNext("World");
        
        subject.Subscribe(msg => Console.WriteLine($"Sub2: {msg}")); // misses Hello/World
        
        subject.OnNext("Late subscriber gets this");
        subject.OnCompleted();
    }
    
    // BehaviorSubject<T> - has current value
    public void BehaviorSubjectExample()
    {
        var theme = new BehaviorSubject<AppTheme>(AppTheme.Light);
        
        // Always gets current value immediately
        theme.Subscribe(t => UpdateUI(t));
        
        theme.OnNext(AppTheme.Dark); // new subscribers get Dark immediately
    }
    
    // ReplaySubject<T> - replays N values to new subscribers
    public void ReplaySubjectExample()
    {
        var log = new ReplaySubject<string>(bufferSize: 10); // last 10 items
        
        log.OnNext("Event 1");
        log.OnNext("Event 2");
        log.OnNext("Event 3");
        
        // New subscriber gets all 3 replayed
        log.Subscribe(e => Console.WriteLine(e));
    }
    
    // AsyncSubject<T> - only emits final value on complete
    public void AsyncSubjectExample()
    {
        var resultSubject = new AsyncSubject<int>();
        
        resultSubject.Subscribe(result => Console.WriteLine($"Final: {result}"));
        
        resultSubject.OnNext(1);
        resultSubject.OnNext(2);
        resultSubject.OnNext(3); // only this is emitted
        resultSubject.OnCompleted();
    }
    
    private static void UpdateUI(AppTheme theme) { }
}
```

---

## Step 373: Operators

```csharp
// ============================================
// Key Rx Operators
// ============================================

public class RxOperators
{
    // Combining observables
    public void CombiningOperators()
    {
        var obs1 = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5);
        var obs2 = Observable.Interval(TimeSpan.FromSeconds(2)).Take(3);
        
        // Merge - interleave
        obs1.Merge(obs2).Subscribe(v => Console.WriteLine($"Merged: {v}"));
        
        // Concat - sequential
        obs1.Concat(obs2).Subscribe(v => Console.WriteLine($"Concat: {v}"));
        
        // CombineLatest - latest from each
        var username = new BehaviorSubject<string>("User");
        var isOnline = new BehaviorSubject<bool>(true);
        
        username.CombineLatest(isOnline, (u, o) => $"{u}: {(o ? "Online" : "Offline")}")
            .Subscribe(status => Console.WriteLine(status));
        
        // Zip - pair by position
        var letters = Observable.Return("A").Concat(Observable.Return("B"));
        var digits = Observable.Return(1).Concat(Observable.Return(2));
        
        letters.Zip(digits, (l, d) => $"{l}{d}")
            .Subscribe(pair => Console.WriteLine(pair));
    }
    
    // Filtering and transformation
    public void FilterTransform()
    {
        var source = Observable.Range(1, 20);
        
        // DistinctUntilChanged
        var taps = new Subject<string>();
        taps.DistinctUntilChanged()
            .Subscribe(t => Console.WriteLine($"Distinct: {t}"));
        
        // Buffer - collect into batches
        source.Buffer(5).Subscribe(batch =>
            Console.WriteLine($"Batch: {string.Join(",", batch)}"));
        
        // Window - like buffer but as observable
        source.Window(3).SelectMany(w => w.ToList())
            .Subscribe(batch => Console.WriteLine($"Window: {batch.Count}"));
        
        // Scan - running aggregate
        source.Scan(0, (acc, x) => acc + x)
            .Subscribe(sum => Console.WriteLine($"Running sum: {sum}"));
        
        // GroupBy
        source.GroupBy(n => n % 3)
            .SelectMany(g => g.ToList().Select(items => (Key: g.Key, Items: items)))
            .Subscribe(g => Console.WriteLine($"Group {g.Key}: {string.Join(",", g.Items)}"));
    }
    
    // Error handling
    public void ErrorHandling()
    {
        var failingSource = Observable.Create<int>(obs =>
        {
            obs.OnNext(1);
            obs.OnError(new Exception("Boom!"));
            return Disposable.Empty;
        });
        
        // Catch - recover from error
        failingSource.Catch(Observable.Return(-1))
            .Subscribe(v => Console.WriteLine($"Value: {v}"));
        
        // Retry - retry N times
        failingSource.Retry(3)
            .Subscribe(_ => { }, ex => Console.WriteLine($"Failed after retry: {ex.Message}"));
        
        // OnErrorResumeNext
        failingSource.OnErrorResumeNext(Observable.Return(99))
            .Subscribe(v => Console.WriteLine($"Resumed: {v}"));
    }
}
```

---

## Step 374: ReactiveUI Setup

```csharp
// ============================================
// ReactiveUI with .NET MAUI
// ============================================

// MauiProgram.cs
builder.UseReactiveUI();

// ReactiveViewModel base
public class ReactiveBaseViewModel : ReactiveObject, IActivatableViewModel
{
    public ViewModelActivator Activator { get; } = new ViewModelActivator();
    
    private string _title = string.Empty;
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }
    
    protected ReactiveBaseViewModel()
    {
        // WhenActivated fires when view activates/deactivates
        this.WhenActivated(disposables =>
        {
            Observable.Timer(TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30))
                .Subscribe(_ => RefreshData())
                .DisposeWith(disposables);
        });
    }
    
    protected virtual void RefreshData() { }
}

// SearchViewModel with Rx
public class SearchViewModel : ReactiveBaseViewModel
{
    private string _query = string.Empty;
    public string Query
    {
        get => _query;
        set => this.RaiseAndSetIfChanged(ref _query, value);
    }
    
    private bool _isSearching;
    public bool IsSearching
    {
        get => _isSearching;
        private set => this.RaiseAndSetIfChanged(ref _isSearching, value);
    }
    
    public ObservableCollection<Product> Results { get; } = new();
    
    public ReactiveCommand<Unit, IEnumerable<Product>> SearchCommand { get; }
    
    public SearchViewModel(IProductService productService)
    {
        // Derived observable - can search when query length >= 2
        var canSearch = this.WhenAnyValue(x => x.Query)
            .Select(q => !string.IsNullOrWhiteSpace(q) && q.Length >= 2);
        
        SearchCommand = ReactiveCommand.CreateFromTask(
            async _ => await productService.SearchAsync(Query),
            canSearch);
        
        // Auto-search with debounce
        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(400))
            .DistinctUntilChanged()
            .Where(q => q.Length >= 2)
            .ObserveOn(RxApp.MainThreadScheduler)
            .InvokeCommand(SearchCommand);
        
        // Bind results
        SearchCommand
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(products =>
            {
                Results.Clear();
                foreach (var p in products) Results.Add(p);
            });
        
        // Track loading state
        SearchCommand.IsExecuting
            .ToProperty(this, x => x.IsSearching);
        
        // Handle errors
        SearchCommand.ThrownExceptions
            .Subscribe(ex => Console.WriteLine($"Search error: {ex.Message}"));
    }
}
```

---

## Step 375: Reactive Forms

```csharp
// ============================================
// Reactive Form Validation
// ============================================

public class LoginViewModel : ReactiveObject
{
    private string _email = string.Empty;
    private string _password = string.Empty;
    private string? _emailError;
    private string? _passwordError;
    
    public string Email
    {
        get => _email;
        set => this.RaiseAndSetIfChanged(ref _email, value);
    }
    
    public string Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }
    
    public string? EmailError
    {
        get => _emailError;
        private set => this.RaiseAndSetIfChanged(ref _emailError, value);
    }
    
    public string? PasswordError
    {
        get => _passwordError;
        private set => this.RaiseAndSetIfChanged(ref _passwordError, value);
    }
    
    public ReactiveCommand<Unit, bool> LoginCommand { get; }
    
    public LoginViewModel(IAuthService authService)
    {
        // Email validation
        this.WhenAnyValue(x => x.Email)
            .Skip(1) // skip initial empty
            .Select(ValidateEmail)
            .BindTo(this, x => x.EmailError);
        
        // Password validation
        this.WhenAnyValue(x => x.Password)
            .Skip(1)
            .Select(ValidatePassword)
            .BindTo(this, x => x.PasswordError);
        
        // Form is valid when no errors
        var isValid = this.WhenAnyValue(
            x => x.EmailError,
            x => x.PasswordError,
            x => x.Email,
            x => x.Password,
            (emailErr, passErr, email, pass) =>
                emailErr == null && passErr == null &&
                !string.IsNullOrEmpty(email) && !string.IsNullOrEmpty(pass));
        
        LoginCommand = ReactiveCommand.CreateFromTask(
            async _ => await authService.LoginAsync(Email, Password),
            isValid);
    }
    
    private static string? ValidateEmail(string email)
    {
        if (string.IsNullOrEmpty(email)) return "กรุณากรอกอีเมล";
        if (!email.Contains('@')) return "รูปแบบอีเมลไม่ถูกต้อง";
        return null;
    }
    
    private static string? ValidatePassword(string password)
    {
        if (string.IsNullOrEmpty(password)) return "กรุณากรอกรหัสผ่าน";
        if (password.Length < 8) return "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร";
        return null;
    }
}
```

---

## Step 376: Event Aggregator with Rx

```csharp
// ============================================
// Rx-based Event Aggregator
// ============================================

public class RxEventBus : IDisposable
{
    private readonly Subject<object> _bus = new();
    
    public IObservable<T> Listen<T>()
        => _bus.OfType<T>().AsObservable();
    
    public void Publish<T>(T @event) where T : class
        => _bus.OnNext(@event);
    
    public void Dispose() => _bus.Dispose();
}

// Domain events
public record ProductAddedToCart(int ProductId, string Name, decimal Price, int Quantity);
public record OrderPlaced(int OrderId, decimal Total);
public record UserLoggedIn(int UserId, string Name);

// Usage
public class CartNotificationService : IDisposable
{
    private readonly CompositeDisposable _disposables = new();
    
    public CartNotificationService(RxEventBus bus)
    {
        // React to product added
        bus.Listen<ProductAddedToCart>()
            .Subscribe(e =>
            {
                Console.WriteLine($"Added: {e.Name} x{e.Quantity}");
            })
            .DisposeWith(_disposables);
        
        // Throttle analytics events
        bus.Listen<ProductAddedToCart>()
            .Buffer(TimeSpan.FromSeconds(5), 10)
            .Where(batch => batch.Count > 0)
            .Subscribe(batch => TrackAnalytics(batch))
            .DisposeWith(_disposables);
        
        // Track high-value orders
        bus.Listen<OrderPlaced>()
            .Where(o => o.Total >= 5000)
            .Subscribe(o => NotifyHighValue(o))
            .DisposeWith(_disposables);
    }
    
    private static void TrackAnalytics(IList<ProductAddedToCart> batch) { }
    private static void NotifyHighValue(OrderPlaced order) { }
    
    public void Dispose() => _disposables.Dispose();
}
```

---

## Step 377: Polling & Live Data

```csharp
// ============================================
// Polling with Rx
// ============================================

public class LivePriceService
{
    private readonly IStockApi _api;
    private readonly Dictionary<string, IObservable<decimal>> _priceStreams = new();
    
    public LivePriceService(IStockApi api) => _api = api;
    
    public IObservable<decimal> GetPriceStream(
        string symbol,
        TimeSpan interval = default)
    {
        var pollInterval = interval == default ? TimeSpan.FromSeconds(5) : interval;
        
        if (!_priceStreams.ContainsKey(symbol))
        {
            _priceStreams[symbol] = Observable
                .Timer(TimeSpan.Zero, pollInterval)
                .SelectMany(_ => Observable.FromAsync(() => _api.GetPriceAsync(symbol)))
                .DistinctUntilChanged()
                .Catch<decimal, Exception>(ex =>
                {
                    Console.WriteLine($"Price error for {symbol}: {ex.Message}");
                    return Observable.Empty<decimal>().Delay(TimeSpan.FromSeconds(10));
                })
                .Retry()
                .Publish()
                .RefCount(); // share one subscription among all subscribers
        }
        
        return _priceStreams[symbol];
    }
}

// ViewModel using live data
public class StockDashboardViewModel : ReactiveObject, IDisposable
{
    private readonly CompositeDisposable _disposables = new();
    
    private decimal _btcPrice;
    public decimal BtcPrice
    {
        get => _btcPrice;
        private set => this.RaiseAndSetIfChanged(ref _btcPrice, value);
    }
    
    private decimal _ethPrice;
    public decimal EthPrice
    {
        get => _ethPrice;
        private set => this.RaiseAndSetIfChanged(ref _ethPrice, value);
    }
    
    public StockDashboardViewModel(LivePriceService priceService)
    {
        priceService.GetPriceStream("BTC")
            .ObserveOn(RxApp.MainThreadScheduler)
            .BindTo(this, x => x.BtcPrice)
            .DisposeWith(_disposables);
        
        priceService.GetPriceStream("ETH")
            .ObserveOn(RxApp.MainThreadScheduler)
            .BindTo(this, x => x.EthPrice)
            .DisposeWith(_disposables);
    }
    
    public void Dispose() => _disposables.Dispose();
}
```

---

## Step 378: ObservableAsPropertyHelper

```csharp
// ============================================
// ObservableAsPropertyHelper - Derived Properties
// ============================================

public class ProductDetailViewModel : ReactiveObject
{
    private int _quantity = 1;
    private decimal _basePrice;
    private bool _hasDiscount;
    private double _discountRate;
    
    public int Quantity
    {
        get => _quantity;
        set => this.RaiseAndSetIfChanged(ref _quantity, value);
    }
    
    public decimal BasePrice
    {
        get => _basePrice;
        set => this.RaiseAndSetIfChanged(ref _basePrice, value);
    }
    
    public bool HasDiscount
    {
        get => _hasDiscount;
        set => this.RaiseAndSetIfChanged(ref _hasDiscount, value);
    }
    
    public double DiscountRate
    {
        get => _discountRate;
        set => this.RaiseAndSetIfChanged(ref _discountRate, value);
    }
    
    // Derived properties using ObservableAsPropertyHelper
    private readonly ObservableAsPropertyHelper<decimal> _unitPrice;
    public decimal UnitPrice => _unitPrice.Value;
    
    private readonly ObservableAsPropertyHelper<decimal> _totalPrice;
    public decimal TotalPrice => _totalPrice.Value;
    
    private readonly ObservableAsPropertyHelper<string> _totalDisplay;
    public string TotalDisplay => _totalDisplay.Value;
    
    public ProductDetailViewModel()
    {
        // UnitPrice = BasePrice * (1 - DiscountRate) if HasDiscount
        _unitPrice = this.WhenAnyValue(
            x => x.BasePrice,
            x => x.HasDiscount,
            x => x.DiscountRate,
            (price, hasDisc, rate) => hasDisc
                ? price * (1 - (decimal)rate)
                : price)
            .ToProperty(this, x => x.UnitPrice);
        
        // TotalPrice = UnitPrice * Quantity
        _totalPrice = this.WhenAnyValue(
            x => x.UnitPrice,
            x => x.Quantity,
            (unit, qty) => unit * qty)
            .ToProperty(this, x => x.TotalPrice);
        
        // Display
        _totalDisplay = this.WhenAnyValue(x => x.TotalPrice)
            .Select(t => $"รวม: {t:N2} บาท")
            .ToProperty(this, x => x.TotalDisplay);
    }
}
```

---

## Step 379: Reactive Caching

```csharp
// ============================================
// Reactive Cache with Expiry
// ============================================

public class ReactiveCache<TKey, TValue> where TKey : notnull
{
    private readonly TimeSpan _expiry;
    private readonly Dictionary<TKey, (TValue Value, DateTimeOffset ExpiresAt)> _cache = new();
    private readonly Subject<TKey> _evictions = new();
    
    public IObservable<TKey> Evictions => _evictions.AsObservable();
    
    public ReactiveCache(TimeSpan? expiry = null)
    {
        _expiry = expiry ?? TimeSpan.FromMinutes(5);
        
        // Auto-cleanup every minute
        Observable.Interval(TimeSpan.FromMinutes(1))
            .Subscribe(_ => Cleanup());
    }
    
    public TValue? Get(TKey key)
    {
        if (_cache.TryGetValue(key, out var entry))
        {
            if (entry.ExpiresAt > DateTimeOffset.UtcNow)
                return entry.Value;
            
            _cache.Remove(key);
            _evictions.OnNext(key);
        }
        return default;
    }
    
    public void Set(TKey key, TValue value)
    {
        _cache[key] = (value, DateTimeOffset.UtcNow.Add(_expiry));
    }
    
    public IObservable<TValue> GetOrFetchAsync(TKey key, Func<TKey, Task<TValue>> fetch)
    {
        var cached = Get(key);
        if (cached is not null)
            return Observable.Return(cached);
        
        return Observable.FromAsync(() => fetch(key))
            .Do(value => Set(key, value));
    }
    
    private void Cleanup()
    {
        var now = DateTimeOffset.UtcNow;
        var expired = _cache.Where(kv => kv.Value.ExpiresAt <= now).Select(kv => kv.Key).ToList();
        foreach (var key in expired)
        {
            _cache.Remove(key);
            _evictions.OnNext(key);
        }
    }
}
```

---

## Step 380: ReactiveUI + MAUI View

```csharp
// ============================================
// ReactiveUI View binding in MAUI
// ============================================

using ReactiveUI;
using ReactiveUI.Maui;

public partial class SearchPage : ReactiveContentPage<SearchViewModel>
{
    public SearchPage(SearchViewModel vm)
    {
        InitializeComponent();
        ViewModel = vm;
        
        this.WhenActivated(disposables =>
        {
            // Two-way binding
            this.Bind(ViewModel,
                vm => vm.Query,
                v => v.SearchEntry.Text)
                .DisposeWith(disposables);
            
            // One-way binding
            this.OneWayBind(ViewModel,
                vm => vm.IsSearching,
                v => v.LoadingIndicator.IsRunning)
                .DisposeWith(disposables);
            
            // Collection binding
            this.OneWayBind(ViewModel,
                vm => vm.Results,
                v => v.ResultsList.ItemsSource)
                .DisposeWith(disposables);
            
            // Command binding
            this.BindCommand(ViewModel,
                vm => vm.SearchCommand,
                v => v.SearchButton)
                .DisposeWith(disposables);
        });
    }
}
```

---

## สรุป Part 38

ใน Part 38 เราได้เรียนรู้:

1. **Rx.NET Fundamentals** - Observable, Subscriber, Dispose
2. **Subject Types** - Subject, BehaviorSubject, ReplaySubject, AsyncSubject
3. **Operators** - Merge, CombineLatest, Buffer, Scan, GroupBy, Retry
4. **ReactiveUI Setup** - ViewModel, Commands, IsExecuting
5. **Reactive Forms** - Validation with WhenAnyValue
6. **Event Aggregator** - RxEventBus, CompositeDisposable
7. **Polling & Live Data** - Publish().RefCount() shared streams
8. **ObservableAsPropertyHelper** - Derived computed properties
9. **Reactive Cache** - TTL caching with observables
10. **ReactiveUI View** - WhenActivated, Bind, BindCommand

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 38 | Steps 371-380*
