# Part 30: Advanced UI Patterns
## Steps 291-300: Professional UI Development

---

## Step 291: Custom Layouts

```csharp
// ============================================
// Custom Layout: WrapLayout
// ============================================

public class WrapLayout : Layout
{
    public static readonly BindableProperty SpacingProperty =
        BindableProperty.Create(nameof(Spacing), typeof(double), typeof(WrapLayout), 8.0,
            propertyChanged: (b, _, _) => ((WrapLayout)b).InvalidateMeasure());
    
    public double Spacing
    {
        get => (double)GetValue(SpacingProperty);
        set => SetValue(SpacingProperty, value);
    }
    
    protected override ILayoutManager CreateLayoutManager() => new WrapLayoutManager(this);
}

public class WrapLayoutManager : ILayoutManager
{
    private readonly WrapLayout _layout;
    
    public WrapLayoutManager(WrapLayout layout) => _layout = layout;
    
    public Size Measure(double widthConstraint, double heightConstraint)
    {
        var spacing = _layout.Spacing;
        var currentX = 0.0;
        var currentY = 0.0;
        var rowHeight = 0.0;
        var totalWidth = 0.0;
        
        foreach (var child in _layout)
        {
            if (!child.IsVisible) continue;
            
            var childSize = child.Measure(widthConstraint, heightConstraint);
            
            if (currentX + childSize.Width > widthConstraint && currentX > 0)
            {
                currentX = 0;
                currentY += rowHeight + spacing;
                rowHeight = 0;
            }
            
            currentX += childSize.Width + spacing;
            rowHeight = Math.Max(rowHeight, childSize.Height);
            totalWidth = Math.Max(totalWidth, currentX);
        }
        
        return new Size(totalWidth, currentY + rowHeight);
    }
    
    public Size ArrangeChildren(Rect bounds)
    {
        var spacing = _layout.Spacing;
        var currentX = bounds.X;
        var currentY = bounds.Y;
        var rowHeight = 0.0;
        
        // First pass: measure all children
        var childSizes = new Dictionary<IView, Size>();
        foreach (var child in _layout)
        {
            if (!child.IsVisible) continue;
            childSizes[child] = child.Measure(bounds.Width, bounds.Height);
        }
        
        // Second pass: arrange
        foreach (var child in _layout)
        {
            if (!child.IsVisible) continue;
            
            var childSize = childSizes[child];
            
            if (currentX + childSize.Width > bounds.Right && currentX > bounds.X)
            {
                currentX = bounds.X;
                currentY += rowHeight + spacing;
                rowHeight = 0;
            }
            
            child.Arrange(new Rect(currentX, currentY, childSize.Width, childSize.Height));
            
            currentX += childSize.Width + spacing;
            rowHeight = Math.Max(rowHeight, childSize.Height);
        }
        
        return bounds.Size;
    }
}
```

---

## Step 292: Drag and Drop

```csharp
// ============================================
// Drag and Drop в MAUI
// ============================================

// XAML:
/*
<CollectionView ItemsSource="{Binding Items}">
    <CollectionView.ItemTemplate>
        <DataTemplate>
            <Frame>
                <Frame.GestureRecognizers>
                    <DragGestureRecognizer CanDrag="True"
                        DragStartingCommand="{Binding DragStartCommand}"
                        CommandParameter="{Binding .}" />
                    <DropGestureRecognizer AllowDrop="True"
                        DragOverCommand="{Binding DragOverCommand}"
                        DropCommand="{Binding DropCommand}"
                        CommandParameter="{Binding .}" />
                </Frame.GestureRecognizers>
                <Label Text="{Binding Name}" />
            </Frame>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
*/

public partial class DragDropViewModel : ObservableObject
{
    [ObservableProperty] private ObservableCollection<TaskItem> _items = new();
    [ObservableProperty] private TaskItem? _draggingItem;
    
    [RelayCommand]
    private void DragStart(TaskItem item)
    {
        DraggingItem = item;
    }
    
    [RelayCommand]
    private void Drop(TaskItem targetItem)
    {
        if (DraggingItem == null || DraggingItem == targetItem) return;
        
        var sourceIndex = Items.IndexOf(DraggingItem);
        var targetIndex = Items.IndexOf(targetItem);
        
        if (sourceIndex < 0 || targetIndex < 0) return;
        
        Items.Move(sourceIndex, targetIndex);
        DraggingItem = null;
    }
    
    [RelayCommand]
    private void DragOver(TaskItem item)
    {
        // Visual feedback during drag
    }
}

public record TaskItem(string Id, string Name, int Order);
```

---

## Step 293: Gesture Recognizers

```csharp
// ============================================
// Custom Gestures
// ============================================

public class SwipeToDeleteView : ContentView
{
    private double _startX;
    private bool _isDeleting;
    
    public static readonly BindableProperty DeleteCommandProperty =
        BindableProperty.Create(nameof(DeleteCommand), typeof(ICommand), typeof(SwipeToDeleteView));
    
    public ICommand? DeleteCommand
    {
        get => (ICommand?)GetValue(DeleteCommandProperty);
        set => SetValue(DeleteCommandProperty, value);
    }
    
    public SwipeToDeleteView()
    {
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += OnPan;
        GestureRecognizers.Add(pan);
    }
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        switch (e.StatusType)
        {
            case GestureStatus.Started:
                _startX = TranslationX;
                break;
            
            case GestureStatus.Running:
                var translation = _startX + e.TotalX;
                if (translation < 0) // Only allow left swipe
                    TranslationX = Math.Max(translation, -Width);
                
                // Show delete indicator
                Opacity = 1 - Math.Abs(TranslationX) / Width * 0.5;
                break;
            
            case GestureStatus.Completed:
                if (TranslationX < -Width * 0.4)
                {
                    // Trigger delete
                    _ = AnimateDeleteAsync();
                }
                else
                {
                    // Snap back
                    _ = this.TranslateTo(0, 0, 200, Easing.SpringOut);
                    this.FadeTo(1, 200);
                }
                break;
            
            case GestureStatus.Canceled:
                _ = this.TranslateTo(0, 0, 200, Easing.SpringOut);
                this.FadeTo(1, 200);
                break;
        }
    }
    
    private async Task AnimateDeleteAsync()
    {
        await Task.WhenAll(
            this.TranslateTo(-Width, 0, 300, Easing.CubicIn),
            this.FadeTo(0, 300)
        );
        
        DeleteCommand?.Execute(BindingContext);
    }
}
```

---

## Step 294: Pull to Refresh Pattern

```csharp
// ============================================
// Pull to Refresh with Custom Animation
// ============================================

// XAML:
/*
<RefreshView Command="{Binding RefreshCommand}" IsRefreshing="{Binding IsRefreshing}">
    <CollectionView ItemsSource="{Binding Items}" />
</RefreshView>
*/

public partial class ProductListViewModel : ObservableObject
{
    private readonly IProductApiService _api;
    
    [ObservableProperty] private ObservableCollection<Product> _items = new();
    [ObservableProperty] private bool _isRefreshing;
    [ObservableProperty] private bool _isLoadingMore;
    [ObservableProperty] private bool _hasMore = true;
    [ObservableProperty] private int _page = 1;
    [ObservableProperty] private string? _errorMessage;
    
    private const int PageSize = 20;
    
    public ProductListViewModel(IProductApiService api) => _api = api;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        Page = 1;
        HasMore = true;
        var result = await _api.GetPagedAsync(Page, PageSize);
        
        Items = new ObservableCollection<Product>(result.Items);
        HasMore = result.HasNextPage;
        Page = 2;
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        IsRefreshing = true;
        try
        {
            await LoadAsync();
        }
        finally
        {
            IsRefreshing = false;
        }
    }
    
    [RelayCommand]
    private async Task LoadMoreAsync()
    {
        if (IsLoadingMore || !HasMore) return;
        
        IsLoadingMore = true;
        try
        {
            var result = await _api.GetPagedAsync(Page, PageSize);
            foreach (var item in result.Items) Items.Add(item);
            HasMore = result.HasNextPage;
            Page++;
        }
        finally
        {
            IsLoadingMore = false;
        }
    }
}
```

---

## Step 295: Bottom Sheet

```csharp
// ============================================
// Custom Bottom Sheet
// ============================================

public class BottomSheetView : ContentView
{
    private double _startY;
    private readonly double _peekHeight = 200;
    
    public static readonly BindableProperty IsExpandedProperty =
        BindableProperty.Create(nameof(IsExpanded), typeof(bool), typeof(BottomSheetView),
            false, propertyChanged: OnIsExpandedChanged);
    
    public bool IsExpanded
    {
        get => (bool)GetValue(IsExpandedProperty);
        set => SetValue(IsExpandedProperty, value);
    }
    
    public BottomSheetView()
    {
        var pan = new PanGestureRecognizer();
        pan.PanUpdated += OnPan;
        GestureRecognizers.Add(pan);
        
        var tap = new TapGestureRecognizer();
        tap.Tapped += OnTapped;
        GestureRecognizers.Add(tap);
    }
    
    private static void OnIsExpandedChanged(BindableObject bindable, object oldValue, object newValue)
    {
        var sheet = (BottomSheetView)bindable;
        _ = sheet.AnimateAsync((bool)newValue);
    }
    
    private void OnTapped(object? sender, TappedEventArgs e)
        => IsExpanded = !IsExpanded;
    
    private void OnPan(object? sender, PanUpdatedEventArgs e)
    {
        switch (e.StatusType)
        {
            case GestureStatus.Started:
                _startY = TranslationY;
                break;
            
            case GestureStatus.Running:
                var newY = _startY + e.TotalY;
                TranslationY = Math.Max(0, newY);
                break;
            
            case GestureStatus.Completed:
                var expanded = TranslationY < Height / 2;
                IsExpanded = expanded;
                break;
        }
    }
    
    private async Task AnimateAsync(bool expand)
    {
        var targetY = expand ? 0 : Height - _peekHeight;
        await this.TranslateTo(0, targetY, 300, Easing.CubicOut);
    }
    
    protected override void OnSizeAllocated(double width, double height)
    {
        base.OnSizeAllocated(width, height);
        // Start at peek position
        if (!IsExpanded) TranslationY = height - _peekHeight;
    }
}
```

---

## Step 296: Skeleton Loading

```csharp
// ============================================
// Skeleton Loading Screen
// ============================================

public class SkeletonView : ContentView
{
    private Animation? _shimmerAnimation;
    
    protected override void OnAttachedToWindow()
    {
        base.OnAttachedToWindow();
        StartShimmer();
    }
    
    protected override void OnDetachingFrom(Page page)
    {
        base.OnDetachingFrom(page);
        StopShimmer();
    }
    
    private void StartShimmer()
    {
        _shimmerAnimation = new Animation(v =>
        {
            // Animate gradient position
            Opacity = 0.5 + v * 0.5;
        }, 0, 1);
        
        _shimmerAnimation.Commit(this, "shimmer", 
            length: 1000, 
            repeat: () => true,
            easing: Easing.SinInOut);
    }
    
    private void StopShimmer()
    {
        this.AbortAnimation("shimmer");
        _shimmerAnimation = null;
    }
}

// XAML Usage:
/*
<StackLayout IsVisible="{Binding IsLoading}">
    <!-- Skeleton for product list item -->
    <Frame Padding="12" Margin="8,4">
        <Grid ColumnDefinitions="60,*" ColumnSpacing="12">
            <!-- Image placeholder -->
            <local:SkeletonView Grid.Column="0" 
                WidthRequest="60" HeightRequest="60"
                BackgroundColor="{AppThemeBinding Light=#E0E0E0, Dark=#333333}"
                CornerRadius="8" />
            
            <!-- Text placeholders -->
            <StackLayout Grid.Column="1" Spacing="8">
                <local:SkeletonView HeightRequest="16" WidthRequest="120"
                    BackgroundColor="{AppThemeBinding Light=#E0E0E0, Dark=#333333}"
                    HorizontalOptions="Start" />
                <local:SkeletonView HeightRequest="12" 
                    BackgroundColor="{AppThemeBinding Light=#E0E0E0, Dark=#333333}" />
                <local:SkeletonView HeightRequest="12" WidthRequest="80"
                    BackgroundColor="{AppThemeBinding Light=#E0E0E0, Dark=#333333}"
                    HorizontalOptions="Start" />
            </StackLayout>
        </Grid>
    </Frame>
</StackLayout>
*/
```

---

## Step 297: Infinite Scroll

```csharp
// ============================================
// Infinite Scroll CollectionView
// ============================================

public class InfiniteScrollBehavior : Behavior<CollectionView>
{
    public static readonly BindableProperty LoadMoreCommandProperty =
        BindableProperty.Create(nameof(LoadMoreCommand), typeof(ICommand), 
            typeof(InfiniteScrollBehavior));
    
    public static readonly BindableProperty ThresholdProperty =
        BindableProperty.Create(nameof(Threshold), typeof(double), 
            typeof(InfiniteScrollBehavior), 0.8);
    
    public ICommand? LoadMoreCommand
    {
        get => (ICommand?)GetValue(LoadMoreCommandProperty);
        set => SetValue(LoadMoreCommandProperty, value);
    }
    
    public double Threshold
    {
        get => (double)GetValue(ThresholdProperty);
        set => SetValue(ThresholdProperty, value);
    }
    
    private CollectionView? _view;
    
    protected override void OnAttachedTo(CollectionView bindable)
    {
        base.OnAttachedTo(bindable);
        _view = bindable;
        bindable.Scrolled += OnScrolled;
    }
    
    protected override void OnDetachingFrom(CollectionView bindable)
    {
        base.OnDetachingFrom(bindable);
        bindable.Scrolled -= OnScrolled;
        _view = null;
    }
    
    private void OnScrolled(object? sender, ItemsViewScrolledEventArgs e)
    {
        if (_view?.ItemsSource == null) return;
        
        var itemCount = 0;
        if (_view.ItemsSource is System.Collections.ICollection collection)
            itemCount = collection.Count;
        
        if (itemCount == 0) return;
        
        // Load more when 80% scrolled
        var threshold = (int)(itemCount * Threshold);
        
        if (e.LastVisibleItemIndex >= threshold)
            LoadMoreCommand?.Execute(null);
    }
}

// XAML:
/*
<CollectionView ItemsSource="{Binding Items}">
    <CollectionView.Behaviors>
        <behaviors:InfiniteScrollBehavior 
            LoadMoreCommand="{Binding LoadMoreCommand}" 
            Threshold="0.8" />
    </CollectionView.Behaviors>
    
    <CollectionView.Footer>
        <ActivityIndicator IsRunning="{Binding IsLoadingMore}"
                          IsVisible="{Binding IsLoadingMore}" />
    </CollectionView.Footer>
</CollectionView>
*/
```

---

## Step 298: Snackbar & Toast

```csharp
// ============================================
// User Feedback UI
// ============================================

public static class UserFeedback
{
    public static async Task ShowSuccessAsync(string message)
    {
        var snackbar = Snackbar.Make(
            $"✅ {message}",
            visualOptions: new SnackbarOptions
            {
                BackgroundColor = Colors.Green,
                TextColor = Colors.White,
                CornerRadius = new CornerRadius(10)
            },
            duration: TimeSpan.FromSeconds(3));
        
        await snackbar.Show();
    }
    
    public static async Task ShowErrorAsync(string message, string? action = null, 
        Action? actionCallback = null)
    {
        if (action != null && actionCallback != null)
        {
            var snackbar = Snackbar.Make(
                $"❌ {message}",
                actionCallback,
                action,
                TimeSpan.FromSeconds(5),
                new SnackbarOptions
                {
                    BackgroundColor = Colors.Red,
                    TextColor = Colors.White
                });
            await snackbar.Show();
        }
        else
        {
            await Toast.Make($"❌ {message}", ToastDuration.Long).Show();
        }
    }
    
    public static async Task ShowInfoAsync(string message)
    {
        await Toast.Make($"ℹ️ {message}", ToastDuration.Short).Show();
    }
    
    public static async Task<bool> ConfirmAsync(string title, string message, 
        string confirm = "ยืนยัน", string cancel = "ยกเลิก")
    {
        return await Shell.Current.DisplayAlert(title, message, confirm, cancel);
    }
    
    public static async Task<string?> PromptAsync(string title, string placeholder = "")
    {
        return await Shell.Current.DisplayPromptAsync(title, string.Empty, 
            "ตกลง", "ยกเลิก", placeholder);
    }
    
    public static async Task<string?> ShowActionSheetAsync(string title, 
        string cancel, params string[] options)
    {
        return await Shell.Current.DisplayActionSheet(title, cancel, null, options);
    }
}
```

---

## Step 299: Advanced CollectionView

```csharp
// ============================================
// Grouped CollectionView with Headers
// ============================================

public class ProductGroup : ObservableCollection<Product>
{
    public string Category { get; }
    public int ProductCount => Count;
    
    public ProductGroup(string category, IEnumerable<Product> products) : base(products)
        => Category = category;
}

public partial class GroupedProductsViewModel : ObservableObject
{
    private readonly IProductRepository _repo;
    
    [ObservableProperty] private ObservableCollection<ProductGroup> _groups = new();
    [ObservableProperty] private bool _isLoading;
    
    public GroupedProductsViewModel(IProductRepository repo) => _repo = repo;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        var products = await _repo.GetAllAsync();
        
        var groups = products
            .GroupBy(p => p.Category)
            .OrderBy(g => g.Key)
            .Select(g => new ProductGroup(g.Key, g.OrderBy(p => p.Name)))
            .ToList();
        
        Groups = new ObservableCollection<ProductGroup>(groups);
        IsLoading = false;
    }
}

// XAML for grouped CollectionView:
/*
<CollectionView ItemsSource="{Binding Groups}" IsGrouped="True">
    <CollectionView.GroupHeaderTemplate>
        <DataTemplate x:DataType="local:ProductGroup">
            <Grid BackgroundColor="{AppThemeBinding Light=#F5F5F5, Dark=#2A2A2A}" Padding="16,8">
                <Label Text="{Binding Category}" FontAttributes="Bold" />
                <Label Text="{Binding ProductCount, StringFormat='{0} รายการ'}"
                       HorizontalOptions="End" TextColor="Gray" />
            </Grid>
        </DataTemplate>
    </CollectionView.GroupHeaderTemplate>
    
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="models:Product">
            <Frame Margin="8,4" Padding="12">
                <Label Text="{Binding Name}" />
            </Frame>
        </DataTemplate>
    </CollectionView.ItemTemplate>
    
    <CollectionView.EmptyView>
        <StackLayout HorizontalOptions="Center" VerticalOptions="Center">
            <Image Source="empty_state.png" WidthRequest="100" />
            <Label Text="ไม่พบข้อมูล" HorizontalOptions="Center" />
        </StackLayout>
    </CollectionView.EmptyView>
</CollectionView>
*/
```

---

## Step 300: UI Testing

```csharp
// ============================================
// UI Testing Patterns
// ============================================

// Using Xamarin.UITest or MAUI Test patterns
// For unit-testable UI logic

public class UITestHelpers
{
    // Test ViewModel binding behavior
    public static async Task AssertCommandExecutesAsync(
        IAsyncRelayCommand command, 
        Action<object?> verify,
        object? parameter = null)
    {
        await command.ExecuteAsync(parameter);
        verify(parameter);
    }
}

// ViewModel testability patterns
public partial class SearchViewModel : ObservableObject
{
    private readonly IProductApiService _api;
    private readonly IDebouncer _debouncer;
    
    [ObservableProperty] private string _searchText = string.Empty;
    [ObservableProperty] private ObservableCollection<Product> _results = new();
    [ObservableProperty] private bool _isSearching;
    
    public SearchViewModel(IProductApiService api, IDebouncer debouncer)
    {
        _api = api;
        _debouncer = debouncer;
    }
    
    partial void OnSearchTextChanged(string value)
    {
        _debouncer.Debounce(() => _ = SearchAsync(value));
    }
    
    private async Task SearchAsync(string query)
    {
        if (query.Length < 2) { Results.Clear(); return; }
        
        IsSearching = true;
        try
        {
            var products = await _api.SearchAsync(query);
            Results = new ObservableCollection<Product>(products);
        }
        finally
        {
            IsSearching = false;
        }
    }
}

public interface IDebouncer
{
    void Debounce(Action action, int milliseconds = 300);
}

public class Debouncer : IDebouncer
{
    private CancellationTokenSource? _cts;
    
    public void Debounce(Action action, int milliseconds = 300)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        var token = _cts.Token;
        
        _ = Task.Delay(milliseconds, token).ContinueWith(
            t => { if (!t.IsCanceled) action(); },
            TaskScheduler.FromCurrentSynchronizationContext());
    }
}

// Test:
public class SearchViewModelTests
{
    [Fact]
    public async Task Search_WithShortQuery_ClearsResults()
    {
        var mockApi = new Mock<IProductApiService>();
        var debouncer = new ImmediateDebouncer(); // No delay for tests
        var vm = new SearchViewModel(mockApi.Object, debouncer);
        
        vm.SearchText = "a"; // Too short
        await Task.Delay(50);
        
        vm.Results.Should().BeEmpty();
        mockApi.Verify(x => x.SearchAsync(It.IsAny<string>()), Times.Never);
    }
}

// Immediate debouncer for tests
public class ImmediateDebouncer : IDebouncer
{
    public void Debounce(Action action, int milliseconds = 300) => action();
}
```

---

## สรุป Part 30

ใน Part 30 เราได้เรียนรู้:

1. **Custom Layout** - WrapLayout implementation
2. **Drag and Drop** - Reorder items
3. **Gesture Recognizers** - Swipe to delete
4. **Pull to Refresh** - Paged refresh pattern
5. **Bottom Sheet** - Custom sliding panel
6. **Skeleton Loading** - Loading placeholders
7. **Infinite Scroll** - Behavior pattern
8. **Snackbar & Toast** - User feedback
9. **Grouped CollectionView** - Category groups
10. **UI Testing** - Debouncer, testable ViewModels

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 30 | Steps 291-300*
