# Part 52: Localization & Internationalization
## Steps 511-520: Thai, English, RTL, Plurals

---

## Step 511: Resource Files Setup

```csharp
// ============================================
// Resource-Based Localization
// ============================================

// Resources/Strings.resx (default English)
// Resources/Strings.th.resx (Thai)
// Resources/Strings.ar.resx (Arabic, RTL)

// Strings.resx entries:
// AppName = "MyShop"
// LoginTitle = "Sign In"
// LoginEmail = "Email"
// LoginPassword = "Password"
// LoginButton = "Sign In"
// ProductsTitle = "Products"
// CartEmpty = "Your cart is empty"
// OrderTotal = "Total: {0:C}"

// Strings.th.resx entries:
// AppName = "มายช็อป"
// LoginTitle = "เข้าสู่ระบบ"
// LoginEmail = "อีเมล"
// LoginPassword = "รหัสผ่าน"
// LoginButton = "เข้าสู่ระบบ"
// ProductsTitle = "สินค้า"
// CartEmpty = "ตะกร้าสินค้าของคุณว่างเปล่า"
// OrderTotal = "ยอดรวม: {0:C}"

public class LocalizationService
{
    private static System.Resources.ResourceManager? _resources;
    private System.Globalization.CultureInfo _culture;
    
    public LocalizationService()
    {
        _resources = new System.Resources.ResourceManager(
            "MyShop.Resources.Strings",
            typeof(LocalizationService).Assembly);
        
        _culture = System.Globalization.CultureInfo.CurrentUICulture;
    }
    
    public string this[string key] => Get(key);
    
    public string Get(string key)
        => _resources?.GetString(key, _culture) ?? $"[{key}]";
    
    public string GetFormat(string key, params object[] args)
    {
        var template = Get(key);
        return string.Format(_culture, template, args);
    }
    
    public void SetLanguage(string languageCode)
    {
        _culture = new System.Globalization.CultureInfo(languageCode);
        System.Globalization.CultureInfo.CurrentCulture = _culture;
        System.Globalization.CultureInfo.CurrentUICulture = _culture;
        
        OnLanguageChanged?.Invoke(languageCode);
    }
    
    public string CurrentLanguage => _culture.TwoLetterISOLanguageName;
    
    public event Action<string>? OnLanguageChanged;
}
```

---

## Step 512: MAUI Localization with XAML

```xml
<!-- Use markup extension for XAML localization -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:loc="clr-namespace:MyShop.Localization"
             x:Class="MyShop.Pages.LoginPage">
    
    <VerticalStackLayout Padding="24" Spacing="16">
        <Label Text="{loc:Translate LoginTitle}"
               FontSize="24"
               FontAttributes="Bold" />
        
        <Entry Placeholder="{loc:Translate LoginEmail}"
               Keyboard="Email" />
        
        <Entry Placeholder="{loc:Translate LoginPassword}"
               IsPassword="True" />
        
        <Button Text="{loc:Translate LoginButton}"
                Command="{Binding LoginCommand}" />
    </VerticalStackLayout>
</ContentPage>
```

```csharp
// Markup extension for XAML
[ContentProperty(nameof(Key))]
public class TranslateExtension : IMarkupExtension<string>
{
    public string Key { get; set; } = string.Empty;
    
    public string ProvideValue(IServiceProvider serviceProvider)
    {
        var localization = IPlatformApplication.Current!.Services
            .GetRequiredService<LocalizationService>();
        return localization[Key];
    }
    
    object IMarkupExtension.ProvideValue(IServiceProvider serviceProvider)
        => ProvideValue(serviceProvider);
}

// Dynamic language change
public partial class SettingsViewModel : ObservableObject
{
    private readonly LocalizationService _loc;
    
    [ObservableProperty] private string _selectedLanguage = "th";
    
    public List<LanguageOption> Languages { get; } =
    [
        new("th", "ไทย", "🇹🇭"),
        new("en", "English", "🇺🇸"),
        new("zh", "中文", "🇨🇳"),
        new("ja", "日本語", "🇯🇵"),
    ];
    
    public SettingsViewModel(LocalizationService loc)
    {
        _loc = loc;
        _selectedLanguage = _loc.CurrentLanguage;
    }
    
    partial void OnSelectedLanguageChanged(string value)
    {
        _loc.SetLanguage(value);
        // Reload app shell to apply changes
        App.Current!.MainPage = new AppShell();
    }
}

public record LanguageOption(string Code, string NativeName, string Flag);
```

---

## Step 513: Number & Date Formatting

```csharp
// ============================================
// Locale-Aware Formatting
// ============================================

public class LocaleFormatter
{
    private readonly System.Globalization.CultureInfo _culture;
    
    public LocaleFormatter(LocalizationService loc)
    {
        _culture = new System.Globalization.CultureInfo(loc.CurrentLanguage);
    }
    
    // Currency
    public string FormatCurrency(decimal amount, string? currencyCode = null)
    {
        var currency = currencyCode ?? GetDefaultCurrency();
        
        return currency switch
        {
            "THB" => $"฿{amount:N0}",
            "USD" => $"${amount:N2}",
            "EUR" => $"€{amount:N2}",
            _ => $"{amount:N2} {currency}"
        };
    }
    
    // Date
    public string FormatDate(DateTime date, string format = "short")
    {
        // Thai Buddhist calendar
        if (_culture.TwoLetterISOLanguageName == "th")
        {
            var thaiCalendar = new System.Globalization.ThaiBuddhistCalendar();
            var thaiYear = thaiCalendar.GetYear(date);
            
            return format switch
            {
                "short" => $"{date.Day}/{date.Month}/{thaiYear}",
                "long" => $"{date.Day} {GetThaiMonth(date.Month)} {thaiYear}",
                "time" => date.ToString("HH:mm", _culture),
                _ => date.ToString(format, _culture)
            };
        }
        
        return format switch
        {
            "short" => date.ToString("M/d/yyyy", _culture),
            "long" => date.ToString("MMMM d, yyyy", _culture),
            "time" => date.ToString("h:mm tt", _culture),
            _ => date.ToString(format, _culture)
        };
    }
    
    // Relative time
    public string FormatRelative(DateTime dateTime)
    {
        var diff = DateTime.Now - dateTime;
        
        if (_culture.TwoLetterISOLanguageName == "th")
        {
            return diff.TotalSeconds switch
            {
                < 60 => "เมื่อกี้",
                < 3600 => $"{(int)diff.TotalMinutes} นาทีที่แล้ว",
                < 86400 => $"{(int)diff.TotalHours} ชั่วโมงที่แล้ว",
                < 2592000 => $"{(int)diff.TotalDays} วันที่แล้ว",
                _ => FormatDate(dateTime, "short")
            };
        }
        
        return diff.TotalSeconds switch
        {
            < 60 => "just now",
            < 3600 => $"{(int)diff.TotalMinutes}m ago",
            < 86400 => $"{(int)diff.TotalHours}h ago",
            < 2592000 => $"{(int)diff.TotalDays}d ago",
            _ => FormatDate(dateTime, "short")
        };
    }
    
    private string GetDefaultCurrency() =>
        _culture.TwoLetterISOLanguageName == "th" ? "THB" : "USD";
    
    private static string GetThaiMonth(int month) =>
        month switch
        {
            1 => "มกราคม", 2 => "กุมภาพันธ์", 3 => "มีนาคม",
            4 => "เมษายน", 5 => "พฤษภาคม", 6 => "มิถุนายน",
            7 => "กรกฎาคม", 8 => "สิงหาคม", 9 => "กันยายน",
            10 => "ตุลาคม", 11 => "พฤศจิกายน", _ => "ธันวาคม"
        };
}
```

---

## Step 514: Pluralization

```csharp
// ============================================
// Plural Rules
// ============================================

public class PluralService
{
    private readonly LocalizationService _loc;
    
    public PluralService(LocalizationService loc) => _loc = loc;
    
    public string GetPlural(string key, int count)
    {
        var lang = _loc.CurrentLanguage;
        
        // Thai has no grammatical plural
        if (lang == "th")
            return string.Format(_loc.GetFormat(key + "_other", count), count);
        
        // English: 1 = singular, else plural
        var form = GetEnglishForm(count);
        var template = _loc.Get($"{key}_{form}");
        
        return string.Format(template, count);
    }
    
    private static string GetEnglishForm(int count)
        => count == 1 ? "one" : "other";
    
    // Plural resource keys:
    // ItemCount_one = "{0} item"
    // ItemCount_other = "{0} items"
    // DayCount_one = "{0} day"
    // DayCount_other = "{0} days"
    // ProductCount_other = "{0} สินค้า" (Thai - only "other")
    
    public string FormatItemCount(int count) => GetPlural("ItemCount", count);
    public string FormatDayCount(int count) => GetPlural("DayCount", count);
}
```

---

## Step 515: RTL Support

```csharp
// ============================================
// Right-to-Left Layout Support
// ============================================

public class LayoutDirectionService
{
    public FlowDirection GetFlowDirection(string? languageCode = null)
    {
        var lang = languageCode ?? System.Globalization.CultureInfo
            .CurrentUICulture.TwoLetterISOLanguageName;
        
        var rtlLanguages = new HashSet<string> { "ar", "he", "fa", "ur", "yi" };
        
        return rtlLanguages.Contains(lang)
            ? FlowDirection.RightToLeft
            : FlowDirection.LeftToRight;
    }
    
    public bool IsRtl => GetFlowDirection() == FlowDirection.RightToLeft;
}

// Apply to Shell/App
public partial class AppShell : Shell
{
    private readonly LayoutDirectionService _layout;
    
    public AppShell(LayoutDirectionService layout)
    {
        InitializeComponent();
        _layout = layout;
        ApplyFlowDirection();
    }
    
    private void ApplyFlowDirection()
    {
        FlowDirection = _layout.GetFlowDirection();
    }
}
```

```xml
<!-- RTL-aware layout -->
<Grid ColumnDefinitions="Auto,*">
    <!-- Icon -->
    <Image Grid.Column="0"
           Source="product.png"
           WidthRequest="50"
           HeightRequest="50" />
    
    <!-- Content adapts to RTL automatically -->
    <VerticalStackLayout Grid.Column="1" Spacing="4">
        <Label Text="{Binding Name}" />
        <Label Text="{Binding Price}" />
    </VerticalStackLayout>
</Grid>

<!-- Explicit RTL support -->
<ContentPage FlowDirection="{Binding FlowDirection}">
    <!-- All content mirrors for RTL -->
</ContentPage>
```

---

## Step 516: Language Detection

```csharp
// ============================================
// Auto Language Detection
// ============================================

public class LanguageDetector
{
    public string DetectLanguage()
    {
        // Use device language
        var deviceLang = System.Globalization.CultureInfo
            .CurrentUICulture.TwoLetterISOLanguageName;
        
        var supported = new[] { "th", "en", "zh", "ja", "ko" };
        
        return supported.Contains(deviceLang) ? deviceLang : "en";
    }
    
    public string GetSavedLanguage(IPreferences prefs)
    {
        var saved = prefs.Get("app_language", string.Empty);
        return string.IsNullOrEmpty(saved) ? DetectLanguage() : saved;
    }
    
    public void SaveLanguage(string lang, IPreferences prefs)
        => prefs.Set("app_language", lang);
}

// App initialization
public partial class App : Application
{
    public App(LocalizationService loc, LanguageDetector detector, IPreferences prefs)
    {
        // Set language before InitializeComponent
        var lang = detector.GetSavedLanguage(prefs);
        loc.SetLanguage(lang);
        
        InitializeComponent();
        MainPage = new AppShell();
    }
}
```

---

## Step 517: Localization Testing

```csharp
// ============================================
// Localization Tests
// ============================================

public class LocalizationTests
{
    [Theory]
    [InlineData("th", "เข้าสู่ระบบ")]
    [InlineData("en", "Sign In")]
    public void GetString_ReturnsCorrectTranslation(string lang, string expected)
    {
        var loc = new LocalizationService();
        loc.SetLanguage(lang);
        
        var result = loc["LoginTitle"];
        
        result.Should().Be(expected);
    }
    
    [Fact]
    public void FormatCurrency_Thai_UsesThaiFormat()
    {
        var loc = new LocalizationService();
        loc.SetLanguage("th");
        var formatter = new LocaleFormatter(loc);
        
        var result = formatter.FormatCurrency(1299.99m, "THB");
        
        result.Should().Be("฿1,300");
    }
    
    [Fact]
    public void FormatDate_Thai_UsesBuddhistCalendar()
    {
        var loc = new LocalizationService();
        loc.SetLanguage("th");
        var formatter = new LocaleFormatter(loc);
        
        var date = new DateTime(2024, 1, 15);
        var result = formatter.FormatDate(date, "short");
        
        result.Should().Contain("2567"); // 2024 + 543 = 2567
    }
    
    [Theory]
    [InlineData("en", 1, "1 item")]
    [InlineData("en", 5, "5 items")]
    [InlineData("th", 1, "1 สินค้า")]
    [InlineData("th", 5, "5 สินค้า")]
    public void Plural_CorrectForm(string lang, int count, string expected)
    {
        var loc = new LocalizationService();
        loc.SetLanguage(lang);
        var plural = new PluralService(loc);
        
        var result = plural.FormatItemCount(count);
        
        result.Should().Be(expected);
    }
    
    [Theory]
    [InlineData("ar", FlowDirection.RightToLeft)]
    [InlineData("he", FlowDirection.RightToLeft)]
    [InlineData("th", FlowDirection.LeftToRight)]
    [InlineData("en", FlowDirection.LeftToRight)]
    public void FlowDirection_CorrectForLanguage(string lang, FlowDirection expected)
    {
        var service = new LayoutDirectionService();
        var result = service.GetFlowDirection(lang);
        result.Should().Be(expected);
    }
}
```

---

## Step 518: App Store Localization

```xml
<!-- iOS: Localizable.strings (Thai) -->
<!-- File: th.lproj/Localizable.strings -->
/*
"NSCameraUsageDescription" = "แอปใช้กล้องเพื่อถ่ายรูปสินค้า";
"NSPhotoLibraryUsageDescription" = "แอปเข้าถึงรูปภาพเพื่อเลือกรูปสินค้า";
"NSLocationWhenInUseUsageDescription" = "แอปใช้ตำแหน่งเพื่อค้นหาร้านค้าใกล้คุณ";
*/

<!-- Android: res/values-th/strings.xml -->
<resources>
    <string name="app_name">มายช็อป</string>
    <string name="permission_camera">แอปต้องการเข้าถึงกล้อง</string>
    <string name="permission_location">แอปต้องการทราบตำแหน่งของคุณ</string>
    
    <!-- Plurals -->
    <plurals name="item_count">
        <item quantity="other">%d รายการ</item>
    </plurals>
</resources>
```

---

## Step 519: Content Negotiation

```csharp
// ============================================
// API Content Language Negotiation
// ============================================

public class LocalizedApiClient
{
    private readonly HttpClient _http;
    private readonly LocalizationService _loc;
    
    public LocalizedApiClient(HttpClient http, LocalizationService loc)
    {
        _http = http;
        _loc = loc;
    }
    
    public async Task<T?> GetAsync<T>(string url, CancellationToken ct = default)
    {
        using var request = new HttpRequestMessage(HttpMethod.Get, url);
        
        // Tell API which language to return content in
        request.Headers.AcceptLanguage.ParseAdd(_loc.CurrentLanguage);
        request.Headers.AcceptLanguage.ParseAdd("en;q=0.8"); // fallback
        
        var response = await _http.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<T>(ct);
    }
}

// Product DTO with locale-aware content
public record LocalizedProductDto(
    int Id,
    string Name,          // API returns in requested language
    string Description,   // API returns in requested language
    decimal Price,
    string Currency,
    int Stock);
```

---

## Step 520: Advanced Formatting

```csharp
// ============================================
// Thai Number Words
// ============================================

public class ThaiNumberFormatter
{
    private static readonly string[] Digits = 
        { "", "หนึ่ง", "สอง", "สาม", "สี่", "ห้า", "หก", "เจ็ด", "แปด", "เก้า" };
    
    private static readonly string[] Positions =
        { "", "สิบ", "ร้อย", "พัน", "หมื่น", "แสน", "ล้าน" };
    
    public static string ToWords(long number)
    {
        if (number == 0) return "ศูนย์";
        if (number < 0) return "ลบ" + ToWords(-number);
        
        if (number >= 1_000_000)
        {
            var millions = number / 1_000_000;
            var remainder = number % 1_000_000;
            return ToWords(millions) + "ล้าน" + (remainder > 0 ? ToWords(remainder) : "");
        }
        
        var result = string.Empty;
        var digits = number.ToString();
        var len = digits.Length;
        
        for (int i = 0; i < len; i++)
        {
            var d = digits[i] - '0';
            var pos = len - i - 1;
            
            if (d == 0) continue;
            if (pos == 1 && d == 1) { result += "สิบ"; continue; }
            if (pos == 1 && d == 2) { result += "ยี่สิบ"; continue; }
            
            result += Digits[d] + Positions[pos];
        }
        
        return result;
    }
    
    public static string ToBahtWords(decimal amount)
    {
        var baht = (long)Math.Floor(amount);
        var satang = (int)Math.Round((amount - baht) * 100);
        
        var result = ToWords(baht) + "บาท";
        if (satang > 0)
            result += ToWords(satang) + "สตางค์";
        else
            result += "ถ้วน";
        
        return result;
    }
}

// Usage
// ThaiNumberFormatter.ToBahtWords(1500.50m)
// → "หนึ่งพันห้าร้อยบาทห้าสิบสตางค์"
```

---

## สรุป Part 52

ใน Part 52 เราได้เรียนรู้:

1. **Resource Files** - .resx files, ResourceManager
2. **XAML Markup Extension** - `{loc:Translate Key}`
3. **Number/Date Formatting** - Buddhist calendar, currency
4. **Plural Rules** - Thai (no plural) vs English
5. **RTL Support** - FlowDirection for Arabic/Hebrew
6. **Language Detection** - Auto-detect from device
7. **Localization Testing** - Theory tests for each language
8. **App Store** - iOS/Android localized strings
9. **API Content Negotiation** - Accept-Language header
10. **Thai Number Words** - บาทสตางค์ in words

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 52 | Steps 511-520*
