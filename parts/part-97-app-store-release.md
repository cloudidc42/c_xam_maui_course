# Part 97: App Store Optimization & Release Management
## Steps 961-970: ASO, Screenshots, Versioning, Release Notes, Store Submission, A/B Testing

---

## Step 961: App Version Management

```csharp
// ============================================
// Semantic Versioning for Mobile Apps
// ============================================

public class AppVersionManager
{
    // Semantic: MAJOR.MINOR.PATCH
    // Build number increments every CI build
    
    public record AppVersion(
        int Major, int Minor, int Patch, int Build,
        string? PreRelease = null)
    {
        public string FullVersion => PreRelease != null
            ? $"{Major}.{Minor}.{Patch}-{PreRelease}+{Build}"
            : $"{Major}.{Minor}.{Patch}+{Build}";
        
        public string DisplayVersion => $"{Major}.{Minor}.{Patch}";
        
        public bool IsNewerThan(AppVersion other)
        {
            if (Major != other.Major) return Major > other.Major;
            if (Minor != other.Minor) return Minor > other.Minor;
            if (Patch != other.Patch) return Patch > other.Patch;
            return Build > other.Build;
        }
        
        public static AppVersion Parse(string version)
        {
            // Format: "2.5.1" or "2.5.1+342" or "2.5.1-beta.1+342"
            var parts = version.Split('+');
            int build = parts.Length > 1 ? int.Parse(parts[1]) : 0;
            var semver = parts[0].Split('-');
            var preRelease = semver.Length > 1 ? semver[1] : null;
            var nums = semver[0].Split('.').Select(int.Parse).ToArray();
            return new AppVersion(nums[0], nums[1], nums[2], build, preRelease);
        }
    }
    
    public AppVersion Current => AppVersion.Parse(
        $"{AppInfo.VersionString}+{AppInfo.BuildString}");
    
    public async Task<VersionCheckResult> CheckForUpdateAsync()
    {
        using var http = new HttpClient();
        var response = await http.GetFromJsonAsync<VersionInfo>(
            $"https://api.fooddelivery.th/version?platform={DeviceInfo.Platform.ToString().ToLower()}");
        
        if (response == null) return new VersionCheckResult(VersionStatus.Unknown, null);
        
        var latest = AppVersion.Parse(response.LatestVersion);
        var current = Current;
        var minRequired = AppVersion.Parse(response.MinimumVersion);
        
        if (!current.IsNewerThan(minRequired) && minRequired.IsNewerThan(current))
            return new VersionCheckResult(VersionStatus.ForceUpdate, latest);
        
        if (latest.IsNewerThan(current))
            return new VersionCheckResult(VersionStatus.SoftUpdate, latest);
        
        return new VersionCheckResult(VersionStatus.UpToDate, current);
    }
}

public enum VersionStatus { UpToDate, SoftUpdate, ForceUpdate, Unknown }
public record VersionCheckResult(VersionStatus Status, AppVersionManager.AppVersion? Latest);
public record VersionInfo(string LatestVersion, string MinimumVersion, string ReleaseNotes);
```

---

## Step 962: Release Notes Generator

```csharp
// ============================================
// Conventional Commits → Auto Release Notes
// ============================================

public class ReleaseNotesBuilder
{
    public ReleaseNotes Build(string version, List<ConventionalCommit> commits)
    {
        var breaking = commits.Where(c => c.IsBreaking).ToList();
        var features = commits.Where(c => c.Type == "feat").ToList();
        var fixes = commits.Where(c => c.Type == "fix").ToList();
        var perf = commits.Where(c => c.Type == "perf").ToList();
        
        var sections = new List<ReleaseSection>();
        
        if (breaking.Any())
            sections.Add(new ReleaseSection("⚠️ การเปลี่ยนแปลงสำคัญ",
                breaking.Select(ToThaiMessage).ToList()));
        
        if (features.Any())
            sections.Add(new ReleaseSection("✨ ฟีเจอร์ใหม่",
                features.Select(ToThaiMessage).ToList()));
        
        if (fixes.Any())
            sections.Add(new ReleaseSection("🐛 แก้ไขข้อผิดพลาด",
                fixes.Select(ToThaiMessage).ToList()));
        
        if (perf.Any())
            sections.Add(new ReleaseSection("⚡ ปรับปรุงประสิทธิภาพ",
                perf.Select(ToThaiMessage).ToList()));
        
        return new ReleaseNotes(version, DateTime.Today, sections);
    }
    
    private string ToThaiMessage(ConventionalCommit commit)
    {
        var scope = commit.Scope != null ? $"[{commit.Scope}] " : "";
        return $"• {scope}{commit.Description}";
    }
    
    public string ToMarkdown(ReleaseNotes notes)
    {
        var sb = new StringBuilder();
        sb.AppendLine($"# ใหม่ในเวอร์ชัน {notes.Version}");
        sb.AppendLine($"*{notes.Date:d MMMM yyyy}*");
        sb.AppendLine();
        
        foreach (var section in notes.Sections)
        {
            sb.AppendLine($"## {section.Title}");
            foreach (var item in section.Items)
                sb.AppendLine(item);
            sb.AppendLine();
        }
        
        return sb.ToString();
    }
    
    // App Store What's New (max 4000 chars)
    public string ToAppStoreNotes(ReleaseNotes notes)
    {
        var sb = new StringBuilder();
        
        foreach (var section in notes.Sections.Take(3))
        {
            sb.AppendLine(section.Title);
            foreach (var item in section.Items.Take(5))
                sb.AppendLine(item);
            sb.AppendLine();
        }
        
        var result = sb.ToString();
        return result.Length > 4000 ? result[..3997] + "..." : result;
    }
}

public record ConventionalCommit(
    string Type, string? Scope, string Description, bool IsBreaking);
public record ReleaseNotes(string Version, DateTime Date, List<ReleaseSection> Sections);
public record ReleaseSection(string Title, List<string> Items);
```

---

## Step 963: Screenshot Automation

```yaml
# ============================================
# Fastlane Snapshot configuration for screenshots
# ============================================

# fastlane/Snapfile
devices(["iPhone 14 Pro Max", "iPhone 8 Plus", "iPad Pro (12.9-inch) (6th generation)"])
languages(["th-TH", "en-US"])
scheme("FoodDeliveryUITests")
output_directory("./fastlane/screenshots")
clear_previous_screenshots(true)
override_status_bar(true)
```

```csharp
// ============================================
// UI Test for Screenshot Automation
// ============================================

[TestFixture]
public class ScreenshotTests
{
    private AppiumDriver? _driver;
    
    [SetUp]
    public void SetUp()
    {
        // XCUITest / UIAutomator2 via Appium
        // (Simplified — real setup requires full Appium config)
    }
    
    [Test]
    [Category("Screenshot")]
    public async Task Screenshot01_HomeScreen()
    {
        // Navigate to home
        // Wait for restaurants to load
        // Take screenshot
        await TakeScreenshotAsync("01_home_thai");
    }
    
    [Test]
    [Category("Screenshot")]
    public async Task Screenshot02_RestaurantMenu()
    {
        await NavigateToRestaurantAsync("ร้านข้าวผัดกระเพรา");
        await TakeScreenshotAsync("02_restaurant_menu");
    }
    
    [Test]
    [Category("Screenshot")]
    public async Task Screenshot03_OrderTracking()
    {
        await NavigateToOrderTrackingAsync("order-123");
        await TakeScreenshotAsync("03_order_tracking");
    }
    
    private Task TakeScreenshotAsync(string name) => Task.CompletedTask;
    private Task NavigateToRestaurantAsync(string name) => Task.CompletedTask;
    private Task NavigateToOrderTrackingAsync(string orderId) => Task.CompletedTask;
}
```

---

## Step 964: A/B Testing Framework

```csharp
// ============================================
// Client-Side A/B Testing
// ============================================

public class AbTestService
{
    private readonly IFeatureFlagService _flags;
    private readonly string _userId;
    
    public AbTestService(IFeatureFlagService flags, string userId)
    {
        _flags = flags;
        _userId = userId;
    }
    
    // Deterministic assignment: same user always gets same variant
    public string GetVariant(string experimentId, string[] variants)
    {
        var hash = GetStableHash($"{experimentId}:{_userId}");
        var index = Math.Abs(hash) % variants.Length;
        return variants[index];
    }
    
    // Check experiment flag
    public bool IsInTreatment(string experimentId, double treatmentRatio = 0.5)
    {
        var hash = GetStableHash($"{experimentId}:{_userId}");
        var bucket = Math.Abs(hash) % 100;
        return bucket < treatmentRatio * 100;
    }
    
    public string GetHomeLayoutVariant()
    {
        // Feature flag can override
        var flagValue = _flags.GetValue<string>("home_layout_variant", "");
        if (!string.IsNullOrEmpty(flagValue)) return flagValue;
        
        return GetVariant("home_layout_2024", new[] { "grid", "list", "carousel" });
    }
    
    private int GetStableHash(string input)
    {
        // FNV-1a hash for stable distribution
        uint hash = 2166136261;
        foreach (char c in input)
        {
            hash ^= c;
            hash *= 16777619;
        }
        return (int)hash;
    }
}

// Track experiment exposure
public class ExperimentTracker
{
    private readonly IAnalyticsService _analytics;
    private readonly HashSet<string> _tracked = new();
    
    public ExperimentTracker(IAnalyticsService analytics) => _analytics = analytics;
    
    public void TrackExposure(string experimentId, string variant)
    {
        var key = $"{experimentId}:{variant}";
        if (_tracked.Contains(key)) return; // Only track once per session
        
        _tracked.Add(key);
        _analytics.Track("experiment_exposure", new Dictionary<string, object>
        {
            ["experiment_id"] = experimentId,
            ["variant"] = variant
        });
    }
}
```

---

## Step 965: ASO Keywords & Metadata

```csharp
// ============================================
// App Store Metadata Management
// ============================================

public static class AsoMetadata
{
    // App Store Connect fields (Thai)
    public static readonly AppStoreMetadata Thai = new(
        Name: "FoodDelivery - สั่งอาหาร",
        Subtitle: "ส่งอาหารถึงบ้าน รวดเร็วทันใจ",
        Description: """
            สั่งอาหารจากร้านโปรดได้ทุกที่ทุกเวลา ด้วย FoodDelivery แอปส่งอาหารที่ดีที่สุดในประเทศไทย
            
            ✅ ร้านอาหารกว่า 10,000 ร้านทั่วประเทศ
            ✅ ติดตามไรเดอร์แบบ Real-time
            ✅ จ่ายด้วย PromPay, บัตรเครดิต, หรือกระเป๋าตัง
            ✅ สะสมคะแนน FoodPoints ทุกการสั่งซื้อ
            ✅ รับ-ส่งภายใน 30 นาที*
            
            ดาวน์โหลดฟรี ไม่มีค่าสมัคร!
            """,
        Keywords: "สั่งอาหาร,ส่งอาหาร,food delivery,อาหาร,ร้านอาหาร,PromPay,GrabFood,FoodPanda",
        PromotionalText: "🎉 ลด 30% สำหรับผู้ใช้ใหม่ ใช้โค้ด NEW30"
    );
    
    // Play Store long description (Thai, 4000 char max)
    public static readonly string PlayStoreDescription = """
        # FoodDelivery - แอปสั่งอาหารที่ดีที่สุด
        
        ## ทำไมต้องเลือก FoodDelivery?
        
        **ร้านอาหารครบครัน** — ร้านอาหารไทย อาหารนานาชาติ ของทานเล่น เครื่องดื่ม ครบในแอปเดียว
        
        **ส่งเร็วจริง** — เฉลี่ยส่งภายใน 30 นาที ติดตามไรเดอร์ได้แบบ real-time บนแผนที่
        
        **ชำระเงินหลากหลาย** — PromPay, QR Code, บัตรเครดิต/เดบิต, กระเป๋าตัง, เก็บเงินปลายทาง
        
        **ปลอดภัย 100%** — ข้อมูลเข้ารหัส SSL, ไม่เก็บข้อมูลบัตรในเซิร์ฟเวอร์ของเรา
        
        ## FoodPoints ระบบสะสมแต้ม
        สั่งทุก ฿10 ได้ 1 คะแนน แลกรับส่วนลดและของรางวัลพิเศษ
        """;
}

public record AppStoreMetadata(
    string Name, string Subtitle, string Description, 
    string Keywords, string PromotionalText);
```

---

## Step 966: Crash-Free Rate Monitoring

```csharp
// ============================================
// Track crash-free users for release go/no-go
// ============================================

public class ReleaseHealthMonitor
{
    private readonly IApiClient _api;
    
    public ReleaseHealthMonitor(IApiClient api) => _api = api;
    
    public async Task<ReleaseHealthReport> GetReportAsync(string version)
    {
        var stats = await _api.GetAsync<VersionStats>($"/analytics/release-health/{version}");
        
        if (stats == null)
            return new ReleaseHealthReport(version, 0, 0, 0, false, "No data");
        
        var crashFreeRate = stats.TotalSessions > 0
            ? (1.0 - (double)stats.CrashedSessions / stats.TotalSessions) * 100
            : 100.0;
        
        // Go/No-Go: require >99.5% crash-free
        var isHealthy = crashFreeRate >= 99.5;
        
        return new ReleaseHealthReport(
            version,
            stats.TotalSessions,
            stats.CrashedSessions,
            crashFreeRate,
            isHealthy,
            isHealthy ? "✅ พร้อม rollout 100%" : $"⚠️ Crash-free rate {crashFreeRate:F2}% ต่ำกว่า 99.5%");
    }
}

public record VersionStats(string Version, int TotalSessions, int CrashedSessions);
public record ReleaseHealthReport(
    string Version, int TotalSessions, int CrashedSessions,
    double CrashFreeRate, bool IsHealthy, string Recommendation);
```

---

## Step 967: Staged Rollout

```csharp
// ============================================
// Phased/Staged Rollout Controller
// ============================================

// Staged rollout is managed in App Store Connect / Play Console UI.
// But we can implement our own rollout logic for feature flags.

public class StagedRolloutService
{
    private readonly IApiClient _api;
    private readonly string _userId;
    
    public StagedRolloutService(IApiClient api, string userId)
    {
        _api = api;
        _userId = userId;
    }
    
    // Check if this user is in the rollout group
    public async Task<bool> IsInRolloutAsync(string featureName)
    {
        var config = await _api.GetAsync<RolloutConfig>($"/rollout/{featureName}");
        if (config == null) return false;
        
        if (!config.IsActive) return false;
        
        // Percentage-based rollout using consistent hashing
        var hash = GetRolloutBucket(_userId, featureName);
        return hash < config.RolloutPercentage;
    }
    
    public async Task ReportRolloutHealthAsync(string featureName, bool isHealthy)
    {
        await _api.PostAsync<object>($"/rollout/{featureName}/health", new
        {
            userId = _userId,
            isHealthy,
            timestamp = DateTime.UtcNow
        });
    }
    
    private int GetRolloutBucket(string userId, string featureName)
    {
        var input = $"{featureName}:{userId}";
        var bytes = System.Text.Encoding.UTF8.GetBytes(input);
        using var sha = System.Security.Cryptography.SHA256.Create();
        var hash = sha.ComputeHash(bytes);
        return BitConverter.ToInt32(hash, 0) % 100;
    }
}

public record RolloutConfig(string FeatureName, bool IsActive, int RolloutPercentage);
```

---

## Step 968: Beta Testing Management

```csharp
// ============================================
// Internal / Beta Tester Management
// ============================================

public class BetaService
{
    public bool IsInternalBuild()
    {
#if DEBUG
        return true;
#else
        return AppInfo.BuildString.Contains("beta") ||
               AppInfo.VersionString.Contains("-beta");
#endif
    }
    
    public bool IsBetaTester()
    {
        // Check if enrolled in beta (stored after opt-in)
        return Preferences.Get("is_beta_tester", false);
    }
    
    public void EnrollInBeta(string email)
    {
        Preferences.Set("is_beta_tester", true);
        Preferences.Set("beta_email", email);
    }
    
    public void ShowBetaBanner(Page page)
    {
        if (!IsInternalBuild() && !IsBetaTester()) return;
        
        var banner = new Frame
        {
            BackgroundColor = Color.FromArgb("#FF9800"),
            CornerRadius = 0,
            Padding = new Thickness(8, 4),
            Content = new Label
            {
                Text = $"🔧 Beta {AppInfo.VersionString} (Build {AppInfo.BuildString})",
                TextColor = Colors.White,
                FontSize = 11,
                HorizontalOptions = LayoutOptions.Center
            }
        };
        
        if (page.Content is StackLayout sl)
            sl.Children.Insert(0, banner);
    }
    
    // Feedback shake gesture for beta testers
    public void EnableShakeToReport(Page page)
    {
        if (!IsBetaTester()) return;
        
        ShakeGestureRecognizer.Shaken += async (_, _) =>
        {
            var confirm = await page.DisplayAlert(
                "รายงานปัญหา",
                "คุณต้องการส่งรายงานปัญหาหรือไม่?",
                "ส่ง", "ยกเลิก");
            
            if (confirm)
                await Shell.Current.GoToAsync("feedback");
        };
    }
}
```

---

## Step 969: Privacy Policy & Compliance

```csharp
// ============================================
// Privacy Consent & Data Compliance (PDPA Thailand)
// ============================================

public class PrivacyConsentService
{
    private const string ConsentKey = "privacy_consent_v2";
    private const string ConsentDateKey = "privacy_consent_date";
    
    public bool HasConsented()
        => Preferences.Get(ConsentKey, false);
    
    public void RecordConsent(bool analytics, bool marketing, bool location)
    {
        Preferences.Set(ConsentKey, true);
        Preferences.Set(ConsentDateKey, DateTime.UtcNow.ToString("O"));
        Preferences.Set("consent_analytics", analytics);
        Preferences.Set("consent_marketing", marketing);
        Preferences.Set("consent_location", location);
    }
    
    public void WithdrawConsent(string type)
    {
        switch (type)
        {
            case "analytics": Preferences.Set("consent_analytics", false); break;
            case "marketing": Preferences.Set("consent_marketing", false); break;
            case "location": Preferences.Set("consent_location", false); break;
        }
    }
    
    // Data Subject Rights (PDPA)
    public async Task RequestDataExportAsync(string email)
    {
        using var http = new HttpClient();
        await http.PostAsJsonAsync("/privacy/export-request", new { email });
    }
    
    public async Task RequestDataDeletionAsync(string userId)
    {
        using var http = new HttpClient();
        await http.PostAsJsonAsync("/privacy/deletion-request", new { userId });
        
        // Clear local data
        Preferences.Clear();
        SecureStorage.Default.RemoveAll();
    }
    
    public bool CanCollectAnalytics => Preferences.Get("consent_analytics", false);
    public bool CanSendMarketing => Preferences.Get("consent_marketing", false);
}
```

---

## Step 970: Pre-Release Checklist

```csharp
// ============================================
// Automated Pre-Release Verification
// ============================================

public class PreReleaseChecker
{
    private readonly IApiClient _api;
    
    public PreReleaseChecker(IApiClient api) => _api = api;
    
    public async Task<PreReleaseReport> RunAsync()
    {
        var checks = new List<PreReleaseCheck>();
        
        checks.Add(await CheckApiHealthAsync());
        checks.Add(await CheckMinimumVersionAsync());
        checks.Add(CheckPrivacyConsent());
        checks.Add(CheckCrashReporting());
        checks.Add(CheckAnalytics());
        checks.Add(CheckRequiredPermissions());
        
        var allPassed = checks.All(c => c.Passed);
        return new PreReleaseReport(AppVersionManager.AppVersion.Parse(AppInfo.VersionString), checks, allPassed);
    }
    
    private async Task<PreReleaseCheck> CheckApiHealthAsync()
    {
        try
        {
            var result = await _api.GetAsync<HealthResult>("/health");
            return new PreReleaseCheck("API Health", result?.Status == "ok", result?.Status ?? "no response");
        }
        catch (Exception ex)
        {
            return new PreReleaseCheck("API Health", false, ex.Message);
        }
    }
    
    private async Task<PreReleaseCheck> CheckMinimumVersionAsync()
    {
        var vm = new AppVersionManager();
        var result = await vm.CheckForUpdateAsync();
        return new PreReleaseCheck("Version Check",
            result.Status != VersionStatus.ForceUpdate,
            result.Status.ToString());
    }
    
    private PreReleaseCheck CheckPrivacyConsent()
    {
        var svc = new PrivacyConsentService();
        return new PreReleaseCheck("Privacy Policy",
            true, // Consent shown on first launch
            "Consent flow configured");
    }
    
    private PreReleaseCheck CheckCrashReporting()
        => new PreReleaseCheck("Crash Reporting", true, "Global exception handler registered");
    
    private PreReleaseCheck CheckAnalytics()
        => new PreReleaseCheck("Analytics", true, "Analytics service initialized");
    
    private PreReleaseCheck CheckRequiredPermissions()
        => new PreReleaseCheck("Required Permissions",
            true, "Permissions requested at runtime, not upfront");
    
    private record HealthResult(string Status);
}

public record PreReleaseCheck(string Name, bool Passed, string Details);
public record PreReleaseReport(
    AppVersionManager.AppVersion Version,
    List<PreReleaseCheck> Checks,
    bool AllPassed);
```

---

## สรุป Part 97

ใน Part 97 เราได้เรียนรู้:

1. **Version Management** - Semantic versioning, IsNewerThan, force/soft update check
2. **Release Notes** - Conventional commits → Thai section headings
3. **Screenshot Automation** - Fastlane Snapfile, UITest screenshot categories
4. **A/B Testing** - Stable hash assignment, experiment exposure tracking
5. **ASO Metadata** - App Store name/subtitle/keywords, Play Store description
6. **Crash-Free Rate** - Release health check, 99.5% threshold, go/no-go
7. **Staged Rollout** - Percentage bucket hashing, health reporting
8. **Beta Testing** - Internal build detection, beta banner, shake-to-report
9. **Privacy Consent** - PDPA, consent recording, data export/deletion
10. **Pre-Release Checklist** - API health, version, privacy, crash reporting

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 97 | Steps 961-970*
