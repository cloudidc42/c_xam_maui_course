# Part 78: Custom Controls & Platform Renderers
## Steps 771-780: Custom Views, Native Renderers, SkiaSharp, Platform-Specific

---

## Step 771: Custom Bindable Control

```csharp
// ============================================
// Custom Rating Bar Control
// ============================================

public class RatingBar : View
{
    public static readonly BindableProperty ValueProperty =
        BindableProperty.Create(nameof(Value), typeof(double), typeof(RatingBar), 0.0,
            BindingMode.TwoWay, validateValue: (_, v) => (double)v is >= 0 and <= 5);
    
    public static readonly BindableProperty MaxValueProperty =
        BindableProperty.Create(nameof(MaxValue), typeof(int), typeof(RatingBar), 5);
    
    public static readonly BindableProperty StarColorProperty =
        BindableProperty.Create(nameof(StarColor), typeof(Color), typeof(RatingBar), Colors.Gold);
    
    public static readonly BindableProperty EmptyColorProperty =
        BindableProperty.Create(nameof(EmptyColor), typeof(Color), typeof(RatingBar),
            Color.FromArgb("#E0E0E0"));
    
    public static readonly BindableProperty StarSizeProperty =
        BindableProperty.Create(nameof(StarSize), typeof(double), typeof(RatingBar), 28.0);
    
    public static readonly BindableProperty IsReadOnlyProperty =
        BindableProperty.Create(nameof(IsReadOnly), typeof(bool), typeof(RatingBar), false);
    
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public int MaxValue
    {
        get => (int)GetValue(MaxValueProperty);
        set => SetValue(MaxValueProperty, value);
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
    
    public double StarSize
    {
        get => (double)GetValue(StarSizeProperty);
        set => SetValue(StarSizeProperty, value);
    }
    
    public bool IsReadOnly
    {
        get => (bool)GetValue(IsReadOnlyProperty);
        set => SetValue(IsReadOnlyProperty, value);
    }
    
    public event EventHandler<double>? ValueChanged;
    
    protected override void OnPropertyChanged(string? propertyName = null)
    {
        base.OnPropertyChanged(propertyName);
        if (propertyName == ValueProperty.PropertyName)
            ValueChanged?.Invoke(this, Value);
    }
}
```

---

## Step 772: SkiaSharp Custom Drawing

```csharp
// ============================================
// SkiaSharp Canvas View for Custom Charts
// ============================================

// Install: SkiaSharp.Views.Maui.Controls

public class DonutChartView : ContentView
{
    private readonly SKCanvasView _canvas;
    
    public static readonly BindableProperty SegmentsProperty =
        BindableProperty.Create(nameof(Segments), typeof(List<ChartSegment>),
            typeof(DonutChartView), null,
            propertyChanged: (b, _, _) => ((DonutChartView)b)._canvas.InvalidateSurface());
    
    public List<ChartSegment>? Segments
    {
        get => (List<ChartSegment>?)GetValue(SegmentsProperty);
        set => SetValue(SegmentsProperty, value);
    }
    
    public DonutChartView()
    {
        _canvas = new SKCanvasView();
        _canvas.PaintSurface += OnPaint;
        Content = _canvas;
    }
    
    private void OnPaint(object? sender, SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.Transparent);
        
        if (Segments == null || Segments.Count == 0) return;
        
        var info = e.Info;
        var size = Math.Min(info.Width, info.Height);
        var cx = info.Width / 2f;
        var cy = info.Height / 2f;
        var outerR = size * 0.45f;
        var innerR = size * 0.25f;
        
        var total = Segments.Sum(s => s.Value);
        var startAngle = -90f;
        
        foreach (var seg in Segments)
        {
            var sweep = (float)(seg.Value / total * 360);
            
            using var path = new SKPath();
            var outerRect = new SKRect(cx - outerR, cy - outerR, cx + outerR, cy + outerR);
            var innerRect = new SKRect(cx - innerR, cy - innerR, cx + innerR, cy + innerR);
            
            path.ArcTo(outerRect, startAngle, sweep, true);
            path.ArcTo(innerRect, startAngle + sweep, -sweep, false);
            path.Close();
            
            using var paint = new SKPaint
            {
                Color = SKColor.Parse(seg.HexColor),
                IsAntialias = true,
                Style = SKPaintStyle.Fill
            };
            
            canvas.DrawPath(path, paint);
            startAngle += sweep;
        }
        
        // Center label
        using var textPaint = new SKPaint
        {
            Color = SKColors.Black,
            TextSize = size * 0.08f,
            IsAntialias = true,
            TextAlign = SKTextAlign.Center
        };
        canvas.DrawText("ยอดขาย", cx, cy - textPaint.TextSize / 2, textPaint);
    }
}

public record ChartSegment(string Label, double Value, string HexColor);
```

---

## Step 773: Bar Chart Control

```csharp
// ============================================
// SkiaSharp Bar Chart
// ============================================

public class BarChartView : ContentView
{
    private readonly SKCanvasView _canvas;
    
    public static readonly BindableProperty DataProperty =
        BindableProperty.Create(nameof(Data), typeof(List<(string Label, double Value)>),
            typeof(BarChartView),
            propertyChanged: (b, _, _) => ((BarChartView)b)._canvas.InvalidateSurface());
    
    public List<(string Label, double Value)>? Data
    {
        get => (List<(string Label, double Value)>?)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    public BarChartView()
    {
        _canvas = new SKCanvasView();
        _canvas.PaintSurface += OnPaint;
        Content = _canvas;
    }
    
    private void OnPaint(object? sender, SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);
        
        if (Data == null || Data.Count == 0) return;
        
        var info = e.Info;
        var w = info.Width;
        var h = info.Height;
        var padding = 40f;
        var maxValue = (float)Data.Max(d => d.Value);
        var barWidth = (w - padding * 2) / Data.Count - 8;
        
        using var barPaint = new SKPaint
        {
            Color = SKColor.Parse("#007AFF"),
            IsAntialias = true
        };
        
        using var textPaint = new SKPaint
        {
            Color = SKColors.Gray,
            TextSize = 24, IsAntialias = true,
            TextAlign = SKTextAlign.Center
        };
        
        for (int i = 0; i < Data.Count; i++)
        {
            var (label, value) = Data[i];
            var barH = (float)(value / maxValue * (h - padding * 2));
            var x = padding + i * (barWidth + 8);
            var y = h - padding - barH;
            
            // Bar
            var rect = new SKRoundRect(new SKRect(x, y, x + barWidth, h - padding), 4);
            canvas.DrawRoundRect(rect, barPaint);
            
            // Label
            canvas.DrawText(label, x + barWidth / 2, h - padding / 3, textPaint);
            
            // Value
            canvas.DrawText($"{value:F0}", x + barWidth / 2, y - 8, textPaint);
        }
    }
}
```

---

## Step 774: Platform Handler

```csharp
// ============================================
// MAUI Handler for Native Custom View
// ============================================

// Define the cross-platform interface
public class NativeVideoPlayer : View
{
    public static readonly BindableProperty SourceProperty =
        BindableProperty.Create(nameof(Source), typeof(string), typeof(NativeVideoPlayer));
    
    public static readonly BindableProperty IsPlayingProperty =
        BindableProperty.Create(nameof(IsPlaying), typeof(bool), typeof(NativeVideoPlayer),
            propertyChanged: (b, _, n) => ((NativeVideoPlayer)b).OnIsPlayingChanged((bool)n));
    
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
    
    private void OnIsPlayingChanged(bool playing) { /* Handler updates native view */ }
}

// MauiProgram.cs registration
// builder.ConfigureMauiHandlers(handlers =>
//     handlers.AddHandler<NativeVideoPlayer, NativeVideoPlayerHandler>());

#if ANDROID
public class NativeVideoPlayerHandler
    : ViewHandler<NativeVideoPlayer, Android.Widget.VideoView>
{
    public static PropertyMapper<NativeVideoPlayer, NativeVideoPlayerHandler> PropertyMapper = new(ViewMapper)
    {
        [nameof(NativeVideoPlayer.Source)] = MapSource,
        [nameof(NativeVideoPlayer.IsPlaying)] = MapIsPlaying,
    };
    
    public NativeVideoPlayerHandler() : base(PropertyMapper) { }
    
    protected override Android.Widget.VideoView CreatePlatformView()
    {
        var videoView = new Android.Widget.VideoView(Context);
        return videoView;
    }
    
    private static void MapSource(NativeVideoPlayerHandler handler, NativeVideoPlayer view)
    {
        if (view.Source != null)
            handler.PlatformView.SetVideoPath(view.Source);
    }
    
    private static void MapIsPlaying(NativeVideoPlayerHandler handler, NativeVideoPlayer view)
    {
        if (view.IsPlaying) handler.PlatformView.Start();
        else handler.PlatformView.Pause();
    }
}
#endif
```

---

## Step 775: Gradient Controls

```csharp
// ============================================
// Gradient Frame with Border
// ============================================

public class GradientCard : ContentView
{
    public static readonly BindableProperty GradientStartProperty =
        BindableProperty.Create(nameof(GradientStart), typeof(Color), typeof(GradientCard),
            Color.FromArgb("#007AFF"),
            propertyChanged: (b, _, _) => ((GradientCard)b).UpdateGradient());
    
    public static readonly BindableProperty GradientEndProperty =
        BindableProperty.Create(nameof(GradientEnd), typeof(Color), typeof(GradientCard),
            Color.FromArgb("#5AC8FA"),
            propertyChanged: (b, _, _) => ((GradientCard)b).UpdateGradient());
    
    public static readonly BindableProperty CornerRadiusProperty =
        BindableProperty.Create(nameof(CornerRadius), typeof(double), typeof(GradientCard), 16.0);
    
    public Color GradientStart
    {
        get => (Color)GetValue(GradientStartProperty);
        set => SetValue(GradientStartProperty, value);
    }
    
    public Color GradientEnd
    {
        get => (Color)GetValue(GradientEndProperty);
        set => SetValue(GradientEndProperty, value);
    }
    
    public double CornerRadius
    {
        get => (double)GetValue(CornerRadiusProperty);
        set => SetValue(CornerRadiusProperty, value);
    }
    
    private readonly Frame _frame;
    
    public GradientCard()
    {
        _frame = new Frame
        {
            HasShadow = true, Padding = 16, ClipsToBounds = true
        };
        
        base.Content = _frame;
        UpdateGradient();
    }
    
    public new View? Content
    {
        get => _frame.Content;
        set => _frame.Content = value;
    }
    
    private void UpdateGradient()
    {
        _frame.Background = new LinearGradientBrush(
            new GradientStopCollection
            {
                new GradientStop(GradientStart, 0),
                new GradientStop(GradientEnd, 1)
            },
            new Point(0, 0),
            new Point(1, 1));
        
        _frame.CornerRadius = (float)CornerRadius;
    }
}
```

---

## Step 776: Touch-Driven Slider

```csharp
// ============================================
// Custom Touch Slider
// ============================================

public class PriceRangeSlider : ContentView
{
    public static readonly BindableProperty MinValueProperty =
        BindableProperty.Create(nameof(MinValue), typeof(double), typeof(PriceRangeSlider), 0.0,
            BindingMode.TwoWay);
    
    public static readonly BindableProperty MaxValueProperty =
        BindableProperty.Create(nameof(MaxValue), typeof(double), typeof(PriceRangeSlider), 1000.0,
            BindingMode.TwoWay);
    
    public static readonly BindableProperty LowerValueProperty =
        BindableProperty.Create(nameof(LowerValue), typeof(double), typeof(PriceRangeSlider), 0.0,
            BindingMode.TwoWay, propertyChanged: (b, _, _) => ((PriceRangeSlider)b).UpdateThumbs());
    
    public static readonly BindableProperty UpperValueProperty =
        BindableProperty.Create(nameof(UpperValue), typeof(double), typeof(PriceRangeSlider), 1000.0,
            BindingMode.TwoWay, propertyChanged: (b, _, _) => ((PriceRangeSlider)b).UpdateThumbs());
    
    public double MinValue { get => (double)GetValue(MinValueProperty); set => SetValue(MinValueProperty, value); }
    public double MaxValue { get => (double)GetValue(MaxValueProperty); set => SetValue(MaxValueProperty, value); }
    public double LowerValue { get => (double)GetValue(LowerValueProperty); set => SetValue(LowerValueProperty, value); }
    public double UpperValue { get => (double)GetValue(UpperValueProperty); set => SetValue(UpperValueProperty, value); }
    
    private readonly BoxView _track;
    private readonly BoxView _activeTrack;
    private readonly BoxView _lowerThumb;
    private readonly BoxView _upperThumb;
    private readonly AbsoluteLayout _layout;
    
    public PriceRangeSlider()
    {
        _track = new BoxView { BackgroundColor = Color.FromArgb("#E5E5EA"), CornerRadius = 2, HeightRequest = 4 };
        _activeTrack = new BoxView { BackgroundColor = Color.FromArgb("#007AFF"), CornerRadius = 2, HeightRequest = 4 };
        _lowerThumb = CreateThumb(true);
        _upperThumb = CreateThumb(false);
        
        _layout = new AbsoluteLayout { HeightRequest = 40 };
        AbsoluteLayout.SetLayoutFlags(_track, AbsoluteLayoutFlags.WidthProportional);
        AbsoluteLayout.SetLayoutBounds(_track, new Rect(0, 18, 1, 4));
        
        _layout.Children.Add(_track);
        _layout.Children.Add(_activeTrack);
        _layout.Children.Add(_lowerThumb);
        _layout.Children.Add(_upperThumb);
        Content = _layout;
    }
    
    private BoxView CreateThumb(bool isLower)
    {
        var thumb = new BoxView
        {
            BackgroundColor = Colors.White,
            CornerRadius = 12,
            WidthRequest = 24, HeightRequest = 24,
            Shadow = new Shadow { Blur = 4, Opacity = 0.3f }
        };
        
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += (s, e) => OnThumbPan(e, isLower);
        thumb.GestureRecognizers.Add(pan);
        
        return thumb;
    }
    
    private void OnThumbPan(PanUpdatedEventArgs e, bool isLower)
    {
        if (e.StatusType != GestureStatus.Running) return;
        
        var range = MaxValue - MinValue;
        var trackWidth = _layout.Width;
        var delta = e.TotalX / trackWidth * range;
        
        if (isLower)
            LowerValue = Math.Clamp(LowerValue + delta, MinValue, UpperValue - 1);
        else
            UpperValue = Math.Clamp(UpperValue + delta, LowerValue + 1, MaxValue);
    }
    
    private void UpdateThumbs()
    {
        if (_layout.Width <= 0) return;
        
        var w = _layout.Width;
        var range = MaxValue - MinValue;
        var lx = (LowerValue - MinValue) / range * w;
        var ux = (UpperValue - MinValue) / range * w;
        
        AbsoluteLayout.SetLayoutBounds(_lowerThumb, new Rect(lx - 12, 8, 24, 24));
        AbsoluteLayout.SetLayoutBounds(_upperThumb, new Rect(ux - 12, 8, 24, 24));
        AbsoluteLayout.SetLayoutBounds(_activeTrack, new Rect(lx, 18, ux - lx, 4));
    }
}
```

---

## Step 777: Image Cropper Control

```csharp
// ============================================
// Interactive Image Cropper
// ============================================

public class ImageCropperView : ContentView
{
    private readonly Image _image;
    private readonly BoxView _overlay;
    private Rect _cropRect;
    
    public event EventHandler<Rect>? CropRectChanged;
    
    public ImageCropperView()
    {
        _image = new Image { Aspect = Aspect.AspectFill };
        _overlay = new BoxView { BackgroundColor = Color.FromArgb("#80000000") };
        
        var layout = new AbsoluteLayout { BackgroundColor = Colors.Black };
        AbsoluteLayout.SetLayoutFlags(_image, AbsoluteLayoutFlags.All);
        AbsoluteLayout.SetLayoutBounds(_image, new Rect(0, 0, 1, 1));
        
        layout.Children.Add(_image);
        layout.Children.Add(_overlay);
        Content = layout;
        
        // Pinch-to-zoom for crop area
        var pinch = new PinchGestureRecognizer();
        pinch.PinchUpdated += OnPinch;
        GestureRecognizers.Add(pinch);
        
        // Pan to move crop area
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += OnPan;
        GestureRecognizers.Add(pan);
    }
    
    public ImageSource? Source
    {
        get => _image.Source;
        set => _image.Source = value;
    }
    
    private void OnPinch(object? sender, PinchGestureUpdatedEventArgs e)
    {
        if (e.Status == GestureStatus.Running)
        {
            var scale = e.Scale;
            _cropRect = new Rect(
                _cropRect.X, _cropRect.Y,
                _cropRect.Width * scale, _cropRect.Height * scale);
            UpdateOverlay();
        }
    }
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        if (e.StatusType == GestureStatus.Running)
        {
            _cropRect = new Rect(
                _cropRect.X + e.TotalX, _cropRect.Y + e.TotalY,
                _cropRect.Width, _cropRect.Height);
            UpdateOverlay();
        }
    }
    
    private void UpdateOverlay()
    {
        CropRectChanged?.Invoke(this, _cropRect);
    }
}
```

---

## Step 778: Signature Pad

```csharp
// ============================================
// Digital Signature Pad using SkiaSharp
// ============================================

public class SignaturePadView : SKCanvasView
{
    private readonly List<List<SKPoint>> _strokes = new();
    private List<SKPoint>? _currentStroke;
    private bool _hasSignature;
    
    public bool HasSignature => _hasSignature;
    
    public SignaturePadView()
    {
        BackgroundColor = Colors.White;
        EnableTouchEvents = true;
        Touch += OnTouch;
    }
    
    private void OnTouch(object? sender, SKTouchEventArgs e)
    {
        switch (e.ActionType)
        {
            case SKTouchAction.Pressed:
                _currentStroke = new List<SKPoint> { e.Location };
                _strokes.Add(_currentStroke);
                break;
            
            case SKTouchAction.Moved:
                _currentStroke?.Add(e.Location);
                _hasSignature = true;
                InvalidateSurface();
                break;
            
            case SKTouchAction.Released:
                _currentStroke = null;
                break;
        }
        
        e.Handled = true;
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        base.OnPaintSurface(e);
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);
        
        using var paint = new SKPaint
        {
            Color = SKColors.Black,
            StrokeWidth = 3,
            IsAntialias = true,
            Style = SKPaintStyle.Stroke,
            StrokeCap = SKStrokeCap.Round,
            StrokeJoin = SKStrokeJoin.Round
        };
        
        foreach (var stroke in _strokes)
        {
            if (stroke.Count < 2) continue;
            
            using var path = new SKPath();
            path.MoveTo(stroke[0]);
            for (int i = 1; i < stroke.Count; i++)
                path.LineTo(stroke[i]);
            
            canvas.DrawPath(path, paint);
        }
    }
    
    public void Clear()
    {
        _strokes.Clear();
        _hasSignature = false;
        InvalidateSurface();
    }
    
    public byte[] ExportAsPng()
    {
        var info = new SKImageInfo(300, 150);
        using var surface = SKSurface.Create(info);
        var canvas = surface.Canvas;
        canvas.Clear(SKColors.White);
        
        // Redraw strokes
        using var paint = new SKPaint
        {
            Color = SKColors.Black, StrokeWidth = 2,
            IsAntialias = true, Style = SKPaintStyle.Stroke
        };
        
        foreach (var stroke in _strokes)
        {
            using var path = new SKPath();
            if (stroke.Count < 2) continue;
            path.MoveTo(stroke[0]);
            for (int i = 1; i < stroke.Count; i++) path.LineTo(stroke[i]);
            canvas.DrawPath(path, paint);
        }
        
        using var image = surface.Snapshot();
        using var data = image.Encode(SKEncodedImageFormat.Png, 100);
        return data.ToArray();
    }
}
```

---

## Step 779: OTP Input Control

```csharp
// ============================================
// OTP (One-Time Password) Input
// ============================================

public class OtpInputView : ContentView
{
    private readonly List<Entry> _digits = new();
    
    public static readonly BindableProperty LengthProperty =
        BindableProperty.Create(nameof(Length), typeof(int), typeof(OtpInputView), 6,
            propertyChanged: (b, _, _) => ((OtpInputView)b).BuildLayout());
    
    public static readonly BindableProperty CodeProperty =
        BindableProperty.Create(nameof(Code), typeof(string), typeof(OtpInputView), "",
            BindingMode.TwoWay);
    
    public int Length { get => (int)GetValue(LengthProperty); set => SetValue(LengthProperty, value); }
    public string Code { get => (string)GetValue(CodeProperty); set => SetValue(CodeProperty, value); }
    
    public event EventHandler<string>? CodeCompleted;
    
    public OtpInputView() => BuildLayout();
    
    private void BuildLayout()
    {
        _digits.Clear();
        
        var stack = new HorizontalStackLayout { Spacing = 8 };
        
        for (int i = 0; i < Length; i++)
        {
            var idx = i;
            var entry = new Entry
            {
                WidthRequest = 48, HeightRequest = 56,
                MaxLength = 1, Keyboard = Keyboard.Numeric,
                HorizontalTextAlignment = TextAlignment.Center,
                FontSize = 20, FontAttributes = FontAttributes.Bold
            };
            
            entry.TextChanged += (s, e) =>
            {
                if (e.NewTextValue.Length == 1 && idx < Length - 1)
                    _digits[idx + 1].Focus();
                
                UpdateCode();
            };
            
            _digits.Add(entry);
            stack.Children.Add(entry);
        }
        
        Content = stack;
    }
    
    private void UpdateCode()
    {
        Code = string.Join("", _digits.Select(d => d.Text ?? ""));
        if (Code.Length == Length)
            CodeCompleted?.Invoke(this, Code);
    }
    
    public void Clear()
    {
        foreach (var d in _digits) d.Text = "";
        _digits.FirstOrDefault()?.Focus();
    }
}
```

---

## Step 780: Phone Number Input

```csharp
// ============================================
// Smart Phone Number Input
// ============================================

public class PhoneNumberEntry : Entry
{
    private bool _isFormatting;
    
    public PhoneNumberEntry()
    {
        Keyboard = Keyboard.Telephone;
        MaxLength = 12; // "089-123-4567"
        Placeholder = "089-123-4567";
        
        TextChanged += OnTextChanged;
    }
    
    private void OnTextChanged(object? sender, TextChangedEventArgs e)
    {
        if (_isFormatting) return;
        
        var digits = new string(e.NewTextValue.Where(char.IsDigit).ToArray());
        
        string formatted = digits.Length switch
        {
            0 => "",
            <= 3 => digits,
            <= 6 => $"{digits[..3]}-{digits[3..]}",
            _ => $"{digits[..3]}-{digits[3..6]}-{digits[6..Math.Min(10, digits.Length)]}"
        };
        
        if (formatted != e.NewTextValue)
        {
            _isFormatting = true;
            Text = formatted;
            CursorPosition = formatted.Length;
            _isFormatting = false;
        }
    }
    
    public string PhoneDigits
        => new string((Text ?? "").Where(char.IsDigit).ToArray());
    
    public bool IsValid
        => PhoneDigits.Length == 10 && (PhoneDigits.StartsWith("06") ||
            PhoneDigits.StartsWith("07") || PhoneDigits.StartsWith("08") ||
            PhoneDigits.StartsWith("09") || PhoneDigits.StartsWith("02"));
}
```

---

## สรุป Part 78

ใน Part 78 เราได้เรียนรู้:

1. **Custom Bindable Control** - BindableProperty, validation, events
2. **SkiaSharp Donut Chart** - SKPath arc segments, gradient fill
3. **Bar Chart** - SKCanvas text/shape drawing, value labels
4. **Platform Handler** - ViewHandler, PropertyMapper, Android VideoView
5. **Gradient Card** - LinearGradientBrush with runtime update
6. **Price Range Slider** - Dual thumbs, pan gesture, track highlighting
7. **Image Cropper** - PinchGesture + PanGesture, crop rectangle
8. **Signature Pad** - SKTouchEvent strokes, PNG export
9. **OTP Input** - Auto-focus next digit, CodeCompleted event
10. **Phone Number Entry** - Auto-format as-you-type, Thai mobile validation

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 78 | Steps 771-780*
