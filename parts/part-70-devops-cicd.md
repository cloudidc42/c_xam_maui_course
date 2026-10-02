# Part 70: DevOps & CI/CD Pipeline
## Steps 691-700: Build Automation, App Distribution, Feature Flags, Canary Releases

---

## Step 691: GitHub Actions Full Pipeline

```yaml
# .github/workflows/main.yml
name: Mobile CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to environment'
        required: true
        default: 'staging'
        type: choice
        options: [ staging, production ]

env:
  DOTNET_VERSION: '8.0.x'
  APP_ID: com.myfooddelivery.app

jobs:
  # ─────────── Lint & Build ───────────
  build:
    name: Build & Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Restore
        run: dotnet restore src/MyFoodDelivery.sln
      
      - name: Build (Release)
        run: dotnet build src/MyFoodDelivery.sln --configuration Release --no-restore
      
      - name: Run Tests
        run: |
          dotnet test src --configuration Release --no-build \
            --collect:"XPlat Code Coverage" \
            --results-directory TestResults \
            --logger "trx"
      
      - name: Publish Coverage
        uses: codecov/codecov-action@v4
        with:
          directory: TestResults
          fail_ci_if_error: true
  
  # ─────────── Android Build ───────────
  android:
    name: Android Build
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Install MAUI Workload
        run: dotnet workload install maui-android
      
      - name: Decode Keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > release.keystore
      
      - name: Build Android APK
        run: |
          dotnet publish src/MyFoodDelivery.MAUI \
            -f net8.0-android \
            -c Release \
            -p:AndroidKeyStore=True \
            -p:AndroidSigningKeyStore=${{ github.workspace }}/release.keystore \
            -p:AndroidSigningKeyAlias=${{ secrets.ANDROID_KEY_ALIAS }} \
            -p:AndroidSigningKeyPass=${{ secrets.ANDROID_KEY_PASS }} \
            -p:AndroidSigningStorePass=${{ secrets.ANDROID_STORE_PASS }} \
            -p:ApplicationVersion=${{ github.run_number }} \
            -o artifacts/android
      
      - name: Build Android AAB
        run: |
          dotnet publish src/MyFoodDelivery.MAUI \
            -f net8.0-android \
            -c Release \
            -p:AndroidPackageFormat=aab \
            -p:AndroidKeyStore=True \
            -p:AndroidSigningKeyStore=${{ github.workspace }}/release.keystore \
            -p:AndroidSigningKeyAlias=${{ secrets.ANDROID_KEY_ALIAS }} \
            -p:AndroidSigningKeyPass=${{ secrets.ANDROID_KEY_PASS }} \
            -p:AndroidSigningStorePass=${{ secrets.ANDROID_STORE_PASS }} \
            -o artifacts/android-aab
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: android-artifacts
          path: artifacts/android*/
          retention-days: 30
  
  # ─────────── iOS Build ───────────
  ios:
    name: iOS Build
    runs-on: macos-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Install MAUI Workload
        run: dotnet workload install maui-ios
      
      - name: Import Certificates
        uses: Apple-Actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_CERT_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_CERT_PASSWORD }}
      
      - name: Download Provisioning Profile
        uses: Apple-Actions/download-provisioning-profiles@v1
        with:
          bundle-id: ${{ env.APP_ID }}
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
      
      - name: Build iOS IPA
        run: |
          dotnet publish src/MyFoodDelivery.MAUI \
            -f net8.0-ios \
            -c Release \
            -p:ArchiveOnBuild=true \
            -p:RuntimeIdentifier=ios-arm64 \
            -o artifacts/ios
      
      - uses: actions/upload-artifact@v4
        with:
          name: ios-artifacts
          path: artifacts/ios/
  
  # ─────────── Deploy to TestFlight ───────────
  deploy-testflight:
    name: Deploy to TestFlight
    runs-on: macos-latest
    needs: ios
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ios-artifacts
          path: artifacts/ios
      
      - name: Upload to TestFlight
        uses: Apple-Actions/upload-testflight-build@v1
        with:
          app-path: artifacts/ios/MyFoodDelivery.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
  
  # ─────────── Deploy to Play Store ───────────
  deploy-playstore:
    name: Deploy to Play Store (Internal)
    runs-on: ubuntu-latest
    needs: android
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: android-artifacts
          path: artifacts
      
      - name: Upload to Play Store (Internal Track)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_JSON }}
          packageName: ${{ env.APP_ID }}
          releaseFiles: artifacts/android-aab/**/*.aab
          track: internal
          status: completed
          changesNotSentForReview: false
```

---

## Step 692: App Version Management

```csharp
// ============================================
// Version Management
// ============================================

public class AppVersionService
{
    private readonly IApiGateway _api;
    
    public AppVersionService(IApiGateway api) => _api = api;
    
    public async Task<VersionCheckResult> CheckForUpdateAsync()
    {
        var current = AppInfo.Current.Version;
        var platform = DeviceInfo.Current.Platform.ToString().ToLower();
        
        var response = await _api.GetAsync<VersionResponse>(
            $"/app/version?platform={platform}");
        
        var latest = Version.Parse(response.LatestVersion);
        var minimum = Version.Parse(response.MinimumVersion);
        
        return new VersionCheckResult(
            CurrentVersion: current,
            LatestVersion: latest,
            MinimumVersion: minimum,
            IsUpdateAvailable: latest > current,
            IsForceUpdate: current < minimum,
            ReleaseNotes: response.ReleaseNotes,
            StoreUrl: response.StoreUrl);
    }
    
    public async Task ShowUpdateDialogIfNeededAsync()
    {
        var result = await CheckForUpdateAsync();
        if (!result.IsUpdateAvailable) return;
        
        if (result.IsForceUpdate)
        {
            // Cannot dismiss
            await Shell.Current.DisplayAlert(
                "กรุณาอัปเดต",
                "แอปเวอร์ชันนี้หมดอายุแล้ว กรุณาอัปเดตเพื่อใช้งานต่อ",
                "อัปเดตเลย");
            
            await Launcher.Default.OpenAsync(new Uri(result.StoreUrl));
            return;
        }
        
        var update = await Shell.Current.DisplayAlert(
            "มีอัปเดตใหม่",
            $"เวอร์ชัน {result.LatestVersion} พร้อมแล้ว\n{result.ReleaseNotes}",
            "อัปเดต", "ทีหลัง");
        
        if (update)
            await Launcher.Default.OpenAsync(new Uri(result.StoreUrl));
    }
}

public record VersionCheckResult(
    Version CurrentVersion, Version LatestVersion, Version MinimumVersion,
    bool IsUpdateAvailable, bool IsForceUpdate,
    string ReleaseNotes, string StoreUrl);

public record VersionResponse(
    string LatestVersion, string MinimumVersion,
    string ReleaseNotes, string StoreUrl);
```

---

## Step 693: Feature Flags with Remote Config

```csharp
// ============================================
// Remote Feature Flags
// ============================================

public class RemoteFeatureFlagService : IFeatureFlagService
{
    private readonly IApiGateway _api;
    private readonly SQLiteConnection _db;
    private readonly IStructuredLogger _logger;
    private Dictionary<string, bool> _cache = new();
    private DateTime _cacheExpiry;
    
    public RemoteFeatureFlagService(IApiGateway api, SQLiteConnection db, IStructuredLogger logger)
    {
        _api = api;
        _db = db;
        _logger = logger;
        _db.CreateTable<CachedFlag>();
    }
    
    public async Task<bool> IsEnabledAsync(FeatureFlag flag)
    {
        var key = flag.ToString();
        
        // Check in-memory cache
        if (_cacheExpiry > DateTime.UtcNow && _cache.TryGetValue(key, out var cached))
            return cached;
        
        // Check SQLite cache
        var dbCached = _db.Table<CachedFlag>()
            .FirstOrDefault(f => f.Key == key && f.ExpiresAt > DateTime.UtcNow);
        if (dbCached != null)
        {
            _cache[key] = dbCached.Value;
            return dbCached.Value;
        }
        
        // Fetch from server
        try
        {
            var flags = await _api.GetAsync<Dictionary<string, bool>>("/feature-flags");
            
            // Cache for 15 minutes
            var expiry = DateTime.UtcNow.AddMinutes(15);
            _cache = flags;
            _cacheExpiry = expiry;
            
            foreach (var (k, v) in flags)
                _db.InsertOrReplace(new CachedFlag { Key = k, Value = v, ExpiresAt = expiry });
            
            return flags.TryGetValue(key, out var value) && value;
        }
        catch
        {
            // Fallback to defaults
            return GetDefault(flag);
        }
    }
    
    private bool GetDefault(FeatureFlag flag) => flag switch
    {
        FeatureFlag.NewCheckoutFlow => false,
        FeatureFlag.PremiumSubscription => true,
        FeatureFlag.RealTimeTracking => true,
        _ => false
    };
}

public class CachedFlag
{
    [PrimaryKey] public string Key { get; set; } = string.Empty;
    public bool Value { get; set; }
    public DateTime ExpiresAt { get; set; }
}
```

---

## Step 694: Canary Release

```csharp
// ============================================
// Canary Release / Gradual Rollout
// ============================================

public class CanaryReleaseService
{
    private readonly IApiGateway _api;
    private readonly SQLiteConnection _db;
    
    public CanaryReleaseService(IApiGateway api, SQLiteConnection db)
    {
        _api = api;
        _db = db;
        _db.CreateTable<UserCanaryGroup>();
    }
    
    public async Task<bool> IsInCanaryGroupAsync(string feature, int userId)
    {
        // Check local assignment
        var local = _db.Table<UserCanaryGroup>()
            .FirstOrDefault(g => g.Feature == feature && g.UserId == userId);
        
        if (local != null) return local.IsInCanary;
        
        // Get rollout percentage from server
        var config = await _api.GetAsync<CanaryConfig>($"/canary/{feature}");
        
        // Deterministic assignment: consistent bucket based on userId
        var bucket = Math.Abs(HashCode.Combine(feature, userId)) % 100;
        var isInCanary = bucket < config.RolloutPercentage;
        
        // Cache the assignment
        _db.Insert(new UserCanaryGroup
        {
            Feature = feature, UserId = userId,
            IsInCanary = isInCanary, AssignedAt = DateTime.UtcNow
        });
        
        return isInCanary;
    }
    
    // Usage: Render new vs old UI
    public async Task<T> SwitchAsync<T>(string feature, int userId,
        Func<Task<T>> newBehavior, Func<Task<T>> oldBehavior)
    {
        var isCanary = await IsInCanaryGroupAsync(feature, userId);
        return isCanary ? await newBehavior() : await oldBehavior();
    }
}

public record CanaryConfig(string Feature, int RolloutPercentage, bool IsActive);

public class UserCanaryGroup
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Feature { get; set; } = string.Empty;
    public int UserId { get; set; }
    public bool IsInCanary { get; set; }
    public DateTime AssignedAt { get; set; }
}
```

---

## Step 695: Environment Configuration

```csharp
// ============================================
// Multi-Environment Configuration
// ============================================

public enum AppEnvironment { Development, Staging, Production }

public class AppConfiguration
{
    public string ApiBaseUrl { get; init; } = string.Empty;
    public string SignalRUrl { get; init; } = string.Empty;
    public string StripePublishableKey { get; init; } = string.Empty;
    public string GoogleMapsKey { get; init; } = string.Empty;
    public string FirebaseProjectId { get; init; } = string.Empty;
    public bool EnableDebugTools { get; init; }
    public bool EnableAnalytics { get; init; }
    public int CacheDurationMinutes { get; init; } = 15;
    public AppEnvironment Environment { get; init; }
}

public static class AppConfigurationFactory
{
    public static AppConfiguration Create()
    {
#if DEBUG
        return Development();
#elif STAGING
        return Staging();
#else
        return Production();
#endif
    }
    
    private static AppConfiguration Development() => new()
    {
        ApiBaseUrl = "http://10.0.2.2:5000/api",
        SignalRUrl = "http://10.0.2.2:5000/hubs",
        StripePublishableKey = "pk_test_...",
        GoogleMapsKey = "dev_key",
        FirebaseProjectId = "myfooddelivery-dev",
        EnableDebugTools = true,
        EnableAnalytics = false,
        Environment = AppEnvironment.Development
    };
    
    private static AppConfiguration Staging() => new()
    {
        ApiBaseUrl = "https://api-staging.myfooddelivery.com/api",
        SignalRUrl = "https://api-staging.myfooddelivery.com/hubs",
        StripePublishableKey = "pk_test_...",
        GoogleMapsKey = "staging_key",
        FirebaseProjectId = "myfooddelivery-staging",
        EnableDebugTools = true,
        EnableAnalytics = true,
        Environment = AppEnvironment.Staging
    };
    
    private static AppConfiguration Production() => new()
    {
        ApiBaseUrl = "https://api.myfooddelivery.com/api",
        SignalRUrl = "https://api.myfooddelivery.com/hubs",
        StripePublishableKey = "pk_live_...",
        GoogleMapsKey = "prod_key",
        FirebaseProjectId = "myfooddelivery-prod",
        EnableDebugTools = false,
        EnableAnalytics = true,
        CacheDurationMinutes = 30,
        Environment = AppEnvironment.Production
    };
}
```

---

## Step 696: Automated Screenshot Testing

```csharp
// ============================================
// Automated App Store Screenshots
// ============================================

// Run UI tests that capture screenshots for App Store
[TestFixture]
public class AppStoreScreenshots : UITestBase
{
    private readonly string _outputDir = "Screenshots/AppStore";
    
    [OneTimeSetUp]
    public void Setup() => Directory.CreateDirectory(_outputDir);
    
    [Test, Order(1)]
    public async Task Capture_HomeScreen()
    {
        await WaitForElement("home_page");
        await Task.Delay(1000); // Let animations finish
        await CaptureScreenshot("01-home");
    }
    
    [Test, Order(2)]
    public async Task Capture_RestaurantDetail()
    {
        await WaitForElement("restaurant_list");
        FindById("restaurant_list")
            .FindElements(By.ClassName("android.widget.FrameLayout"))[0]
            .Click();
        
        await WaitForElement("restaurant_detail");
        await Task.Delay(500);
        await CaptureScreenshot("02-restaurant");
    }
    
    [Test, Order(3)]
    public async Task Capture_Cart()
    {
        // Add item to cart first
        await WaitForElement("add_to_cart_button");
        FindById("add_to_cart_button").Click();
        FindById("cart_tab").Click();
        
        await WaitForElement("cart_page");
        await CaptureScreenshot("03-cart");
    }
    
    [Test, Order(4)]
    public async Task Capture_OrderTracking()
    {
        // Navigate to order history → tracking
        FindById("profile_tab").Click();
        await WaitForElement("order_history_button");
        FindById("order_history_button").Click();
        
        await WaitForElement("order_list");
        FindById("order_list")
            .FindElements(By.ClassName("android.widget.FrameLayout"))[0]
            .Click();
        
        await WaitForElement("tracking_page");
        await Task.Delay(1000);
        await CaptureScreenshot("04-tracking");
    }
    
    private async Task CaptureScreenshot(string name)
    {
        var screenshot = ((ITakesScreenshot)Driver).GetScreenshot();
        var path = Path.Combine(_outputDir, $"{name}.png");
        await File.WriteAllBytesAsync(path, screenshot.AsByteArray);
        Console.WriteLine($"Screenshot saved: {path}");
    }
}
```

---

## Step 697: Crash-Free Rate Monitoring

```csharp
// ============================================
// Crash-Free Rate & Release Health
// ============================================

public class ReleaseHealthMonitor
{
    private readonly SQLiteConnection _db;
    private readonly IApiGateway _api;
    private int _sessionId;
    
    public ReleaseHealthMonitor(SQLiteConnection db, IApiGateway api)
    {
        _db = db;
        _api = api;
        _db.CreateTable<SessionRecord>();
        _db.CreateTable<CrashRecord>();
    }
    
    public void StartSession()
    {
        var session = new SessionRecord
        {
            AppVersion = AppInfo.Current.VersionString,
            Platform = DeviceInfo.Current.Platform.ToString(),
            StartedAt = DateTime.UtcNow,
            IsCrashFree = true
        };
        _db.Insert(session);
        _sessionId = session.Id;
    }
    
    public void RecordCrash(Exception ex)
    {
        // Mark session as crashed
        _db.Execute("UPDATE SessionRecord SET IsCrashFree=0 WHERE Id=?", _sessionId);
        
        _db.Insert(new CrashRecord
        {
            SessionId = _sessionId,
            ExceptionType = ex.GetType().Name,
            Message = ex.Message,
            OccurredAt = DateTime.UtcNow
        });
    }
    
    public void EndSession()
    {
        _db.Execute("UPDATE SessionRecord SET EndedAt=? WHERE Id=?",
            DateTime.UtcNow, _sessionId);
    }
    
    public async Task ReportMetricsAsync()
    {
        var version = AppInfo.Current.VersionString;
        var total = _db.ExecuteScalar<int>(
            "SELECT COUNT(*) FROM SessionRecord WHERE AppVersion=?", version);
        var crashFree = _db.ExecuteScalar<int>(
            "SELECT COUNT(*) FROM SessionRecord WHERE AppVersion=? AND IsCrashFree=1", version);
        
        if (total == 0) return;
        
        var crashFreeRate = (double)crashFree / total;
        
        await _api.PostAsync<object, object>("/metrics/crash-free-rate", new
        {
            Version = version,
            TotalSessions = total,
            CrashFreeSessions = crashFree,
            CrashFreeRate = crashFreeRate
        });
    }
}

public class SessionRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string AppVersion { get; set; } = string.Empty;
    public string Platform { get; set; } = string.Empty;
    public bool IsCrashFree { get; set; } = true;
    public DateTime StartedAt { get; set; }
    public DateTime? EndedAt { get; set; }
}

public class CrashRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public int SessionId { get; set; }
    public string ExceptionType { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public DateTime OccurredAt { get; set; }
}
```

---

## Step 698: Blue-Green Deployment Strategy

```csharp
// ============================================
// Blue-Green API Deployment Support
// ============================================

public class BlueGreenApiClient
{
    private readonly HttpClient _blue;
    private readonly HttpClient _green;
    private readonly FeatureFlagService _flags;
    private readonly IStructuredLogger _logger;
    
    public BlueGreenApiClient(
        IHttpClientFactory factory,
        FeatureFlagService flags, IStructuredLogger logger)
    {
        _blue = factory.CreateClient("api-blue");
        _green = factory.CreateClient("api-green");
        _flags = flags;
        _logger = logger;
    }
    
    public async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request)
    {
        var useGreen = await _flags.IsEnabledAsync(FeatureFlag.UseGreenEnvironment);
        var client = useGreen ? _green : _blue;
        
        _logger.Info($"Routing to {(useGreen ? "green" : "blue")} environment");
        
        return await client.SendAsync(request);
    }
}

// MauiProgram registration for blue-green
// builder.Services.AddHttpClient("api-blue",
//     c => c.BaseAddress = new Uri("https://api-blue.myfooddelivery.com"));
// builder.Services.AddHttpClient("api-green",
//     c => c.BaseAddress = new Uri("https://api-green.myfooddelivery.com"));
```

---

## Step 699: App Store Metadata Generator

```csharp
// ============================================
// App Store Metadata (Fastlane-compatible)
// ============================================

public class AppStoreMetadataGenerator
{
    public void GenerateFastlaneMetadata(string outputDir)
    {
        Directory.CreateDirectory(outputDir);
        
        // iOS metadata
        var iosDir = Path.Combine(outputDir, "ios");
        Directory.CreateDirectory(iosDir);
        
        File.WriteAllText(Path.Combine(iosDir, "name.txt"), "MyFoodDelivery - สั่งอาหารออนไลน์");
        
        File.WriteAllText(Path.Combine(iosDir, "subtitle.txt"), "ส่งถึงบ้านใน 30 นาที");
        
        File.WriteAllText(Path.Combine(iosDir, "description.txt"), """
            สั่งอาหารง่ายๆ จากร้านอาหารชั้นนำใกล้คุณ
            
            ✅ กว่า 1,000 ร้านอาหาร
            ✅ ส่งเร็วภายใน 30 นาที
            ✅ ติดตาม rider แบบ real-time
            ✅ ชำระผ่าน PromptPay, บัตรเครดิต, เงินสด
            
            ดาวน์โหลดฟรี วันนี้!
            """);
        
        File.WriteAllText(Path.Combine(iosDir, "keywords.txt"),
            "food delivery,สั่งอาหาร,delivery,อาหาร,ส่งอาหาร");
        
        File.WriteAllText(Path.Combine(iosDir, "release_notes.txt"), $"""
            เวอร์ชัน {AppInfo.Current.VersionString}
            - ปรับปรุงประสิทธิภาพการค้นหาร้านอาหาร
            - เพิ่มการแจ้งเตือนสถานะออร์เดอร์
            - แก้ไขข้อผิดพลาดเล็กน้อย
            """);
        
        Console.WriteLine($"Fastlane metadata generated at {outputDir}");
    }
}
```

---

## Step 700: Production Checklist

```csharp
// ============================================
// Production Launch Checklist
// ============================================

/*
 PRE-LAUNCH CHECKLIST
 ====================

 SECURITY
 ✅ All API calls use HTTPS
 ✅ JWT tokens in SecureStorage
 ✅ SQLite database encrypted
 ✅ Certificate pinning enabled
 ✅ No hardcoded secrets
 ✅ ProGuard / R8 obfuscation enabled (Android)
 ✅ Bitcode enabled (iOS)
 ✅ Debug logs removed / disabled

 PERFORMANCE
 ✅ App startup time < 2 seconds
 ✅ Screen transitions smooth (60fps)
 ✅ Images lazy-loaded and cached
 ✅ No memory leaks (use MemoryProfiler)
 ✅ Background tasks don't drain battery
 ✅ API responses cached appropriately

 TESTING
 ✅ Unit test coverage ≥ 80%
 ✅ Integration tests pass
 ✅ UI tests on real devices (iOS + Android)
 ✅ Tested on low-end devices
 ✅ Tested offline mode
 ✅ Tested edge cases (empty lists, error states)
 ✅ Accessibility audit (screen readers)

 APP STORE REQUIREMENTS
 ✅ Privacy policy URL included
 ✅ Data collection disclosure complete
 ✅ App Store screenshots (6.5", 5.5" for iOS)
 ✅ Play Store screenshots (phone + tablet)
 ✅ App icons (all required sizes)
 ✅ Splash screen configured
 ✅ Permissions justified in description

 MONITORING
 ✅ Crash reporting configured
 ✅ Analytics events verified
 ✅ Health check endpoints working
 ✅ Error alerting set up
 ✅ Performance baselines captured

 DEPLOYMENT
 ✅ Build numbers automated (CI build number)
 ✅ Version code monotonically increasing
 ✅ Release notes written
 ✅ Beta testers group notified
 ✅ Staged rollout configured (Play Store)
 ✅ TestFlight group invited (iOS)
 ✅ Server-side feature flags reviewed
 ✅ API backward compatibility verified
 ✅ Database migration scripts ready
 ✅ Rollback plan documented
*/

public static class LaunchChecklist
{
    public static async Task<(bool Passed, List<string> Failures)> RunAsync(
        HealthCheckService health, AppConfiguration config)
    {
        var failures = new List<string>();
        
        // Check environment
        if (config.EnableDebugTools && config.Environment == AppEnvironment.Production)
            failures.Add("Debug tools are enabled in production!");
        
        // Check health
        var report = await health.CheckAllAsync();
        if (!report.IsHealthy)
        {
            foreach (var check in report.Checks.Where(c => c.Status == HealthStatus.Unhealthy))
                failures.Add($"Health check failed: {check.Name} - {check.Description}");
        }
        
        // Check API connectivity
        if (config.ApiBaseUrl.StartsWith("http://") && config.Environment == AppEnvironment.Production)
            failures.Add("Production API must use HTTPS!");
        
        return (!failures.Any(), failures);
    }
}
```

---

## สรุป Part 70

ใน Part 70 เราได้เรียนรู้:

1. **GitHub Actions CI/CD** - Build, test, sign, distribute pipeline
2. **Android AAB Build** - Keystore signing, Play Store deployment
3. **iOS IPA Build** - Certificate, provisioning profile, TestFlight
4. **App Version Management** - Force update, optional update dialogs
5. **Remote Feature Flags** - SQLite cache, fallback defaults
6. **Canary Release** - Deterministic user bucket assignment
7. **Multi-Environment Config** - Dev/Staging/Production settings
8. **Automated Screenshots** - App Store screenshot capture
9. **Crash-Free Rate** - Session tracking, metrics reporting
10. **Production Checklist** - Security, testing, App Store requirements

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 70 | Steps 691-700*
