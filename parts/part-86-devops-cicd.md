# Part 86: DevOps & CI/CD
## Steps 851-860: GitHub Actions, App Distribution, Crash Reporting, Feature Flags

---

## Step 851: Full CI/CD Pipeline

```yaml
# .github/workflows/maui-cicd.yml
name: MAUI CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '9.0.x'
  PROJECT: 'src/FoodDelivery.Maui/FoodDelivery.Maui.csproj'

jobs:
  # ─────────────────────────────────────────
  # 1. Build & Test
  # ─────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '${{ env.DOTNET_VERSION }}' }
      
      - name: Restore
        run: dotnet restore
      
      - name: Build (non-MAUI projects)
        run: dotnet build --no-restore --configuration Release
          src/FoodDelivery.Domain
          src/FoodDelivery.Application
          src/FoodDelivery.Infrastructure
      
      - name: Test with coverage
        run: |
          dotnet test tests/ \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            --logger "github" \
            --filter "Category!=E2E"
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage/**/coverage.cobertura.xml
  
  # ─────────────────────────────────────────
  # 2. Android Build
  # ─────────────────────────────────────────
  android:
    runs-on: macos-15
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '${{ env.DOTNET_VERSION }}' }
      
      - name: Install MAUI workloads
        run: dotnet workload install maui-android
      
      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > keystore.jks
      
      - name: Build Android
        run: |
          dotnet publish ${{ env.PROJECT }} \
            -f net9.0-android \
            -c Release \
            -p:AndroidKeyStore=true \
            -p:AndroidSigningKeyStore=../../keystore.jks \
            -p:AndroidSigningKeyAlias=${{ secrets.ANDROID_KEY_ALIAS }} \
            -p:AndroidSigningKeyPass=${{ secrets.ANDROID_KEY_PASSWORD }} \
            -p:AndroidSigningStorePass=${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
      
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: '**/*.apk'
  
  # ─────────────────────────────────────────
  # 3. iOS Build
  # ─────────────────────────────────────────
  ios:
    runs-on: macos-15
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '${{ env.DOTNET_VERSION }}' }
      
      - name: Install MAUI workloads
        run: dotnet workload install maui-ios
      
      - name: Install provisioning profile
        run: |
          echo "${{ secrets.IOS_PROVISION_BASE64 }}" | base64 -d > profile.mobileprovision
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/
      
      - name: Build iOS
        run: |
          dotnet publish ${{ env.PROJECT }} \
            -f net9.0-ios \
            -c Release \
            -p:ArchiveOnBuild=true \
            -p:RuntimeIdentifier=ios-arm64
      
      - name: Upload IPA
        uses: actions/upload-artifact@v4
        with:
          name: ios-release
          path: '**/*.ipa'
```

---

## Step 852: Automated App Distribution

```yaml
# .github/workflows/distribute.yml
name: Distribute to Testers

on:
  push:
    branches: [develop]
    tags: ['v*-beta*']

jobs:
  distribute-android:
    runs-on: macos-15
    needs: [android-build]  # reuse build job above
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with: { name: android-release }
      
      - name: Distribute to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_CREDENTIALS }}
          groups: internal-testers
          releaseNotes: |
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            ${{ github.event.head_commit.message }}
          file: path/to/app-release.apk
  
  distribute-ios:
    runs-on: macos-15
    needs: [ios-build]
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with: { name: ios-release }
      
      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: path/to/App.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_API_KEY }}
```

---

## Step 853: Crash Reporting

```csharp
// ============================================
// Crash Reporting & Error Tracking
// ============================================

public class CrashReportingService
{
    private readonly ILogger<CrashReportingService> _logger;
    
    public CrashReportingService(ILogger<CrashReportingService> logger)
    {
        _logger = logger;
        SetupGlobalHandler();
    }
    
    private void SetupGlobalHandler()
    {
        // .NET unhandled exceptions
        AppDomain.CurrentDomain.UnhandledException += (_, args) =>
        {
            if (args.ExceptionObject is Exception ex)
                ReportCrash(ex, "AppDomain.UnhandledException", isFatal: true);
        };
        
        // Task exceptions
        TaskScheduler.UnobservedTaskException += (_, args) =>
        {
            ReportCrash(args.Exception, "TaskScheduler.UnobservedTaskException", isFatal: false);
            args.SetObserved();
        };
        
        // MAUI unhandled exceptions
        MauiExceptions.UnhandledException += (_, args) =>
        {
            ReportCrash(args.ExceptionObject as Exception
                ?? new Exception(args.ExceptionObject?.ToString()), "MAUI", isFatal: true);
        };
    }
    
    public void ReportCrash(Exception ex, string source, bool isFatal = false)
    {
        var report = new CrashReport
        {
            Id = Guid.NewGuid().ToString(),
            ExceptionType = ex.GetType().FullName ?? "Unknown",
            Message = ex.Message,
            StackTrace = ex.StackTrace ?? "",
            Source = source,
            IsFatal = isFatal,
            AppVersion = AppInfo.VersionString,
            Platform = DeviceInfo.Platform.ToString(),
            OsVersion = DeviceInfo.VersionString,
            DeviceModel = DeviceInfo.Model,
            Timestamp = DateTime.UtcNow
        };
        
        _logger.LogCritical(ex, "Crash [{Source}] {Message}", source, ex.Message);
        
        // Send to backend asynchronously (best-effort)
        Task.Run(async () =>
        {
            try { await SendReportAsync(report); }
            catch { /* Don't crash the crash reporter */ }
        });
    }
    
    private async Task SendReportAsync(CrashReport report)
    {
        // Store locally first in case of network failure
        await LocalCrashStore.SaveAsync(report);
        
        // Then flush to backend
        await LocalCrashStore.FlushToBackendAsync();
    }
    
    public void SetUserContext(string userId, string email)
    {
        Preferences.Set("crash_user_id", userId);
        Preferences.Set("crash_user_email", email);
    }
    
    public void AddBreadcrumb(string message, Dictionary<string, string>? data = null)
    {
        var breadcrumb = new Breadcrumb(message, DateTime.UtcNow, data);
        BreadcrumbBuffer.Add(breadcrumb);
    }
}

public record CrashReport(
    string Id, string ExceptionType, string Message, string StackTrace,
    string Source, bool IsFatal, string AppVersion, string Platform,
    string OsVersion, string DeviceModel, DateTime Timestamp)
{
    public CrashReport() : this("", "", "", "", "", false, "", "", "", "", DateTime.MinValue) { }
    
    public string Id { get; set; } = Id;
    public string ExceptionType { get; set; } = ExceptionType;
    public string Message { get; set; } = Message;
    public string StackTrace { get; set; } = StackTrace;
    public string Source { get; set; } = Source;
    public bool IsFatal { get; set; } = IsFatal;
    public string AppVersion { get; set; } = AppVersion;
    public string Platform { get; set; } = Platform;
    public string OsVersion { get; set; } = OsVersion;
    public string DeviceModel { get; set; } = DeviceModel;
    public DateTime Timestamp { get; set; } = Timestamp;
}
```

---

## Step 854: Feature Flag System

```csharp
// ============================================
// Feature Flag Management
// ============================================

public interface IFeatureFlags
{
    bool IsEnabled(string flag);
    Task<bool> IsEnabledAsync(string flag);
    T GetValue<T>(string flag, T defaultValue);
}

public class FeatureFlagService : IFeatureFlags
{
    private Dictionary<string, object> _flags = new();
    private readonly HttpClient _http;
    private readonly ILogger<FeatureFlagService> _logger;
    private DateTime _lastFetch = DateTime.MinValue;
    private const int CacheMinutes = 5;
    
    public FeatureFlagService(HttpClient http, ILogger<FeatureFlagService> logger)
    {
        _http = http;
        _logger = logger;
        LoadDefaults();
    }
    
    private void LoadDefaults()
    {
        // Compile-time defaults – safe to ship disabled
        _flags = new Dictionary<string, object>
        {
            ["new_checkout_flow"] = false,
            ["ml_recommendations"] = false,
            ["group_ordering"] = false,
            ["loyalty_points"] = true,
            ["dark_mode_beta"] = true,
            ["max_items_per_order"] = 50,
            ["search_radius_km"] = 10
        };
        
        // Load persisted flags
        var cached = Preferences.Get("feature_flags_json", "");
        if (!string.IsNullOrEmpty(cached))
        {
            try
            {
                var parsed = System.Text.Json.JsonSerializer
                    .Deserialize<Dictionary<string, System.Text.Json.JsonElement>>(cached);
                if (parsed != null)
                    foreach (var kv in parsed)
                        _flags[kv.Key] = DeserializeValue(kv.Value);
            }
            catch { /* use defaults */ }
        }
    }
    
    public async Task RefreshAsync()
    {
        if ((DateTime.UtcNow - _lastFetch).TotalMinutes < CacheMinutes) return;
        
        try
        {
            var userId = Preferences.Get("user_id", "");
            var response = await _http.GetFromJsonAsync<Dictionary<string, System.Text.Json.JsonElement>>(
                $"/api/feature-flags?userId={userId}");
            
            if (response != null)
            {
                foreach (var kv in response)
                    _flags[kv.Key] = DeserializeValue(kv.Value);
                
                Preferences.Set("feature_flags_json",
                    System.Text.Json.JsonSerializer.Serialize(_flags));
                
                _lastFetch = DateTime.UtcNow;
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Feature flag refresh failed, using cached");
        }
    }
    
    public bool IsEnabled(string flag)
        => _flags.TryGetValue(flag, out var v) && v is bool b && b;
    
    public async Task<bool> IsEnabledAsync(string flag)
    {
        await RefreshAsync();
        return IsEnabled(flag);
    }
    
    public T GetValue<T>(string flag, T defaultValue)
    {
        if (!_flags.TryGetValue(flag, out var v)) return defaultValue;
        try { return (T)Convert.ChangeType(v, typeof(T)); }
        catch { return defaultValue; }
    }
    
    private static object DeserializeValue(System.Text.Json.JsonElement el) =>
        el.ValueKind switch
        {
            System.Text.Json.JsonValueKind.True => true,
            System.Text.Json.JsonValueKind.False => false,
            System.Text.Json.JsonValueKind.Number => el.GetDouble(),
            System.Text.Json.JsonValueKind.String => el.GetString() ?? "",
            _ => el.ToString()
        };
}
```

---

## Step 855: App Version Management

```csharp
// ============================================
// Version Check & Force Update
// ============================================

public class VersionService
{
    private readonly HttpClient _http;
    
    public VersionService(HttpClient http) => _http = http;
    
    public async Task<VersionCheckResult> CheckVersionAsync()
    {
        var current = new Version(AppInfo.VersionString);
        
        try
        {
            var info = await _http.GetFromJsonAsync<AppVersionInfo>(
                $"/api/app-version?platform={DeviceInfo.Platform.ToString().ToLower()}");
            
            if (info == null) return VersionCheckResult.UpToDate;
            
            var minimum = new Version(info.MinimumVersion);
            var latest = new Version(info.LatestVersion);
            
            if (current < minimum)
                return VersionCheckResult.ForceUpdate(info.LatestVersion, info.UpdateUrl, info.UpdateMessage);
            
            if (current < latest)
                return VersionCheckResult.SoftUpdate(info.LatestVersion, info.UpdateUrl);
            
            return VersionCheckResult.UpToDate;
        }
        catch
        {
            return VersionCheckResult.UpToDate; // fail open
        }
    }
}

public record AppVersionInfo(
    string MinimumVersion, string LatestVersion,
    string UpdateUrl, string UpdateMessage);

public record VersionCheckResult(
    VersionStatus Status, string? NewVersion, string? StoreUrl, string? Message)
{
    public static VersionCheckResult UpToDate =>
        new(VersionStatus.UpToDate, null, null, null);
    
    public static VersionCheckResult ForceUpdate(string v, string url, string msg) =>
        new(VersionStatus.ForceUpdate, v, url, msg);
    
    public static VersionCheckResult SoftUpdate(string v, string url) =>
        new(VersionStatus.SoftUpdate, v, url, null);
}

public enum VersionStatus { UpToDate, SoftUpdate, ForceUpdate }

// ViewModel usage
public class AppStartupViewModel : BaseViewModel
{
    private readonly VersionService _version;
    
    public async Task CheckVersionAsync()
    {
        var result = await _version.CheckVersionAsync();
        
        switch (result.Status)
        {
            case VersionStatus.ForceUpdate:
                await Application.Current!.MainPage!.DisplayAlert(
                    "อัปเดตจำเป็น",
                    result.Message ?? "กรุณาอัปเดตแอปเพื่อใช้งานต่อ",
                    "อัปเดต");
                await Launcher.Default.OpenAsync(result.StoreUrl!);
                break;
            
            case VersionStatus.SoftUpdate:
                var ok = await Application.Current!.MainPage!.DisplayAlert(
                    "มีเวอร์ชั่นใหม่", $"เวอร์ชั่น {result.NewVersion} พร้อมใช้งาน", "อัปเดต", "ภายหลัง");
                if (ok) await Launcher.Default.OpenAsync(result.StoreUrl!);
                break;
        }
    }
}
```

---

## Step 856: Analytics & Telemetry

```csharp
// ============================================
// Analytics Service (GA4-compatible)
// ============================================

public interface IAnalytics
{
    void Track(string eventName, Dictionary<string, object>? properties = null);
    void Screen(string screenName, Dictionary<string, object>? properties = null);
    void SetUser(string userId, Dictionary<string, object>? traits = null);
}

public class AnalyticsService : IAnalytics
{
    private readonly HttpClient _http;
    private readonly Queue<AnalyticsEvent> _buffer = new();
    private readonly Timer _flushTimer;
    private string _userId = "anonymous";
    private string _sessionId = Guid.NewGuid().ToString();
    
    public AnalyticsService(HttpClient http)
    {
        _http = http;
        _flushTimer = new Timer(async _ => await FlushAsync(), null,
            TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
    }
    
    public void Track(string eventName, Dictionary<string, object>? properties = null)
    {
        var evt = new AnalyticsEvent
        {
            Name = eventName,
            UserId = _userId,
            SessionId = _sessionId,
            Properties = properties ?? new(),
            Timestamp = DateTime.UtcNow,
            Platform = DeviceInfo.Platform.ToString(),
            AppVersion = AppInfo.VersionString
        };
        
        lock (_buffer) _buffer.Enqueue(evt);
    }
    
    public void Screen(string screenName, Dictionary<string, object>? properties = null)
        => Track("screen_view", new Dictionary<string, object>
        {
            ["screen_name"] = screenName,
            ...(properties ?? new())
        });
    
    public void SetUser(string userId, Dictionary<string, object>? traits = null)
    {
        _userId = userId;
        Preferences.Set("analytics_user_id", userId);
        Track("user_identified", traits);
    }
    
    private async Task FlushAsync()
    {
        List<AnalyticsEvent> batch;
        lock (_buffer)
        {
            if (!_buffer.Any()) return;
            batch = _buffer.ToList();
            _buffer.Clear();
        }
        
        try
        {
            await _http.PostAsJsonAsync("/api/analytics/batch", new { events = batch });
        }
        catch
        {
            // Re-queue on failure
            lock (_buffer)
                foreach (var evt in batch)
                    _buffer.Enqueue(evt);
        }
    }
}

public class AnalyticsEvent
{
    public string Name { get; set; } = "";
    public string UserId { get; set; } = "";
    public string SessionId { get; set; } = "";
    public Dictionary<string, object> Properties { get; set; } = new();
    public DateTime Timestamp { get; set; }
    public string Platform { get; set; } = "";
    public string AppVersion { get; set; } = "";
}
```

---

## Step 857: App Performance Monitoring

```csharp
// ============================================
// APM - Application Performance Monitoring
// ============================================

public class AppPerformanceMonitor
{
    private readonly IAnalytics _analytics;
    private readonly Dictionary<string, Stopwatch> _traces = new();
    
    public AppPerformanceMonitor(IAnalytics analytics) => _analytics = analytics;
    
    public IDisposable StartTrace(string name)
    {
        var sw = Stopwatch.StartNew();
        return new TraceHandle(name, sw, this);
    }
    
    internal void EndTrace(string name, Stopwatch sw)
    {
        sw.Stop();
        _analytics.Track("performance_trace", new Dictionary<string, object>
        {
            ["trace_name"] = name,
            ["duration_ms"] = sw.ElapsedMilliseconds,
            ["is_slow"] = sw.ElapsedMilliseconds > GetSlowThreshold(name)
        });
    }
    
    private long GetSlowThreshold(string name) => name switch
    {
        "app_start" => 3000,
        "page_load" => 1000,
        "search" => 300,
        "checkout" => 2000,
        _ => 1000
    };
    
    private class TraceHandle : IDisposable
    {
        private readonly string _name;
        private readonly Stopwatch _sw;
        private readonly AppPerformanceMonitor _monitor;
        
        public TraceHandle(string name, Stopwatch sw, AppPerformanceMonitor monitor)
        {
            _name = name; _sw = sw; _monitor = monitor;
        }
        
        public void Dispose() => _monitor.EndTrace(_name, _sw);
    }
}
```

---

## Step 858: Environment Configuration

```csharp
// ============================================
// Environment-Specific Configuration
// ============================================

public enum AppEnvironment { Development, Staging, Production }

public record EnvironmentConfig(
    string ApiBaseUrl, string WsBaseUrl, string MapApiKey,
    bool EnableCrashReporting, bool EnableAnalytics,
    LogLevel MinLogLevel, bool EnableCertificatePinning);

public static class AppConfig
{
    public static AppEnvironment Current => GetEnvironment();
    
    public static EnvironmentConfig Config => Current switch
    {
        AppEnvironment.Development => new(
            ApiBaseUrl: "https://api-dev.fooddelivery.th",
            WsBaseUrl: "wss://ws-dev.fooddelivery.th",
            MapApiKey: "dev-map-key",
            EnableCrashReporting: false,
            EnableAnalytics: false,
            MinLogLevel: LogLevel.Debug,
            EnableCertificatePinning: false),
        
        AppEnvironment.Staging => new(
            ApiBaseUrl: "https://api-staging.fooddelivery.th",
            WsBaseUrl: "wss://ws-staging.fooddelivery.th",
            MapApiKey: "staging-map-key",
            EnableCrashReporting: true,
            EnableAnalytics: true,
            MinLogLevel: LogLevel.Information,
            EnableCertificatePinning: false),
        
        _ => new( // Production
            ApiBaseUrl: "https://api.fooddelivery.th",
            WsBaseUrl: "wss://ws.fooddelivery.th",
            MapApiKey: Environment.GetEnvironmentVariable("MAP_API_KEY") ?? "",
            EnableCrashReporting: true,
            EnableAnalytics: true,
            MinLogLevel: LogLevel.Warning,
            EnableCertificatePinning: true),
    };
    
    private static AppEnvironment GetEnvironment()
    {
#if DEBUG
        return AppEnvironment.Development;
#else
        var env = Environment.GetEnvironmentVariable("APP_ENVIRONMENT") ?? "Production";
        return Enum.TryParse<AppEnvironment>(env, out var result)
            ? result : AppEnvironment.Production;
#endif
    }
}
```

---

## Step 859: Release Automation

```csharp
// ============================================
// Automated Release Notes Generator
// ============================================

// Called as part of release workflow
public class ReleaseNotesGenerator
{
    public async Task<string> GenerateAsync(string fromTag, string toTag)
    {
        var commits = await GetCommitsAsync(fromTag, toTag);
        
        var features = commits.Where(c => c.StartsWith("feat:")).ToList();
        var fixes = commits.Where(c => c.StartsWith("fix:")).ToList();
        var perf = commits.Where(c => c.StartsWith("perf:")).ToList();
        
        var sb = new StringBuilder();
        sb.AppendLine($"## Release {toTag}");
        sb.AppendLine($"*{DateTime.Now:d MMMM yyyy}*");
        sb.AppendLine();
        
        if (features.Any())
        {
            sb.AppendLine("### ✨ ฟีเจอร์ใหม่");
            foreach (var f in features)
                sb.AppendLine($"- {f.Replace("feat:", "").Trim()}");
            sb.AppendLine();
        }
        
        if (fixes.Any())
        {
            sb.AppendLine("### 🐛 แก้ไขบัก");
            foreach (var f in fixes)
                sb.AppendLine($"- {f.Replace("fix:", "").Trim()}");
            sb.AppendLine();
        }
        
        if (perf.Any())
        {
            sb.AppendLine("### ⚡ ปรับปรุงประสิทธิภาพ");
            foreach (var f in perf)
                sb.AppendLine($"- {f.Replace("perf:", "").Trim()}");
        }
        
        return sb.ToString();
    }
    
    private Task<List<string>> GetCommitsAsync(string from, string to)
    {
        // Call git log or GitHub API
        return Task.FromResult(new List<string>
        {
            "feat: เพิ่มระบบติดตามออเดอร์แบบเรียลไทม์",
            "fix: แก้ไขปัญหาการคำนวณราคาส่วนลด",
            "perf: ลดเวลาโหลดหน้าแรก 40%"
        });
    }
}
```

---

## Step 860: Monitoring Dashboard

```csharp
// ============================================
// App Health Dashboard (in-app admin panel)
// ============================================

public class AppHealthViewModel : BaseViewModel
{
    private readonly IAnalytics _analytics;
    private readonly FeatureFlagService _flags;
    private readonly VersionService _version;
    
    public ObservableCollection<HealthMetric> Metrics { get; } = new();
    
    public async Task LoadAsync()
    {
        IsBusy = true;
        Metrics.Clear();
        
        Metrics.Add(new HealthMetric("App Version", AppInfo.VersionString, MetricStatus.Info));
        Metrics.Add(new HealthMetric("Environment", AppConfig.Current.ToString(), MetricStatus.Info));
        Metrics.Add(new HealthMetric("Platform", DeviceInfo.Platform.ToString(), MetricStatus.Info));
        
        var versionCheck = await _version.CheckVersionAsync();
        Metrics.Add(new HealthMetric(
            "Version Status",
            versionCheck.Status.ToString(),
            versionCheck.Status == VersionStatus.ForceUpdate ? MetricStatus.Critical :
            versionCheck.Status == VersionStatus.SoftUpdate ? MetricStatus.Warning :
            MetricStatus.Good));
        
        Metrics.Add(new HealthMetric(
            "Feature Flags",
            $"{_flags.GetValue("enabled_flags_count", 0)} active",
            MetricStatus.Info));
        
        Metrics.Add(new HealthMetric(
            "Connectivity",
            Connectivity.Current.NetworkAccess.ToString(),
            Connectivity.Current.NetworkAccess == NetworkAccess.Internet
                ? MetricStatus.Good : MetricStatus.Critical));
        
        IsBusy = false;
    }
}

public record HealthMetric(string Name, string Value, MetricStatus Status);
public enum MetricStatus { Good, Info, Warning, Critical }
```

---

## สรุป Part 86

ใน Part 86 เราได้เรียนรู้:

1. **Full CI/CD Pipeline** - GitHub Actions build/test/sign Android & iOS
2. **App Distribution** - Firebase App Distribution, TestFlight automated upload
3. **Crash Reporting** - Global exception handler, breadcrumbs, local queue
4. **Feature Flags** - Remote config with local defaults, cache, user targeting
5. **Version Management** - Minimum version check, force/soft update flow
6. **Analytics** - Event buffer, session tracking, batch flush
7. **Performance APM** - Named traces, slow threshold alerting
8. **Environment Config** - Dev/Staging/Production config records
9. **Release Automation** - Conventional commit notes generator
10. **Health Dashboard** - In-app admin panel with traffic-light status

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 86 | Steps 851-860*
