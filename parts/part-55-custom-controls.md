# Part 55: Custom Controls & UI Components
## Steps 541-550: Handlers, Renderers, Design System

---

## Step 541: Custom Handlers (MAUI)

```csharp
// ============================================
// Custom Handler for Platform Control
// ============================================

// Custom Entry with no underline (Android)
public class BorderlessEntry : Entry { }

public class BorderlessEntryHandler : EntryHandler
{
    protected override void ConnectHandler(
        Microsoft.Maui.Platform.AppCompatEditText platformView)
    {
        base.ConnectHandler(platformView);
#if ANDROID
        platformView.Background = null;
#endif
    }
}

#if IOS
public class BorderlessEntryHandler : EntryHandler
{
    protected override UIKit.UITextField CreatePlatformView()
    {
        var field = base.CreatePlatformView();
        field.BorderStyle = UIKit.UITextBorderStyle.None;
        return field;
    }
}
#endif

// Register in MauiProgram.cs
// builder.ConfigureMauiHandlers(h =>
//     h.AddHandler<BorderlessEntry, BorderlessEntryHandler>());

// Custom video player
public class VideoPlayer : View
{
    public static readonly BindableProperty SourceProperty =
        BindableProperty.Create(nameof(Source), typeof(string), typeof(VideoPlayer));
    
    public static readonly BindableProperty IsPlayingProperty =
        BindableProperty.Create(nameof(IsPlaying), typeof(bool), typeof(VideoPlayer),
            propertyChanged: OnIsPlayingChanged);
    
    public string? Source
    {
        get => (string?)GetValue(SourceProperty);
        set => SetValue(SourceProperty, value);
    }
    
    public bool IsPlaying
    {
        get => (bool)GetValue(IsPlayingProperty);
        set => SetValue(IsPlayingProperty, value);
    }
    
    private static void OnIsPlayingChanged(BindableObject bindable, object oldValue, object newValue)
    {
        // Handler will respond
    }
}

public class VideoPlayerHandler : ViewHandler<VideoPlayer, object>
{
    public VideoPlayerHandler() : base(VideoPlayerMapper) { }
    
    public static PropertyMapper<VideoPlayer, VideoPlayerHandler> VideoPlayerMapper = new(ViewMapper)
    {
        [nameof(VideoPlayer.Source)] = MapSource,
        [nameof(VideoPlayer.IsPlaying)] = MapIsPlaying,
    };
    
    protected override object CreatePlatformView() => throw new NotImplementedException();
    
    private static void MapSource(VideoPlayerHandler handler, VideoPlayer player) { }
    private static void MapIsPlaying(VideoPlayerHandler handler, VideoPlayer player) { }
}
```

---

## Step 542: BindableProperty Patterns

```csharp
// ============================================
// Custom BindableProperty Patterns
// ============================================

public class RatingView : ContentView
{
    // Standard BindableProperty
    public static readonly BindableProperty ValueProperty =
        BindableProperty.Create(
            propertyName: nameof(Value),
            returnType: typeof(int),
            declaringType: typeof(RatingView),
            defaultValue: 0,
            defaultBindingMode: BindingMode.TwoWay,
            validateValue: (_, v) => (int)v >= 0 && (int)v <= 5,
            propertyChanged: OnValueChanged,
            coerceValue: (_, v) => Math.Clamp((int)v, 0, 5));
    
    public static readonly BindableProperty MaxProperty =
        BindableProperty.Create(nameof(Max), typeof(int), typeof(RatingView), 5);
    
    public static readonly BindableProperty StarColorProperty =
        BindableProperty.Create(nameof(StarColor), typeof(Color), typeof(RatingView), Colors.Gold);
    
    // Attached property
    public static readonly BindableProperty IsReadOnlyProperty =
        BindableProperty.CreateAttached("IsReadOnly", typeof(bool), typeof(RatingView), false);
    
    public static bool GetIsReadOnly(BindableObject view) =>
        (bool)view.GetValue(IsReadOnlyProperty);
    public static void SetIsReadOnly(BindableObject view, bool value) =>
        view.SetValue(IsReadOnlyProperty, value);
    
    public int Value
    {
        get => (int)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public int Max
    {
        get => (int)GetValue(MaxProperty);
        set => SetValue(MaxProperty, value);
    }
    
    public Color StarColor
    {
        get => (Color)GetValue(StarColorProperty);
        set => SetValue(StarColorProperty, value);
    }
    
    // ValueChanged event
    public event EventHandler<ValueChangedEventArgs>? ValueChanged;
    
    private static void OnValueChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var view = (RatingView)bindable;
        view.UpdateStars();
        view.ValueChanged?.Invoke(view, new ValueChangedEventArgs((int)oldValue, (int)newValue));
    }
    
    public RatingView()
    {
        BuildUI();
    }
    
    private HorizontalStackLayout _starsLayout = null!;
    
    private void BuildUI()
    {
        _starsLayout = new HorizontalStackLayout { Spacing = 4 };
        Content = _starsLayout;
        UpdateStars();
    }
    
    private void UpdateStars()
    {
        _starsLayout.Children.Clear();
        for (int i = 1; i <= Max; i++)
        {
            var star = i <= Value ? "⭐" : "☆";
            var idx = i;
            var label = new Label
            {
                Text = star,
                FontSize = 24,
                GestureRecognizers =
                {
                    new TapGestureRecognizer
                    {
                        Command = new Command(() => Value = idx)
                    }
                }
            };
            _starsLayout.Children.Add(label);
        }
    }
}
```

---

## Step 543: Design System

```csharp
// ============================================
// Design System: Tokens + Components
// ============================================

// ResourceDictionary - design tokens
public class DesignTokens : ResourceDictionary
{
    public DesignTokens()
    {
        // Color palette
        Add("Primary", Color.FromArgb("#6750A4"));
        Add("OnPrimary", Color.FromArgb("#FFFFFF"));
        Add("PrimaryContainer", Color.FromArgb("#EADDFF"));
        Add("Secondary", Color.FromArgb("#625B71"));
        Add("Surface", Color.FromArgb("#FFFBFE"));
        Add("OnSurface", Color.FromArgb("#1C1B1F"));
        Add("SurfaceVariant", Color.FromArgb("#E7E0EC"));
        Add("Error", Color.FromArgb("#B3261E"));
        Add("Success", Color.FromArgb("#198038"));
        Add("Warning", Color.FromArgb("#FF8800"));
        
        // Spacing
        Add("SpaceXS", 4.0);
        Add("SpaceS", 8.0);
        Add("SpaceM", 16.0);
        Add("SpaceL", 24.0);
        Add("SpaceXL", 32.0);
        
        // Typography
        Add("FontDisplay", 57.0);
        Add("FontHeadline", 32.0);
        Add("FontTitle", 22.0);
        Add("FontBody", 16.0);
        Add("FontLabel", 14.0);
        Add("FontCaption", 12.0);
        
        // Border radius
        Add("RadiusS", 4.0);
        Add("RadiusM", 8.0);
        Add("RadiusL", 12.0);
        Add("RadiusXL", 16.0);
        Add("RadiusFull", 100.0);
        
        // Elevation (shadows)
        Add("ElevationS", new Shadow { Offset = new Point(0, 1), Radius = 2, Opacity = 0.1f });
        Add("ElevationM", new Shadow { Offset = new Point(0, 2), Radius = 6, Opacity = 0.15f });
        Add("ElevationL", new Shadow { Offset = new Point(0, 4), Radius = 12, Opacity = 0.2f });
    }
}
```

---

## Step 544: Component Library

```csharp
// ============================================
// Reusable Component Library
// ============================================

// Primary Button
public class PrimaryButton : Button
{
    public PrimaryButton()
    {
        BackgroundColor = (Color)Application.Current!.Resources["Primary"];
        TextColor = (Color)Application.Current!.Resources["OnPrimary"];
        CornerRadius = 20;
        HeightRequest = 48;
        FontAttributes = FontAttributes.Bold;
        Padding = new Thickness(24, 0);
    }
}

// Outlined Button
public class OutlinedButton : Button
{
    public OutlinedButton()
    {
        BackgroundColor = Colors.Transparent;
        TextColor = (Color)Application.Current!.Resources["Primary"];
        BorderColor = (Color)Application.Current!.Resources["Primary"];
        BorderWidth = 1.5;
        CornerRadius = 20;
        HeightRequest = 48;
        Padding = new Thickness(24, 0);
    }
}

// Card component
public class Card : Border
{
    public Card()
    {
        BackgroundColor = (Color)Application.Current!.Resources["Surface"];
        StrokeShape = new RoundRectangle { CornerRadius = new CornerRadius(12) };
        Stroke = Colors.Transparent;
        Shadow = (Shadow)Application.Current!.Resources["ElevationM"];
        Padding = 16;
    }
}

// Chip (tag/filter)
public class Chip : Border
{
    private readonly Label _label;
    
    public static readonly BindableProperty TextProperty =
        BindableProperty.Create(nameof(Text), typeof(string), typeof(Chip), string.Empty,
            propertyChanged: (b, _, n) => ((Chip)b)._label.Text = (string)n);
    
    public static readonly BindableProperty IsSelectedProperty =
        BindableProperty.Create(nameof(IsSelected), typeof(bool), typeof(Chip), false,
            propertyChanged: (b, _, n) => ((Chip)b).UpdateStyle((bool)n));
    
    public string Text
    {
        get => (string)GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }
    
    public bool IsSelected
    {
        get => (bool)GetValue(IsSelectedProperty);
        set => SetValue(IsSelectedProperty, value);
    }
    
    public Command? SelectCommand { get; set; }
    
    public Chip()
    {
        _label = new Label { FontSize = 14, Margin = new Thickness(12, 6) };
        Content = _label;
        
        StrokeShape = new RoundRectangle { CornerRadius = new CornerRadius(16) };
        Padding = 0;
        
        UpdateStyle(false);
        
        GestureRecognizers.Add(new TapGestureRecognizer
        {
            Command = new Command(() =>
            {
                IsSelected = !IsSelected;
                SelectCommand?.Execute(this);
            })
        });
    }
    
    private void UpdateStyle(bool selected)
    {
        if (selected)
        {
            BackgroundColor = (Color)Application.Current!.Resources["Primary"];
            Stroke = Colors.Transparent;
            _label.TextColor = Colors.White;
        }
        else
        {
            BackgroundColor = Colors.Transparent;
            Stroke = (Color)Application.Current!.Resources["Primary"];
            _label.TextColor = (Color)Application.Current!.Resources["Primary"];
        }
    }
}
```

---

## Step 545: Skeleton Loading

```csharp
// ============================================
// Skeleton Screen Loading
// ============================================

public class SkeletonView : ContentView
{
    private Animation? _shimmerAnimation;
    private float _shimmerPosition = -1.0f;
    
    public static readonly BindableProperty IsShimmeringProperty =
        BindableProperty.Create(nameof(IsShimmering), typeof(bool), typeof(SkeletonView), true,
            propertyChanged: (b, _, n) =>
            {
                var view = (SkeletonView)b;
                if ((bool)n) view.StartShimmer();
                else view.StopShimmer();
            });
    
    public bool IsShimmering
    {
        get => (bool)GetValue(IsShimmeringProperty);
        set => SetValue(IsShimmeringProperty, value);
    }
    
    public SkeletonView()
    {
        BackgroundColor = Color.FromArgb("#E0E0E0");
        StartShimmer();
    }
    
    private void StartShimmer()
    {
        _shimmerAnimation?.Dispose();
        _shimmerAnimation = new Animation(
            v => { _shimmerPosition = (float)v; InvalidateMeasure(); },
            -1, 2, Easing.Linear);
        _shimmerAnimation.Commit(this, "shimmer", 16, 1200,
            repeat: () => IsShimmering);
    }
    
    private void StopShimmer() => this.AbortAnimation("shimmer");
}

// Skeleton product card
public class SkeletonProductCard : ContentView
{
    public SkeletonProductCard()
    {
        Content = new Grid
        {
            Padding = 16,
            ColumnDefinitions = GridRowsColumns.Columns.Define(60, Star),
            RowDefinitions = GridRowsColumns.Rows.Define(Auto, Auto, Auto),
            Children =
            {
                new SkeletonView
                {
                    WidthRequest = 50, HeightRequest = 50,
                    CornerRadius = 25
                }.Column(0).RowSpan(3),
                
                new SkeletonView
                {
                    HeightRequest = 16, Margin = new Thickness(12, 0, 0, 8)
                }.Column(1).Row(0),
                
                new SkeletonView
                {
                    HeightRequest = 12, Margin = new Thickness(12, 0, 60, 8)
                }.Column(1).Row(1),
                
                new SkeletonView
                {
                    HeightRequest = 12, Margin = new Thickness(12, 0, 100, 0)
                }.Column(1).Row(2),
            }
        };
    }
}
```

---

## Step 546: Toast & Snackbar

```csharp
// ============================================
// Toast & Snackbar Notifications
// ============================================

public class ToastService
{
    public static Task ShowAsync(string message, int durationMs = 2000)
    {
        return MainThread.InvokeOnMainThreadAsync(async () =>
        {
            var toast = CreateToast(message);
            var overlay = GetOverlay();
            
            overlay.Children.Add(toast);
            
            // Animate in
            toast.Opacity = 0;
            toast.TranslationY = 50;
            await Task.WhenAll(
                toast.FadeTo(1, 300),
                toast.TranslateTo(0, 0, 300, Easing.CubicOut));
            
            await Task.Delay(durationMs);
            
            // Animate out
            await Task.WhenAll(
                toast.FadeTo(0, 300),
                toast.TranslateTo(0, 50, 300, Easing.CubicIn));
            
            overlay.Children.Remove(toast);
        });
    }
    
    private static View CreateToast(string message)
    {
        return new Border
        {
            BackgroundColor = Color.FromArgb("#323232"),
            StrokeShape = new RoundRectangle { CornerRadius = 8 },
            Stroke = Colors.Transparent,
            Padding = new Thickness(16, 10),
            Margin = new Thickness(24, 0, 24, 48),
            HorizontalOptions = LayoutOptions.Center,
            VerticalOptions = LayoutOptions.End,
            Content = new Label
            {
                Text = message,
                TextColor = Colors.White,
                FontSize = 14
            }
        };
    }
    
    private static AbsoluteLayout GetOverlay()
    {
        // Get or create overlay on current page
        var page = Shell.Current.CurrentPage;
        if (page.Content is AbsoluteLayout overlay) return overlay;
        
        var newOverlay = new AbsoluteLayout();
        AbsoluteLayout.SetLayoutBounds(page.Content, new Rect(0, 0, 1, 1));
        AbsoluteLayout.SetLayoutFlags(page.Content, AbsoluteLayoutFlags.All);
        
        var root = new AbsoluteLayout();
        root.Children.Add(page.Content);
        page.Content = root;
        
        return root;
    }
    
    // Snackbar with action
    public static async Task ShowSnackbarAsync(
        string message, string? actionText = null, Func<Task>? onAction = null,
        int durationMs = 4000)
    {
        // Use CommunityToolkit.Maui Snackbar if available
        // var snackbar = Snackbar.Make(message, onAction, actionText, TimeSpan.FromMilliseconds(durationMs));
        // await snackbar.Show();
        
        await ShowAsync(message, durationMs);
    }
}
```

---

## Step 547: Bottom Sheet

```csharp
// ============================================
// Custom Bottom Sheet
// ============================================

public class BottomSheet : ContentView
{
    private double _startY;
    private double _currentHeight;
    private double _maxHeight => (Shell.Current.CurrentPage.Height * 0.9);
    private double _minHeight = 200;
    
    public event Action? Dismissed;
    
    public static readonly BindableProperty TitleProperty =
        BindableProperty.Create(nameof(Title), typeof(string), typeof(BottomSheet));
    
    public string? Title
    {
        get => (string?)GetValue(TitleProperty);
        set => SetValue(TitleProperty, value);
    }
    
    private readonly ContentView _contentContainer = new();
    
    public View? SheetContent
    {
        get => _contentContainer.Content;
        set => _contentContainer.Content = value;
    }
    
    public BottomSheet()
    {
        BuildUI();
    }
    
    private void BuildUI()
    {
        var handle = new BoxView
        {
            WidthRequest = 40, HeightRequest = 4,
            CornerRadius = 2,
            BackgroundColor = Colors.Gray,
            HorizontalOptions = LayoutOptions.Center,
            Margin = new Thickness(0, 8, 0, 0)
        };
        
        var dragRecognizer = new PanGestureRecognizer();
        dragRecognizer.PanUpdated += OnPan;
        handle.GestureRecognizers.Add(dragRecognizer);
        
        Content = new Border
        {
            BackgroundColor = Colors.White,
            StrokeShape = new RoundRectangle
            {
                CornerRadius = new CornerRadius(16, 16, 0, 0)
            },
            Stroke = Colors.Transparent,
            Content = new VerticalStackLayout
            {
                Children =
                {
                    handle,
                    new Label
                    {
                        Text = Title,
                        FontSize = 18,
                        FontAttributes = FontAttributes.Bold,
                        Margin = new Thickness(16, 8)
                    },
                    _contentContainer
                }
            }
        };
    }
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        switch (e.StatusType)
        {
            case GestureStatus.Started:
                _startY = TranslationY;
                break;
            case GestureStatus.Running:
                TranslationY = Math.Max(0, _startY + e.TotalY);
                break;
            case GestureStatus.Completed:
                if (TranslationY > Height * 0.4)
                    DismissAsync();
                else
                    TranslateTo(0, 0, 250, Easing.CubicOut);
                break;
        }
    }
    
    public async Task ShowAsync()
    {
        TranslationY = Height;
        IsVisible = true;
        await TranslateTo(0, 0, 350, Easing.CubicOut);
    }
    
    public async Task DismissAsync()
    {
        await TranslateTo(0, Height, 300, Easing.CubicIn);
        IsVisible = false;
        Dismissed?.Invoke();
    }
}
```

---

## Step 548: Data Visualization

```csharp
// ============================================
// Custom Charts with SkiaSharp
// ============================================

public class BarChartView : SKCanvasView
{
    public static readonly BindableProperty DataProperty =
        BindableProperty.Create(nameof(Data), typeof(List<BarData>), typeof(BarChartView),
            propertyChanged: (b, _, _) => ((BarChartView)b).InvalidateSurface());
    
    public List<BarData>? Data
    {
        get => (List<BarData>?)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);
        
        if (Data == null || Data.Count == 0) return;
        
        var info = e.Info;
        var padding = 40f;
        var chartWidth = info.Width - padding * 2;
        var chartHeight = info.Height - padding * 2;
        
        var maxValue = Data.Max(d => d.Value);
        var barWidth = chartWidth / Data.Count * 0.7f;
        var spacing = chartWidth / Data.Count * 0.3f;
        
        using var barPaint = new SKPaint { IsAntialias = true };
        using var textPaint = new SKPaint
        {
            Color = SKColors.DarkGray,
            TextSize = 12,
            IsAntialias = true
        };
        
        for (int i = 0; i < Data.Count; i++)
        {
            var item = Data[i];
            var barHeight = (float)(item.Value / maxValue * chartHeight);
            var x = padding + i * (barWidth + spacing);
            var y = padding + chartHeight - barHeight;
            
            barPaint.Color = SKColor.Parse(item.Color ?? "#6750A4");
            canvas.DrawRoundRect(x, y, barWidth, barHeight, 4, 4, barPaint);
            
            // Label
            canvas.DrawText(item.Label, x, info.Height - 8, textPaint);
            
            // Value
            canvas.DrawText(item.Value.ToString("N0"), x, y - 4, textPaint);
        }
    }
}

public record BarData(string Label, double Value, string? Color = null);

// Line chart
public class LineChartView : SKCanvasView
{
    public static readonly BindableProperty PointsProperty =
        BindableProperty.Create(nameof(Points), typeof(List<float>), typeof(LineChartView),
            propertyChanged: (b, _, _) => ((LineChartView)b).InvalidateSurface());
    
    public List<float>? Points
    {
        get => (List<float>?)GetValue(PointsProperty);
        set => SetValue(PointsProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);
        
        if (Points == null || Points.Count < 2) return;
        
        var info = e.Info;
        var padding = 20f;
        var max = Points.Max();
        var min = Points.Min();
        var range = max - min;
        
        // Draw gradient fill
        var path = new SKPath();
        var step = (info.Width - padding * 2) / (Points.Count - 1);
        
        for (int i = 0; i < Points.Count; i++)
        {
            var x = padding + i * step;
            var normalized = range > 0 ? (Points[i] - min) / range : 0.5f;
            var y = padding + (1 - normalized) * (info.Height - padding * 2);
            
            if (i == 0) path.MoveTo(x, y);
            else path.LineTo(x, y);
        }
        
        // Fill
        var fillPath = path.Clone();
        fillPath.LineTo(padding + (Points.Count - 1) * step, info.Height);
        fillPath.LineTo(padding, info.Height);
        fillPath.Close();
        
        using var fillPaint = new SKPaint
        {
            Shader = SKShader.CreateLinearGradient(
                new SKPoint(0, 0), new SKPoint(0, info.Height),
                new[] { SKColor.Parse("#6750A4").WithAlpha(80), SKColors.Transparent },
                null, SKShaderTileMode.Clamp),
            Style = SKPaintStyle.Fill
        };
        canvas.DrawPath(fillPath, fillPaint);
        
        // Line
        using var linePaint = new SKPaint
        {
            Color = SKColor.Parse("#6750A4"),
            StrokeWidth = 2.5f,
            IsAntialias = true,
            Style = SKPaintStyle.Stroke
        };
        canvas.DrawPath(path, linePaint);
    }
}
```

---

## Step 549: Drag & Drop

```csharp
// ============================================
// Drag & Drop Reorderable List
// ============================================

public partial class DraggableListViewModel : ObservableObject
{
    [ObservableProperty]
    private ObservableCollection<DraggableItem> _items = new(
        Enumerable.Range(1, 5).Select(i => new DraggableItem(i, $"รายการที่ {i}")));
    
    private int _draggingIndex = -1;
    
    public void OnDragStarted(int index) => _draggingIndex = index;
    
    public void OnDropped(int targetIndex)
    {
        if (_draggingIndex < 0 || _draggingIndex == targetIndex) return;
        
        var item = Items[_draggingIndex];
        Items.RemoveAt(_draggingIndex);
        Items.Insert(targetIndex, item);
        
        _draggingIndex = -1;
        
        // Update order numbers
        for (int i = 0; i < Items.Count; i++)
            Items[i] = Items[i] with { SortOrder = i };
    }
}

public record DraggableItem(int Id, string Name, int SortOrder = 0);
```

---

## Step 550: Theming System

```csharp
// ============================================
// Dynamic Theming
// ============================================

public class ThemeService
{
    private readonly IPreferences _prefs;
    
    public ThemeService(IPreferences prefs) => _prefs = prefs;
    
    public AppTheme CurrentTheme => GetSavedTheme();
    
    public void ApplyTheme(AppTheme theme)
    {
        _prefs.Set("app_theme", theme.ToString());
        
        Application.Current!.UserAppTheme = theme switch
        {
            AppTheme.Light => Microsoft.Maui.ApplicationModel.AppTheme.Light,
            AppTheme.Dark => Microsoft.Maui.ApplicationModel.AppTheme.Dark,
            _ => Microsoft.Maui.ApplicationModel.AppTheme.Unspecified
        };
    }
    
    private AppTheme GetSavedTheme()
    {
        var saved = _prefs.Get("app_theme", "System");
        return Enum.TryParse<AppTheme>(saved, out var theme) ? theme : AppTheme.System;
    }
}

public enum AppTheme { System, Light, Dark }

// Custom dark theme ResourceDictionary
public class DarkTheme : ResourceDictionary
{
    public DarkTheme()
    {
        Add("Primary", Color.FromArgb("#D0BCFF"));
        Add("OnPrimary", Color.FromArgb("#381E72"));
        Add("Surface", Color.FromArgb("#1C1B1F"));
        Add("OnSurface", Color.FromArgb("#E6E1E5"));
        Add("SurfaceVariant", Color.FromArgb("#49454F"));
        Add("Background", Color.FromArgb("#1C1B1F"));
    }
}
```

---

## สรุป Part 55

ใน Part 55 เราได้เรียนรู้:

1. **Custom Handlers** - BorderlessEntry, VideoPlayer
2. **BindableProperty** - Validate, Coerce, Attached
3. **Design System** - Tokens, colors, spacing, elevation
4. **Component Library** - PrimaryButton, Card, Chip
5. **Skeleton Loading** - Shimmer animation
6. **Toast & Snackbar** - Overlay notifications
7. **Bottom Sheet** - Drag to dismiss
8. **Data Visualization** - Bar chart, Line chart, SkiaSharp
9. **Drag & Drop** - Reorderable list
10. **Theming System** - Dynamic dark/light theme

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 55 | Steps 541-550*
