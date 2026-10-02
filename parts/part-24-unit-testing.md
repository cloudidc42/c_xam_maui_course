# Part 24: Unit Testing ใน C# และ MAUI
## Steps 231-240: Testing Strategies

---

## Step 231: Unit Testing Basics

```csharp
// ============================================
// xUnit + FluentAssertions + Moq
// ============================================

// NuGet:
// xunit
// xunit.runner.visualstudio
// FluentAssertions
// Moq
// Microsoft.NET.Test.Sdk

// Basic Test
public class CalculatorTests
{
    private readonly Calculator _sut; // system under test
    
    public CalculatorTests()
    {
        _sut = new Calculator();
    }
    
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange
        int a = 5, b = 3;
        
        // Act
        int result = _sut.Add(a, b);
        
        // Assert
        result.Should().Be(8);
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsException()
    {
        // Arrange + Act
        Action act = () => _sut.Divide(10, 0);
        
        // Assert
        act.Should().Throw<DivideByZeroException>()
            .WithMessage("*zero*");
    }
    
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    [InlineData(100, -50, 50)]
    public void Add_Various_ReturnsExpected(int a, int b, int expected)
    {
        _sut.Add(a, b).Should().Be(expected);
    }
    
    [Theory]
    [MemberData(nameof(GetTestData))]
    public void Multiply_DataDriven(int a, int b, int expected)
    {
        _sut.Multiply(a, b).Should().Be(expected);
    }
    
    public static IEnumerable<object[]> GetTestData()
    {
        yield return new object[] { 2, 3, 6 };
        yield return new object[] { -2, 3, -6 };
        yield return new object[] { 0, 100, 0 };
    }
}

public class Calculator
{
    public int Add(int a, int b) => a + b;
    public int Multiply(int a, int b) => a * b;
    public int Divide(int a, int b)
    {
        if (b == 0) throw new DivideByZeroException("Cannot divide by zero");
        return a / b;
    }
}
```

---

## Step 232: Testing with Moq

```csharp
// ============================================
// Mocking Dependencies with Moq
// ============================================

using Moq;
using FluentAssertions;

public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    private readonly Mock<ILogger<ProductService>> _mockLogger;
    private readonly ProductService _sut;
    
    public ProductServiceTests()
    {
        _mockRepo = new Mock<IProductRepository>();
        _mockLogger = new Mock<ILogger<ProductService>>();
        _sut = new ProductService(_mockRepo.Object, _mockLogger.Object);
    }
    
    [Fact]
    public async Task GetAllAsync_ReturnsProducts()
    {
        // Arrange
        var products = new List<ProductDto>
        {
            new(1, "iPhone", "Electronics", 35000, 100, null, DateTime.UtcNow),
            new(2, "iPad", "Electronics", 42000, 50, null, DateTime.UtcNow)
        };
        
        _mockRepo.Setup(r => r.GetAllAsync())
                 .ReturnsAsync(products);
        
        // Act
        var result = await _sut.GetAllAsync();
        
        // Assert
        result.Should().HaveCount(2);
        result.Should().ContainSingle(p => p.Name == "iPhone");
        
        // Verify interaction
        _mockRepo.Verify(r => r.GetAllAsync(), Times.Once);
    }
    
    [Fact]
    public async Task GetByIdAsync_NotFound_ThrowsNotFoundException()
    {
        // Arrange
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
                 .ReturnsAsync((ProductDto?)null);
        
        // Act
        Func<Task> act = () => _sut.GetByIdAsync(999);
        
        // Assert
        await act.Should().ThrowAsync<NotFoundException>()
            .WithMessage("*999*");
    }
    
    [Fact]
    public async Task CreateAsync_ValidProduct_ReturnsCreated()
    {
        // Arrange
        var dto = new ProductCreateDto("New Product", "Electronics", 1000, 10);
        var expectedProduct = new ProductDto(1, dto.Name, dto.Category, 
            dto.Price, dto.Stock, null, DateTime.UtcNow);
        
        _mockRepo.Setup(r => r.AddAsync(It.IsAny<ProductDto>()))
                 .ReturnsAsync(expectedProduct);
        
        // Act
        var result = await _sut.CreateAsync(dto);
        
        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be("New Product");
        result.Price.Should().Be(1000);
        
        _mockRepo.Verify(r => r.AddAsync(It.Is<ProductDto>(p => p.Name == "New Product")), Times.Once);
    }
    
    [Fact]
    public async Task DeleteAsync_WhenProductExists_CallsRepository()
    {
        // Arrange
        var product = new ProductDto(1, "Test", "Cat", 100, 10, null, DateTime.UtcNow);
        _mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(product);
        _mockRepo.Setup(r => r.DeleteAsync(1)).ReturnsAsync(true);
        
        // Act
        await _sut.DeleteAsync(1);
        
        // Assert
        _mockRepo.Verify(r => r.DeleteAsync(1), Times.Once);
    }
    
    // Mock sequence
    [Fact]
    public async Task RetryOnFailure_EventuallySucceeds()
    {
        int callCount = 0;
        _mockRepo.Setup(r => r.GetAllAsync())
            .ReturnsAsync(() =>
            {
                callCount++;
                if (callCount < 3) throw new HttpRequestException("Network error");
                return new List<ProductDto>();
            });
        
        var result = await RetryPolicy.ExecuteAsync(() => _sut.GetAllAsync());
        
        result.Should().NotBeNull();
        callCount.Should().Be(3);
    }
}

// Interfaces & implementations for testing
public interface IProductRepository
{
    Task<List<ProductDto>> GetAllAsync();
    Task<ProductDto?> GetByIdAsync(int id);
    Task<ProductDto> AddAsync(ProductDto product);
    Task<bool> DeleteAsync(int id);
}

public class ProductService
{
    private readonly IProductRepository _repo;
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(IProductRepository repo, ILogger<ProductService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
    
    public Task<List<ProductDto>> GetAllAsync() => _repo.GetAllAsync();
    
    public async Task<ProductDto> GetByIdAsync(int id)
    {
        var product = await _repo.GetByIdAsync(id);
        if (product == null) throw new NotFoundException($"Product {id} not found");
        return product;
    }
    
    public Task<ProductDto> CreateAsync(ProductCreateDto dto)
    {
        var product = new ProductDto(0, dto.Name, dto.Category, dto.Price, dto.Stock, null, DateTime.UtcNow);
        return _repo.AddAsync(product);
    }
    
    public async Task DeleteAsync(int id)
    {
        await GetByIdAsync(id); // throws if not found
        await _repo.DeleteAsync(id);
    }
}

public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}
```

---

## Step 233: Testing ViewModels

```csharp
// ============================================
// ViewModel Unit Tests
// ============================================

public class ProductsViewModelTests
{
    private readonly Mock<IProductApiService> _mockApi;
    private readonly Mock<INavService> _mockNav;
    private readonly ProductsApiViewModel _sut;
    
    public ProductsViewModelTests()
    {
        _mockApi = new Mock<IProductApiService>();
        _mockNav = new Mock<INavService>();
        _sut = new ProductsApiViewModel(_mockApi.Object, _mockNav.Object);
    }
    
    [Fact]
    public async Task LoadProductsCommand_Success_PopulatesProducts()
    {
        // Arrange
        var products = new List<ProductDto>
        {
            new(1, "Product 1", "Cat", 100, 10, null, DateTime.UtcNow),
            new(2, "Product 2", "Cat", 200, 20, null, DateTime.UtcNow),
        };
        
        _mockApi.Setup(a => a.GetProductsAsync(
            It.IsAny<int>(), It.IsAny<int>(), 
            It.IsAny<string?>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(ApiResult<PaginatedResult<ProductDto>>.Success(
                new PaginatedResult<ProductDto>(products, 2, 1, 20)));
        
        // Act
        await _sut.LoadProductsCommand.ExecuteAsync(null);
        
        // Assert
        _sut.Products.Should().HaveCount(2);
        _sut.IsLoading.Should().BeFalse();
        _sut.HasError.Should().BeFalse();
    }
    
    [Fact]
    public async Task LoadProductsCommand_Failure_SetsError()
    {
        // Arrange
        _mockApi.Setup(a => a.GetProductsAsync(
            It.IsAny<int>(), It.IsAny<int>(),
            It.IsAny<string?>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(ApiResult<PaginatedResult<ProductDto>>.Failure(
                new ApiError("SERVER_ERROR", "Internal server error"), 500));
        
        // Act
        await _sut.LoadProductsCommand.ExecuteAsync(null);
        
        // Assert
        _sut.HasError.Should().BeTrue();
        _sut.ErrorMessage.Should().Contain("server error");
        _sut.Products.Should().BeEmpty();
        _sut.IsLoading.Should().BeFalse();
    }
    
    [Fact]
    public async Task DeleteProductCommand_ConfirmedByUser_RemovesFromList()
    {
        // Arrange
        var product = new ProductDto(1, "Test Product", "Cat", 100, 10, null, DateTime.UtcNow);
        _sut.Products.Add(product);
        
        _mockNav.Setup(n => n.ShowConfirmAsync(
            It.IsAny<string>(), It.IsAny<string>(),
            It.IsAny<string>(), It.IsAny<string>()))
            .ReturnsAsync(true);
        
        _mockApi.Setup(a => a.DeleteProductAsync(1, It.IsAny<CancellationToken>()))
            .ReturnsAsync(ApiResult<bool>.Success(true));
        
        // Act
        await _sut.DeleteProductCommand.ExecuteAsync(product);
        
        // Assert
        _sut.Products.Should().BeEmpty();
        _mockApi.Verify(a => a.DeleteProductAsync(1, It.IsAny<CancellationToken>()), Times.Once);
    }
    
    [Fact]
    public async Task DeleteProductCommand_DeclinedByUser_KeepsItem()
    {
        // Arrange
        var product = new ProductDto(1, "Test Product", "Cat", 100, 10, null, DateTime.UtcNow);
        _sut.Products.Add(product);
        
        _mockNav.Setup(n => n.ShowConfirmAsync(
            It.IsAny<string>(), It.IsAny<string>(),
            It.IsAny<string>(), It.IsAny<string>()))
            .ReturnsAsync(false);
        
        // Act
        await _sut.DeleteProductCommand.ExecuteAsync(product);
        
        // Assert
        _sut.Products.Should().HaveCount(1);
        _mockApi.Verify(a => a.DeleteProductAsync(It.IsAny<int>(), 
            It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

---

## Step 234: Integration Tests

```csharp
// ============================================
// Integration Tests (SQLite in-memory)
// ============================================

public class DatabaseIntegrationTests : IDisposable
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ProductRepository _repo;
    
    public DatabaseIntegrationTests()
    {
        // Use in-memory SQLite for tests
        _db = new SQLiteAsyncConnection(":memory:");
        _db.CreateTableAsync<ProductEntity>().GetAwaiter().GetResult();
        _repo = new ProductRepository(_db);
    }
    
    [Fact]
    public async Task AddAsync_ValidProduct_ReturnsId()
    {
        var product = new ProductEntity
        {
            Name = "Test Product",
            Category = "Electronics",
            Price = 1000,
            Stock = 10
        };
        
        int rowsAffected = await _repo.AddAsync(product);
        
        rowsAffected.Should().Be(1);
        product.Id.Should().BeGreaterThan(0);
    }
    
    [Fact]
    public async Task GetAllAsync_AfterInserts_ReturnsAll()
    {
        await _repo.AddAsync(new ProductEntity { Name = "P1", Price = 100, Stock = 10 });
        await _repo.AddAsync(new ProductEntity { Name = "P2", Price = 200, Stock = 20 });
        await _repo.AddAsync(new ProductEntity { Name = "P3", Price = 300, Stock = 30 });
        
        var all = await _repo.GetAllAsync();
        
        all.Should().HaveCount(3);
    }
    
    [Fact]
    public async Task UpdateAsync_ChangesValue()
    {
        var product = new ProductEntity { Name = "Old Name", Price = 100, Stock = 5 };
        await _repo.AddAsync(product);
        
        product.Name = "New Name";
        product.Price = 200;
        await _repo.UpdateAsync(product);
        
        var updated = await _repo.GetByIdAsync(product.Id);
        
        updated.Should().NotBeNull();
        updated!.Name.Should().Be("New Name");
        updated.Price.Should().Be(200);
    }
    
    [Fact]
    public async Task DeleteAsync_RemovesProduct()
    {
        var product = new ProductEntity { Name = "To Delete", Price = 100, Stock = 1 };
        await _repo.AddAsync(product);
        
        await _repo.DeleteAsync(product.Id);
        
        var found = await _repo.GetByIdAsync(product.Id);
        found.Should().BeNull();
    }
    
    [Fact]
    public async Task GetLowStockAsync_ReturnsOnlyLowStock()
    {
        await _repo.AddAsync(new ProductEntity { Name = "Normal", Stock = 50 });
        await _repo.AddAsync(new ProductEntity { Name = "Low", Stock = 3, IsActive = true });
        await _repo.AddAsync(new ProductEntity { Name = "Out", Stock = 0, IsActive = true });
        
        var lowStock = await _repo.GetLowStockAsync(5);
        
        lowStock.Should().HaveCount(2); // Low and Out
        lowStock.Should().NotContain(p => p.Name == "Normal");
    }
    
    public void Dispose() => _db.CloseAsync().GetAwaiter().GetResult();
}
```

---

## Step 235: Test Fixtures และ Shared Context

```csharp
// ============================================
// Test Fixtures
// ============================================

// Shared expensive resource across tests
public class DatabaseFixture : IDisposable
{
    public SQLiteAsyncConnection Db { get; }
    public ProductRepository Products { get; }
    
    public DatabaseFixture()
    {
        Db = new SQLiteAsyncConnection(":memory:");
        Db.CreateTablesAsync<ProductEntity, OrderEntity, OrderItemEntity>()
           .GetAwaiter().GetResult();
        Products = new ProductRepository(Db);
        
        // Seed test data
        SeedDataAsync().GetAwaiter().GetResult();
    }
    
    private async Task SeedDataAsync()
    {
        await Db.InsertAllAsync(new[]
        {
            new ProductEntity { Name = "Product A", Category = "Electronics", Price = 1000, Stock = 100, IsActive = true },
            new ProductEntity { Name = "Product B", Category = "Electronics", Price = 2000, Stock = 50, IsActive = true },
            new ProductEntity { Name = "Product C", Category = "Shoes", Price = 500, Stock = 5, IsActive = true },
            new ProductEntity { Name = "Inactive", Category = "Electronics", Price = 100, Stock = 0, IsActive = false },
        });
    }
    
    public void Dispose() => Db.CloseAsync().GetAwaiter().GetResult();
}

// Share fixture across tests in same class
public class ProductQueryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    
    public ProductQueryTests(DatabaseFixture fixture) => _fixture = fixture;
    
    [Fact]
    public async Task GetActivesAsync_ExcludesInactive()
    {
        var actives = await _fixture.Products.GetActivesAsync();
        actives.Should().NotContain(p => !p.IsActive);
        actives.Should().HaveCount(3);
    }
    
    [Fact]
    public async Task GetByCategoryAsync_Electronics_ReturnsTwo()
    {
        var electronics = await _fixture.Products.GetByCategoryAsync("Electronics");
        electronics.Should().HaveCount(2);
    }
    
    [Fact]
    public async Task GetLowStockAsync_Threshold5_ReturnsOne()
    {
        var lowStock = await _fixture.Products.GetLowStockAsync(5);
        lowStock.Should().ContainSingle(p => p.Name == "Product C");
    }
}
```

---

## Step 236: Testing Async Code

```csharp
// ============================================
// Testing Async Methods
// ============================================

public class AsyncTests
{
    [Fact]
    public async Task AsyncMethod_CompletesWithinTimeout()
    {
        var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        
        // Should complete before cancellation
        var result = await SomeAsyncOperation(cts.Token);
        
        result.Should().BeTrue();
    }
    
    [Fact]
    public async Task CancellableOperation_WhenCancelled_ThrowsOperationCancelled()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel(); // Cancel immediately
        
        Func<Task> act = async () => await SomeAsyncOperation(cts.Token);
        
        await act.Should().ThrowAsync<OperationCanceledException>();
    }
    
    [Fact]
    public async Task Retry_AfterTransientFailures_Succeeds()
    {
        int attempts = 0;
        
        var result = await RetryPolicy.ExecuteAsync(async () =>
        {
            attempts++;
            if (attempts < 3)
                throw new HttpRequestException("Transient error");
            
            return "success";
        }, maxRetries: 3);
        
        result.Should().Be("success");
        attempts.Should().Be(3);
    }
    
    [Fact]
    public async Task Observable_RaisesPropertyChanged()
    {
        var vm = new SimpleViewModel();
        var changes = new List<string>();
        
        vm.PropertyChanged += (_, e) => changes.Add(e.PropertyName!);
        
        vm.Name = "Test";
        vm.Name = "Changed";
        
        changes.Should().ContainInOrder("Name", "Name");
    }
    
    private static async Task<bool> SomeAsyncOperation(CancellationToken ct)
    {
        await Task.Delay(100, ct);
        return true;
    }
}

public partial class SimpleViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = string.Empty;
}
```

---

## Step 237: Snapshot Testing

```csharp
// ============================================
// Snapshot / Approval Testing
// ============================================

// NuGet: Verify.Xunit

// Example using simple JSON comparison
public class SnapshotTests
{
    private readonly string _snapshotDir;
    
    public SnapshotTests()
    {
        _snapshotDir = Path.Combine(AppContext.BaseDirectory, "Snapshots");
        Directory.CreateDirectory(_snapshotDir);
    }
    
    [Fact]
    public async Task ProductList_MatchesSnapshot()
    {
        // Arrange
        var products = new List<ProductDto>
        {
            new(1, "iPhone", "Electronics", 35000, 100, null, new DateTime(2024, 1, 1)),
            new(2, "iPad", "Electronics", 42000, 50, null, new DateTime(2024, 1, 2)),
        };
        
        var json = System.Text.Json.JsonSerializer.Serialize(products, 
            new System.Text.Json.JsonSerializerOptions { WriteIndented = true });
        
        var snapshotPath = Path.Combine(_snapshotDir, "product_list.json");
        
        if (!File.Exists(snapshotPath))
        {
            // First run - create snapshot
            await File.WriteAllTextAsync(snapshotPath, json);
            return;
        }
        
        // Subsequent runs - compare
        var snapshot = await File.ReadAllTextAsync(snapshotPath);
        json.Should().Be(snapshot);
    }
}
```

---

## Step 238: Code Coverage

```csharp
// ============================================
// Code Coverage với Coverlet
// ============================================

// project.csproj:
/*
<ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.0" />
</ItemGroup>
*/

// Run tests with coverage:
// dotnet test --collect:"XPlat Code Coverage"

// Generate report (NuGet: dotnet-reportgenerator-globaltool):
// reportgenerator -reports:coverage.xml -targetdir:coveragereport

// ============================================
// Test Helpers
// ============================================

public static class TestHelpers
{
    // Build mock HttpClient
    public static HttpClient CreateMockHttpClient(
        Dictionary<string, (int StatusCode, string Content)> responses)
    {
        var handler = new MockHttpMessageHandler(responses);
        return new HttpClient(handler) { BaseAddress = new Uri("https://api.test.com/") };
    }
    
    // Create in-memory database
    public static async Task<SQLiteAsyncConnection> CreateTestDatabaseAsync<T>()
        where T : new()
    {
        var db = new SQLiteAsyncConnection(":memory:");
        await db.CreateTableAsync<T>();
        return db;
    }
    
    // Wait for condition
    public static async Task WaitForAsync(Func<bool> condition, 
        TimeSpan? timeout = null)
    {
        var deadline = DateTime.UtcNow.Add(timeout ?? TimeSpan.FromSeconds(5));
        
        while (!condition() && DateTime.UtcNow < deadline)
            await Task.Delay(50);
        
        condition().Should().BeTrue("condition was never met");
    }
}

// Mock HttpMessageHandler
public class MockHttpMessageHandler : HttpMessageHandler
{
    private readonly Dictionary<string, (int StatusCode, string Content)> _responses;
    
    public MockHttpMessageHandler(Dictionary<string, (int StatusCode, string Content)> responses)
        => _responses = responses;
    
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var path = request.RequestUri?.PathAndQuery ?? "/";
        
        if (_responses.TryGetValue(path, out var response))
        {
            return Task.FromResult(new HttpResponseMessage
            {
                StatusCode = (System.Net.HttpStatusCode)response.StatusCode,
                Content = new StringContent(response.Content, 
                    System.Text.Encoding.UTF8, "application/json")
            });
        }
        
        return Task.FromResult(new HttpResponseMessage
        {
            StatusCode = System.Net.HttpStatusCode.NotFound
        });
    }
}
```

---

## Step 239: Testing Best Practices

```csharp
// ============================================
// Test Organization Best Practices
// ============================================

// 1. ARRANGE-ACT-ASSERT pattern
[Fact]
public void Test_FollowsAaa()
{
    // ====== ARRANGE ======
    var service = new OrderService(new Mock<IUnitOfWork>().Object);
    
    // ====== ACT ======
    // var result = service.DoSomething();
    
    // ====== ASSERT ======
    // result.Should().Be(expected);
}

// 2. One assertion per test (ideally)
// BAD:
[Fact]
public void BadTest_MultipleAssertions()
{
    var product = new ProductEntity { Name = "Test", Price = 100, Stock = 10 };
    product.Name.Should().Be("Test");
    product.Price.Should().Be(100);
    product.Stock.Should().Be(10);
    // If first fails, others don't run
}

// GOOD:
public class ProductEntityTests
{
    [Fact] public void Name_IsSetCorrectly() { }
    [Fact] public void Price_IsSetCorrectly() { }
    [Fact] public void Stock_IsSetCorrectly() { }
}

// 3. Descriptive names: Method_Condition_ExpectedResult
[Fact]
public async Task Login_WithValidCredentials_ReturnsToken() { }

[Fact]
public async Task Login_WithInvalidPassword_ThrowsUnauthorizedException() { }

[Fact]
public async Task Login_WithEmptyEmail_ThrowsValidationException() { }

// 4. Don't test implementation, test behavior
// BAD: Tests that _mockRepo.GetAllAsync was called exactly once
// GOOD: Tests that the returned data matches expected

// 5. Use builder pattern for complex test data
public class ProductBuilder
{
    private ProductEntity _product = new()
    {
        Name = "Default Product",
        Category = "Default",
        Price = 100,
        Stock = 10,
        IsActive = true
    };
    
    public ProductBuilder WithName(string name) { _product.Name = name; return this; }
    public ProductBuilder WithPrice(decimal price) { _product.Price = price; return this; }
    public ProductBuilder WithStock(int stock) { _product.Stock = stock; return this; }
    public ProductBuilder Inactive() { _product.IsActive = false; return this; }
    
    public ProductEntity Build() => _product;
    
    // Usage: var product = new ProductBuilder().WithName("iPhone").WithPrice(35000).Build();
}
```

---

## Step 240: Performance Tests

```csharp
// ============================================
// BenchmarkDotNet - Performance Testing
// ============================================

// NuGet: BenchmarkDotNet

using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob]
public class StringBenchmarks
{
    private const int N = 1000;
    private readonly string[] _words = Enumerable.Range(1, N)
        .Select(i => $"word{i}").ToArray();
    
    [Benchmark(Baseline = true)]
    public string StringConcatenation()
    {
        var result = "";
        foreach (var word in _words)
            result += word + " ";
        return result;
    }
    
    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var word in _words)
            sb.Append(word).Append(' ');
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringJoin()
        => string.Join(" ", _words);
    
    [Benchmark]
    public string LinqAggregate()
        => _words.Aggregate((a, b) => a + " " + b);
}

// Run benchmarks:
// BenchmarkRunner.Run<StringBenchmarks>();

// ============================================
// Quick Performance Test
// ============================================

public class PerformanceTests
{
    [Fact]
    public async Task LoadProducts_CompletesWithin500ms()
    {
        var mock = new Mock<IProductApiService>();
        mock.Setup(a => a.GetProductsAsync(
            It.IsAny<int>(), It.IsAny<int>(),
            It.IsAny<string?>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(ApiResult<PaginatedResult<ProductDto>>.Success(
                new PaginatedResult<ProductDto>(new List<ProductDto>(), 0, 1, 20)));
        
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        var vm = new ProductsApiViewModel(mock.Object, new Mock<INavService>().Object);
        await vm.LoadProductsCommand.ExecuteAsync(null);
        
        sw.Stop();
        
        sw.ElapsedMilliseconds.Should().BeLessThan(500,
            "product loading should complete within 500ms");
    }
}
```

---

## สรุป Part 24

ใน Part 24 เราได้เรียนรู้:

1. **xUnit** - Fact, Theory, InlineData, MemberData
2. **FluentAssertions** - Readable assertions
3. **Moq** - Mocking interfaces, Verify interactions
4. **Testing ViewModels** - Mock services, test commands
5. **Integration Tests** - SQLite in-memory
6. **Test Fixtures** - Shared context (IClassFixture)
7. **Async Testing** - Task assertions, CancellationToken
8. **Snapshot Testing** - Compare JSON output
9. **Code Coverage** - Coverlet, reportgenerator
10. **Best Practices** - AAA pattern, naming, builders
11. **Performance Tests** - BenchmarkDotNet

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 24 | Steps 231-240*
