# Part 27: Security ใน .NET MAUI
## Steps 261-270: App Security

---

## Step 261: Security Overview

```csharp
// ============================================
// Security Principles
// ============================================

/*
 * Mobile App Security ต้องระวัง:
 * 
 * 1. Data Storage Security
 *    - อย่าเก็บ sensitive data ใน plain text
 *    - ใช้ SecureStorage สำหรับ tokens, passwords
 *    - Encrypt database files
 *
 * 2. Network Security
 *    - ใช้ HTTPS เสมอ
 *    - Certificate Pinning
 *    - Validate server certificates
 *
 * 3. Authentication
 *    - JWT token validation
 *    - Refresh token rotation
 *    - Biometric authentication
 *
 * 4. Code Security
 *    - Obfuscation
 *    - Anti-tampering
 *    - Remove debug info from production
 *
 * 5. Input Validation
 *    - Sanitize all user input
 *    - Prevent injection attacks
 */
```

---

## Step 262: Encryption

```csharp
// ============================================
// AES Encryption
// ============================================

using System.Security.Cryptography;
using System.Text;

public class EncryptionService
{
    private readonly byte[] _key;
    
    public EncryptionService(string passphrase)
    {
        // Derive key from passphrase
        using var deriveBytes = new Rfc2898DeriveBytes(
            passphrase,
            salt: Encoding.UTF8.GetBytes("MySalt12345678"),
            iterations: 100_000,
            HashAlgorithmName.SHA256);
        
        _key = deriveBytes.GetBytes(32); // 256-bit key
    }
    
    public string Encrypt(string plainText)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        using var ms = new MemoryStream();
        
        // Prepend IV
        ms.Write(aes.IV, 0, aes.IV.Length);
        
        using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        using (var sw = new StreamWriter(cs))
        {
            sw.Write(plainText);
        }
        
        return Convert.ToBase64String(ms.ToArray());
    }
    
    public string Decrypt(string cipherText)
    {
        var data = Convert.FromBase64String(cipherText);
        
        using var aes = Aes.Create();
        aes.Key = _key;
        
        // Extract IV (first 16 bytes)
        var iv = data[..16];
        var cipher = data[16..];
        aes.IV = iv;
        
        using var decryptor = aes.CreateDecryptor();
        using var ms = new MemoryStream(cipher);
        using var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read);
        using var sr = new StreamReader(cs);
        
        return sr.ReadToEnd();
    }
    
    // Hash (one-way)
    public static string Hash(string input)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(bytes).ToLower();
    }
    
    // HMAC for message authentication
    public string CreateHmac(string message, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(message));
        return Convert.ToBase64String(hash);
    }
    
    public bool VerifyHmac(string message, string secret, string expectedHmac)
    {
        var actualHmac = CreateHmac(message, secret);
        // Constant-time comparison to prevent timing attacks
        return CryptographicOperations.FixedTimeEquals(
            Convert.FromBase64String(actualHmac),
            Convert.FromBase64String(expectedHmac));
    }
}
```

---

## Step 263: Secure Storage

```csharp
// ============================================
// Encrypted Local Storage
// ============================================

public class SecureDataService
{
    private readonly EncryptionService _encryption;
    private readonly string _storagePath;
    
    public SecureDataService()
    {
        // Get device-unique passphrase
        var deviceId = DeviceInfo.Current.Idiom.ToString() + 
                       DeviceInfo.Current.Model +
                       "SecretSalt42";
        
        _encryption = new EncryptionService(deviceId);
        _storagePath = Path.Combine(FileSystem.AppDataDirectory, "secure.dat");
    }
    
    private Dictionary<string, string> LoadData()
    {
        if (!File.Exists(_storagePath)) return new();
        
        try
        {
            var encrypted = File.ReadAllText(_storagePath);
            var json = _encryption.Decrypt(encrypted);
            return System.Text.Json.JsonSerializer.Deserialize<Dictionary<string, string>>(json)
                   ?? new();
        }
        catch
        {
            return new();
        }
    }
    
    private void SaveData(Dictionary<string, string> data)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(data);
        var encrypted = _encryption.Encrypt(json);
        File.WriteAllText(_storagePath, encrypted);
    }
    
    public void Set(string key, string value)
    {
        var data = LoadData();
        data[key] = value;
        SaveData(data);
    }
    
    public string? Get(string key)
    {
        var data = LoadData();
        return data.GetValueOrDefault(key);
    }
    
    public void Remove(string key)
    {
        var data = LoadData();
        data.Remove(key);
        SaveData(data);
    }
    
    public void Clear()
    {
        if (File.Exists(_storagePath))
            File.Delete(_storagePath);
    }
}

// ============================================
// Token Manager
// ============================================

public class TokenManager
{
    private const string AccessTokenKey = "access_token";
    private const string RefreshTokenKey = "refresh_token";
    private const string TokenExpiryKey = "token_expiry";
    
    public async Task SaveTokensAsync(string accessToken, string refreshToken, 
        DateTimeOffset expiry)
    {
        await SecureStorage.Default.SetAsync(AccessTokenKey, accessToken);
        await SecureStorage.Default.SetAsync(RefreshTokenKey, refreshToken);
        await SecureStorage.Default.SetAsync(TokenExpiryKey, expiry.ToString("O"));
    }
    
    public async Task<string?> GetValidAccessTokenAsync()
    {
        var expiry = await SecureStorage.Default.GetAsync(TokenExpiryKey);
        
        if (expiry != null && DateTimeOffset.TryParse(expiry, out var expiryDate))
        {
            if (expiryDate > DateTimeOffset.UtcNow.AddMinutes(5))
                return await SecureStorage.Default.GetAsync(AccessTokenKey);
        }
        
        // Token expired or expiring soon - try refresh
        return await TryRefreshTokenAsync();
    }
    
    private async Task<string?> TryRefreshTokenAsync()
    {
        var refreshToken = await SecureStorage.Default.GetAsync(RefreshTokenKey);
        if (string.IsNullOrEmpty(refreshToken)) return null;
        
        try
        {
            // Call auth API to refresh
            // var response = await _authApi.RefreshAsync(refreshToken);
            // await SaveTokensAsync(response.AccessToken, response.RefreshToken, response.Expiry);
            // return response.AccessToken;
            return null; // placeholder
        }
        catch
        {
            ClearTokens();
            return null;
        }
    }
    
    public void ClearTokens()
    {
        SecureStorage.Default.Remove(AccessTokenKey);
        SecureStorage.Default.Remove(RefreshTokenKey);
        SecureStorage.Default.Remove(TokenExpiryKey);
    }
    
    public async Task<bool> IsAuthenticatedAsync()
        => await GetValidAccessTokenAsync() != null;
}
```

---

## Step 264: Certificate Pinning

```csharp
// ============================================
// Certificate Pinning
// ============================================

public class PinnedCertificateHandler : HttpClientHandler
{
    // SHA-256 fingerprint of the server certificate
    private static readonly string[] AllowedFingerprints =
    {
        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        // Add backup certificate fingerprint
    };
    
    public PinnedCertificateHandler()
    {
        ServerCertificateCustomValidationCallback = ValidateCertificate;
    }
    
    private bool ValidateCertificate(
        HttpRequestMessage request,
        System.Security.Cryptography.X509Certificates.X509Certificate2? cert,
        System.Security.Cryptography.X509Certificates.X509Chain? chain,
        System.Net.Security.SslPolicyErrors errors)
    {
        if (cert == null) return false;
        
        // Get certificate fingerprint
        var fingerprint = cert.GetCertHashString(HashAlgorithmName.SHA256);
        var b64Fingerprint = "sha256/" + Convert.ToBase64String(
            Convert.FromHexString(fingerprint));
        
        // Check against pinned certificates
        return AllowedFingerprints.Contains(b64Fingerprint);
    }
}

// Usage:
// var handler = new PinnedCertificateHandler();
// var client = new HttpClient(handler);
```

---

## Step 265: Input Validation & Sanitization

```csharp
// ============================================
// Input Validation
// ============================================

public static class InputSanitizer
{
    // Remove potential SQL injection
    public static string SanitizeForDatabase(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;
        
        // Never build SQL with user input - use parameterized queries
        // This is just an example; always use parameterized queries with SQLite
        return input.Replace("'", "''");
    }
    
    // Remove HTML tags
    public static string StripHtml(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;
        return System.Text.RegularExpressions.Regex.Replace(input, "<.*?>", string.Empty);
    }
    
    // Validate and sanitize URL
    public static bool IsValidUrl(string url, out string sanitized)
    {
        sanitized = url?.Trim() ?? string.Empty;
        
        if (!Uri.TryCreate(sanitized, UriKind.Absolute, out var uri))
            return false;
        
        // Only allow http/https
        if (uri.Scheme != "http" && uri.Scheme != "https")
            return false;
        
        sanitized = uri.ToString();
        return true;
    }
    
    // Email validation
    public static bool IsValidEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email)) return false;
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email.Trim();
        }
        catch
        {
            return false;
        }
    }
    
    // Phone validation (Thai format)
    public static bool IsValidThaiPhone(string phone)
    {
        var clean = System.Text.RegularExpressions.Regex.Replace(phone, @"[\s\-\(\)]", "");
        return System.Text.RegularExpressions.Regex.IsMatch(clean, @"^(\+66|0)[0-9]{8,9}$");
    }
    
    // Sanitize file path (prevent directory traversal)
    public static string SanitizeFilePath(string filename)
    {
        // Remove directory separators
        filename = Path.GetFileName(filename);
        
        // Remove special characters
        var invalid = Path.GetInvalidFileNameChars();
        foreach (var c in invalid)
            filename = filename.Replace(c.ToString(), "_");
        
        // Limit length
        if (filename.Length > 255)
            filename = filename[..255];
        
        return filename;
    }
}
```

---

## Step 266: JWT Token Handling

```csharp
// ============================================
// JWT Token Parsing (without library)
// ============================================

public class JwtParser
{
    public static JwtPayload? ParsePayload(string token)
    {
        if (string.IsNullOrEmpty(token)) return null;
        
        var parts = token.Split('.');
        if (parts.Length != 3) return null;
        
        try
        {
            var payloadBase64 = parts[1];
            
            // Pad base64
            var padding = 4 - payloadBase64.Length % 4;
            if (padding != 4)
                payloadBase64 += new string('=', padding);
            
            payloadBase64 = payloadBase64.Replace('-', '+').Replace('_', '/');
            
            var json = System.Text.Encoding.UTF8.GetString(
                Convert.FromBase64String(payloadBase64));
            
            return System.Text.Json.JsonSerializer.Deserialize<JwtPayload>(json,
                new System.Text.Json.JsonSerializerOptions 
                { 
                    PropertyNameCaseInsensitive = true 
                });
        }
        catch
        {
            return null;
        }
    }
    
    public static bool IsExpired(string token)
    {
        var payload = ParsePayload(token);
        if (payload == null) return true;
        
        var expiry = DateTimeOffset.FromUnixTimeSeconds(payload.Exp);
        return expiry <= DateTimeOffset.UtcNow;
    }
    
    public static TimeSpan? GetTimeToExpiry(string token)
    {
        var payload = ParsePayload(token);
        if (payload == null) return null;
        
        var expiry = DateTimeOffset.FromUnixTimeSeconds(payload.Exp);
        var remaining = expiry - DateTimeOffset.UtcNow;
        return remaining > TimeSpan.Zero ? remaining : TimeSpan.Zero;
    }
}

public class JwtPayload
{
    [System.Text.Json.Serialization.JsonPropertyName("sub")]
    public string Sub { get; set; } = string.Empty;
    
    [System.Text.Json.Serialization.JsonPropertyName("email")]
    public string? Email { get; set; }
    
    [System.Text.Json.Serialization.JsonPropertyName("name")]
    public string? Name { get; set; }
    
    [System.Text.Json.Serialization.JsonPropertyName("role")]
    public string? Role { get; set; }
    
    [System.Text.Json.Serialization.JsonPropertyName("exp")]
    public long Exp { get; set; }
    
    [System.Text.Json.Serialization.JsonPropertyName("iat")]
    public long Iat { get; set; }
    
    [System.Text.Json.Serialization.JsonPropertyName("nbf")]
    public long Nbf { get; set; }
}
```

---

## Step 267: App Security Checklist

```csharp
// ============================================
// Security Checklist Implementation
// ============================================

public class SecurityAuditService
{
    // 1. Root/Jailbreak Detection
    public static bool IsDeviceCompromised()
    {
        if (DeviceInfo.Current.DeviceType == DeviceType.Virtual)
            return false; // Emulator - ok for development
        
        // Check for common jailbreak/root files
        if (DeviceInfo.Current.Platform == DevicePlatform.iOS)
        {
            var suspiciousPaths = new[]
            {
                "/Applications/Cydia.app",
                "/Library/MobileSubstrate/MobileSubstrate.dylib",
                "/bin/bash",
                "/usr/sbin/sshd",
                "/etc/apt"
            };
            return suspiciousPaths.Any(File.Exists);
        }
        
        if (DeviceInfo.Current.Platform == DevicePlatform.Android)
        {
            var suspiciousPaths = new[]
            {
                "/system/bin/su",
                "/system/xbin/su",
                "/sbin/su",
                "/data/local/xbin/su",
                "/data/local/bin/su"
            };
            return suspiciousPaths.Any(File.Exists);
        }
        
        return false;
    }
    
    // 2. Screen Recording Detection
    public static bool IsScreenBeingRecorded()
    {
        // iOS specific
        if (DeviceInfo.Current.Platform == DevicePlatform.iOS)
        {
            // Check UIScreen.main.isCaptured via platform-specific code
        }
        return false;
    }
    
    // 3. Prevent Screenshot (Android)
    public static void PreventScreenshot(Window window)
    {
        // Android: WindowManager.LayoutParams.FLAG_SECURE
        // Handled via platform-specific code
    }
    
    // 4. Clear data on logout
    public static async Task SecureLogoutAsync()
    {
        // Clear all secure storage
        SecureStorage.Default.RemoveAll();
        
        // Clear preferences
        Preferences.Default.Clear();
        
        // Clear cache
        var cacheDir = FileSystem.CacheDirectory;
        if (Directory.Exists(cacheDir))
        {
            foreach (var file in Directory.GetFiles(cacheDir))
                File.Delete(file);
        }
        
        // Navigate to login
        await Shell.Current.GoToAsync("//login");
    }
    
    // 5. App integrity check
    public static bool VerifyAppIntegrity()
    {
        // Compare app signature with known good signature
        // Platform-specific implementation needed
        return true;
    }
}
```

---

## Step 268: OAuth 2.0 Implementation

```csharp
// ============================================
// OAuth 2.0 + PKCE
// ============================================

public class OAuthService
{
    private readonly HttpClient _http;
    
    // OAuth configuration
    private const string ClientId = "your-client-id";
    private const string AuthorizationEndpoint = "https://auth.example.com/authorize";
    private const string TokenEndpoint = "https://auth.example.com/token";
    private const string RedirectUri = "myapp://oauth/callback";
    private const string Scopes = "openid profile email";
    
    public OAuthService(HttpClient http) => _http = http;
    
    // Generate PKCE code verifier and challenge
    private static (string Verifier, string Challenge) GeneratePkce()
    {
        using var rng = RandomNumberGenerator.Create();
        var bytes = new byte[32];
        rng.GetBytes(bytes);
        
        var verifier = Convert.ToBase64String(bytes)
            .TrimEnd('=').Replace('+', '-').Replace('/', '_');
        
        var hash = SHA256.HashData(System.Text.Encoding.ASCII.GetBytes(verifier));
        var challenge = Convert.ToBase64String(hash)
            .TrimEnd('=').Replace('+', '-').Replace('/', '_');
        
        return (verifier, challenge);
    }
    
    public (string AuthUrl, string CodeVerifier) GetAuthorizationUrl()
    {
        var (verifier, challenge) = GeneratePkce();
        var state = Guid.NewGuid().ToString("N");
        
        var query = new Dictionary<string, string>
        {
            ["response_type"] = "code",
            ["client_id"] = ClientId,
            ["redirect_uri"] = RedirectUri,
            ["scope"] = Scopes,
            ["state"] = state,
            ["code_challenge"] = challenge,
            ["code_challenge_method"] = "S256"
        };
        
        var queryString = string.Join("&", query.Select(
            kv => $"{Uri.EscapeDataString(kv.Key)}={Uri.EscapeDataString(kv.Value)}"));
        
        return ($"{AuthorizationEndpoint}?{queryString}", verifier);
    }
    
    public async Task<OAuthTokenResponse?> ExchangeCodeAsync(
        string code, string codeVerifier)
    {
        var content = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "authorization_code",
            ["code"] = code,
            ["redirect_uri"] = RedirectUri,
            ["client_id"] = ClientId,
            ["code_verifier"] = codeVerifier
        });
        
        var response = await _http.PostAsync(TokenEndpoint, content);
        if (!response.IsSuccessStatusCode) return null;
        
        return await response.Content.ReadFromJsonAsync<OAuthTokenResponse>();
    }
    
    public async Task<OAuthTokenResponse?> RefreshTokenAsync(string refreshToken)
    {
        var content = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "refresh_token",
            ["refresh_token"] = refreshToken,
            ["client_id"] = ClientId
        });
        
        var response = await _http.PostAsync(TokenEndpoint, content);
        if (!response.IsSuccessStatusCode) return null;
        
        return await response.Content.ReadFromJsonAsync<OAuthTokenResponse>();
    }
    
    // Open browser for OAuth
    public async Task<string?> AuthenticateAsync()
    {
        var (authUrl, verifier) = GetAuthorizationUrl();
        
        // Open browser
        var result = await WebAuthenticator.Default.AuthenticateAsync(
            new WebAuthenticatorOptions
            {
                Url = new Uri(authUrl),
                CallbackUrl = new Uri(RedirectUri)
            });
        
        if (result.Properties.TryGetValue("code", out var code))
        {
            var tokens = await ExchangeCodeAsync(code, verifier);
            return tokens?.AccessToken;
        }
        
        return null;
    }
}

public record OAuthTokenResponse(
    [property: System.Text.Json.Serialization.JsonPropertyName("access_token")]
    string AccessToken,
    [property: System.Text.Json.Serialization.JsonPropertyName("refresh_token")]
    string RefreshToken,
    [property: System.Text.Json.Serialization.JsonPropertyName("expires_in")]
    int ExpiresIn,
    [property: System.Text.Json.Serialization.JsonPropertyName("token_type")]
    string TokenType
);
```

---

## Step 269: Role-Based Authorization

```csharp
// ============================================
// Role-Based Access Control (RBAC)
// ============================================

public enum UserRole { Guest, User, Moderator, Admin, SuperAdmin }

public interface ICurrentUserService
{
    int? UserId { get; }
    string? Email { get; }
    UserRole Role { get; }
    bool IsAuthenticated { get; }
    bool IsInRole(UserRole minimumRole);
}

public class CurrentUserService : ICurrentUserService
{
    private readonly TokenManager _tokenManager;
    private JwtPayload? _cachedPayload;
    
    public CurrentUserService(TokenManager tokenManager) => _tokenManager = tokenManager;
    
    private async Task<JwtPayload?> GetPayloadAsync()
    {
        var token = await _tokenManager.GetValidAccessTokenAsync();
        if (token == null) return null;
        return JwtParser.ParsePayload(token);
    }
    
    public int? UserId => int.TryParse(_cachedPayload?.Sub, out var id) ? id : null;
    public string? Email => _cachedPayload?.Email;
    public UserRole Role => Enum.TryParse<UserRole>(_cachedPayload?.Role, out var role) 
        ? role : UserRole.Guest;
    public bool IsAuthenticated => _cachedPayload != null;
    
    public bool IsInRole(UserRole minimumRole) => Role >= minimumRole;
}

// ============================================
// Authorization Guard
// ============================================

public class AuthorizationService
{
    private readonly ICurrentUserService _currentUser;
    private readonly INavService _nav;
    
    public AuthorizationService(ICurrentUserService currentUser, INavService nav)
    {
        _currentUser = currentUser;
        _nav = nav;
    }
    
    public async Task<bool> RequireAuthAsync(UserRole minimumRole = UserRole.User)
    {
        if (!_currentUser.IsAuthenticated)
        {
            await _nav.GoToRootAsync("login");
            return false;
        }
        
        if (!_currentUser.IsInRole(minimumRole))
        {
            await _nav.ShowAlertAsync("ไม่มีสิทธิ์", 
                $"คุณต้องมีสิทธิ์ระดับ {minimumRole} ขึ้นไป");
            return false;
        }
        
        return true;
    }
    
    public bool CanEdit(int ownerId) 
        => _currentUser.UserId == ownerId || _currentUser.IsInRole(UserRole.Admin);
    
    public bool CanDelete(int ownerId)
        => _currentUser.UserId == ownerId || _currentUser.IsInRole(UserRole.Admin);
    
    public bool CanModerate() => _currentUser.IsInRole(UserRole.Moderator);
    public bool CanAdmin() => _currentUser.IsInRole(UserRole.Admin);
}

// Usage in ViewModel
public partial class AdminViewModel : ObservableObject
{
    private readonly AuthorizationService _auth;
    
    public AdminViewModel(AuthorizationService auth) => _auth = auth;
    
    [RelayCommand]
    private async Task DeleteUserAsync(int userId)
    {
        if (!await _auth.RequireAuthAsync(UserRole.Admin)) return;
        
        // Proceed with admin action
    }
    
    [RelayCommand]
    private async Task LoadAdminDataAsync()
    {
        if (!_auth.CanAdmin())
        {
            await Shell.Current.GoToAsync("..");
            return;
        }
        
        // Load admin data
    }
}
```

---

## Step 270: Security Testing

```csharp
// ============================================
// Security Testing
// ============================================

public class SecurityTests
{
    private readonly EncryptionService _encryption;
    
    public SecurityTests()
    {
        _encryption = new EncryptionService("TestPassphrase123!");
    }
    
    [Fact]
    public void Encrypt_ThenDecrypt_ReturnsOriginal()
    {
        var original = "Sensitive data: password123!";
        var encrypted = _encryption.Encrypt(original);
        var decrypted = _encryption.Decrypt(encrypted);
        
        decrypted.Should().Be(original);
        encrypted.Should().NotBe(original);
    }
    
    [Fact]
    public void EncryptSameDataTwice_ProducesDifferentCiphertext()
    {
        var data = "Same data";
        var cipher1 = _encryption.Encrypt(data);
        var cipher2 = _encryption.Encrypt(data);
        
        // Should be different due to random IV
        cipher1.Should().NotBe(cipher2);
        
        // But both should decrypt to original
        _encryption.Decrypt(cipher1).Should().Be(data);
        _encryption.Decrypt(cipher2).Should().Be(data);
    }
    
    [Theory]
    [InlineData("user@example.com", true)]
    [InlineData("invalid-email", false)]
    [InlineData("user@", false)]
    [InlineData("@domain.com", false)]
    [InlineData("", false)]
    public void ValidateEmail(string email, bool expected)
    {
        InputSanitizer.IsValidEmail(email).Should().Be(expected);
    }
    
    [Theory]
    [InlineData("0812345678", true)]
    [InlineData("+66812345678", true)]
    [InlineData("081234567", false)] // Too short
    [InlineData("081234567890", false)] // Too long
    [InlineData("1812345678", false)] // Wrong prefix
    public void ValidateThaiPhone(string phone, bool expected)
    {
        InputSanitizer.IsValidThaiPhone(phone).Should().Be(expected);
    }
    
    [Fact]
    public void SanitizeFilePath_PreventDirectoryTraversal()
    {
        var malicious = "../../../etc/passwd";
        var safe = InputSanitizer.SanitizeFilePath(malicious);
        
        safe.Should().NotContain("..");
        safe.Should().NotContain("/");
        safe.Should().NotContain("\\");
    }
    
    [Fact]
    public void JwtParser_ValidToken_ExtractsPayload()
    {
        // Create a test JWT (header.payload.signature)
        var header = Convert.ToBase64String(
            System.Text.Encoding.UTF8.GetBytes("{\"alg\":\"HS256\",\"typ\":\"JWT\"}"))
            .TrimEnd('=').Replace('+', '-').Replace('/', '_');
        
        var payloadJson = "{\"sub\":\"123\",\"email\":\"test@test.com\",\"exp\":9999999999}";
        var payload = Convert.ToBase64String(
            System.Text.Encoding.UTF8.GetBytes(payloadJson))
            .TrimEnd('=').Replace('+', '-').Replace('/', '_');
        
        var token = $"{header}.{payload}.signature";
        
        var parsed = JwtParser.ParsePayload(token);
        
        parsed.Should().NotBeNull();
        parsed!.Sub.Should().Be("123");
        parsed.Email.Should().Be("test@test.com");
    }
}
```

---

## สรุป Part 27

ใน Part 27 เราได้เรียนรู้:

1. **Security Principles** - หลักการ security ของ mobile app
2. **AES Encryption** - Encrypt/Decrypt ข้อมูล
3. **HMAC** - Message authentication
4. **Secure Storage** - Encrypted local storage
5. **Token Manager** - JWT token lifecycle
6. **Certificate Pinning** - ป้องกัน MITM attacks
7. **Input Validation** - Sanitize user input
8. **JWT Parsing** - Decode token payload
9. **OAuth 2.0 + PKCE** - Modern authentication flow
10. **RBAC** - Role-based authorization
11. **Security Testing** - Test encryption, validation

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 27 | Steps 261-270*
