# Part 46: App Store & Play Store Deployment
## Steps 451-460: Build, Sign, Publish, Update

---

## Step 451: Android Build & Signing

```bash
# ============================================
# Android Release Build
# ============================================

# 1. Create Keystore
keytool -genkey -v -keystore myapp.keystore \
    -alias myapp \
    -keyalg RSA \
    -keysize 2048 \
    -validity 10000

# 2. Build Release APK
dotnet publish -f net9.0-android \
    -c Release \
    -p:AndroidKeyStore=true \
    -p:AndroidSigningKeyStore=myapp.keystore \
    -p:AndroidSigningKeyAlias=myapp \
    -p:AndroidSigningKeyPass=mypassword \
    -p:AndroidSigningStorePass=mystorepass

# 3. Build Release AAB (preferred for Play Store)
dotnet publish -f net9.0-android \
    -c Release \
    -p:AndroidPackageFormat=aab \
    -p:AndroidKeyStore=true \
    -p:AndroidSigningKeyStore=myapp.keystore \
    -p:AndroidSigningKeyAlias=myapp \
    -p:AndroidSigningKeyPass=mypassword \
    -p:AndroidSigningStorePass=mystorepass

# 4. Verify signature
apksigner verify --verbose myapp.apk
```

```xml
<!-- Android/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.mycompany.myapp"
          android:versionCode="1"
          android:versionName="1.0.0">
    
    <uses-sdk android:minSdkVersion="23" android:targetSdkVersion="34" />
    
    <!-- Required permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
                     android:maxSdkVersion="32" />
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="29" />
    
    <application
        android:label="@string/app_name"
        android:icon="@mipmap/appicon"
        android:roundIcon="@mipmap/appicon_round"
        android:allowBackup="true"
        android:supportsRtl="true"
        android:theme="@style/Maui.SplashTheme"
        android:networkSecurityConfig="@xml/network_security_config"
        android:usesCleartextTraffic="false">
        
        <activity android:name="com.microsoft.maui.MainActivity"
                  android:exported="true"
                  android:launchMode="singleTop"
                  android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
                  android:windowSoftInputMode="adjustPan"
                  android:theme="@style/Maui.SplashTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            
            <!-- Deep links -->
            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data android:scheme="https"
                      android:host="myapp.com"
                      android:pathPrefix="/open" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## Step 452: iOS Build & Signing

```csharp
// ============================================
// iOS Build Configuration
// ============================================

/*
 * iOS Signing requires:
 * 1. Apple Developer Account ($99/year)
 * 2. App ID in Apple Developer Portal
 * 3. Provisioning Profile
 * 4. Code Signing Certificate
 *
 * Visual Studio / Rider: Manage automatically
 * CLI: Manual or fastlane
 */
```

```xml
<!-- Info.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>com.mycompany.myapp</string>
    
    <key>CFBundleVersion</key>
    <string>1</string>
    
    <key>CFBundleShortVersionString</key>
    <string>1.0.0</string>
    
    <key>CFBundleDisplayName</key>
    <string>MyApp</string>
    
    <!-- Privacy descriptions (required by App Store) -->
    <key>NSCameraUsageDescription</key>
    <string>แอปใช้กล้องเพื่อถ่ายรูปสินค้า</string>
    
    <key>NSPhotoLibraryUsageDescription</key>
    <string>แอปเข้าถึงรูปภาพเพื่อเลือกรูปสินค้า</string>
    
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>แอปใช้ตำแหน่งเพื่อค้นหาร้านค้าใกล้คุณ</string>
    
    <key>NSMicrophoneUsageDescription</key>
    <string>แอปใช้ไมค์เพื่อค้นหาด้วยเสียง</string>
    
    <!-- URL Schemes -->
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleURLName</key>
            <string>com.mycompany.myapp</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>myapp</string>
            </array>
        </dict>
    </array>
    
    <!-- Associated Domains for deep links -->
    <key>com.apple.developer.associated-domains</key>
    <array>
        <string>applinks:myapp.com</string>
    </array>
    
    <!-- App Transport Security -->
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <false/>
    </dict>
</dict>
</plist>
```

```bash
# Build iOS IPA
dotnet publish -f net9.0-ios \
    -c Release \
    -p:ArchiveOnBuild=true \
    -p:CodesignKey="Apple Distribution: My Company" \
    -p:CodesignProvision="MyApp Distribution"

# Upload with Transporter or altool
xcrun altool --upload-app \
    --type ios \
    --file bin/Release/net9.0-ios/MyApp.ipa \
    --username "developer@email.com" \
    --password "@keychain:AC_PASSWORD"
```

---

## Step 453: Version Management

```csharp
// ============================================
// Automated Version Management
// ============================================

// .csproj
/*
<PropertyGroup>
    <ApplicationDisplayVersion>1.2.3</ApplicationDisplayVersion>
    <ApplicationVersion>42</ApplicationVersion>
</PropertyGroup>
*/

// Automated versioning script (version.sh or version.ps1)
/*
#!/bin/bash
# Auto-increment build number based on git commit count

VERSION_NAME="1.2.3"
BUILD_NUMBER=$(git rev-list --count HEAD)

echo "Version: $VERSION_NAME ($BUILD_NUMBER)"

# Update .csproj
sed -i "s/<ApplicationVersion>.*<\/ApplicationVersion>/<ApplicationVersion>$BUILD_NUMBER<\/ApplicationVersion>/" MyApp.csproj
sed -i "s/<ApplicationDisplayVersion>.*<\/ApplicationDisplayVersion>/<ApplicationDisplayVersion>$VERSION_NAME<\/ApplicationDisplayVersion>/" MyApp.csproj
*/

public class VersionInfo
{
    public string DisplayVersion => AppInfo.Current.VersionString;
    public string BuildNumber => AppInfo.Current.BuildString;
    public string FullVersion => $"{DisplayVersion} ({BuildNumber})";
    
    public async Task<bool> IsUpdateAvailableAsync(HttpClient http)
    {
        try
        {
            var response = await http.GetFromJsonAsync<VersionCheckResponse>(
                "https://myapp.com/api/version/latest");
            
            if (response == null) return false;
            
            return IsHigherVersion(response.LatestVersion, DisplayVersion);
        }
        catch { return false; }
    }
    
    private static bool IsHigherVersion(string latest, string current)
    {
        if (!Version.TryParse(latest, out var v1)) return false;
        if (!Version.TryParse(current, out var v2)) return false;
        return v1 > v2;
    }
    
    // Open app store page
    public async Task OpenStorePageAsync()
    {
#if ANDROID
        var packageName = AppInfo.Current.PackageName;
        await Launcher.Default.OpenAsync(
            new Uri($"market://details?id={packageName}"));
#elif IOS
        const string appId = "1234567890"; // Your App Store ID
        await Launcher.Default.OpenAsync(
            new Uri($"itms-apps://itunes.apple.com/app/id{appId}"));
#endif
    }
}

public record VersionCheckResponse(string LatestVersion, bool ForceUpdate, string? ReleaseNotes);
```

---

## Step 454: In-App Update

```csharp
// ============================================
// In-App Update Dialog
// ============================================

public class AppUpdateService
{
    private readonly VersionInfo _version;
    private readonly HttpClient _http;
    
    public AppUpdateService(VersionInfo version, HttpClient http)
    {
        _version = version;
        _http = http;
    }
    
    public async Task CheckAndPromptAsync(Page page)
    {
        var response = await CheckVersionAsync();
        if (response == null) return;
        
        if (response.ForceUpdate)
        {
            // Must update - can't dismiss
            await page.DisplayAlert(
                "ต้องอัปเดต",
                $"กรุณาอัปเดตแอปเป็นเวอร์ชัน {response.LatestVersion}\n\n{response.ReleaseNotes}",
                "อัปเดตเดี๋ยวนี้");
            
            await _version.OpenStorePageAsync();
        }
        else if (await _version.IsUpdateAvailableAsync(_http))
        {
            var shouldUpdate = await page.DisplayAlert(
                "มีอัปเดต",
                $"เวอร์ชันใหม่ {response.LatestVersion} พร้อมแล้ว\n\n{response.ReleaseNotes}",
                "อัปเดตเดี๋ยวนี้",
                "ภายหลัง");
            
            if (shouldUpdate)
                await _version.OpenStorePageAsync();
        }
    }
    
    private async Task<VersionCheckResponse?> CheckVersionAsync()
    {
        try
        {
            return await _http.GetFromJsonAsync<VersionCheckResponse>(
                "https://myapp.com/api/version/check");
        }
        catch { return null; }
    }
}
```

---

## Step 455: Crash Reporting

```csharp
// ============================================
// Firebase Crashlytics / AppCenter Integration
// ============================================

public class CrashReportingService
{
    public void Initialize()
    {
        // Global unhandled exception handler
        AppDomain.CurrentDomain.UnhandledException += (sender, e) =>
        {
            if (e.ExceptionObject is Exception ex)
                RecordException(ex, "UnhandledException", fatal: e.IsTerminating);
        };
        
        // MAUI unhandled exceptions
        Microsoft.Maui.Handlers.WindowHandler.Mapper.AppendToMapping(
            "WindowCrashHandler", (handler, _) =>
            {
#if ANDROID
                Android.Runtime.AndroidEnvironment.UnhandledExceptionRaiser += (s, e) =>
                {
                    RecordException(e.Exception, "AndroidUnhandled", fatal: true);
                    e.Handled = true;
                };
#endif
            });
    }
    
    public void RecordException(Exception ex, string? context = null, bool fatal = false)
    {
        // Firebase Crashlytics
#if ANDROID || IOS
        // Firebase.Crashlytics.Crashlytics.Instance.RecordException(ex);
#endif
        
        // Custom crash log to SQLite (for debugging)
        LogCrashLocally(ex, context, fatal);
    }
    
    public void SetUserId(string userId)
    {
        // Firebase.Crashlytics.Crashlytics.Instance.SetUserId(userId);
    }
    
    public void SetCustomKey(string key, string value)
    {
        // Firebase.Crashlytics.Crashlytics.Instance.SetCustomValue(key, value);
    }
    
    private void LogCrashLocally(Exception ex, string? context, bool fatal)
    {
        try
        {
            var logDir = Path.Combine(FileSystem.AppDataDirectory, "crash_logs");
            Directory.CreateDirectory(logDir);
            
            var fileName = $"crash_{DateTime.Now:yyyyMMdd_HHmmss}.txt";
            var filePath = Path.Combine(logDir, fileName);
            
            var content = $"""
                Timestamp: {DateTime.UtcNow:O}
                Context: {context ?? "unknown"}
                Fatal: {fatal}
                Type: {ex.GetType().FullName}
                Message: {ex.Message}
                Stack:
                {ex.StackTrace}
                Inner:
                {ex.InnerException?.ToString()}
                Device: {DeviceInfo.Current.Model} {DeviceInfo.Current.VersionString}
                App: {AppInfo.Current.VersionString} ({AppInfo.Current.BuildString})
                """;
            
            File.WriteAllText(filePath, content);
        }
        catch { /* Don't crash the crash handler */ }
    }
    
    public List<string> GetLocalCrashLogs()
    {
        var logDir = Path.Combine(FileSystem.AppDataDirectory, "crash_logs");
        if (!Directory.Exists(logDir)) return [];
        
        return Directory.GetFiles(logDir, "crash_*.txt")
            .OrderByDescending(f => f)
            .Take(10)
            .ToList();
    }
}
```

---

## Step 456: Analytics Integration

```csharp
// ============================================
// Firebase Analytics / Custom Analytics
// ============================================

public interface IAnalyticsService
{
    void TrackScreen(string screenName);
    void TrackEvent(string eventName, Dictionary<string, string>? parameters = null);
    void TrackPurchase(decimal amount, string currency, string orderId);
    void TrackSearch(string query, int resultCount);
    void SetUserProperty(string name, string value);
    void SetUserId(string userId);
}

public class AnalyticsService : IAnalyticsService
{
    private readonly SQLiteConnection _db;
    private string? _userId;
    
    public AnalyticsService(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<AnalyticsEvent>();
    }
    
    public void TrackScreen(string screenName)
        => Track("screen_view", new() { { "screen_name", screenName } });
    
    public void TrackEvent(string eventName, Dictionary<string, string>? parameters = null)
        => Track(eventName, parameters ?? new());
    
    public void TrackPurchase(decimal amount, string currency, string orderId)
        => Track("purchase", new()
        {
            { "amount", amount.ToString("F2") },
            { "currency", currency },
            { "order_id", orderId }
        });
    
    public void TrackSearch(string query, int resultCount)
        => Track("search", new()
        {
            { "query", query },
            { "result_count", resultCount.ToString() }
        });
    
    public void SetUserProperty(string name, string value)
    {
        _db.Insert(new AnalyticsEvent
        {
            Name = "__user_property",
            Parameters = System.Text.Json.JsonSerializer.Serialize(new { name, value }),
            UserId = _userId,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public void SetUserId(string userId) => _userId = userId;
    
    private void Track(string name, Dictionary<string, string> parameters)
    {
        _db.Insert(new AnalyticsEvent
        {
            Name = name,
            Parameters = System.Text.Json.JsonSerializer.Serialize(parameters),
            UserId = _userId,
            SessionId = GetSessionId(),
            Timestamp = DateTime.UtcNow,
            DeviceModel = DeviceInfo.Current.Model,
            OsVersion = DeviceInfo.Current.VersionString,
            AppVersion = AppInfo.Current.VersionString
        });
    }
    
    private string _sessionId = Guid.NewGuid().ToString("N")[..8];
    private string GetSessionId() => _sessionId;
    
    // Batch send to server
    public async Task FlushAsync(HttpClient http, CancellationToken ct = default)
    {
        var pending = _db.Table<AnalyticsEvent>()
            .Where(e => !e.Sent)
            .Take(100)
            .ToList();
        
        if (pending.Count == 0) return;
        
        var response = await http.PostAsJsonAsync("api/analytics/events",
            new { events = pending }, ct);
        
        if (response.IsSuccessStatusCode)
        {
            foreach (var e in pending)
            {
                e.Sent = true;
                _db.Update(e);
            }
        }
    }
}

public class AnalyticsEvent
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Parameters { get; set; } = "{}";
    public string? UserId { get; set; }
    public string? SessionId { get; set; }
    public string? DeviceModel { get; set; }
    public string? OsVersion { get; set; }
    public string? AppVersion { get; set; }
    public DateTime Timestamp { get; set; }
    public bool Sent { get; set; }
}
```

---

## Step 457: Feature Flags Production

```csharp
// ============================================
// Remote Feature Flags
// ============================================

public class RemoteFeatureFlagService
{
    private readonly HttpClient _http;
    private readonly IPreferences _prefs;
    private Dictionary<string, bool> _flags = new();
    
    public RemoteFeatureFlagService(HttpClient http, IPreferences prefs)
    {
        _http = http;
        _prefs = prefs;
        LoadLocalCache();
    }
    
    public async Task RefreshAsync(CancellationToken ct = default)
    {
        try
        {
            var flags = await _http.GetFromJsonAsync<Dictionary<string, bool>>(
                "api/feature-flags", ct);
            
            if (flags == null) return;
            
            _flags = flags;
            _prefs.Set("feature_flags", System.Text.Json.JsonSerializer.Serialize(flags));
        }
        catch
        {
            // Use cached flags
        }
    }
    
    public bool IsEnabled(string flagName, bool defaultValue = false)
        => _flags.GetValueOrDefault(flagName, defaultValue);
    
    private void LoadLocalCache()
    {
        var json = _prefs.Get("feature_flags", "{}");
        _flags = System.Text.Json.JsonSerializer
            .Deserialize<Dictionary<string, bool>>(json) ?? new();
    }
}

// Feature flag constants
public static class FeatureFlags
{
    public const string NewCheckoutFlow = "new_checkout_flow";
    public const string AIRecommendations = "ai_recommendations";
    public const string DarkModeV2 = "dark_mode_v2";
    public const string InstantDelivery = "instant_delivery";
    public const string LiveChat = "live_chat";
    public const string SocialLogin = "social_login";
}

// Usage in ViewModel
public class CheckoutViewModel
{
    private readonly RemoteFeatureFlagService _flags;
    
    public CheckoutViewModel(RemoteFeatureFlagService flags) => _flags = flags;
    
    public bool ShowNewFlow => _flags.IsEnabled(FeatureFlags.NewCheckoutFlow);
    public bool ShowInstantDelivery => _flags.IsEnabled(FeatureFlags.InstantDelivery);
}
```

---

## Step 458: Play Store Automated Upload

```yaml
# .github/workflows/play-store.yml
name: Deploy to Play Store

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install MAUI workloads
        run: dotnet workload install maui-android
      
      - name: Extract version from tag
        id: version
        run: |
          TAG=${GITHUB_REF#refs/tags/v}
          echo "VERSION=$TAG" >> $GITHUB_ENV
          BUILD=$(git rev-list --count HEAD)
          echo "BUILD=$BUILD" >> $GITHUB_ENV
      
      - name: Build Release AAB
        run: |
          dotnet publish -f net9.0-android -c Release \
            -p:ApplicationVersion=${{ env.BUILD }} \
            -p:ApplicationDisplayVersion=${{ env.VERSION }} \
            -p:AndroidPackageFormat=aab \
            -p:AndroidKeyStore=true \
            -p:AndroidSigningKeyStore=myapp.keystore \
            -p:AndroidSigningKeyAlias=myapp \
            -p:AndroidSigningKeyPass=${{ secrets.KEYSTORE_PASS }} \
            -p:AndroidSigningStorePass=${{ secrets.KEYSTORE_STORE_PASS }}
        env:
          KEYSTORE_CONTENT: ${{ secrets.KEYSTORE_BASE64 }}
      
      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > myapp.keystore
      
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_JSON }}
          packageName: com.mycompany.myapp
          releaseFiles: bin/Release/net9.0-android/publish/com.mycompany.myapp-Signed.aab
          track: internal
          status: completed
          inAppUpdatePriority: 2
```

---

## Step 459: App Store Connect Upload

```yaml
# .github/workflows/app-store.yml
name: Deploy to App Store

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy-ios:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install MAUI workloads
        run: dotnet workload install maui-ios
      
      - name: Import Certificate
        uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_P12_PASSWORD }}
      
      - name: Download Provisioning Profile
        uses: apple-actions/download-provisioning-profiles@v1
        with:
          bundle-id: com.mycompany.myapp
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
      
      - name: Build IPA
        run: |
          dotnet publish -f net9.0-ios -c Release \
            -p:ArchiveOnBuild=true \
            -p:CodesignKey="${{ secrets.CODESIGN_KEY }}" \
            -p:CodesignProvision="${{ secrets.PROVISION_PROFILE }}"
      
      - name: Upload to App Store
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: bin/Release/net9.0-ios/MyApp.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
```

---

## Step 460: App Review Checklist

```csharp
// ============================================
// Pre-submission Review Checklist
// ============================================

/*
 * ✅ Android Play Store Checklist:
 * 
 * Content & Assets:
 * □ App icon 512x512 PNG
 * □ Feature graphic 1024x500 PNG
 * □ Screenshots: Phone (min 2), Tablet (min 1)
 * □ App description (Thai + English)
 * □ Short description (80 chars)
 * □ Privacy policy URL
 * □ App category selected
 * 
 * Technical:
 * □ Target SDK 34+ (Android 14)
 * □ 64-bit support (arm64-v8a)
 * □ No debug/test code in release
 * □ Signed with production keystore
 * □ ProGuard/R8 enabled
 * □ No unused permissions
 * □ Tested on multiple devices
 * 
 * Compliance:
 * □ Data safety form filled
 * □ Content rating questionnaire
 * □ Financial/medical declarations if applicable
 * 
 * ✅ iOS App Store Checklist:
 * 
 * Content & Assets:
 * □ App icon 1024x1024 PNG (no alpha)
 * □ Screenshots: 6.7" iPhone, 12.9" iPad
 * □ App description
 * □ Keywords (100 chars)
 * □ Privacy policy URL
 * □ Support URL
 * □ Age rating questionnaire
 * 
 * Technical:
 * □ Info.plist privacy descriptions
 * □ No private APIs
 * □ No third-party code violations
 * □ Tested on latest iOS
 * □ Handles app backgrounding correctly
 * □ Supports iOS 16+
 * 
 * Compliance:
 * □ Privacy nutrition label complete
 * □ Sign in with Apple if social login
 * □ In-app purchase if virtual goods
 */

public class AppStoreReadinessChecker
{
    public AppReadinessReport Check()
    {
        var issues = new List<string>();
        
        // Check app version
        if (string.IsNullOrEmpty(AppInfo.Current.VersionString))
            issues.Add("App version not set");
        
        // Check build number
        if (string.IsNullOrEmpty(AppInfo.Current.BuildString))
            issues.Add("Build number not set");
        
        // Check connectivity handling
        // Check crash handler
        // Check permission descriptions
        
        return new AppReadinessReport(
            IsReady: issues.Count == 0,
            Issues: issues,
            Version: AppInfo.Current.VersionString,
            Build: AppInfo.Current.BuildString);
    }
}

public record AppReadinessReport(
    bool IsReady, List<string> Issues, string Version, string Build);
```

---

## สรุป Part 46

ใน Part 46 เราได้เรียนรู้:

1. **Android Build & Sign** - Keystore, AAB, APK
2. **iOS Build & Sign** - Certificates, Provisioning, IPA
3. **Version Management** - Semantic version, auto build number
4. **In-App Update** - Force/optional update dialog
5. **Crash Reporting** - Firebase Crashlytics, local logs
6. **Analytics** - Event tracking, batched upload
7. **Feature Flags** - Remote flags with local cache
8. **Play Store Upload** - GitHub Actions automated deploy
9. **App Store Upload** - iOS automated CI/CD
10. **Review Checklist** - Complete submission requirements

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 46 | Steps 451-460*
