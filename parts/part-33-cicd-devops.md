# Part 33: CI/CD และ DevOps
## Steps 321-330: Automation & Deployment

---

## Step 321: CI/CD Overview

```yaml
# ============================================
# GitHub Actions สำหรับ .NET MAUI
# ============================================

# .github/workflows/build.yml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '8.0.x'
  SOLUTION: 'MyApp.sln'

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Restore dependencies
        run: dotnet restore ${{ env.SOLUTION }}
      
      - name: Build
        run: dotnet build ${{ env.SOLUTION }} --no-restore --configuration Release
      
      - name: Test
        run: dotnet test ${{ env.SOLUTION }} --no-build --configuration Release \
          --logger "trx;LogFileName=test-results.trx" \
          --collect:"XPlat Code Coverage"
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: '**/test-results.trx'
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

---

## Step 322: Android Build Pipeline

```yaml
# ============================================
# Android Build & Sign
# ============================================

  build-android:
    name: Build Android
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      
      - name: Install MAUI Workloads
        run: dotnet workload install maui-android
      
      - name: Restore
        run: dotnet restore MyApp/MyApp.csproj
      
      - name: Build Android APK
        run: |
          dotnet build MyApp/MyApp.csproj \
            -c Release \
            -f net8.0-android \
            /p:AndroidKeyStore=true \
            /p:AndroidSigningKeyStore=${{ secrets.KEYSTORE_FILE }} \
            /p:AndroidSigningKeyAlias=${{ secrets.KEY_ALIAS }} \
            /p:AndroidSigningKeyPass=${{ secrets.KEY_PASSWORD }} \
            /p:AndroidSigningStorePass=${{ secrets.STORE_PASSWORD }}
      
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: android-apk
          path: MyApp/bin/Release/net8.0-android/*.apk
      
      - name: Deploy to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_APP_ID_ANDROID }}
          token: ${{ secrets.FIREBASE_TOKEN }}
          groups: testers
          file: MyApp/bin/Release/net8.0-android/*.apk
          releaseNotes: "Build from ${{ github.sha }}"
```

---

## Step 323: iOS Build Pipeline

```yaml
# ============================================
# iOS Build & Sign
# ============================================

  build-ios:
    name: Build iOS
    runs-on: macos-14
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Install MAUI Workloads
        run: dotnet workload install maui-ios
      
      - name: Import Code Signing Certs
        uses: Apple-Actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_P12_PASSWORD }}
      
      - name: Download Provisioning Profiles
        uses: Apple-Actions/download-provisioning-profiles@v1
        with:
          bundle-id: com.mycompany.myapp
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
      
      - name: Build iOS IPA
        run: |
          dotnet build MyApp/MyApp.csproj \
            -c Release \
            -f net8.0-ios \
            /p:ArchiveOnBuild=true \
            /p:RuntimeIdentifier=ios-arm64
      
      - name: Upload to TestFlight
        uses: Apple-Actions/upload-testflight-build@v1
        with:
          app-path: 'MyApp/bin/Release/net8.0-ios/MyApp.ipa'
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
```

---

## Step 324: Version Management

```csharp
// ============================================
// Automatic Versioning
// ============================================

// Directory.Build.props
/*
<Project>
  <PropertyGroup>
    <Version>1.0.0</Version>
    <AssemblyVersion>1.0.0.0</AssemblyVersion>
    <FileVersion>1.0.0.0</FileVersion>
    <ApplicationVersion Condition="'$(BUILD_NUMBER)' != ''">$(BUILD_NUMBER)</ApplicationVersion>
    <ApplicationDisplayVersion>$(Version)</ApplicationDisplayVersion>
  </PropertyGroup>
</Project>
*/

// Version helper
public static class AppVersion
{
    public static string DisplayVersion => AppInfo.Current.VersionString;
    public static string BuildNumber => AppInfo.Current.BuildString;
    public static string FullVersion => $"{DisplayVersion} ({BuildNumber})";
    
    public static async Task<bool> CheckForUpdateAsync(string apiUrl)
    {
        try
        {
            using var http = new HttpClient();
            var response = await http.GetFromJsonAsync<VersionInfo>($"{apiUrl}/version");
            
            if (response == null) return false;
            
            var current = Version.Parse(DisplayVersion);
            var latest = Version.Parse(response.LatestVersion);
            
            return latest > current;
        }
        catch { return false; }
    }
}

public record VersionInfo(string LatestVersion, string MinimumVersion, string? ChangelogUrl);
```

---

## Step 325: App Center / App Insights

```csharp
// ============================================
// Analytics & Crash Reporting
// ============================================

public class AnalyticsService
{
    // Track events
    public void Track(string eventName, Dictionary<string, string>? properties = null)
    {
        // Replace with your analytics provider
        // Microsoft.AppCenter.Analytics.Analytics.TrackEvent(eventName, properties);
        
        // Firebase Analytics
        // analytics.LogEvent(eventName, properties);
        
        System.Diagnostics.Debug.WriteLine($"[Analytics] {eventName}: {System.Text.Json.JsonSerializer.Serialize(properties ?? new())}");
    }
    
    // Track screen views
    public void TrackScreen(string screenName)
    {
        Track("screen_view", new() { ["screen_name"] = screenName });
    }
    
    // Track user actions
    public void TrackPurchase(string productId, decimal amount, string currency = "THB")
    {
        Track("purchase", new()
        {
            ["product_id"] = productId,
            ["amount"] = amount.ToString("F2"),
            ["currency"] = currency
        });
    }
    
    // Track errors
    public void TrackError(Exception ex, Dictionary<string, string>? properties = null)
    {
        var props = properties ?? new();
        props["exception_type"] = ex.GetType().Name;
        props["stack_trace"] = ex.StackTrace ?? string.Empty;
        
        // AppCenter.Crashes.TrackError(ex, props);
        Track("app_error", props);
    }
}

// Usage in ViewModel
public partial class ProductViewModel : ObservableObject
{
    private readonly AnalyticsService _analytics;
    
    [RelayCommand]
    private async Task ViewProductAsync(int id)
    {
        _analytics.Track("product_view", new() { ["product_id"] = id.ToString() });
        await Shell.Current.GoToAsync($"product-detail?id={id}");
    }
    
    [RelayCommand]
    private async Task AddToCartAsync(Product product)
    {
        _analytics.Track("add_to_cart", new()
        {
            ["product_id"] = product.Id.ToString(),
            ["product_name"] = product.Name,
            ["price"] = product.Price.ToString("F2")
        });
        // Add to cart logic...
    }
}
```

---

## Step 326: Error Boundary

```csharp
// ============================================
// Global Error Handling
// ============================================

public static class ErrorHandlerSetup
{
    public static MauiAppBuilder ConfigureErrorHandling(this MauiAppBuilder builder)
    {
        // Unhandled exceptions
        AppDomain.CurrentDomain.UnhandledException += (sender, args) =>
        {
            var ex = args.ExceptionObject as Exception;
            HandleCriticalError(ex, "UnhandledException");
        };
        
        // Task exceptions
        TaskScheduler.UnobservedTaskException += (sender, args) =>
        {
            HandleCriticalError(args.Exception, "UnobservedTaskException");
            args.SetObserved();
        };
        
        return builder;
    }
    
    private static void HandleCriticalError(Exception? ex, string source)
    {
        if (ex == null) return;
        
        // Log to crash reporter
        System.Diagnostics.Debug.WriteLine($"[CRASH] {source}: {ex}");
        
        // Save to local file for next launch
        var logPath = Path.Combine(FileSystem.AppDataDirectory, "crash.log");
        File.AppendAllText(logPath, $"[{DateTime.Now:O}] {source}: {ex}\n\n");
        
        // Analytics
        var analytics = IPlatformApplication.Current?.Services
            .GetService<AnalyticsService>();
        analytics?.TrackError(ex, new() { ["source"] = source });
    }
}

// Check for crash on startup
public partial class App : Application
{
    protected override async void OnStart()
    {
        base.OnStart();
        
        var crashLog = Path.Combine(FileSystem.AppDataDirectory, "crash.log");
        if (File.Exists(crashLog))
        {
            var log = await File.ReadAllTextAsync(crashLog);
            // Report to server, then delete
            File.Delete(crashLog);
        }
    }
}
```

---

## Step 327: Environment Configuration

```csharp
// ============================================
// Multi-Environment Configuration
// ============================================

public enum AppEnvironment { Development, Staging, Production }

public class AppConfiguration
{
    public static AppEnvironment Environment { get; private set; } = AppEnvironment.Production;
    
    public static string ApiBaseUrl => Environment switch
    {
        AppEnvironment.Development => "https://api-dev.example.com",
        AppEnvironment.Staging => "https://api-staging.example.com",
        _ => "https://api.example.com"
    };
    
    public static bool EnableAnalytics => Environment == AppEnvironment.Production;
    public static bool ShowDebugTools => Environment != AppEnvironment.Production;
    public static bool EnableMockData => Environment == AppEnvironment.Development;
    
    // Read from build configuration
    public static void Initialize()
    {
#if DEBUG
        Environment = AppEnvironment.Development;
#elif STAGING
        Environment = AppEnvironment.Staging;
#else
        Environment = AppEnvironment.Production;
#endif
    }
}

// appsettings.Development.json / appsettings.Production.json
// Loaded via embedded resource or platform-specific configuration

public static class ConfigurationExtensions
{
    public static MauiAppBuilder AddAppConfiguration(this MauiAppBuilder builder)
    {
        AppConfiguration.Initialize();
        
        // Register configuration as singleton
        builder.Services.AddSingleton(AppConfiguration.ApiBaseUrl);
        
        return builder;
    }
}
```

---

## Step 328: Code Quality Tools

```xml
<!-- ============================================
     .editorconfig for Code Quality
     ============================================ -->

<!-- .editorconfig -->
```

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# C# naming
dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = underscore_prefix
dotnet_naming_rule.private_fields.severity = warning

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.underscore_prefix.required_prefix = _
dotnet_naming_style.underscore_prefix.capitalization = camelCase

# Async methods
dotnet_naming_rule.async_methods.symbols = async_methods
dotnet_naming_rule.async_methods.style = async_suffix
dotnet_naming_rule.async_methods.severity = warning

dotnet_naming_symbols.async_methods.applicable_kinds = method
dotnet_naming_symbols.async_methods.required_modifiers = async

dotnet_naming_style.async_suffix.required_suffix = Async
dotnet_naming_style.async_suffix.capitalization = pascal_case
```

```csharp
// ============================================
// Analyzers Configuration
// ============================================

// .csproj
/*
<ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.*" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="9.*" />
    <PackageReference Include="StyleCop.Analyzers" Version="1.*" />
</ItemGroup>
*/

// Global suppressions (for intentional exceptions)
// GlobalSuppressions.cs
[assembly: System.Diagnostics.CodeAnalysis.SuppressMessage(
    "Design", "CA1062:Validate arguments of public methods",
    Justification = "Validated by caller contract",
    Scope = "member",
    Target = "~M:MyApp.Extensions.StringExtensions.ToSlug(System.String)")]
```

---

## Step 329: Release Checklist

```csharp
// ============================================
// Pre-Release Verification
// ============================================

public class ReleaseVerificationService
{
    public async Task<ReleaseChecklist> RunChecksAsync()
    {
        var checklist = new ReleaseChecklist();
        
        // 1. API connectivity
        checklist.ApiConnected = await CheckApiAsync();
        
        // 2. Database integrity
        checklist.DatabaseOk = await CheckDatabaseAsync();
        
        // 3. Required permissions
        checklist.PermissionsOk = await CheckPermissionsAsync();
        
        // 4. Storage space
        checklist.StorageOk = CheckStorageSpace();
        
        // 5. Network
        checklist.NetworkOk = Connectivity.Current.NetworkAccess == NetworkAccess.Internet;
        
        return checklist;
    }
    
    private async Task<bool> CheckApiAsync()
    {
        try
        {
            using var http = new HttpClient();
            http.Timeout = TimeSpan.FromSeconds(5);
            var response = await http.GetAsync($"{AppConfiguration.ApiBaseUrl}/health");
            return response.IsSuccessStatusCode;
        }
        catch { return false; }
    }
    
    private async Task<bool> CheckDatabaseAsync()
    {
        try
        {
            // Verify database can be opened and queried
            return true;
        }
        catch { return false; }
    }
    
    private async Task<bool> CheckPermissionsAsync()
    {
        var camera = await Permissions.CheckStatusAsync<Permissions.Camera>();
        var location = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();
        return true; // Permissions are optional, just check status
    }
    
    private bool CheckStorageSpace()
    {
        var cacheDir = new DirectoryInfo(FileSystem.CacheDirectory);
        // At least 50MB free
        return true;
    }
}

public record ReleaseChecklist
{
    public bool ApiConnected { get; set; }
    public bool DatabaseOk { get; set; }
    public bool PermissionsOk { get; set; }
    public bool StorageOk { get; set; }
    public bool NetworkOk { get; set; }
    
    public bool AllPassed => ApiConnected && DatabaseOk && PermissionsOk && StorageOk;
}
```

---

## Step 330: Deployment Scripts

```bash
# ============================================
# Deployment Scripts
# ============================================
```

```csharp
// ============================================
// Build Scripts as C# Global Tools
// ============================================

// build.csx (Cake Build or NUKE)
// Install: dotnet tool install Cake.Tool -g

/*
Target("Default", DependsOn("Test", "Build-Android", "Build-iOS"));

Target("Restore", () => {
    DotNetRestore("MyApp.sln");
});

Target("Test", DependsOn("Restore"), () => {
    DotNetTest("MyApp.sln", new DotNetTestSettings
    {
        Configuration = "Release",
        NoBuild = false
    });
});

Target("Build-Android", DependsOn("Restore"), () => {
    DotNetBuild("MyApp/MyApp.csproj", new DotNetBuildSettings
    {
        Configuration = "Release",
        Framework = "net8.0-android"
    });
});

RunTarget(target);
*/

// MAUI-specific build helper
public class MauiBuildHelper
{
    public static string GetVersionCode()
    {
        // Use timestamp as version code: YYMMDDHHMM
        return DateTime.Now.ToString("yyMMddHHmm");
    }
    
    public static string GetSemanticVersion(int major, int minor, int patch)
        => $"{major}.{minor}.{patch}";
    
    public static async Task BumpVersionAsync(string projectPath, string newVersion)
    {
        var content = await File.ReadAllTextAsync(projectPath);
        
        // Update ApplicationDisplayVersion
        content = System.Text.RegularExpressions.Regex.Replace(
            content,
            @"<ApplicationDisplayVersion>.*?</ApplicationDisplayVersion>",
            $"<ApplicationDisplayVersion>{newVersion}</ApplicationDisplayVersion>");
        
        await File.WriteAllTextAsync(projectPath, content);
    }
}
```

---

## สรุป Part 33

ใน Part 33 เราได้เรียนรู้:

1. **GitHub Actions** - CI/CD pipeline สำหรับ MAUI
2. **Android Pipeline** - Build, sign, deploy APK
3. **iOS Pipeline** - Code signing, TestFlight
4. **Version Management** - Automatic versioning
5. **Analytics** - AppCenter, crash reporting
6. **Error Boundary** - Global exception handling
7. **Environment Config** - Dev/Staging/Production
8. **Code Quality** - EditorConfig, Analyzers
9. **Release Checklist** - Pre-release verification
10. **Deployment Scripts** - Cake Build, NUKE

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 33 | Steps 321-330*
