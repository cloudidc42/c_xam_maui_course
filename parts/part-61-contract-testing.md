# Part 61: Contract Testing & API Versioning
## Steps 601-610: Consumer-Driven Contracts, API Versions, Compatibility

---

## Step 601: Contract Test Concepts

```csharp
// ============================================
// Consumer-Driven Contract Testing
// ============================================

/*
 * Consumer-Driven Contract Testing แนวคิด:
 * 
 * Provider (API Server) ←→ Consumer (Mobile App)
 * 
 * 1. Consumer เขียน contract ว่าต้องการ API แบบใด
 * 2. Provider verify ว่า API ของตัวเองตรงกับ contract
 * 3. ทำให้ API breaking changes ถูกตรวจพบทันที
 * 
 * Tools:
 * - Pact.Net: สร้าง pact (contract) files
 * - PactNet: verify provider
 */

// Consumer contract model
public class ProductApiContract
{
    // What the mobile app EXPECTS from the API
    public record GetProductResponse(
        int Id, string Name, string Category,
        decimal Price, string? ImageUrl, bool InStock);
    
    public record GetProductsPageResponse(
        List<GetProductResponse> Items,
        int TotalCount, int Page, int PageSize);
    
    public record CreateOrderRequest(
        int CustomerId, List<OrderLineRequest> Lines,
        string ShippingAddress);
    
    public record OrderLineRequest(int ProductId, int Quantity);
    
    public record CreateOrderResponse(
        int OrderId, string Status, decimal Total,
        DateTime EstimatedDelivery);
}

// Contract test for product API
public class ProductApiContractTests
{
    private readonly string _providerBaseUrl;
    
    public ProductApiContractTests()
        => _providerBaseUrl = "http://localhost:5000";
    
    [Fact]
    public async Task GetProduct_ReturnsExpectedContract()
    {
        using var client = new HttpClient { BaseAddress = new Uri(_providerBaseUrl) };
        
        var response = await client.GetAsync("/api/products/1");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var content = await response.Content.ReadFromJsonAsync<
            ProductApiContract.GetProductResponse>();
        
        // Verify contract shape
        content.Should().NotBeNull();
        content!.Id.Should().BeGreaterThan(0);
        content.Name.Should().NotBeNullOrEmpty();
        content.Category.Should().NotBeNullOrEmpty();
        content.Price.Should().BeGreaterThan(0);
    }
    
    [Fact]
    public async Task GetProducts_ReturnsPagedContract()
    {
        using var client = new HttpClient { BaseAddress = new Uri(_providerBaseUrl) };
        
        var response = await client.GetAsync("/api/products?page=1&pageSize=20");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var content = await response.Content.ReadFromJsonAsync<
            ProductApiContract.GetProductsPageResponse>();
        
        content.Should().NotBeNull();
        content!.Items.Should().NotBeNull();
        content.TotalCount.Should().BeGreaterThanOrEqualTo(0);
        content.Page.Should().Be(1);
        content.PageSize.Should().Be(20);
    }
}
```

---

## Step 602: Mock Server for Consumer Tests

```csharp
// ============================================
// WireMock for Consumer-Side Contract Tests
// ============================================

// Using WireMock.Net to simulate provider
public class ProductApiMockServer : IDisposable
{
    private readonly WireMockServer _server;
    
    public string BaseUrl => _server.Url!;
    
    public ProductApiMockServer()
    {
        _server = WireMockServer.Start();
        SetupStubs();
    }
    
    private void SetupStubs()
    {
        // GET /api/products/1
        _server.Given(
            Request.Create()
                .WithPath("/api/products/1")
                .UsingGet())
            .RespondWith(
                Response.Create()
                    .WithStatusCode(200)
                    .WithHeader("Content-Type", "application/json")
                    .WithBodyAsJson(new
                    {
                        id = 1,
                        name = "เสื้อผ้า",
                        category = "แฟชั่น",
                        price = 299.0m,
                        imageUrl = "https://example.com/shirt.jpg",
                        inStock = true
                    }));
        
        // GET /api/products/999 → 404
        _server.Given(
            Request.Create()
                .WithPath("/api/products/999")
                .UsingGet())
            .RespondWith(
                Response.Create()
                    .WithStatusCode(404)
                    .WithBodyAsJson(new { error = "Product not found" }));
        
        // POST /api/orders
        _server.Given(
            Request.Create()
                .WithPath("/api/orders")
                .UsingPost()
                .WithBody(new JsonMatcher(new { customerId = 1 }, true)))
            .RespondWith(
                Response.Create()
                    .WithStatusCode(201)
                    .WithBodyAsJson(new
                    {
                        orderId = 1001,
                        status = "Confirmed",
                        total = 299.0m,
                        estimatedDelivery = DateTime.Now.AddDays(3)
                    }));
    }
    
    public void Dispose() => _server.Dispose();
}

// Test using mock server
public class ProductServiceContractTests : IClassFixture<ProductApiMockServer>
{
    private readonly ProductApiMockServer _mock;
    
    public ProductServiceContractTests(ProductApiMockServer mock) => _mock = mock;
    
    [Fact]
    public async Task GetProduct_WithMockServer_ReturnsExpected()
    {
        var service = new ProductApiService(
            new HttpClient { BaseAddress = new Uri(_mock.BaseUrl) });
        
        var product = await service.GetByIdAsync(1);
        
        product.Should().NotBeNull();
        product!.Id.Should().Be(1);
        product.Name.Should().Be("เสื้อผ้า");
        product.Price.Should().Be(299.0m);
    }
    
    [Fact]
    public async Task GetProduct_NotFound_ThrowsOrReturnsNull()
    {
        var service = new ProductApiService(
            new HttpClient { BaseAddress = new Uri(_mock.BaseUrl) });
        
        var product = await service.GetByIdAsync(999);
        
        product.Should().BeNull();
    }
}
```

---

## Step 603: API Client with Versioning

```csharp
// ============================================
// API Client Supporting Multiple Versions
// ============================================

public interface IApiVersion
{
    string Version { get; }
    string Prefix { get; }
}

public class ApiV1 : IApiVersion
{
    public string Version => "1.0";
    public string Prefix => "/api/v1";
}

public class ApiV2 : IApiVersion
{
    public string Version => "2.0";
    public string Prefix => "/api/v2";
}

public class VersionedApiClient
{
    private readonly HttpClient _http;
    private readonly IApiVersion _version;
    
    public VersionedApiClient(HttpClient http, IApiVersion version)
    {
        _http = http;
        _version = version;
        _http.DefaultRequestHeaders.Add("Api-Version", version.Version);
    }
    
    public async Task<T?> GetAsync<T>(string path, CancellationToken ct = default)
    {
        var fullPath = $"{_version.Prefix}{path}";
        var response = await _http.GetAsync(fullPath, ct);
        
        if (response.StatusCode == HttpStatusCode.NotFound)
            return default;
        
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
    
    public async Task<TResponse?> PostAsync<TRequest, TResponse>(
        string path, TRequest body, CancellationToken ct = default)
    {
        var fullPath = $"{_version.Prefix}{path}";
        var response = await _http.PostAsJsonAsync(fullPath, body, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<TResponse>(cancellationToken: ct);
    }
}

// Adapter for breaking API changes
public class ProductApiAdapterV2
{
    private readonly VersionedApiClient _client;
    
    public ProductApiAdapterV2(VersionedApiClient client) => _client = client;
    
    // V2 renamed some fields - adapter normalizes
    public async Task<ProductDto?> GetProductAsync(int id)
    {
        var raw = await _client.GetAsync<ProductV2Response>($"/products/{id}");
        if (raw == null) return null;
        
        // V2 uses "title" instead of "name"
        return new ProductDto(raw.Id, raw.Title, raw.Category, raw.BasePrice);
    }
}

public record ProductV2Response(
    int Id, string Title,     // was "Name" in V1
    string Category, decimal BasePrice,  // was "Price" in V1
    bool Available, List<string> Tags);
```

---

## Step 604: Snapshot Testing

```csharp
// ============================================
// Snapshot Testing with Verify
// ============================================

// Using Verify library for snapshot tests
// dotnet add package Verify.Xunit

[UsesVerify]
public class ProductSnapshotTests
{
    [Fact]
    public async Task GetProductList_MatchesSnapshot()
    {
        var products = new List<ProductDto>
        {
            new(1, "เสื้อผ้า", "แฟชั่น", 299m),
            new(2, "รองเท้า", "แฟชั่น", 599m),
            new(3, "กระเป๋า", "กระเป๋า", 999m)
        };
        
        await Verify(products);
        // Creates: ProductSnapshotTests.GetProductList_MatchesSnapshot.verified.json
        // Subsequent runs compare against that snapshot
    }
    
    [Fact]
    public async Task OrderSummary_MatchesSnapshot()
    {
        var order = new
        {
            OrderId = 1001,
            Customer = "สมชาย",
            Lines = new[]
            {
                new { Product = "เสื้อผ้า", Qty = 2, Price = 299m },
                new { Product = "รองเท้า", Qty = 1, Price = 599m }
            },
            Total = 1197m,
            Status = "Confirmed"
        };
        
        await Verify(order);
    }
    
    // Snapshot comparison rules:
    // - First run: CREATES snapshot file (.verified.json)
    // - Later runs: COMPARES with snapshot
    // - To UPDATE: delete snapshot file or run with --accept
    // - Good for: API response shapes, serialized models
}
```

---

## Step 605: API Breaking Change Detection

```csharp
// ============================================
// Backward Compatibility Tests
// ============================================

public class BackwardCompatibilityTests
{
    // Ensure we don't break existing clients by removing fields
    [Fact]
    public void ProductResponse_RequiredFields_AlwaysPresent()
    {
        var response = new ProductApiContract.GetProductResponse(
            1, "Test", "Cat", 100m, null, true);
        
        // These fields MUST always exist (can't be removed)
        response.Id.Should().BeGreaterThan(0);
        response.Name.Should().NotBeNull();
        response.Category.Should().NotBeNull();
        response.Price.Should().BeGreaterThan(0);
        // inStock and imageUrl can be null/optional
    }
    
    [Fact]
    public void OrderRequest_MinimalFields_AlwaysAccepted()
    {
        // V1 clients only send these fields
        var minimalRequest = new ProductApiContract.CreateOrderRequest(
            CustomerId: 1,
            Lines: new List<ProductApiContract.OrderLineRequest>
            {
                new(ProductId: 1, Quantity: 1)
            },
            ShippingAddress: "123 ถนนสุขุมวิท กรุงเทพ");
        
        // Validate minimum contract
        minimalRequest.CustomerId.Should().BeGreaterThan(0);
        minimalRequest.Lines.Should().NotBeEmpty();
        minimalRequest.ShippingAddress.Should().NotBeNullOrEmpty();
    }
    
    // Test that JSON deserialization is lenient (ignores extra fields)
    [Fact]
    public void ProductResponse_ExtraFields_Ignored()
    {
        var json = """
            {
                "id": 1,
                "name": "เสื้อผ้า",
                "category": "แฟชั่น",
                "price": 299.0,
                "inStock": true,
                "newField": "this is a V3 field",
                "anotherNewField": 42
            }
            """;
        
        var options = new System.Text.Json.JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
            UnmappedMemberHandling = System.Text.Json.Serialization.JsonUnmappedMemberHandling.Skip
        };
        
        // Should not throw despite extra fields
        var product = System.Text.Json.JsonSerializer.Deserialize<
            ProductApiContract.GetProductResponse>(json, options);
        
        product.Should().NotBeNull();
        product!.Id.Should().Be(1);
    }
}
```

---

## Step 606: API Client Resilience

```csharp
// ============================================
// Polly Resilience Policies
// ============================================

public class ResilientApiClientFactory
{
    public static HttpClient Create(string baseUrl)
    {
        var retryPolicy = HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt)), // 2, 4, 8 sec
                onRetry: (outcome, timespan, attempt, ctx) =>
                {
                    System.Diagnostics.Debug.WriteLine(
                        $"Retry {attempt} after {timespan.TotalSeconds}s: " +
                        outcome.Exception?.Message ?? outcome.Result?.StatusCode.ToString());
                });
        
        var circuitBreakerPolicy = HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (outcome, duration) =>
                    System.Diagnostics.Debug.WriteLine($"Circuit broken for {duration.TotalSeconds}s"),
                onReset: () =>
                    System.Diagnostics.Debug.WriteLine("Circuit reset"));
        
        var timeoutPolicy = Policy.TimeoutAsync<HttpResponseMessage>(10);
        
        var combined = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy, timeoutPolicy);
        
        var handler = new PolicyHttpMessageHandler(combined)
        {
            InnerHandler = new HttpClientHandler()
        };
        
        return new HttpClient(handler) { BaseAddress = new Uri(baseUrl) };
    }
}

// Service registration with Polly
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddResilientHttpClient(
        this IServiceCollection services, string name, string baseUrl)
    {
        services.AddHttpClient(name, client =>
        {
            client.BaseAddress = new Uri(baseUrl);
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddTransientHttpErrorPolicy(builder =>
            builder.WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i))))
        .AddTransientHttpErrorPolicy(builder =>
            builder.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
        
        return services;
    }
}
```

---

## Step 607: API Contract DTOs

```csharp
// ============================================
// Shared DTOs (Consumer & Provider Agree)
// ============================================

// Shared contract types - both app and API reference these
namespace MyApp.Contracts.V1
{
    // Requests
    public record LoginRequest(string Email, string Password);
    public record RegisterRequest(string Email, string Password, string Name, string Phone);
    public record RefreshTokenRequest(string RefreshToken);
    
    public record CreateOrderRequest(
        List<OrderLineRequest> Lines,
        AddressRequest ShippingAddress,
        string PaymentMethod,
        string? CouponCode = null);
    
    public record OrderLineRequest(int ProductId, int Quantity);
    
    public record AddressRequest(
        string RecipientName, string Phone, string AddressLine,
        string Province, string PostalCode, string Country = "TH");
    
    // Responses
    public record LoginResponse(
        string AccessToken, string RefreshToken,
        DateTime AccessTokenExpiry, UserResponse User);
    
    public record UserResponse(int Id, string Name, string Email, string Role);
    
    public record ProductResponse(
        int Id, string Name, string Description, string Category,
        decimal Price, decimal? OriginalPrice, string? ImageUrl,
        bool InStock, int StockQuantity, List<string> Tags);
    
    public record PagedResponse<T>(
        List<T> Items, int TotalCount, int Page, int PageSize,
        bool HasNextPage, bool HasPreviousPage);
    
    public record OrderResponse(
        int Id, string Status, decimal Total,
        List<OrderLineResponse> Lines, AddressResponse ShippingAddress,
        DateTime CreatedAt, DateTime? EstimatedDelivery,
        string? TrackingNumber);
    
    public record OrderLineResponse(
        int ProductId, string ProductName, decimal UnitPrice,
        int Quantity, decimal LineTotal);
    
    public record AddressResponse(
        string RecipientName, string Phone, string AddressLine,
        string Province, string PostalCode, string Country);
    
    // Error envelope
    public record ErrorResponse(
        string Code, string Message,
        Dictionary<string, List<string>>? Errors = null);
    
    // API Problem Details (RFC 7807)
    public record ProblemDetails(
        string Type, string Title, int Status, string Detail,
        string Instance, Dictionary<string, object>? Extensions = null);
}
```

---

## Step 608: Integration Test Server

```csharp
// ============================================
// WebApplicationFactory Integration Tests
// ============================================

// Test the full stack: Controller → Service → Repository → DB
public class ProductApiIntegrationTests :
    IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ProductApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real DB with in-memory for tests
                services.RemoveAll<SQLiteConnection>();
                services.AddSingleton(_ =>
                    new SQLiteConnection(":memory:"));
            });
        });
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_ReturnsSuccessAndValidData()
    {
        var response = await _client.GetAsync("/api/v1/products?page=1&pageSize=10");
        
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadFromJsonAsync<
            MyApp.Contracts.V1.PagedResponse<MyApp.Contracts.V1.ProductResponse>>();
        
        content.Should().NotBeNull();
        content!.Page.Should().Be(1);
        content.PageSize.Should().Be(10);
    }
    
    [Fact]
    public async Task CreateOrder_WithValidData_Returns201()
    {
        // Arrange: Login first to get token
        var loginResponse = await _client.PostAsJsonAsync(
            "/api/v1/auth/login",
            new MyApp.Contracts.V1.LoginRequest("test@test.com", "Test1234!"));
        
        loginResponse.EnsureSuccessStatusCode();
        
        var loginData = await loginResponse.Content.ReadFromJsonAsync<
            MyApp.Contracts.V1.LoginResponse>();
        
        _client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue(
                "Bearer", loginData!.AccessToken);
        
        // Act
        var orderRequest = new MyApp.Contracts.V1.CreateOrderRequest(
            Lines: new List<MyApp.Contracts.V1.OrderLineRequest>
            {
                new(ProductId: 1, Quantity: 2)
            },
            ShippingAddress: new("สมชาย", "0812345678", "123 ถ.สุขุมวิท", "กรุงเทพ", "10110"),
            PaymentMethod: "promptpay");
        
        var response = await _client.PostAsJsonAsync("/api/v1/orders", orderRequest);
        
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        
        var order = await response.Content.ReadFromJsonAsync<
            MyApp.Contracts.V1.OrderResponse>();
        
        order!.Status.Should().Be("Confirmed");
        order.Total.Should().BeGreaterThan(0);
    }
}
```

---

## Step 609: API Response Validation

```csharp
// ============================================
// Response Validation Middleware
// ============================================

public class ApiResponseValidator
{
    private readonly HttpClient _client;
    
    public ApiResponseValidator(HttpClient client) => _client = client;
    
    public async Task<ValidationResult<T>> GetAndValidateAsync<T>(
        string url, Func<T, ValidationReport> validate)
    {
        HttpResponseMessage response;
        
        try
        {
            response = await _client.GetAsync(url);
        }
        catch (HttpRequestException ex)
        {
            return ValidationResult<T>.NetworkError(ex.Message);
        }
        
        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadFromJsonAsync<
                MyApp.Contracts.V1.ErrorResponse>();
            return ValidationResult<T>.ApiError(
                (int)response.StatusCode, error?.Message ?? "Unknown error");
        }
        
        T? data;
        try
        {
            data = await response.Content.ReadFromJsonAsync<T>();
        }
        catch (Exception ex)
        {
            return ValidationResult<T>.ParseError(ex.Message);
        }
        
        if (data == null)
            return ValidationResult<T>.ParseError("Response was null");
        
        var report = validate(data);
        return report.IsValid
            ? ValidationResult<T>.Success(data)
            : ValidationResult<T>.ContractViolation(report);
    }
}

public class ValidationResult<T>
{
    public bool IsSuccess { get; private init; }
    public T? Data { get; private init; }
    public string? Error { get; private init; }
    public ValidationReport? ContractReport { get; private init; }
    
    public static ValidationResult<T> Success(T data) =>
        new() { IsSuccess = true, Data = data };
    
    public static ValidationResult<T> NetworkError(string msg) =>
        new() { IsSuccess = false, Error = $"Network: {msg}" };
    
    public static ValidationResult<T> ApiError(int status, string msg) =>
        new() { IsSuccess = false, Error = $"HTTP {status}: {msg}" };
    
    public static ValidationResult<T> ParseError(string msg) =>
        new() { IsSuccess = false, Error = $"Parse: {msg}" };
    
    public static ValidationResult<T> ContractViolation(ValidationReport report) =>
        new() { IsSuccess = false, ContractReport = report };
}

public class ValidationReport
{
    private readonly List<string> _violations = new();
    
    public bool IsValid => _violations.Count == 0;
    public IReadOnlyList<string> Violations => _violations.AsReadOnly();
    
    public void Require(bool condition, string violation)
    {
        if (!condition) _violations.Add(violation);
    }
    
    public void RequireNotNull(object? value, string field)
        => Require(value != null, $"{field} must not be null");
    
    public void RequirePositive(decimal value, string field)
        => Require(value > 0, $"{field} must be positive");
    
    public void RequireNotEmpty(string? value, string field)
        => Require(!string.IsNullOrEmpty(value), $"{field} must not be empty");
}
```

---

## Step 610: Contract Test CI

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    types: [opened, synchronize]

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    
    services:
      api:
        image: myapp-api:latest
        ports:
          - 5000:80
        env:
          ASPNETCORE_ENVIRONMENT: Testing
          ConnectionStrings__Default: "Data Source=:memory:"
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x
      
      - name: Wait for API
        run: |
          for i in {1..30}; do
            curl -f http://localhost:5000/health && break
            sleep 2
          done
      
      - name: Run Contract Tests
        run: |
          dotnet test tests/ContractTests \
            --logger trx \
            --results-directory ./contract-results \
            -- ApiBaseUrl=http://localhost:5000
      
      - name: Publish Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Contract Test Results
          path: ./contract-results/*.trx
          reporter: dotnet-trx
```

---

## สรุป Part 61

ใน Part 61 เราได้เรียนรู้:

1. **Contract Test Concepts** - Consumer-driven contracts
2. **Mock Server** - WireMock.Net for consumer tests
3. **API Versioning Client** - IApiVersion, adapters
4. **Snapshot Testing** - Verify library
5. **Breaking Change Detection** - Compatibility tests
6. **Polly Resilience** - Retry, circuit breaker
7. **Shared Contract DTOs** - Request/response types
8. **Integration Tests** - WebApplicationFactory
9. **Response Validation** - ValidationResult<T>
10. **Contract Tests CI** - GitHub Actions workflow

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 61 | Steps 601-610*
