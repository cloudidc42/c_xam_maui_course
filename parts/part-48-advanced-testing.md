# Part 48: Advanced Testing
## Steps 471-480: Unit Tests, Integration Tests, UI Tests

---

## Step 471: Unit Testing with xUnit

```csharp
// ============================================
// xUnit Unit Tests
// ============================================

// Project setup:
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
// dotnet add package Moq
// dotnet add package FluentAssertions
// dotnet add package Microsoft.NET.Test.Sdk

using Xunit;
using FluentAssertions;
using Moq;

// Test naming: MethodName_Scenario_ExpectedBehavior
public class MoneyTests
{
    [Fact]
    public void Add_SameCurrency_ReturnsSum()
    {
        var m1 = new Money(100, "THB");
        var m2 = new Money(200, "THB");
        
        var result = m1.Add(m2);
        
        result.Amount.Should().Be(300);
        result.Currency.Should().Be("THB");
    }
    
    [Fact]
    public void Add_DifferentCurrency_ThrowsException()
    {
        var thb = new Money(100, "THB");
        var usd = new Money(3, "USD");
        
        var act = () => thb.Add(usd);
        
        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*currencies*");
    }
    
    [Theory]
    [InlineData(100, 1, 100)]
    [InlineData(100, 3, 300)]
    [InlineData(50.5, 2, 101)]
    public void Multiply_ValidQuantity_ReturnsCorrectAmount(
        decimal amount, int qty, decimal expected)
    {
        var money = new Money(amount, "THB");
        
        var result = money.Multiply(qty);
        
        result.Amount.Should().Be(expected);
    }
    
    [Fact]
    public void Zero_AnyCall_ReturnsZeroAmount()
    {
        var zero = Money.Zero("THB");
        
        zero.Amount.Should().Be(0);
        zero.IsZero.Should().BeTrue();
    }
}

public class OrderTests
{
    [Fact]
    public void AddLine_NewProduct_IncreasesLineCount()
    {
        var order = Order.Create(1, TestData.Address);
        
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 2);
        
        order.Lines.Should().HaveCount(1);
        order.TotalAmount.Amount.Should().Be(200);
    }
    
    [Fact]
    public void AddLine_SameProduct_IncreasesQuantity()
    {
        var order = Order.Create(1, TestData.Address);
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 2);
        
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 3);
        
        order.Lines.Should().HaveCount(1);
        order.Lines[0].Quantity.Should().Be(5);
        order.TotalAmount.Amount.Should().Be(500);
    }
    
    [Fact]
    public void Confirm_EmptyOrder_ThrowsDomainException()
    {
        var order = Order.Create(1, TestData.Address);
        
        var act = () => order.Confirm();
        
        act.Should().Throw<OrderDomainException>()
           .WithMessage("*สินค้า*");
    }
    
    [Fact]
    public void Confirm_ValidOrder_RaisesOrderConfirmedEvent()
    {
        var order = Order.Create(1, TestData.Address);
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 1);
        
        order.Confirm();
        
        order.DomainEvents.Should().ContainSingle(e => e is OrderConfirmedEvent);
        order.Status.Should().Be(OrderStatus.Confirmed);
    }
    
    [Fact]
    public void Ship_AfterConfirm_StatusBecomesShipped()
    {
        var order = CreateConfirmedOrder();
        
        order.Ship("TH123456789");
        
        order.Status.Should().Be(OrderStatus.Shipped);
        order.ShippedAt.Should().NotBeNull();
        order.DomainEvents.Should().Contain(e => e is OrderShippedEvent);
    }
    
    private static Order CreateConfirmedOrder()
    {
        var order = Order.Create(1, TestData.Address);
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 1);
        order.Confirm();
        order.ClearDomainEvents();
        return order;
    }
}

public static class TestData
{
    public static Address Address => new("123 ถนนสุขุมวิท", "กรุงเทพฯ", "กรุงเทพมหานคร", "10110");
}
```

---

## Step 472: Mocking with Moq

```csharp
// ============================================
// Moq - Advanced Mocking
// ============================================

public class OrderApplicationServiceTests
{
    private readonly Mock<IOrderPort> _ordersMock = new();
    private readonly Mock<IPaymentPort> _paymentMock = new();
    private readonly Mock<INotificationPort> _notificationsMock = new();
    private readonly Mock<IDomainEventDispatcher> _eventsMock = new();
    
    private OrderApplicationService CreateService() =>
        new(_ordersMock.Object, _paymentMock.Object,
            _notificationsMock.Object, _eventsMock.Object);
    
    [Fact]
    public async Task PlaceOrderAsync_ValidOrder_ReturnsSuccess()
    {
        // Arrange
        _paymentMock
            .Setup(p => p.ChargeAsync(It.IsAny<int>(), It.IsAny<decimal>(), It.IsAny<PaymentMethod>()))
            .ReturnsAsync(new PaymentResult { IsSuccess = true, PaymentId = "pay_123" });
        
        _ordersMock
            .Setup(o => o.SaveOrderAsync(It.IsAny<Order>()))
            .Returns(Task.CompletedTask);
        
        var command = new PlaceOrderCommand(
            CustomerId: 1,
            ShippingAddress: TestData.Address,
            Items: [new OrderItemDto(1, "สินค้า", 100m, 2)],
            PaymentMethod: PaymentMethod.Card);
        
        // Act
        var result = await CreateService().PlaceOrderAsync(command);
        
        // Assert
        result.IsSuccess.Should().BeTrue();
        
        _paymentMock.Verify(
            p => p.ChargeAsync(It.IsAny<int>(), 200m, PaymentMethod.Card),
            Times.Once);
        
        _ordersMock.Verify(
            o => o.SaveOrderAsync(It.IsAny<Order>()),
            Times.Once);
    }
    
    [Fact]
    public async Task PlaceOrderAsync_PaymentFailed_ReturnsFailure()
    {
        _paymentMock
            .Setup(p => p.ChargeAsync(It.IsAny<int>(), It.IsAny<decimal>(), It.IsAny<PaymentMethod>()))
            .ReturnsAsync(new PaymentResult { IsSuccess = false, Error = "บัตรถูกปฏิเสธ" });
        
        var command = new PlaceOrderCommand(1, TestData.Address,
            [new(1, "สินค้า", 100m, 1)], PaymentMethod.Card);
        
        var result = await CreateService().PlaceOrderAsync(command);
        
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("บัตรถูกปฏิเสธ");
        
        _ordersMock.Verify(o => o.SaveOrderAsync(It.IsAny<Order>()), Times.Never);
    }
    
    [Fact]
    public async Task PlaceOrderAsync_EmptyItems_ReturnsFailure()
    {
        var command = new PlaceOrderCommand(1, TestData.Address, [], PaymentMethod.Card);
        
        var result = await CreateService().PlaceOrderAsync(command);
        
        result.IsSuccess.Should().BeFalse();
        _paymentMock.VerifyNoOtherCalls();
    }
}

// Testing ViewModels
public class ProductListViewModelTests
{
    [Fact]
    public async Task LoadAsync_Success_PopulatesProducts()
    {
        var mockService = new Mock<IProductService>();
        mockService
            .Setup(s => s.GetAllAsync())
            .ReturnsAsync([
                new Product { Id = 1, Name = "สินค้า A", Price = 100 },
                new Product { Id = 2, Name = "สินค้า B", Price = 200 }
            ]);
        
        var vm = new ProductListViewModel(mockService.Object);
        
        await vm.LoadCommand.ExecuteAsync(null);
        
        vm.Products.Should().HaveCount(2);
        vm.IsLoading.Should().BeFalse();
        vm.ErrorMessage.Should().BeNull();
    }
    
    [Fact]
    public async Task LoadAsync_Exception_SetsErrorMessage()
    {
        var mockService = new Mock<IProductService>();
        mockService
            .Setup(s => s.GetAllAsync())
            .ThrowsAsync(new HttpRequestException("Network error"));
        
        var vm = new ProductListViewModel(mockService.Object);
        
        await vm.LoadCommand.ExecuteAsync(null);
        
        vm.Products.Should().BeEmpty();
        vm.ErrorMessage.Should().NotBeNull();
    }
}
```

---

## Step 473: Integration Tests

```csharp
// ============================================
// Integration Tests with SQLite In-Memory
// ============================================

public class OrderRepositoryIntegrationTests : IDisposable
{
    private readonly SQLiteConnection _db;
    private readonly SQLiteOrderAdapter _repo;
    
    public OrderRepositoryIntegrationTests()
    {
        _db = new SQLiteConnection(":memory:");
        _db.CreateTable<OrderData>();
        _db.CreateTable<OrderLineData>();
        _repo = new SQLiteOrderAdapter(_db);
    }
    
    [Fact]
    public async Task SaveAndGet_ValidOrder_Roundtrips()
    {
        var order = Order.Create(1, TestData.Address);
        order.AddLine(1, new ProductName("สินค้า"), Money.Thai(100), 2);
        order.Confirm();
        
        await _repo.SaveOrderAsync(order);
        var loaded = await _repo.GetOrderAsync(order.Id);
        
        loaded.Should().NotBeNull();
        loaded!.CustomerId.Should().Be(1);
        loaded.Status.Should().Be(OrderStatus.Confirmed);
        loaded.Lines.Should().HaveCount(1);
        loaded.TotalAmount.Amount.Should().Be(200);
    }
    
    [Fact]
    public async Task GetCustomerOrders_MultipleOrders_ReturnsAll()
    {
        for (int i = 0; i < 3; i++)
        {
            var order = Order.Create(customerId: 42, TestData.Address);
            order.AddLine(i + 1, new ProductName($"สินค้า {i}"), Money.Thai(100), 1);
            await _repo.SaveOrderAsync(order);
        }
        
        var orders = await _repo.GetCustomerOrdersAsync(42);
        
        orders.Should().HaveCount(3);
    }
    
    public void Dispose() => _db.Dispose();
}

// HTTP Integration Test
public class ProductApiIntegrationTests : IClassFixture<ApiTestFactory>
{
    private readonly HttpClient _client;
    
    public ProductApiIntegrationTests(ApiTestFactory factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_Returns200WithProducts()
    {
        var response = await _client.GetAsync("/api/products");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var products = await response.Content
            .ReadFromJsonAsync<List<ProductDto>>();
        
        products.Should().NotBeNull();
        products!.Should().NotBeEmpty();
    }
    
    [Fact]
    public async Task CreateProduct_ValidData_Returns201()
    {
        var command = new { Name = "สินค้าใหม่", Price = 299.99, CategoryId = 1 };
        
        var response = await _client.PostAsJsonAsync("/api/products", command);
        
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

---

## Step 474: UI Tests with Playwright

```csharp
// ============================================
// Playwright UI Tests (MAUI Blazor Hybrid)
// ============================================

// dotnet add package Microsoft.Playwright.NUnit

using Microsoft.Playwright.NUnit;

[TestFixture]
public class CheckoutFlowTests : PageTest
{
    [Test]
    public async Task AddToCart_ProductPage_CartBadgeUpdates()
    {
        await Page.GotoAsync("http://localhost:5000/products");
        
        // Wait for products to load
        await Page.WaitForSelectorAsync("[data-testid='product-card']");
        
        // Click add to cart
        await Page.ClickAsync("[data-testid='add-to-cart-btn']:first-child");
        
        // Assert badge updated
        var badge = await Page.TextContentAsync("[data-testid='cart-badge']");
        badge.Should().Be("1");
    }
    
    [Test]
    public async Task Checkout_ValidForm_ShowsConfirmation()
    {
        await Page.GotoAsync("http://localhost:5000/checkout");
        
        // Fill form
        await Page.FillAsync("#firstname", "สมชาย");
        await Page.FillAsync("#lastname", "ใจดี");
        await Page.FillAsync("#address", "123 ถนนสุขุมวิท");
        await Page.FillAsync("#phone", "0812345678");
        
        // Select payment
        await Page.ClickAsync("[data-testid='payment-card']");
        await Page.FillAsync("#card-number", "4111111111111111");
        await Page.FillAsync("#card-expiry", "12/28");
        await Page.FillAsync("#card-cvv", "123");
        
        // Submit
        await Page.ClickAsync("[data-testid='place-order-btn']");
        
        // Assert confirmation page
        await Page.WaitForURLAsync("**/order-confirmation/**");
        
        var heading = await Page.TextContentAsync("h1");
        heading.Should().Contain("ขอบคุณสำหรับการสั่งซื้อ");
    }
    
    [Test]
    public async Task Search_QueryWithResults_DisplaysProducts()
    {
        await Page.GotoAsync("http://localhost:5000/products");
        
        await Page.FillAsync("[data-testid='search-input']", "เสื้อ");
        await Page.Keyboard.PressAsync("Enter");
        
        await Page.WaitForSelectorAsync("[data-testid='product-card']");
        
        var cards = await Page.QuerySelectorAllAsync("[data-testid='product-card']");
        cards.Count.Should().BeGreaterThan(0);
    }
}
```

---

## Step 475: Snapshot Testing

```csharp
// ============================================
// Snapshot Testing for UI Components
// ============================================

// dotnet add package Verify.Xunit

using VerifyXunit;

[UsesVerify]
public class ProductCardSnapshotTests
{
    [Fact]
    public async Task ProductCard_Renders_MatchesSnapshot()
    {
        var product = new Product
        {
            Id = 1,
            Name = "สินค้าทดสอบ",
            Price = 299.99m,
            Category = "Electronics",
            Stock = 10
        };
        
        var vm = new ProductCardViewModel(product);
        var rendered = RenderViewModel(vm);
        
        await Verify(rendered);
    }
    
    private static object RenderViewModel(ProductCardViewModel vm)
    {
        return new
        {
            vm.Name,
            vm.PriceDisplay,
            vm.IsInStock,
            vm.StockStatus
        };
    }
}

// Usage: first run creates .verified.txt files
// Subsequent runs compare against them

// ViewModel snapshot test
public class OrderSummarySnapshotTests
{
    [Fact]
    public async Task OrderSummary_MultipleItems_MatchesSnapshot()
    {
        var order = Order.Create(1, TestData.Address);
        order.AddLine(1, new ProductName("เสื้อ"), Money.Thai(299), 2);
        order.AddLine(2, new ProductName("กางเกง"), Money.Thai(499), 1);
        order.Confirm();
        
        var summary = new
        {
            LineCount = order.Lines.Count,
            Total = order.TotalAmount.ToString(),
            Status = order.Status.ToString(),
            Lines = order.Lines.Select(l => new
            {
                l.Name.Value,
                l.Quantity,
                Price = l.UnitPrice.ToString(),
                LineTotal = l.LineTotal.ToString()
            })
        };
        
        await Verify(summary);
    }
}
```

---

## Step 476: Load Testing

```csharp
// ============================================
// Simple Load Testing
// ============================================

public class ApiLoadTests
{
    private readonly HttpClient _client;
    
    public ApiLoadTests()
    {
        _client = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };
    }
    
    [Fact]
    public async Task ProductSearch_100ConcurrentRequests_AllSucceed()
    {
        const int concurrency = 100;
        var tasks = Enumerable.Range(0, concurrency)
            .Select(_ => _client.GetAsync("/api/products?q=test"))
            .ToList();
        
        var responses = await Task.WhenAll(tasks);
        
        responses.Should().AllSatisfy(r =>
            r.IsSuccessStatusCode.Should().BeTrue());
    }
    
    [Fact]
    public async Task GetProduct_HighConcurrency_P99Under500ms()
    {
        const int requests = 200;
        var stopwatches = new List<TimeSpan>();
        var semaphore = new SemaphoreSlim(50); // max 50 concurrent
        
        var tasks = Enumerable.Range(1, requests).Select(async id =>
        {
            await semaphore.WaitAsync();
            try
            {
                var sw = System.Diagnostics.Stopwatch.StartNew();
                await _client.GetAsync($"/api/products/{(id % 10) + 1}");
                sw.Stop();
                
                lock (stopwatches) stopwatches.Add(sw.Elapsed);
            }
            finally { semaphore.Release(); }
        });
        
        await Task.WhenAll(tasks);
        
        var sorted = stopwatches.OrderBy(t => t).ToList();
        var p99 = sorted[(int)(sorted.Count * 0.99)];
        
        p99.Should().BeLessThan(TimeSpan.FromMilliseconds(500),
            "P99 response time should be under 500ms");
    }
}
```

---

## Step 477: Behavior-Driven Development (BDD)

```csharp
// ============================================
// BDD with SpecFlow-style scenarios
// ============================================

// dotnet add package Reqnroll
// Feature: Order placement
// 
// Scenario: Customer places a valid order
//   Given I am a logged-in customer
//   And my cart has 2 items worth 500 THB total
//   When I confirm the order with valid payment
//   Then the order status should be "Confirmed"
//   And I should receive a confirmation email

public class OrderPlacementSteps
{
    private Order? _order;
    private bool _emailSent;
    private readonly Mock<IEmailService> _emailMock = new();
    
    [Given("I am a logged-in customer")]
    public void GivenLoggedInCustomer()
    {
        // Customer context set up
    }
    
    [Given("my cart has (\\d+) items worth (\\d+) THB total")]
    public void GivenCartWithItems(int count, decimal total)
    {
        _order = Order.Create(1, TestData.Address);
        var unitPrice = total / count;
        for (int i = 0; i < count; i++)
            _order.AddLine(i + 1, new ProductName($"สินค้า {i+1}"), Money.Thai(unitPrice), 1);
    }
    
    [When("I confirm the order with valid payment")]
    public void WhenConfirmOrder()
    {
        _order!.Confirm();
    }
    
    [Then("the order status should be \"(.*)\"")]
    public void ThenOrderStatus(string status)
    {
        _order!.Status.ToString().Should().Be(status);
    }
    
    [Then("I should receive a confirmation email")]
    public void ThenEmailSent()
    {
        _emailMock.Verify(e => e.SendOrderConfirmationAsync(It.IsAny<int>(), It.IsAny<int>()));
    }
}
```

---

## Step 478: Test Coverage

```bash
# ============================================
# Test Coverage with Coverlet
# ============================================

# Install
dotnet add package coverlet.msbuild
dotnet add package coverlet.collector

# Run with coverage
dotnet test \
    --collect:"XPlat Code Coverage" \
    --results-directory ./coverage

# Generate HTML report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
    -reports:"./coverage/**/coverage.cobertura.xml" \
    -targetdir:"./coverage/report" \
    -reporttypes:Html

# Enforce minimum coverage
dotnet test /p:CollectCoverage=true \
    /p:CoverletOutputFormat=opencover \
    /p:Threshold=80 \
    /p:ThresholdType=line
```

```csharp
// ============================================
// Custom Test Fixtures
// ============================================

public class DatabaseFixture : IDisposable
{
    public SQLiteConnection Db { get; }
    
    public DatabaseFixture()
    {
        Db = new SQLiteConnection(":memory:");
        Db.CreateTable<OrderData>();
        Db.CreateTable<ProductData>();
        Db.CreateTable<CustomerData>();
        
        // Seed data
        SeedTestData();
    }
    
    private void SeedTestData()
    {
        Db.InsertAll(Enumerable.Range(1, 10).Select(i => new CustomerData
        {
            Id = i,
            Name = $"ลูกค้า {i}",
            Email = $"customer{i}@test.com"
        }));
        
        Db.InsertAll(Enumerable.Range(1, 20).Select(i => new ProductData
        {
            Id = i,
            Name = $"สินค้า {i}",
            Price = i * 100,
            Stock = 50,
            CategoryId = (i % 3) + 1
        }));
    }
    
    public void Dispose() => Db.Dispose();
}

[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class IntegrationTestsWithFixture
{
    private readonly DatabaseFixture _fixture;
    
    public IntegrationTestsWithFixture(DatabaseFixture fixture) => _fixture = fixture;
    
    [Fact]
    public void GetProducts_WithSeedData_Returns20Products()
    {
        var products = _fixture.Db.Table<ProductData>().ToList();
        products.Should().HaveCount(20);
    }
}
```

---

## Step 479: Mutation Testing

```bash
# ============================================
# Mutation Testing with Stryker.NET
# ============================================

# Install
dotnet tool install -g dotnet-stryker

# Run
dotnet stryker --project MyApp.csproj

# Configuration (stryker-config.json)
{
    "stryker-config": {
        "mutation-level": "Advanced",
        "reporters": ["html", "json", "cleartext"],
        "threshold": {
            "high": 80,
            "low": 60,
            "break": 0
        },
        "ignore-methods": [
            "ToString",
            "GetHashCode",
            "Dispose"
        ]
    }
}

# Mutation types Stryker performs:
# - Arithmetic: + → -, * → /
# - Equality: == → !=, > → >=
# - Boolean: && → ||, true → false
# - String: "" → "Stryker was here"
# - Null: value → null
# - Block: remove blocks of code
```

---

## Step 480: Test Architecture

```csharp
// ============================================
// Test Architecture: Builder Pattern for Tests
// ============================================

// Order Builder for tests
public class OrderBuilder
{
    private int _customerId = 1;
    private Address _address = TestData.Address;
    private readonly List<(int ProductId, string Name, decimal Price, int Qty)> _lines = new();
    private bool _confirmed;
    
    public OrderBuilder ForCustomer(int id) { _customerId = id; return this; }
    public OrderBuilder WithAddress(Address addr) { _address = addr; return this; }
    
    public OrderBuilder WithLine(int productId, string name, decimal price, int qty)
    {
        _lines.Add((productId, name, price, qty));
        return this;
    }
    
    public OrderBuilder Confirmed() { _confirmed = true; return this; }
    
    public Order Build()
    {
        var order = Order.Create(_customerId, _address);
        foreach (var (id, name, price, qty) in _lines)
            order.AddLine(id, new ProductName(name), Money.Thai(price), qty);
        if (_confirmed) order.Confirm();
        return order;
    }
}

// Usage
public class OrderBuilderTests
{
    [Fact]
    public void Ship_ConfirmedOrder_Succeeds()
    {
        var order = new OrderBuilder()
            .ForCustomer(42)
            .WithLine(1, "สินค้า", 100, 2)
            .Confirmed()
            .Build();
        
        order.Ship("TH123");
        
        order.Status.Should().Be(OrderStatus.Shipped);
    }
}

// Object Mother
public static class OrderMother
{
    public static Order NewOrder() => new OrderBuilder()
        .WithLine(1, "สินค้าทดสอบ", 299, 1)
        .Build();
    
    public static Order ConfirmedOrder() => new OrderBuilder()
        .WithLine(1, "สินค้าทดสอบ", 299, 2)
        .Confirmed()
        .Build();
    
    public static Order LargeOrder() => new OrderBuilder()
        .WithLine(1, "สินค้า 1", 1000, 5)
        .WithLine(2, "สินค้า 2", 500, 3)
        .WithLine(3, "สินค้า 3", 250, 10)
        .Confirmed()
        .Build();
}
```

---

## สรุป Part 48

ใน Part 48 เราได้เรียนรู้:

1. **xUnit Unit Tests** - Fact, Theory, FluentAssertions
2. **Mocking with Moq** - Setup, Verify, ReturnsAsync
3. **Integration Tests** - SQLite in-memory, HTTP tests
4. **Playwright UI Tests** - End-to-end browser testing
5. **Snapshot Testing** - Verify library
6. **Load Testing** - Concurrency, P99 latency
7. **BDD** - Behavior-driven steps
8. **Coverage** - Coverlet, minimum threshold
9. **Mutation Testing** - Stryker.NET
10. **Test Architecture** - Builder pattern, Object Mother

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 48 | Steps 471-480*
