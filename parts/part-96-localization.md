# Part 96: Advanced Localization & Internationalization
## Steps 951-960: Thai Locale, RTL, Currency, Date Formatting, Plurals, Cultural Adaptation

---

## Step 951: Resource-Based Localization

```csharp
// ============================================
// .resx Localization for Thai & English
// ============================================

// Resources/Strings.resx (English default)
// Resources/Strings.th.resx (Thai)

// Auto-generated resource class:
// public static string OrderConfirmed => ResourceManager.GetString("OrderConfirmed");

// MauiProgram.cs:
// builder.Services.AddLocalization(o => o.ResourcesPath = "Resources");

// In AppShell.xaml.cs:
public class LocalizationService
{
    private const string LanguageKey = "app_language";
    
    public string CurrentLanguage
        => Preferences.Get(LanguageKey, "th"); // Default Thai
    
    public void SetLanguage(string languageCode)
    {
        Preferences.Set(LanguageKey, languageCode);
        
        var culture = new System.Globalization.CultureInfo(languageCode);
        System.Globalization.CultureInfo.DefaultThreadCurrentCulture = culture;
        System.Globalization.CultureInfo.DefaultThreadCurrentUICulture = culture;
        
        Thread.CurrentThread.CurrentCulture = culture;
        Thread.CurrentThread.CurrentUICulture = culture;
        
        // Notify app to reload strings
        WeakReferenceMessenger.Default.Send(new LanguageChangedMessage(languageCode));
    }
    
    public void RestoreLanguage()
        => SetLanguage(CurrentLanguage);
}

public record LanguageChangedMessage(string LanguageCode);
```

---

## Step 952: Thai Date & Time Formatting

```csharp
// ============================================
// Thai Buddhist Era, Thai numerals, relative time
// ============================================

public static class ThaiDateFormatter
{
    private static readonly System.Globalization.CultureInfo ThaiCulture
        = new System.Globalization.CultureInfo("th-TH");
    
    // Thai Buddhist Era (พ.ศ.)
    public static string ToBuddhistEra(DateTime date)
    {
        var buddhistYear = date.Year + 543;
        return $"{date.Day} {GetThaiMonth(date.Month)} {buddhistYear}";
    }
    
    public static string ToThaiShortDate(DateTime date)
        => date.ToString("d MMM yyyy", ThaiCulture);
    
    public static string ToThaiLongDate(DateTime date)
        => date.ToString("dddd ที่ d MMMM พ.ศ. " + (date.Year + 543), ThaiCulture);
    
    public static string ToRelativeTime(DateTime date)
    {
        var diff = DateTime.Now - date;
        
        return diff.TotalSeconds switch
        {
            < 60 => "เมื่อกี้",
            < 3600 => $"{(int)diff.TotalMinutes} นาทีที่แล้ว",
            < 86400 => $"{(int)diff.TotalHours} ชั่วโมงที่แล้ว",
            < 86400 * 7 => $"{(int)diff.TotalDays} วันที่แล้ว",
            < 86400 * 30 => $"{(int)(diff.TotalDays / 7)} สัปดาห์ที่แล้ว",
            < 86400 * 365 => $"{(int)(diff.TotalDays / 30)} เดือนที่แล้ว",
            _ => $"{(int)(diff.TotalDays / 365)} ปีที่แล้ว"
        };
    }
    
    // Time of day greeting
    public static string GetThaiTimeGreeting()
    {
        var hour = DateTime.Now.Hour;
        return hour switch
        {
            >= 5 and < 12 => "สวัสดีตอนเช้า",
            >= 12 and < 17 => "สวัสดีตอนบ่าย",
            >= 17 and < 20 => "สวัสดีตอนเย็น",
            _ => "สวัสดีตอนกลางคืน"
        };
    }
    
    private static string GetThaiMonth(int month) => month switch
    {
        1 => "มกราคม", 2 => "กุมภาพันธ์", 3 => "มีนาคม",
        4 => "เมษายน", 5 => "พฤษภาคม", 6 => "มิถุนายน",
        7 => "กรกฎาคม", 8 => "สิงหาคม", 9 => "กันยายน",
        10 => "ตุลาคม", 11 => "พฤศจิกายน", 12 => "ธันวาคม",
        _ => ""
    };
}
```

---

## Step 953: Thai Currency & Number Formatting

```csharp
// ============================================
// Thai Baht Formatting
// ============================================

public static class ThaiCurrencyFormatter
{
    private static readonly System.Globalization.CultureInfo ThaiCulture
        = new System.Globalization.CultureInfo("th-TH");
    
    public static string FormatBaht(decimal amount, bool showDecimals = false)
    {
        if (showDecimals)
            return amount.ToString("C2", ThaiCulture); // ฿1,234.50
        
        return $"฿{amount:N0}"; // ฿1,234
    }
    
    // Short form for large amounts
    public static string FormatShort(decimal amount) => amount switch
    {
        >= 1_000_000 => $"฿{amount / 1_000_000:N1}ล.",
        >= 1_000 => $"฿{amount / 1_000:N1}K",
        _ => $"฿{amount:N0}"
    };
    
    // Verbal Thai Baht
    public static string ToThaiWords(decimal amount)
    {
        var baht = (int)Math.Floor(amount);
        var satang = (int)Math.Round((amount - baht) * 100);
        
        var result = $"{ThaiNumberWords(baht)}บาท";
        if (satang > 0)
            result += $"{ThaiNumberWords(satang)}สตางค์";
        else
            result += "ถ้วน";
        
        return result;
    }
    
    private static string ThaiNumberWords(int number)
    {
        if (number == 0) return "ศูนย์";
        
        var units = new[] { "", "หนึ่ง", "สอง", "สาม", "สี่", "ห้า", "หก", "เจ็ด", "แปด", "เก้า" };
        var positions = new[] { "", "สิบ", "ร้อย", "พัน", "หมื่น", "แสน", "ล้าน" };
        
        var result = "";
        var digits = number.ToString().Reverse().Select(c => c - '0').ToArray();
        
        for (int i = digits.Length - 1; i >= 0; i--)
        {
            var digit = digits[i];
            if (digit == 0) continue;
            
            if (i == 1 && digit == 1) result += "สิบ";
            else if (i == 1 && digit == 2) result += "ยี่สิบ";
            else if (i == 0 && digit == 1 && digits.Length > 1) result += "เอ็ด";
            else result += units[digit] + positions[i];
        }
        
        return result;
    }
}
```

---

## Step 954: Plural Forms

```csharp
// ============================================
// Thai doesn't have plural forms, but other locales do
// ============================================

public class PluralFormatter
{
    private readonly string _languageCode;
    
    public PluralFormatter(string languageCode = "th")
        => _languageCode = languageCode;
    
    public string Format(string key, int count)
    {
        return _languageCode switch
        {
            "th" => GetThai(key, count),  // Thai: no plural
            "en" => GetEnglish(key, count),
            _ => GetThai(key, count)
        };
    }
    
    private string GetThai(string key, int count) => key switch
    {
        "items" => $"{count} รายการ",
        "orders" => $"{count} คำสั่งซื้อ",
        "restaurants" => $"{count} ร้านอาหาร",
        "minutes" => $"{count} นาที",
        "points" => $"{count} คะแนน",
        _ => $"{count} {key}"
    };
    
    private string GetEnglish(string key, int count) => key switch
    {
        "items" => count == 1 ? "1 item" : $"{count} items",
        "orders" => count == 1 ? "1 order" : $"{count} orders",
        "restaurants" => count == 1 ? "1 restaurant" : $"{count} restaurants",
        "minutes" => count == 1 ? "1 minute" : $"{count} minutes",
        "points" => count == 1 ? "1 point" : $"{count} points",
        _ => $"{count} {key}"
    };
}
```

---

## Step 955: Thai Phone Number Format

```csharp
// ============================================
// Thai Phone Formatting & Validation
// ============================================

public static class ThaiPhoneFormatter
{
    // Format: 0x-xxxx-xxxx or 0xx-xxx-xxxx
    public static string Format(string rawPhone)
    {
        var digits = new string(rawPhone.Where(char.IsDigit).ToArray());
        
        return digits.Length switch
        {
            10 when digits[1] is '2' or '3' or '4' or '5' or '7' => // Bangkok landline
                $"{digits[0..2]}-{digits[2..6]}-{digits[6..10]}",
            10 when digits[1] is '6' or '8' or '9' => // Mobile
                $"{digits[0..3]}-{digits[3..6]}-{digits[6..10]}",
            9 => // Older landline
                $"{digits[0..2]}-{digits[2..5]}-{digits[5..9]}",
            _ => rawPhone
        };
    }
    
    // Mask middle digits for display: 081-xxx-9876
    public static string Mask(string phone)
    {
        var formatted = Format(phone);
        var parts = formatted.Split('-');
        if (parts.Length == 3)
            return $"{parts[0]}-xxx-{parts[2]}";
        return formatted;
    }
    
    // Validate Thai mobile (06x, 08x, 09x)
    public static bool IsValidMobile(string phone)
    {
        var digits = new string(phone.Where(char.IsDigit).ToArray());
        return Regex.IsMatch(digits, @"^0[689]\d{8}$");
    }
    
    // Normalize to E.164 (+66)
    public static string ToE164(string phone)
    {
        var digits = new string(phone.Where(char.IsDigit).ToArray());
        if (digits.StartsWith("0"))
            digits = "66" + digits[1..];
        return $"+{digits}";
    }
}
```

---

## Step 956: Thai Address Formatting

```csharp
// ============================================
// Thai Address Structure
// ============================================

public record ThaiAddress(
    string HouseNumber,
    string? MooNumber,
    string? SoiName,
    string RoadName,
    string Subdistrict, // ตำบล/แขวง
    string District,    // อำเภอ/เขต
    string Province,    // จังหวัด
    string PostalCode
);

public static class ThaiAddressFormatter
{
    public static string Format(ThaiAddress address)
    {
        var parts = new List<string>();
        
        parts.Add($"เลขที่ {address.HouseNumber}");
        
        if (!string.IsNullOrEmpty(address.MooNumber))
            parts.Add($"หมู่ {address.MooNumber}");
        
        if (!string.IsNullOrEmpty(address.SoiName))
            parts.Add($"ซอย{address.SoiName}");
        
        parts.Add($"ถนน{address.RoadName}");
        
        var isMetropolitan = address.Province is "กรุงเทพมหานคร";
        
        parts.Add(isMetropolitan
            ? $"แขวง{address.Subdistrict}"
            : $"ตำบล{address.Subdistrict}");
        
        parts.Add(isMetropolitan
            ? $"เขต{address.District}"
            : $"อำเภอ{address.District}");
        
        parts.Add(address.Province);
        parts.Add(address.PostalCode);
        
        return string.Join(" ", parts);
    }
    
    public static string FormatShort(ThaiAddress address)
        => $"{address.HouseNumber} {address.RoadName} {address.District} {address.Province} {address.PostalCode}";
    
    // Province → region mapping for delivery zone pricing
    public static ThaiRegion GetRegion(string province) => province switch
    {
        "กรุงเทพมหานคร" or "นนทบุรี" or "ปทุมธานี" or "สมุทรปราการ" => ThaiRegion.Bangkok,
        "เชียงใหม่" or "เชียงราย" or "ลำปาง" => ThaiRegion.North,
        "ขอนแก่น" or "นครราชสีมา" or "อุดรธานี" => ThaiRegion.Northeast,
        "สุราษฎร์ธานี" or "ภูเก็ต" or "สงขลา" => ThaiRegion.South,
        _ => ThaiRegion.Central
    };
}

public enum ThaiRegion { Bangkok, Central, North, Northeast, South }
```

---

## Step 957: Locale-Aware Sorting

```csharp
// ============================================
// Thai-aware string comparison and sorting
// ============================================

public class ThaiStringComparer : IComparer<string>
{
    private static readonly System.Globalization.CompareInfo ThaiComparer
        = System.Globalization.CultureInfo.GetCultureInfo("th-TH").CompareInfo;
    
    public int Compare(string? x, string? y)
        => ThaiComparer.Compare(x, y, System.Globalization.CompareOptions.None);
    
    public static readonly ThaiStringComparer Instance = new();
}

// Usage:
// var sorted = restaurants.OrderBy(r => r.Name, ThaiStringComparer.Instance).ToList();

// Thai consonant class ordering (for dictionary-style UI)
public static class ThaiAlphabet
{
    // Thai consonants in traditional dictionary order
    private static readonly string[] ThaiOrder = {
        "ก", "ข", "ฃ", "ค", "ฅ", "ฆ", "ง",
        "จ", "ฉ", "ช", "ซ", "ฌ", "ญ",
        "ฎ", "ฏ", "ฐ", "ฑ", "ฒ", "ณ",
        "ด", "ต", "ถ", "ท", "ธ", "น",
        "บ", "ป", "ผ", "ฝ", "พ", "ฟ", "ภ", "ม",
        "ย", "ร", "ล", "ว",
        "ศ", "ษ", "ส", "ห", "ฬ", "อ", "ฮ"
    };
    
    public static string GetSection(string name)
    {
        if (string.IsNullOrEmpty(name)) return "#";
        var first = name[0].ToString();
        return ThaiOrder.Contains(first) ? first : "#";
    }
    
    public static List<AlphabetSection<T>> GroupByThaiAlphabet<T>(
        IEnumerable<T> items, Func<T, string> nameSelector)
    {
        return items
            .OrderBy(i => nameSelector(i), ThaiStringComparer.Instance)
            .GroupBy(i => GetSection(nameSelector(i)))
            .OrderBy(g => Array.IndexOf(ThaiOrder, g.Key))
            .Select(g => new AlphabetSection<T>(g.Key, g.ToList()))
            .ToList();
    }
}

public record AlphabetSection<T>(string Letter, List<T> Items);
```

---

## Step 958: RTL Layout Support

```csharp
// ============================================
// Arabic/Hebrew RTL layout (for multi-regional apps)
// ============================================

public static class DirectionHelper
{
    private static readonly HashSet<string> RtlLanguages = new()
        { "ar", "he", "fa", "ur", "ks" };
    
    public static bool IsRtl(string languageCode)
        => RtlLanguages.Contains(languageCode.Split('-')[0].ToLower());
    
    public static FlowDirection GetFlowDirection(string languageCode)
        => IsRtl(languageCode) ? FlowDirection.RightToLeft : FlowDirection.LeftToRight;
    
    public static void ApplyFlowDirection(Page page, string languageCode)
    {
        page.FlowDirection = GetFlowDirection(languageCode);
    }
    
    // Mirror padding for RTL
    public static Thickness MirrorForRtl(Thickness thickness, bool isRtl)
    {
        if (!isRtl) return thickness;
        return new Thickness(thickness.Right, thickness.Top, thickness.Left, thickness.Bottom);
    }
    
    // Flip horizontal alignment
    public static LayoutOptions MirrorAlignment(LayoutOptions alignment, bool isRtl)
    {
        if (!isRtl) return alignment;
        if (alignment == LayoutOptions.Start) return LayoutOptions.End;
        if (alignment == LayoutOptions.End) return LayoutOptions.Start;
        return alignment;
    }
}
```

---

## Step 959: Translation Service Integration

```csharp
// ============================================
// Auto-translate user content (reviews, chat)
// ============================================

public class TranslationService
{
    private readonly HttpClient _http;
    private readonly IMultiLayerCache _cache;
    
    public TranslationService(HttpClient http, IMultiLayerCache cache)
    {
        _http = http;
        _cache = cache;
    }
    
    public async Task<string> TranslateAsync(
        string text, string targetLang = "th", string? sourceLang = null)
    {
        var cacheKey = $"translate:{targetLang}:{text.GetHashCode()}";
        var cached = await _cache.GetAsync<string>(cacheKey);
        if (cached != null) return cached;
        
        var response = await _http.PostAsJsonAsync("/translate", new
        {
            q = text,
            target = targetLang,
            source = sourceLang,
            format = "text"
        });
        
        var result = await response.Content.ReadFromJsonAsync<TranslateResult>();
        var translated = result?.Data?.Translations?.FirstOrDefault()?.TranslatedText ?? text;
        
        await _cache.SetAsync(cacheKey, translated,
            new CacheOptions(L2Ttl: TimeSpan.FromDays(30)));
        
        return translated;
    }
    
    // Detect language
    public async Task<string?> DetectLanguageAsync(string text)
    {
        var response = await _http.PostAsJsonAsync("/detect", new { q = text });
        var result = await response.Content.ReadFromJsonAsync<DetectResult>();
        return result?.Data?.Detections?.FirstOrDefault()?.Language;
    }
}

public record TranslateResult(TranslateData? Data);
public record TranslateData(List<TranslationRow>? Translations);
public record TranslationRow(string TranslatedText);
public record DetectResult(DetectData? Data);
public record DetectData(List<DetectionRow>? Detections);
public record DetectionRow(string Language, double Confidence);
```

---

## Step 960: Localization Tests

```csharp
// ============================================
// Localization Tests
// ============================================

[TestFixture]
public class LocalizationTests
{
    [TestCase(100, "หนึ่งร้อยบาทถ้วน")]
    [TestCase(250, "สองร้อยห้าสิบบาทถ้วน")]
    [TestCase(1000, "หนึ่งพันบาทถ้วน")]
    public void ThaiCurrencyWords_CorrectlyConverts(decimal amount, string expected)
    {
        var result = ThaiCurrencyFormatter.ToThaiWords(amount);
        Assert.That(result, Is.EqualTo(expected));
    }
    
    [TestCase("0812345678", "081-234-5678")]
    [TestCase("0234567890", "02-3456-7890")]
    public void ThaiPhone_FormatsCorrectly(string raw, string expected)
    {
        Assert.That(ThaiPhoneFormatter.Format(raw), Is.EqualTo(expected));
    }
    
    [Test]
    public void ThaiDate_ConvertsToBuddhistEra()
    {
        var date = new DateTime(2025, 10, 1);
        var result = ThaiDateFormatter.ToBuddhistEra(date);
        Assert.That(result, Does.Contain("2568")); // 2025 + 543
    }
    
    [TestCase("ก", 0)]
    [TestCase("ข", 1)]
    [TestCase("อ", 44)]
    public void ThaiAlphabet_GetsSectionOrder(string letter, int expectedIndex)
    {
        var section = ThaiAlphabet.GetSection(letter + "ข้าว");
        Assert.That(section, Is.EqualTo(letter));
    }
    
    [Test]
    public void ThaiPhone_IsValidMobile()
    {
        Assert.That(ThaiPhoneFormatter.IsValidMobile("0812345678"), Is.True);
        Assert.That(ThaiPhoneFormatter.IsValidMobile("0212345678"), Is.False); // landline
        Assert.That(ThaiPhoneFormatter.IsValidMobile("0712345678"), Is.False); // invalid prefix
    }
}
```

---

## สรุป Part 96

ใน Part 96 เราได้เรียนรู้:

1. **Resource Localization** - .resx files, CultureInfo switch, LanguageChanged event
2. **Thai Date Formatting** - พ.ศ. Buddhist era, relative time, greeting by time of day
3. **Thai Currency** - ฿ format, short form (K/ล.), ToThaiWords verbal amount
4. **Plural Forms** - Thai no-plural vs English singular/plural switch
5. **Thai Phone** - 10-digit formatting (mobile/landline), masking, E.164
6. **Thai Address** - บ้านเลขที่/ซอย/ถนน/ตำบล/อำเภอ/จังหวัด structure
7. **Thai Sorting** - CultureInfo CompareInfo, consonant section grouping
8. **RTL Layout** - FlowDirection, Padding mirroring, Alignment flipping
9. **Translation API** - Auto-translate user content, detect language, cache
10. **Localization Tests** - Currency words, phone format, Buddhist era, section ordering

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 96 | Steps 951-960*
