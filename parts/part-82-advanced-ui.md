# Part 82: Advanced UI Techniques
## Steps 811-820: Glassmorphism, Neumorphism, Dark Mode, Adaptive Layouts

---

## Step 811: Dark Mode Support

```csharp
// ============================================
// Complete Dark Mode Implementation
// ============================================

// App.xaml – define theme-aware resources
// <Application.Resources>
//   <ResourceDictionary>
//     <Color x:Key="Background" Light="#F2F2F7" Dark="#000000" />
//     ... use AppThemeBinding for all colors

// Theme manager
public class ThemeManager
{
    private static AppTheme _currentTheme = AppTheme.Unspecified;
    
    public static AppTheme CurrentTheme => _currentTheme;
    
    public static void ApplyTheme(AppTheme theme)
    {
        _currentTheme = theme;
        
        if (Application.Current == null) return;
        
        Application.Current.UserAppTheme = theme;
        Preferences.Set("app_theme", (int)theme);
        
        WeakReferenceMessenger.Default.Send(new ThemeChangedMessage(theme));
    }
    
    public static void RestoreTheme()
    {
        var saved = Preferences.Get("app_theme", (int)AppTheme.Unspecified);
        ApplyTheme((AppTheme)saved);
    }
    
    public static bool IsDark =>
        _currentTheme == AppTheme.Dark ||
        (_currentTheme == AppTheme.Unspecified &&
         Application.Current?.RequestedTheme == AppTheme.Dark);
}

public record ThemeChangedMessage(AppTheme Theme);

// In XAML: use AppThemeBinding
// BackgroundColor="{AppThemeBinding Light=#FFFFFF, Dark=#1C1C1E}"
// TextColor="{AppThemeBinding Light=#000000, Dark=#FFFFFF}"
```

---

## Step 812: Glassmorphism Effect

```csharp
// ============================================
// Glassmorphism Card
// ============================================

public class GlassmorphismCard : ContentView
{
    private readonly BoxView _blur;
    private readonly Frame _glass;
    
    public GlassmorphismCard()
    {
        // Frosted glass effect using semi-transparent overlays
        _blur = new BoxView
        {
            BackgroundColor = Color.FromArgb("#20FFFFFF"),
        };
        
        _glass = new Frame
        {
            BackgroundColor = Color.FromArgb("#30FFFFFF"),
            BorderColor = Color.FromArgb("#40FFFFFF"),
            CornerRadius = 20,
            HasShadow = false,
            Padding = 20,
            Shadow = new Shadow
            {
                Brush = Brush.White,
                Opacity = 0.3f,
                Radius = 15,
                Offset = new Point(0, 4)
            }
        };
        
        var grid = new Grid();
        grid.Add(_blur);
        grid.Add(_glass);
        
        Content = grid;
        ClipsToBounds = true;
    }
    
    public new View? Content
    {
        get => _glass.Content;
        set => _glass.Content = value;
    }
}

// Frosted background page
public class FrostedPage : ContentPage
{
    public FrostedPage()
    {
        Background = new LinearGradientBrush(
            new GradientStopCollection
            {
                new GradientStop(Color.FromArgb("#667eea"), 0),
                new GradientStop(Color.FromArgb("#764ba2"), 1)
            },
            new Point(0, 0), new Point(1, 1));
        
        Content = new VerticalStackLayout
        {
            Padding = 24,
            Spacing = 16,
            Children =
            {
                new GlassmorphismCard
                {
                    Content = new VerticalStackLayout
                    {
                        Children =
                        {
                            new Label { Text = "ยอดออเดอร์วันนี้",
                                TextColor = Colors.White, FontSize = 14 },
                            new Label { Text = "฿12,500",
                                TextColor = Colors.White, FontSize = 32,
                                FontAttributes = FontAttributes.Bold }
                        }
                    }
                }
            }
        };
    }
}
```

---

## Step 813: Neumorphism Design

```csharp
// ============================================
// Neumorphic Button
// ============================================

public class NeumorphicButton : ContentView
{
    private static readonly Color BackgroundColor = Color.FromArgb("#E8ECF0");
    private static readonly Color DarkShadow = Color.FromArgb("#B8BECE");
    private static readonly Color LightShadow = Color.FromArgb("#FFFFFF");
    
    private readonly Frame _outer;
    
    public string Text { get; set; } = "";
    
    public event EventHandler? Clicked;
    
    public NeumorphicButton()
    {
        _outer = new Frame
        {
            BackgroundColor = BackgroundColor,
            CornerRadius = 12,
            HasShadow = false,
            Padding = new Thickness(20, 14),
            // Note: True neumorphism requires platform-specific shadow layers
            // This is an approximation using MAUI Shadow
            Shadow = new Shadow
            {
                Brush = new SolidColorBrush(LightShadow),
                Radius = 10,
                Opacity = 0.8f,
                Offset = new Point(-6, -6)
            }
        };
        
        var label = new Label
        {
            Text = Text, TextColor = Color.FromArgb("#666666"),
            FontSize = 16, HorizontalOptions = LayoutOptions.Center
        };
        
        _outer.Content = label;
        Content = _outer;
        
        var tap = new TapGestureRecognizer();
        tap.Tapped += async (_, _) =>
        {
            await PressAnimationAsync();
            Clicked?.Invoke(this, EventArgs.Empty);
        };
        GestureRecognizers.Add(tap);
    }
    
    private async Task PressAnimationAsync()
    {
        // Pressed: flip shadow direction
        _outer.Shadow = new Shadow
        {
            Brush = new SolidColorBrush(DarkShadow),
            Radius = 4,
            Opacity = 0.5f,
            Offset = new Point(3, 3)
        };
        
        await Task.Delay(150);
        
        // Released: restore
        _outer.Shadow = new Shadow
        {
            Brush = new SolidColorBrush(LightShadow),
            Radius = 10,
            Opacity = 0.8f,
            Offset = new Point(-6, -6)
        };
    }
}
```

---

## Step 814: Adaptive Layout

```csharp
// ============================================
// Adaptive Layout (Phone/Tablet/Desktop)
// ============================================

public class AdaptiveLayoutPage : ContentPage
{
    private readonly FlexLayout _mainLayout;
    
    public AdaptiveLayoutPage()
    {
        _mainLayout = new FlexLayout
        {
            Wrap = FlexWrap.Wrap,
            JustifyContent = FlexJustify.Start,
            AlignItems = FlexAlignItems.Start
        };
        
        Content = new ScrollView { Content = _mainLayout };
        
        SizeChanged += OnSizeChanged;
        PopulateCards();
    }
    
    private void OnSizeChanged(object? sender, EventArgs e)
    {
        var width = Width;
        var cardWidth = width switch
        {
            < 600 => width - 32,       // Phone: 1 column
            < 1024 => (width - 48) / 2, // Tablet: 2 columns
            _ => (width - 64) / 3       // Desktop: 3 columns
        };
        
        foreach (var child in _mainLayout.Children.OfType<View>())
        {
            FlexLayout.SetBasis(child, new FlexBasis((float)cardWidth));
        }
    }
    
    private void PopulateCards()
    {
        for (int i = 0; i < 12; i++)
        {
            var card = CreateCard($"ร้านอาหาร {i + 1}");
            FlexLayout.SetBasis(card, new FlexBasis(1, true)); // start flexible
            _mainLayout.Add(card);
        }
    }
    
    private View CreateCard(string name) => new Frame
    {
        Margin = 8, CornerRadius = 12,
        Content = new Label { Text = name, FontSize = 14 }
    };
}

// Breakpoint helper
public static class ScreenBreakpoint
{
    public static bool IsPhone => DeviceDisplay.MainDisplayInfo.Width / DeviceDisplay.MainDisplayInfo.Density < 600;
    public static bool IsTablet => !IsPhone && DeviceDisplay.MainDisplayInfo.Width / DeviceDisplay.MainDisplayInfo.Density < 1024;
    public static bool IsDesktop => DeviceDisplay.MainDisplayInfo.Width / DeviceDisplay.MainDisplayInfo.Density >= 1024;
    
    public static T Choose<T>(T phone, T tablet, T desktop) =>
        IsPhone ? phone : IsTablet ? tablet : desktop;
}
```

---

## Step 815: Micro-Interactions

```csharp
// ============================================
// Micro-Interaction Library
// ============================================

public static class MicroInteractions
{
    // Like button: heart burst
    public static async Task HeartAsync(View heart)
    {
        heart.TextColor = Colors.Red; // assumes Label
        await Task.WhenAll(
            heart.ScaleTo(1.4, 100, Easing.CubicOut),
            heart.RotateTo(20, 50));
        await Task.WhenAll(
            heart.ScaleTo(1, 100, Easing.SpringOut),
            heart.RotateTo(0, 50));
    }
    
    // Success checkmark animation
    public static async Task SuccessCheckAsync(View check)
    {
        check.Opacity = 0;
        check.Scale = 0.3;
        check.IsVisible = true;
        
        await Task.WhenAll(
            check.FadeTo(1, 200),
            check.ScaleTo(1.2, 200, Easing.CubicOut));
        await check.ScaleTo(1, 100, Easing.SpringIn);
    }
    
    // Error shake
    public static async Task ErrorShakeAsync(View view)
    {
        var original = view.TranslationX;
        for (int i = 0; i < 3; i++)
        {
            await view.TranslateTo(original + 10, 0, 50);
            await view.TranslateTo(original - 10, 0, 50);
        }
        await view.TranslateTo(original, 0, 50);
    }
    
    // Button press ripple
    public static async Task RippleAsync(View button, Color rippleColor)
    {
        var ripple = new BoxView
        {
            BackgroundColor = rippleColor.WithAlpha(0.3f),
            CornerRadius = 999,
            WidthRequest = 0, HeightRequest = 0,
            HorizontalOptions = LayoutOptions.Center,
            VerticalOptions = LayoutOptions.Center
        };
        
        if (button.Parent is Layout layout)
        {
            layout.Add(ripple);
            await Task.WhenAll(
                ripple.ScaleTo(3, 300),
                ripple.FadeTo(0, 300));
            layout.Remove(ripple);
        }
    }
    
    // Count-up animation
    public static async Task CountUpAsync(Label label, int from, int to, uint duration = 1000)
    {
        var steps = 30;
        var delay = (int)(duration / steps);
        
        for (int i = 0; i <= steps; i++)
        {
            var value = from + (int)((to - from) * Math.Sqrt((double)i / steps));
            label.Text = ThaiFormatter.Currency(value);
            await Task.Delay(delay);
        }
        
        label.Text = ThaiFormatter.Currency(to);
    }
}
```

---

## Step 816: Responsive Typography

```csharp
// ============================================
// Responsive Type Scale
// ============================================

public static class TypeScale
{
    private static double _baseFontSize = 16;
    
    public static void SetBase(double size) => _baseFontSize = size;
    
    // Modular scale 1.25 (Major Third)
    private const double Ratio = 1.25;
    
    public static double Xs => _baseFontSize / (Ratio * Ratio);    // ~10
    public static double Sm => _baseFontSize / Ratio;              // ~13
    public static double Base => _baseFontSize;                    // 16
    public static double Md => _baseFontSize * Ratio;              // 20
    public static double Lg => _baseFontSize * Ratio * Ratio;      // 25
    public static double Xl => _baseFontSize * Math.Pow(Ratio, 3); // 31
    public static double Xxl => _baseFontSize * Math.Pow(Ratio, 4);// 39
    
    // Line height multipliers
    public static double LineHeightCompact => 1.2;
    public static double LineHeightNormal => 1.5;
    public static double LineHeightRelaxed => 1.75;
    
    // Responsive: scale with device
    public static double Responsive(double size)
    {
        var density = DeviceDisplay.MainDisplayInfo.Density;
        return size * Math.Clamp(density, 0.9, 1.2);
    }
}

// Apply type scale in style dictionary
public static class AppStyles
{
    public static Style CreateHeading1Style() => new Style(typeof(Label))
    {
        Setters =
        {
            new Setter { Property = Label.FontSizeProperty, Value = TypeScale.Xxl },
            new Setter { Property = Label.FontAttributesProperty, Value = FontAttributes.Bold },
            new Setter { Property = Label.LineHeightProperty, Value = TypeScale.LineHeightCompact }
        }
    };
    
    public static Style CreateBodyStyle() => new Style(typeof(Label))
    {
        Setters =
        {
            new Setter { Property = Label.FontSizeProperty, Value = TypeScale.Base },
            new Setter { Property = Label.LineHeightProperty, Value = TypeScale.LineHeightNormal }
        }
    };
}
```

---

## Step 817: Status Bar & Navigation Bar Styling

```csharp
// ============================================
// Platform Status Bar Theming
// ============================================

public class StatusBarService
{
    public static void SetDark()
    {
#if ANDROID
        SetAndroidStatusBar(Colors.Black, Colors.White, true);
#elif IOS
        UIKit.UIApplication.SharedApplication.StatusBarStyle =
            UIKit.UIStatusBarStyle.LightContent;
#endif
    }
    
    public static void SetLight()
    {
#if ANDROID
        SetAndroidStatusBar(Colors.White, Colors.Black, false);
#elif IOS
        UIKit.UIApplication.SharedApplication.StatusBarStyle =
            UIKit.UIStatusBarStyle.DarkContent;
#endif
    }
    
    public static void SetTransparent()
    {
#if ANDROID
        if (Android.App.Application.Context is Android.App.Activity activity)
        {
            activity.Window?.SetFlags(
                Android.Views.WindowManagerFlags.LayoutNoLimits,
                Android.Views.WindowManagerFlags.LayoutNoLimits);
        }
#endif
    }
    
#if ANDROID
    private static void SetAndroidStatusBar(Color bg, Color fg, bool darkIcons)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            if (Android.App.Application.Context is not Android.App.Activity activity) return;
            
            var window = activity.Window;
            if (window == null) return;
            
            window.SetStatusBarColor(bg.ToPlatform());
            
            var flags = window.DecorView.SystemUiFlags;
            if (darkIcons)
                flags |= Android.Views.SystemUiFlags.LightStatusBar;
            else
                flags &= ~Android.Views.SystemUiFlags.LightStatusBar;
            
            window.DecorView.SystemUiFlags = flags;
        });
    }
#endif
}
```

---

## Step 818: Custom Tab Bar

```csharp
// ============================================
// Custom Tab Bar with Floating Button
// ============================================

public class FloatingTabBar : ContentView
{
    private readonly HorizontalStackLayout _tabs;
    private readonly ObservableCollection<TabItem> _items;
    private int _selectedIndex;
    
    public event EventHandler<int>? TabSelected;
    
    public FloatingTabBar(IEnumerable<TabItem> items)
    {
        _items = new ObservableCollection<TabItem>(items);
        _tabs = new HorizontalStackLayout { Spacing = 0 };
        
        // Floating card effect
        var card = new Frame
        {
            BackgroundColor = Colors.White,
            CornerRadius = 30,
            HasShadow = true,
            Padding = new Thickness(8, 8),
            Margin = new Thickness(24, 0, 24, 32),
            Content = _tabs,
            Shadow = new Shadow { Blur = 20, Opacity = 0.2f, Offset = new Point(0, 8) }
        };
        
        Content = card;
        
        BuildTabs();
    }
    
    private void BuildTabs()
    {
        for (int i = 0; i < _items.Count; i++)
        {
            var idx = i;
            var item = _items[i];
            
            var btn = new VerticalStackLayout
            {
                HorizontalOptions = LayoutOptions.Center,
                Spacing = 2, Padding = new Thickness(16, 6),
                Children =
                {
                    new Image { Source = item.Icon, HeightRequest = 24, WidthRequest = 24 },
                    new Label { Text = item.Label, FontSize = 10, HorizontalOptions = LayoutOptions.Center }
                }
            };
            
            FlexLayout.SetGrow(btn, 1);
            
            btn.GestureRecognizers.Add(new TapGestureRecognizer
            {
                Command = new Command(() => SelectTab(idx))
            });
            
            _tabs.Children.Add(btn);
        }
    }
    
    private async void SelectTab(int idx)
    {
        if (idx == _selectedIndex) return;
        
        // Animate selection
        var previous = _tabs.Children.ElementAt(_selectedIndex) as VerticalStackLayout;
        var current = _tabs.Children.ElementAt(idx) as VerticalStackLayout;
        
        if (previous != null) await previous.ScaleTo(1, 100);
        if (current != null)
        {
            await current.ScaleTo(0.9, 50);
            await current.ScaleTo(1.05, 100, Easing.SpringOut);
            await current.ScaleTo(1, 50);
        }
        
        _selectedIndex = idx;
        TabSelected?.Invoke(this, idx);
    }
}

public record TabItem(string Label, string Icon);
```

---

## Step 819: Parallax Header

```csharp
// ============================================
// Parallax Header for Detail Pages
// ============================================

public class ParallaxHeaderPage : ContentPage
{
    private Image? _headerImage;
    private View? _stickyHeader;
    private const double HeaderHeight = 250;
    
    public ParallaxHeaderPage()
    {
        var scrollView = new ScrollView();
        scrollView.Scrolled += OnScrolled;
        
        _headerImage = new Image
        {
            Source = "restaurant_banner.jpg",
            Aspect = Aspect.AspectFill,
            HeightRequest = HeaderHeight
        };
        
        _stickyHeader = new Frame
        {
            BackgroundColor = Colors.White,
            Opacity = 0,
            HeightRequest = 60,
            Content = new Label { Text = "ร้านอาหาร", FontSize = 18, FontAttributes = FontAttributes.Bold }
        };
        
        var content = new VerticalStackLayout
        {
            Children =
            {
                _headerImage,
                new BoxView { HeightRequest = 800 } // placeholder content
            }
        };
        
        scrollView.Content = content;
        
        var grid = new Grid
        {
            RowDefinitions = Rows.Define(Auto, Star),
            Children = { scrollView, _stickyHeader }
        };
        
        Grid.SetRowSpan(scrollView, 2);
        
        Content = grid;
        NavigationPage.SetHasNavigationBar(this, false);
    }
    
    private void OnScrolled(object? sender, ScrolledEventArgs e)
    {
        var scrollY = e.ScrollY;
        
        // Parallax: header moves at half speed
        _headerImage!.TranslationY = scrollY * 0.5;
        
        // Fade header image out
        _headerImage.Opacity = Math.Max(0, 1 - scrollY / HeaderHeight);
        
        // Fade in sticky header
        _stickyHeader!.Opacity = Math.Clamp((scrollY - HeaderHeight * 0.5) / (HeaderHeight * 0.5), 0, 1);
    }
}
```

---

## Step 820: Bottom Sheet

```csharp
// ============================================
// Native-feeling Bottom Sheet
// ============================================

public class BottomSheet : ContentView
{
    private double _startY;
    private readonly Grid _backdrop;
    private readonly Frame _sheet;
    private bool _isOpen;
    
    public event EventHandler? Dismissed;
    
    public View? SheetContent
    {
        get => _sheet.Content;
        set => _sheet.Content = value;
    }
    
    public BottomSheet()
    {
        _backdrop = new Grid { BackgroundColor = Color.FromArgb("#80000000"), Opacity = 0 };
        _backdrop.GestureRecognizers.Add(new TapGestureRecognizer
        {
            Command = new Command(async () => await DismissAsync())
        });
        
        _sheet = new Frame
        {
            BackgroundColor = Colors.White,
            CornerRadius = 20,
            Padding = new Thickness(0, 8, 0, 0),
            HasShadow = true,
            TranslationY = 1000 // start off-screen
        };
        
        // Drag handle
        var handle = new BoxView
        {
            WidthRequest = 36, HeightRequest = 4, CornerRadius = 2,
            BackgroundColor = Color.FromArgb("#E0E0E0"),
            HorizontalOptions = LayoutOptions.Center, Margin = new Thickness(0, 4, 0, 8)
        };
        
        var layout = new VerticalStackLayout { Children = { handle } };
        // SheetContent will be added to layout
        _sheet.Content = layout;
        
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += OnPan;
        _sheet.GestureRecognizers.Add(pan);
        
        // Build overlay
        Content = new Grid { Children = { _backdrop, _sheet } };
        IsVisible = false;
    }
    
    public async Task ShowAsync()
    {
        IsVisible = true;
        _isOpen = true;
        
        await Task.WhenAll(
            _backdrop.FadeTo(1, 250),
            _sheet.TranslateTo(0, 0, 350, Easing.SpringOut));
    }
    
    public async Task DismissAsync()
    {
        _isOpen = false;
        
        await Task.WhenAll(
            _backdrop.FadeTo(0, 200),
            _sheet.TranslateTo(0, 800, 300, Easing.CubicIn));
        
        IsVisible = false;
        Dismissed?.Invoke(this, EventArgs.Empty);
    }
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        switch (e.StatusType)
        {
            case GestureStatus.Started:
                _startY = _sheet.TranslationY;
                break;
            
            case GestureStatus.Running:
                var newY = Math.Max(0, _startY + e.TotalY);
                _sheet.TranslationY = newY;
                _backdrop.Opacity = Math.Max(0, 1 - newY / 300);
                break;
            
            case GestureStatus.Completed:
                if (_sheet.TranslationY > 150)
                    _ = DismissAsync();
                else
                    _ = _sheet.TranslateTo(0, 0, 200, Easing.SpringOut);
                break;
        }
    }
}
```

---

## สรุป Part 82

ใน Part 82 เราได้เรียนรู้:

1. **Dark Mode** - AppTheme, UserAppTheme, AppThemeBinding, ThemeManager
2. **Glassmorphism** - Semi-transparent overlays, frosted glass effect
3. **Neumorphism** - Light/dark shadow layers, press animation
4. **Adaptive Layout** - FlexLayout responsive grid, breakpoint helper
5. **Micro-Interactions** - Heart burst, success check, shake, ripple, count-up
6. **Responsive Typography** - Modular type scale, line height tokens
7. **Status Bar Styling** - Platform-specific light/dark icons
8. **Custom Tab Bar** - Floating rounded card, spring animation
9. **Parallax Header** - Half-speed scroll, opacity fade
10. **Bottom Sheet** - Pan gesture dismiss, backdrop, spring reveal

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 82 | Steps 811-820*
