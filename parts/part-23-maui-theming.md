# Part 23: Theming และ Localization ใน .NET MAUI
## Steps 221-230: ธีม, สี, ภาษา

---

## Step 221: App Themes (Dark/Light Mode)

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- App.xaml - Global Theme Resources -->
<!-- ============================================ -->
<Application xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.App">
    <Application.Resources>
        <ResourceDictionary>
            
            <!-- Merge resource dictionaries -->
            <ResourceDictionary.MergedDictionaries>
                <ResourceDictionary Source="Resources/Colors.xaml" />
                <ResourceDictionary Source="Resources/Styles.xaml" />
            </ResourceDictionary.MergedDictionaries>
            
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Colors.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">

    <!-- Brand Colors -->
    <Color x:Key="Primary">#512BD4</Color>
    <Color x:Key="PrimaryDark">#3D21A0</Color>
    <Color x:Key="PrimaryLight">#7B61FF</Color>
    <Color x:Key="Secondary">#DFD8F7</Color>
    <Color x:Key="Accent">#FF6B6B</Color>
    
    <!-- Semantic Colors -->
    <Color x:Key="Success">#34D399</Color>
    <Color x:Key="Warning">#FBBF24</Color>
    <Color x:Key="Danger">#EF4444</Color>
    <Color x:Key="Info">#60A5FA</Color>
    
    <!-- AppThemeBinding Colors -->
    <Color x:Key="PageBackground" x:DynamicResource="{AppThemeBinding Light=White, Dark=#1C1C1E}" />
    
    <!-- Or using OnPlatform/AppTheme -->
    <AppTheme x:Key="BackgroundTheme"
              Light="White"
              Dark="#1C1C1E" />
    
    <AppTheme x:Key="TextPrimaryTheme"
              Light="#1C1C1E"
              Dark="White" />
    
    <AppTheme x:Key="CardBackgroundTheme"
              Light="#F5F5F5"
              Dark="#2C2C2E" />
    
    <AppTheme x:Key="BorderColorTheme"
              Light="#E5E7EB"
              Dark="#3A3A3C" />
    
    <AppTheme x:Key="ShadowColorTheme"
              Light="#00000020"
              Dark="#00000060" />

</ResourceDictionary>
```

---

## Step 222: Dynamic Styles

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Styles.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">

    <!-- Button Styles -->
    <Style x:Key="PrimaryButtonStyle" TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource Primary}" />
        <Setter Property="TextColor" Value="White" />
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="FontSize" Value="16" />
        <Setter Property="FontAttributes" Value="Bold" />
        <Setter Property="HeightRequest" Value="50" />
        <Setter Property="VisualStateManager.VisualStateGroups">
            <VisualStateGroupList>
                <VisualStateGroup x:Name="CommonStates">
                    <VisualState x:Name="Normal">
                        <VisualState.Setters>
                            <Setter Property="BackgroundColor" Value="{StaticResource Primary}" />
                            <Setter Property="Scale" Value="1" />
                        </VisualState.Setters>
                    </VisualState>
                    <VisualState x:Name="Pressed">
                        <VisualState.Setters>
                            <Setter Property="BackgroundColor" Value="{StaticResource PrimaryDark}" />
                            <Setter Property="Scale" Value="0.97" />
                        </VisualState.Setters>
                    </VisualState>
                    <VisualState x:Name="Disabled">
                        <VisualState.Setters>
                            <Setter Property="BackgroundColor" Value="#CCCCCC" />
                            <Setter Property="TextColor" Value="#888888" />
                        </VisualState.Setters>
                    </VisualState>
                </VisualStateGroup>
            </VisualStateGroupList>
        </Setter>
    </Style>
    
    <Style x:Key="SecondaryButtonStyle" TargetType="Button">
        <Setter Property="BackgroundColor" Value="Transparent" />
        <Setter Property="TextColor" Value="{StaticResource Primary}" />
        <Setter Property="BorderColor" Value="{StaticResource Primary}" />
        <Setter Property="BorderWidth" Value="2" />
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="FontSize" Value="16" />
        <Setter Property="HeightRequest" Value="50" />
    </Style>
    
    <!-- Label Styles -->
    <Style x:Key="TitleStyle" TargetType="Label">
        <Setter Property="FontSize" Value="24" />
        <Setter Property="FontAttributes" Value="Bold" />
        <Setter Property="TextColor" Value="{AppThemeBinding Light=#1C1C1E, Dark=White}" />
    </Style>
    
    <Style x:Key="SubtitleStyle" TargetType="Label">
        <Setter Property="FontSize" Value="18" />
        <Setter Property="FontAttributes" Value="Bold" />
        <Setter Property="TextColor" Value="{AppThemeBinding Light=#1C1C1E, Dark=White}" />
    </Style>
    
    <Style x:Key="BodyStyle" TargetType="Label">
        <Setter Property="FontSize" Value="15" />
        <Setter Property="TextColor" Value="{AppThemeBinding Light=#4B5563, Dark=#9CA3AF}" />
        <Setter Property="LineBreakMode" Value="WordWrap" />
    </Style>
    
    <!-- Card Style -->
    <Style x:Key="CardStyle" TargetType="Frame">
        <Setter Property="BackgroundColor" Value="{AppThemeBinding Light=White, Dark=#2C2C2E}" />
        <Setter Property="CornerRadius" Value="16" />
        <Setter Property="BorderColor" Value="{AppThemeBinding Light=#E5E7EB, Dark=#3A3A3C}" />
        <Setter Property="HasShadow" Value="True" />
        <Setter Property="Padding" Value="16" />
    </Style>
    
    <!-- Entry Style -->
    <Style x:Key="ModernEntryStyle" TargetType="Entry">
        <Setter Property="BackgroundColor" Value="{AppThemeBinding Light=#F9FAFB, Dark=#2C2C2E}" />
        <Setter Property="TextColor" Value="{AppThemeBinding Light=#1C1C1E, Dark=White}" />
        <Setter Property="PlaceholderColor" Value="{AppThemeBinding Light=#9CA3AF, Dark=#6B7280}" />
        <Setter Property="HeightRequest" Value="50" />
        <Setter Property="FontSize" Value="15" />
    </Style>
    
</ResourceDictionary>
```

---

## Step 223: Theme Switching

```csharp
// ============================================
// Theme Management
// ============================================

public enum AppThemeMode { System, Light, Dark }

public class ThemeService
{
    private const string ThemeKey = "app_theme";
    
    public AppThemeMode CurrentTheme
    {
        get
        {
            var value = Preferences.Default.Get(ThemeKey, "System");
            return Enum.TryParse<AppThemeMode>(value, out var theme) ? theme : AppThemeMode.System;
        }
        private set
        {
            Preferences.Default.Set(ThemeKey, value.ToString());
            ApplyTheme(value);
        }
    }
    
    public void Initialize()
    {
        ApplyTheme(CurrentTheme);
    }
    
    public void SetTheme(AppThemeMode theme)
    {
        CurrentTheme = theme;
    }
    
    public void ToggleDarkMode()
    {
        CurrentTheme = CurrentTheme == AppThemeMode.Dark 
            ? AppThemeMode.Light 
            : AppThemeMode.Dark;
    }
    
    private static void ApplyTheme(AppThemeMode theme)
    {
        var appTheme = theme switch
        {
            AppThemeMode.Light => Microsoft.Maui.ApplicationModel.AppTheme.Light,
            AppThemeMode.Dark => Microsoft.Maui.ApplicationModel.AppTheme.Dark,
            _ => Microsoft.Maui.ApplicationModel.AppTheme.Unspecified
        };
        
        if (Application.Current != null)
            Application.Current.UserAppTheme = appTheme;
    }
    
    public bool IsDarkMode => Application.Current?.RequestedTheme == 
        Microsoft.Maui.ApplicationModel.AppTheme.Dark;
}

// ============================================
// Dynamic Color via Code
// ============================================

public static class ThemeExtensions
{
    public static Color GetThemeColor(this Application app, string lightHex, string darkHex)
    {
        bool isDark = app.RequestedTheme == Microsoft.Maui.ApplicationModel.AppTheme.Dark;
        return Color.FromArgb(isDark ? darkHex : lightHex);
    }
    
    public static void ApplyCustomColors(this Application app, 
        Dictionary<string, (string Light, string Dark)> colors)
    {
        bool isDark = app.RequestedTheme == Microsoft.Maui.ApplicationModel.AppTheme.Dark;
        
        foreach (var (key, (light, dark)) in colors)
        {
            app.Resources[key] = Color.FromArgb(isDark ? dark : light);
        }
    }
}
```

---

## Step 224: Custom Fonts

```xml
<!-- MauiProgram.cs font registration: -->
<!--
builder
    .ConfigureFonts(fonts =>
    {
        fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
        fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
        fonts.AddFont("NotoSansThai-Regular.ttf", "NotoSansThai");
        fonts.AddFont("NotoSansThai-Bold.ttf", "NotoSansThaiB");
        fonts.AddFont("fa-solid-900.ttf", "FontAwesome");
    });
-->

<!-- Usage in XAML -->
<!-- <Label Text="สวัสดีชาวโลก" FontFamily="NotoSansThai" FontSize="20" /> -->
<!-- <Label FontFamily="FontAwesome" Text="&#xf000;" FontSize="24" /> -->
```

```csharp
// ============================================
// Font Icons (FontAwesome)
// ============================================

public static class FontAwesomeIcons
{
    public const string Home = "";
    public const string ShoppingCart = "";
    public const string User = "";
    public const string Heart = "";
    public const string Search = "";
    public const string Bell = "";
    public const string Settings = "";
    public const string Camera = "";
    public const string Location = "";
    public const string Check = "";
    public const string Times = "";
    public const string Edit = "";
    public const string Delete = "";
    public const string Plus = "";
    public const string Star = "";
    public const string Share = "";
    public const string Download = "";
    public const string Upload = "";
    public const string Filter = "";
    public const string Sort = "";
    public const string Menu = "";
    public const string Back = "";
    public const string Forward = "";
}

// XAML:
// <Label FontFamily="FontAwesome" Text="{x:Static local:FontAwesomeIcons.Home}" />
```

---

## Step 225: Localization / i18n

```csharp
// ============================================
// Localization Setup
// ============================================

// Create Resources folder with .resx files:
// Resources/Strings/AppResources.resx (default - English)
// Resources/Strings/AppResources.th.resx (Thai)

// AppResources.resx content:
// Key: Hello | Value: Hello
// Key: Welcome | Value: Welcome to {0}!
// Key: Cancel | Value: Cancel

// AppResources.th.resx content:
// Key: Hello | Value: สวัสดี
// Key: Welcome | Value: ยินดีต้อนรับสู่ {0}!
// Key: Cancel | Value: ยกเลิก

// ============================================
// Localization Service
// ============================================

using System.Globalization;
using System.Resources;

public class LocalizationService
{
    private static ResourceManager _resourceManager = 
        new ResourceManager("MyApp.Resources.Strings.AppResources", 
            typeof(LocalizationService).Assembly);
    
    private CultureInfo _currentCulture = CultureInfo.CurrentUICulture;
    
    public static string Get(string key, params object[] args)
    {
        try
        {
            var value = _resourceManager.GetString(key, CultureInfo.CurrentUICulture)
                        ?? key;
            return args.Length > 0 ? string.Format(value, args) : value;
        }
        catch
        {
            return key;
        }
    }
    
    public void SetLanguage(string languageCode)
    {
        var culture = new CultureInfo(languageCode);
        CultureInfo.CurrentCulture = culture;
        CultureInfo.CurrentUICulture = culture;
        _currentCulture = culture;
        
        Preferences.Default.Set("app_language", languageCode);
        
        // Restart app or reload UI
    }
    
    public string CurrentLanguage => _currentCulture.TwoLetterISOLanguageName;
    
    public List<LanguageOption> AvailableLanguages { get; } = new()
    {
        new("th", "ภาษาไทย", "🇹🇭"),
        new("en", "English", "🇺🇸"),
        new("zh", "中文", "🇨🇳"),
        new("ja", "日本語", "🇯🇵"),
        new("ko", "한국어", "🇰🇷"),
    };
    
    private static ResourceManager _resourceManager2 = null!;
    
    public static LocalizationService Instance { get; } = new();
    
    private LocalizationService()
    {
        var savedLang = Preferences.Default.Get("app_language", "th");
        SetLanguage(savedLang);
    }
}

public record LanguageOption(string Code, string DisplayName, string Flag);
```

---

## Step 226: Markup Extension for Localization

```csharp
// ============================================
// XAML Markup Extension
// ============================================

using System.Resources;

[ContentProperty(nameof(Key))]
public class TranslateExtension : IMarkupExtension<string>
{
    private static readonly ResourceManager ResourceManager =
        new ResourceManager("MyApp.Resources.Strings.AppResources",
            typeof(TranslateExtension).Assembly);
    
    public string Key { get; set; } = string.Empty;
    
    public string ProvideValue(IServiceProvider serviceProvider)
    {
        if (string.IsNullOrEmpty(Key))
            return string.Empty;
        
        return ResourceManager.GetString(Key, CultureInfo.CurrentUICulture) ?? Key;
    }
    
    object IMarkupExtension.ProvideValue(IServiceProvider serviceProvider)
        => ProvideValue(serviceProvider);
}

// Usage in XAML:
// xmlns:i18n="clr-namespace:MyApp.Extensions"
// <Label Text="{i18n:Translate Key=Hello}" />
// <Button Text="{i18n:Translate Key=Cancel}" />
```

---

## Step 227: Number/Currency Formatting

```csharp
// ============================================
// Culture-aware Formatting
// ============================================

public static class FormatExtensions
{
    public static string ToCurrency(this decimal amount, string? currencyCode = null)
    {
        var culture = CultureInfo.CurrentCulture;
        
        if (currencyCode != null)
        {
            var region = new RegionInfo(currencyCode);
            return $"{region.CurrencySymbol}{amount:N2}";
        }
        
        return amount.ToString("C", culture);
    }
    
    public static string ToThaiCurrency(this decimal amount)
        => $"฿{amount:N2}";
    
    public static string ToRelativeTime(this DateTime dateTime)
    {
        var diff = DateTime.UtcNow - dateTime.ToUniversalTime();
        
        return diff switch
        {
            { TotalSeconds: < 60 } => "เมื่อกี้",
            { TotalMinutes: < 60 } d => $"{(int)d.TotalMinutes} นาทีที่แล้ว",
            { TotalHours: < 24 } d => $"{(int)d.TotalHours} ชั่วโมงที่แล้ว",
            { TotalDays: < 7 } d => $"{(int)d.TotalDays} วันที่แล้ว",
            { TotalDays: < 30 } d => $"{(int)(d.TotalDays / 7)} สัปดาห์ที่แล้ว",
            { TotalDays: < 365 } d => $"{(int)(d.TotalDays / 30)} เดือนที่แล้ว",
            _ => $"{(int)(diff.TotalDays / 365)} ปีที่แล้ว"
        };
    }
    
    public static string ToThaiDate(this DateTime date)
    {
        var thaiYear = date.Year + 543;
        var thaiMonths = new[]
        {
            "มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน",
            "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม",
            "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"
        };
        
        return $"{date.Day} {thaiMonths[date.Month - 1]} {thaiYear}";
    }
    
    public static string ToThaiShortDate(this DateTime date)
    {
        var thaiYear = date.Year + 543;
        return $"{date.Day:D2}/{date.Month:D2}/{thaiYear}";
    }
    
    public static string FormatFileSize(this long bytes)
    {
        string[] units = { "B", "KB", "MB", "GB", "TB" };
        double size = bytes;
        int unit = 0;
        
        while (size >= 1024 && unit < units.Length - 1)
        {
            size /= 1024;
            unit++;
        }
        
        return $"{size:F1} {units[unit]}";
    }
}
```

---

## Step 228: Visual State Manager

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- Visual State Manager -->
<!-- ============================================ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.Views.VsmDemoPage">
    
    <ContentPage.Resources>
        
        <!-- Button VSM -->
        <Style TargetType="Button" x:Key="AnimatedButton">
            <Setter Property="VisualStateManager.VisualStateGroups">
                <VisualStateGroupList>
                    <VisualStateGroup x:Name="CommonStates">
                        
                        <VisualState x:Name="Normal">
                            <VisualState.Setters>
                                <Setter Property="Scale" Value="1.0" />
                                <Setter Property="Opacity" Value="1.0" />
                            </VisualState.Setters>
                        </VisualState>
                        
                        <VisualState x:Name="Pressed">
                            <VisualState.Setters>
                                <Setter Property="Scale" Value="0.95" />
                                <Setter Property="Opacity" Value="0.8" />
                            </VisualState.Setters>
                        </VisualState>
                        
                        <VisualState x:Name="Disabled">
                            <VisualState.Setters>
                                <Setter Property="Opacity" Value="0.4" />
                            </VisualState.Setters>
                        </VisualState>
                        
                        <VisualState x:Name="Focused">
                            <VisualState.Setters>
                                <Setter Property="BorderColor" Value="{StaticResource Primary}" />
                                <Setter Property="BorderWidth" Value="2" />
                            </VisualState.Setters>
                        </VisualState>
                        
                    </VisualStateGroup>
                </VisualStateGroupList>
            </Setter>
        </Style>
        
    </ContentPage.Resources>
    
    <VerticalStackLayout Spacing="20" Padding="20">
        
        <!-- Trigger VSM from ViewModel -->
        <Label x:Name="StatusLabel" Text="สถานะ">
            <VisualStateManager.VisualStateGroups>
                <VisualStateGroup x:Name="StatusStates">
                    
                    <VisualState x:Name="Normal">
                        <VisualState.Setters>
                            <Setter Property="TextColor" Value="Black" />
                            <Setter Property="FontAttributes" Value="None" />
                        </VisualState.Setters>
                    </VisualState>
                    
                    <VisualState x:Name="Success">
                        <VisualState.Setters>
                            <Setter Property="TextColor" Value="#34D399" />
                            <Setter Property="FontAttributes" Value="Bold" />
                        </VisualState.Setters>
                    </VisualState>
                    
                    <VisualState x:Name="Error">
                        <VisualState.Setters>
                            <Setter Property="TextColor" Value="#EF4444" />
                            <Setter Property="FontAttributes" Value="Bold" />
                        </VisualState.Setters>
                    </VisualState>
                    
                    <VisualState x:Name="Warning">
                        <VisualState.Setters>
                            <Setter Property="TextColor" Value="#FBBF24" />
                            <Setter Property="FontAttributes" Value="Bold" />
                        </VisualState.Setters>
                    </VisualState>
                    
                </VisualStateGroup>
            </VisualStateManager.VisualStateGroups>
        </Label>
        
    </VerticalStackLayout>
    
</ContentPage>
```

---

## Step 229: Animations

```csharp
// ============================================
// MAUI Animations
// ============================================

public partial class AnimationPage : ContentPage
{
    private Image _image = new Image { Source = "dotnet_bot.png", HeightRequest = 200 };
    
    // Fade
    private Task FadeAsync()
        => _image.FadeTo(0, 500, Easing.Linear)
                 .ContinueWith(_ => _image.FadeTo(1, 500, Easing.Linear));
    
    // Scale
    private Task ScaleAsync()
        => _image.ScaleTo(1.5, 300, Easing.SpringOut)
                 .ContinueWith(_ => _image.ScaleTo(1, 300, Easing.SpringIn));
    
    // Translate
    private Task SlideAsync()
        => _image.TranslateTo(100, 0, 400, Easing.CubicOut)
                 .ContinueWith(_ => _image.TranslateTo(0, 0, 400, Easing.CubicIn));
    
    // Rotate
    private Task RotateAsync()
        => _image.RotateTo(360, 600, Easing.Linear);
    
    // Combined animations
    private async Task CombinedAnimationAsync()
    {
        var fadeTask = _image.FadeTo(0.5, 500);
        var scaleTask = _image.ScaleTo(1.2, 500, Easing.SpringOut);
        await Task.WhenAll(fadeTask, scaleTask);
        
        await Task.WhenAll(
            _image.FadeTo(1, 300),
            _image.ScaleTo(1, 300));
    }
    
    // Custom animation with Animation class
    private Task CustomAnimationAsync()
    {
        var animation = new Animation
        {
            { 0, 0.5, new Animation(v => _image.Opacity = v, 1, 0) },
            { 0.5, 1, new Animation(v => _image.Opacity = v, 0, 1) }
        };
        
        var tcs = new TaskCompletionSource<bool>();
        animation.Commit(_image, "PulseAnimation",
            rate: 16, length: 1000,
            easing: Easing.Linear,
            finished: (_, _) => tcs.TrySetResult(true));
        
        return tcs.Task;
    }
    
    // Repeat animation
    private void StartPulse()
    {
        var animation = new Animation(v => _image.Scale = v, 1, 1.05);
        animation.Commit(_image, "Pulse",
            rate: 16, length: 800,
            easing: Easing.SinInOut,
            repeat: () => true);
    }
    
    private void StopPulse()
    {
        _image.AbortAnimation("Pulse");
        _image.Scale = 1;
    }
    
    // Spring back
    private async Task SpringBackAsync(View view, double overshoot = 0.1)
    {
        await view.ScaleTo(1 + overshoot, 100, Easing.CubicOut);
        await view.ScaleTo(1, 200, Easing.SpringIn);
    }
}
```

---

## Step 230: Custom Controls

```csharp
// ============================================
// Custom Control - Rating Stars
// ============================================

public class RatingControl : ContentView
{
    public static readonly BindableProperty RatingProperty =
        BindableProperty.Create(nameof(Rating), typeof(int), typeof(RatingControl), 0,
            propertyChanged: OnRatingChanged);
    
    public static readonly BindableProperty MaxRatingProperty =
        BindableProperty.Create(nameof(MaxRating), typeof(int), typeof(RatingControl), 5);
    
    public static readonly BindableProperty StarColorProperty =
        BindableProperty.Create(nameof(StarColor), typeof(Color), typeof(RatingControl), Colors.Gold);
    
    public static readonly BindableProperty EmptyColorProperty =
        BindableProperty.Create(nameof(EmptyColor), typeof(Color), typeof(RatingControl), Colors.Gray);
    
    public int Rating
    {
        get => (int)GetValue(RatingProperty);
        set => SetValue(RatingProperty, value);
    }
    
    public int MaxRating
    {
        get => (int)GetValue(MaxRatingProperty);
        set => SetValue(MaxRatingProperty, value);
    }
    
    public Color StarColor
    {
        get => (Color)GetValue(StarColorProperty);
        set => SetValue(StarColorProperty, value);
    }
    
    public Color EmptyColor
    {
        get => (Color)GetValue(EmptyColorProperty);
        set => SetValue(EmptyColorProperty, value);
    }
    
    public event EventHandler<int>? RatingChanged;
    
    private HorizontalStackLayout _starsLayout = new();
    
    public RatingControl()
    {
        Content = _starsLayout;
        BuildStars();
    }
    
    private void BuildStars()
    {
        _starsLayout.Children.Clear();
        _starsLayout.Spacing = 4;
        
        for (int i = 1; i <= MaxRating; i++)
        {
            int starIndex = i;
            
            var star = new Label
            {
                Text = "★",
                FontSize = 28,
                TextColor = starIndex <= Rating ? StarColor : EmptyColor
            };
            
            star.GestureRecognizers.Add(new TapGestureRecognizer
            {
                Command = new Command(() =>
                {
                    Rating = starIndex;
                    RatingChanged?.Invoke(this, starIndex);
                })
            });
            
            _starsLayout.Children.Add(star);
        }
    }
    
    private static void OnRatingChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var control = (RatingControl)bindable;
        
        for (int i = 0; i < control._starsLayout.Children.Count; i++)
        {
            if (control._starsLayout.Children[i] is Label label)
                label.TextColor = i < (int)newValue ? control.StarColor : control.EmptyColor;
        }
    }
}

// ============================================
// Custom Control - Badge
// ============================================

public class BadgeView : ContentView
{
    public static readonly BindableProperty CountProperty =
        BindableProperty.Create(nameof(Count), typeof(int), typeof(BadgeView), 0,
            propertyChanged: OnCountChanged);
    
    public int Count
    {
        get => (int)GetValue(CountProperty);
        set => SetValue(CountProperty, value);
    }
    
    private readonly Label _countLabel;
    
    public BadgeView()
    {
        _countLabel = new Label
        {
            TextColor = Colors.White,
            FontSize = 11,
            FontAttributes = FontAttributes.Bold,
            HorizontalOptions = LayoutOptions.Center,
            VerticalOptions = LayoutOptions.Center
        };
        
        var badge = new Border
        {
            BackgroundColor = Colors.Red,
            StrokeShape = new RoundRectangle { CornerRadius = 10 },
            Padding = new Thickness(5, 2),
            MinimumWidthRequest = 20,
            IsVisible = Count > 0,
            Content = _countLabel
        };
        
        Content = badge;
        IsVisible = Count > 0;
    }
    
    private static void OnCountChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var control = (BadgeView)bindable;
        int count = (int)newValue;
        control._countLabel.Text = count > 99 ? "99+" : count.ToString();
        control.IsVisible = count > 0;
    }
}
```

---

## สรุป Part 23

ใน Part 23 เราได้เรียนรู้:

1. **App Themes** - Light/Dark mode, AppThemeBinding
2. **Resource Dictionaries** - Colors.xaml, Styles.xaml, MergedDictionaries
3. **Dynamic Styles** - Visual State Manager, Pressed/Disabled states
4. **Theme Switching** - Runtime theme change
5. **Custom Fonts** - Thai fonts, FontAwesome icons
6. **Localization** - .resx files, CultureInfo, Markup Extension
7. **Number Formatting** - Currency, dates, Thai date formats
8. **Visual State Manager** - States สำหรับ custom UI
9. **Animations** - Fade, Scale, Translate, Rotate, Combined
10. **Custom Controls** - RatingControl, BadgeView

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 23 | Steps 221-230*
