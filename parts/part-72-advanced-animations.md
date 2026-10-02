# Part 72: Advanced Animations & Transitions
## Steps 711-720: Lottie, Shared Element Transitions, Particle Effects, Motion Design

---

## Step 711: Lottie Animations

```csharp
// ============================================
// Lottie Animation Integration
// ============================================

// Install: SkiaSharp.Extended.UI.Maui (Lottie support)

// In XAML:
// <skia:SKLottieView Source="Animations/loading.json"
//                    IsAnimating="True"
//                    RepeatCount="-1"
//                    HeightRequest="120" WidthRequest="120" />

// Custom Lottie control wrapper
public class LottieView : ContentView
{
    public static readonly BindableProperty SourceProperty =
        BindableProperty.Create(nameof(Source), typeof(string), typeof(LottieView));
    
    public static readonly BindableProperty IsPlayingProperty =
        BindableProperty.Create(nameof(IsPlaying), typeof(bool), typeof(LottieView), true,
            propertyChanged: (b, _, n) => ((LottieView)b).UpdatePlayState((bool)n));
    
    public static readonly BindableProperty LoopProperty =
        BindableProperty.Create(nameof(Loop), typeof(bool), typeof(LottieView), true);
    
    public static readonly BindableProperty SpeedProperty =
        BindableProperty.Create(nameof(Speed), typeof(double), typeof(LottieView), 1.0);
    
    public string Source { get => (string)GetValue(SourceProperty); set => SetValue(SourceProperty, value); }
    public bool IsPlaying { get => (bool)GetValue(IsPlayingProperty); set => SetValue(IsPlayingProperty, value); }
    public bool Loop { get => (bool)GetValue(LoopProperty); set => SetValue(LoopProperty, value); }
    public double Speed { get => (double)GetValue(SpeedProperty); set => SetValue(SpeedProperty, value); }
    
    public event EventHandler? AnimationCompleted;
    
    private void UpdatePlayState(bool playing)
    {
        // Would control underlying Lottie player
    }
    
    public void Stop() => IsPlaying = false;
    public void Play() => IsPlaying = true;
    public void Pause() => IsPlaying = false;
}

// Usage in loading overlay
public class AnimatedLoadingPage : ContentPage
{
    private readonly LottieView _lottie;
    
    public AnimatedLoadingPage()
    {
        Content = new Grid
        {
            Children =
            {
                new BoxView { BackgroundColor = Color.FromArgb("#80000000") },
                new VerticalStackLayout
                {
                    HorizontalOptions = LayoutOptions.Center,
                    VerticalOptions = LayoutOptions.Center,
                    Children =
                    {
                        (_lottie = new LottieView
                        {
                            Source = "loading_food.json",
                            IsPlaying = true,
                            Loop = true,
                            HeightRequest = 150, WidthRequest = 150
                        }),
                        new Label
                        {
                            Text = "กำลังโหลด...",
                            TextColor = Colors.White,
                            FontSize = 16,
                            HorizontalOptions = LayoutOptions.Center
                        }
                    }
                }
            }
        };
    }
}
```

---

## Step 712: Shared Element Transitions

```csharp
// ============================================
// Shared Element Transitions (Navigation)
// ============================================

// TransitionService handles cross-page element animations
public class TransitionService
{
    private static readonly Dictionary<string, WeakReference<View>> _elements = new();
    
    public static void Register(string key, View element)
        => _elements[key] = new WeakReference<View>(element);
    
    public static void Unregister(string key)
        => _elements.Remove(key);
    
    public static View? GetElement(string key)
        => _elements.TryGetValue(key, out var weak)
            && weak.TryGetTarget(out var view) ? view : null;
}

// Source page: restaurant card
public class RestaurantCardView : ContentView
{
    private readonly string _restaurantId;
    
    public RestaurantCardView(string restaurantId)
    {
        _restaurantId = restaurantId;
    }
    
    protected override void OnAppearing() { }
    
    public void RegisterForTransition()
    {
        TransitionService.Register($"restaurant_image_{_restaurantId}", this);
    }
}

// MAUI doesn't have native shared element transitions like Android,
// but we can simulate with positioned overlays

public class NavigationTransitionHelper
{
    // Animate source element position to full screen
    public static async Task AnimateToFullScreenAsync(View sourceView, Page destinationPage)
    {
        var sourceBounds = sourceView.GetAbsoluteBounds();
        
        // Create an overlay that matches source position
        var overlay = new Frame
        {
            BackgroundColor = Colors.White,
            CornerRadius = (float)(sourceView is Frame f ? f.CornerRadius : 0)
        };
        
        // Add to window overlay
        if (Application.Current?.Windows[0] is Window win)
        {
            // Position at source
            AbsoluteLayout.SetLayoutBounds(overlay, sourceBounds);
            
            // Animate to full screen
            await Task.WhenAll(
                overlay.LayoutTo(new Rect(0, 0,
                    win.Width, win.Height), 300, Easing.CubicOut),
                overlay.FadeTo(0, 300));
        }
    }
}
```

---

## Step 713: Page Transition Animations

```csharp
// ============================================
// Custom Page Transitions
// ============================================

public class SlideNavigationPage : NavigationPage
{
    protected override async Task<Page> OnPushAsync(Page page, bool animated)
    {
        if (!animated) return await base.OnPushAsync(page, animated);
        
        page.Opacity = 0;
        page.TranslationX = Width;
        
        var result = await base.OnPushAsync(page, false);
        
        await Task.WhenAll(
            page.FadeTo(1, 250, Easing.CubicOut),
            page.TranslateTo(0, 0, 300, Easing.CubicOut));
        
        return result;
    }
    
    protected override async Task<Page> OnPopAsync(bool animated)
    {
        if (!animated) return await base.OnPopAsync(animated);
        
        var currentPage = CurrentPage;
        
        _ = Task.WhenAll(
            currentPage.FadeTo(0, 250),
            currentPage.TranslateTo(Width, 0, 300, Easing.CubicIn));
        
        return await base.OnPopAsync(false);
    }
}

// Reveal transition (circle expand)
public class RevealTransition
{
    public static async Task RevealAsync(View view, Point origin, uint duration = 400)
    {
        view.Opacity = 0;
        
        // Scale from 0 around the origin point
        view.AnchorX = origin.X / view.Width;
        view.AnchorY = origin.Y / view.Height;
        view.Scale = 0;
        
        await Task.WhenAll(
            view.FadeTo(1, duration / 2),
            view.ScaleTo(1, duration, Easing.CubicOut));
    }
    
    public static async Task CollapseAsync(View view, uint duration = 300)
    {
        await Task.WhenAll(
            view.FadeTo(0, duration),
            view.ScaleTo(0, duration, Easing.CubicIn));
    }
}

// Crossfade between views
public class CrossFadeTransition
{
    public static async Task TransitionAsync(View from, View to, uint duration = 300)
    {
        to.Opacity = 0;
        to.IsVisible = true;
        
        await Task.WhenAll(
            from.FadeTo(0, duration),
            to.FadeTo(1, duration));
        
        from.IsVisible = false;
    }
}
```

---

## Step 714: Animated List Items

```csharp
// ============================================
// Staggered List Animation
// ============================================

public class StaggeredAnimationController
{
    public static async Task AnimateInAsync(
        IEnumerable<View> items,
        uint perItemDelay = 50,
        uint itemDuration = 300)
    {
        // Set initial state
        foreach (var item in items)
        {
            item.Opacity = 0;
            item.TranslationY = 30;
        }
        
        // Animate each item with delay
        var tasks = items.Select((item, i) =>
            Task.Run(async () =>
            {
                await Task.Delay((int)(i * perItemDelay));
                await MainThread.InvokeOnMainThreadAsync(() =>
                    Task.WhenAll(
                        item.FadeTo(1, itemDuration, Easing.CubicOut),
                        item.TranslateTo(0, 0, itemDuration, Easing.CubicOut)));
            }));
        
        await Task.WhenAll(tasks);
    }
    
    public static async Task AnimateOutAsync(
        IEnumerable<View> items,
        uint perItemDelay = 30,
        uint itemDuration = 200)
    {
        var itemList = items.Reverse().ToList();
        var tasks = itemList.Select((item, i) =>
            Task.Run(async () =>
            {
                await Task.Delay((int)(i * perItemDelay));
                await MainThread.InvokeOnMainThreadAsync(() =>
                    Task.WhenAll(
                        item.FadeTo(0, itemDuration),
                        item.TranslateTo(0, -20, itemDuration, Easing.CubicIn)));
            }));
        
        await Task.WhenAll(tasks);
    }
}

// Auto-stagger CollectionView items
public partial class AnimatedCollectionView : CollectionView
{
    private bool _animated;
    
    protected override void OnChildAdded(Element child)
    {
        base.OnChildAdded(child);
        
        if (!_animated && child is View view)
        {
            view.Opacity = 0;
            view.TranslationY = 20;
            
            Dispatcher.Dispatch(async () =>
            {
                await Task.Delay(50 * (int)GetItemIndex(view));
                await Task.WhenAll(
                    view.FadeTo(1, 300, Easing.CubicOut),
                    view.TranslateTo(0, 0, 300, Easing.CubicOut));
            });
        }
    }
    
    private int GetItemIndex(View view) => 0; // simplified
    
    public void MarkAnimated() => _animated = true;
}
```

---

## Step 715: Pull-to-Refresh Animation

```csharp
// ============================================
// Custom Pull-to-Refresh with Animation
// ============================================

public class CustomRefreshView : ContentView
{
    private readonly LottieView _refreshIcon;
    private bool _isRefreshing;
    private double _pullDistance;
    private const double TriggerDistance = 80;
    
    public event EventHandler? RefreshRequested;
    
    public CustomRefreshView()
    {
        _refreshIcon = new LottieView
        {
            Source = "refresh_arrow.json",
            IsPlaying = false,
            HeightRequest = 40, WidthRequest = 40,
            HorizontalOptions = LayoutOptions.Center
        };
        
        var panGesture = new PanGestureRecognizer();
        panGesture.PanUpdated += OnPan;
        GestureRecognizers.Add(panGesture);
    }
    
    private async void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        if (_isRefreshing) return;
        
        switch (e.StatusType)
        {
            case GestureStatus.Running:
                _pullDistance = Math.Max(0, e.TotalY);
                var progress = Math.Min(1.0, _pullDistance / TriggerDistance);
                _refreshIcon.Opacity = progress;
                _refreshIcon.TranslationY = -40 + (progress * 60);
                _refreshIcon.Rotation = progress * 360;
                break;
            
            case GestureStatus.Completed:
                if (_pullDistance >= TriggerDistance)
                    await TriggerRefreshAsync();
                else
                    await AnimateBackAsync();
                break;
        }
    }
    
    private async Task TriggerRefreshAsync()
    {
        _isRefreshing = true;
        _refreshIcon.IsPlaying = true;
        _refreshIcon.Loop = true;
        
        RefreshRequested?.Invoke(this, EventArgs.Empty);
        
        // Auto-hide after callback
        await Task.Delay(2000);
        await AnimateBackAsync();
        _isRefreshing = false;
        _refreshIcon.IsPlaying = false;
    }
    
    private async Task AnimateBackAsync()
    {
        await Task.WhenAll(
            _refreshIcon.FadeTo(0, 200),
            _refreshIcon.TranslateTo(0, -40, 200, Easing.CubicIn));
        _pullDistance = 0;
    }
}
```

---

## Step 716: Skeleton Screen Animation

```csharp
// ============================================
// Animated Skeleton Loading
// ============================================

public class SkeletonView : ContentView
{
    private bool _isAnimating;
    private readonly BoxView _shimmer;
    
    public CornerRadius CornerRadius
    {
        get => _frame.CornerRadius;
        set => _frame.CornerRadius = (float)value.TopLeft;
    }
    
    private readonly Frame _frame;
    
    public SkeletonView()
    {
        _frame = new Frame
        {
            BackgroundColor = Color.FromArgb("#E5E5EA"),
            HasShadow = false, Padding = 0,
            ClipsToBounds = true
        };
        
        _shimmer = new BoxView
        {
            WidthRequest = 100, HorizontalOptions = LayoutOptions.Start
        };
        _shimmer.Background = new LinearGradientBrush(new GradientStopCollection
        {
            new GradientStop(Color.FromArgb("#00E5E5EA"), 0),
            new GradientStop(Color.FromArgb("#80FFFFFF"), 0.5f),
            new GradientStop(Color.FromArgb("#00E5E5EA"), 1),
        }, new Point(0, 0), new Point(1, 0));
        
        _frame.Content = _shimmer;
        Content = _frame;
    }
    
    protected override void OnAppearing()
    {
        base.OnAppearing();
        StartShimmer();
    }
    
    private void StartShimmer()
    {
        if (_isAnimating) return;
        _isAnimating = true;
        
        _shimmer.TranslationX = -Width;
        
        var anim = new Animation(v =>
        {
            _shimmer.TranslationX = (Width * 2) * v - Width;
        });
        
        anim.Commit(this, "shimmer", 16, 1500,
            Easing.Linear, repeat: () => _isAnimating);
    }
    
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _isAnimating = false;
        this.AbortAnimation("shimmer");
    }
}
```

---

## Step 717: Confetti Animation

```csharp
// ============================================
// Confetti / Celebration Animation
// ============================================

public class ConfettiView : ContentView
{
    private readonly AbsoluteLayout _canvas;
    private readonly List<ConfettiParticle> _particles = new();
    private bool _running;
    
    public ConfettiView()
    {
        _canvas = new AbsoluteLayout();
        Content = _canvas;
        InputTransparent = true;
    }
    
    public async Task BurstAsync(int count = 80)
    {
        _running = true;
        SpawnParticles(count);
        await AnimateAsync();
        Cleanup();
    }
    
    private void SpawnParticles(int count)
    {
        var colors = new[] {
            Colors.Red, Colors.Green, Colors.Blue, Colors.Yellow, Colors.Purple, Colors.Orange
        };
        var rng = new Random();
        
        for (int i = 0; i < count; i++)
        {
            var box = new BoxView
            {
                Color = colors[rng.Next(colors.Length)],
                WidthRequest = rng.Next(6, 12),
                HeightRequest = rng.Next(8, 16),
                CornerRadius = 2,
                Rotation = rng.Next(0, 360)
            };
            
            var x = rng.NextDouble() * Width;
            AbsoluteLayout.SetLayoutBounds(box, new Rect(x, -20, AbsoluteLayout.AutoSize, AbsoluteLayout.AutoSize));
            _canvas.Children.Add(box);
            
            _particles.Add(new ConfettiParticle(box, x, rng.NextDouble() * 2 + 1));
        }
    }
    
    private async Task AnimateAsync()
    {
        var start = DateTime.Now;
        
        while (_running && (DateTime.Now - start).TotalSeconds < 3)
        {
            foreach (var p in _particles)
            {
                p.Y += p.SpeedY * 5;
                p.Box.Rotation += p.SpeedX * 3;
                AbsoluteLayout.SetLayoutBounds(p.Box, new Rect(p.X, p.Y,
                    AbsoluteLayout.AutoSize, AbsoluteLayout.AutoSize));
            }
            
            await Task.Delay(16); // ~60fps
        }
    }
    
    private void Cleanup()
    {
        _canvas.Children.Clear();
        _particles.Clear();
    }
}

public class ConfettiParticle
{
    public View Box { get; }
    public double X { get; }
    public double Y { get; set; }
    public double SpeedY { get; }
    public double SpeedX { get; }
    
    public ConfettiParticle(View box, double x, double speed)
    {
        Box = box; X = x; Y = -20;
        SpeedY = speed; SpeedX = new Random().NextDouble() - 0.5;
    }
}
```

---

## Step 718: Swipe Gesture Actions

```csharp
// ============================================
// Swipe-to-Delete / Swipe Actions
// ============================================

public class SwipeActionView : ContentView
{
    private readonly Grid _container;
    private readonly ContentView _contentContainer;
    private readonly HorizontalStackLayout _actionBar;
    private double _startX;
    private const double MaxSwipe = -80;
    
    public View Content
    {
        get => _contentContainer.Content;
        set => _contentContainer.Content = value;
    }
    
    public SwipeActionView()
    {
        _actionBar = new HorizontalStackLayout
        {
            Spacing = 0,
            HorizontalOptions = LayoutOptions.End,
            VerticalOptions = LayoutOptions.Fill
        };
        
        _contentContainer = new ContentView();
        
        _container = new Grid { Children = { _actionBar, _contentContainer } };
        base.Content = _container;
        
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += OnPan;
        _contentContainer.GestureRecognizers.Add(pan);
    }
    
    public void AddAction(string label, Color color, Command command)
    {
        var btn = new Button
        {
            Text = label,
            BackgroundColor = color,
            TextColor = Colors.White,
            WidthRequest = 80,
            Command = command
        };
        _actionBar.Children.Add(btn);
    }
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        switch (e.StatusType)
        {
            case GestureStatus.Started:
                _startX = _contentContainer.TranslationX;
                break;
            
            case GestureStatus.Running:
                var newX = Math.Clamp(_startX + e.TotalX, MaxSwipe, 0);
                _contentContainer.TranslationX = newX;
                break;
            
            case GestureStatus.Completed:
                _ = SnapAsync(_contentContainer.TranslationX < MaxSwipe / 2
                    ? MaxSwipe : 0);
                break;
        }
    }
    
    private Task SnapAsync(double target)
        => _contentContainer.TranslateTo(target, 0, 200, Easing.SpringOut);
    
    public Task CloseAsync() => SnapAsync(0);
}
```

---

## Step 719: Motion Design System

```csharp
// ============================================
// Motion Design Tokens
// ============================================

public static class Motion
{
    // Duration tokens
    public static class Duration
    {
        public const uint Instant = 0;
        public const uint Quick = 150;
        public const uint Normal = 250;
        public const uint Moderate = 350;
        public const uint Slow = 500;
        public const uint Lazy = 750;
    }
    
    // Easing tokens
    public static class Easing
    {
        public static Microsoft.Maui.Easing Standard => Microsoft.Maui.Easing.CubicInOut;
        public static Microsoft.Maui.Easing Enter => Microsoft.Maui.Easing.CubicOut;
        public static Microsoft.Maui.Easing Exit => Microsoft.Maui.Easing.CubicIn;
        public static Microsoft.Maui.Easing Spring => Microsoft.Maui.Easing.SpringOut;
        public static Microsoft.Maui.Easing Bounce => Microsoft.Maui.Easing.BounceOut;
    }
    
    // Preset animations
    public static async Task FadeIn(View view)
    {
        view.Opacity = 0;
        await view.FadeTo(1, Duration.Normal, Easing.Enter);
    }
    
    public static async Task FadeOut(View view)
        => await view.FadeTo(0, Duration.Normal, Easing.Exit);
    
    public static async Task SlideUp(View view)
    {
        view.TranslationY = 24;
        view.Opacity = 0;
        await Task.WhenAll(
            view.TranslateTo(0, 0, Duration.Moderate, Easing.Enter),
            view.FadeTo(1, Duration.Normal, Easing.Enter));
    }
    
    public static async Task Pop(View view)
    {
        view.Scale = 0.8;
        view.Opacity = 0;
        await Task.WhenAll(
            view.ScaleTo(1, Duration.Normal, Easing.Spring),
            view.FadeTo(1, Duration.Quick, Easing.Enter));
    }
    
    public static async Task Dismiss(View view)
    {
        await Task.WhenAll(
            view.ScaleTo(0.8, Duration.Quick, Easing.Exit),
            view.FadeTo(0, Duration.Quick, Easing.Exit));
    }
    
    public static async Task Pulse(View view, double scale = 1.1)
    {
        await view.ScaleTo(scale, Duration.Quick, Easing.CubicOut);
        await view.ScaleTo(1, Duration.Quick, Easing.CubicIn);
    }
}
```

---

## Step 720: Scroll-Driven Animations

```csharp
// ============================================
// Scroll-Driven Animation (Parallax)
// ============================================

public class ParallaxScrollView : ScrollView
{
    private View? _heroImage;
    private View? _header;
    
    public void SetHeroImage(View image) => _heroImage = image;
    public void SetHeader(View header) => _header = header;
    
    protected override void OnScrolled(ScrolledEventArgs e)
    {
        base.OnScrolled(e);
        
        var scrollY = e.ScrollY;
        
        // Parallax hero: moves at half speed
        if (_heroImage != null)
        {
            _heroImage.TranslationY = scrollY * 0.5;
            _heroImage.Opacity = 1 - (scrollY / 200.0);
        }
        
        // Sticky header: fade in as hero fades
        if (_header != null)
        {
            _header.Opacity = Math.Clamp(scrollY / 100.0, 0, 1);
        }
    }
}

// Animate items as they scroll into view
public class ScrollAnimationManager
{
    private readonly HashSet<View> _animated = new();
    
    public void CheckVisibility(View item, double scrollY, double viewHeight)
    {
        if (_animated.Contains(item)) return;
        
        var itemY = item.GetAbsoluteBounds().Y;
        var threshold = scrollY + viewHeight - 60;
        
        if (itemY < threshold)
        {
            _animated.Add(item);
            _ = Motion.SlideUp(item);
        }
    }
}
```

---

## สรุป Part 72

ใน Part 72 เราได้เรียนรู้:

1. **Lottie Animations** - JSON animations, play/pause/loop
2. **Shared Element Transitions** - Element registration, position-based
3. **Page Transitions** - Slide, reveal, crossfade
4. **Staggered List Animations** - Delayed sequential animation
5. **Pull-to-Refresh** - Custom pull animation with progress
6. **Skeleton Shimmer** - Shimmer gradient animation loop
7. **Confetti** - Particle burst celebration effect
8. **Swipe Actions** - Pan gesture with snap, delete actions
9. **Motion Design System** - Duration and easing tokens
10. **Scroll-Driven Animations** - Parallax, sticky header, reveal on scroll

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 72 | Steps 711-720*
