# Part 51: Enterprise Security
## Steps 501-510: Auth, RBAC, Certificate Pinning, Secure Storage

---

## Step 501: JWT Authentication

```csharp
// ============================================
// JWT Token Management
// ============================================

public class AuthTokenService
{
    private const string AccessTokenKey = "auth_access_token";
    private const string RefreshTokenKey = "auth_refresh_token";
    private const string ExpiresAtKey = "auth_expires_at";
    
    private readonly ISecureStorage _storage;
    private readonly HttpClient _http;
    
    public AuthTokenService(ISecureStorage storage, HttpClient http)
    {
        _storage = storage;
        _http = http;
    }
    
    public async Task<string?> GetValidTokenAsync(CancellationToken ct = default)
    {
        var accessToken = await _storage.GetAsync(AccessTokenKey);
        var expiresAtStr = await _storage.GetAsync(ExpiresAtKey);
        
        if (string.IsNullOrEmpty(accessToken)) return null;
        
        // Check expiry with 5-minute buffer
        if (DateTime.TryParse(expiresAtStr, out var expiresAt) &&
            expiresAt > DateTime.UtcNow.AddMinutes(5))
        {
            return accessToken;
        }
        
        // Try refresh
        return await RefreshAsync(ct);
    }
    
    public async Task SaveTokensAsync(TokenResponse tokens)
    {
        await _storage.SetAsync(AccessTokenKey, tokens.AccessToken);
        await _storage.SetAsync(RefreshTokenKey, tokens.RefreshToken);
        await _storage.SetAsync(ExpiresAtKey,
            DateTime.UtcNow.AddSeconds(tokens.ExpiresIn).ToString("O"));
    }
    
    public async Task ClearAsync()
    {
        _storage.Remove(AccessTokenKey);
        _storage.Remove(RefreshTokenKey);
        _storage.Remove(ExpiresAtKey);
    }
    
    private async Task<string?> RefreshAsync(CancellationToken ct)
    {
        var refreshToken = await _storage.GetAsync(RefreshTokenKey);
        if (string.IsNullOrEmpty(refreshToken)) return null;
        
        var response = await _http.PostAsJsonAsync("auth/refresh",
            new { RefreshToken = refreshToken }, ct);
        
        if (!response.IsSuccessStatusCode)
        {
            await ClearAsync();
            return null;
        }
        
        var tokens = await response.Content.ReadFromJsonAsync<TokenResponse>(ct);
        if (tokens == null) return null;
        
        await SaveTokensAsync(tokens);
        return tokens.AccessToken;
    }
}

public record TokenResponse(string AccessToken, string RefreshToken, int ExpiresIn);

// DelegatingHandler to auto-attach token
public class AuthHeaderHandler : DelegatingHandler
{
    private readonly AuthTokenService _tokens;
    
    public AuthHeaderHandler(AuthTokenService tokens) => _tokens = tokens;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokens.GetValidTokenAsync(ct);
        
        if (!string.IsNullOrEmpty(token))
            request.Headers.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        
        var response = await base.SendAsync(request, ct);
        
        // Handle 401 - token revoked
        if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            await _tokens.ClearAsync();
            MessagingCenter.Send<object>(this, "SessionExpired");
        }
        
        return response;
    }
}
```

---

## Step 502: Biometric Authentication

```csharp
// ============================================
// Fingerprint / Face ID
// ============================================

public interface IBiometricService
{
    Task<BiometricAvailability> GetAvailabilityAsync();
    Task<bool> AuthenticateAsync(string reason);
}

public enum BiometricAvailability { Available, NotEnrolled, HardwareNotPresent, Unavailable }

#if ANDROID
public class AndroidBiometricService : IBiometricService
{
    public async Task<BiometricAvailability> GetAvailabilityAsync()
    {
        await Task.CompletedTask;
        var manager = AndroidX.Biometric.BiometricManager.From(
            Android.App.Application.Context);
        
        var canAuth = manager.CanAuthenticate(
            AndroidX.Biometric.BiometricManager.Authenticators.BiometricWeak);
        
        return canAuth switch
        {
            AndroidX.Biometric.BiometricManager.BiometricSuccess => BiometricAvailability.Available,
            AndroidX.Biometric.BiometricManager.BiometricErrorNoneEnrolled => BiometricAvailability.NotEnrolled,
            AndroidX.Biometric.BiometricManager.BiometricErrorHwUnavailable => BiometricAvailability.HardwareNotPresent,
            _ => BiometricAvailability.Unavailable
        };
    }
    
    public async Task<bool> AuthenticateAsync(string reason)
    {
        var tcs = new TaskCompletionSource<bool>();
        
        var activity = Platform.CurrentActivity!;
        var executor = AndroidX.Core.Content.ContextCompat
            .GetMainExecutor(activity);
        
        var prompt = new AndroidX.Biometric.BiometricPrompt(
            (AndroidX.Fragment.App.FragmentActivity)activity, executor,
            new BiometricCallback(tcs));
        
        var info = new AndroidX.Biometric.BiometricPrompt.PromptInfo.Builder()
            .SetTitle("ยืนยันตัวตน")
            .SetSubtitle(reason)
            .SetNegativeButtonText("ยกเลิก")
            .Build();
        
        activity.RunOnUiThread(() => prompt.Authenticate(info));
        
        return await tcs.Task;
    }
    
    private class BiometricCallback : AndroidX.Biometric.BiometricPrompt.AuthenticationCallback
    {
        private readonly TaskCompletionSource<bool> _tcs;
        public BiometricCallback(TaskCompletionSource<bool> tcs) => _tcs = tcs;
        
        public override void OnAuthenticationSucceeded(AndroidX.Biometric.BiometricPrompt.AuthenticationResult result)
            => _tcs.TrySetResult(true);
        
        public override void OnAuthenticationError(int errorCode, Java.Lang.ICharSequence errString)
            => _tcs.TrySetResult(false);
        
        public override void OnAuthenticationFailed()
            => _tcs.TrySetResult(false);
    }
}
#endif
```

---

## Step 503: Role-Based Access Control

```csharp
// ============================================
// RBAC
// ============================================

public class UserSession
{
    public int UserId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public List<string> Roles { get; set; } = new();
    public List<string> Permissions { get; set; } = new();
    
    public bool HasRole(string role) => Roles.Contains(role);
    public bool HasPermission(string permission) => Permissions.Contains(permission);
    
    public bool IsAdmin => HasRole("admin");
    public bool IsManager => HasRole("manager") || IsAdmin;
    
    // Permission constants
    public static class Permissions
    {
        public const string ProductCreate = "product.create";
        public const string ProductUpdate = "product.update";
        public const string ProductDelete = "product.delete";
        public const string OrderView = "order.view";
        public const string OrderManage = "order.manage";
        public const string UserManage = "user.manage";
        public const string ReportView = "report.view";
        public const string ReportExport = "report.export";
    }
}

public interface ICurrentUserService
{
    UserSession? Current { get; }
    bool IsAuthenticated { get; }
}

public class CurrentUserService : ICurrentUserService
{
    private UserSession? _session;
    
    public UserSession? Current => _session;
    public bool IsAuthenticated => _session != null;
    
    public void SetUser(UserSession session) => _session = session;
    public void Clear() => _session = null;
}

// Guard class for ViewModels
public class AuthGuard
{
    private readonly ICurrentUserService _currentUser;
    private readonly INavigationService _navigation;
    
    public AuthGuard(ICurrentUserService currentUser, INavigationService navigation)
    {
        _currentUser = currentUser;
        _navigation = navigation;
    }
    
    public async Task<bool> RequireAuthAsync()
    {
        if (_currentUser.IsAuthenticated) return true;
        await _navigation.GoToAsync<LoginPage>();
        return false;
    }
    
    public async Task<bool> RequirePermissionAsync(string permission)
    {
        if (!await RequireAuthAsync()) return false;
        
        if (_currentUser.Current!.HasPermission(permission)) return true;
        
        await Shell.Current.DisplayAlert(
            "ไม่มีสิทธิ์", "คุณไม่มีสิทธิ์เข้าถึงส่วนนี้", "ตกลง");
        return false;
    }
}

// Usage in ViewModel
public partial class ProductManageViewModel : ObservableObject
{
    private readonly AuthGuard _guard;
    
    [RelayCommand]
    private async Task DeleteProductAsync(int id)
    {
        if (!await _guard.RequirePermissionAsync(UserSession.Permissions.ProductDelete))
            return;
        
        // proceed with delete
    }
}
```

---

## Step 504: Certificate Pinning

```csharp
// ============================================
// SSL Certificate Pinning
// ============================================

public class PinnedHttpClientHandler : HttpClientHandler
{
    // SHA-256 hash of the server's public key
    private static readonly HashSet<string> _pinnedHashes = new()
    {
        "abc123def456...", // production cert hash
        "xyz789uvw012...", // backup cert hash
    };
    
    public PinnedHttpClientHandler()
    {
        ServerCertificateCustomValidationCallback = ValidateCertificate;
    }
    
    private static bool ValidateCertificate(
        HttpRequestMessage request,
        System.Security.Cryptography.X509Certificates.X509Certificate2? certificate,
        System.Security.Cryptography.X509Certificates.X509Chain? chain,
        System.Net.Security.SslPolicyErrors sslPolicyErrors)
    {
        if (certificate == null) return false;
        if (sslPolicyErrors != System.Net.Security.SslPolicyErrors.None) return false;
        
        // Get public key hash
        var pubKey = certificate.GetPublicKey();
        var hash = System.Security.Cryptography.SHA256.HashData(pubKey);
        var hashHex = Convert.ToHexString(hash).ToLowerInvariant();
        
        return _pinnedHashes.Contains(hashHex);
    }
}

// Register in DI
// builder.Services.AddHttpClient<IApiClient, ApiClient>()
//     .ConfigurePrimaryHttpMessageHandler(() => new PinnedHttpClientHandler());

// Dynamic pin from remote config (allows cert rotation)
public class DynamicCertificatePinner
{
    private HashSet<string> _pins = new();
    private readonly HttpClient _plainHttp; // unpinned, for fetching pins
    
    public DynamicCertificatePinner(HttpClient plainHttp) => _plainHttp = plainHttp;
    
    public async Task RefreshPinsAsync()
    {
        var response = await _plainHttp.GetFromJsonAsync<CertPins>(
            "https://config.myapp.com/cert-pins.json");
        
        if (response != null)
            _pins = response.Pins.ToHashSet();
    }
    
    public bool IsValid(string certHash) => _pins.Contains(certHash);
}

public record CertPins(List<string> Pins, DateTime ValidUntil);
```

---

## Step 505: Secure Data Storage

```csharp
// ============================================
// Secure Storage Patterns
// ============================================

public class SecureDataService
{
    private readonly ISecureStorage _secureStorage;
    private readonly IPreferences _preferences;
    
    public SecureDataService(ISecureStorage secureStorage, IPreferences preferences)
    {
        _secureStorage = secureStorage;
        _preferences = preferences;
    }
    
    // Tokens and secrets → SecureStorage (Keychain/Keystore)
    public async Task SaveTokenAsync(string token)
        => await _secureStorage.SetAsync("auth_token", token);
    
    public async Task<string?> GetTokenAsync()
        => await _secureStorage.GetAsync("auth_token");
    
    // User preferences → Preferences (UserDefaults/SharedPreferences)
    public void SaveTheme(string theme)
        => _preferences.Set("app_theme", theme);
    
    public string GetTheme()
        => _preferences.Get("app_theme", "system");
    
    // Sensitive data encryption for SQLite
    public string EncryptValue(string value, string key)
    {
        var keyBytes = System.Text.Encoding.UTF8.GetBytes(key.PadRight(32)[..32]);
        var iv = System.Security.Cryptography.RandomNumberGenerator.GetBytes(16);
        
        using var aes = System.Security.Cryptography.Aes.Create();
        aes.Key = keyBytes;
        aes.IV = iv;
        
        using var encryptor = aes.CreateEncryptor();
        var plainBytes = System.Text.Encoding.UTF8.GetBytes(value);
        var cipherBytes = encryptor.TransformFinalBlock(plainBytes, 0, plainBytes.Length);
        
        var combined = new byte[iv.Length + cipherBytes.Length];
        iv.CopyTo(combined, 0);
        cipherBytes.CopyTo(combined, iv.Length);
        
        return Convert.ToBase64String(combined);
    }
    
    public string DecryptValue(string encrypted, string key)
    {
        var keyBytes = System.Text.Encoding.UTF8.GetBytes(key.PadRight(32)[..32]);
        var combined = Convert.FromBase64String(encrypted);
        
        var iv = combined[..16];
        var cipherBytes = combined[16..];
        
        using var aes = System.Security.Cryptography.Aes.Create();
        aes.Key = keyBytes;
        aes.IV = iv;
        
        using var decryptor = aes.CreateDecryptor();
        var plainBytes = decryptor.TransformFinalBlock(cipherBytes, 0, cipherBytes.Length);
        
        return System.Text.Encoding.UTF8.GetString(plainBytes);
    }
}
```

---

## Step 506: OAuth 2.0 / OIDC

```csharp
// ============================================
// OAuth 2.0 with PKCE
// ============================================

public class OAuthService
{
    private readonly HttpClient _http;
    private readonly AuthTokenService _tokens;
    
    private const string ClientId = "myapp";
    private const string RedirectUri = "myapp://callback";
    private const string Scopes = "openid profile email offline_access";
    
    public OAuthService(HttpClient http, AuthTokenService tokens)
    {
        _http = http;
        _tokens = tokens;
    }
    
    public async Task<bool> LoginAsync()
    {
        // Generate PKCE
        var codeVerifier = GenerateCodeVerifier();
        var codeChallenge = GenerateCodeChallenge(codeVerifier);
        var state = Convert.ToBase64String(
            System.Security.Cryptography.RandomNumberGenerator.GetBytes(16));
        
        // Build authorization URL
        var authUrl = $"https://auth.myapp.com/authorize?" +
            $"response_type=code" +
            $"&client_id={ClientId}" +
            $"&redirect_uri={Uri.EscapeDataString(RedirectUri)}" +
            $"&scope={Uri.EscapeDataString(Scopes)}" +
            $"&state={state}" +
            $"&code_challenge={codeChallenge}" +
            $"&code_challenge_method=S256";
        
        // Open browser for login
        var result = await WebAuthenticator.Default.AuthenticateAsync(
            new Uri(authUrl), new Uri(RedirectUri));
        
        if (!result.Properties.TryGetValue("code", out var code)) return false;
        if (result.Properties.GetValueOrDefault("state") != state) return false;
        
        // Exchange code for tokens
        var tokenResponse = await ExchangeCodeAsync(code, codeVerifier);
        if (tokenResponse == null) return false;
        
        await _tokens.SaveTokensAsync(tokenResponse);
        return true;
    }
    
    private async Task<TokenResponse?> ExchangeCodeAsync(string code, string verifier)
    {
        var body = new Dictionary<string, string>
        {
            ["grant_type"] = "authorization_code",
            ["code"] = code,
            ["redirect_uri"] = RedirectUri,
            ["client_id"] = ClientId,
            ["code_verifier"] = verifier
        };
        
        var response = await _http.PostAsync("https://auth.myapp.com/token",
            new FormUrlEncodedContent(body));
        
        if (!response.IsSuccessStatusCode) return null;
        
        return await response.Content.ReadFromJsonAsync<TokenResponse>();
    }
    
    private static string GenerateCodeVerifier()
    {
        var bytes = System.Security.Cryptography.RandomNumberGenerator.GetBytes(32);
        return Convert.ToBase64String(bytes)
            .Replace("+", "-").Replace("/", "_").TrimEnd('=');
    }
    
    private static string GenerateCodeChallenge(string verifier)
    {
        var hash = System.Security.Cryptography.SHA256.HashData(
            System.Text.Encoding.ASCII.GetBytes(verifier));
        return Convert.ToBase64String(hash)
            .Replace("+", "-").Replace("/", "_").TrimEnd('=');
    }
}
```

---

## Step 507: App Hardening

```csharp
// ============================================
// App Hardening Against Reverse Engineering
// ============================================

public class SecurityHardeningService
{
    public bool IsDeviceRooted()
    {
#if ANDROID
        var rootPaths = new[]
        {
            "/system/app/Superuser.apk",
            "/system/xbin/su",
            "/system/bin/su",
            "/data/local/xbin/su"
        };
        
        return rootPaths.Any(File.Exists) || CheckBuildTags();
#elif IOS
        var jailbreakPaths = new[]
        {
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash"
        };
        
        return jailbreakPaths.Any(File.Exists);
#else
        return false;
#endif
    }
    
#if ANDROID
    private static bool CheckBuildTags()
    {
        var tags = Android.OS.Build.Tags;
        return tags?.Contains("test-keys") == true;
    }
#endif
    
    public bool IsRunningInEmulator()
    {
#if ANDROID
        return Android.OS.Build.Fingerprint?.StartsWith("generic") == true ||
               Android.OS.Build.Fingerprint?.StartsWith("unknown") == true ||
               Android.OS.Build.Model?.Contains("Emulator") == true ||
               Android.OS.Build.Model?.Contains("Android SDK") == true;
#elif IOS
        return DeviceInfo.DeviceType == DeviceType.Virtual;
#else
        return false;
#endif
    }
    
    public bool IsBeingDebugged()
    {
        return System.Diagnostics.Debugger.IsAttached;
    }
    
    public SecurityAssessment Assess()
    {
        return new SecurityAssessment(
            IsRooted: IsDeviceRooted(),
            IsEmulator: IsRunningInEmulator(),
            IsDebugged: IsBeingDebugged());
    }
}

public record SecurityAssessment(bool IsRooted, bool IsEmulator, bool IsDebugged)
{
    public bool IsHighRisk => IsRooted || IsDebugged;
    public bool IsMediumRisk => IsEmulator;
}
```

---

## Step 508: Audit Trail

```csharp
// ============================================
// Audit Trail
// ============================================

public class AuditService
{
    private readonly SQLiteConnection _db;
    private readonly ICurrentUserService _user;
    
    public AuditService(SQLiteConnection db, ICurrentUserService user)
    {
        _db = db;
        _user = user;
        _db.CreateTable<AuditEntry>();
    }
    
    public void Log(string action, string entityType, int entityId,
        object? before = null, object? after = null)
    {
        _db.Insert(new AuditEntry
        {
            UserId = _user.Current?.UserId,
            UserName = _user.Current?.Name,
            Action = action,
            EntityType = entityType,
            EntityId = entityId,
            Before = before != null
                ? System.Text.Json.JsonSerializer.Serialize(before) : null,
            After = after != null
                ? System.Text.Json.JsonSerializer.Serialize(after) : null,
            Timestamp = DateTime.UtcNow,
            IpAddress = GetClientIp(),
            DeviceInfo = $"{DeviceInfo.Current.Model} {DeviceInfo.Current.VersionString}"
        });
    }
    
    public List<AuditEntry> GetEntityHistory(string entityType, int entityId)
    {
        return _db.Table<AuditEntry>()
            .Where(e => e.EntityType == entityType && e.EntityId == entityId)
            .OrderByDescending(e => e.Timestamp)
            .ToList();
    }
    
    public List<AuditEntry> GetUserActivity(int userId, DateTime? from = null)
    {
        var query = _db.Table<AuditEntry>()
            .Where(e => e.UserId == userId);
        
        if (from.HasValue)
            query = query.Where(e => e.Timestamp >= from.Value);
        
        return query.OrderByDescending(e => e.Timestamp).Take(100).ToList();
    }
    
    private static string? GetClientIp() => null; // mobile apps don't have client IP
}

public class AuditEntry
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public int? UserId { get; set; }
    public string? UserName { get; set; }
    public string Action { get; set; } = string.Empty;
    public string EntityType { get; set; } = string.Empty;
    public int EntityId { get; set; }
    public string? Before { get; set; }
    public string? After { get; set; }
    public DateTime Timestamp { get; set; }
    public string? IpAddress { get; set; }
    public string? DeviceInfo { get; set; }
}

// Audit actions
public static class AuditActions
{
    public const string Create = "CREATE";
    public const string Update = "UPDATE";
    public const string Delete = "DELETE";
    public const string View = "VIEW";
    public const string Login = "LOGIN";
    public const string Logout = "LOGOUT";
    public const string Export = "EXPORT";
}
```

---

## Step 509: Rate Limiting Client-Side

```csharp
// ============================================
// Client-Side Rate Limiting
// ============================================

public class RateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DateTime> _requestTimes = new();
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    private readonly object _lock = new();
    
    public RateLimiter(int maxRequestsPerWindow, TimeSpan window)
    {
        _maxRequests = maxRequestsPerWindow;
        _window = window;
        _semaphore = new SemaphoreSlim(maxRequestsPerWindow, maxRequestsPerWindow);
    }
    
    public async Task WaitAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        
        lock (_lock)
        {
            var now = DateTime.UtcNow;
            var cutoff = now - _window;
            
            while (_requestTimes.Count > 0 && _requestTimes.Peek() < cutoff)
                _requestTimes.Dequeue();
            
            _requestTimes.Enqueue(now);
            
            if (_requestTimes.Count < _maxRequests)
                _semaphore.Release();
        }
    }
}

// Search debounce + rate limiting
public partial class SearchViewModel : ObservableObject
{
    private readonly RateLimiter _rateLimiter = new(10, TimeSpan.FromMinutes(1));
    private CancellationTokenSource? _searchCts;
    
    [ObservableProperty] private string _searchText = string.Empty;
    
    partial void OnSearchTextChanged(string value)
    {
        _searchCts?.Cancel();
        _searchCts = new CancellationTokenSource();
        
        // Debounce 300ms
        Task.Delay(300, _searchCts.Token).ContinueWith(async t =>
        {
            if (t.IsCanceled) return;
            
            await _rateLimiter.WaitAsync();
            await PerformSearchAsync(value);
        });
    }
    
    private async Task PerformSearchAsync(string query) { /* ... */ }
}
```

---

## Step 510: Security Testing

```csharp
// ============================================
// Security Test Helpers
// ============================================

public class SecurityTestSuite
{
    [Fact]
    public void SecureStorage_StoresAndRetrieves()
    {
        var storage = SecureStorage.Default;
        storage.SetAsync("test_key", "sensitive_value").Wait();
        
        var retrieved = storage.GetAsync("test_key").Result;
        
        retrieved.Should().Be("sensitive_value");
        storage.Remove("test_key");
    }
    
    [Fact]
    public void EncryptDecrypt_RoundTrip_Succeeds()
    {
        var service = new SecureDataService(SecureStorage.Default, Preferences.Default);
        var original = "sensitive data 123";
        var key = "my-secret-key-32";
        
        var encrypted = service.EncryptValue(original, key);
        var decrypted = service.DecryptValue(encrypted, key);
        
        encrypted.Should().NotBe(original);
        decrypted.Should().Be(original);
    }
    
    [Fact]
    public void SqlInjection_IsPreventedByParameterized()
    {
        var db = new SQLiteConnection(":memory:");
        db.CreateTable<Product>();
        
        // Insert legitimate product
        db.Insert(new Product { Name = "สินค้า", Price = 100 });
        
        // Attempt SQL injection
        var maliciousInput = "'; DROP TABLE Product; --";
        var results = db.Query<Product>(
            "SELECT * FROM Product WHERE Name = ?", maliciousInput);
        
        // Should return empty, not crash or drop table
        results.Should().BeEmpty();
        db.Table<Product>().Count().Should().Be(1); // table still exists
    }
}
```

---

## สรุป Part 51

ใน Part 51 เราได้เรียนรู้:

1. **JWT Authentication** - Token storage, refresh, auto-attach
2. **Biometric Auth** - Fingerprint/FaceID on Android
3. **RBAC** - Roles, permissions, guards
4. **Certificate Pinning** - SHA-256 public key pinning
5. **Secure Storage** - Keychain/Keystore, AES encryption
6. **OAuth 2.0 + PKCE** - Authorization code flow
7. **App Hardening** - Root/jailbreak detection
8. **Audit Trail** - SQLite audit log
9. **Rate Limiting** - Client-side request limiting
10. **Security Testing** - Injection prevention tests

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 51 | Steps 501-510*
