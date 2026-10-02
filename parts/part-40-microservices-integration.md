# Part 40: Microservices Integration
## Steps 391-400: API Gateway, gRPC, GraphQL, Message Queue

---

## Step 391: API Gateway Pattern

```csharp
// ============================================
// API Gateway Client
// ============================================

public class ApiGatewayClient
{
    private readonly HttpClient _http;
    private readonly TokenService _tokens;
    private readonly ILogger<ApiGatewayClient> _logger;
    
    public ApiGatewayClient(HttpClient http, TokenService tokens, ILogger<ApiGatewayClient> logger)
    {
        _http = http;
        _tokens = tokens;
        _logger = logger;
    }
    
    public async Task<ApiResult<T>> GetAsync<T>(
        string path, CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(HttpMethod.Get, path, null, ct);
    }
    
    public async Task<ApiResult<T>> PostAsync<T>(
        string path, object body, CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(HttpMethod.Post, path, body, ct);
    }
    
    public async Task<ApiResult<T>> PutAsync<T>(
        string path, object body, CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(HttpMethod.Put, path, body, ct);
    }
    
    public async Task<ApiResult<T>> DeleteAsync<T>(
        string path, CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(HttpMethod.Delete, path, null, ct);
    }
    
    private async Task<ApiResult<T>> ExecuteAsync<T>(
        HttpMethod method, string path, object? body, CancellationToken ct)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        try
        {
            var request = new HttpRequestMessage(method, path);
            
            if (body != null)
                request.Content = JsonContent.Create(body);
            
            // Attach auth token
            var token = await _tokens.GetAccessTokenAsync(ct);
            if (token != null)
                request.Headers.Authorization = 
                    new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
            
            var response = await _http.SendAsync(request, ct);
            
            _logger.LogInformation("[API] {Method} {Path} => {Status} ({Ms}ms)",
                method, path, (int)response.StatusCode, sw.ElapsedMilliseconds);
            
            if (response.IsSuccessStatusCode)
            {
                var data = await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
                return ApiResult<T>.Success(data!);
            }
            
            var error = await response.Content.ReadAsStringAsync(ct);
            return ApiResult<T>.Failure(error, (int)response.StatusCode);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "[API] {Method} {Path} failed", method, path);
            return ApiResult<T>.Failure(ex.Message, 0);
        }
    }
}

public class ApiResult<T>
{
    public bool IsSuccess { get; private init; }
    public T? Data { get; private init; }
    public string? Error { get; private init; }
    public int StatusCode { get; private init; }
    
    public static ApiResult<T> Success(T data) => new() { IsSuccess = true, Data = data };
    public static ApiResult<T> Failure(string error, int code) =>
        new() { IsSuccess = false, Error = error, StatusCode = code };
    
    public TResult Match<TResult>(
        Func<T, TResult> onSuccess,
        Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(Data!) : onFailure(Error!);
}
```

---

## Step 392: gRPC Client

```csharp
// ============================================
// gRPC in .NET MAUI
// ============================================

// NuGet: Grpc.Net.Client, Google.Protobuf, Grpc.Tools
// product.proto:
/*
syntax = "proto3";
option csharp_namespace = "MyApp.Grpc";

service ProductService {
    rpc GetProduct (GetProductRequest) returns (ProductReply);
    rpc ListProducts (ListProductsRequest) returns (stream ProductReply);
    rpc CreateProduct (CreateProductRequest) returns (ProductReply);
}

message GetProductRequest { int32 id = 1; }
message ListProductsRequest { string category = 1; int32 page = 2; int32 page_size = 3; }
message CreateProductRequest { string name = 1; double price = 2; string category = 3; }
message ProductReply { int32 id = 1; string name = 2; double price = 3; string category = 4; }
*/

public class GrpcProductService
{
    private readonly ProductService.ProductServiceClient _client;
    
    public GrpcProductService(string serverUrl)
    {
        var channel = GrpcChannel.ForAddress(serverUrl, new GrpcChannelOptions
        {
            HttpHandler = new SocketsHttpHandler
            {
                PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
                KeepAlivePingDelay = TimeSpan.FromSeconds(60),
                KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
                EnableMultipleHttp2Connections = true
            }
        });
        
        _client = new ProductService.ProductServiceClient(channel);
    }
    
    public async Task<ProductReply?> GetProductAsync(int id, CancellationToken ct = default)
    {
        try
        {
            return await _client.GetProductAsync(
                new GetProductRequest { Id = id },
                cancellationToken: ct);
        }
        catch (Grpc.Core.RpcException ex) when (ex.StatusCode == Grpc.Core.StatusCode.NotFound)
        {
            return null;
        }
    }
    
    public async IAsyncEnumerable<ProductReply> StreamProductsAsync(
        string category,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        var call = _client.ListProducts(
            new ListProductsRequest { Category = category, Page = 1, PageSize = 20 },
            cancellationToken: ct);
        
        await foreach (var product in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return product;
        }
    }
    
    public async Task<ProductReply> CreateProductAsync(
        string name, decimal price, string category)
    {
        return await _client.CreateProductAsync(new CreateProductRequest
        {
            Name = name,
            Price = (double)price,
            Category = category
        });
    }
}

// ViewModel using gRPC streaming
public partial class ProductStreamViewModel : ObservableObject
{
    private readonly GrpcProductService _grpc;
    
    [ObservableProperty] private ObservableCollection<Product> _products = new();
    [ObservableProperty] private bool _isStreaming;
    
    private CancellationTokenSource? _cts;
    
    public ProductStreamViewModel(GrpcProductService grpc) => _grpc = grpc;
    
    [RelayCommand]
    private async Task StartStreamAsync(string category)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        IsStreaming = true;
        Products.Clear();
        
        try
        {
            await foreach (var reply in _grpc.StreamProductsAsync(category, _cts.Token))
            {
                Products.Add(new Product
                {
                    Id = reply.Id,
                    Name = reply.Name,
                    Price = (decimal)reply.Price
                });
            }
        }
        catch (OperationCanceledException) { /* Normal cancel */ }
        finally
        {
            IsStreaming = false;
        }
    }
    
    [RelayCommand]
    private void StopStream() => _cts?.Cancel();
}
```

---

## Step 393: GraphQL Client

```csharp
// ============================================
// GraphQL Client
// ============================================

// NuGet: StrawberryShake or raw HttpClient

public class GraphQLClient
{
    private readonly HttpClient _http;
    private readonly string _endpoint;
    
    public GraphQLClient(HttpClient http, string endpoint)
    {
        _http = http;
        _endpoint = endpoint;
    }
    
    public async Task<GraphQLResult<T>> QueryAsync<T>(
        string query, object? variables = null, CancellationToken ct = default)
    {
        var request = new { query, variables };
        
        var response = await _http.PostAsJsonAsync(_endpoint, request, ct);
        var result = await response.Content
            .ReadFromJsonAsync<GraphQLResponse<T>>(cancellationToken: ct);
        
        if (result?.Errors?.Count > 0)
            return GraphQLResult<T>.Failure(result.Errors);
        
        return GraphQLResult<T>.Success(result!.Data!);
    }
    
    public async Task<GraphQLResult<T>> MutateAsync<T>(
        string mutation, object variables, CancellationToken ct = default)
    {
        return await QueryAsync<T>(mutation, variables, ct);
    }
    
    // Subscription via WebSocket
    public async IAsyncEnumerable<T> SubscribeAsync<T>(
        string subscription, object? variables = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        var wsUrl = _endpoint.Replace("https://", "wss://").Replace("http://", "ws://");
        using var ws = new System.Net.WebSockets.ClientWebSocket();
        ws.Options.AddSubProtocol("graphql-ws");
        
        await ws.ConnectAsync(new Uri(wsUrl), ct);
        
        // Send init
        await SendWsMessageAsync(ws, new { type = "connection_init" }, ct);
        
        // Send subscription
        await SendWsMessageAsync(ws, new
        {
            type = "start",
            id = "1",
            payload = new { query = subscription, variables }
        }, ct);
        
        var buffer = new byte[4096];
        
        while (!ct.IsCancellationRequested)
        {
            var result = await ws.ReceiveAsync(buffer, ct);
            if (result.MessageType == System.Net.WebSockets.WebSocketMessageType.Close) break;
            
            var json = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count);
            var msg = System.Text.Json.JsonSerializer.Deserialize<WsMessage<T>>(json);
            
            if (msg?.Type == "data" && msg.Payload?.Data != null)
                yield return msg.Payload.Data;
        }
    }
    
    private static async Task SendWsMessageAsync(
        System.Net.WebSockets.ClientWebSocket ws, object message, CancellationToken ct)
    {
        var json = System.Text.Json.JsonSerializer.SerializeToUtf8Bytes(message);
        await ws.SendAsync(json, System.Net.WebSockets.WebSocketMessageType.Text, true, ct);
    }
}

// Queries
public static class ProductQueries
{
    public const string GetProducts = @"
        query GetProducts($category: String, $page: Int!, $pageSize: Int!) {
            products(category: $category, page: $page, pageSize: $pageSize) {
                items {
                    id
                    name
                    price
                    imageUrl
                    category { name }
                }
                totalCount
                hasNextPage
            }
        }";
    
    public const string CreateProduct = @"
        mutation CreateProduct($input: CreateProductInput!) {
            createProduct(input: $input) {
                id
                name
                price
            }
        }";
    
    public const string OnProductUpdated = @"
        subscription OnProductUpdated {
            productUpdated {
                id
                name
                price
                stock
            }
        }";
}
```

---

## Step 394: Message Queue Integration

```csharp
// ============================================
// Outbox Pattern (Message Queue)
// ============================================

public class OutboxMessage
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string MessageType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public string Status { get; set; } = "Pending"; // Pending, Sent, Failed
    public int RetryCount { get; set; }
    public string? Error { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
}

public class OutboxService
{
    private readonly SQLiteConnection _db;
    private readonly IMessagePublisher _publisher;
    
    public OutboxService(SQLiteConnection db, IMessagePublisher publisher)
    {
        _db = db;
        _publisher = publisher;
        _db.CreateTable<OutboxMessage>();
    }
    
    public void QueueMessage<T>(T message) where T : class
    {
        _db.Insert(new OutboxMessage
        {
            MessageType = typeof(T).Name,
            Payload = System.Text.Json.JsonSerializer.Serialize(message),
            CreatedAt = DateTime.UtcNow
        });
    }
    
    public async Task ProcessPendingAsync(CancellationToken ct = default)
    {
        var pending = _db.Table<OutboxMessage>()
            .Where(m => m.Status == "Pending" && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(10)
            .ToList();
        
        foreach (var message in pending)
        {
            if (ct.IsCancellationRequested) break;
            
            try
            {
                await _publisher.PublishAsync(message.MessageType, message.Payload, ct);
                
                message.Status = "Sent";
                message.ProcessedAt = DateTime.UtcNow;
                _db.Update(message);
            }
            catch (Exception ex)
            {
                message.RetryCount++;
                message.Error = ex.Message;
                message.Status = message.RetryCount >= 5 ? "Failed" : "Pending";
                _db.Update(message);
            }
        }
    }
    
    public List<OutboxMessage> GetFailed()
        => _db.Table<OutboxMessage>().Where(m => m.Status == "Failed").ToList();
    
    public void RetryFailed()
    {
        _db.Execute(@"
            UPDATE OutboxMessages SET Status = 'Pending', RetryCount = 0 
            WHERE Status = 'Failed'");
    }
}

public interface IMessagePublisher
{
    Task PublishAsync(string messageType, string payload, CancellationToken ct = default);
}

// HTTP-based publisher (to a backend message broker)
public class HttpMessagePublisher : IMessagePublisher
{
    private readonly HttpClient _http;
    
    public HttpMessagePublisher(HttpClient http) => _http = http;
    
    public async Task PublishAsync(string messageType, string payload, CancellationToken ct = default)
    {
        var request = new { type = messageType, payload };
        var response = await _http.PostAsJsonAsync("api/messages", request, ct);
        response.EnsureSuccessStatusCode();
    }
}
```

---

## Step 395: Service Discovery

```csharp
// ============================================
// Service Registry & Discovery
// ============================================

public class ServiceRegistry
{
    private readonly Dictionary<string, ServiceInfo> _services = new();
    private readonly HttpClient _http;
    
    public ServiceRegistry(HttpClient http) => _http = http;
    
    // Register local service info
    public void Register(string name, string url)
    {
        _services[name] = new ServiceInfo(name, url, DateTime.UtcNow);
    }
    
    // Fetch service list from registry server
    public async Task RefreshFromRegistryAsync(string registryUrl)
    {
        var services = await _http.GetFromJsonAsync<List<ServiceInfo>>(
            $"{registryUrl}/services");
        
        if (services == null) return;
        
        foreach (var svc in services)
            _services[svc.Name] = svc;
    }
    
    public string? GetUrl(string serviceName)
        => _services.TryGetValue(serviceName, out var svc) ? svc.Url : null;
    
    // Health check all services
    public async Task<Dictionary<string, bool>> HealthCheckAllAsync()
    {
        var tasks = _services.ToDictionary(
            kv => kv.Key,
            kv => CheckHealthAsync(kv.Value.Url));
        
        await Task.WhenAll(tasks.Values);
        
        return tasks.ToDictionary(
            kv => kv.Key,
            kv => kv.Value.Result);
    }
    
    private async Task<bool> CheckHealthAsync(string url)
    {
        try
        {
            var response = await _http.GetAsync($"{url}/health");
            return response.IsSuccessStatusCode;
        }
        catch { return false; }
    }
}

public record ServiceInfo(string Name, string Url, DateTime RegisteredAt);
```

---

## Step 396: Saga Pattern

```csharp
// ============================================
// Saga for Distributed Transactions
// ============================================

public interface ISagaStep
{
    string Name { get; }
    Task ExecuteAsync(SagaContext context);
    Task CompensateAsync(SagaContext context);
}

public class SagaContext
{
    public Dictionary<string, object> Data { get; } = new();
    public List<string> CompletedSteps { get; } = new();
    public string? FailedStep { get; set; }
    public Exception? Error { get; set; }
}

public class SagaOrchestrator
{
    private readonly List<ISagaStep> _steps;
    
    public SagaOrchestrator(IEnumerable<ISagaStep> steps)
    {
        _steps = steps.ToList();
    }
    
    public async Task<SagaResult> ExecuteAsync(SagaContext context)
    {
        for (int i = 0; i < _steps.Count; i++)
        {
            var step = _steps[i];
            
            try
            {
                await step.ExecuteAsync(context);
                context.CompletedSteps.Add(step.Name);
            }
            catch (Exception ex)
            {
                context.FailedStep = step.Name;
                context.Error = ex;
                
                // Compensate completed steps in reverse
                await CompensateAsync(context, i - 1);
                
                return SagaResult.Failure(context);
            }
        }
        
        return SagaResult.Success(context);
    }
    
    private async Task CompensateAsync(SagaContext context, int fromIndex)
    {
        for (int i = fromIndex; i >= 0; i--)
        {
            var step = _steps[i];
            try
            {
                await step.CompensateAsync(context);
            }
            catch (Exception ex)
            {
                // Log compensation failure
                Console.WriteLine($"Compensation failed for {step.Name}: {ex.Message}");
            }
        }
    }
}

// Order Saga Steps
public class ReserveInventoryStep : ISagaStep
{
    private readonly IInventoryService _inventory;
    
    public string Name => "ReserveInventory";
    public ReserveInventoryStep(IInventoryService inventory) => _inventory = inventory;
    
    public async Task ExecuteAsync(SagaContext context)
    {
        var orderId = (int)context.Data["OrderId"];
        var items = (List<OrderItem>)context.Data["Items"];
        
        var reservationId = await _inventory.ReserveAsync(orderId, items);
        context.Data["ReservationId"] = reservationId;
    }
    
    public async Task CompensateAsync(SagaContext context)
    {
        if (context.Data.TryGetValue("ReservationId", out var id))
            await _inventory.ReleaseReservationAsync((string)id);
    }
}

public class ChargePaymentStep : ISagaStep
{
    private readonly IPaymentService _payment;
    
    public string Name => "ChargePayment";
    public ChargePaymentStep(IPaymentService payment) => _payment = payment;
    
    public async Task ExecuteAsync(SagaContext context)
    {
        var amount = (decimal)context.Data["TotalAmount"];
        var paymentMethod = (string)context.Data["PaymentMethod"];
        
        var paymentId = await _payment.ChargeAsync(amount, paymentMethod);
        context.Data["PaymentId"] = paymentId;
    }
    
    public async Task CompensateAsync(SagaContext context)
    {
        if (context.Data.TryGetValue("PaymentId", out var id))
            await _payment.RefundAsync((string)id);
    }
}
```

---

## Step 397: Circuit Breaker

```csharp
// ============================================
// Circuit Breaker Pattern
// ============================================

public enum CircuitState { Closed, Open, HalfOpen }

public class CircuitBreaker
{
    private readonly int _failureThreshold;
    private readonly TimeSpan _resetTimeout;
    private readonly TimeSpan _halfOpenTimeout;
    
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount;
    private DateTime _openedAt;
    private readonly object _lock = new();
    
    public CircuitState State => _state;
    
    public CircuitBreaker(
        int failureThreshold = 5,
        TimeSpan? resetTimeout = null,
        TimeSpan? halfOpenTimeout = null)
    {
        _failureThreshold = failureThreshold;
        _resetTimeout = resetTimeout ?? TimeSpan.FromSeconds(30);
        _halfOpenTimeout = halfOpenTimeout ?? TimeSpan.FromSeconds(5);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        lock (_lock)
        {
            switch (_state)
            {
                case CircuitState.Open:
                    if (DateTime.UtcNow - _openedAt > _resetTimeout)
                    {
                        _state = CircuitState.HalfOpen;
                    }
                    else
                    {
                        throw new CircuitBreakerOpenException("Circuit breaker is OPEN");
                    }
                    break;
                    
                case CircuitState.HalfOpen:
                    // Allow one test request
                    break;
            }
        }
        
        try
        {
            var result = await action();
            
            lock (_lock)
            {
                if (_state == CircuitState.HalfOpen)
                {
                    _state = CircuitState.Closed;
                    _failureCount = 0;
                }
            }
            
            return result;
        }
        catch
        {
            lock (_lock)
            {
                _failureCount++;
                
                if (_state == CircuitState.HalfOpen || _failureCount >= _failureThreshold)
                {
                    _state = CircuitState.Open;
                    _openedAt = DateTime.UtcNow;
                }
            }
            throw;
        }
    }
}

public class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}

// Usage wrapper
public class ResilientHttpClient
{
    private readonly HttpClient _http;
    private readonly CircuitBreaker _breaker = new();
    private readonly int _maxRetries = 3;
    
    public ResilientHttpClient(HttpClient http) => _http = http;
    
    public async Task<T?> GetAsync<T>(string url, CancellationToken ct = default)
    {
        for (int attempt = 0; attempt < _maxRetries; attempt++)
        {
            try
            {
                return await _breaker.ExecuteAsync(async () =>
                    await _http.GetFromJsonAsync<T>(url, ct));
            }
            catch (CircuitBreakerOpenException)
            {
                throw; // Don't retry when circuit is open
            }
            catch when (attempt < _maxRetries - 1)
            {
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
            }
        }
        return default;
    }
}
```

---

## Step 398: Health Monitoring

```csharp
// ============================================
// App Health Monitoring
// ============================================

public class HealthCheckService
{
    private readonly List<IHealthCheck> _checks;
    
    public HealthCheckService(IEnumerable<IHealthCheck> checks)
    {
        _checks = checks.ToList();
    }
    
    public async Task<HealthReport> CheckAllAsync(CancellationToken ct = default)
    {
        var results = await Task.WhenAll(_checks.Select(async check =>
        {
            var sw = System.Diagnostics.Stopwatch.StartNew();
            try
            {
                var status = await check.CheckAsync(ct);
                return new HealthCheckResult(check.Name, status, null, sw.Elapsed);
            }
            catch (Exception ex)
            {
                return new HealthCheckResult(check.Name, HealthStatus.Unhealthy, ex.Message, sw.Elapsed);
            }
        }));
        
        var overallStatus = results.All(r => r.Status == HealthStatus.Healthy)
            ? HealthStatus.Healthy
            : results.Any(r => r.Status == HealthStatus.Unhealthy)
                ? HealthStatus.Unhealthy
                : HealthStatus.Degraded;
        
        return new HealthReport(overallStatus, results.ToList());
    }
}

public enum HealthStatus { Healthy, Degraded, Unhealthy }

public interface IHealthCheck
{
    string Name { get; }
    Task<HealthStatus> CheckAsync(CancellationToken ct);
}

public class ApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _http;
    private readonly string _url;
    
    public string Name => "API";
    
    public ApiHealthCheck(HttpClient http, string url)
    {
        _http = http;
        _url = url;
    }
    
    public async Task<HealthStatus> CheckAsync(CancellationToken ct)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(5));
        
        var response = await _http.GetAsync(_url, cts.Token);
        return response.IsSuccessStatusCode ? HealthStatus.Healthy : HealthStatus.Unhealthy;
    }
}

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly SQLiteConnection _db;
    
    public string Name => "Database";
    
    public DatabaseHealthCheck(SQLiteConnection db) => _db = db;
    
    public Task<HealthStatus> CheckAsync(CancellationToken ct)
    {
        try
        {
            _db.ExecuteScalar<int>("SELECT 1");
            return Task.FromResult(HealthStatus.Healthy);
        }
        catch
        {
            return Task.FromResult(HealthStatus.Unhealthy);
        }
    }
}

public record HealthCheckResult(string Name, HealthStatus Status, string? Error, TimeSpan Duration);
public record HealthReport(HealthStatus Overall, List<HealthCheckResult> Results);
```

---

## Step 399: Distributed Tracing

```csharp
// ============================================
// Distributed Tracing (OpenTelemetry)
// ============================================

using System.Diagnostics;

public class TracingService
{
    private static readonly ActivitySource Source = new("MyApp.Mobile", "1.0.0");
    
    public Activity? StartActivity(string name, ActivityKind kind = ActivityKind.Internal)
    {
        return Source.StartActivity(name, kind);
    }
    
    public Activity? StartHttpActivity(string method, string url)
    {
        var activity = Source.StartActivity("HTTP " + method, ActivityKind.Client);
        activity?.SetTag("http.method", method);
        activity?.SetTag("http.url", url);
        return activity;
    }
    
    public Activity? StartDatabaseActivity(string operation, string table)
    {
        var activity = Source.StartActivity("DB " + operation, ActivityKind.Client);
        activity?.SetTag("db.operation", operation);
        activity?.SetTag("db.table", table);
        return activity;
    }
    
    // Propagate trace context in HTTP headers
    public void InjectTraceContext(HttpRequestMessage request, Activity? activity)
    {
        if (activity == null) return;
        
        request.Headers.TryAddWithoutValidation(
            "traceparent", $"00-{activity.TraceId}-{activity.SpanId}-01");
        
        if (!string.IsNullOrEmpty(activity.TraceStateString))
            request.Headers.TryAddWithoutValidation("tracestate", activity.TraceStateString);
    }
}

// Traced HTTP handler
public class TracingHandler : DelegatingHandler
{
    private readonly TracingService _tracing;
    
    public TracingHandler(TracingService tracing) => _tracing = tracing;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        using var activity = _tracing.StartHttpActivity(
            request.Method.Method,
            request.RequestUri?.ToString() ?? "");
        
        _tracing.InjectTraceContext(request, activity);
        
        try
        {
            var response = await base.SendAsync(request, ct);
            activity?.SetTag("http.status_code", (int)response.StatusCode);
            
            if (!response.IsSuccessStatusCode)
                activity?.SetStatus(ActivityStatusCode.Error, response.ReasonPhrase);
            
            return response;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

---

## Step 400: API Versioning

```csharp
// ============================================
// API Versioning Strategy
// ============================================

public class VersionedApiClient
{
    private readonly HttpClient _http;
    private readonly string _baseUrl;
    private string _version = "v1";
    
    public VersionedApiClient(HttpClient http, string baseUrl)
    {
        _http = http;
        _baseUrl = baseUrl;
    }
    
    public VersionedApiClient UseVersion(string version)
    {
        _version = version;
        return this;
    }
    
    // URL versioning: /api/v1/products
    public Task<T?> GetAsync<T>(string endpoint, CancellationToken ct = default)
        => _http.GetFromJsonAsync<T>($"{_baseUrl}/api/{_version}/{endpoint}", ct);
    
    // Header versioning
    public async Task<T?> GetWithHeaderVersionAsync<T>(
        string endpoint, CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, $"{_baseUrl}/api/{endpoint}");
        request.Headers.Add("API-Version", _version);
        var response = await _http.SendAsync(request, ct);
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
    
    // Feature negotiation
    public async Task<ApiCapabilities> GetServerCapabilitiesAsync()
    {
        return await _http.GetFromJsonAsync<ApiCapabilities>(
            $"{_baseUrl}/api/capabilities") ?? new ApiCapabilities();
    }
}

public class ApiCapabilities
{
    public string MinVersion { get; set; } = "v1";
    public string MaxVersion { get; set; } = "v1";
    public List<string> SupportedFeatures { get; set; } = new();
}

// Version compatibility
public class VersionCompatibilityService
{
    private readonly ApiCapabilities _serverCaps;
    private const string ClientVersion = "v2";
    
    public VersionCompatibilityService(ApiCapabilities serverCaps)
    {
        _serverCaps = serverCaps;
    }
    
    public string GetBestVersion()
    {
        var clientVersionNum = ParseVersion(ClientVersion);
        var serverMaxNum = ParseVersion(_serverCaps.MaxVersion);
        var serverMinNum = ParseVersion(_serverCaps.MinVersion);
        
        if (clientVersionNum < serverMinNum)
            throw new InvalidOperationException("App ล้าสมัยเกินไป กรุณาอัปเดต");
        
        return clientVersionNum <= serverMaxNum
            ? ClientVersion
            : _serverCaps.MaxVersion;
    }
    
    private static int ParseVersion(string version)
    {
        return int.TryParse(version.TrimStart('v'), out var n) ? n : 1;
    }
}
```

---

## สรุป Part 40

ใน Part 40 เราได้เรียนรู้:

1. **API Gateway** - Central client with logging & auth
2. **gRPC** - Streaming, typed contracts
3. **GraphQL** - Query/mutation/subscription
4. **Message Queue** - Outbox pattern
5. **Service Discovery** - Dynamic endpoint resolution
6. **Saga Pattern** - Distributed transaction with compensation
7. **Circuit Breaker** - Resilient API calls
8. **Health Monitoring** - Composite health checks
9. **Distributed Tracing** - OpenTelemetry/ActivitySource
10. **API Versioning** - URL/header versioning, capability negotiation

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 40 | Steps 391-400*
