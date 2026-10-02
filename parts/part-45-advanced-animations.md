# Part 45: Advanced Animations & Transitions
## Steps 441-450: Complex Animations, Shared Element, Lottie

---

## Step 441: Animation Fundamentals

```csharp
// ============================================
// Animation Basics & Easing
// ============================================

using Microsoft.Maui.Controls;

public class AnimationBasics
{
    // Basic view animations
    public async Task DemoBasicAnimations(View view)
    {
        // Fade
        await view.FadeTo(0, 500, Easing.CubicOut);
        await view.FadeTo(1, 500, Easing.CubicIn);
        
        // Scale
        await view.ScaleTo(1.5, 300, Easing.SpringOut);
        await view.ScaleTo(1.0, 300, Easing.SpringIn);
        
        // Translate
        await view.TranslateTo(100, 0, 400, Easing.CubicInOut);
        await view.TranslateTo(0, 0, 400, Easing.CubicInOut);
        
        // Rotate
        await view.RotateTo(360, 600, Easing.Linear);
        view.Rotation = 0;
        
        // Combined
        await Task.WhenAll(
            view.FadeTo(0.5, 500),
            view.ScaleTo(0.8, 500),
            view.TranslateTo(50, -50, 500)
        );
        
        // Reset
        await Task.WhenAll(
            view.FadeTo(1, 500),
            view.ScaleTo(1, 500),
            view.TranslateTo(0, 0, 500)
        );
    }
    
    // Relative translate (from current position)
    public async Task SlideIn(View view, bool fromLeft = true)
    {
        var startX = fromLeft ? -view.Width : view.Width;
        view.TranslationX = startX;
        await view.TranslateTo(0, 0, 400, Easing.CubicOut);
    }
    
    public async Task SlideOut(View view, bool toRight = true)
    {
        var endX = toRight ? view.Width : -view.Width;
        await view.TranslateTo(endX, 0, 400, Easing.CubicIn);
    }
}
```

---

## Step 442: Custom Easing Functions

```csharp
// ============================================
// Custom Easing & Keyframe Animations
// ============================================

public static class CustomEasings
{
    // Bounce easing
    public static Easing Bounce = new Easing(t =>
    {
        if (t < 1 / 2.75) return 7.5625 * t * t;
        if (t < 2 / 2.75) { t -= 1.5 / 2.75; return 7.5625 * t * t + 0.75; }
        if (t < 2.5 / 2.75) { t -= 2.25 / 2.75; return 7.5625 * t * t + 0.9375; }
        t -= 2.625 / 2.75;
        return 7.5625 * t * t + 0.984375;
    });
    
    // Elastic easing
    public static Easing Elastic = new Easing(t =>
    {
        if (t == 0 || t == 1) return t;
        var p = 0.3;
        var s = p / 4;
        t -= 1;
        return -Math.Pow(2, 10 * t) * Math.Sin((t - s) * (2 * Math.PI) / p);
    });
    
    // Back easing (overshoots)
    public static Easing BackOut = new Easing(t =>
    {
        var s = 1.70158;
        t -= 1;
        return t * t * ((s + 1) * t + s) + 1;
    });
    
    // Smooth step
    public static Easing SmoothStep = new Easing(t =>
        t * t * (3 - 2 * t));
    
    // Smoother step
    public static Easing SmootherStep = new Easing(t =>
        t * t * t * (t * (t * 6 - 15) + 10));
}

// Keyframe animation
public class KeyframeAnimation
{
    private record Keyframe(double Progress, Dictionary<string, double> Values);
    
    private readonly List<Keyframe> _keyframes = new();
    private readonly View _target;
    
    public KeyframeAnimation(View target) => _target = target;
    
    public KeyframeAnimation At(double progress, Action<KeyframeBuilder> configure)
    {
        var builder = new KeyframeBuilder();
        configure(builder);
        _keyframes.Add(new Keyframe(progress, builder.Values));
        return this;
    }
    
    public async Task PlayAsync(uint duration, Easing? easing = null)
    {
        _keyframes.Sort((a, b) => a.Progress.CompareTo(b.Progress));
        
        await _target.AnimateAsync("keyframe", t =>
        {
            // Find surrounding keyframes
            var prev = _keyframes.LastOrDefault(k => k.Progress <= t)
                ?? _keyframes.First();
            var next = _keyframes.FirstOrDefault(k => k.Progress > t)
                ?? _keyframes.Last();
            
            if (prev == next)
            {
                ApplyValues(prev.Values, 1.0);
                return;
            }
            
            var localT = (t - prev.Progress) / (next.Progress - prev.Progress);
            
            // Interpolate all properties
            foreach (var (prop, endValue) in next.Values)
            {
                var startValue = prev.Values.GetValueOrDefault(prop, 0);
                var current = startValue + (endValue - startValue) * localT;
                ApplyProperty(prop, current);
            }
        }, length: duration, easing: easing);
    }
    
    private void ApplyValues(Dictionary<string, double> values, double t)
    {
        foreach (var (prop, value) in values)
            ApplyProperty(prop, value);
    }
    
    private void ApplyProperty(string prop, double value)
    {
        switch (prop)
        {
            case "opacity": _target.Opacity = value; break;
            case "scale": _target.Scale = value; break;
            case "scaleX": _target.ScaleX = value; break;
            case "scaleY": _target.ScaleY = value; break;
            case "translateX": _target.TranslationX = value; break;
            case "translateY": _target.TranslationY = value; break;
            case "rotation": _target.Rotation = value; break;
        }
    }
}

public class KeyframeBuilder
{
    public Dictionary<string, double> Values { get; } = new();
    
    public KeyframeBuilder Opacity(double v) { Values["opacity"] = v; return this; }
    public KeyframeBuilder Scale(double v) { Values["scale"] = v; return this; }
    public KeyframeBuilder TranslateX(double v) { Values["translateX"] = v; return this; }
    public KeyframeBuilder TranslateY(double v) { Values["translateY"] = v; return this; }
    public KeyframeBuilder Rotation(double v) { Values["rotation"] = v; return this; }
}
```

---

## Step 443: Page Transitions

```csharp
// ============================================
// Custom Page Transitions
// ============================================

public class SlideTransition : INavigationTransition
{
    private readonly SlideDirection _direction;
    
    public SlideTransition(SlideDirection direction = SlideDirection.Left)
        => _direction = direction;
    
    public async Task PlayAsync(Page appearing, Page disappearing, bool isPush)
    {
        var width = appearing.Width;
        
        // Setup initial positions
        var appearingStartX = _direction switch
        {
            SlideDirection.Left => width,
            SlideDirection.Right => -width,
            _ => 0
        };
        
        appearing.TranslationX = appearingStartX;
        
        var disappearingEndX = _direction switch
        {
            SlideDirection.Left => -width * 0.3,
            SlideDirection.Right => width * 0.3,
            _ => 0
        };
        
        // Animate both pages
        await Task.WhenAll(
            appearing.TranslateTo(0, 0, 350, Easing.CubicOut),
            disappearing.TranslateTo(disappearingEndX, 0, 350, Easing.CubicOut),
            disappearing.FadeTo(0.5, 350)
        );
    }
    
    public async Task ResetAsync(Page page)
    {
        page.TranslationX = 0;
        page.Opacity = 1;
        await Task.CompletedTask;
    }
}

public enum SlideDirection { Left, Right, Up, Down }

// Card flip transition
public class FlipTransition
{
    public async Task FlipAsync(View front, View back, bool showBack)
    {
        await front.RotateYTo(90, 200, Easing.CubicIn);
        
        front.IsVisible = !showBack;
        back.IsVisible = showBack;
        back.RotationY = -90;
        
        await back.RotateYTo(0, 200, Easing.CubicOut);
    }
}
```

---

## Step 444: Lottie Animations

```csharp
// ============================================
// Lottie Animations
// ============================================

// NuGet: SkiaSharp.Extended.UI.Maui (for Lottie)

/*
XAML:
<skia:SKLottieView x:Name="LottieView"
                   Source="Embedded://loading.json"
                   IsAnimating="{Binding IsLoading}"
                   RepeatCount="-1"
                   RepeatMode="Reverse"
                   HeightRequest="200"
                   WidthRequest="200" />
*/

public class LottieAnimationService
{
    // Play once and stop
    public static async Task PlayOnceAsync(SKLottieView lottie)
    {
        lottie.RepeatCount = 0;
        lottie.IsAnimating = true;
        
        // Wait for animation to complete
        var tcs = new TaskCompletionSource<bool>();
        lottie.AnimationCompleted += (_, _) => tcs.SetResult(true);
        await tcs.Task;
    }
    
    // Play success animation
    public static async Task ShowSuccessAsync(SKLottieView lottie)
    {
        lottie.Source = SKLottieImageSource.FromFile("Embedded://success.json");
        await PlayOnceAsync(lottie);
    }
    
    // Play error animation
    public static async Task ShowErrorAsync(SKLottieView lottie)
    {
        lottie.Source = SKLottieImageSource.FromFile("Embedded://error.json");
        await PlayOnceAsync(lottie);
    }
}

// Empty state with Lottie
public class EmptyStateView : ContentView
{
    private readonly SKLottieView _lottie;
    private readonly Label _messageLabel;
    private readonly Button _actionButton;
    
    public static readonly BindableProperty AnimationSourceProperty =
        BindableProperty.Create(nameof(AnimationSource), typeof(string),
            typeof(EmptyStateView), "Embedded://empty.json");
    
    public static readonly BindableProperty MessageProperty =
        BindableProperty.Create(nameof(Message), typeof(string),
            typeof(EmptyStateView), "ไม่มีข้อมูล");
    
    public string AnimationSource
    {
        get => (string)GetValue(AnimationSourceProperty);
        set => SetValue(AnimationSourceProperty, value);
    }
    
    public string Message
    {
        get => (string)GetValue(MessageProperty);
        set => SetValue(MessageProperty, value);
    }
    
    public EmptyStateView()
    {
        _lottie = new SKLottieView
        {
            HeightRequest = 200,
            WidthRequest = 200,
            RepeatCount = -1
        };
        
        _messageLabel = new Label
        {
            HorizontalTextAlignment = TextAlignment.Center,
            TextColor = Colors.Gray,
            FontSize = 16
        };
        
        _actionButton = new Button { IsVisible = false };
        
        Content = new VerticalStackLayout
        {
            HorizontalOptions = LayoutOptions.Center,
            VerticalOptions = LayoutOptions.Center,
            Spacing = 16,
            Children = { _lottie, _messageLabel, _actionButton }
        };
    }
    
    protected override void OnPropertyChanged(string? propertyName = null)
    {
        base.OnPropertyChanged(propertyName);
        
        if (propertyName == AnimationSourceProperty.PropertyName)
            _lottie.Source = SKLottieImageSource.FromFile(AnimationSource);
        
        if (propertyName == MessageProperty.PropertyName)
            _messageLabel.Text = Message;
    }
}
```

---

## Step 445: Shared Element Transition

```csharp
// ============================================
// Shared Element Transition (Hero Animation)
// ============================================

public class HeroTransitionService
{
    // Register a view as a hero element
    private static readonly Dictionary<string, WeakReference<View>> _heroes = new();
    
    public static void RegisterHero(string heroId, View view)
    {
        _heroes[heroId] = new WeakReference<View>(view);
    }
    
    public static View? GetHero(string heroId)
    {
        if (_heroes.TryGetValue(heroId, out var weakRef) &&
            weakRef.TryGetTarget(out var view))
            return view;
        return null;
    }
    
    // Animate from source position to destination
    public static async Task AnimateHeroAsync(
        string heroId, View destination, Page destinationPage)
    {
        var source = GetHero(heroId);
        if (source == null) return;
        
        // Get absolute positions
        var sourceBounds = GetAbsoluteBounds(source);
        var destBounds = GetAbsoluteBounds(destination);
        
        // Create overlay clone
        var overlay = new BoxView
        {
            BackgroundColor = Colors.Transparent,
            X = sourceBounds.X,
            Y = sourceBounds.Y,
            Width = sourceBounds.Width,
            Height = sourceBounds.Height
        };
        
        // Add to page overlay
        if (Application.Current?.Windows.FirstOrDefault()?.Page is ContentPage page)
        {
            var grid = new Grid();
            grid.Add(destinationPage.Content);
            grid.Add(overlay);
            page.Content = grid;
        }
        
        // Animate overlay to destination
        var tasks = new List<Task>
        {
            overlay.LayoutTo(destBounds, 350, Easing.CubicInOut),
        };
        
        if (sourceBounds.Width != destBounds.Width)
        {
            // Scale animation placeholder
        }
        
        await Task.WhenAll(tasks);
    }
    
    private static Rect GetAbsoluteBounds(View view)
    {
        // Get position relative to window
        // Note: full implementation requires platform-specific code
        return new Rect(view.X, view.Y, view.Width, view.Height);
    }
}

// Product card with hero transition
public class ProductCard : ContentView
{
    private readonly string _heroId;
    private readonly Image _productImage;
    
    public ProductCard(Product product)
    {
        _heroId = $"product-image-{product.Id}";
        
        _productImage = new Image
        {
            Source = product.ImageUrl,
            HeightRequest = 200
        };
        
        // Register as hero
        Loaded += (_, _) => HeroTransitionService.RegisterHero(_heroId, _productImage);
        
        var tap = new TapGestureRecognizer();
        tap.Tapped += async (_, _) =>
        {
            await NavigateWithHeroAsync(product);
        };
        GestureRecognizers.Add(tap);
        
        Content = new VerticalStackLayout
        {
            Children = { _productImage, new Label { Text = product.Name } }
        };
    }
    
    private async Task NavigateWithHeroAsync(Product product)
    {
        // Navigate first
        await Shell.Current.GoToAsync($"product-detail?id={product.Id}&heroId={_heroId}");
    }
}
```

---

## Step 446: Particle System

```csharp
// ============================================
// Particle System with SkiaSharp
// ============================================

public class ParticleView : SKCanvasView
{
    private readonly List<Particle> _particles = new();
    private System.Diagnostics.Stopwatch _sw = new();
    private bool _isRunning;
    
    public void Start()
    {
        _isRunning = true;
        _sw.Start();
        Device.StartTimer(TimeSpan.FromMilliseconds(16), () =>
        {
            Update();
            InvalidateSurface();
            return _isRunning;
        });
    }
    
    public void Emit(PointF origin, int count = 20, EmitConfig? config = null)
    {
        var cfg = config ?? new EmitConfig();
        var rng = new Random();
        
        for (int i = 0; i < count; i++)
        {
            _particles.Add(new Particle
            {
                X = origin.X,
                Y = origin.Y,
                VelocityX = (float)(rng.NextDouble() * 2 - 1) * cfg.Speed,
                VelocityY = -(float)rng.NextDouble() * cfg.Speed,
                Size = (float)(rng.NextDouble() * 6 + 2),
                Life = 1.0f,
                Decay = (float)(rng.NextDouble() * 0.02 + 0.01),
                Color = cfg.Colors[rng.Next(cfg.Colors.Count)]
            });
        }
    }
    
    private void Update()
    {
        var gravity = 0.2f;
        
        for (int i = _particles.Count - 1; i >= 0; i--)
        {
            var p = _particles[i];
            p.X += p.VelocityX;
            p.Y += p.VelocityY;
            p.VelocityY += gravity;
            p.Life -= p.Decay;
            
            if (p.Life <= 0)
                _particles.RemoveAt(i);
        }
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear();
        
        using var paint = new SKPaint { IsAntialias = true };
        
        foreach (var p in _particles)
        {
            paint.Color = p.Color.WithAlpha((byte)(p.Life * 255));
            canvas.DrawCircle(p.X, p.Y, p.Size * p.Life, paint);
        }
    }
    
    public void Stop() => _isRunning = false;
}

public class Particle
{
    public float X, Y;
    public float VelocityX, VelocityY;
    public float Size;
    public float Life;
    public float Decay;
    public SKColor Color;
}

public class EmitConfig
{
    public float Speed { get; set; } = 5f;
    public List<SKColor> Colors { get; set; } = new()
    {
        SKColors.Gold, SKColors.Orange, SKColors.Red, SKColors.Yellow
    };
}
```

---

## Step 447: Ripple Effect

```csharp
// ============================================
// Ripple / Wave Effect
// ============================================

public class RippleView : SKCanvasView
{
    private readonly List<RippleCircle> _ripples = new();
    private bool _animating;
    
    public Color RippleColor { get; set; } = Colors.Blue;
    public float MaxRadius { get; set; } = 100;
    public uint Duration { get; set; } = 600;
    
    public void Ripple(float x, float y)
    {
        _ripples.Add(new RippleCircle { X = x, Y = y, Progress = 0 });
        
        if (!_animating) StartAnimation();
    }
    
    private void StartAnimation()
    {
        _animating = true;
        this.AnimateAsync("ripple", t =>
        {
            foreach (var r in _ripples)
                r.Progress = (float)t;
            
            _ripples.RemoveAll(r => r.Progress >= 1.0f);
            InvalidateSurface();
        }, length: Duration, repeat: () => _ripples.Count > 0)
        .ContinueWith(_ => _animating = false);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear();
        
        using var paint = new SKPaint { IsAntialias = true, Style = SKPaintStyle.Stroke, StrokeWidth = 2 };
        
        foreach (var r in _ripples)
        {
            var alpha = (byte)((1 - r.Progress) * 200);
            paint.Color = new SKColor(
                (byte)(RippleColor.Red * 255),
                (byte)(RippleColor.Green * 255),
                (byte)(RippleColor.Blue * 255),
                alpha);
            
            canvas.DrawCircle(r.X, r.Y, r.Progress * MaxRadius, paint);
        }
    }
}

public class RippleCircle
{
    public float X, Y, Progress;
}
```

---

## Step 448: Number Counter Animation

```csharp
// ============================================
// Animated Number Counter
// ============================================

public class AnimatedCounterView : ContentView
{
    private readonly Label _label;
    private double _currentValue;
    
    public static readonly BindableProperty TargetValueProperty =
        BindableProperty.Create(nameof(TargetValue), typeof(double),
            typeof(AnimatedCounterView), 0.0,
            propertyChanged: OnTargetValueChanged);
    
    public static readonly BindableProperty FormatProperty =
        BindableProperty.Create(nameof(Format), typeof(string),
            typeof(AnimatedCounterView), "N0");
    
    public double TargetValue
    {
        get => (double)GetValue(TargetValueProperty);
        set => SetValue(TargetValueProperty, value);
    }
    
    public string Format
    {
        get => (string)GetValue(FormatProperty);
        set => SetValue(FormatProperty, value);
    }
    
    public AnimatedCounterView()
    {
        _label = new Label
        {
            HorizontalOptions = LayoutOptions.Center,
            VerticalOptions = LayoutOptions.Center
        };
        Content = _label;
    }
    
    private static void OnTargetValueChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var counter = (AnimatedCounterView)bindable;
        counter.AnimateToValue((double)newValue);
    }
    
    private void AnimateToValue(double target)
    {
        var start = _currentValue;
        
        this.AnimateAsync("counter", t =>
        {
            _currentValue = start + (target - start) * t;
            _label.Text = _currentValue.ToString(Format);
        }, length: 800, easing: Easing.CubicOut)
        .ContinueWith(_ => _currentValue = target);
    }
}
```

---

## Step 449: Staggered List Animation

```csharp
// ============================================
// Staggered Animation for Lists
// ============================================

public class StaggeredAnimationHelper
{
    // Animate list items with stagger delay
    public static async Task AnimateListItemsAsync(
        IEnumerable<View> items,
        uint itemDuration = 300,
        uint staggerDelay = 50)
    {
        var itemList = items.ToList();
        var tasks = new List<Task>();
        
        for (int i = 0; i < itemList.Count; i++)
        {
            var item = itemList[i];
            var delay = (uint)(i * staggerDelay);
            
            // Set initial state
            item.Opacity = 0;
            item.TranslationY = 30;
            
            // Schedule animation
            tasks.Add(AnimateItemAsync(item, delay, itemDuration));
        }
        
        await Task.WhenAll(tasks);
    }
    
    private static async Task AnimateItemAsync(View item, uint delay, uint duration)
    {
        await Task.Delay((int)delay);
        
        await Task.WhenAll(
            item.FadeTo(1, duration, Easing.CubicOut),
            item.TranslateTo(0, 0, duration, Easing.CubicOut)
        );
    }
    
    // CollectionView with animated items
    public static void SetupCollectionViewAnimation(CollectionView cv)
    {
        cv.Scrolled += (s, e) =>
        {
            // Animate newly visible items
        };
    }
}

// Custom animated CollectionView
public partial class AnimatedCollectionPage : ContentPage
{
    public AnimatedCollectionPage()
    {
        InitializeComponent();
    }
    
    private void OnCollectionViewAppearing(object sender, EventArgs e)
    {
        var visibleItems = ProductList.GetVisualTreeDescendants()
            .OfType<View>()
            .Where(v => v is Frame || v is Border)
            .ToList();
        
        _ = StaggeredAnimationHelper.AnimateListItemsAsync(visibleItems);
    }
}
```

---

## Step 450: Morphing Shape Animation

```csharp
// ============================================
// Shape Morphing Animation with SkiaSharp
// ============================================

public class MorphingShapeView : SKCanvasView
{
    private float _morphProgress; // 0 = circle, 1 = square
    private SKColor _color = SKColors.Blue;
    
    public static readonly BindableProperty MorphProgressProperty =
        BindableProperty.Create(nameof(MorphProgress), typeof(float),
            typeof(MorphingShapeView), 0f,
            propertyChanged: (b, _, n) =>
            {
                ((MorphingShapeView)b)._morphProgress = (float)n;
                ((MorphingShapeView)b).InvalidateSurface();
            });
    
    public float MorphProgress
    {
        get => (float)GetValue(MorphProgressProperty);
        set => SetValue(MorphProgressProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear();
        
        var w = e.Info.Width;
        var h = e.Info.Height;
        var cx = w / 2f;
        var cy = h / 2f;
        var size = Math.Min(w, h) * 0.4f;
        
        using var paint = new SKPaint
        {
            IsAntialias = true,
            Color = _color
        };
        
        using var path = BuildMorphPath(cx, cy, size, _morphProgress);
        canvas.DrawPath(path, paint);
    }
    
    private static SKPath BuildMorphPath(float cx, float cy, float size, float t)
    {
        var path = new SKPath();
        const int points = 8;
        
        for (int i = 0; i < points; i++)
        {
            var angle = (float)(2 * Math.PI * i / points - Math.PI / 2);
            
            // Circle point
            var circleX = cx + (float)(Math.Cos(angle) * size);
            var circleY = cy + (float)(Math.Sin(angle) * size);
            
            // Square point
            var squareAngle = (float)(Math.Round(angle / (Math.PI / 2)) * Math.PI / 2);
            var squareX = cx + (float)(Math.Cos(squareAngle) * size);
            var squareY = cy + (float)(Math.Sin(squareAngle) * size);
            
            // Interpolate
            var x = circleX + (squareX - circleX) * t;
            var y = circleY + (squareY - circleY) * t;
            
            if (i == 0) path.MoveTo(x, y);
            else path.LineTo(x, y);
        }
        
        path.Close();
        return path;
    }
    
    public async Task MorphToSquareAsync()
    {
        await this.AnimateAsync("morph", t => MorphProgress = (float)t, length: 500,
            easing: CustomEasings.SmoothStep);
    }
    
    public async Task MorphToCircleAsync()
    {
        await this.AnimateAsync("morph", t => MorphProgress = 1 - (float)t, length: 500,
            easing: CustomEasings.SmoothStep);
    }
}
```

---

## สรุป Part 45

ใน Part 45 เราได้เรียนรู้:

1. **Animation Basics** - Fade, Scale, Translate, Rotate
2. **Custom Easing** - Bounce, Elastic, Back, SmoothStep
3. **Page Transitions** - Slide, card flip
4. **Lottie Animations** - JSON-based vector animations
5. **Shared Element (Hero)** - Cross-page shared transitions
6. **Particle System** - SkiaSharp particle emitter
7. **Ripple Effect** - Touch ripple wave
8. **Number Counter** - Animated value counting
9. **Staggered List** - Sequential item animations
10. **Shape Morphing** - Interpolated path animations

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 45 | Steps 441-450*
