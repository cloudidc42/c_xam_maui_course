# Part 85: GraphQL Integration
## Steps 841-850: GraphQL Queries, Mutations, Subscriptions, Caching

---

## Step 841: GraphQL Client Setup

```csharp
// ============================================
// StrawberryShake / Kiota GraphQL Client
// ============================================

// Install: dotnet add package StrawberryShake.Transport.Http
// OR use Refit-based approach shown below

// Schema-first approach using raw HttpClient
public class GraphQlClient
{
    private readonly HttpClient _http;
    
    public GraphQlClient(HttpClient http) => _http = http;
    
    public async Task<T?> QueryAsync<T>(string query, object? variables = null)
    {
        var request = new
        {
            query,
            variables = variables ?? new { }
        };
        
        var response = await _http.PostAsJsonAsync("/graphql", request);
        response.EnsureSuccessStatusCode();
        
        var result = await response.Content.ReadFromJsonAsync<GraphQlResponse<T>>();
        
        if (result?.Errors?.Any() == true)
            throw new GraphQlException(result.Errors);
        
        return result?.Data;
    }
    
    public async Task<T?> MutateAsync<T>(string mutation, object? variables = null)
        => await QueryAsync<T>(mutation, variables);
}

public record GraphQlResponse<T>(T? Data, List<GraphQlError>? Errors);
public record GraphQlError(string Message, List<GraphQlLocation>? Locations, List<string>? Path);
public record GraphQlLocation(int Line, int Column);

public class GraphQlException : Exception
{
    public List<GraphQlError> Errors { get; }
    public GraphQlException(List<GraphQlError> errors)
        : base(string.Join("; ", errors.Select(e => e.Message)))
        => Errors = errors;
}
```

---

## Step 842: Restaurant Queries

```csharp
// ============================================
// Typed GraphQL Queries
// ============================================

public static class RestaurantQueries
{
    public const string GetNearby = @"
        query GetNearbyRestaurants($lat: Float!, $lng: Float!, $radius: Int!) {
            nearbyRestaurants(lat: $lat, lng: $lng, radius: $radius) {
                id
                name
                rating
                deliveryFee
                estimatedMinutes
                categories
                thumbnail
                isOpen
            }
        }";
    
    public const string GetById = @"
        query GetRestaurant($id: ID!) {
            restaurant(id: $id) {
                id
                name
                description
                rating
                reviewCount
                menu {
                    categories {
                        name
                        items {
                            id
                            name
                            description
                            price
                            image
                            tags
                            isAvailable
                            options {
                                id
                                name
                                required
                                choices { id name price }
                            }
                        }
                    }
                }
            }
        }";
    
    public const string SearchRestaurants = @"
        query SearchRestaurants($query: String!, $filters: RestaurantFilters) {
            searchRestaurants(query: $query, filters: $filters) {
                totalCount
                items {
                    id
                    name
                    rating
                    thumbnail
                    tags
                }
            }
        }";
}

// Restaurant repository using GraphQL
public class GraphQlRestaurantRepository
{
    private readonly GraphQlClient _client;
    private readonly IMemoryCache _cache;
    
    public GraphQlRestaurantRepository(GraphQlClient client, IMemoryCache cache)
    {
        _client = client;
        _cache = cache;
    }
    
    public async Task<List<RestaurantSummary>> GetNearbyAsync(
        double lat, double lng, int radiusMeters = 5000)
    {
        var cacheKey = $"nearby:{lat:F4}:{lng:F4}:{radiusMeters}";
        
        if (_cache.TryGetValue(cacheKey, out List<RestaurantSummary>? cached))
            return cached!;
        
        var result = await _client.QueryAsync<NearbyRestaurantsData>(
            RestaurantQueries.GetNearby,
            new { lat, lng, radius = radiusMeters });
        
        var restaurants = result?.NearbyRestaurants ?? new List<RestaurantSummary>();
        _cache.Set(cacheKey, restaurants, TimeSpan.FromMinutes(5));
        
        return restaurants;
    }
}

public record NearbyRestaurantsData(List<RestaurantSummary> NearbyRestaurants);
public record RestaurantSummary(
    string Id, string Name, float Rating, int DeliveryFee,
    int EstimatedMinutes, List<string> Categories, string Thumbnail, bool IsOpen);
```

---

## Step 843: GraphQL Mutations

```csharp
// ============================================
// Order Mutations
// ============================================

public static class OrderMutations
{
    public const string PlaceOrder = @"
        mutation PlaceOrder($input: PlaceOrderInput!) {
            placeOrder(input: $input) {
                orderId
                status
                estimatedMinutes
                total
                paymentUrl
            }
        }";
    
    public const string CancelOrder = @"
        mutation CancelOrder($orderId: ID!, $reason: String) {
            cancelOrder(orderId: $orderId, reason: $reason) {
                success
                refundAmount
                refundEstimatedDays
            }
        }";
    
    public const string RateOrder = @"
        mutation RateOrder($orderId: ID!, $rating: Int!, $comment: String) {
            rateOrder(orderId: $orderId, rating: $rating, comment: $comment) {
                success
                reviewId
            }
        }";
}

public class OrderGraphQlService
{
    private readonly GraphQlClient _client;
    private readonly IEventBus _eventBus;
    
    public OrderGraphQlService(GraphQlClient client, IEventBus eventBus)
    {
        _client = client;
        _eventBus = eventBus;
    }
    
    public async Task<PlaceOrderResult> PlaceOrderAsync(PlaceOrderInput input)
    {
        var result = await _client.MutateAsync<PlaceOrderData>(
            OrderMutations.PlaceOrder, new { input });
        
        if (result?.PlaceOrder == null)
            throw new InvalidOperationException("PlaceOrder returned null");
        
        _eventBus.Publish(new OrderPlacedEvent(result.PlaceOrder.OrderId));
        
        return result.PlaceOrder;
    }
}

public record PlaceOrderInput(
    string RestaurantId,
    List<OrderItemInput> Items,
    string DeliveryAddressId,
    string PaymentMethodId,
    string? PromoCode);

public record OrderItemInput(string MenuItemId, int Quantity, List<string>? SelectedOptions);
public record PlaceOrderData(PlaceOrderResult PlaceOrder);
public record PlaceOrderResult(
    string OrderId, string Status, int EstimatedMinutes, decimal Total, string? PaymentUrl);
```

---

## Step 844: GraphQL Subscriptions (WebSocket)

```csharp
// ============================================
// GraphQL Subscriptions over WebSocket
// ============================================

public class GraphQlSubscriptionClient : IAsyncDisposable
{
    private readonly ClientWebSocket _ws;
    private readonly string _url;
    private CancellationTokenSource? _cts;
    
    public GraphQlSubscriptionClient(string wsUrl)
    {
        _ws = new ClientWebSocket();
        _url = wsUrl;
        _ws.Options.AddSubProtocol("graphql-transport-ws");
    }
    
    public async Task ConnectAsync()
    {
        await _ws.ConnectAsync(new Uri(_url), CancellationToken.None);
        
        // Send connection_init
        await SendAsync(new { type = "connection_init", payload = new { } });
        
        // Wait for connection_ack
        var ack = await ReceiveAsync();
        if (ack?["type"]?.ToString() != "connection_ack")
            throw new InvalidOperationException("GraphQL WS handshake failed");
        
        _cts = new CancellationTokenSource();
        _ = PingLoopAsync(_cts.Token);
    }
    
    public async IAsyncEnumerable<T?> SubscribeAsync<T>(
        string subscriptionId, string query, object? variables = null)
    {
        await SendAsync(new
        {
            id = subscriptionId,
            type = "subscribe",
            payload = new { query, variables = variables ?? new { } }
        });
        
        while (_ws.State == WebSocketState.Open)
        {
            var message = await ReceiveAsync();
            if (message == null) continue;
            
            var type = message["type"]?.ToString();
            var id = message["id"]?.ToString();
            
            if (id != subscriptionId) continue;
            
            if (type == "complete") yield break;
            
            if (type == "next")
            {
                var payload = message["payload"];
                var data = payload?["data"]?.Deserialize<T>();
                yield return data;
            }
            
            if (type == "error")
                throw new GraphQlException(
                    message["payload"]?.Deserialize<List<GraphQlError>>() ?? new());
        }
    }
    
    private async Task PingLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(30_000, ct);
            await SendAsync(new { type = "ping" });
        }
    }
    
    private async Task SendAsync(object payload)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(payload);
        var bytes = System.Text.Encoding.UTF8.GetBytes(json);
        await _ws.SendAsync(bytes, WebSocketMessageType.Text, true, CancellationToken.None);
    }
    
    private async Task<System.Text.Json.Nodes.JsonObject?> ReceiveAsync()
    {
        var buffer = new byte[4096];
        var result = await _ws.ReceiveAsync(buffer, CancellationToken.None);
        var json = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count);
        return System.Text.Json.Nodes.JsonNode.Parse(json) as System.Text.Json.Nodes.JsonObject;
    }
    
    public async ValueTask DisposeAsync()
    {
        _cts?.Cancel();
        if (_ws.State == WebSocketState.Open)
            await _ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Client closing", CancellationToken.None);
        _ws.Dispose();
    }
}

// Live order tracking via GraphQL subscription
public class LiveOrderService
{
    private readonly GraphQlSubscriptionClient _sub;
    
    public event EventHandler<OrderStatusUpdate>? StatusChanged;
    
    public async Task WatchOrderAsync(string orderId)
    {
        const string query = @"
            subscription WatchOrder($orderId: ID!) {
                orderStatusChanged(orderId: $orderId) {
                    orderId
                    status
                    message
                    estimatedMinutes
                    riderLocation { lat lng heading }
                }
            }";
        
        await foreach (var update in _sub.SubscribeAsync<OrderUpdateData>(
            orderId, query, new { orderId }))
        {
            if (update?.OrderStatusChanged != null)
                StatusChanged?.Invoke(this, update.OrderStatusChanged);
        }
    }
}
```

---

## Step 845: GraphQL Cache

```csharp
// ============================================
// Normalized GraphQL Cache
// ============================================

public class GraphQlNormalizedCache
{
    // Cache by type+id: "Restaurant:abc123" → {id, name, rating, ...}
    private readonly ConcurrentDictionary<string, System.Text.Json.Nodes.JsonObject> _store = new();
    
    public void Merge(string typeName, System.Text.Json.Nodes.JsonObject entity)
    {
        var id = entity["id"]?.ToString();
        if (id == null) return;
        
        var key = $"{typeName}:{id}";
        
        _store.AddOrUpdate(key, entity, (_, existing) =>
        {
            // Merge: new values win
            foreach (var prop in entity)
                existing[prop.Key] = prop.Value?.DeepClone();
            return existing;
        });
    }
    
    public T? Get<T>(string typeName, string id) where T : class
    {
        var key = $"{typeName}:{id}";
        if (_store.TryGetValue(key, out var obj))
            return obj.Deserialize<T>();
        return null;
    }
    
    public void Invalidate(string typeName, string id)
        => _store.TryRemove($"{typeName}:{id}", out _);
    
    public void InvalidateAll(string typeName)
    {
        var keys = _store.Keys.Where(k => k.StartsWith($"{typeName}:")).ToList();
        foreach (var key in keys) _store.TryRemove(key, out _);
    }
}

// Optimistic update pattern
public class OptimisticCartService
{
    private readonly GraphQlClient _client;
    private readonly GraphQlNormalizedCache _cache;
    
    public async Task<bool> AddItemAsync(string menuItemId, int quantity)
    {
        // Apply optimistic update immediately
        var optimisticCartItem = new System.Text.Json.Nodes.JsonObject
        {
            ["id"] = $"temp-{menuItemId}",
            ["menuItemId"] = menuItemId,
            ["quantity"] = quantity,
            ["isPending"] = true
        };
        _cache.Merge("CartItem", optimisticCartItem);
        
        const string mutation = @"
            mutation AddToCart($menuItemId: ID!, $quantity: Int!) {
                addToCart(menuItemId: $menuItemId, quantity: $quantity) {
                    id menuItemId quantity
                }
            }";
        
        try
        {
            var result = await _client.MutateAsync<AddToCartData>(
                mutation, new { menuItemId, quantity });
            
            // Replace optimistic with real
            _cache.Invalidate("CartItem", $"temp-{menuItemId}");
            if (result?.AddToCart != null)
                _cache.Merge("CartItem",
                    System.Text.Json.JsonSerializer.SerializeToNode(result.AddToCart)!.AsObject());
            
            return true;
        }
        catch
        {
            // Rollback optimistic update
            _cache.Invalidate("CartItem", $"temp-{menuItemId}");
            return false;
        }
    }
}
```

---

## Step 846: Query Batching

```csharp
// ============================================
// Request Batching (reduce round trips)
// ============================================

public class BatchingGraphQlClient
{
    private readonly HttpClient _http;
    private readonly List<PendingQuery> _queue = new();
    private readonly SemaphoreSlim _lock = new(1, 1);
    private Timer? _flushTimer;
    private const int WindowMs = 50;
    
    public BatchingGraphQlClient(HttpClient http) => _http = http;
    
    public Task<T?> QueryAsync<T>(string query, object? variables = null)
    {
        var tcs = new TaskCompletionSource<T?>();
        
        _lock.Wait();
        try
        {
            _queue.Add(new PendingQuery(
                Query: query,
                Variables: variables,
                Resolve: data => tcs.SetResult(
                    data == null ? default : System.Text.Json.JsonSerializer.Deserialize<T>(data.ToString()!)),
                Reject: ex => tcs.SetException(ex)));
            
            // Debounce flush
            _flushTimer?.Dispose();
            _flushTimer = new Timer(_ => _ = FlushAsync(), null, WindowMs, Timeout.Infinite);
        }
        finally { _lock.Release(); }
        
        return tcs.Task;
    }
    
    private async Task FlushAsync()
    {
        List<PendingQuery> batch;
        
        await _lock.WaitAsync();
        try
        {
            batch = new List<PendingQuery>(_queue);
            _queue.Clear();
        }
        finally { _lock.Release(); }
        
        if (!batch.Any()) return;
        
        var requests = batch.Select((q, i) => new
        {
            id = i.ToString(),
            query = q.Query,
            variables = q.Variables ?? new { }
        }).ToArray();
        
        var response = await _http.PostAsJsonAsync("/graphql/batch", requests);
        var results = await response.Content
            .ReadFromJsonAsync<List<BatchResult>>();
        
        for (int i = 0; i < batch.Count && i < (results?.Count ?? 0); i++)
        {
            var res = results![i];
            if (res.Errors?.Any() == true)
                batch[i].Reject(new GraphQlException(res.Errors));
            else
                batch[i].Resolve(res.Data);
        }
    }
    
    private record PendingQuery(
        string Query, object? Variables,
        Action<object?> Resolve, Action<Exception> Reject);
    
    private record BatchResult(object? Data, List<GraphQlError>? Errors);
}
```

---

## Step 847: Pagination with Cursors

```csharp
// ============================================
// Relay-style Cursor Pagination
// ============================================

public record Connection<T>(List<Edge<T>> Edges, PageInfo PageInfo, int TotalCount);
public record Edge<T>(T Node, string Cursor);
public record PageInfo(bool HasNextPage, bool HasPreviousPage, string? StartCursor, string? EndCursor);

public class PaginatedRestaurantService
{
    private readonly GraphQlClient _client;
    
    public PaginatedRestaurantService(GraphQlClient client) => _client = client;
    
    private const string GetRestaurantsQuery = @"
        query GetRestaurants($first: Int!, $after: String, $filter: RestaurantFilter) {
            restaurants(first: $first, after: $after, filter: $filter) {
                totalCount
                pageInfo {
                    hasNextPage
                    endCursor
                }
                edges {
                    cursor
                    node {
                        id name rating thumbnail
                    }
                }
            }
        }";
    
    public async IAsyncEnumerable<List<RestaurantSummary>> GetAllPagesAsync(
        int pageSize = 20, object? filter = null)
    {
        string? cursor = null;
        
        while (true)
        {
            var result = await _client.QueryAsync<RestaurantsData>(
                GetRestaurantsQuery,
                new { first = pageSize, after = cursor, filter });
            
            var page = result?.Restaurants;
            if (page == null || !page.Edges.Any()) yield break;
            
            yield return page.Edges.Select(e => e.Node).ToList();
            
            if (!page.PageInfo.HasNextPage) yield break;
            cursor = page.PageInfo.EndCursor;
        }
    }
    
    // Infinite scroll ViewModel
    public async Task LoadMoreAsync(
        ObservableRangeCollection<RestaurantSummary> items,
        ref bool hasMore, ref string? cursor)
    {
        if (!hasMore) return;
        
        var result = await _client.QueryAsync<RestaurantsData>(
            GetRestaurantsQuery, new { first = 20, after = cursor });
        
        if (result?.Restaurants == null) return;
        
        items.AddRange(result.Restaurants.Edges.Select(e => e.Node));
        hasMore = result.Restaurants.PageInfo.HasNextPage;
        cursor = result.Restaurants.PageInfo.EndCursor;
    }
}

public record RestaurantsData(Connection<RestaurantSummary> Restaurants);
```

---

## Step 848: Fragments & Directives

```csharp
// ============================================
// GraphQL Fragments for DRY Queries
// ============================================

public static class GraphQlFragments
{
    public const string RestaurantFields = @"
        fragment RestaurantFields on Restaurant {
            id name rating thumbnail isOpen
            categories estimatedMinutes deliveryFee
        }";
    
    public const string MenuItemFields = @"
        fragment MenuItemFields on MenuItem {
            id name description price image
            isAvailable tags
            options { id name required choices { id name price } }
        }";
    
    // Query using fragments
    public const string HomePageQuery = @"
        query HomePage($lat: Float!, $lng: Float!) {
            featured: nearbyRestaurants(lat: $lat, lng: $lng, radius: 3000, limit: 5) {
                ...RestaurantFields
            }
            nearby: nearbyRestaurants(lat: $lat, lng: $lng, radius: 5000, limit: 20) {
                ...RestaurantFields
            }
        }
        " + RestaurantFields;
    
    // @include / @skip directives
    public const string ConditionalQuery = @"
        query GetUser($id: ID!, $withOrders: Boolean!, $withProfile: Boolean = true) {
            user(id: $id) {
                id name email
                profile @include(if: $withProfile) {
                    avatar bio preferences
                }
                orders @include(if: $withOrders) {
                    id status total createdAt
                }
            }
        }";
}

public class FragmentQueryBuilder
{
    private readonly List<string> _fragments = new();
    private string _query = "";
    
    public FragmentQueryBuilder WithQuery(string query) { _query = query; return this; }
    
    public FragmentQueryBuilder WithFragment(string fragment) { _fragments.Add(fragment); return this; }
    
    public string Build() => _query + "\n" + string.Join("\n", _fragments);
}
```

---

## Step 849: Error Handling

```csharp
// ============================================
// GraphQL Error Handling Strategies
// ============================================

public class GraphQlErrorHandler
{
    private readonly ILogger<GraphQlErrorHandler> _logger;
    
    public GraphQlErrorHandler(ILogger<GraphQlErrorHandler> logger) => _logger = logger;
    
    public T HandleResult<T>(GraphQlResponse<T> response)
    {
        if (response.Errors?.Any() != true)
            return response.Data ?? throw new InvalidOperationException("Null data");
        
        // Classify errors
        foreach (var error in response.Errors)
        {
            var extensions = error.Message; // simplified
            
            switch (GetErrorCategory(error))
            {
                case GraphQlErrorCategory.Authentication:
                    throw new UnauthorizedException(error.Message);
                
                case GraphQlErrorCategory.NotFound:
                    throw new NotFoundException(error.Message);
                
                case GraphQlErrorCategory.Validation:
                    throw new ValidationException(error.Message);
                
                default:
                    _logger.LogError("GraphQL error: {Message}", error.Message);
                    break;
            }
        }
        
        // Partial data with errors: return what we have
        if (response.Data != null)
        {
            _logger.LogWarning("Partial GraphQL response with {Count} errors", response.Errors.Count);
            return response.Data;
        }
        
        throw new GraphQlException(response.Errors);
    }
    
    private GraphQlErrorCategory GetErrorCategory(GraphQlError error) =>
        error.Message.Contains("UNAUTHENTICATED") ? GraphQlErrorCategory.Authentication :
        error.Message.Contains("NOT_FOUND") ? GraphQlErrorCategory.NotFound :
        error.Message.Contains("BAD_USER_INPUT") ? GraphQlErrorCategory.Validation :
        GraphQlErrorCategory.Unknown;
}

public enum GraphQlErrorCategory { Authentication, NotFound, Validation, Unknown }
```

---

## Step 850: Testing GraphQL Clients

```csharp
// ============================================
// Testing GraphQL Queries with Mock Server
// ============================================

public class FakeGraphQlClient : GraphQlClient
{
    private readonly Dictionary<string, object> _responses = new();
    
    public FakeGraphQlClient() : base(null!) { }
    
    public void SetupResponse<T>(string queryKey, T response)
        => _responses[queryKey] = response!;
    
    public new Task<T?> QueryAsync<T>(string query, object? variables = null)
    {
        var key = ExtractOperationName(query);
        if (_responses.TryGetValue(key, out var resp))
            return Task.FromResult((T?)resp);
        throw new InvalidOperationException($"No mock for query: {key}");
    }
    
    private string ExtractOperationName(string query)
    {
        var match = System.Text.RegularExpressions.Regex.Match(
            query, @"(query|mutation|subscription)\s+(\w+)");
        return match.Success ? match.Groups[2].Value : query[..20];
    }
}

[TestFixture]
public class GraphQlRestaurantTests
{
    [Test]
    public async Task GetNearby_ReturnsRestaurants()
    {
        var fake = new FakeGraphQlClient();
        fake.SetupResponse("GetNearbyRestaurants", new NearbyRestaurantsData(
            new List<RestaurantSummary>
            {
                new("r1", "ครัวไทย", 4.5f, 30, 25, new() { "Thai" }, "thumb.jpg", true)
            }));
        
        var repo = new GraphQlRestaurantRepository(fake, new MemoryCache(new MemoryCacheOptions()));
        var result = await repo.GetNearbyAsync(13.7563, 100.5018);
        
        Assert.That(result.Count, Is.EqualTo(1));
        Assert.That(result[0].Name, Is.EqualTo("ครัวไทย"));
    }
}
```

---

## สรุป Part 85

ใน Part 85 เราได้เรียนรู้:

1. **GraphQL Client Setup** - Raw HTTP client, error model, response wrapping
2. **Typed Queries** - Const query strings, typed response records
3. **Mutations** - PlaceOrder, Cancel, Rate with domain events
4. **Subscriptions** - WebSocket `graphql-transport-ws`, async enumerable
5. **Normalized Cache** - Type:id keys, merge strategy, optimistic updates
6. **Request Batching** - 50ms debounce window, bulk HTTP request
7. **Cursor Pagination** - Relay-style edges/nodes, infinite scroll
8. **Fragments & Directives** - Reusable fragments, @include/@skip
9. **Error Handling** - Error category classification, partial data
10. **Testing** - FakeGraphQlClient, operation name extraction

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 85 | Steps 841-850*
