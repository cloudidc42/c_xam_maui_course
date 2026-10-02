# Part 53: CI/CD & DevOps
## Steps 521-530: GitHub Actions, Environments, Monitoring

---

## Step 521: Full CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore -c Release
      
      - name: Test
        run: dotnet test --no-build -c Release \
          --collect:"XPlat Code Coverage" \
          --results-directory ./coverage
      
      - name: Coverage Report
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/**/coverage.cobertura.xml
      
      - name: Security Scan
        run: dotnet list package --vulnerable
  
  android-build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install workloads
        run: dotnet workload install maui-android
      
      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > myapp.keystore
      
      - name: Build AAB
        run: |
          BUILD_NUMBER=$(git rev-list --count HEAD)
          dotnet publish -f net9.0-android -c Release \
            -p:ApplicationVersion=$BUILD_NUMBER \
            -p:AndroidPackageFormat=aab \
            -p:AndroidKeyStore=true \
            -p:AndroidSigningKeyStore=myapp.keystore \
            -p:AndroidSigningKeyAlias=myapp \
            -p:AndroidSigningKeyPass=${{ secrets.KEYSTORE_PASS }} \
            -p:AndroidSigningStorePass=${{ secrets.KEYSTORE_STORE_PASS }}
      
      - name: Upload AAB
        uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: '**/*-Signed.aab'
          retention-days: 30
  
  ios-build:
    needs: test
    runs-on: macos-14
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install workloads
        run: dotnet workload install maui-ios
      
      - name: Import certs
        uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_P12_PASSWORD }}
      
      - name: Build IPA
        run: |
          BUILD_NUMBER=$(git rev-list --count HEAD)
          dotnet publish -f net9.0-ios -c Release \
            -p:ApplicationVersion=$BUILD_NUMBER \
            -p:ArchiveOnBuild=true
      
      - name: Upload IPA
        uses: actions/upload-artifact@v4
        with:
          name: ios-release
          path: '**/*.ipa'
```

---

## Step 522: Environment Management

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  deploy-android:
    environment: ${{ inputs.environment }}
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: android-release
      
      - name: Deploy to ${{ inputs.environment }}
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_JSON }}
          packageName: com.mycompany.myapp
          releaseFiles: '*.aab'
          track: ${{ inputs.environment == 'production' && 'production' || 'internal' }}
          status: ${{ inputs.environment == 'production' && 'inProgress' || 'completed' }}
          userFraction: ${{ inputs.environment == 'production' && 0.1 || 1.0 }}
```

---

## Step 523: Version & Release Management

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  create-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Extract version
        id: version
        run: |
          echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Generate changelog
        id: changelog
        run: |
          PREV_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
          if [ -z "$PREV_TAG" ]; then
            COMMITS=$(git log --pretty=format:"- %s" | head -20)
          else
            COMMITS=$(git log $PREV_TAG..HEAD --pretty=format:"- %s")
          fi
          echo "CHANGES<<EOF" >> $GITHUB_OUTPUT
          echo "$COMMITS" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          name: "v${{ steps.version.outputs.VERSION }}"
          body: |
            ## What's Changed
            ${{ steps.changelog.outputs.CHANGES }}
          draft: false
          prerelease: false
```

---

## Step 524: Code Quality Gates

```yaml
# .github/workflows/quality.yml
name: Code Quality

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install dotnet-format
        run: dotnet tool install -g dotnet-format
      
      - name: Check formatting
        run: dotnet format --verify-no-changes --severity warn
      
      - name: Run Roslyn Analyzers
        run: dotnet build -p:TreatWarningsAsErrors=true
      
      - name: Check dependencies
        run: dotnet list package --outdated
  
  mutation-test:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Install Stryker
        run: dotnet tool install -g dotnet-stryker
      
      - name: Run mutation tests
        run: dotnet stryker --threshold-break 70
```

---

## Step 525: Monitoring & Alerting

```csharp
// ============================================
// App Performance Monitoring
// ============================================

public class AppMonitoringService
{
    private readonly HttpClient _http;
    private static readonly string Endpoint = 
        "https://monitoring.myapp.com/api/metrics";
    
    public AppMonitoringService(HttpClient http) => _http = http;
    
    // Track app start time
    private static DateTime _appStart = DateTime.UtcNow;
    
    public async Task TrackAppStartAsync()
    {
        var metric = new Metric
        {
            Name = "app.start",
            Value = (DateTime.UtcNow - _appStart).TotalMilliseconds,
            Tags = GetCommonTags()
        };
        
        await SendAsync(metric);
    }
    
    public async Task TrackScreenLoadAsync(string screen, double durationMs)
    {
        await SendAsync(new Metric
        {
            Name = "screen.load",
            Value = durationMs,
            Tags = GetCommonTags() with { Screen = screen }
        });
    }
    
    public async Task TrackApiCallAsync(
        string endpoint, int statusCode, double durationMs)
    {
        await SendAsync(new Metric
        {
            Name = "api.call",
            Value = durationMs,
            Tags = GetCommonTags() with
            {
                Endpoint = endpoint,
                StatusCode = statusCode.ToString()
            }
        });
    }
    
    public async Task TrackErrorAsync(Exception ex, string? context = null)
    {
        await SendAsync(new Metric
        {
            Name = "app.error",
            Value = 1,
            Tags = GetCommonTags() with
            {
                ErrorType = ex.GetType().Name,
                Context = context
            }
        });
    }
    
    private MetricTags GetCommonTags() => new()
    {
        AppVersion = AppInfo.Current.VersionString,
        Platform = DeviceInfo.Current.Platform.ToString(),
        OsVersion = DeviceInfo.Current.VersionString,
        DeviceModel = DeviceInfo.Current.Model
    };
    
    private async Task SendAsync(Metric metric)
    {
        try
        {
            await _http.PostAsJsonAsync(Endpoint, metric);
        }
        catch { /* Don't crash on monitoring failure */ }
    }
}

public class Metric
{
    public string Name { get; set; } = string.Empty;
    public double Value { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public MetricTags Tags { get; set; } = new();
}

public record MetricTags
{
    public string? AppVersion { get; init; }
    public string? Platform { get; init; }
    public string? OsVersion { get; init; }
    public string? DeviceModel { get; init; }
    public string? Screen { get; init; }
    public string? Endpoint { get; init; }
    public string? StatusCode { get; init; }
    public string? ErrorType { get; init; }
    public string? Context { get; init; }
}
```

---

## Step 526: Health Checks

```csharp
// ============================================
// App Health Check System
// ============================================

public interface IHealthCheck
{
    string Name { get; }
    Task<HealthCheckResult> CheckAsync(CancellationToken ct = default);
}

public record HealthCheckResult(bool IsHealthy, string? Details = null, TimeSpan? ResponseTime = null);

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly SQLiteConnection _db;
    public string Name => "database";
    
    public DatabaseHealthCheck(SQLiteConnection db) => _db = db;
    
    public async Task<HealthCheckResult> CheckAsync(CancellationToken ct = default)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            await Task.Run(() => _db.ExecuteScalar<int>("SELECT 1"), ct);
            sw.Stop();
            return new HealthCheckResult(true, "OK", sw.Elapsed);
        }
        catch (Exception ex)
        {
            return new HealthCheckResult(false, ex.Message, sw.Elapsed);
        }
    }
}

public class ApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _http;
    public string Name => "api";
    
    public ApiHealthCheck(HttpClient http) => _http = http;
    
    public async Task<HealthCheckResult> CheckAsync(CancellationToken ct = default)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var response = await _http.GetAsync("health", ct);
            sw.Stop();
            return new HealthCheckResult(
                response.IsSuccessStatusCode,
                response.StatusCode.ToString(),
                sw.Elapsed);
        }
        catch (Exception ex)
        {
            return new HealthCheckResult(false, ex.Message, sw.Elapsed);
        }
    }
}

public class HealthCheckService
{
    private readonly IEnumerable<IHealthCheck> _checks;
    
    public HealthCheckService(IEnumerable<IHealthCheck> checks) => _checks = checks;
    
    public async Task<AppHealthReport> CheckAllAsync()
    {
        var results = await Task.WhenAll(
            _checks.Select(async c => (c.Name, await c.CheckAsync())));
        
        return new AppHealthReport(results.ToDictionary(r => r.Name, r => r.Item2));
    }
}

public class AppHealthReport
{
    public Dictionary<string, HealthCheckResult> Checks { get; }
    public bool IsHealthy => Checks.Values.All(c => c.IsHealthy);
    public DateTime Timestamp { get; } = DateTime.UtcNow;
    
    public AppHealthReport(Dictionary<string, HealthCheckResult> checks) => Checks = checks;
    
    public override string ToString()
    {
        var lines = Checks.Select(c =>
            $"  {(c.Value.IsHealthy ? "✅" : "❌")} {c.Key}: {c.Value.Details}");
        return $"Health: {(IsHealthy ? "OK" : "DEGRADED")}\n{string.Join("\n", lines)}";
    }
}
```

---

## Step 527: Blue-Green Deployment Config

```csharp
// ============================================
// Environment-Specific Config
// ============================================

// appsettings.Development.json
/*
{
    "Api": { "BaseUrl": "https://dev.api.myapp.com/" },
    "Features": { "EnableDebugMenu": true }
}
*/

// appsettings.Staging.json
/*
{
    "Api": { "BaseUrl": "https://staging.api.myapp.com/" },
    "Features": { "EnableDebugMenu": true }
}
*/

// appsettings.Production.json
/*
{
    "Api": { "BaseUrl": "https://api.myapp.com/" },
    "Features": { "EnableDebugMenu": false }
}
*/

public class EnvironmentConfig
{
    public static string Current
    {
        get
        {
#if DEBUG
            return "Development";
#elif STAGING
            return "Staging";
#else
            return "Production";
#endif
        }
    }
    
    public static bool IsProduction => Current == "Production";
    public static bool IsDevelopment => Current == "Development";
}

// Conditional services registration
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        // ...
        
        if (EnvironmentConfig.IsProduction)
        {
            builder.Services.AddSingleton<ICrashReporter, FirebaseCrashReporter>();
            builder.Services.AddSingleton<IAnalytics, FirebaseAnalytics>();
        }
        else
        {
            builder.Services.AddSingleton<ICrashReporter, ConsoleCrashReporter>();
            builder.Services.AddSingleton<IAnalytics, DebugAnalytics>();
        }
        
        return builder.Build();
    }
}
```

---

## Step 528: Rollback Strategy

```csharp
// ============================================
// Feature Flag Rollback
// ============================================

public class EmergencyRollbackService
{
    private readonly RemoteFeatureFlagService _flags;
    private readonly IPreferences _prefs;
    
    public EmergencyRollbackService(RemoteFeatureFlagService flags, IPreferences prefs)
    {
        _flags = flags;
        _prefs = prefs;
    }
    
    // Kill switch - disable entire feature set
    public bool IsKillSwitchActivated => _flags.IsEnabled("kill_switch_v1_2_3");
    
    // Force app version check
    public async Task<bool> ShouldForceUpdateAsync(HttpClient http)
    {
        try
        {
            var response = await http.GetFromJsonAsync<VersionCheckResponse>(
                "api/version/minimum");
            
            if (response == null) return false;
            
            if (!Version.TryParse(response.MinimumVersion, out var minVer)) return false;
            if (!Version.TryParse(AppInfo.Current.VersionString, out var current)) return false;
            
            return current < minVer;
        }
        catch { return false; }
    }
    
    // Emergency API endpoint switch
    public string GetApiEndpoint()
    {
        // Check if we should use backup endpoint
        if (_flags.IsEnabled("use_backup_api"))
            return "https://backup.api.myapp.com/";
        
        return "https://api.myapp.com/";
    }
}
```

---

## Step 529: Automated E2E Testing in CI

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
  workflow_dispatch:

jobs:
  e2e-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Android Emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          script: |
            dotnet test tests/E2E --logger "trx;LogFileName=e2e-results.xml"
      
      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-results
          path: tests/E2E/e2e-results.xml
      
      - name: Publish results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: E2E Test Results
          path: tests/E2E/e2e-results.xml
          reporter: dotnet-trx
```

---

## Step 530: Deployment Dashboard

```csharp
// ============================================
// Deployment Status Tracking
// ============================================

public partial class DeploymentDashboardViewModel : ObservableObject
{
    private readonly HealthCheckService _health;
    private readonly HttpClient _http;
    
    [ObservableProperty] private AppHealthReport? _healthReport;
    [ObservableProperty] private string? _currentVersion;
    [ObservableProperty] private string? _latestVersion;
    [ObservableProperty] private bool _isUpdateAvailable;
    [ObservableProperty] private List<DeploymentRecord> _recentDeployments = new();
    
    public DeploymentDashboardViewModel(HealthCheckService health, HttpClient http)
    {
        _health = health;
        _http = http;
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        HealthReport = await _health.CheckAllAsync();
        CurrentVersion = AppInfo.Current.VersionString;
        
        await LoadVersionInfoAsync();
        await LoadDeploymentHistoryAsync();
    }
    
    private async Task LoadVersionInfoAsync()
    {
        var info = await _http.GetFromJsonAsync<VersionCheckResponse>(
            "api/version/latest");
        
        if (info != null)
        {
            LatestVersion = info.LatestVersion;
            IsUpdateAvailable = LatestVersion != CurrentVersion;
        }
    }
    
    private async Task LoadDeploymentHistoryAsync()
    {
        var history = await _http.GetFromJsonAsync<List<DeploymentRecord>>(
            "api/deployments/recent");
        
        RecentDeployments = history ?? new();
    }
}

public record DeploymentRecord(
    string Version,
    string Environment,
    DateTime DeployedAt,
    string DeployedBy,
    bool IsSuccess,
    string? Notes);
```

---

## สรุป Part 53

ใน Part 53 เราได้เรียนรู้:

1. **Full CI Pipeline** - Test, build Android + iOS
2. **Environment Management** - Staging, Production gates
3. **Version & Release** - Tags, auto-changelog
4. **Quality Gates** - Format check, analyzers
5. **Monitoring** - Custom metrics, error tracking
6. **Health Checks** - DB, API, composite report
7. **Blue-Green Config** - Per-environment settings
8. **Rollback Strategy** - Kill switch, force update
9. **E2E Testing in CI** - Scheduled Android emulator tests
10. **Deployment Dashboard** - Health + version + history

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 53 | Steps 521-530*
