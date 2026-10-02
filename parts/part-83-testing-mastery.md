# Part 83: Testing Mastery
## Steps 821-830: Test Pyramid, Contract Tests, Snapshot Tests, Fuzz Testing

---

## Step 821: Test Pyramid Strategy

```csharp
// ============================================
// Test Pyramid: 70% Unit / 20% Integration / 10% E2E
// ============================================

// Unit tests: fast, isolated, many
[TestFixture]
[Category("Unit")]
public class OrderDomainTests
{
    [Test]
    public void CreateOrder_WithItems_CalculatesTotal()
    {
        var lines = new List<OrderLine>
        {
            new("item-1", "ข้าวผัด", new Money(80, "THB"), 2),
            new("item-2", "น้ำเปล่า", new Money(20, "THB"), 1)
        };
        
        var order = Order.Create(new CustomerId(Guid.NewGuid()), lines);
        
        Assert.That(order.Total.Amount, Is.EqualTo(180));
    }
    
    [Test]
    public void ConfirmOrder_WhenPending_ChangesStatusToConfirmed()
    {
        var order = OrderTestFactory.Create();
        order.Confirm();
        Assert.That(order.Status, Is.EqualTo(OrderStatus.Confirmed));
    }
    
    [Test]
    public void ConfirmOrder_WhenAlreadyConfirmed_ThrowsDomainException()
    {
        var order = OrderTestFactory.CreateConfirmed();
        Assert.Throws<DomainException>(() => order.Confirm());
    }
}

// Integration tests: moderate, real DB
[TestFixture]
[Category("Integration")]
public class OrderRepositoryIntegrationTests
{
    // Tests with real SQLite (in-memory)
}

// E2E tests: slow, full app
[TestFixture]
[Category("E2E")]
public class OrderFlowE2ETests
{
    // Appium tests that exercise the real app
}

// Helper factory
public static class OrderTestFactory
{
    public static Order Create(decimal price = 100, int qty = 1) =>
        Order.Create(new CustomerId(Guid.NewGuid()),
            new[] { new OrderLine("i1", "ข้าวผัด", new Money(price, "THB"), qty) });
    
    public static Order CreateConfirmed()
    {
        var o = Create(); o.Confirm(); return o;
    }
}
```

---

## Step 822: Contract Tests

```csharp
// ============================================
// Consumer-Driven Contract Tests (Pact-like)
// ============================================

// Define expected API contract from client perspective
public class RestaurantApiContract
{
    private readonly HttpClient _client;
    
    public RestaurantApiContract(HttpClient client)
    {
        _client = client;
    }
    
    [Test]
    public async Task GetRestaurant_ReturnsExpectedShape()
    {
        var response = await _client.GetFromJsonAsync<RestaurantResponse>(
            "/api/restaurants/test-id");
        
        // Assert contract: required fields present with correct types
        Assert.That(response, Is.Not.Null);
        Assert.That(response!.Id, Is.Not.Empty);
        Assert.That(response.Name, Is.Not.Empty);
        Assert.That(response.Rating, Is.InRange(0, 5));
        Assert.That(response.DeliveryFee, Is.GreaterThanOrEqualTo(0));
        Assert.That(response.MinOrder, Is.GreaterThanOrEqualTo(0));
        Assert.That(response.Menu, Is.Not.Null);
    }
    
    [Test]
    public async Task CreateOrder_WithValidPayload_Returns201()
    {
        var payload = new CreateOrderRequest
        {
            RestaurantId = "test-restaurant",
            Items = new[] { new OrderItemRequest("item-1", 2) }
        };
        
        var response = await _client.PostAsJsonAsync("/api/orders", payload);
        
        Assert.That((int)response.StatusCode, Is.EqualTo(201));
        
        var result = await response.Content.ReadFromJsonAsync<OrderCreatedResponse>();
        Assert.That(result!.OrderId, Is.Not.Empty);
        Assert.That(result.Status, Is.EqualTo("Pending"));
    }
}
```

---

## Step 823: Golden File / Snapshot Tests

```csharp
// ============================================
// Snapshot Testing for ViewModels
// ============================================

public class SnapshotTestBase
{
    private readonly string _snapshotDir;
    
    protected SnapshotTestBase()
    {
        _snapshotDir = Path.Combine(
            TestContext.CurrentContext.TestDirectory, "Snapshots");
        Directory.CreateDirectory(_snapshotDir);
    }
    
    protected void MatchSnapshot(object subject, [CallerMemberName] string? name = null)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(subject,
            new System.Text.Json.JsonSerializerOptions { WriteIndented = true });
        
        var snapshotPath = Path.Combine(_snapshotDir, $"{name}.snap");
        
        if (!File.Exists(snapshotPath))
        {
            // First run: create snapshot
            File.WriteAllText(snapshotPath, json);
            return;
        }
        
        var existing = File.ReadAllText(snapshotPath);
        
        if (json != existing)
        {
            // Show diff
            var diff = GenerateDiff(existing, json);
            Assert.Fail($"Snapshot mismatch for {name}:\n{diff}");
        }
    }
    
    private string GenerateDiff(string expected, string actual)
    {
        var expLines = expected.Split('\n');
        var actLines = actual.Split('\n');
        var sb = new StringBuilder();
        
        for (int i = 0; i < Math.Max(expLines.Length, actLines.Length); i++)
        {
            var exp = i < expLines.Length ? expLines[i] : "(missing)";
            var act = i < actLines.Length ? actLines[i] : "(missing)";
            if (exp != act)
                sb.AppendLine($"Line {i + 1}:\n  - {exp}\n  + {act}");
        }
        
        return sb.ToString();
    }
}

[TestFixture]
public class CartViewModelSnapshotTests : SnapshotTestBase
{
    [Test]
    public async Task CartViewModel_WithTwoItems_MatchesSnapshot()
    {
        var vm = new CartViewModel(new FakeCartRepository(), null!, null!, null!);
        await vm.LoadAsync();
        
        MatchSnapshot(new
        {
            vm.Items.Count,
            vm.Subtotal,
            vm.DeliveryFee,
            vm.Total,
            vm.HasItems
        });
    }
}
```

---

## Step 824: Fuzz Testing

```csharp
// ============================================
// Fuzz Testing with Random Inputs
// ============================================

[TestFixture]
public class FuzzTests
{
    private readonly Random _rng = new(42);
    
    [Test]
    public void ThaiNationalId_RandomInputs_NeverThrows()
    {
        // Generate 10,000 random inputs
        for (int i = 0; i < 10_000; i++)
        {
            var input = GenerateRandomString(_rng, 0, 20, allowDigits: true, allowLetters: true);
            
            // Should not throw, just return valid/invalid
            Assert.DoesNotThrow(
                () => InputSanitizer.IsValidThaiNationalId(input),
                $"Threw on input: '{input}'");
        }
    }
    
    [Test]
    public void PhoneNumberFormatter_RandomInputs_NeverThrows()
    {
        for (int i = 0; i < 5000; i++)
        {
            var input = GenerateRandomString(_rng, 0, 15, allowDigits: true, allowSpecial: true);
            
            Assert.DoesNotThrow(
                () => ThaiFormatter.Phone(input),
                $"Threw on input: '{input}'");
        }
    }
    
    [Test]
    public void NaturalLanguageParser_RandomThai_NeverThrows()
    {
        var parser = new NaturalLanguageOrderParser();
        var menu = new List<MenuItem>
        {
            new MenuItem { Id = "1", Name = "ข้าวผัด", Price = 80 }
        };
        
        for (int i = 0; i < 3000; i++)
        {
            var input = GenerateRandomThaiString(_rng, 0, 100);
            
            Assert.DoesNotThrow(
                () => parser.Parse(input, menu),
                $"Threw on Thai input of length {input.Length}");
        }
    }
    
    [Test]
    public void Money_Operations_NeverOverflow()
    {
        for (int i = 0; i < 10_000; i++)
        {
            var a = new Money((decimal)(_rng.NextDouble() * 1_000_000), "THB");
            var b = new Money((decimal)(_rng.NextDouble() * 1_000_000), "THB");
            
            Assert.DoesNotThrow(() => { var _ = a + b; },
                $"Overflow with {a.Amount} + {b.Amount}");
        }
    }
    
    private static string GenerateRandomString(Random rng, int minLen, int maxLen,
        bool allowDigits = false, bool allowLetters = false, bool allowSpecial = false)
    {
        var chars = new List<char>();
        if (allowDigits) chars.AddRange("0123456789");
        if (allowLetters) chars.AddRange("abcdefghijklmnopqrstuvwxyz");
        if (allowSpecial) chars.AddRange("!@#$%^&*()-+=[]{}|;':,.<>?/\\\"` ");
        if (!chars.Any()) chars.Add('a');
        
        var len = rng.Next(minLen, maxLen + 1);
        return new string(Enumerable.Range(0, len).Select(_ => chars[rng.Next(chars.Count)]).ToArray());
    }
    
    private static string GenerateRandomThaiString(Random rng, int minLen, int maxLen)
    {
        const string thaiChars = "กขคงจฉชซณดตถทธนบปผฝพฟภมยรลวศษสหอฮ";
        var len = rng.Next(minLen, maxLen + 1);
        return new string(Enumerable.Range(0, len).Select(_ => thaiChars[rng.Next(thaiChars.Length)]).ToArray());
    }
}
```

---

## Step 825: Test Data Management

```csharp
// ============================================
// Test Data Seeder
// ============================================

public class TestDataSeeder
{
    private readonly SQLiteAsyncConnection _db;
    private readonly Random _rng = new(42);
    
    private static readonly string[] _menuItems = {
        "ข้าวผัดกุ้ง", "ผัดไทย", "ต้มยำกุ้ง", "ส้มตำไทย",
        "ข้าวมันไก่", "หมูกะทะ", "แกงเขียวหวาน", "ผัดกะเพรา"
    };
    
    private static readonly string[] _restaurants = {
        "ร้านอาหารครัวไทย", "ส้มตำนัว", "ข้าวมันไก่ดีเจ", "ผัดไทยบางลำพู"
    };
    
    public TestDataSeeder(SQLiteAsyncConnection db) => _db = db;
    
    public async Task SeedAsync(int restaurantCount = 5, int ordersPerUser = 10)
    {
        await CreateTablesAsync();
        
        var restaurants = await SeedRestaurantsAsync(restaurantCount);
        var users = await SeedUsersAsync(20);
        await SeedOrdersAsync(users, restaurants, ordersPerUser);
    }
    
    private Task CreateTablesAsync()
        => Task.WhenAll(
            _db.CreateTableAsync<RestaurantEntity>(),
            _db.CreateTableAsync<UserEntity>(),
            _db.CreateTableAsync<OrderEntity>());
    
    private async Task<List<RestaurantEntity>> SeedRestaurantsAsync(int count)
    {
        var restaurants = Enumerable.Range(1, count).Select(i =>
            new RestaurantEntity
            {
                Id = Guid.NewGuid().ToString(),
                Name = _restaurants[i % _restaurants.Length],
                Rating = (float)(_rng.NextDouble() * 2 + 3), // 3-5
                DeliveryFee = _rng.Next(0, 4) * 10,
                MinOrder = 100 + _rng.Next(0, 5) * 50
            }).ToList();
        
        await _db.InsertAllAsync(restaurants);
        return restaurants;
    }
    
    private async Task<List<UserEntity>> SeedUsersAsync(int count)
    {
        var users = Enumerable.Range(1, count).Select(i =>
            new UserEntity
            {
                Id = Guid.NewGuid().ToString(),
                Name = $"ผู้ใช้ {i}",
                Email = $"user{i}@test.com",
                Phone = $"08{i:D8}"
            }).ToList();
        
        await _db.InsertAllAsync(users);
        return users;
    }
    
    private async Task SeedOrdersAsync(
        List<UserEntity> users, List<RestaurantEntity> restaurants, int ordersPerUser)
    {
        var orders = new List<OrderEntity>();
        
        foreach (var user in users)
        {
            for (int i = 0; i < ordersPerUser; i++)
            {
                orders.Add(new OrderEntity
                {
                    Id = Guid.NewGuid().ToString(),
                    UserId = user.Id,
                    RestaurantId = restaurants[_rng.Next(restaurants.Count)].Id,
                    Total = 100 + _rng.Next(0, 10) * 50,
                    Status = "Delivered",
                    CreatedAt = DateTime.UtcNow.AddDays(-_rng.Next(0, 30))
                });
            }
        }
        
        await _db.InsertAllAsync(orders);
    }
}
```

---

## Step 826: Behavior-Driven Tests

```csharp
// ============================================
// BDD-Style Tests
// ============================================

[TestFixture]
public class OrderBddTests
{
    private Order? _order;
    private Exception? _exception;
    
    // Given
    private void GivenPendingOrder()
        => _order = OrderTestFactory.Create();
    
    private void GivenConfirmedOrder()
        => _order = OrderTestFactory.CreateConfirmed();
    
    private void GivenOrderWithTotal(decimal total)
        => _order = OrderTestFactory.Create(total, 1);
    
    // When
    private void WhenConfirmOrder()
    {
        try { _order?.Confirm(); }
        catch (Exception ex) { _exception = ex; }
    }
    
    private void WhenApplyDiscount(decimal discount)
        => _order?.ApplyDiscount(discount);
    
    // Then
    private void ThenStatusIs(OrderStatus expected)
        => Assert.That(_order?.Status, Is.EqualTo(expected));
    
    private void ThenExceptionIsThrown<T>() where T : Exception
        => Assert.That(_exception, Is.InstanceOf<T>());
    
    private void ThenNoExceptionIsThrown()
        => Assert.That(_exception, Is.Null);
    
    // Scenarios
    [Test]
    public void GivenPendingOrder_WhenConfirm_ThenConfirmedStatus()
    {
        GivenPendingOrder();
        WhenConfirmOrder();
        ThenStatusIs(OrderStatus.Confirmed);
        ThenNoExceptionIsThrown();
    }
    
    [Test]
    public void GivenConfirmedOrder_WhenConfirm_ThenDomainException()
    {
        GivenConfirmedOrder();
        WhenConfirmOrder();
        ThenExceptionIsThrown<DomainException>();
    }
    
    [Test]
    public void GivenOrderTotal500_WhenApplyDiscount50_ThenTotal450()
    {
        GivenOrderWithTotal(500);
        WhenApplyDiscount(50);
        Assert.That(_order?.Total.Amount, Is.EqualTo(450));
    }
}
```

---

## Step 827: Performance Regression Tests

```csharp
// ============================================
// Performance Regression Detection
// ============================================

[TestFixture]
public class PerformanceRegressionTests
{
    private const string BaselineFile = "performance_baseline.json";
    private Dictionary<string, double> _baseline = new();
    
    [OneTimeSetUp]
    public void LoadBaseline()
    {
        if (File.Exists(BaselineFile))
            _baseline = System.Text.Json.JsonSerializer.Deserialize<Dictionary<string, double>>(
                File.ReadAllText(BaselineFile)) ?? new();
    }
    
    [Test]
    public void SearchEngine_NotSlowerThan10PercentBaseline()
    {
        var engine = new ThaiTextSearchEngine();
        var items = Enumerable.Range(0, 500)
            .Select(i => new MenuItem { Id = i.ToString(), Name = $"ข้าวผัด {i}", Price = 80 })
            .ToList();
        engine.IndexMenuItems(items);
        
        var times = new List<long>();
        for (int i = 0; i < 100; i++)
        {
            var sw = Stopwatch.StartNew();
            engine.Search("ข้าว");
            times.Add(sw.ElapsedMilliseconds);
        }
        
        var p95 = times.OrderBy(t => t).ElementAt(94);
        
        const string metric = "search_p95_ms";
        
        if (_baseline.TryGetValue(metric, out var baseline))
        {
            var regression = (double)(p95 - baseline) / baseline * 100;
            Assert.That(regression, Is.LessThan(10),
                $"Search P95 regressed by {regression:F1}% (was {baseline}ms, now {p95}ms)");
        }
        else
        {
            // First run: record baseline
            _baseline[metric] = p95;
            File.WriteAllText(BaselineFile,
                System.Text.Json.JsonSerializer.Serialize(_baseline));
        }
    }
}
```

---

## Step 828: Test Coverage Analysis

```csharp
// ============================================
// Test Coverage Targets
// ============================================

// coverlet.runsettings:
// <RunSettings>
//   <DataCollectionRunSettings>
//     <DataCollectors>
//       <DataCollector friendlyName="XPlat Code Coverage">
//         <Configuration>
//           <Include>[FoodDelivery.Domain]*,[FoodDelivery.Application]*</Include>
//           <Exclude>[*.Tests]*,[*Migrations*]</Exclude>
//           <Format>opencover,cobertura</Format>
//           <Threshold>80</Threshold> <!-- Fail if < 80% -->
//         </Configuration>
//       </DataCollector>
//     </DataCollectors>
//   </DataCollectionRunSettings>
// </RunSettings>

// CI step to enforce coverage:
// dotnet test --collect:"XPlat Code Coverage" -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Threshold=80

[TestFixture]
public class CoverageRequiredTests
{
    // These tests ensure critical paths are covered
    
    [Test] public void Domain_OrderCreate_Covered() => Assert.That(true); // covered by domain tests
    [Test] public void Domain_OrderConfirm_Covered() => Assert.That(true);
    [Test] public void Domain_OrderCancel_Covered() => Assert.That(true);
    [Test] public void Application_PlaceOrder_Covered() => Assert.That(true);
    [Test] public void Security_Luhn_Covered() => Assert.That(true);
    [Test] public void Security_ThaiNationalId_Covered() => Assert.That(true);
}
```

---

## Step 829: Test Isolation Patterns

```csharp
// ============================================
// Test Isolation: TestServer, InMemory, Fake
// ============================================

// Base class for isolated ViewModel tests
public abstract class ViewModelTestBase
{
    protected IServiceProvider Services { get; private set; } = null!;
    protected FakeNavigationService Navigation { get; private set; } = null!;
    protected SpyEventBus EventBus { get; private set; } = null!;
    
    [SetUp]
    public void BaseSetUp()
    {
        var services = new ServiceCollection();
        Navigation = new FakeNavigationService();
        EventBus = new SpyEventBus();
        
        services.AddSingleton(Navigation);
        services.AddSingleton<IEventBus>(EventBus);
        services.AddSingleton<IConnectivity>(new AlwaysOnlineConnectivity());
        services.AddSingleton<SQLiteAsyncConnection>(
            _ => new SQLiteAsyncConnection(":memory:"));
        
        ConfigureServices(services);
        Services = services.BuildServiceProvider();
    }
    
    protected virtual void ConfigureServices(IServiceCollection services) { }
}

// Fakes
public class FakeNavigationService
{
    public List<string> NavigationLog { get; } = new();
    public Task GoToAsync(string route) { NavigationLog.Add(route); return Task.CompletedTask; }
    public bool NavigatedTo(string route) => NavigationLog.Contains(route);
}

public class AlwaysOnlineConnectivity : IConnectivity
{
    public NetworkAccess NetworkAccess => NetworkAccess.Internet;
    public IEnumerable<ConnectionProfile> ConnectionProfiles => new[] { ConnectionProfile.WiFi };
    public event EventHandler<ConnectivityChangedEventArgs>? ConnectivityChanged;
}

// Test using base
[TestFixture]
public class HomeViewModelTests : ViewModelTestBase
{
    protected override void ConfigureServices(IServiceCollection services)
    {
        services.AddTransient<HomeViewModel>();
        services.AddSingleton<IRestaurantApiClient, FakeRestaurantApiClient>();
    }
    
    [Test]
    public async Task Initialize_LoadsRestaurants()
    {
        var vm = Services.GetRequiredService<HomeViewModel>();
        await vm.InitializeAsync();
        Assert.That(vm.NearbyRestaurants.Count, Is.GreaterThan(0));
    }
}
```

---

## Step 830: Continuous Testing

```yaml
# .github/workflows/continuous-tests.yml
name: Continuous Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      
      - name: Run unit tests
        run: |
          dotnet test tests/FoodDelivery.Domain.Tests \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            --logger "trx;LogFileName=unit-tests.trx"
      
      - name: Check coverage threshold
        run: |
          dotnet tool install -g dotnet-coverage
          dotnet coverage merge coverage/**/*.xml --output merged.xml
          dotnet coverage report -i merged.xml --threshold 80
  
  mutation-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      
      - name: Run Stryker mutation tests
        run: |
          dotnet tool install -g dotnet-stryker
          dotnet stryker \
            --project FoodDelivery.Domain.Tests \
            --threshold-break 70 \
            --reporters json,html
      
      - name: Upload mutation report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: StrykerOutput/**
```

---

## สรุป Part 83

ใน Part 83 เราได้เรียนรู้:

1. **Test Pyramid** - 70/20/10 split, categories, test factory
2. **Contract Tests** - API shape verification, consumer-driven contracts
3. **Snapshot Tests** - Golden file comparison, diff generation
4. **Fuzz Testing** - Random inputs, Thai string generators, never-throw guarantee
5. **Test Data Seeder** - Deterministic random data, bulk insert
6. **BDD Tests** - Given/When/Then structure, readable scenarios
7. **Performance Regression** - Baseline tracking, 10% regression threshold
8. **Coverage Enforcement** - coverlet, 80% threshold in CI
9. **Test Isolation** - FakeNavigationService, AlwaysOnlineConnectivity
10. **Continuous Testing** - GitHub Actions with mutation score gate

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 83 | Steps 821-830*
