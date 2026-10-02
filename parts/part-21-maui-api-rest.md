# Part 21: REST API ใน .NET MAUI
## Steps 201-210: การเชื่อมต่อ API

---

## Step 201: HttpClient Setup ใน MAUI

```csharp
// ============================================
// HttpClient Best Practices ใน MAUI
// ============================================

// MauiProgram.cs
public static class MauiProgram2
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();
        
        // Named HttpClient
        builder.Services.AddHttpClient("MainApi", client =>
        {
            client.BaseAddress = new Uri("https://api.myapp.com/");
            client.Timeout = TimeSpan.FromSeconds(30);
            client.DefaultRequestHeaders.Add("Accept", "application/json");
            client.DefaultRequestHeaders.Add("X-App-Version", AppInfo.Current.VersionString);
        });
        
        // Typed HttpClient
        builder.Services.AddHttpClient<IProductApiService, ProductApiService>(client =>
        {
            client.BaseAddress = new Uri("https://api.myapp.com/");
            client.Timeout = TimeSpan.FromSeconds(30);
        });
        
        // With DelegatingHandler (middleware)
        builder.Services.AddTransient<AuthHeaderHandler>();
        builder.Services.AddHttpClient<IAuthApiService, AuthApiService>()
            .AddHttpMessageHandler<AuthHeaderHandler>();
        
        return builder.Build();
    }
}

// ============================================
// Auth Header Handler
// ============================================

public class AuthHeaderHandler : DelegatingHandler
{
    private readonly SecureStorageService _secure;
    
    public AuthHeaderHandler(SecureStorageService secure) => _secure = secure;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _secure.GetTokenAsync();
        
        if (!string.IsNullOrEmpty(token))
            request.Headers.Authorization = 
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        
        var response = await base.SendAsync(request, ct);
        
        // Auto refresh on 401
        if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            var refreshToken = await _secure.GetRefreshTokenAsync();
            if (!string.IsNullOrEmpty(refreshToken))
            {
                // Try to refresh
                // var newToken = await RefreshTokenAsync(refreshToken, ct);
                // if (newToken != null) { ... retry ... }
            }
        }
        
        return response;
    }
}
```

---

## Step 202: API Service Interface

```csharp
// ============================================
// Domain Models (DTOs)
// ============================================

public record ProductDto(
    int Id,
    string Name,
    string? Category,
    decimal Price,
    int Stock,
    string? ImageUrl,
    DateTime CreatedAt
);

public record ProductCreateDto(string Name, string? Category, decimal Price, int Stock);
public record ProductUpdateDto(string Name, string? Category, decimal Price, int Stock);

public record PaginatedResult<T>(
    List<T> Items,
    int TotalCount,
    int Page,
    int PageSize
)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

public record ApiError(string Code, string Message, Dictionary<string, string[]>? Errors = null);

// ============================================
// Result Type for API
// ============================================

public class ApiResult<T>
{
    public bool IsSuccess { get; }
    public T? Data { get; }
    public ApiError? Error { get; }
    public int StatusCode { get; }
    
    private ApiResult(bool success, T? data, ApiError? error, int statusCode)
    {
        IsSuccess = success;
        Data = data;
        Error = error;
        StatusCode = statusCode;
    }
    
    public static ApiResult<T> Success(T data, int statusCode = 200)
        => new(true, data, null, statusCode);
    
    public static ApiResult<T> Failure(ApiError error, int statusCode)
        => new(false, default, error, statusCode);
    
    public static ApiResult<T> NetworkError(string message)
        => new(false, default, new ApiError("NETWORK_ERROR", message), 0);
}
```

---

## Step 203: API Service Implementation

```csharp
// ============================================
// API Service
// ============================================

using System.Net.Http.Json;
using System.Text.Json;

public interface IProductApiService
{
    Task<ApiResult<PaginatedResult<ProductDto>>> GetProductsAsync(
        int page = 1, int size = 20, string? category = null, CancellationToken ct = default);
    Task<ApiResult<ProductDto>> GetProductAsync(int id, CancellationToken ct = default);
    Task<ApiResult<ProductDto>> CreateProductAsync(ProductCreateDto dto, CancellationToken ct = default);
    Task<ApiResult<ProductDto>> UpdateProductAsync(int id, ProductUpdateDto dto, CancellationToken ct = default);
    Task<ApiResult<bool>> DeleteProductAsync(int id, CancellationToken ct = default);
}

public class ProductApiService : IProductApiService
{
    private readonly HttpClient _http;
    
    private static readonly JsonSerializerOptions _jsonOptions = new()
    {
        PropertyNameCaseInsensitive = true,
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };
    
    public ProductApiService(HttpClient http) => _http = http;
    
    public async Task<ApiResult<PaginatedResult<ProductDto>>> GetProductsAsync(
        int page = 1, int size = 20, string? category = null, CancellationToken ct = default)
    {
        try
        {
            var query = $"products?page={page}&size={size}";
            if (!string.IsNullOrEmpty(category))
                query += $"&category={Uri.EscapeDataString(category)}";
            
            var response = await _http.GetAsync(query, ct);
            
            if (!response.IsSuccessStatusCode)
                return await HandleErrorAsync<PaginatedResult<ProductDto>>(response);
            
            var result = await response.Content.ReadFromJsonAsync<PaginatedResult<ProductDto>>(
                _jsonOptions, ct);
            
            return ApiResult<PaginatedResult<ProductDto>>.Success(result!);
        }
        catch (HttpRequestException ex)
        {
            return ApiResult<PaginatedResult<ProductDto>>.NetworkError(ex.Message);
        }
        catch (OperationCanceledException)
        {
            return ApiResult<PaginatedResult<ProductDto>>.NetworkError("Request cancelled");
        }
    }
    
    public async Task<ApiResult<ProductDto>> GetProductAsync(int id, CancellationToken ct = default)
    {
        try
        {
            var response = await _http.GetAsync($"products/{id}", ct);
            
            if (!response.IsSuccessStatusCode)
                return await HandleErrorAsync<ProductDto>(response);
            
            var product = await response.Content.ReadFromJsonAsync<ProductDto>(_jsonOptions, ct);
            return ApiResult<ProductDto>.Success(product!);
        }
        catch (HttpRequestException ex)
        {
            return ApiResult<ProductDto>.NetworkError(ex.Message);
        }
    }
    
    public async Task<ApiResult<ProductDto>> CreateProductAsync(
        ProductCreateDto dto, CancellationToken ct = default)
    {
        try
        {
            var response = await _http.PostAsJsonAsync("products", dto, _jsonOptions, ct);
            
            if (!response.IsSuccessStatusCode)
                return await HandleErrorAsync<ProductDto>(response);
            
            var product = await response.Content.ReadFromJsonAsync<ProductDto>(_jsonOptions, ct);
            return ApiResult<ProductDto>.Success(product!, 201);
        }
        catch (HttpRequestException ex)
        {
            return ApiResult<ProductDto>.NetworkError(ex.Message);
        }
    }
    
    public async Task<ApiResult<ProductDto>> UpdateProductAsync(
        int id, ProductUpdateDto dto, CancellationToken ct = default)
    {
        try
        {
            var response = await _http.PutAsJsonAsync($"products/{id}", dto, _jsonOptions, ct);
            
            if (!response.IsSuccessStatusCode)
                return await HandleErrorAsync<ProductDto>(response);
            
            var product = await response.Content.ReadFromJsonAsync<ProductDto>(_jsonOptions, ct);
            return ApiResult<ProductDto>.Success(product!);
        }
        catch (HttpRequestException ex)
        {
            return ApiResult<ProductDto>.NetworkError(ex.Message);
        }
    }
    
    public async Task<ApiResult<bool>> DeleteProductAsync(int id, CancellationToken ct = default)
    {
        try
        {
            var response = await _http.DeleteAsync($"products/{id}", ct);
            
            if (!response.IsSuccessStatusCode)
                return await HandleErrorAsync<bool>(response);
            
            return ApiResult<bool>.Success(true);
        }
        catch (HttpRequestException ex)
        {
            return ApiResult<bool>.NetworkError(ex.Message);
        }
    }
    
    private static async Task<ApiResult<T>> HandleErrorAsync<T>(HttpResponseMessage response)
    {
        try
        {
            var error = await response.Content.ReadFromJsonAsync<ApiError>(_jsonOptions);
            return ApiResult<T>.Failure(
                error ?? new ApiError("UNKNOWN", "Unknown error"),
                (int)response.StatusCode);
        }
        catch
        {
            return ApiResult<T>.Failure(
                new ApiError(response.StatusCode.ToString(), response.ReasonPhrase ?? "Error"),
                (int)response.StatusCode);
        }
    }
}
```

---

## Step 204: ViewModel กับ API

```csharp
// ============================================
// Products ViewModel with API
// ============================================

public partial class ProductsApiViewModel : ObservableObject
{
    private readonly IProductApiService _api;
    private readonly INavService _nav;
    
    [ObservableProperty]
    private ObservableCollection<ProductDto> _products = new();
    
    [ObservableProperty]
    private bool _isLoading;
    
    [ObservableProperty]
    private bool _hasError;
    
    [ObservableProperty]
    private string _errorMessage = string.Empty;
    
    [ObservableProperty]
    private bool _isRefreshing;
    
    [ObservableProperty]
    private bool _hasMoreItems = true;
    
    [ObservableProperty]
    private string _searchText = string.Empty;
    
    private int _currentPage = 1;
    private const int PageSize = 20;
    private CancellationTokenSource? _cts;
    
    public ProductsApiViewModel(IProductApiService api, INavService nav)
    {
        _api = api;
        _nav = nav;
    }
    
    [RelayCommand]
    private async Task LoadProductsAsync()
    {
        _currentPage = 1;
        HasMoreItems = true;
        IsLoading = true;
        HasError = false;
        
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        
        try
        {
            var result = await _api.GetProductsAsync(1, PageSize, null, _cts.Token);
            
            if (result.IsSuccess && result.Data != null)
            {
                Products = new ObservableCollection<ProductDto>(result.Data.Items);
                HasMoreItems = result.Data.HasNextPage;
            }
            else
            {
                HasError = true;
                ErrorMessage = result.Error?.Message ?? "เกิดข้อผิดพลาด";
            }
        }
        finally
        {
            IsLoading = false;
            IsRefreshing = false;
        }
    }
    
    [RelayCommand]
    private async Task LoadMoreAsync()
    {
        if (!HasMoreItems || IsLoading) return;
        
        _currentPage++;
        IsLoading = true;
        
        var result = await _api.GetProductsAsync(_currentPage, PageSize);
        
        if (result.IsSuccess && result.Data != null)
        {
            foreach (var item in result.Data.Items)
                Products.Add(item);
            HasMoreItems = result.Data.HasNextPage;
        }
        
        IsLoading = false;
    }
    
    [RelayCommand]
    private Task RefreshAsync()
    {
        IsRefreshing = true;
        return LoadProductsAsync();
    }
    
    [RelayCommand]
    private async Task DeleteProductAsync(ProductDto product)
    {
        bool confirmed = await _nav.ShowConfirmAsync(
            "ยืนยันการลบ", $"ต้องการลบ '{product.Name}'?");
        
        if (!confirmed) return;
        
        var result = await _api.DeleteProductAsync(product.Id);
        
        if (result.IsSuccess)
        {
            var item = Products.FirstOrDefault(p => p.Id == product.Id);
            if (item != null) Products.Remove(item);
        }
        else
        {
            await _nav.ShowAlertAsync("ผิดพลาด", result.Error?.Message ?? "ลบไม่สำเร็จ");
        }
    }
}
```

---

## Step 205: Offline-First Pattern

```csharp
// ============================================
// Offline-First Repository
// ============================================

public interface IConnectivityService
{
    bool IsConnected { get; }
    event EventHandler<ConnectivityChangedEventArgs> ConnectivityChanged;
}

public class ConnectivityService : IConnectivityService
{
    public bool IsConnected => 
        Connectivity.Current.NetworkAccess == NetworkAccess.Internet;
    
    public event EventHandler<ConnectivityChangedEventArgs>? ConnectivityChanged;
    
    public ConnectivityService()
    {
        Connectivity.Current.ConnectivityChanged += OnConnectivityChanged;
    }
    
    private void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
        => ConnectivityChanged?.Invoke(sender, e);
}

// ============================================
// Offline-First Service
// ============================================

public class OfflineFirstProductService
{
    private readonly IProductApiService _api;
    private readonly DatabaseService _db;
    private readonly IConnectivityService _connectivity;
    private readonly PreferencesService _prefs;
    
    public OfflineFirstProductService(
        IProductApiService api,
        DatabaseService db,
        IConnectivityService connectivity,
        PreferencesService prefs)
    {
        _api = api;
        _db = db;
        _connectivity = connectivity;
        _prefs = prefs;
    }
    
    public async Task<List<ProductEntity>> GetProductsAsync()
    {
        // Always return local data first (fast)
        var localData = await _db.GetAllProductsAsync();
        
        // If online, sync in background
        if (_connectivity.IsConnected)
            _ = SyncProductsAsync();
        
        return localData;
    }
    
    private async Task SyncProductsAsync()
    {
        var result = await _api.GetProductsAsync();
        if (!result.IsSuccess || result.Data == null) return;
        
        foreach (var dto in result.Data.Items)
        {
            await _db.InsertOrReplaceAsync(new ProductEntity
            {
                Id = dto.Id,
                Name = dto.Name,
                Category = dto.Category,
                Price = dto.Price,
                Stock = dto.Stock,
                IsActive = true,
                CreatedAt = dto.CreatedAt
            });
        }
        
        _prefs.LastSync = DateTime.UtcNow;
    }
    
    public async Task<bool> CreateProductAsync(ProductCreateDto dto)
    {
        if (_connectivity.IsConnected)
        {
            var result = await _api.CreateProductAsync(dto);
            if (result.IsSuccess && result.Data != null)
            {
                await _db.InsertProductAsync(new ProductEntity
                {
                    Id = result.Data.Id,
                    Name = result.Data.Name,
                    Category = result.Data.Category,
                    Price = result.Data.Price,
                    Stock = result.Data.Stock
                });
                return true;
            }
            return false;
        }
        else
        {
            // Save locally with temp ID, sync later
            await _db.InsertProductAsync(new ProductEntity
            {
                Id = -(int)DateTimeOffset.UtcNow.ToUnixTimeSeconds(), // temp negative ID
                Name = dto.Name,
                Category = dto.Category,
                Price = dto.Price,
                Stock = dto.Stock
            });
            // Queue for sync
            return true;
        }
    }
}
```

---

## Step 206: Image Loading

```csharp
// ============================================
// Image from URL ใน MAUI
// ============================================

// XAML:
/*
<Image Source="https://example.com/image.jpg"
       Aspect="AspectFill"
       HeightRequest="200"
       WidthRequest="200"
       IsAnimationPlaying="True">
    <Image.Behaviors>
        <toolkit:IconTintColorBehavior TintColor="Gray" />
    </Image.Behaviors>
</Image>

<!-- With CachingStrategy -->
<Image Source="{Binding ImageUrl}"
       Aspect="AspectFill" />
*/

// ============================================
// Custom Image Cache Service
// ============================================

public class ImageCacheService
{
    private readonly HttpClient _http;
    private readonly string _cacheDir;
    
    public ImageCacheService(HttpClient http)
    {
        _http = http;
        _cacheDir = Path.Combine(FileSystem.CacheDirectory, "images");
        Directory.CreateDirectory(_cacheDir);
    }
    
    public async Task<ImageSource> GetImageAsync(string url)
    {
        var filename = Convert.ToBase64String(
            System.Text.Encoding.UTF8.GetBytes(url))
            .Replace('/', '_').Replace('+', '-')[..40] + ".jpg";
        
        var cachePath = Path.Combine(_cacheDir, filename);
        
        if (File.Exists(cachePath))
        {
            var age = DateTime.UtcNow - File.GetLastWriteTimeUtc(cachePath);
            if (age < TimeSpan.FromDays(7))
                return ImageSource.FromFile(cachePath);
        }
        
        try
        {
            var bytes = await _http.GetByteArrayAsync(url);
            await File.WriteAllBytesAsync(cachePath, bytes);
            return ImageSource.FromFile(cachePath);
        }
        catch
        {
            return "placeholder.png";
        }
    }
    
    public void ClearCache()
    {
        foreach (var file in Directory.GetFiles(_cacheDir))
            File.Delete(file);
    }
    
    public long GetCacheSizeBytes()
        => Directory.GetFiles(_cacheDir).Sum(f => new FileInfo(f).Length);
}
```

---

## Step 207: Upload และ Download

```csharp
// ============================================
// File Upload
// ============================================

public class UploadService
{
    private readonly HttpClient _http;
    
    public UploadService(HttpClient http) => _http = http;
    
    public async Task<ApiResult<string>> UploadImageAsync(
        Stream imageStream, string filename, IProgress<double>? progress = null)
    {
        try
        {
            using var content = new MultipartFormDataContent();
            using var streamContent = new ProgressStreamContent(imageStream, progress);
            
            streamContent.Headers.ContentType = 
                new System.Net.Http.Headers.MediaTypeHeaderValue("image/jpeg");
            
            content.Add(streamContent, "file", filename);
            
            var response = await _http.PostAsync("upload", content);
            
            if (!response.IsSuccessStatusCode)
                return ApiResult<string>.Failure(
                    new ApiError("UPLOAD_FAILED", "อัพโหลดไม่สำเร็จ"),
                    (int)response.StatusCode);
            
            var result = await response.Content.ReadFromJsonAsync<UploadResult>();
            return ApiResult<string>.Success(result!.Url);
        }
        catch (Exception ex)
        {
            return ApiResult<string>.NetworkError(ex.Message);
        }
    }
    
    // Pick photo and upload
    public async Task<ApiResult<string>> PickAndUploadPhotoAsync()
    {
        var photo = await MediaPicker.Default.PickPhotoAsync(new MediaPickerOptions
        {
            Title = "เลือกรูปภาพ"
        });
        
        if (photo == null)
            return ApiResult<string>.Failure(new ApiError("CANCELLED", "ยกเลิก"), 0);
        
        using var stream = await photo.OpenReadAsync();
        return await UploadImageAsync(stream, photo.FileName);
    }
    
    // Take photo and upload
    public async Task<ApiResult<string>> TakePhotoAndUploadAsync()
    {
        if (!MediaPicker.Default.IsCaptureSupported)
            return ApiResult<string>.Failure(new ApiError("NOT_SUPPORTED", "ไม่รองรับ"), 0);
        
        var photo = await MediaPicker.Default.CapturePhotoAsync();
        if (photo == null)
            return ApiResult<string>.Failure(new ApiError("CANCELLED", "ยกเลิก"), 0);
        
        using var stream = await photo.OpenReadAsync();
        return await UploadImageAsync(stream, photo.FileName);
    }
}

public record UploadResult(string Url, string Filename, long Size);

// Progress-enabled stream content
public class ProgressStreamContent : StreamContent
{
    private readonly Stream _stream;
    private readonly IProgress<double>? _progress;
    
    public ProgressStreamContent(Stream stream, IProgress<double>? progress) : base(stream)
    {
        _stream = stream;
        _progress = progress;
    }
    
    protected override async Task SerializeToStreamAsync(Stream stream, 
        System.Net.TransportContext? context)
    {
        var buffer = new byte[8192];
        long totalRead = 0;
        long totalLength = _stream.Length;
        int read;
        
        while ((read = await _stream.ReadAsync(buffer)) > 0)
        {
            await stream.WriteAsync(buffer.AsMemory(0, read));
            totalRead += read;
            _progress?.Report((double)totalRead / totalLength);
        }
    }
}

// ============================================
// File Download
// ============================================

public class DownloadService
{
    private readonly HttpClient _http;
    
    public DownloadService(HttpClient http) => _http = http;
    
    public async Task<string> DownloadFileAsync(
        string url, string filename, IProgress<double>? progress = null,
        CancellationToken ct = default)
    {
        var savePath = Path.Combine(FileSystem.CacheDirectory, filename);
        
        using var response = await _http.GetAsync(url, 
            HttpCompletionOption.ResponseHeadersRead, ct);
        
        response.EnsureSuccessStatusCode();
        
        var totalBytes = response.Content.Headers.ContentLength ?? -1;
        
        using var stream = await response.Content.ReadAsStreamAsync(ct);
        using var fileStream = File.Create(savePath);
        
        var buffer = new byte[8192];
        long downloadedBytes = 0;
        int bytesRead;
        
        while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
        {
            await fileStream.WriteAsync(buffer.AsMemory(0, bytesRead), ct);
            downloadedBytes += bytesRead;
            
            if (totalBytes > 0)
                progress?.Report((double)downloadedBytes / totalBytes);
        }
        
        return savePath;
    }
}
```

---

## Step 208: Error Handling

```csharp
// ============================================
// Global Error Handler
// ============================================

public class GlobalExceptionHandler
{
    public static void Setup()
    {
        // UI thread exceptions
        MauiExceptions.UnhandledException += (sender, e) =>
        {
            LogException(e.ExceptionObject as Exception);
        };
        
        // Background thread exceptions
        TaskScheduler.UnobservedTaskException += (sender, e) =>
        {
            LogException(e.Exception);
            e.SetObserved();
        };
    }
    
    private static void LogException(Exception? ex)
    {
        if (ex == null) return;
        
        Console.WriteLine($"[ERROR] {ex.GetType().Name}: {ex.Message}");
        Console.WriteLine(ex.StackTrace);
        
        // Report to crash service (Sentry, AppCenter, etc.)
        // Sentry.SentrySdk.CaptureException(ex);
    }
}

// ============================================
// Retry Policy
// ============================================

public class RetryPolicy
{
    public static async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 3,
        TimeSpan? initialDelay = null,
        Func<Exception, bool>? shouldRetry = null)
    {
        initialDelay ??= TimeSpan.FromSeconds(1);
        shouldRetry ??= ex => ex is HttpRequestException || ex is TimeoutException;
        
        Exception? lastException = null;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (shouldRetry(ex) && attempt < maxRetries)
            {
                lastException = ex;
                var delay = TimeSpan.FromMilliseconds(
                    initialDelay.Value.TotalMilliseconds * Math.Pow(2, attempt));
                await Task.Delay(delay);
            }
        }
        
        throw lastException!;
    }
}

// Usage
// var result = await RetryPolicy.ExecuteAsync(() => _api.GetProductsAsync());
```

---

## Step 209: Mock API สำหรับ Testing

```csharp
// ============================================
// Mock API Service
// ============================================

public class MockProductApiService : IProductApiService
{
    private readonly List<ProductDto> _products = new()
    {
        new(1, "iPhone 15", "Electronics", 35000, 100, null, DateTime.UtcNow),
        new(2, "Samsung S24", "Electronics", 30000, 150, null, DateTime.UtcNow),
        new(3, "Nike Air Max", "Shoes", 4500, 200, null, DateTime.UtcNow),
    };
    
    private int _nextId = 4;
    
    public async Task<ApiResult<PaginatedResult<ProductDto>>> GetProductsAsync(
        int page = 1, int size = 20, string? category = null, CancellationToken ct = default)
    {
        await Task.Delay(300, ct); // Simulate network
        
        var filtered = string.IsNullOrEmpty(category)
            ? _products
            : _products.Where(p => p.Category == category).ToList();
        
        var paged = filtered
            .Skip((page - 1) * size)
            .Take(size)
            .ToList();
        
        return ApiResult<PaginatedResult<ProductDto>>.Success(
            new PaginatedResult<ProductDto>(paged, filtered.Count, page, size));
    }
    
    public async Task<ApiResult<ProductDto>> GetProductAsync(int id, CancellationToken ct = default)
    {
        await Task.Delay(100, ct);
        
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null)
            return ApiResult<ProductDto>.Failure(
                new ApiError("NOT_FOUND", "ไม่พบสินค้า"), 404);
        
        return ApiResult<ProductDto>.Success(product);
    }
    
    public async Task<ApiResult<ProductDto>> CreateProductAsync(
        ProductCreateDto dto, CancellationToken ct = default)
    {
        await Task.Delay(200, ct);
        
        var product = new ProductDto(_nextId++, dto.Name, dto.Category, 
            dto.Price, dto.Stock, null, DateTime.UtcNow);
        _products.Add(product);
        
        return ApiResult<ProductDto>.Success(product, 201);
    }
    
    public async Task<ApiResult<ProductDto>> UpdateProductAsync(
        int id, ProductUpdateDto dto, CancellationToken ct = default)
    {
        await Task.Delay(200, ct);
        
        int idx = _products.FindIndex(p => p.Id == id);
        if (idx < 0)
            return ApiResult<ProductDto>.Failure(new ApiError("NOT_FOUND", "ไม่พบสินค้า"), 404);
        
        var updated = new ProductDto(id, dto.Name, dto.Category, 
            dto.Price, dto.Stock, _products[idx].ImageUrl, _products[idx].CreatedAt);
        _products[idx] = updated;
        
        return ApiResult<ProductDto>.Success(updated);
    }
    
    public async Task<ApiResult<bool>> DeleteProductAsync(int id, CancellationToken ct = default)
    {
        await Task.Delay(100, ct);
        
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null)
            return ApiResult<bool>.Failure(new ApiError("NOT_FOUND", "ไม่พบสินค้า"), 404);
        
        _products.Remove(product);
        return ApiResult<bool>.Success(true);
    }
}
```

---

## Step 210: Authentication Flow

```csharp
// ============================================
// Auth Service
// ============================================

public record LoginRequest(string Email, string Password);
public record LoginResponse(string AccessToken, string RefreshToken, UserProfile User);
public record UserProfile(int Id, string Name, string Email, string Role);

public interface IAuthApiService
{
    Task<ApiResult<LoginResponse>> LoginAsync(LoginRequest request, CancellationToken ct = default);
    Task<ApiResult<LoginResponse>> RefreshTokenAsync(string refreshToken, CancellationToken ct = default);
    Task<ApiResult<bool>> LogoutAsync(CancellationToken ct = default);
    Task<ApiResult<bool>> RegisterAsync(RegisterRequest request, CancellationToken ct = default);
}

public record RegisterRequest(string Name, string Email, string Password);

// ============================================
// Auth ViewModel
// ============================================

public partial class LoginViewModel : ObservableObject
{
    private readonly IAuthApiService _auth;
    private readonly SecureStorageService _secure;
    private readonly INavService _nav;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกอีเมล")]
    [EmailAddress(ErrorMessage = "อีเมลไม่ถูกต้อง")]
    private string _email = string.Empty;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกรหัสผ่าน")]
    [MinLength(6, ErrorMessage = "รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร")]
    private string _password = string.Empty;
    
    [ObservableProperty]
    private bool _isLoading;
    
    [ObservableProperty]
    private string _errorMessage = string.Empty;
    
    [ObservableProperty]
    private bool _showPassword;
    
    public LoginViewModel(IAuthApiService auth, SecureStorageService secure, INavService nav)
    {
        _auth = auth;
        _secure = secure;
        _nav = nav;
    }
    
    [RelayCommand]
    private async Task LoginAsync()
    {
        ValidateAllProperties();
        if (HasErrors) return;
        
        IsLoading = true;
        ErrorMessage = string.Empty;
        
        var result = await _auth.LoginAsync(new LoginRequest(Email, Password));
        
        if (result.IsSuccess && result.Data != null)
        {
            await _secure.SaveTokenAsync(result.Data.AccessToken);
            await _secure.SaveRefreshTokenAsync(result.Data.RefreshToken);
            await _nav.GoToRootAsync("home");
        }
        else
        {
            ErrorMessage = result.Error?.Message ?? "เข้าสู่ระบบไม่สำเร็จ";
        }
        
        IsLoading = false;
    }
    
    [RelayCommand]
    private void TogglePasswordVisibility()
        => ShowPassword = !ShowPassword;
    
    [RelayCommand]
    private Task GoToRegisterAsync()
        => _nav.GoToAsync("register");
    
    [RelayCommand]
    private Task ForgotPasswordAsync()
        => _nav.GoToAsync("forgot-password");
}
```

---

## สรุป Part 21

ใน Part 21 เราได้เรียนรู้:

1. **HttpClient Setup** - Named/Typed, DelegatingHandler
2. **Auth Header Handler** - Auto inject Bearer token
3. **API Service Interface** - DTOs, PaginatedResult, ApiResult<T>
4. **CRUD Operations** - GET/POST/PUT/DELETE
5. **ViewModel กับ API** - Load, Refresh, Load More, Delete
6. **Offline-First** - Local cache + background sync
7. **Image Loading** - URL to ImageSource, custom cache
8. **File Upload/Download** - MultipartFormData, Progress
9. **Error Handling** - Global handler, retry policy
10. **Authentication Flow** - Login, token storage, auto-redirect

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 21 | Steps 201-210*
