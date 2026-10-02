# Part 95: Enterprise SSO & Identity
## Steps 941-950: OAuth2, OIDC, JWT, Social Login, RBAC, Multi-Tenant

---

## Step 941: OAuth2 Authorization Code + PKCE

```csharp
// ============================================
// PKCE Flow for Mobile (RFC 7636)
// ============================================

// NuGet: IdentityModel.OidcClient

public class OAuthService
{
    private readonly OidcClient _client;
    
    public OAuthService()
    {
        _client = new OidcClient(new OidcClientOptions
        {
            Authority = "https://auth.fooddelivery.th",
            ClientId = "mobile-app",
            Scope = "openid profile email orders:read orders:write",
            RedirectUri = "fooddelivery://auth/callback",
            PostLogoutRedirectUri = "fooddelivery://auth/logout",
            Browser = new MauiWebAuthBrowser()
        });
    }
    
    public async Task<LoginResult> LoginAsync()
    {
        var result = await _client.LoginAsync(new LoginRequest());
        
        if (result.IsError)
            return new LoginResult(false, result.Error, null);
        
        // Store tokens securely
        await SecureStorage.Default.SetAsync("access_token", result.AccessToken);
        await SecureStorage.Default.SetAsync("refresh_token", result.RefreshToken);
        await SecureStorage.Default.SetAsync("id_token", result.IdentityToken);
        await SecureStorage.Default.SetAsync("token_expiry",
            result.AccessTokenExpiration.ToString("O"));
        
        return new LoginResult(true, null, result.User);
    }
    
    public async Task<string?> GetValidTokenAsync()
    {
        var token = await SecureStorage.Default.GetAsync("access_token");
        var expiryStr = await SecureStorage.Default.GetAsync("token_expiry");
        
        if (token == null) return null;
        
        if (DateTimeOffset.TryParse(expiryStr, out var expiry))
        {
            if (expiry > DateTimeOffset.UtcNow.AddMinutes(5))
                return token;
        }
        
        // Token expired — refresh
        return await RefreshAsync();
    }
    
    private async Task<string?> RefreshAsync()
    {
        var refreshToken = await SecureStorage.Default.GetAsync("refresh_token");
        if (refreshToken == null) return null;
        
        var result = await _client.RefreshTokenAsync(refreshToken);
        if (result.IsError) return null;
        
        await SecureStorage.Default.SetAsync("access_token", result.AccessToken);
        await SecureStorage.Default.SetAsync("token_expiry",
            result.AccessTokenExpiration.ToString("O"));
        
        return result.AccessToken;
    }
    
    public async Task LogoutAsync()
    {
        var idToken = await SecureStorage.Default.GetAsync("id_token");
        SecureStorage.Default.Remove("access_token");
        SecureStorage.Default.Remove("refresh_token");
        SecureStorage.Default.Remove("id_token");
        SecureStorage.Default.Remove("token_expiry");
        
        await _client.LogoutAsync(new LogoutRequest { IdTokenHint = idToken });
    }
}

public record LoginResult(bool Success, string? Error, System.Security.Claims.ClaimsPrincipal? User);
```

---

## Step 942: MAUI WebAuth Browser

```csharp
// ============================================
// Platform WebAuthentication for OAuth redirect
// ============================================

public class MauiWebAuthBrowser : IBrowser
{
    public async Task<BrowserResult> InvokeAsync(BrowserOptions options, CancellationToken cancellationToken = default)
    {
        try
        {
            var result = await WebAuthenticator.Default.AuthenticateAsync(
                new WebAuthenticatorOptions
                {
                    Url = new Uri(options.StartUrl),
                    CallbackUrl = new Uri(options.EndUrl),
                    PrefersEphemeralWebBrowserSession = true
                });
            
            var url = options.EndUrl;
            foreach (var (key, value) in result.Properties)
                url += (url.Contains('?') ? "&" : "?") + $"{key}={value}";
            
            return new BrowserResult
            {
                ResultType = BrowserResultType.Success,
                Response = url
            };
        }
        catch (TaskCanceledException)
        {
            return new BrowserResult { ResultType = BrowserResultType.UserCancel };
        }
        catch (Exception ex)
        {
            return new BrowserResult
            {
                ResultType = BrowserResultType.UnknownError,
                Error = ex.Message
            };
        }
    }
}
```

---

## Step 943: Social Login (Google, LINE, Apple)

```csharp
// ============================================
// Social Identity Providers
// ============================================

public class SocialLoginService
{
    private readonly OAuthService _oauth;
    private readonly IApiClient _api;
    
    public SocialLoginService(OAuthService oauth, IApiClient api)
    {
        _oauth = oauth;
        _api = api;
    }
    
    // Google Sign-In via OIDC
    public async Task<SocialLoginResult> SignInWithGoogleAsync()
    {
        // Google uses standard OIDC — use OidcClient with Google endpoints
        var client = new OidcClient(new OidcClientOptions
        {
            Authority = "https://accounts.google.com",
            ClientId = AppConfig.Config.GoogleClientId,
            Scope = "openid profile email",
            RedirectUri = "fooddelivery://google/callback",
            Browser = new MauiWebAuthBrowser()
        });
        
        var result = await client.LoginAsync();
        if (result.IsError) return new SocialLoginResult(false, result.Error);
        
        // Exchange with our backend
        return await ExchangeWithBackendAsync("google", result.IdentityToken!);
    }
    
    // LINE Login (Thai market dominant)
    public async Task<SocialLoginResult> SignInWithLineAsync()
    {
        var state = Guid.NewGuid().ToString("N");
        var nonce = Guid.NewGuid().ToString("N");
        
        var authUrl = $"https://access.line.me/oauth2/v2.1/authorize" +
            $"?response_type=code" +
            $"&client_id={AppConfig.Config.LineChannelId}" +
            $"&redirect_uri={Uri.EscapeDataString("fooddelivery://line/callback")}" +
            $"&state={state}" +
            $"&scope=profile%20openid%20email" +
            $"&nonce={nonce}";
        
        var browser = new MauiWebAuthBrowser();
        var result = await browser.InvokeAsync(new BrowserOptions(authUrl, "fooddelivery://line/callback"));
        
        if (result.ResultType != BrowserResultType.Success)
            return new SocialLoginResult(false, "User cancelled");
        
        var code = ExtractQueryParam(result.Response, "code");
        if (code == null) return new SocialLoginResult(false, "No authorization code");
        
        return await ExchangeWithBackendAsync("line", code, "authorization_code");
    }
    
    // Sign in with Apple (required for iOS App Store apps with 3rd-party login)
    public async Task<SocialLoginResult> SignInWithAppleAsync()
    {
        // NuGet: Plugin.Apple.SignIn
        var result = await AppleSignInAuthenticator.AuthenticateAsync(
            new AppleSignInAuthenticator.Options { IncludeEmailScope = true, IncludeFullNameScope = true });
        
        if (result == null) return new SocialLoginResult(false, "Authentication failed");
        
        return await ExchangeWithBackendAsync("apple", result.IdToken!);
    }
    
    private async Task<SocialLoginResult> ExchangeWithBackendAsync(
        string provider, string token, string grantType = "id_token")
    {
        try
        {
            var response = await _api.PostAsync<BackendAuthResult>("/auth/social", new
            {
                provider,
                token,
                grantType
            });
            
            if (response?.AccessToken != null)
            {
                await SecureStorage.Default.SetAsync("access_token", response.AccessToken);
                return new SocialLoginResult(true, null);
            }
            
            return new SocialLoginResult(false, "Backend exchange failed");
        }
        catch (Exception ex)
        {
            return new SocialLoginResult(false, ex.Message);
        }
    }
    
    private string? ExtractQueryParam(string url, string param)
    {
        var uri = new Uri(url);
        var query = System.Web.HttpUtility.ParseQueryString(uri.Query);
        return query[param];
    }
}

public record SocialLoginResult(bool Success, string? Error);
public record BackendAuthResult(string? AccessToken, string? RefreshToken);
```

---

## Step 944: JWT Token Parsing

```csharp
// ============================================
// Decode JWT without validation (for display)
// ============================================

public class JwtTokenParser
{
    public JwtClaims Parse(string token)
    {
        var parts = token.Split('.');
        if (parts.Length != 3)
            throw new FormatException("Invalid JWT format");
        
        var payload = DecodeBase64Url(parts[1]);
        var claims = System.Text.Json.JsonDocument.Parse(payload);
        
        var root = claims.RootElement;
        
        return new JwtClaims(
            Sub: root.TryGetProperty("sub", out var sub) ? sub.GetString() : null,
            Email: root.TryGetProperty("email", out var email) ? email.GetString() : null,
            Name: root.TryGetProperty("name", out var name) ? name.GetString() : null,
            Picture: root.TryGetProperty("picture", out var pic) ? pic.GetString() : null,
            Roles: root.TryGetProperty("roles", out var roles)
                ? roles.EnumerateArray().Select(r => r.GetString()!).ToList()
                : new List<string>(),
            Exp: root.TryGetProperty("exp", out var exp)
                ? DateTimeOffset.FromUnixTimeSeconds(exp.GetInt64())
                : (DateTimeOffset?)null,
            Iss: root.TryGetProperty("iss", out var iss) ? iss.GetString() : null,
            TenantId: root.TryGetProperty("tenant_id", out var tid) ? tid.GetString() : null
        );
    }
    
    private static string DecodeBase64Url(string base64Url)
    {
        var padded = base64Url
            .Replace('-', '+').Replace('_', '/')
            .PadRight(base64Url.Length + (4 - base64Url.Length % 4) % 4, '=');
        
        var bytes = Convert.FromBase64String(padded);
        return System.Text.Encoding.UTF8.GetString(bytes);
    }
}

public record JwtClaims(
    string? Sub, string? Email, string? Name, string? Picture,
    List<string> Roles, DateTimeOffset? Exp, string? Iss, string? TenantId);
```

---

## Step 945: Role-Based Access Control (RBAC)

```csharp
// ============================================
// Permission-based access control
// ============================================

public static class Permissions
{
    public const string ViewOrders = "orders:read";
    public const string PlaceOrders = "orders:write";
    public const string ManageRestaurant = "restaurant:manage";
    public const string ViewAnalytics = "analytics:read";
    public const string ManageUsers = "users:manage";
    public const string AdminAccess = "admin:access";
}

public class AuthorizationService
{
    private readonly OAuthService _oauth;
    private JwtClaims? _cachedClaims;
    private readonly JwtTokenParser _parser = new();
    
    public AuthorizationService(OAuthService oauth) => _oauth = oauth;
    
    public async Task<bool> HasPermissionAsync(string permission)
    {
        var claims = await GetClaimsAsync();
        return claims?.Roles.Contains(permission) == true;
    }
    
    public async Task<bool> IsInRoleAsync(string role)
    {
        var claims = await GetClaimsAsync();
        return claims?.Roles.Contains(role) == true;
    }
    
    public async Task<string?> GetUserIdAsync()
        => (await GetClaimsAsync())?.Sub;
    
    public async Task<string?> GetTenantIdAsync()
        => (await GetClaimsAsync())?.TenantId;
    
    private async Task<JwtClaims?> GetClaimsAsync()
    {
        if (_cachedClaims != null) return _cachedClaims;
        
        var token = await _oauth.GetValidTokenAsync();
        if (token == null) return null;
        
        _cachedClaims = _parser.Parse(token);
        return _cachedClaims;
    }
    
    public void ClearCache() => _cachedClaims = null;
}

// Permission guard for UI
public static class AuthGuard
{
    public static async Task RequireAsync(AuthorizationService auth, string permission)
    {
        if (!await auth.HasPermissionAsync(permission))
            throw new UnauthorizedAccessException($"Missing permission: {permission}");
    }
}

// Usage:
// await AuthGuard.RequireAsync(_auth, Permissions.ManageRestaurant);
```

---

## Step 946: Multi-Tenant Session

```csharp
// ============================================
// Multi-Tenant: switch between business accounts
// ============================================

public class TenantContext
{
    private string? _currentTenantId;
    
    public string? CurrentTenantId => _currentTenantId;
    
    public event EventHandler<string>? TenantChanged;
    
    public async Task SwitchTenantAsync(string tenantId)
    {
        _currentTenantId = tenantId;
        
        // Store last tenant
        Preferences.Set("last_tenant_id", tenantId);
        
        // Clear per-tenant caches
        WeakReferenceMessenger.Default.Send(new TenantSwitchedMessage(tenantId));
        TenantChanged?.Invoke(this, tenantId);
    }
    
    public Task RestoreLastTenantAsync()
    {
        var last = Preferences.Get("last_tenant_id", "");
        if (!string.IsNullOrEmpty(last))
            _currentTenantId = last;
        return Task.CompletedTask;
    }
}

public record TenantSwitchedMessage(string TenantId);

// Tenant-aware HTTP handler
public class TenantHttpHandler : DelegatingHandler
{
    private readonly TenantContext _tenant;
    
    public TenantHttpHandler(TenantContext tenant) => _tenant = tenant;
    
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (!string.IsNullOrEmpty(_tenant.CurrentTenantId))
            request.Headers.Add("X-Tenant-Id", _tenant.CurrentTenantId);
        
        return base.SendAsync(request, cancellationToken);
    }
}
```

---

## Step 947: Session Timeout & Auto-Lock

```csharp
// ============================================
// Idle session timeout with auto-logout
// ============================================

public class SessionTimeoutService
{
    private Timer? _timer;
    private DateTime _lastActivity = DateTime.UtcNow;
    private readonly TimeSpan _timeout;
    private readonly OAuthService _auth;
    
    public event EventHandler? SessionExpired;
    
    public SessionTimeoutService(OAuthService auth, TimeSpan? timeout = null)
    {
        _auth = auth;
        _timeout = timeout ?? TimeSpan.FromMinutes(30);
    }
    
    public void Start()
    {
        _timer = new Timer(CheckTimeout, null,
            TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1));
    }
    
    public void RecordActivity()
        => _lastActivity = DateTime.UtcNow;
    
    private async void CheckTimeout(object? _)
    {
        if (DateTime.UtcNow - _lastActivity <= _timeout) return;
        
        _timer?.Dispose();
        await _auth.LogoutAsync();
        
        await MainThread.InvokeOnMainThreadAsync(() =>
            SessionExpired?.Invoke(this, EventArgs.Empty));
    }
    
    public void Stop() => _timer?.Dispose();
}

// App.xaml.cs
// _sessionTimeout.SessionExpired += async (_, _) =>
//     await Shell.Current.GoToAsync("//login?reason=timeout");
```

---

## Step 948: Device Registration & Trust

```csharp
// ============================================
// Trusted Device Management
// ============================================

public class DeviceTrustService
{
    private readonly IApiClient _api;
    private readonly string _deviceId;
    
    public DeviceTrustService(IApiClient api)
    {
        _api = api;
        _deviceId = GetOrCreateDeviceId();
    }
    
    public async Task<bool> IsDeviceTrustedAsync()
    {
        var fingerprint = await GetDeviceFingerprintAsync();
        var storedToken = await SecureStorage.Default.GetAsync("device_trust_token");
        
        if (storedToken == null) return false;
        
        return await _api.PostAsync<TrustCheckResult>("/auth/device/verify", new
        {
            deviceId = _deviceId,
            fingerprint,
            trustToken = storedToken
        }) is { IsTrusted: true };
    }
    
    public async Task RegisterDeviceAsync(string displayName)
    {
        var fingerprint = await GetDeviceFingerprintAsync();
        
        var result = await _api.PostAsync<DeviceRegistrationResult>("/auth/device/register", new
        {
            deviceId = _deviceId,
            displayName,
            fingerprint,
            platform = DeviceInfo.Platform.ToString(),
            osVersion = DeviceInfo.VersionString,
            model = DeviceInfo.Model
        });
        
        if (result?.TrustToken != null)
            await SecureStorage.Default.SetAsync("device_trust_token", result.TrustToken);
    }
    
    private async Task<string> GetDeviceFingerprintAsync()
    {
        var data = $"{DeviceInfo.Platform}|{DeviceInfo.Model}|{DeviceInfo.Idiom}|{_deviceId}";
        var bytes = System.Text.Encoding.UTF8.GetBytes(data);
        using var sha = System.Security.Cryptography.SHA256.Create();
        return Convert.ToHexString(sha.ComputeHash(bytes));
    }
    
    private static string GetOrCreateDeviceId()
    {
        var existing = SecureStorage.Default.GetAsync("device_id").Result;
        if (existing != null) return existing;
        
        var id = Guid.NewGuid().ToString("N");
        SecureStorage.Default.SetAsync("device_id", id).Wait();
        return id;
    }
}

public record TrustCheckResult(bool IsTrusted);
public record DeviceRegistrationResult(string TrustToken);
```

---

## Step 949: Account Linking

```csharp
// ============================================
// Link multiple identity providers to one account
// ============================================

public class AccountLinkingService
{
    private readonly IApiClient _api;
    private readonly OAuthService _oauth;
    
    public AccountLinkingService(IApiClient api, OAuthService oauth)
    {
        _api = api;
        _oauth = oauth;
    }
    
    public async Task<List<LinkedProvider>> GetLinkedProvidersAsync()
        => await _api.GetAsync<List<LinkedProvider>>("/auth/linked-providers") ?? new();
    
    public async Task<bool> LinkGoogleAsync()
    {
        var client = new OidcClient(new OidcClientOptions
        {
            Authority = "https://accounts.google.com",
            ClientId = AppConfig.Config.GoogleClientId,
            Scope = "openid profile email",
            RedirectUri = "fooddelivery://link/google",
            Browser = new MauiWebAuthBrowser()
        });
        
        var result = await client.LoginAsync();
        if (result.IsError) return false;
        
        return await _api.PostAsync<bool>("/auth/link", new
        {
            provider = "google",
            idToken = result.IdentityToken
        });
    }
    
    public async Task<bool> UnlinkProviderAsync(string provider)
    {
        // Ensure at least one login method remains
        var linked = await GetLinkedProvidersAsync();
        if (linked.Count <= 1) return false; // Can't unlink last provider
        
        return await _api.DeleteAsync<bool>($"/auth/linked-providers/{provider}");
    }
    
    public async Task<bool> MergeAccountsAsync(string targetAccountToken)
    {
        return await _api.PostAsync<bool>("/auth/merge", new
        {
            targetToken = targetAccountToken
        });
    }
}

public record LinkedProvider(string Provider, string Email, DateTime LinkedAt, bool IsPrimary);
```

---

## Step 950: SSO Tests

```csharp
// ============================================
// Auth Service Tests
// ============================================

[TestFixture]
public class AuthTests
{
    [Test]
    public void JwtParser_DecodesClaims()
    {
        // Create a test JWT (not signed, just for parsing)
        var header = Base64UrlEncode("""{"alg":"RS256","typ":"JWT"}""");
        var payload = Base64UrlEncode("""{"sub":"user-123","email":"test@example.com","roles":["orders:read","orders:write"],"exp":9999999999,"iss":"https://auth.fooddelivery.th"}""");
        var token = $"{header}.{payload}.signature";
        
        var parser = new JwtTokenParser();
        var claims = parser.Parse(token);
        
        Assert.That(claims.Sub, Is.EqualTo("user-123"));
        Assert.That(claims.Email, Is.EqualTo("test@example.com"));
        Assert.That(claims.Roles, Contains.Item("orders:read"));
        Assert.That(claims.Roles, Contains.Item("orders:write"));
    }
    
    [Test]
    public async Task AuthorizationService_HasPermission_WhenRolePresent()
    {
        var mockOauth = Substitute.For<OAuthService>();
        var header = Base64UrlEncode("""{"alg":"RS256","typ":"JWT"}""");
        var payload = Base64UrlEncode("""{"sub":"user-1","roles":["orders:read"],"exp":9999999999}""");
        mockOauth.GetValidTokenAsync().Returns($"{header}.{payload}.sig");
        
        var authSvc = new AuthorizationService(mockOauth);
        Assert.That(await authSvc.HasPermissionAsync(Permissions.ViewOrders), Is.True);
        Assert.That(await authSvc.HasPermissionAsync(Permissions.AdminAccess), Is.False);
    }
    
    private string Base64UrlEncode(string input)
        => Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(input))
            .Replace('+', '-').Replace('/', '_').TrimEnd('=');
}
```

---

## สรุป Part 95

ใน Part 95 เราได้เรียนรู้:

1. **OAuth2 + PKCE** - OidcClient, PKCE flow, token storage, refresh
2. **MAUI WebAuth** - WebAuthenticator, callback URL handling
3. **Social Login** - Google OIDC, LINE OAuth2, Sign in with Apple
4. **JWT Parsing** - Base64Url decode, claims extraction, expiry check
5. **RBAC** - Permission constants, claim-based role check
6. **Multi-Tenant** - TenantContext, X-Tenant-Id header injection
7. **Session Timeout** - 30-minute idle auto-logout timer
8. **Device Trust** - Device fingerprint, trust token, registration
9. **Account Linking** - Multiple providers per account, merge
10. **Auth Tests** - JWT decode, permission check, mock OAuthService

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 95 | Steps 941-950*
