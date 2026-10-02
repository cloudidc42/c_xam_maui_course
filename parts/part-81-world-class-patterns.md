# Part 81: World-Class Engineering Practices
## Steps 801-810: Zero-Downtime, Chaos Engineering, Multi-Region, SLOs

---

## Step 801: SLO/SLI/SLA Framework

```csharp
// ============================================
// Service Level Objectives
// ============================================

public class SloTracker
{
    private readonly SQLiteAsyncConnection _db;
    
    // Define SLOs
    private static readonly Slo[] _slos =
    {
        new("api_latency_p99", "API latency P99", 2000, "ms", SloType.Latency),
        new("crash_free_rate", "Crash-free rate", 99.9, "%", SloType.ErrorRate),
        new("checkout_success", "Checkout success rate", 99.5, "%", SloType.ErrorRate),
        new("sync_success", "Sync success rate", 99.0, "%", SloType.ErrorRate),
        new("app_startup", "App startup time", 3000, "ms", SloType.Latency),
    };
    
    public SloTracker(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<SloMeasurement>().Wait();
    }
    
    public async Task RecordMeasurementAsync(string sloId, double value)
    {
        await _db.InsertAsync(new SloMeasurement
        {
            SloId = sloId,
            Value = value,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public async Task<SloReport> GenerateReportAsync(TimeSpan window)
    {
        var since = DateTime.UtcNow - window;
        var measurements = await _db.Table<SloMeasurement>()
            .Where(m => m.Timestamp >= since)
            .ToListAsync();
        
        var results = _slos.Select(slo =>
        {
            var data = measurements.Where(m => m.SloId == slo.Id).ToList();
            if (!data.Any()) return new SloResult(slo, null, SloStatus.Insufficient);
            
            var compliance = slo.Type == SloType.Latency
                ? data.Count(m => m.Value <= slo.Target) / (double)data.Count * 100
                : data.Average(m => m.Value);
            
            var status = compliance >= slo.Target ? SloStatus.Met : SloStatus.Breached;
            return new SloResult(slo, compliance, status);
        }).ToList();
        
        return new SloReport(results, since, DateTime.UtcNow);
    }
}

public record Slo(string Id, string Name, double Target, string Unit, SloType Type);
public enum SloType { Latency, ErrorRate }
public enum SloStatus { Met, Breached, Insufficient }
public record SloResult(Slo Slo, double? Compliance, SloStatus Status);
public record SloReport(List<SloResult> Results, DateTime Start, DateTime End);

[Table("slo_measurements")]
public class SloMeasurement
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string SloId { get; set; } = "";
    public double Value { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## Step 802: Error Budget

```csharp
// ============================================
// Error Budget Management
// ============================================

public class ErrorBudgetService
{
    private const double ApiSuccessTarget = 0.999; // 99.9%
    private const double MonthlyMinutes = 43200;
    
    public ErrorBudget CalculateApiErrorBudget(
        int totalRequests, int failedRequests, TimeSpan window)
    {
        var allowedFailureRate = 1 - ApiSuccessTarget;
        var allowedFailures = (int)(totalRequests * allowedFailureRate);
        var actualFailures = failedRequests;
        
        var budgetUsedPercent = allowedFailures == 0 ? 100
            : (double)actualFailures / allowedFailures * 100;
        
        return new ErrorBudget
        {
            TotalRequests = totalRequests,
            FailedRequests = actualFailures,
            AllowedFailures = allowedFailures,
            RemainingFailures = Math.Max(0, allowedFailures - actualFailures),
            BudgetUsedPercent = Math.Min(100, budgetUsedPercent),
            IsExhausted = actualFailures > allowedFailures,
            Window = window
        };
    }
    
    public double CalculateDowntimeBudgetMinutes()
        => MonthlyMinutes * (1 - ApiSuccessTarget); // ~43 minutes/month for 99.9%
    
    public string GetBurnRate(ErrorBudget budget, TimeSpan elapsed)
    {
        var expectedBurnRate = elapsed.TotalHours / (30 * 24); // fraction of month elapsed
        var actualBurnRate = budget.BudgetUsedPercent / 100;
        
        var multiplier = expectedBurnRate == 0 ? 0 : actualBurnRate / expectedBurnRate;
        
        return multiplier switch
        {
            > 10 => $"🔴 Burn rate {multiplier:F1}x – ดำเนินการด่วน!",
            > 3 => $"🟡 Burn rate {multiplier:F1}x – ตรวจสอบ",
            > 1 => $"🟠 Burn rate {multiplier:F1}x – เฝ้าระวัง",
            _ => $"🟢 Burn rate {multiplier:F1}x – ปกติ"
        };
    }
}

public class ErrorBudget
{
    public int TotalRequests { get; set; }
    public int FailedRequests { get; set; }
    public int AllowedFailures { get; set; }
    public int RemainingFailures { get; set; }
    public double BudgetUsedPercent { get; set; }
    public bool IsExhausted { get; set; }
    public TimeSpan Window { get; set; }
}
```

---

## Step 803: Chaos Engineering

```csharp
// ============================================
// Chaos Monkey for Mobile
// ============================================

#if DEBUG || CHAOS_TESTING
public class ChaosMonkey
{
    private readonly Random _rng = new();
    private bool _enabled;
    
    // Scenarios
    private static readonly ChaosScenario[] _scenarios =
    {
        new("network_failure", 0.05, "API calls fail 5% of the time"),
        new("slow_network", 0.10, "API calls take 3x longer 10% of the time"),
        new("db_failure", 0.02, "Database operations fail 2% of the time"),
        new("memory_pressure", 0.01, "Simulate low memory warning"),
    };
    
    public void Enable() => _enabled = true;
    public void Disable() => _enabled = false;
    
    public async Task<T> ExecuteAsync<T>(string operation, Func<Task<T>> action)
    {
        if (!_enabled) return await action();
        
        var scenario = _scenarios.FirstOrDefault(s =>
            _rng.NextDouble() < s.Probability);
        
        if (scenario == null) return await action();
        
        return scenario.Id switch
        {
            "network_failure" => throw new HttpRequestException($"[Chaos] {scenario.Description}"),
            "slow_network" => await SlowAsync(action),
            "db_failure" => throw new SQLiteException($"[Chaos] {scenario.Description}", 1),
            _ => await action()
        };
    }
    
    private async Task<T> SlowAsync<T>(Func<Task<T>> action)
    {
        await Task.Delay(TimeSpan.FromSeconds(3 * _rng.Next(1, 4)));
        return await action();
    }
}

public record ChaosScenario(string Id, double Probability, string Description);
#endif
```

---

## Step 804: Circuit Breaker

```csharp
// ============================================
// Circuit Breaker Pattern
// ============================================

public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount;
    private DateTime _openedAt;
    
    private readonly int _failureThreshold;
    private readonly TimeSpan _recoveryTimeout;
    
    public CircuitBreaker(int failureThreshold = 5,
        TimeSpan? recoveryTimeout = null)
    {
        _failureThreshold = failureThreshold;
        _recoveryTimeout = recoveryTimeout ?? TimeSpan.FromMinutes(1);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (_state == CircuitState.Open)
        {
            if (DateTime.UtcNow - _openedAt < _recoveryTimeout)
                throw new CircuitBreakerOpenException("Circuit is open – refusing request");
            
            _state = CircuitState.HalfOpen; // Try one request
        }
        
        try
        {
            var result = await action();
            OnSuccess();
            return result;
        }
        catch (Exception ex) when (IsTransient(ex))
        {
            OnFailure();
            throw;
        }
    }
    
    private void OnSuccess()
    {
        _failureCount = 0;
        _state = CircuitState.Closed;
    }
    
    private void OnFailure()
    {
        _failureCount++;
        if (_failureCount >= _failureThreshold)
        {
            _state = CircuitState.Open;
            _openedAt = DateTime.UtcNow;
        }
    }
    
    private static bool IsTransient(Exception ex)
        => ex is HttpRequestException or TimeoutException or TaskCanceledException;
    
    public CircuitState State => _state;
}

public enum CircuitState { Closed, Open, HalfOpen }
public class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}
```

---

## Step 805: Rate Limiting

```csharp
// ============================================
// Client-Side Rate Limiting (Token Bucket)
// ============================================

public class TokenBucketRateLimiter
{
    private double _tokens;
    private DateTime _lastRefill;
    private readonly double _capacity;
    private readonly double _refillRate; // tokens per second
    private readonly SemaphoreSlim _lock = new(1, 1);
    
    public TokenBucketRateLimiter(double capacity, double refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;
        _tokens = capacity;
        _lastRefill = DateTime.UtcNow;
    }
    
    public async Task<bool> TryAcquireAsync(double cost = 1)
    {
        await _lock.WaitAsync();
        try
        {
            Refill();
            if (_tokens >= cost)
            {
                _tokens -= cost;
                return true;
            }
            return false;
        }
        finally
        {
            _lock.Release();
        }
    }
    
    public async Task WaitAsync(double cost = 1, CancellationToken ct = default)
    {
        while (true)
        {
            if (await TryAcquireAsync(cost)) return;
            await Task.Delay(100, ct);
        }
    }
    
    private void Refill()
    {
        var now = DateTime.UtcNow;
        var elapsed = (now - _lastRefill).TotalSeconds;
        _tokens = Math.Min(_capacity, _tokens + elapsed * _refillRate);
        _lastRefill = now;
    }
}

// API client with rate limiting
public class RateLimitedApiClient
{
    private readonly HttpClient _client;
    private readonly TokenBucketRateLimiter _limiter;
    private readonly CircuitBreaker _breaker;
    
    public RateLimitedApiClient(HttpClient client)
    {
        _client = client;
        _limiter = new TokenBucketRateLimiter(capacity: 10, refillRate: 2); // 10 burst, 2/s
        _breaker = new CircuitBreaker(failureThreshold: 5);
    }
    
    public async Task<T?> GetAsync<T>(string url, CancellationToken ct = default)
    {
        await _limiter.WaitAsync(ct: ct);
        
        return await _breaker.ExecuteAsync(async () =>
        {
            var response = await _client.GetFromJsonAsync<T>(url, ct);
            return response;
        });
    }
}
```

---

## Step 806: Distributed Tracing

```csharp
// ============================================
// Distributed Tracing with Correlation IDs
// ============================================

public class DistributedTraceContext
{
    private static readonly AsyncLocal<TraceContext?> _current = new();
    
    public static TraceContext? Current => _current.Value;
    
    public static TraceContext Start(string? parentTraceId = null)
    {
        var context = new TraceContext(
            TraceId: parentTraceId ?? Guid.NewGuid().ToString("N"),
            SpanId: Guid.NewGuid().ToString("N")[..8],
            ParentSpanId: null,
            StartedAt: DateTime.UtcNow);
        
        _current.Value = context;
        return context;
    }
    
    public static TraceContext StartChildSpan(string name)
    {
        var parent = Current;
        var child = new TraceContext(
            TraceId: parent?.TraceId ?? Guid.NewGuid().ToString("N"),
            SpanId: Guid.NewGuid().ToString("N")[..8],
            ParentSpanId: parent?.SpanId,
            StartedAt: DateTime.UtcNow);
        
        _current.Value = child;
        return child;
    }
}

public record TraceContext(
    string TraceId,
    string SpanId,
    string? ParentSpanId,
    DateTime StartedAt);

// Propagating trace through HTTP
public class TracingHttpHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var ctx = DistributedTraceContext.Current;
        if (ctx != null)
        {
            request.Headers.Add("X-Trace-Id", ctx.TraceId);
            request.Headers.Add("X-Span-Id", ctx.SpanId);
            if (ctx.ParentSpanId != null)
                request.Headers.Add("X-Parent-Span-Id", ctx.ParentSpanId);
        }
        
        return await base.SendAsync(request, ct);
    }
}
```

---

## Step 807: Feature Experiment Framework

```csharp
// ============================================
// Structured Feature Experiments
// ============================================

public class ExperimentFramework
{
    private readonly CanaryReleaseService _canary;
    private readonly UserAnalyticsService _analytics;
    
    public ExperimentFramework(CanaryReleaseService canary, UserAnalyticsService analytics)
    {
        _canary = canary;
        _analytics = analytics;
    }
    
    public async Task<T> RunAsync<T>(Experiment<T> experiment, string userId)
    {
        var isInTreatment = await _canary.IsInCanaryGroupAsync(
            experiment.Id, userId, experiment.TreatmentPercent);
        
        var variant = isInTreatment ? "treatment" : "control";
        
        // Track exposure
        WeakReferenceMessenger.Default.Send(new ExperimentExposure(
            experiment.Id, userId, variant));
        
        // Execute appropriate variant
        var result = isInTreatment
            ? await experiment.TreatmentFunc()
            : await experiment.ControlFunc();
        
        return result;
    }
}

public class Experiment<T>
{
    public string Id { get; init; } = "";
    public string Description { get; init; } = "";
    public int TreatmentPercent { get; init; } = 50;
    public Func<Task<T>> ControlFunc { get; init; } = null!;
    public Func<Task<T>> TreatmentFunc { get; init; } = null!;
    public Metric[] Metrics { get; init; } = Array.Empty<Metric>();
}

public record Metric(string Name, MetricType Type, double SuccessThreshold);
public enum MetricType { Conversion, Retention, Revenue }
public record ExperimentExposure(string ExperimentId, string UserId, string Variant);
```

---

## Step 808: Observability Stack

```csharp
// ============================================
// Full Observability: Metrics + Traces + Logs
// ============================================

public class ObservabilityStack : IDisposable
{
    private readonly ILogger _logger;
    private readonly Meter _meter;
    private readonly ActivitySource _tracer;
    
    // Metrics
    private readonly Counter<int> _requestCount;
    private readonly Histogram<double> _requestDuration;
    private readonly ObservableGauge<int> _activeUsers;
    private int _activeUserCount;
    
    public ObservabilityStack(string serviceName, string version)
    {
        _logger = LoggerFactory.Create(b => b.AddConsole())
            .CreateLogger(serviceName);
        _meter = new Meter(serviceName, version);
        _tracer = new ActivitySource(serviceName, version);
        
        _requestCount = _meter.CreateCounter<int>("app.requests");
        _requestDuration = _meter.CreateHistogram<double>("app.request.duration", "ms");
        _activeUsers = _meter.CreateObservableGauge("app.active_users",
            () => _activeUserCount);
    }
    
    public async Task<T> InstrumentAsync<T>(
        string operation, Dictionary<string, string>? tags, Func<Task<T>> action)
    {
        using var activity = _tracer.StartActivity(operation);
        
        if (tags != null)
            foreach (var (k, v) in tags)
                activity?.SetTag(k, v);
        
        var sw = Stopwatch.StartNew();
        
        try
        {
            var result = await action();
            
            _requestCount.Add(1, new TagList { { "operation", operation }, { "status", "success" } });
            _requestDuration.Record(sw.Elapsed.TotalMilliseconds,
                new TagList { { "operation", operation } });
            
            _logger.LogInformation("[{Op}] Completed in {Ms}ms",
                operation, sw.ElapsedMilliseconds);
            
            return result;
        }
        catch (Exception ex)
        {
            _requestCount.Add(1, new TagList { { "operation", operation }, { "status", "error" } });
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            
            _logger.LogError(ex, "[{Op}] Failed after {Ms}ms",
                operation, sw.ElapsedMilliseconds);
            
            throw;
        }
    }
    
    public void SetActiveUsers(int count) => _activeUserCount = count;
    
    public void Dispose()
    {
        _meter.Dispose();
        _tracer.Dispose();
    }
}
```

---

## Step 809: Zero-Downtime Migration

```csharp
// ============================================
// Zero-Downtime Database Schema Migration
// ============================================

public class ZeroDowntimeMigrator
{
    private readonly SQLiteAsyncConnection _db;
    
    public ZeroDowntimeMigrator(SQLiteAsyncConnection db) => _db = db;
    
    // Strategy: Expand → Migrate → Contract
    // Step 1: Add new nullable column (non-breaking)
    public async Task ExpandAsync()
    {
        try
        {
            await _db.ExecuteAsync(
                "ALTER TABLE Orders ADD COLUMN NewStatusCode TEXT");
        }
        catch (SQLiteException ex) when (ex.Message.Contains("duplicate column"))
        {
            // Already migrated – idempotent
        }
    }
    
    // Step 2: Dual-write old and new columns during transition
    // Step 3: Backfill new column from old
    public async Task BackfillAsync(int batchSize = 500)
    {
        int offset = 0;
        while (true)
        {
            var rows = await _db.QueryAsync<OrderMigrationRow>(
                @"SELECT Id, Status FROM Orders 
                  WHERE NewStatusCode IS NULL 
                  LIMIT ? OFFSET ?", batchSize, offset);
            
            if (!rows.Any()) break;
            
            await _db.RunInTransactionAsync(conn =>
            {
                foreach (var row in rows)
                    conn.Execute(
                        "UPDATE Orders SET NewStatusCode = ? WHERE Id = ?",
                        MapStatus(row.Status), row.Id);
            });
            
            offset += batchSize;
            await Task.Delay(10); // Gentle pace – don't block reads
        }
    }
    
    // Step 4: Drop old column after all code uses new (next release)
    public Task ContractAsync()
        => _db.ExecuteAsync("ALTER TABLE Orders DROP COLUMN Status");
    
    private static string MapStatus(string old) => old switch
    {
        "0" => "Pending",
        "1" => "Confirmed",
        "2" => "Delivering",
        "3" => "Delivered",
        _ => old
    };
    
    public class OrderMigrationRow
    {
        public int Id { get; set; }
        public string Status { get; set; } = "";
    }
}
```

---

## Step 810: Production Runbook

```csharp
// ============================================
// Automated Production Runbook
// ============================================

public class ProductionRunbook
{
    private readonly DiagnosticService _diagnostics;
    private readonly SloTracker _slos;
    private readonly ErrorBudgetService _errorBudget;
    
    public ProductionRunbook(DiagnosticService diagnostics,
        SloTracker slos, ErrorBudgetService errorBudget)
    {
        _diagnostics = diagnostics;
        _slos = slos;
        _errorBudget = errorBudget;
    }
    
    // Run when on-call paged
    public async Task<RunbookReport> InvestigateAsync()
    {
        var report = new RunbookReport { GeneratedAt = DateTime.UtcNow };
        
        // 1. Health check
        var health = await _diagnostics.GenerateReportAsync();
        report.HealthStatus = health;
        
        // 2. SLO status
        var sloReport = await _slos.GenerateReportAsync(TimeSpan.FromHours(1));
        report.BreachedSlos = sloReport.Results
            .Where(r => r.Status == SloStatus.Breached).ToList();
        
        // 3. Suggested actions
        report.SuggestedActions = GenerateSuggestedActions(report);
        
        return report;
    }
    
    private List<string> GenerateSuggestedActions(RunbookReport report)
    {
        var actions = new List<string>();
        
        if (report.BreachedSlos.Any(s => s.Slo.Id == "api_latency_p99"))
        {
            actions.Add("1. ตรวจสอบ API gateway logs สำหรับ slow queries");
            actions.Add("2. ตรวจสอบ database query plan ด้วย EXPLAIN");
            actions.Add("3. พิจารณาเปิด circuit breaker");
        }
        
        if (report.BreachedSlos.Any(s => s.Slo.Id == "crash_free_rate"))
        {
            actions.Add("1. ตรวจสอบ Crashlytics/Sentry สำหรับ crash stack traces");
            actions.Add("2. ตรวจสอบ app version distribution");
            actions.Add("3. พิจารณา force update ถ้า crash รุนแรง");
        }
        
        return actions;
    }
}

public class RunbookReport
{
    public DateTime GeneratedAt { get; set; }
    public object? HealthStatus { get; set; }
    public List<SloResult> BreachedSlos { get; set; } = new();
    public List<string> SuggestedActions { get; set; } = new();
    
    public string Format()
    {
        var sb = new StringBuilder();
        sb.AppendLine($"=== Runbook Report: {GeneratedAt:u} ===");
        sb.AppendLine($"Breached SLOs: {BreachedSlos.Count}");
        foreach (var slo in BreachedSlos)
            sb.AppendLine($"  ❌ {slo.Slo.Name}: {slo.Compliance:F1}/{slo.Slo.Target}{slo.Slo.Unit}");
        sb.AppendLine("\nSuggested Actions:");
        foreach (var action in SuggestedActions)
            sb.AppendLine($"  {action}");
        return sb.ToString();
    }
}
```

---

## สรุป Part 81

ใน Part 81 เราได้เรียนรู้ระดับ World-Class:

1. **SLO/SLI Framework** - Define, track, report service level objectives
2. **Error Budget** - Calculate burn rate, exhaustion, monthly downtime budget
3. **Chaos Engineering** - Chaos Monkey for mobile: network/DB/memory failures
4. **Circuit Breaker** - Open/HalfOpen/Closed states with recovery timeout
5. **Rate Limiting** - Token bucket algorithm, burst + sustained rate
6. **Distributed Tracing** - Correlation IDs, HTTP header propagation
7. **A/B Experiment Framework** - Structured variant assignment and tracking
8. **Observability Stack** - Metrics + Traces + Logs unified
9. **Zero-Downtime Migration** - Expand-Migrate-Contract database strategy
10. **Production Runbook** - Automated incident investigation and suggested actions

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 81 | Steps 801-810*
