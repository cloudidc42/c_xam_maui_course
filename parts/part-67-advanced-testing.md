# Part 67: Advanced Testing Strategies
## Steps 661-670: UI Tests, Mutation Testing, Load Testing, TDD

---

## Step 661: UI Testing with Appium/MAUI

```csharp
// ============================================
// UI Testing Framework Setup
// ============================================

// UITestBase.cs
[TestFixture]
public abstract class UITestBase
{
    protected AppiumDriver Driver { get; private set; } = null!;
    
    [OneTimeSetUp]
    public void OneTimeSetUp()
    {
        var options = GetPlatformOptions();
        Driver = new AndroidDriver(new Uri("http://localhost:4723"), options);
        Driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);
    }
    
    [OneTimeTearDown]
    public void OneTimeTearDown() => Driver?.Quit();
    
    private AppiumOptions GetPlatformOptions()
    {
        var options = new AppiumOptions();
        options.PlatformName = "Android";
        options.App = Path.Combine(
            AppContext.BaseDirectory, "../../../", "MyFoodDelivery.apk");
        options.AddAdditionalAppiumOption("appPackage", "com.myfooddelivery");
        options.AddAdditionalAppiumOption("appActivity", "com.myfooddelivery.MainActivity");
        options.AddAdditionalAppiumOption("noReset", true);
        return options;
    }
    
    protected AppiumElement FindByText(string text)
        => Driver.FindElement(MobileBy.XPath($"//*[@text='{text}']"));
    
    protected AppiumElement FindById(string resourceId)
        => Driver.FindElement(MobileBy.Id($"com.myfooddelivery:id/{resourceId}"));
    
    protected async Task WaitForElement(string resourceId, int timeoutSeconds = 15)
    {
        var wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(timeoutSeconds));
        wait.Until(d => d.FindElements(MobileBy.Id(resourceId)).Count > 0);
    }
    
    protected void Tap(AppiumElement element) => element.Click();
    protected void TypeText(AppiumElement element, string text)
    {
        element.Clear();
        element.SendKeys(text);
    }
    
    protected void SwipeUp()
    {
        var size = Driver.Manage().Window.Size;
        Driver.ExecuteScript("mobile:swipe", new Dictionary<string, object>
        {
            ["direction"] = "up",
            ["speed"] = 2000
        });
    }
}

// Login test
[TestFixture]
public class LoginUITests : UITestBase
{
    [Test]
    public async Task Login_WithValidCredentials_NavigatesHome()
    {
        // Arrange
        await WaitForElement("email_input");
        
        var emailField = FindById("email_input");
        var passwordField = FindById("password_input");
        var loginButton = FindById("login_button");
        
        // Act
        TypeText(emailField, "test@example.com");
        TypeText(passwordField, "Test1234!");
        Tap(loginButton);
        
        // Assert
        await WaitForElement("home_page", 20);
        var homePage = FindById("home_page");
        Assert.That(homePage.Displayed, Is.True);
    }
    
    [Test]
    public async Task Login_WithInvalidCredentials_ShowsError()
    {
        await WaitForElement("email_input");
        
        TypeText(FindById("email_input"), "wrong@example.com");
        TypeText(FindById("password_input"), "wrongpass");
        Tap(FindById("login_button"));
        
        await WaitForElement("error_label");
        Assert.That(FindById("error_label").Displayed, Is.True);
    }
}
```

---

## Step 662: Screenshot Tests

```csharp
// ============================================
// Screenshot / Visual Regression Tests
// ============================================

[TestFixture]
public class VisualRegressionTests : UITestBase
{
    private readonly string _baselineDir = "Screenshots/Baseline";
    private readonly string _actualDir = "Screenshots/Actual";
    
    [SetUp]
    public void SetUp()
    {
        Directory.CreateDirectory(_baselineDir);
        Directory.CreateDirectory(_actualDir);
    }
    
    [Test]
    public async Task HomeScreen_MatchesBaseline()
    {
        await WaitForElement("home_page");
        await Task.Delay(500); // Wait for animations
        
        var screenshot = await TakeScreenshotAsync("home");
        
        if (IsBaselineExists("home"))
            CompareWithBaseline("home", screenshot);
        else
            SaveAsBaseline("home", screenshot);
    }
    
    [Test]
    public async Task RestaurantDetail_MatchesBaseline()
    {
        // Navigate to restaurant
        await WaitForElement("restaurant_list");
        FindById("restaurant_list").FindElements(By.ClassName("android.widget.FrameLayout"))[0].Click();
        
        await WaitForElement("restaurant_detail");
        await Task.Delay(500);
        
        var screenshot = await TakeScreenshotAsync("restaurant_detail");
        
        if (IsBaselineExists("restaurant_detail"))
            CompareWithBaseline("restaurant_detail", screenshot);
        else
            SaveAsBaseline("restaurant_detail", screenshot);
    }
    
    private async Task<byte[]> TakeScreenshotAsync(string name)
    {
        var screenshot = ((ITakesScreenshot)Driver).GetScreenshot();
        var bytes = screenshot.AsByteArray;
        var path = Path.Combine(_actualDir, $"{name}.png");
        await File.WriteAllBytesAsync(path, bytes);
        return bytes;
    }
    
    private bool IsBaselineExists(string name)
        => File.Exists(Path.Combine(_baselineDir, $"{name}.png"));
    
    private void SaveAsBaseline(string name, byte[] screenshot)
        => File.WriteAllBytes(Path.Combine(_baselineDir, $"{name}.png"), screenshot);
    
    private void CompareWithBaseline(string name, byte[] actual)
    {
        var baseline = File.ReadAllBytes(Path.Combine(_baselineDir, $"{name}.png"));
        var similarity = CalculateImageSimilarity(baseline, actual);
        Assert.That(similarity, Is.GreaterThan(0.95),
            $"Screenshot '{name}' differs from baseline by more than 5%");
    }
    
    private double CalculateImageSimilarity(byte[] a, byte[] b)
    {
        // Simplified pixel comparison
        if (a.Length != b.Length) return 0;
        var matching = a.Zip(b, (x, y) => Math.Abs(x - y) < 10).Count(m => m);
        return (double)matching / a.Length;
    }
}
```

---

## Step 663: Mutation Testing

```csharp
// ============================================
// Mutation Testing (using Stryker.NET concepts)
// ============================================

// Target: test quality, not just coverage
// Run: dotnet stryker --project "MyApp.Domain"

// Example: Domain logic that should be mutation-tested
public class PricingCalculator
{
    public decimal CalculateDeliveryFee(decimal subtotal, double distanceKm)
    {
        // Base fee
        decimal fee = 20;
        
        // Distance surcharge: +5 per km after 3km
        if (distanceKm > 3)
            fee += (decimal)((distanceKm - 3) * 5);
        
        // Free delivery threshold
        if (subtotal >= 300)
            fee = 0;
        
        return fee;
    }
}

// What Stryker mutates:
// - ">" to ">=" on distanceKm > 3
// - "- 3" removed
// - "* 5" changed to "* 4"  
// - ">= 300" to "> 300"
// - "fee = 0" to "fee = 1"
// Tests MUST catch all of these mutations

// ✅ Comprehensive tests that survive mutation
[TestFixture]
public class PricingCalculatorMutationTests
{
    private readonly PricingCalculator _calc = new();
    
    // Boundary: exactly 3km
    [Test] public void Fee_At3km_IsBaseFee() =>
        Assert.That(_calc.CalculateDeliveryFee(100, 3), Is.EqualTo(20));
    
    // Boundary: just over 3km
    [Test] public void Fee_At3_1km_AddsDistance() =>
        Assert.That(_calc.CalculateDeliveryFee(100, 3.1), Is.EqualTo(20.5m));
    
    // Distance factor
    [Test] public void Fee_At5km_Adds10Baht() =>
        Assert.That(_calc.CalculateDeliveryFee(100, 5), Is.EqualTo(30m));
    
    // Boundary: exactly 300 subtotal
    [Test] public void Fee_At300Subtotal_IsFree() =>
        Assert.That(_calc.CalculateDeliveryFee(300, 1), Is.EqualTo(0));
    
    // Just below free threshold
    [Test] public void Fee_At299Subtotal_NotFree() =>
        Assert.That(_calc.CalculateDeliveryFee(299, 1), Is.GreaterThan(0));
    
    // Combination
    [Test] public void Fee_At500Subtotal10km_IsFree() =>
        Assert.That(_calc.CalculateDeliveryFee(500, 10), Is.EqualTo(0));
    
    // Zero distance
    [Test] public void Fee_At0km_IsBaseFee() =>
        Assert.That(_calc.CalculateDeliveryFee(100, 0), Is.EqualTo(20));
}
```

---

## Step 664: Property-Based Testing

```csharp
// ============================================
// Property-Based Testing with FsCheck
// ============================================

using FsCheck;
using FsCheck.NUnit;

[TestFixture]
public class CartPropertyTests
{
    [Property]
    public Property Cart_TotalAlwaysEqualsItemsSum()
    {
        return Prop.ForAll(
            GenCartItems(),
            items =>
            {
                var cart = new Cart(1);
                foreach (var (item, qty) in items)
                    cart.AddItem(item, qty);
                
                var expectedTotal = items.Sum(x => x.Item1.Price.Amount * x.Item2);
                return cart.Total.Amount == expectedTotal;
            });
    }
    
    [Property]
    public Property Cart_ItemCountNeverExceedsMax()
    {
        return Prop.ForAll(
            Arb.From(Gen.Choose(1, 200)),
            (count) =>
            {
                var cart = new Cart(1);
                var product = Product.Create("Test", Money.Of(10, "THB"), 1);
                
                try
                {
                    for (int i = 0; i < count; i++)
                        cart.AddItem(product, 1);
                    
                    return cart.TotalQuantity <= Cart.MaxItems;
                }
                catch (DomainException)
                {
                    return count > Cart.MaxItems; // Expected to throw
                }
            });
    }
    
    [Property]
    public Property Money_AdditionIsCommutative()
    {
        return Prop.ForAll(
            Arb.From(Gen.Choose(1, 10000).Select(v => Money.Of(v, "THB"))),
            Arb.From(Gen.Choose(1, 10000).Select(v => Money.Of(v, "THB"))),
            (a, b) => (a + b).Amount == (b + a).Amount);
    }
    
    [Property]
    public Property Haversine_DistanceIsSymmetric()
    {
        var locationGen = from lat in Gen.Choose(-900000, 900000).Select(v => v / 10000.0)
                          from lng in Gen.Choose(-1800000, 1800000).Select(v => v / 10000.0)
                          select new Location(lat, lng);
        
        return Prop.ForAll(
            Arb.From(locationGen), Arb.From(locationGen),
            (a, b) => Math.Abs(a.DistanceTo(b) - b.DistanceTo(a)) < 0.001);
    }
    
    private Arbitrary<(Product, int)[]> GenCartItems()
    {
        var itemGen = from price in Gen.Choose(1, 1000).Select(v => Money.Of(v, "THB"))
                      from qty in Gen.Choose(1, 5)
                      select (Product.Create($"P_{price}", price, 1), qty);
        
        return Arb.From(Gen.ArrayOf(Gen.Choose(0, 5).SelectMany(n =>
            Gen.ListOf(n, itemGen).Select(l => l.ToArray()))).SelectMany(a => a));
    }
}
```

---

## Step 665: Load Testing with k6

```javascript
// ============================================
// k6 Load Test Script
// ============================================

// load-test.js - run with: k6 run load-test.js

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreationTime = new Trend('order_creation_time');

export const options = {
    scenarios: {
        // Ramp-up load
        ramp_up: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '30s', target: 50 },
                { duration: '1m', target: 100 },
                { duration: '2m', target: 100 },
                { duration: '30s', target: 0 },
            ],
        },
        // Spike test
        spike: {
            executor: 'constant-arrival-rate',
            startTime: '3m',
            duration: '30s',
            rate: 500,
            timeUnit: '1s',
            preAllocatedVUs: 200,
        },
    },
    thresholds: {
        'http_req_duration': ['p(95)<2000', 'p(99)<5000'],
        'error_rate': ['rate<0.01'],
        'order_creation_time': ['p(95)<3000'],
    },
};

const BASE_URL = 'https://api-staging.myfooddelivery.com';

export function setup() {
    // Login and get auth token
    const res = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
        email: 'loadtest@example.com',
        password: 'LoadTest123!'
    }), { headers: { 'Content-Type': 'application/json' } });
    
    return { token: res.json('token') };
}

export default function({ token }) {
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
    };
    
    // Browse restaurants
    const restaurantsRes = http.get(`${BASE_URL}/restaurants?lat=13.7&lng=100.5`, { headers });
    check(restaurantsRes, { 'restaurants 200': r => r.status === 200 });
    errorRate.add(restaurantsRes.status !== 200);
    
    sleep(1);
    
    // View restaurant detail
    const restaurants = restaurantsRes.json('items');
    if (restaurants && restaurants.length > 0) {
        const rid = restaurants[0].id;
        const detailRes = http.get(`${BASE_URL}/restaurants/${rid}`, { headers });
        check(detailRes, { 'detail 200': r => r.status === 200 });
    }
    
    sleep(1);
    
    // Place order
    const start = Date.now();
    const orderRes = http.post(`${BASE_URL}/orders`, JSON.stringify({
        restaurantId: 1,
        items: [{ menuItemId: 1, quantity: 2 }],
        deliveryAddress: { street: '123 Test St', lat: 13.7, lng: 100.5 }
    }), { headers });
    
    orderCreationTime.add(Date.now() - start);
    
    check(orderRes, {
        'order created': r => r.status === 201,
        'has orderId': r => r.json('id') !== null,
    });
    errorRate.add(orderRes.status !== 201);
    
    sleep(2);
}

export function teardown(data) {
    console.log('Load test completed');
}
```

---

## Step 666: API Load Test (C# NBomber)

```csharp
// ============================================
// NBomber Load Testing in C#
// ============================================

public class ApiLoadTests
{
    [Test, Explicit("Load test - run manually")]
    public void API_CanHandle100ConcurrentUsers()
    {
        var httpClient = new HttpClient { BaseAddress = new Uri("https://api-staging.myfooddelivery.com") };
        httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", GetTestToken());
        
        var scenario = Scenario.Create("browse_and_order", async ctx =>
        {
            // Step 1: List restaurants
            var step1 = await Step.Run("list_restaurants", ctx, async () =>
            {
                var resp = await httpClient.GetAsync("/restaurants?lat=13.7&lng=100.5");
                return resp.IsSuccessStatusCode
                    ? Response.Ok(statusCode: (int)resp.StatusCode)
                    : Response.Fail(statusCode: (int)resp.StatusCode);
            });
            
            // Step 2: Get menu
            var step2 = await Step.Run("get_menu", ctx, async () =>
            {
                var resp = await httpClient.GetAsync("/restaurants/1/menu");
                return resp.IsSuccessStatusCode
                    ? Response.Ok() : Response.Fail();
            });
            
            // Step 3: Create order
            var step3 = await Step.Run("create_order", ctx, async () =>
            {
                var body = JsonSerializer.Serialize(new
                {
                    restaurantId = 1,
                    items = new[] { new { menuItemId = 1, quantity = 1 } }
                });
                
                var resp = await httpClient.PostAsync("/orders",
                    new StringContent(body, Encoding.UTF8, "application/json"));
                
                return resp.IsSuccessStatusCode
                    ? Response.Ok(statusCode: 201) : Response.Fail();
            });
            
            return Response.Ok();
        })
        .WithLoadSimulations(
            Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromMinutes(2)),
            Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(3))
        );
        
        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .WithReportFileName("load_test_report")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();
        
        // Assertions
        var allStats = stats.ScenarioStats;
        Assert.Multiple(() =>
        {
            Assert.That(allStats[0].Fail.Request.Percent, Is.LessThan(1),
                "Error rate must be < 1%");
            Assert.That(allStats[0].Ok.Latency.Percent99, Is.LessThan(5000),
                "p99 must be < 5s");
        });
    }
    
    private string GetTestToken() => "test_token";
}
```

---

## Step 667: Test Data Builders

```csharp
// ============================================
// Test Data Builders (Fluent API)
// ============================================

public class RestaurantBuilder
{
    private int _id = 1;
    private string _name = "Test Restaurant";
    private string _category = "Thai";
    private decimal _deliveryFee = 30;
    private int _deliveryMinutes = 30;
    private bool _isOpen = true;
    private double _rating = 4.5;
    private Location _location = new(13.7, 100.5);
    private List<MenuItem> _menuItems = new();
    
    public RestaurantBuilder WithId(int id) { _id = id; return this; }
    public RestaurantBuilder WithName(string name) { _name = name; return this; }
    public RestaurantBuilder WithCategory(string category) { _category = category; return this; }
    public RestaurantBuilder WithDeliveryFee(decimal fee) { _deliveryFee = fee; return this; }
    public RestaurantBuilder WithDeliveryMinutes(int minutes) { _deliveryMinutes = minutes; return this; }
    public RestaurantBuilder Closed() { _isOpen = false; return this; }
    public RestaurantBuilder WithRating(double rating) { _rating = rating; return this; }
    public RestaurantBuilder AtLocation(double lat, double lng) { _location = new(lat, lng); return this; }
    
    public RestaurantBuilder WithMenuItem(Action<MenuItemBuilder>? configure = null)
    {
        var builder = new MenuItemBuilder();
        configure?.Invoke(builder);
        _menuItems.Add(builder.Build());
        return this;
    }
    
    public Restaurant Build()
    {
        var restaurant = Restaurant.Create(
            _name, _category, _location,
            Money.Of(_deliveryFee, "THB"), _deliveryMinutes);
        
        foreach (var item in _menuItems)
            restaurant.AddMenuItem(item.Name, item.Description, item.Price, item.Category);
        
        if (!_isOpen) restaurant.Close();
        
        return restaurant;
    }
}

public class MenuItemBuilder
{
    private string _name = "Test Item";
    private string _description = "Test Description";
    private decimal _price = 100;
    private string _category = "Main";
    
    public MenuItemBuilder WithName(string name) { _name = name; return this; }
    public MenuItemBuilder WithPrice(decimal price) { _price = price; return this; }
    public MenuItemBuilder InCategory(string cat) { _category = cat; return this; }
    
    public MenuItem Build() => new MenuItem
    {
        Name = _name, Description = _description,
        Price = Money.Of(_price, "THB"), Category = _category
    };
}

public class FoodOrderBuilder
{
    private int _customerId = 1;
    private int _restaurantId = 1;
    private List<(int itemId, decimal price, int qty)> _lines = new();
    
    public FoodOrderBuilder ForCustomer(int id) { _customerId = id; return this; }
    public FoodOrderBuilder AtRestaurant(int id) { _restaurantId = id; return this; }
    
    public FoodOrderBuilder WithItem(int itemId, decimal price, int qty = 1)
    {
        _lines.Add((itemId, price, qty));
        return this;
    }
    
    public FoodOrder Build()
    {
        var lines = _lines.Select(l => new OrderLine(l.itemId, $"Item{l.itemId}", Money.Of(l.price, "THB"), l.qty)).ToList();
        return FoodOrder.Place(_customerId, _restaurantId, lines, Money.Of(30, "THB"));
    }
}

// Usage in tests
public class OrderTests
{
    [Test]
    public void Order_WithMultipleItems_HasCorrectTotal()
    {
        var order = new FoodOrderBuilder()
            .ForCustomer(42)
            .WithItem(1, 120, 2)
            .WithItem(2, 80, 1)
            .Build();
        
        Assert.That(order.Subtotal.Amount, Is.EqualTo(320m));
    }
    
    [Test]
    public void ClosedRestaurant_CannotAcceptOrders()
    {
        var restaurant = new RestaurantBuilder()
            .WithName("Closed Place")
            .Closed()
            .Build();
        
        Assert.That(restaurant.IsOpen, Is.False);
    }
}
```

---

## Step 668: Test Doubles

```csharp
// ============================================
// Test Doubles (Fakes, Stubs, Spies)
// ============================================

// Fake in-memory repository
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<int, FoodOrder> _store = new();
    private int _nextId = 1;
    
    public Task<FoodOrder?> GetByIdAsync(int id, CancellationToken ct = default)
        => Task.FromResult(_store.TryGetValue(id, out var o) ? o : null);
    
    public Task<List<FoodOrder>> GetByCustomerAsync(int customerId, CancellationToken ct = default)
        => Task.FromResult(_store.Values
            .Where(o => o.CustomerId == customerId)
            .ToList());
    
    public Task SaveAsync(FoodOrder order, CancellationToken ct = default)
    {
        if (order.Id == 0) order = order with { Id = _nextId++ };
        _store[order.Id] = order;
        return Task.CompletedTask;
    }
    
    public Task DeleteAsync(int id, CancellationToken ct = default)
    {
        _store.Remove(id);
        return Task.CompletedTask;
    }
    
    public int Count() => _store.Count;
    public void Clear() => _store.Clear();
}

// Spy/Interceptor for event bus
public class SpyEventBus : IEventBus
{
    public List<IDomainEvent> PublishedEvents { get; } = new();
    
    public Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        PublishedEvents.Add(@event);
        return Task.CompletedTask;
    }
    
    public void Register<TEvent>(IEventHandler<TEvent> handler) where TEvent : IDomainEvent { }
    
    public bool HasPublished<TEvent>() where TEvent : IDomainEvent
        => PublishedEvents.OfType<TEvent>().Any();
    
    public TEvent? GetLastPublished<TEvent>() where TEvent : IDomainEvent
        => PublishedEvents.OfType<TEvent>().LastOrDefault();
}

// Usage
[TestFixture]
public class PlaceOrderTests
{
    [Test]
    public async Task PlaceOrder_PublishesOrderPlacedEvent()
    {
        var spyBus = new SpyEventBus();
        var repo = new InMemoryOrderRepository();
        
        var service = new PlaceOrderService(repo, spyBus);
        
        await service.PlaceOrderAsync(new PlaceOrderCommand
        {
            CustomerId = 1, RestaurantId = 1,
            Items = new() { new(1, 2) }
        });
        
        Assert.That(spyBus.HasPublished<OrderPlacedEvent>(), Is.True);
        
        var evt = spyBus.GetLastPublished<OrderPlacedEvent>();
        Assert.That(evt!.CustomerId, Is.EqualTo(1));
    }
}
```

---

## Step 669: TDD Workflow

```csharp
// ============================================
// TDD: Red → Green → Refactor
// ============================================

// RED: Write failing test first
[Test]
public void CartDiscount_WithGoldTier_Applies10Percent()
{
    // Arrange
    var pricingService = new PricingService();
    var customer = new Customer { Tier = CustomerTier.Gold };
    var cart = new CartBuilder()
        .WithItem(price: 200, qty: 6) // 1200 total → qualifies for Gold (≥1000)
        .Build();
    
    // Act
    var discount = pricingService.CalculateDiscount(cart, customer);
    
    // Assert
    Assert.That(discount.Amount, Is.EqualTo(120m)); // 10% of 1200
}

// GREEN: Minimal implementation
public class PricingService
{
    public Money CalculateDiscount(Cart cart, Customer customer)
    {
        var subtotal = cart.Total.Amount;
        
        if (customer.Tier == CustomerTier.Gold && subtotal >= 1000)
            return Money.Of(subtotal * 0.1m, "THB");
        
        return Money.Of(0, "THB");
    }
}

// Now test for Silver tier
[Test]
public void CartDiscount_WithSilverTier_Applies5Percent()
{
    var pricingService = new PricingService();
    var customer = new Customer { Tier = CustomerTier.Silver };
    var cart = new CartBuilder().WithItem(price: 100, qty: 5).Build(); // 500
    
    var discount = pricingService.CalculateDiscount(cart, customer);
    Assert.That(discount.Amount, Is.EqualTo(25m)); // 5% of 500
}

// REFACTOR: Generalize
public class PricingServiceV2
{
    private readonly Dictionary<CustomerTier, (decimal MinSpend, decimal DiscountRate)> _tierRules = new()
    {
        [CustomerTier.Gold]   = (1000m, 0.10m),
        [CustomerTier.Silver] = (500m,  0.05m),
        [CustomerTier.Bronze] = (300m,  0.02m),
        [CustomerTier.Regular] = (0m,   0m),
    };
    
    public Money CalculateDiscount(Cart cart, Customer customer)
    {
        var subtotal = cart.Total.Amount;
        
        if (_tierRules.TryGetValue(customer.Tier, out var rule))
        {
            if (subtotal >= rule.MinSpend && rule.DiscountRate > 0)
                return Money.Of(subtotal * rule.DiscountRate, "THB");
        }
        
        return Money.Of(0, "THB");
    }
}
```

---

## Step 670: CI Test Pipeline

```yaml
# .github/workflows/tests.yml
name: Tests

on:
  push:
    branches: [ main, develop, claude/** ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Unit Tests
        run: |
          dotnet test tests/MyApp.Domain.Tests \
            --no-build --configuration Release \
            --collect:"XPlat Code Coverage" \
            --logger "trx;LogFileName=unit-tests.trx"
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: '**/coverage.cobertura.xml'
          fail_ci_if_error: true
  
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Integration Tests
        run: |
          dotnet test tests/MyApp.Integration.Tests \
            --no-build --configuration Release \
            --logger "trx;LogFileName=integration.trx"
        env:
          ConnectionStrings__Postgres: "Host=localhost;Database=testdb;Username=testuser;Password=testpass"
  
  mutation-tests:
    name: Mutation Tests (Stryker)
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Stryker
        run: dotnet tool install -g dotnet-stryker
      
      - name: Run Mutations
        run: dotnet stryker --project "src/MyApp.Domain" --threshold-break 80
      
      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: StrykerOutput/
  
  coverage-gate:
    name: Coverage Gate
    needs: [unit-tests]
    runs-on: ubuntu-latest
    steps:
      - name: Check Coverage >= 80%
        run: |
          COVERAGE=$(grep -oP 'line-rate="\K[^"]+' coverage.xml | head -1)
          echo "Coverage: $COVERAGE"
          if (( $(echo "$COVERAGE < 0.80" | bc -l) )); then
            echo "Coverage $COVERAGE is below 80%"
            exit 1
          fi
```

---

## สรุป Part 67

ใน Part 67 เราได้เรียนรู้:

1. **UI Testing (Appium)** - Driver setup, element finding, form input
2. **Visual Regression** - Screenshot comparison, baseline management
3. **Mutation Testing** - Stryker, writing tests that survive mutations
4. **Property-Based Testing** - FsCheck, math properties, domain invariants
5. **Load Testing (k6)** - Scenarios, thresholds, ramp-up/spike
6. **NBomber (C#)** - Load simulation, assertions
7. **Test Data Builders** - Fluent builders for domain objects
8. **Test Doubles** - Fake repository, spy event bus
9. **TDD Workflow** - Red → Green → Refactor cycle
10. **CI Test Pipeline** - Unit, integration, mutation in GitHub Actions

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 67 | Steps 661-670*
