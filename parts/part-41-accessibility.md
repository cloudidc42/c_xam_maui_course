# Part 41: Accessibility (a11y)
## Steps 401-410: TalkBack, VoiceOver, Semantic Properties

---

## Step 401: Accessibility Overview

```csharp
// ============================================
// Accessibility ใน .NET MAUI
// ============================================

/*
 * Accessibility คืออะไร?
 * - ทำให้แอปใช้งานได้สำหรับผู้พิการ
 * - TalkBack (Android) / VoiceOver (iOS) - screen reader
 * - Switch Access - ใช้ switch แทน touch
 * - High Contrast, Large Text
 *
 * APIs ใน .NET MAUI:
 * - SemanticProperties - ชื่อ, hint, description
 * - AutomationProperties - test automation
 * - Heading levels
 * - Live regions - dynamic content announcements
 */

// Basic semantic properties
var button = new Button { Text = "ถ่ายรูป" };
SemanticProperties.SetDescription(button, "ถ่ายรูปภาพสินค้าใหม่");
SemanticProperties.SetHint(button, "เปิดกล้องถ่ายรูป");

var image = new Image { Source = "product.png" };
SemanticProperties.SetDescription(image, "รูปภาพสินค้า: MacBook Pro สีดำ");

// In XAML
/*
<Button Text="ซื้อเลย"
        SemanticProperties.Description="ซื้อสินค้า MacBook Pro ราคา 59,900 บาท"
        SemanticProperties.Hint="กดเพื่อเพิ่มสินค้าในตะกร้า" />

<Image Source="product.png"
       SemanticProperties.Description="MacBook Pro 14 นิ้ว สีดำ" />

<Label Text="รายการสินค้า"
       SemanticProperties.HeadingLevel="Level1" />
*/
```

---

## Step 402: Semantic Properties in Detail

```xml
<!-- ============================================
     Semantic Properties ใน XAML
     ============================================ -->

<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             Title="ตะกร้าสินค้า"
             SemanticProperties.Description="หน้าตะกร้าสินค้า">

    <ScrollView>
        <VerticalStackLayout Padding="16" Spacing="12">
            
            <!-- Heading hierarchy -->
            <Label Text="ตะกร้าของฉัน"
                   Style="{StaticResource Title}"
                   SemanticProperties.HeadingLevel="Level1" />
            
            <!-- Decorative image - set Description to empty -->
            <Image Source="cart_banner.png"
                   SemanticProperties.Description="" />
            
            <!-- Product item with full context -->
            <Frame>
                <Grid ColumnDefinitions="80,*,Auto">
                    <Image Grid.Column="0"
                           Source="{Binding ImageUrl}"
                           SemanticProperties.Description="{Binding ImageDescription}" />
                    
                    <VerticalStackLayout Grid.Column="1" Spacing="4">
                        <Label Text="{Binding Name}"
                               SemanticProperties.HeadingLevel="Level3" />
                        <Label Text="{Binding PriceText}" />
                    </VerticalStackLayout>
                    
                    <!-- Meaningful button label -->
                    <Button Grid.Column="2"
                            Text="ลบ"
                            SemanticProperties.Description="{Binding RemoveDescription}"
                            Command="{Binding RemoveCommand}" />
                </Grid>
            </Frame>
            
            <!-- Summary section -->
            <Label Text="สรุปรายการ"
                   SemanticProperties.HeadingLevel="Level2" />
            
            <!-- Live region - announces changes -->
            <Label Text="{Binding TotalText}"
                   SemanticProperties.Description="{Binding TotalAnnouncementText}" />
            
        </VerticalStackLayout>
    </ScrollView>
    
</ContentPage>
```

```csharp
// Cart item ViewModel with accessibility descriptions
public partial class CartItemViewModel : ObservableObject
{
    [ObservableProperty] private string _name = string.Empty;
    [ObservableProperty] private decimal _price;
    [ObservableProperty] private int _quantity;
    [ObservableProperty] private string? _imageUrl;
    
    public string PriceText => Price.ToString("C0");
    
    public string ImageDescription => $"รูปสินค้า {Name}";
    
    public string RemoveDescription => $"ลบ {Name} ออกจากตะกร้า";
    
    public string QuantityDescription => $"จำนวน {Quantity} ชิ้น";
}
```

---

## Step 403: Focus Management

```csharp
// ============================================
// Keyboard & Focus Management
// ============================================

public partial class LoginPage : ContentPage
{
    public LoginPage()
    {
        InitializeComponent();
        
        // Set initial focus
        Loaded += (_, _) => EmailEntry.Focus();
        
        // Tab order
        EmailEntry.TabIndex = 0;
        PasswordEntry.TabIndex = 1;
        LoginButton.TabIndex = 2;
        
        // Keyboard Return key navigation
        EmailEntry.ReturnType = ReturnType.Next;
        EmailEntry.ReturnCommand = new Command(() => PasswordEntry.Focus());
        
        PasswordEntry.ReturnType = ReturnType.Done;
        PasswordEntry.ReturnCommand = new Command(async () =>
        {
            PasswordEntry.Unfocus();
            await ((LoginViewModel)BindingContext).LoginCommand.ExecuteAsync(null);
        });
    }
}

// Programmatic focus from ViewModel
public partial class FormViewModel : ObservableObject
{
    private readonly IPageNavigator _navigator;
    
    [RelayCommand]
    private async Task SubmitAsync()
    {
        if (!Validate()) return;
        
        var result = await SaveAsync();
        
        if (!result.IsSuccess)
        {
            // Announce error to screen reader
            await SemanticScreenReader.Default.AnnounceAsync(
                $"เกิดข้อผิดพลาด: {result.Error}");
        }
    }
}
```

---

## Step 404: Screen Reader Announcements

```csharp
// ============================================
// SemanticScreenReader - Dynamic Announcements
// ============================================

public class AccessibilityService
{
    // Announce important state changes
    public async Task AnnounceAsync(string message)
    {
        // Only announce on main thread
        await MainThread.InvokeOnMainThreadAsync(async () =>
        {
            await SemanticScreenReader.Default.AnnounceAsync(message);
        });
    }
    
    // Announce loading state
    public async Task AnnounceLoadingAsync(string context)
        => await AnnounceAsync($"กำลังโหลด{context}");
    
    public async Task AnnounceLoadedAsync(string context, int count)
        => await AnnounceAsync($"โหลด{context}แล้ว จำนวน {count} รายการ");
    
    // Announce navigation
    public async Task AnnounceNavigationAsync(string pageName)
        => await AnnounceAsync($"เปิดหน้า{pageName}");
    
    // Announce form errors
    public async Task AnnounceFormErrorAsync(string fieldName, string error)
        => await AnnounceAsync($"{fieldName}: {error}");
}

// ViewModel with announcements
public partial class ProductListViewModel : ObservableObject
{
    private readonly AccessibilityService _a11y;
    
    [ObservableProperty] private ObservableCollection<Product> _products = new();
    [ObservableProperty] private bool _isLoading;
    
    public ProductListViewModel(AccessibilityService a11y) => _a11y = a11y;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        await _a11y.AnnounceLoadingAsync("รายการสินค้า");
        
        // Load...
        
        IsLoading = false;
        await _a11y.AnnounceLoadedAsync("รายการสินค้า", Products.Count);
    }
}
```

---

## Step 405: High Contrast & Large Text

```csharp
// ============================================
// Respond to System Accessibility Settings
// ============================================

public class AccessibilitySettingsService
{
    // Detect large text setting
    public bool IsLargeTextEnabled
    {
        get
        {
#if ANDROID
            var config = Android.App.Application.Context.Resources?.Configuration;
            return config?.FontScale > 1.3f;
#elif IOS
            return UIKit.UIApplication.SharedApplication.PreferredContentSizeCategory
                .ToString().Contains("Accessibility");
#else
            return false;
#endif
        }
    }
    
    // Detect high contrast
    public bool IsHighContrastEnabled
    {
        get
        {
#if ANDROID
            var am = Android.App.Application.Context
                .GetSystemService(Android.Content.Context.AccessibilityService) 
                as Android.Views.Accessibility.AccessibilityManager;
            return am?.IsHighTextContrastEnabled ?? false;
#elif IOS
            return UIKit.UIAccessibility.DarkerSystemColorsEnabled;
#else
            return false;
#endif
        }
    }
    
    // Detect reduced motion
    public bool IsReducedMotionEnabled
    {
        get
        {
#if IOS
            return UIKit.UIAccessibility.IsReduceMotionEnabled;
#elif ANDROID
            // Android doesn't have direct API, check animation scale
            var scale = Android.Provider.Settings.Global.GetFloat(
                Android.App.Application.Context.ContentResolver,
                Android.Provider.Settings.Global.AnimatorDurationScale, 1.0f);
            return scale == 0;
#else
            return false;
#endif
        }
    }
}

// Adaptive styling
public class AdaptiveStyleService
{
    private readonly AccessibilitySettingsService _a11y;
    
    public AdaptiveStyleService(AccessibilitySettingsService a11y) => _a11y = a11y;
    
    public double GetBaseFontSize(double normal = 16)
        => _a11y.IsLargeTextEnabled ? normal * 1.3 : normal;
    
    public Color GetTextColor(Color normal, Color highContrast)
        => _a11y.IsHighContrastEnabled ? highContrast : normal;
    
    public uint GetAnimationDuration(uint normal = 300)
        => _a11y.IsReducedMotionEnabled ? 0u : normal;
}
```

---

## Step 406: Custom Accessible Controls

```csharp
// ============================================
// Accessible Custom Control
// ============================================

public class AccessibleRatingControl : ContentView
{
    public static readonly BindableProperty RatingProperty =
        BindableProperty.Create(nameof(Rating), typeof(int), typeof(AccessibleRatingControl), 0,
            propertyChanged: OnRatingChanged);
    
    public static readonly BindableProperty MaxRatingProperty =
        BindableProperty.Create(nameof(MaxRating), typeof(int), typeof(AccessibleRatingControl), 5);
    
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
    
    private readonly HorizontalStackLayout _starsLayout;
    
    public AccessibleRatingControl()
    {
        _starsLayout = new HorizontalStackLayout { Spacing = 4 };
        
        // Make the whole control accessible as a slider
        SemanticProperties.SetDescription(this, GetDescription());
        SetSemanticFocus(this, true);
        
        Content = _starsLayout;
        RenderStars();
        
        // Keyboard: left/right arrows change rating
        var gesture = new TapGestureRecognizer();
        GestureRecognizers.Add(gesture);
    }
    
    private void RenderStars()
    {
        _starsLayout.Children.Clear();
        
        for (int i = 1; i <= MaxRating; i++)
        {
            var star = new Label
            {
                Text = i <= Rating ? "★" : "☆",
                FontSize = 24,
                TextColor = i <= Rating ? Colors.Orange : Colors.Gray
            };
            
            // Each star is individually accessible
            var starIndex = i;
            SemanticProperties.SetDescription(star, 
                $"ดาว {starIndex} จาก {MaxRating}");
            
            var tap = new TapGestureRecognizer();
            tap.Tapped += (_, _) => Rating = starIndex;
            star.GestureRecognizers.Add(tap);
            
            _starsLayout.Children.Add(star);
        }
    }
    
    private string GetDescription()
        => $"คะแนน {Rating} ดาวจาก {MaxRating} ดาว กดเพื่อเปลี่ยนคะแนน";
    
    private static void OnRatingChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var control = (AccessibleRatingControl)bindable;
        control.RenderStars();
        
        SemanticProperties.SetDescription(control, control.GetDescription());
        SemanticScreenReader.Default.AnnounceAsync(
            $"คะแนน {newValue} ดาว จาก {control.MaxRating} ดาว");
    }
    
    private static void SetSemanticFocus(View view, bool isFocused)
    {
        // Platform-specific focus handling
        Microsoft.Maui.Controls.PlatformConfiguration.AndroidSpecific
            .AccessibilityTraversalAfter.SetAccessibilityTraversalAfter(view, view);
    }
}
```

---

## Step 407: Automated Accessibility Testing

```csharp
// ============================================
// Accessibility Test Helpers
// ============================================

public static class AccessibilityTestHelpers
{
    // Check all visible elements have descriptions
    public static List<AccessibilityIssue> AuditPage(ContentPage page)
    {
        var issues = new List<AccessibilityIssue>();
        
        AuditElement(page.Content, issues);
        
        return issues;
    }
    
    private static void AuditElement(IView? element, List<AccessibilityIssue> issues)
    {
        if (element == null) return;
        
        if (element is View view)
        {
            // Check image has description
            if (view is Image image)
            {
                var desc = SemanticProperties.GetDescription(image);
                if (string.IsNullOrEmpty(desc))
                    issues.Add(new AccessibilityIssue(
                        "Missing Description",
                        $"Image at {GetElementPath(image)} has no description",
                        Severity.Warning));
            }
            
            // Check button has meaningful label
            if (view is Button button)
            {
                var desc = SemanticProperties.GetDescription(button);
                var text = button.Text;
                
                if (string.IsNullOrEmpty(text) && string.IsNullOrEmpty(desc))
                    issues.Add(new AccessibilityIssue(
                        "Missing Label",
                        $"Button at {GetElementPath(button)} has no text or description",
                        Severity.Error));
                
                if (text?.Length == 1) // single character
                    issues.Add(new AccessibilityIssue(
                        "Ambiguous Label",
                        $"Button '{text}' may be unclear to screen reader users",
                        Severity.Warning));
            }
            
            // Check touch targets >= 44x44
            if (view is Button or ImageButton)
            {
                if (view.WidthRequest > 0 && view.WidthRequest < 44)
                    issues.Add(new AccessibilityIssue(
                        "Small Touch Target",
                        $"Touch target {view.WidthRequest}x{view.HeightRequest} < 44x44",
                        Severity.Warning));
            }
        }
        
        // Recurse into children
        if (element is IContainerView container)
        {
            foreach (var child in container)
                AuditElement(child, issues);
        }
        else if (element is Layout layout)
        {
            foreach (var child in layout.Children)
                AuditElement(child, issues);
        }
    }
    
    private static string GetElementPath(View view)
    {
        return view.GetType().Name + (view.ClassId != null ? $"#{view.ClassId}" : "");
    }
}

public record AccessibilityIssue(string Type, string Message, Severity Severity);
public enum Severity { Info, Warning, Error }
```

---

## Step 408: Localization & RTL

```csharp
// ============================================
// RTL (Right-to-Left) Layout Support
// ============================================

public class RtlLayoutService
{
    private readonly LocalizationService _localization;
    
    public RtlLayoutService(LocalizationService localization)
        => _localization = localization;
    
    public bool IsRtl => _localization.CurrentCulture.TextInfo.IsRightToLeft;
    
    public FlowDirection FlowDirection =>
        IsRtl ? FlowDirection.RightToLeft : FlowDirection.LeftToRight;
    
    public void ApplyToPage(ContentPage page)
    {
        page.FlowDirection = FlowDirection;
    }
}

// RTL-aware layout
public class RtlAwareLayout
{
    private static FlowDirection SystemFlowDirection
        => CultureInfo.CurrentUICulture.TextInfo.IsRightToLeft
            ? FlowDirection.RightToLeft
            : FlowDirection.LeftToRight;
    
    // Flip horizontal margins for RTL
    public static Thickness GetRtlMargin(
        double left, double top, double right, double bottom)
    {
        return SystemFlowDirection == FlowDirection.RightToLeft
            ? new Thickness(right, top, left, bottom)
            : new Thickness(left, top, right, bottom);
    }
    
    // Flip icon for RTL (e.g. back arrow)
    public static bool ShouldFlipIcon(string iconType)
    {
        var flipTypes = new HashSet<string> { "back", "forward", "arrow", "chevron", "check" };
        return SystemFlowDirection == FlowDirection.RightToLeft &&
               flipTypes.Any(t => iconType.Contains(t, StringComparison.OrdinalIgnoreCase));
    }
}

// XAML usage
/*
<ContentPage FlowDirection="{Binding FlowDirection}">
    <Image Source="back_arrow.png"
           ScaleX="{Binding IsRtl, Converter={StaticResource BoolToFlipConverter}}" />
</ContentPage>
*/
```

---

## Step 409: Color Contrast

```csharp
// ============================================
// WCAG Color Contrast Checker
// ============================================

public class ColorContrastService
{
    // WCAG 2.1 minimum contrast ratios
    private const double NormalTextMinRatio = 4.5;
    private const double LargeTextMinRatio = 3.0;
    private const double UIComponentMinRatio = 3.0;
    
    public ContrastResult CheckContrast(Color foreground, Color background)
    {
        var fgLuminance = GetRelativeLuminance(foreground);
        var bgLuminance = GetRelativeLuminance(background);
        
        var ratio = fgLuminance > bgLuminance
            ? (fgLuminance + 0.05) / (bgLuminance + 0.05)
            : (bgLuminance + 0.05) / (fgLuminance + 0.05);
        
        return new ContrastResult(
            Ratio: ratio,
            PassesNormalText: ratio >= NormalTextMinRatio,
            PassesLargeText: ratio >= LargeTextMinRatio,
            PassesUIComponents: ratio >= UIComponentMinRatio,
            Level: ratio >= 7 ? "AAA" : ratio >= 4.5 ? "AA" : ratio >= 3 ? "AA Large" : "Fail");
    }
    
    private static double GetRelativeLuminance(Color color)
    {
        double r = Linearize(color.Red);
        double g = Linearize(color.Green);
        double b = Linearize(color.Blue);
        
        return 0.2126 * r + 0.7152 * g + 0.0722 * b;
    }
    
    private static double Linearize(float channel)
    {
        double c = channel;
        return c <= 0.04045 ? c / 12.92 : Math.Pow((c + 0.055) / 1.055, 2.4);
    }
    
    // Auto-select text color with best contrast
    public Color GetAccessibleTextColor(Color background)
    {
        var whiteContrast = CheckContrast(Colors.White, background);
        var blackContrast = CheckContrast(Colors.Black, background);
        
        return whiteContrast.Ratio >= blackContrast.Ratio ? Colors.White : Colors.Black;
    }
}

public record ContrastResult(
    double Ratio, bool PassesNormalText, bool PassesLargeText,
    bool PassesUIComponents, string Level);
```

---

## Step 410: Accessibility Checklist & Testing

```csharp
// ============================================
// Comprehensive A11y Test Suite
// ============================================

public class AccessibilityChecklist
{
    public static List<CheckItem> GetChecklist() =>
    [
        new("Touch targets >= 44x44dp", Category.Touch),
        new("All images have alt text", Category.Images),
        new("Interactive elements have labels", Category.Labels),
        new("Color contrast >= 4.5:1 for normal text", Category.Color),
        new("Color contrast >= 3:1 for large text", Category.Color),
        new("Reading order is logical", Category.Navigation),
        new("Keyboard/switch accessible", Category.Navigation),
        new("Screen reader announces state changes", Category.ScreenReader),
        new("Errors announced by screen reader", Category.ScreenReader),
        new("Loading states announced", Category.ScreenReader),
        new("Headings used hierarchically (H1, H2...)", Category.Structure),
        new("Form labels associated with inputs", Category.Forms),
        new("Required fields indicated", Category.Forms),
        new("Error messages clear and specific", Category.Forms),
        new("RTL layout supported (if applicable)", Category.Localization),
        new("Large text doesn't break layout", Category.Visual),
        new("Content visible without color alone", Category.Color),
        new("Animations respect reduce motion", Category.Motion),
    ];
    
    public record CheckItem(string Description, Category Category, bool Passed = false);
    public enum Category { Touch, Images, Labels, Color, Navigation, ScreenReader, Structure, Forms, Localization, Visual, Motion }
}

// UITest helper
public static class AccessibilityUiTests
{
    // Check element description (in UI tests)
    public static void AssertHasDescription(IApp app, string automationId, string expected)
    {
        // Example using Xamarin.UITest / MAUI UITest
        // app.Query(c => c.Marked(automationId)).First().Description
        //    .Should().Contain(expected);
    }
    
    public static void AssertTouchTargetSize(IApp app, string automationId,
        int minWidth = 44, int minHeight = 44)
    {
        // Check element size
        // var rect = app.Query(c => c.Marked(automationId)).First().Rect;
        // rect.Width.Should().BeGreaterThanOrEqualTo(minWidth);
        // rect.Height.Should().BeGreaterThanOrEqualTo(minHeight);
    }
}
```

---

## สรุป Part 41

ใน Part 41 เราได้เรียนรู้:

1. **A11y Overview** - TalkBack, VoiceOver, WCAG
2. **SemanticProperties** - Description, Hint, HeadingLevel
3. **Focus Management** - TabIndex, ReturnType, programmatic focus
4. **Screen Reader** - AnnounceAsync for dynamic changes
5. **High Contrast/Large Text** - Detect and respond to settings
6. **Custom Controls** - Accessible rating control
7. **Automated Audit** - Scan pages for issues
8. **RTL Support** - FlowDirection for Arabic/Hebrew
9. **Color Contrast** - WCAG 4.5:1 checker
10. **Testing Checklist** - Complete a11y test guide

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 41 | Steps 401-410*
