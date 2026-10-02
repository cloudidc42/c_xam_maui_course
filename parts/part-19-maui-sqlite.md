# Part 19: SQLite Database ใน .NET MAUI
## Steps 181-190: Local Database Storage

---

## Step 181: SQLite-net-pcl

```csharp
// ============================================
// SQLite.NET - Local Database
// ============================================

// NuGet packages:
// sqlite-net-pcl
// SQLitePCLRaw.bundle_green

// Model class
using SQLite;

[Table("products")]
public class ProductEntity
{
    [PrimaryKey, AutoIncrement]
    public int Id { get; set; }
    
    [MaxLength(200), NotNull]
    public string Name { get; set; } = string.Empty;
    
    [MaxLength(100)]
    public string? Category { get; set; }
    
    public decimal Price { get; set; }
    
    public int Stock { get; set; }
    
    public bool IsActive { get; set; } = true;
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    [Ignore]
    public string DisplayName => $"{Name} - ฿{Price:N0}";
}

[Table("orders")]
public class OrderEntity
{
    [PrimaryKey, AutoIncrement]
    public int Id { get; set; }
    
    public string CustomerName { get; set; } = string.Empty;
    
    public decimal TotalAmount { get; set; }
    
    public string Status { get; set; } = "Pending";
    
    public DateTime OrderDate { get; set; } = DateTime.UtcNow;
}

[Table("order_items")]
public class OrderItemEntity
{
    [PrimaryKey, AutoIncrement]
    public int Id { get; set; }
    
    [Indexed]
    public int OrderId { get; set; }
    
    public int ProductId { get; set; }
    
    public string ProductName { get; set; } = string.Empty;
    
    public decimal Price { get; set; }
    
    public int Quantity { get; set; }
    
    public decimal Subtotal => Price * Quantity;
}
```

---

## Step 182: Database Connection

```csharp
// ============================================
// Database Service
// ============================================

using SQLite;

public class DatabaseService : IDisposable
{
    private SQLiteAsyncConnection? _db;
    private readonly string _dbPath;
    private bool _initialized;
    
    public DatabaseService()
    {
        _dbPath = Path.Combine(FileSystem.AppDataDirectory, "myapp.db3");
    }
    
    private async Task InitializeAsync()
    {
        if (_initialized) return;
        
        _db = new SQLiteAsyncConnection(_dbPath, 
            SQLiteOpenFlags.ReadWrite | SQLiteOpenFlags.Create | SQLiteOpenFlags.SharedCache);
        
        // Create tables
        await _db.CreateTableAsync<ProductEntity>();
        await _db.CreateTableAsync<OrderEntity>();
        await _db.CreateTableAsync<OrderItemEntity>();
        
        // Seed data if empty
        var count = await _db.Table<ProductEntity>().CountAsync();
        if (count == 0)
            await SeedDataAsync();
        
        _initialized = true;
    }
    
    private async Task SeedDataAsync()
    {
        var products = new[]
        {
            new ProductEntity { Name = "iPhone 15", Category = "Electronics", Price = 35000, Stock = 100 },
            new ProductEntity { Name = "Samsung S24", Category = "Electronics", Price = 30000, Stock = 150 },
            new ProductEntity { Name = "Nike Air Max", Category = "Shoes", Price = 4500, Stock = 200 },
            new ProductEntity { Name = "iPad Pro", Category = "Electronics", Price = 42000, Stock = 50 },
        };
        await _db!.InsertAllAsync(products);
    }
    
    private async Task<SQLiteAsyncConnection> GetConnectionAsync()
    {
        await InitializeAsync();
        return _db!;
    }
    
    public async Task<List<ProductEntity>> GetAllProductsAsync()
    {
        var db = await GetConnectionAsync();
        return await db.Table<ProductEntity>().ToListAsync();
    }
    
    public async Task<List<ProductEntity>> SearchProductsAsync(string keyword)
    {
        var db = await GetConnectionAsync();
        return await db.Table<ProductEntity>()
            .Where(p => p.Name.Contains(keyword) || (p.Category != null && p.Category.Contains(keyword)))
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<ProductEntity?> GetProductByIdAsync(int id)
    {
        var db = await GetConnectionAsync();
        return await db.Table<ProductEntity>().FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public async Task<int> InsertProductAsync(ProductEntity product)
    {
        var db = await GetConnectionAsync();
        return await db.InsertAsync(product);
    }
    
    public async Task<int> UpdateProductAsync(ProductEntity product)
    {
        var db = await GetConnectionAsync();
        return await db.UpdateAsync(product);
    }
    
    public async Task<int> DeleteProductAsync(int id)
    {
        var db = await GetConnectionAsync();
        return await db.DeleteAsync<ProductEntity>(id);
    }
    
    public async Task<int> InsertOrReplaceAsync(ProductEntity product)
    {
        var db = await GetConnectionAsync();
        return await db.InsertOrReplaceAsync(product);
    }
    
    public void Dispose()
    {
        _db?.CloseAsync().GetAwaiter().GetResult();
        _db = null;
    }
}
```

---

## Step 183: Repository Pattern

```csharp
// ============================================
// Generic Repository
// ============================================

public interface IRepository<T> where T : new()
{
    Task<List<T>> GetAllAsync();
    Task<T?> GetByIdAsync(int id);
    Task<int> AddAsync(T entity);
    Task<int> UpdateAsync(T entity);
    Task<int> DeleteAsync(int id);
}

public class Repository<T> : IRepository<T> where T : new()
{
    protected readonly SQLiteAsyncConnection _db;
    
    public Repository(SQLiteAsyncConnection db) => _db = db;
    
    public Task<List<T>> GetAllAsync()
        => _db.Table<T>().ToListAsync();
    
    public Task<T?> GetByIdAsync(int id)
        => _db.GetAsync<T>(id);
    
    public Task<int> AddAsync(T entity)
        => _db.InsertAsync(entity);
    
    public Task<int> UpdateAsync(T entity)
        => _db.UpdateAsync(entity);
    
    public Task<int> DeleteAsync(int id)
        => _db.DeleteAsync<T>(id);
    
    public Task<int> AddRangeAsync(IEnumerable<T> entities)
        => _db.InsertAllAsync(entities);
}

// Specific repository with extra methods
public class ProductRepository : Repository<ProductEntity>
{
    public ProductRepository(SQLiteAsyncConnection db) : base(db) { }
    
    public Task<List<ProductEntity>> GetActivesAsync()
        => _db.Table<ProductEntity>().Where(p => p.IsActive).ToListAsync();
    
    public Task<List<ProductEntity>> GetByCategoryAsync(string category)
        => _db.Table<ProductEntity>()
              .Where(p => p.Category == category)
              .OrderBy(p => p.Name)
              .ToListAsync();
    
    public Task<List<ProductEntity>> GetLowStockAsync(int threshold = 10)
        => _db.Table<ProductEntity>()
              .Where(p => p.Stock <= threshold && p.IsActive)
              .ToListAsync();
    
    public async Task<decimal> GetTotalInventoryValueAsync()
    {
        var products = await GetActivesAsync();
        return products.Sum(p => p.Price * p.Stock);
    }
    
    public Task<int> UpdateStockAsync(int productId, int newStock)
        => _db.ExecuteAsync(
            "UPDATE products SET Stock = ? WHERE Id = ?",
            newStock, productId);
    
    // Raw SQL query
    public Task<List<ProductEntity>> GetTopProductsAsync(int count = 10)
        => _db.QueryAsync<ProductEntity>(
            "SELECT * FROM products WHERE IsActive = 1 ORDER BY Price DESC LIMIT ?",
            count);
}

// ============================================
// Unit of Work
// ============================================

public interface IUnitOfWork : IDisposable
{
    ProductRepository Products { get; }
    Repository<OrderEntity> Orders { get; }
    Repository<OrderItemEntity> OrderItems { get; }
    Task<bool> BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly SQLiteAsyncConnection _db;
    
    public ProductRepository Products { get; }
    public Repository<OrderEntity> Orders { get; }
    public Repository<OrderItemEntity> OrderItems { get; }
    
    public UnitOfWork(string dbPath)
    {
        _db = new SQLiteAsyncConnection(dbPath);
        _db.CreateTablesAsync<ProductEntity, OrderEntity, OrderItemEntity>().GetAwaiter().GetResult();
        
        Products = new ProductRepository(_db);
        Orders = new Repository<OrderEntity>(_db);
        OrderItems = new Repository<OrderItemEntity>(_db);
    }
    
    public async Task<bool> BeginTransactionAsync()
    {
        await _db.ExecuteAsync("BEGIN TRANSACTION");
        return true;
    }
    
    public Task CommitAsync() => _db.ExecuteAsync("COMMIT");
    
    public Task RollbackAsync() => _db.ExecuteAsync("ROLLBACK");
    
    public void Dispose() => _db?.CloseAsync().GetAwaiter().GetResult();
}
```

---

## Step 184: Transactions

```csharp
// ============================================
// Transactions
// ============================================

public class OrderService
{
    private readonly IUnitOfWork _uow;
    
    public OrderService(IUnitOfWork uow) => _uow = uow;
    
    public async Task<OrderEntity> CreateOrderAsync(string customerName, 
        List<(int ProductId, int Quantity)> items)
    {
        await _uow.BeginTransactionAsync();
        
        try
        {
            var order = new OrderEntity
            {
                CustomerName = customerName,
                OrderDate = DateTime.UtcNow,
                Status = "Pending"
            };
            
            await _uow.Orders.AddAsync(order);
            
            decimal total = 0;
            
            foreach (var (productId, qty) in items)
            {
                var product = await _uow.Products.GetByIdAsync(productId)
                    ?? throw new InvalidOperationException($"Product {productId} not found");
                
                if (product.Stock < qty)
                    throw new InvalidOperationException($"Insufficient stock for {product.Name}");
                
                await _uow.OrderItems.AddAsync(new OrderItemEntity
                {
                    OrderId = order.Id,
                    ProductId = productId,
                    ProductName = product.Name,
                    Price = product.Price,
                    Quantity = qty
                });
                
                // Update stock
                await _uow.Products.UpdateStockAsync(productId, product.Stock - qty);
                
                total += product.Price * qty;
            }
            
            order.TotalAmount = total;
            order.Status = "Confirmed";
            await _uow.Orders.UpdateAsync(order);
            
            await _uow.CommitAsync();
            
            return order;
        }
        catch
        {
            await _uow.RollbackAsync();
            throw;
        }
    }
}
```

---

## Step 185: ViewModel กับ Database

```csharp
// ============================================
// ViewModel with Database
// ============================================

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class ProductsDbViewModel : ObservableObject
{
    private readonly DatabaseService _db;
    
    [ObservableProperty]
    private ObservableCollection<ProductEntity> _products = new();
    
    [ObservableProperty]
    private string _searchText = string.Empty;
    
    [ObservableProperty]
    private bool _isLoading;
    
    [ObservableProperty]
    private string _errorMessage = string.Empty;
    
    public ProductsDbViewModel(DatabaseService db) => _db = db;
    
    [RelayCommand]
    private async Task LoadProductsAsync()
    {
        IsLoading = true;
        ErrorMessage = string.Empty;
        
        try
        {
            var items = string.IsNullOrEmpty(SearchText)
                ? await _db.GetAllProductsAsync()
                : await _db.SearchProductsAsync(SearchText);
            
            Products = new ObservableCollection<ProductEntity>(items);
        }
        catch (Exception ex)
        {
            ErrorMessage = $"เกิดข้อผิดพลาด: {ex.Message}";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    partial void OnSearchTextChanged(string value)
    {
        // Debounce search
        _ = DebouncedSearchAsync(value);
    }
    
    private CancellationTokenSource? _searchCts;
    
    private async Task DebouncedSearchAsync(string query)
    {
        _searchCts?.Cancel();
        _searchCts = new CancellationTokenSource();
        
        try
        {
            await Task.Delay(300, _searchCts.Token);
            await LoadProductsAsync();
        }
        catch (OperationCanceledException) { }
    }
    
    [RelayCommand]
    private async Task DeleteProductAsync(ProductEntity product)
    {
        await _db.DeleteProductAsync(product.Id);
        Products.Remove(product);
    }
    
    [RelayCommand]
    private async Task ToggleActiveAsync(ProductEntity product)
    {
        product.IsActive = !product.IsActive;
        await _db.UpdateProductAsync(product);
        
        // Notify item changed
        int idx = Products.IndexOf(product);
        if (idx >= 0)
        {
            Products.RemoveAt(idx);
            Products.Insert(idx, product);
        }
    }
}
```

---

## Step 186: Preferences & SecureStorage

```csharp
// ============================================
// Preferences - key-value storage
// ============================================

public class PreferencesService
{
    private const string ThemeKey = "app_theme";
    private const string LanguageKey = "app_language";
    private const string FontSizeKey = "font_size";
    private const string LastSyncKey = "last_sync";
    
    // Theme
    public string Theme
    {
        get => Preferences.Default.Get(ThemeKey, "System");
        set => Preferences.Default.Set(ThemeKey, value);
    }
    
    // Language
    public string Language
    {
        get => Preferences.Default.Get(LanguageKey, "th");
        set => Preferences.Default.Set(LanguageKey, value);
    }
    
    // Font size
    public int FontSize
    {
        get => Preferences.Default.Get(FontSizeKey, 16);
        set => Preferences.Default.Set(FontSizeKey, value);
    }
    
    // DateTime
    public DateTime? LastSync
    {
        get
        {
            var ticks = Preferences.Default.Get<long>(LastSyncKey, 0);
            return ticks == 0 ? null : new DateTime(ticks, DateTimeKind.Utc);
        }
        set => Preferences.Default.Set(LastSyncKey, value?.Ticks ?? 0L);
    }
    
    public void Clear() => Preferences.Default.Clear();
    
    public void Remove(string key) => Preferences.Default.Remove(key);
    
    public bool ContainsKey(string key) => Preferences.Default.ContainsKey(key);
}

// ============================================
// SecureStorage - encrypted key-value
// ============================================

public class SecureStorageService
{
    public async Task<string?> GetTokenAsync()
        => await SecureStorage.Default.GetAsync("auth_token");
    
    public async Task SaveTokenAsync(string token)
        => await SecureStorage.Default.SetAsync("auth_token", token);
    
    public async Task<string?> GetRefreshTokenAsync()
        => await SecureStorage.Default.GetAsync("refresh_token");
    
    public async Task SaveRefreshTokenAsync(string token)
        => await SecureStorage.Default.SetAsync("refresh_token", token);
    
    public async Task<UserCredentials?> GetCredentialsAsync()
    {
        var username = await SecureStorage.Default.GetAsync("username");
        var password = await SecureStorage.Default.GetAsync("password");
        
        if (username == null || password == null)
            return null;
        
        return new UserCredentials(username, password);
    }
    
    public async Task SaveCredentialsAsync(string username, string password)
    {
        await SecureStorage.Default.SetAsync("username", username);
        await SecureStorage.Default.SetAsync("password", password);
    }
    
    public void ClearAll()
    {
        SecureStorage.Default.RemoveAll();
    }
    
    public void Remove(string key)
    {
        SecureStorage.Default.Remove(key);
    }
}

public record UserCredentials(string Username, string Password);
```

---

## Step 187: File System ใน MAUI

```csharp
// ============================================
// MAUI File System Locations
// ============================================

public class FileSystemService
{
    // App Data Directory - persistent, private
    public string AppDataDir => FileSystem.AppDataDirectory;
    // ~Android: /data/data/<package>/files
    // ~iOS: <app>/Documents
    
    // Cache Directory - can be cleared by OS
    public string CacheDir => FileSystem.CacheDirectory;
    
    // App Package Directory - read-only, bundled files
    public string AppPackageDir => FileSystem.AppPackageDirectory;
    
    // ============================================
    // Bundled resource files
    // ============================================
    
    public async Task<string> ReadBundledFileAsync(string filename)
    {
        using var stream = await FileSystem.OpenAppPackageFileAsync(filename);
        using var reader = new StreamReader(stream);
        return await reader.ReadToEndAsync();
    }
    
    public async Task<T?> ReadBundledJsonAsync<T>(string filename)
    {
        var json = await ReadBundledFileAsync(filename);
        return System.Text.Json.JsonSerializer.Deserialize<T>(json);
    }
    
    // ============================================
    // App data files
    // ============================================
    
    private string GetDataPath(string filename)
        => Path.Combine(AppDataDir, filename);
    
    public async Task SaveJsonAsync<T>(string filename, T data)
    {
        var path = GetDataPath(filename);
        var json = System.Text.Json.JsonSerializer.Serialize(data, new System.Text.Json.JsonSerializerOptions
        {
            WriteIndented = true
        });
        await File.WriteAllTextAsync(path, json);
    }
    
    public async Task<T?> LoadJsonAsync<T>(string filename)
    {
        var path = GetDataPath(filename);
        if (!File.Exists(path)) return default;
        
        var json = await File.ReadAllTextAsync(path);
        return System.Text.Json.JsonSerializer.Deserialize<T>(json);
    }
    
    public bool FileExists(string filename)
        => File.Exists(GetDataPath(filename));
    
    public void DeleteFile(string filename)
    {
        var path = GetDataPath(filename);
        if (File.Exists(path))
            File.Delete(path);
    }
    
    public long GetFileSizeBytes(string filename)
    {
        var path = GetDataPath(filename);
        return File.Exists(path) ? new FileInfo(path).Length : 0;
    }
    
    // Cache
    public async Task<string?> GetCachedDataAsync(string key, TimeSpan maxAge)
    {
        var path = Path.Combine(CacheDir, $"{key}.cache");
        if (!File.Exists(path)) return null;
        
        var info = new FileInfo(path);
        if (DateTime.UtcNow - info.LastWriteTimeUtc > maxAge)
        {
            File.Delete(path);
            return null;
        }
        
        return await File.ReadAllTextAsync(path);
    }
    
    public Task SaveCacheAsync(string key, string data)
        => File.WriteAllTextAsync(Path.Combine(CacheDir, $"{key}.cache"), data);
    
    public void ClearCache()
    {
        foreach (var file in Directory.GetFiles(CacheDir, "*.cache"))
            File.Delete(file);
    }
}
```

---

## Step 188: Settings Page

```csharp
// ============================================
// Settings ViewModel
// ============================================

public partial class SettingsViewModel : ObservableObject
{
    private readonly PreferencesService _prefs;
    private readonly SecureStorageService _secure;
    
    [ObservableProperty]
    private string _selectedTheme;
    
    [ObservableProperty]
    private int _fontSize;
    
    [ObservableProperty]
    private bool _isLoggedIn;
    
    public List<string> ThemeOptions { get; } = new() { "System", "Light", "Dark" };
    public List<int> FontSizeOptions { get; } = new() { 12, 14, 16, 18, 20 };
    
    public SettingsViewModel(PreferencesService prefs, SecureStorageService secure)
    {
        _prefs = prefs;
        _secure = secure;
        _selectedTheme = prefs.Theme;
        _fontSize = prefs.FontSize;
        
        _ = CheckLoginAsync();
    }
    
    partial void OnSelectedThemeChanged(string value)
    {
        _prefs.Theme = value;
        Application.Current!.UserAppTheme = value switch
        {
            "Dark" => AppTheme.Dark,
            "Light" => AppTheme.Light,
            _ => AppTheme.Unspecified
        };
    }
    
    partial void OnFontSizeChanged(int value)
        => _prefs.FontSize = value;
    
    private async Task CheckLoginAsync()
        => IsLoggedIn = await _secure.GetTokenAsync() != null;
    
    [RelayCommand]
    private async Task LogoutAsync()
    {
        _secure.ClearAll();
        IsLoggedIn = false;
        await Shell.Current.GoToAsync("//login");
    }
    
    [RelayCommand]
    private void ClearCache()
    {
        // Clear app cache
        _prefs.LastSync = null;
    }
    
    [RelayCommand]
    private async Task ExportDataAsync()
    {
        // Share file
        var file = Path.Combine(FileSystem.CacheDirectory, "export.json");
        await File.WriteAllTextAsync(file, "{}");
        
        await Share.Default.RequestAsync(new ShareFileRequest
        {
            Title = "Export Data",
            File = new ShareFile(file)
        });
    }
}
```

---

## Step 189: Background Sync

```csharp
// ============================================
// Background Sync Service
// ============================================

public class SyncService
{
    private readonly DatabaseService _db;
    private readonly HttpClient _http;
    private readonly PreferencesService _prefs;
    private CancellationTokenSource? _cts;
    
    public event EventHandler<SyncEventArgs>? SyncCompleted;
    
    public SyncService(DatabaseService db, HttpClient http, PreferencesService prefs)
    {
        _db = db;
        _http = http;
        _prefs = prefs;
    }
    
    public async Task SyncAsync(CancellationToken ct = default)
    {
        try
        {
            // Pull from server
            var response = await _http.GetStringAsync("https://api.example.com/products", ct);
            var products = System.Text.Json.JsonSerializer.Deserialize<List<ProductEntity>>(response);
            
            if (products != null)
            {
                foreach (var product in products)
                    await _db.InsertOrReplaceAsync(product);
            }
            
            _prefs.LastSync = DateTime.UtcNow;
            SyncCompleted?.Invoke(this, new SyncEventArgs(true, products?.Count ?? 0));
        }
        catch (Exception ex)
        {
            SyncCompleted?.Invoke(this, new SyncEventArgs(false, 0, ex.Message));
        }
    }
    
    public void StartPeriodicSync(TimeSpan interval)
    {
        _cts = new CancellationTokenSource();
        _ = RunPeriodicSyncAsync(interval, _cts.Token);
    }
    
    public void StopPeriodicSync() => _cts?.Cancel();
    
    private async Task RunPeriodicSyncAsync(TimeSpan interval, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await SyncAsync(ct);
            await Task.Delay(interval, ct);
        }
    }
}

public record SyncEventArgs(bool Success, int RecordsCount, string? Error = null);
```

---

## Step 190: Complete App Architecture

```csharp
// ============================================
// MauiProgram.cs with Database DI
// ============================================

/*
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();
        
        // Database
        string dbPath = Path.Combine(FileSystem.AppDataDirectory, "myapp.db3");
        builder.Services.AddSingleton<SQLiteAsyncConnection>(
            _ => new SQLiteAsyncConnection(dbPath));
        builder.Services.AddSingleton<DatabaseService>();
        builder.Services.AddSingleton<ProductRepository>();
        builder.Services.AddSingleton<IUnitOfWork>(sp =>
            new UnitOfWork(dbPath));
        
        // Services
        builder.Services.AddSingleton<PreferencesService>();
        builder.Services.AddSingleton<SecureStorageService>();
        builder.Services.AddSingleton<FileSystemService>();
        builder.Services.AddSingleton<SyncService>();
        
        // ViewModels
        builder.Services.AddTransient<ProductsDbViewModel>();
        builder.Services.AddTransient<SettingsViewModel>();
        
        // Pages
        builder.Services.AddTransient<ProductsPage3>();
        builder.Services.AddTransient<SettingsPage3>();
        
        return builder.Build();
    }
}
*/

// ============================================
// Dummy page class declarations
// ============================================
public class ProductsPage3 : ContentPage { }
public class SettingsPage3 : ContentPage { }
```

---

## สรุป Part 19

ใน Part 19 เราได้เรียนรู้:

1. **SQLite.NET** - ORM สำหรับ local database
2. **Table Attributes** - Table, PrimaryKey, AutoIncrement, Indexed
3. **CRUD Operations** - Insert, Select, Update, Delete
4. **Repository Pattern** - Generic และ Specific repositories
5. **Unit of Work** - จัดการ transactions
6. **Transactions** - BEGIN/COMMIT/ROLLBACK
7. **Preferences** - Key-value storage สำหรับ settings
8. **SecureStorage** - Encrypted storage สำหรับ sensitive data
9. **File System** - AppDataDirectory, CacheDirectory
10. **Background Sync** - Periodic sync กับ server

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 19 | Steps 181-190*
