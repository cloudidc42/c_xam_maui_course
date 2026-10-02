# Part 16: Networking และ HTTP Client
## Steps 151-160: การสื่อสารเครือข่าย

---

## Step 151: HttpClient พื้นฐาน

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json;

// ============================================
// HttpClient Basics
// ============================================

// ❌ Bad: สร้างใหม่ทุกครั้ง (socket exhaustion)
// for (int i = 0; i < 100; i++)
// {
//     using var client = new HttpClient();
//     var response = await client.GetAsync("https://api.example.com");
// }

// ✅ Good: Singleton หรือ IHttpClientFactory
public class ApiClient
{
    private readonly HttpClient _client;
    
    public ApiClient(HttpClient client)
    {
        _client = client;
        _client.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
        _client.DefaultRequestHeaders.Add("Accept", "application/json");
        _client.Timeout = TimeSpan.FromSeconds(30);
    }
    
    // GET request
    public async Task<string> GetRawAsync(string path, CancellationToken ct = default)
    {
        var response = await _client.GetAsync(path, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
    
    // GET with deserialization
    public async Task<T?> GetAsync<T>(string path, CancellationToken ct = default)
    {
        var response = await _client.GetAsync(path, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
    
    // POST
    public async Task<TResponse?> PostAsync<TRequest, TResponse>(
        string path, TRequest body, CancellationToken ct = default)
    {
        var response = await _client.PostAsJsonAsync(path, body, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<TResponse>(cancellationToken: ct);
    }
    
    // PUT
    public async Task<TResponse?> PutAsync<TRequest, TResponse>(
        string path, TRequest body, CancellationToken ct = default)
    {
        var response = await _client.PutAsJsonAsync(path, body, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<TResponse>(cancellationToken: ct);
    }
    
    // DELETE
    public async Task DeleteAsync(string path, CancellationToken ct = default)
    {
        var response = await _client.DeleteAsync(path, ct);
        response.EnsureSuccessStatusCode();
    }
}

// Models
public record Post(int? Id, int UserId, string Title, string Body);
public record Comment(int PostId, int Id, string Name, string Email, string Body);

// ใช้งาน
var apiClient = new ApiClient(new HttpClient());

// GET posts
var posts = await apiClient.GetAsync<List<Post>>("posts");
Console.WriteLine($"Posts: {posts?.Count}");

// GET single post
var post = await apiClient.GetAsync<Post>("posts/1");
Console.WriteLine($"Post: {post?.Title}");

// POST new post
var newPost = await apiClient.PostAsync<Post, Post>("posts", 
    new Post(null, 1, "My New Post", "Content here..."));
Console.WriteLine($"Created: ID={newPost?.Id}");
```

---

## Step 152: HTTP Error Handling

```csharp
// ============================================
// Robust HTTP Client
// ============================================

public class RobustHttpClient
{
    private readonly HttpClient _client;
    
    public RobustHttpClient(HttpClient client) => _client = client;
    
    public async Task<Result<T>> GetAsync<T>(string url, CancellationToken ct = default)
    {
        try
        {
            var response = await _client.GetAsync(url, ct);
            
            return response.StatusCode switch
            {
                System.Net.HttpStatusCode.OK => 
                    Result<T>.Success(
                        (await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct))!),
                
                System.Net.HttpStatusCode.NotFound =>
                    Result<T>.Failure("Resource not found", "NOT_FOUND"),
                
                System.Net.HttpStatusCode.Unauthorized =>
                    Result<T>.Failure("Authentication required", "UNAUTHORIZED"),
                
                System.Net.HttpStatusCode.Forbidden =>
                    Result<T>.Failure("Access denied", "FORBIDDEN"),
                
                System.Net.HttpStatusCode.TooManyRequests =>
                    Result<T>.Failure("Rate limit exceeded", "RATE_LIMITED"),
                
                var status when (int)status >= 500 =>
                    Result<T>.Failure($"Server error: {(int)status}", "SERVER_ERROR"),
                
                var status =>
                    Result<T>.Failure($"HTTP error: {(int)status}", "HTTP_ERROR")
            };
        }
        catch (HttpRequestException ex)
        {
            return Result<T>.Failure($"Network error: {ex.Message}", "NETWORK_ERROR");
        }
        catch (TaskCanceledException ex) when (ex.InnerException is TimeoutException)
        {
            return Result<T>.Failure("Request timed out", "TIMEOUT");
        }
        catch (TaskCanceledException)
        {
            return Result<T>.Failure("Request cancelled", "CANCELLED");
        }
        catch (JsonException ex)
        {
            return Result<T>.Failure($"Invalid response format: {ex.Message}", "PARSE_ERROR");
        }
    }
}

// ============================================
// Retry with Polly-style
// ============================================

public class RetryHttpClient
{
    private readonly HttpClient _client;
    private readonly int _maxRetries;
    private readonly TimeSpan _initialDelay;
    
    public RetryHttpClient(HttpClient client, int maxRetries = 3, 
        TimeSpan? initialDelay = null)
    {
        _client = client;
        _maxRetries = maxRetries;
        _initialDelay = initialDelay ?? TimeSpan.FromSeconds(1);
    }
    
    public async Task<HttpResponseMessage> SendWithRetryAsync(
        HttpRequestMessage request, CancellationToken ct = default)
    {
        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            try
            {
                var response = await _client.SendAsync(request, ct);
                
                // Retry on 429 (rate limit) and 5xx
                bool shouldRetry = attempt < _maxRetries && (
                    response.StatusCode == System.Net.HttpStatusCode.TooManyRequests ||
                    (int)response.StatusCode >= 500
                );
                
                if (!shouldRetry) return response;
                
                // Check Retry-After header
                var delay = response.Headers.RetryAfter?.Delta ?? _initialDelay * attempt;
                Console.WriteLine($"Retry {attempt}/{_maxRetries} after {delay}");
                await Task.Delay(delay, ct);
                
                // Must recreate the request for retry
                request = CloneRequest(request);
            }
            catch (HttpRequestException) when (attempt < _maxRetries)
            {
                Console.WriteLine($"Network error, retry {attempt}/{_maxRetries}");
                await Task.Delay(_initialDelay * attempt, ct);
            }
        }
        
        throw new Exception($"All {_maxRetries} retries failed");
    }
    
    private HttpRequestMessage CloneRequest(HttpRequestMessage original)
    {
        var clone = new HttpRequestMessage(original.Method, original.RequestUri);
        foreach (var header in original.Headers)
            clone.Headers.TryAddWithoutValidation(header.Key, header.Value);
        return clone;
    }
}
```

---

## Step 153: Authentication Headers

```csharp
// ============================================
// Authentication
// ============================================

public class AuthenticatedHttpClient
{
    private readonly HttpClient _client;
    private string? _bearerToken;
    private string? _apiKey;
    
    public AuthenticatedHttpClient(HttpClient client) => _client = client;
    
    // Bearer token
    public void SetBearerToken(string token)
    {
        _bearerToken = token;
        _client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }
    
    // API Key
    public void SetApiKey(string key, string headerName = "X-API-Key")
    {
        _apiKey = key;
        _client.DefaultRequestHeaders.Remove(headerName);
        _client.DefaultRequestHeaders.Add(headerName, key);
    }
    
    // Basic Auth
    public void SetBasicAuth(string username, string password)
    {
        string credentials = Convert.ToBase64String(
            System.Text.Encoding.ASCII.GetBytes($"{username}:{password}"));
        _client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Basic", credentials);
    }
    
    // Per-request auth
    public async Task<T?> GetWithTokenAsync<T>(string url, string token, CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, url);
        request.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        
        var response = await _client.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
}

// ============================================
// OAuth Token Refresh
// ============================================

public class OAuthHttpClient
{
    private readonly HttpClient _client;
    private string? _accessToken;
    private string? _refreshToken;
    private DateTime _tokenExpiry;
    
    public OAuthHttpClient(HttpClient client) => _client = client;
    
    private bool IsTokenExpired => DateTime.UtcNow >= _tokenExpiry.AddMinutes(-5);
    
    private async Task EnsureValidTokenAsync()
    {
        if (_accessToken == null || IsTokenExpired)
            await RefreshTokenAsync();
    }
    
    private async Task RefreshTokenAsync()
    {
        var request = new Dictionary<string, string>
        {
            { "grant_type", "refresh_token" },
            { "refresh_token", _refreshToken! },
            { "client_id", "my-client-id" }
        };
        
        var response = await _client.PostAsync("/oauth/token",
            new FormUrlEncodedContent(request));
        
        response.EnsureSuccessStatusCode();
        
        var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
        _accessToken = tokenResponse!.AccessToken;
        _refreshToken = tokenResponse.RefreshToken;
        _tokenExpiry = DateTime.UtcNow.AddSeconds(tokenResponse.ExpiresIn);
    }
    
    public async Task<T?> GetAsync<T>(string url, CancellationToken ct = default)
    {
        await EnsureValidTokenAsync();
        
        var request = new HttpRequestMessage(HttpMethod.Get, url);
        request.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", _accessToken);
        
        var response = await _client.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }
    
    public void SetTokens(string access, string refresh, int expiresIn)
    {
        _accessToken = access;
        _refreshToken = refresh;
        _tokenExpiry = DateTime.UtcNow.AddSeconds(expiresIn);
    }
}

public record TokenResponse(
    [property: JsonPropertyName("access_token")] string AccessToken,
    [property: JsonPropertyName("refresh_token")] string RefreshToken,
    [property: JsonPropertyName("expires_in")] int ExpiresIn
);
```

---

## Step 154: HTTP Streaming

```csharp
// ============================================
// Server-Sent Events (SSE)
// ============================================

public class SseClient
{
    private readonly HttpClient _client;
    
    public SseClient(HttpClient client) => _client = client;
    
    public async IAsyncEnumerable<string> StreamEventsAsync(
        string url,
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, url);
        request.Headers.Accept.Add(
            new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("text/event-stream"));
        
        var response = await _client.SendAsync(
            request, HttpCompletionOption.ResponseHeadersRead, ct);
        
        response.EnsureSuccessStatusCode();
        
        using var stream = await response.Content.ReadAsStreamAsync(ct);
        using var reader = new StreamReader(stream);
        
        while (!reader.EndOfStream)
        {
            ct.ThrowIfCancellationRequested();
            
            string? line = await reader.ReadLineAsync(ct);
            if (string.IsNullOrEmpty(line)) continue;
            
            if (line.StartsWith("data: "))
                yield return line["data: ".Length..];
        }
    }
}

// ============================================
// Large File Download with Progress
// ============================================

public class FileDownloader
{
    private readonly HttpClient _client;
    
    public FileDownloader(HttpClient client) => _client = client;
    
    public async Task DownloadAsync(
        string url, 
        string filePath,
        IProgress<(long BytesDownloaded, long? TotalBytes, double? Percentage)>? progress = null,
        CancellationToken ct = default)
    {
        var response = await _client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
        response.EnsureSuccessStatusCode();
        
        long? totalBytes = response.Content.Headers.ContentLength;
        
        await using var stream = await response.Content.ReadAsStreamAsync(ct);
        await using var fileStream = new FileStream(filePath, FileMode.Create);
        
        byte[] buffer = new byte[8192];
        long bytesDownloaded = 0;
        int bytesRead;
        
        while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
        {
            await fileStream.WriteAsync(buffer.AsMemory(0, bytesRead), ct);
            bytesDownloaded += bytesRead;
            
            double? percentage = totalBytes.HasValue 
                ? 100.0 * bytesDownloaded / totalBytes.Value 
                : null;
            
            progress?.Report((bytesDownloaded, totalBytes, percentage));
        }
    }
}
```

---

## Step 155: REST API Client

```csharp
// ============================================
// Typed REST API Client
// ============================================

public class RestApiClient<T, TId>
    where T : class
    where TId : notnull
{
    private readonly HttpClient _client;
    private readonly string _resourcePath;
    private readonly JsonSerializerOptions _jsonOptions;
    
    public RestApiClient(HttpClient client, string resourcePath)
    {
        _client = client;
        _resourcePath = resourcePath.TrimEnd('/');
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }
    
    public async Task<List<T>> GetAllAsync(
        Dictionary<string, string>? queryParams = null,
        CancellationToken ct = default)
    {
        string url = BuildUrl(_resourcePath, queryParams);
        var response = await _client.GetAsync(url, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<T>>(_jsonOptions, ct) ?? new();
    }
    
    public async Task<T?> GetByIdAsync(TId id, CancellationToken ct = default)
    {
        var response = await _client.GetAsync($"{_resourcePath}/{id}", ct);
        if (response.StatusCode == System.Net.HttpStatusCode.NotFound) return null;
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(_jsonOptions, ct);
    }
    
    public async Task<T?> CreateAsync(T entity, CancellationToken ct = default)
    {
        var content = JsonContent.Create(entity, options: _jsonOptions);
        var response = await _client.PostAsync(_resourcePath, content, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(_jsonOptions, ct);
    }
    
    public async Task<T?> UpdateAsync(TId id, T entity, CancellationToken ct = default)
    {
        var content = JsonContent.Create(entity, options: _jsonOptions);
        var response = await _client.PutAsync($"{_resourcePath}/{id}", content, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(_jsonOptions, ct);
    }
    
    public async Task<bool> DeleteAsync(TId id, CancellationToken ct = default)
    {
        var response = await _client.DeleteAsync($"{_resourcePath}/{id}", ct);
        return response.IsSuccessStatusCode;
    }
    
    private string BuildUrl(string path, Dictionary<string, string>? queryParams)
    {
        if (queryParams == null || !queryParams.Any()) return path;
        string query = string.Join("&", 
            queryParams.Select(kv => $"{Uri.EscapeDataString(kv.Key)}={Uri.EscapeDataString(kv.Value)}"));
        return $"{path}?{query}";
    }
}

// ใช้งาน
var client = new HttpClient { BaseAddress = new Uri("https://jsonplaceholder.typicode.com/") };

var postClient = new RestApiClient<Post, int>(client, "posts");

var allPosts = await postClient.GetAllAsync(new() { { "_limit", "5" } });
Console.WriteLine($"Posts: {allPosts.Count}");

var post2 = await postClient.GetByIdAsync(1);
Console.WriteLine($"Post 1: {post2?.Title}");

var created = await postClient.CreateAsync(new Post(null, 1, "New Post", "Content"));
Console.WriteLine($"Created: {created?.Id}");
```

---

## Step 156: WebSocket

```csharp
using System.Net.WebSockets;
using System.Text;

// ============================================
// WebSocket Client
// ============================================

public class WebSocketClient
{
    private readonly ClientWebSocket _ws = new();
    private CancellationTokenSource? _cts;
    
    public event Action<string>? OnMessage;
    public event Action<Exception>? OnError;
    public event Action? OnClosed;
    
    public async Task ConnectAsync(string url, CancellationToken ct = default)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        await _ws.ConnectAsync(new Uri(url), _cts.Token);
        Console.WriteLine($"Connected to {url}");
        _ = ReceiveLoopAsync(_cts.Token);
    }
    
    public async Task SendAsync(string message, CancellationToken ct = default)
    {
        byte[] bytes = Encoding.UTF8.GetBytes(message);
        await _ws.SendAsync(
            new ArraySegment<byte>(bytes),
            WebSocketMessageType.Text,
            endOfMessage: true,
            ct
        );
    }
    
    public async Task SendJsonAsync<T>(T message, CancellationToken ct = default)
    {
        string json = JsonSerializer.Serialize(message);
        await SendAsync(json, ct);
    }
    
    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        var buffer = new byte[4096];
        var messageBuffer = new StringBuilder();
        
        while (_ws.State == WebSocketState.Open && !ct.IsCancellationRequested)
        {
            WebSocketReceiveResult result;
            try
            {
                result = await _ws.ReceiveAsync(new ArraySegment<byte>(buffer), ct);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                OnError?.Invoke(ex);
                break;
            }
            
            if (result.MessageType == WebSocketMessageType.Close)
            {
                await _ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "", ct);
                OnClosed?.Invoke();
                break;
            }
            
            string chunk = Encoding.UTF8.GetString(buffer, 0, result.Count);
            messageBuffer.Append(chunk);
            
            if (result.EndOfMessage)
            {
                OnMessage?.Invoke(messageBuffer.ToString());
                messageBuffer.Clear();
            }
        }
    }
    
    public async Task DisconnectAsync()
    {
        _cts?.Cancel();
        if (_ws.State == WebSocketState.Open)
            await _ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", default);
    }
}

// ============================================
// WebSocket สำหรับ Real-time Chat
// ============================================

public record ChatMessage(string UserId, string Username, string Content, DateTime Timestamp);

public class ChatClient
{
    private readonly WebSocketClient _ws;
    private readonly string _userId;
    private readonly string _username;
    
    public event Action<ChatMessage>? OnMessageReceived;
    
    public ChatClient(string userId, string username)
    {
        _userId = userId;
        _username = username;
        _ws = new WebSocketClient();
        _ws.OnMessage += HandleMessage;
    }
    
    private void HandleMessage(string json)
    {
        var msg = JsonSerializer.Deserialize<ChatMessage>(json);
        if (msg != null) OnMessageReceived?.Invoke(msg);
    }
    
    public async Task ConnectAsync(string serverUrl)
        => await _ws.ConnectAsync(serverUrl);
    
    public async Task SendMessageAsync(string content)
    {
        var msg = new ChatMessage(_userId, _username, content, DateTime.UtcNow);
        await _ws.SendJsonAsync(msg);
    }
}
```

---

## Step 157: HTTP Caching

```csharp
// ============================================
// HTTP Caching Headers
// ============================================

public class CachingHttpClient
{
    private readonly HttpClient _client;
    private readonly Dictionary<string, (string Content, string? ETag, DateTime Expires)> _cache = new();
    
    public CachingHttpClient(HttpClient client) => _client = client;
    
    public async Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        // Check cache
        if (_cache.TryGetValue(url, out var cached))
        {
            if (cached.Expires > DateTime.UtcNow)
            {
                Console.WriteLine($"Cache HIT (fresh): {url}");
                return cached.Content;
            }
            
            // Stale - validate with ETag
            if (cached.ETag != null)
            {
                var request = new HttpRequestMessage(HttpMethod.Get, url);
                request.Headers.IfNoneMatch.Add(
                    new System.Net.Http.Headers.EntityTagHeaderValue(cached.ETag));
                
                var validateResponse = await _client.SendAsync(request, ct);
                
                if (validateResponse.StatusCode == System.Net.HttpStatusCode.NotModified)
                {
                    Console.WriteLine($"Cache HIT (revalidated): {url}");
                    return cached.Content;
                }
                
                // Resource changed
                return await UpdateCacheAsync(url, validateResponse, ct);
            }
        }
        
        Console.WriteLine($"Cache MISS: {url}");
        var response = await _client.GetAsync(url, ct);
        return await UpdateCacheAsync(url, response, ct);
    }
    
    private async Task<string> UpdateCacheAsync(string url, HttpResponseMessage response, CancellationToken ct)
    {
        response.EnsureSuccessStatusCode();
        
        string content = await response.Content.ReadAsStringAsync(ct);
        
        string? etag = response.Headers.ETag?.Tag;
        
        // Parse Cache-Control
        DateTime expires = DateTime.UtcNow.AddSeconds(60); // default 60s
        if (response.Headers.CacheControl?.MaxAge.HasValue == true)
            expires = DateTime.UtcNow.Add(response.Headers.CacheControl.MaxAge!.Value);
        
        _cache[url] = (content, etag, expires);
        return content;
    }
}
```

---

## Step 158: Rate Limiting Client

```csharp
// ============================================
// Rate-Limited HTTP Client
// ============================================

public class RateLimitedClient
{
    private readonly HttpClient _client;
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DateTime> _requestTimes = new();
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    
    public RateLimitedClient(HttpClient client, int maxRequests = 10, 
        int windowSeconds = 1)
    {
        _client = client;
        _maxRequests = maxRequests;
        _window = TimeSpan.FromSeconds(windowSeconds);
        _semaphore = new SemaphoreSlim(1, 1);
    }
    
    public async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            await EnforceRateLimitAsync(ct);
            var response = await _client.SendAsync(request, ct);
            return response;
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    private async Task EnforceRateLimitAsync(CancellationToken ct)
    {
        var now = DateTime.UtcNow;
        
        // Remove old timestamps
        while (_requestTimes.Count > 0 && now - _requestTimes.Peek() > _window)
            _requestTimes.Dequeue();
        
        // Wait if rate limit reached
        if (_requestTimes.Count >= _maxRequests)
        {
            var oldest = _requestTimes.Peek();
            var waitTime = _window - (now - oldest);
            Console.WriteLine($"Rate limit: waiting {waitTime.TotalMilliseconds:F0}ms");
            await Task.Delay(waitTime, ct);
        }
        
        _requestTimes.Enqueue(DateTime.UtcNow);
    }
}
```

---

## Step 159: HTTP Multipart

```csharp
// ============================================
// File Upload (Multipart)
// ============================================

public class FileUploadClient
{
    private readonly HttpClient _client;
    
    public FileUploadClient(HttpClient client) => _client = client;
    
    // Upload single file
    public async Task<string> UploadFileAsync(
        string url, 
        string filePath, 
        string fieldName = "file",
        Dictionary<string, string>? additionalFields = null,
        CancellationToken ct = default)
    {
        using var content = new MultipartFormDataContent();
        
        // Add file
        var fileBytes = await File.ReadAllBytesAsync(filePath, ct);
        var fileContent = new ByteArrayContent(fileBytes);
        fileContent.Headers.ContentType = 
            new System.Net.Http.Headers.MediaTypeHeaderValue(
                GetMimeType(Path.GetExtension(filePath)));
        content.Add(fileContent, fieldName, Path.GetFileName(filePath));
        
        // Add other fields
        if (additionalFields != null)
            foreach (var (key, value) in additionalFields)
                content.Add(new StringContent(value), key);
        
        var response = await _client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
    
    // Upload multiple files
    public async Task<string> UploadFilesAsync(
        string url, 
        IEnumerable<string> filePaths,
        CancellationToken ct = default)
    {
        using var content = new MultipartFormDataContent();
        
        foreach (var filePath in filePaths)
        {
            var fileBytes = await File.ReadAllBytesAsync(filePath, ct);
            var fileContent = new ByteArrayContent(fileBytes);
            content.Add(fileContent, "files[]", Path.GetFileName(filePath));
        }
        
        var response = await _client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
    
    // Upload stream (for large files or camera captures in MAUI)
    public async Task<string> UploadStreamAsync(
        string url,
        Stream stream,
        string fileName,
        string contentType = "application/octet-stream",
        CancellationToken ct = default)
    {
        using var content = new MultipartFormDataContent();
        var streamContent = new StreamContent(stream);
        streamContent.Headers.ContentType =
            new System.Net.Http.Headers.MediaTypeHeaderValue(contentType);
        content.Add(streamContent, "file", fileName);
        
        var response = await _client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
    
    private static string GetMimeType(string ext) => ext.ToLower() switch
    {
        ".jpg" or ".jpeg" => "image/jpeg",
        ".png" => "image/png",
        ".pdf" => "application/pdf",
        ".zip" => "application/zip",
        ".txt" => "text/plain",
        _ => "application/octet-stream"
    };
}
```

---

## Step 160: Complete API Service

```csharp
// ============================================
// Production-ready API Service
// ============================================

public class ThaiWeatherService
{
    private readonly HttpClient _client;
    private readonly string _apiKey;
    private readonly TypedCache _cache = new();
    
    public ThaiWeatherService(HttpClient client, string apiKey)
    {
        _client = client;
        _apiKey = apiKey;
        _client.BaseAddress = new Uri("https://api.openweathermap.org/data/2.5/");
    }
    
    public async Task<Result<WeatherData>> GetCurrentWeatherAsync(
        string city, 
        CancellationToken ct = default)
    {
        string cacheKey = $"weather:{city.ToLower()}";
        
        if (_cache.TryGet<WeatherData>(cacheKey, out var cached))
            return Result<WeatherData>.Success(cached!);
        
        try
        {
            string url = $"weather?q={Uri.EscapeDataString(city)}&appid={_apiKey}&units=metric&lang=th";
            var response = await _client.GetAsync(url, ct);
            
            if (!response.IsSuccessStatusCode)
                return Result<WeatherData>.Failure(
                    $"Weather API error: {(int)response.StatusCode}");
            
            var data = await response.Content.ReadFromJsonAsync<OpenWeatherResponse>(cancellationToken: ct);
            if (data == null)
                return Result<WeatherData>.Failure("Invalid response");
            
            var weather = new WeatherData(
                City: data.Name,
                Country: data.Sys.Country,
                Temperature: data.Main.Temp,
                FeelsLike: data.Main.FeelsLike,
                Humidity: data.Main.Humidity,
                Description: data.Weather.FirstOrDefault()?.Description ?? "",
                Icon: data.Weather.FirstOrDefault()?.Icon ?? "",
                WindSpeed: data.Wind.Speed,
                UpdatedAt: DateTimeOffset.FromUnixTimeSeconds(data.Dt).DateTime
            );
            
            _cache.Set(cacheKey, weather, TimeSpan.FromMinutes(10));
            return Result<WeatherData>.Success(weather);
        }
        catch (Exception ex)
        {
            return Result<WeatherData>.Failure($"Error: {ex.Message}");
        }
    }
}

public record WeatherData(
    string City, string Country, double Temperature, double FeelsLike,
    int Humidity, string Description, string Icon, double WindSpeed, DateTime UpdatedAt);

// OpenWeatherMap response models
public class OpenWeatherResponse
{
    [JsonPropertyName("name")] public string Name { get; set; } = string.Empty;
    [JsonPropertyName("dt")] public long Dt { get; set; }
    [JsonPropertyName("main")] public MainData Main { get; set; } = new();
    [JsonPropertyName("weather")] public List<WeatherInfo> Weather { get; set; } = new();
    [JsonPropertyName("wind")] public WindData Wind { get; set; } = new();
    [JsonPropertyName("sys")] public SysData Sys { get; set; } = new();
}

public class MainData
{
    [JsonPropertyName("temp")] public double Temp { get; set; }
    [JsonPropertyName("feels_like")] public double FeelsLike { get; set; }
    [JsonPropertyName("humidity")] public int Humidity { get; set; }
}

public class WeatherInfo
{
    [JsonPropertyName("description")] public string Description { get; set; } = string.Empty;
    [JsonPropertyName("icon")] public string Icon { get; set; } = string.Empty;
}

public class WindData
{
    [JsonPropertyName("speed")] public double Speed { get; set; }
}

public class SysData
{
    [JsonPropertyName("country")] public string Country { get; set; } = string.Empty;
}

// ใช้งาน
var httpClient = new HttpClient();
var weatherService = new ThaiWeatherService(httpClient, "YOUR_API_KEY");

var result = await weatherService.GetCurrentWeatherAsync("Bangkok");
result.Match(
    w => Console.WriteLine($"{w.City}, {w.Country}: {w.Temperature:F1}°C, {w.Description}"),
    err => Console.WriteLine($"Error: {err}")
);
```

---

## สรุป Part 16

ใน Part 16 เราได้เรียนรู้:

1. **HttpClient Basics** - GET, POST, PUT, DELETE
2. **Error Handling** - Status codes, Retry logic
3. **Authentication** - Bearer, API Key, OAuth refresh
4. **Streaming** - SSE, Large file download
5. **Typed Client** - Generic REST client
6. **WebSocket** - Real-time communication
7. **HTTP Caching** - ETag, Cache-Control
8. **Rate Limiting** - Throttle requests
9. **Multipart Upload** - File upload
10. **Complete Service** - Weather API

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 16 | Steps 151-160*
