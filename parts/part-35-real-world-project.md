# Part 35: Real-World Project - E-Commerce App
## Steps 341-350: Complete App Development

---

## Step 341: Project Architecture

```
// ============================================
// E-Commerce App Architecture
// ============================================

/*
ShopMAUI/
├── ShopMAUI/                          # MAUI App Project
│   ├── MauiProgram.cs
│   ├── AppShell.xaml
│   ├── Platforms/
│   │   ├── Android/
│   │   └── iOS/
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── LoginPage.xaml
│   │   │   ├── LoginViewModel.cs
│   │   │   ├── RegisterPage.xaml
│   │   │   └── RegisterViewModel.cs
│   │   ├── Products/
│   │   │   ├── ProductListPage.xaml
│   │   │   ├── ProductListViewModel.cs
│   │   │   ├── ProductDetailPage.xaml
│   │   │   └── ProductDetailViewModel.cs
│   │   ├── Cart/
│   │   │   ├── CartPage.xaml
│   │   │   └── CartViewModel.cs
│   │   ├── Orders/
│   │   │   ├── OrderListPage.xaml
│   │   │   ├── OrderDetailPage.xaml
│   │   │   └── OrderViewModel.cs
│   │   └── Profile/
│   │       ├── ProfilePage.xaml
│   │       └── ProfileViewModel.cs
│   ├── Common/
│   │   ├── Controls/
│   │   ├── Converters/
│   │   ├── Extensions/
│   │   └── Behaviors/
│   └── Resources/
│       ├── Styles/
│       └── Images/
│
├── ShopMAUI.Core/                     # Business Logic
│   ├── Entities/
│   ├── Interfaces/
│   ├── Services/
│   └── DTOs/
│
├── ShopMAUI.Infrastructure/           # Data Layer
│   ├── Database/
│   ├── Api/
│   └── Storage/
│
└── ShopMAUI.Tests/                    # Tests
    ├── Unit/
    └── Integration/
*/
```

---

## Step 342: MauiProgram.cs

```csharp
// ============================================
// Complete MauiProgram.cs
// ============================================

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            .UseMauiCommunityToolkit()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-SemiBold.ttf", "OpenSansSemiBold");
                fonts.AddFont("fa-solid-900.ttf", "FontAwesome");
            });
        
        // Logging
        builder.Logging
            .AddDebug()
            .SetMinimumLevel(
#if DEBUG
                LogLevel.Debug
#else
                LogLevel.Warning
#endif
            );
        
        var services = builder.Services;
        
        // Database
        services.AddSingleton<SQLiteAsyncConnection>(sp =>
        {
            var dbPath = Path.Combine(FileSystem.AppDataDirectory, "shop.db");
            var conn = new SQLiteAsyncConnection(dbPath);
            conn.CreateTablesAsync<Product, Cart, CartItem, Order, OrderItem, UserProfile>()
                .GetAwaiter().GetResult();
            return conn;
        });
        
        // HTTP Clients
        services.AddHttpClient("api", client =>
        {
            client.BaseAddress = new Uri(AppConfiguration.ApiBaseUrl);
            client.DefaultRequestHeaders.Add("Accept", "application/json");
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddHttpMessageHandler<AuthHeaderHandler>()
        .AddHttpMessageHandler<RetryHandler>();
        
        // Auth
        services.AddSingleton<TokenManager>();
        services.AddSingleton<AuthService>();
        services.AddScoped<AuthHeaderHandler>();
        
        // Repositories
        services.AddScoped<IProductRepository, CachedProductRepository>(sp =>
            new CachedProductRepository(
                new SqliteProductRepository(sp.GetRequiredService<SQLiteAsyncConnection>())));
        
        services.AddScoped<ICartRepository, CartRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUserProfileRepository, UserProfileRepository>();
        
        // API Services
        services.AddScoped<IProductApiService, ProductApiService>();
        services.AddScoped<IOrderApiService, OrderApiService>();
        services.AddScoped<IAuthApiService, AuthApiService>();
        
        // Business Services
        services.AddScoped<CartService>();
        services.AddScoped<OrderService>();
        services.AddSingleton<NotificationCenter>();
        
        // Event Bus
        services.AddSingleton<IEventBus, WeakEventBus>();
        
        // ViewModels
        services.AddTransient<LoginViewModel>();
        services.AddTransient<RegisterViewModel>();
        services.AddTransient<ProductListViewModel>();
        services.AddTransient<ProductDetailViewModel>();
        services.AddTransient<CartViewModel>();
        services.AddTransient<OrderListViewModel>();
        services.AddTransient<OrderDetailViewModel>();
        services.AddTransient<ProfileViewModel>();
        
        // Pages
        services.AddTransient<LoginPage>();
        services.AddTransient<RegisterPage>();
        services.AddTransient<ProductListPage>();
        services.AddTransient<ProductDetailPage>();
        services.AddTransient<CartPage>();
        services.AddTransient<OrderListPage>();
        services.AddTransient<OrderDetailPage>();
        services.AddTransient<ProfilePage>();
        
        return builder.Build();
    }
}
```

---

## Step 343: AppShell

```xml
<!-- AppShell.xaml -->
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:pages="clr-namespace:ShopMAUI.Features"
       x:Class="ShopMAUI.AppShell"
       Shell.NavBarHasShadow="True">
    
    <!-- Unauthenticated routes (flyout disabled) -->
    <ShellContent Route="login" ContentTemplate="{DataTemplate pages:LoginPage}" 
                  Shell.FlyoutItemIsVisible="False" />
    <ShellContent Route="register" ContentTemplate="{DataTemplate pages:RegisterPage}"
                  Shell.FlyoutItemIsVisible="False" />
    
    <!-- Authenticated tab bar -->
    <TabBar Route="main" x:Name="MainTabs">
        
        <ShellContent Title="สินค้า" Icon="shopping.png"
                      Route="products"
                      ContentTemplate="{DataTemplate pages:ProductListPage}" />
        
        <ShellContent Title="ตะกร้า" Icon="cart.png"
                      Route="cart"
                      ContentTemplate="{DataTemplate pages:CartPage}">
            <!-- Cart badge from ViewModel -->
        </ShellContent>
        
        <ShellContent Title="คำสั่งซื้อ" Icon="orders.png"
                      Route="orders"
                      ContentTemplate="{DataTemplate pages:OrderListPage}" />
        
        <ShellContent Title="โปรไฟล์" Icon="profile.png"
                      Route="profile"
                      ContentTemplate="{DataTemplate pages:ProfilePage}" />
    </TabBar>
    
</Shell>
```

```csharp
// AppShell.xaml.cs
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        
        Routing.RegisterRoute("product-detail", typeof(ProductDetailPage));
        Routing.RegisterRoute("order-detail", typeof(OrderDetailPage));
        Routing.RegisterRoute("checkout", typeof(CheckoutPage));
        Routing.RegisterRoute("payment", typeof(PaymentPage));
    }
}
```

---

## Step 344: Product List

```csharp
// ============================================
// Product List - Full Implementation
// ============================================

public partial class ProductListViewModel : PageViewModel
{
    private readonly IProductApiService _api;
    private readonly CartService _cartService;
    private readonly IEventBus _bus;
    
    [ObservableProperty] private ObservableCollection<ProductCardViewModel> _products = new();
    [ObservableProperty] private bool _isRefreshing;
    [ObservableProperty] private bool _isLoadingMore;
    [ObservableProperty] private bool _hasMore = true;
    [ObservableProperty] private string _searchText = string.Empty;
    [ObservableProperty] private string _selectedCategory = "ทั้งหมด";
    [ObservableProperty] private ProductSortOrder _sortOrder = ProductSortOrder.Newest;
    [ObservableProperty] private ObservableCollection<string> _categories = new() { "ทั้งหมด" };
    
    private int _page = 1;
    private const int PageSize = 20;
    
    public ProductListViewModel(
        IProductApiService api,
        CartService cartService,
        IEventBus bus)
    {
        _api = api;
        _cartService = cartService;
        _bus = bus;
    }
    
    [RelayCommand]
    private async Task InitializeAsync()
    {
        await ExecuteAsync(async () =>
        {
            var cats = await _api.GetCategoriesAsync();
            Categories = new ObservableCollection<string>(new[] { "ทั้งหมด" }.Concat(cats));
            await LoadProductsAsync(reset: true);
        });
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        IsRefreshing = true;
        await LoadProductsAsync(reset: true);
        IsRefreshing = false;
    }
    
    [RelayCommand]
    private async Task LoadMoreAsync()
    {
        if (IsLoadingMore || !HasMore) return;
        IsLoadingMore = true;
        await LoadProductsAsync(reset: false);
        IsLoadingMore = false;
    }
    
    private async Task LoadProductsAsync(bool reset)
    {
        if (reset) { _page = 1; HasMore = true; }
        
        var result = await _api.GetPagedAsync(new ProductQueryParams
        {
            Page = _page,
            PageSize = PageSize,
            Search = SearchText.Length >= 2 ? SearchText : null,
            Category = SelectedCategory == "ทั้งหมด" ? null : SelectedCategory,
            Sort = SortOrder
        });
        
        var vms = result.Items.Select(p => new ProductCardViewModel(p, _cartService, _bus));
        
        if (reset)
            Products = new ObservableCollection<ProductCardViewModel>(vms);
        else
            foreach (var vm in vms) Products.Add(vm);
        
        HasMore = result.HasNextPage;
        _page++;
    }
    
    [RelayCommand]
    private async Task ViewProductAsync(int productId)
    {
        await Shell.Current.GoToAsync($"product-detail?id={productId}");
    }
    
    [RelayCommand]
    private void FilterByCategory(string category)
    {
        SelectedCategory = category;
        _ = LoadProductsAsync(reset: true);
    }
    
    partial void OnSearchTextChanged(string value)
    {
        if (value.Length == 0 || value.Length >= 2)
            _ = LoadProductsAsync(reset: true);
    }
}

public partial class ProductCardViewModel : ObservableObject
{
    private readonly CartService _cart;
    private readonly IEventBus _bus;
    
    public Product Product { get; }
    
    [ObservableProperty] private bool _isAddingToCart;
    [ObservableProperty] private bool _isFavorite;
    
    public ProductCardViewModel(Product product, CartService cart, IEventBus bus)
    {
        Product = product;
        _cart = cart;
        _bus = bus;
    }
    
    [RelayCommand]
    private async Task AddToCartAsync()
    {
        IsAddingToCart = true;
        try
        {
            await _cart.AddItemAsync(Product.Id, 1);
            _bus.Publish(new CartUpdatedEvent(
                await _cart.GetTotalCountAsync(),
                await _cart.GetTotalPriceAsync()));
            
            await Toast.Make("เพิ่มในตะกร้าแล้ว", ToastDuration.Short).Show();
        }
        finally
        {
            IsAddingToCart = false;
        }
    }
    
    [RelayCommand]
    private async Task ToggleFavoriteAsync()
    {
        IsFavorite = !IsFavorite;
        // Save to favorites
    }
}

public enum ProductSortOrder { Newest, PriceLow, PriceHigh, Popular }
```

---

## Step 345: Cart

```csharp
// ============================================
// Cart Service & ViewModel
// ============================================

public class CartService
{
    private readonly ICartRepository _repo;
    private readonly ICurrentUserService _user;
    
    public CartService(ICartRepository repo, ICurrentUserService user)
    {
        _repo = repo;
        _user = user;
    }
    
    public async Task AddItemAsync(int productId, int quantity = 1)
    {
        var cart = await GetOrCreateCartAsync();
        var existing = cart.Items.FirstOrDefault(i => i.ProductId == productId);
        
        if (existing != null)
            existing.Quantity += quantity;
        else
            cart.Items.Add(new CartItem { ProductId = productId, Quantity = quantity });
        
        await _repo.SaveCartAsync(cart);
    }
    
    public async Task RemoveItemAsync(int productId)
    {
        var cart = await GetOrCreateCartAsync();
        var item = cart.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            cart.Items.Remove(item);
            await _repo.SaveCartAsync(cart);
        }
    }
    
    public async Task UpdateQuantityAsync(int productId, int quantity)
    {
        if (quantity <= 0) { await RemoveItemAsync(productId); return; }
        
        var cart = await GetOrCreateCartAsync();
        var item = cart.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            item.Quantity = quantity;
            await _repo.SaveCartAsync(cart);
        }
    }
    
    public async Task<Cart> GetCartAsync()
        => await GetOrCreateCartAsync();
    
    public async Task<int> GetTotalCountAsync()
    {
        var cart = await GetOrCreateCartAsync();
        return cart.Items.Sum(i => i.Quantity);
    }
    
    public async Task<decimal> GetTotalPriceAsync()
    {
        var cart = await GetOrCreateCartAsync();
        return cart.Items.Sum(i => i.Price * i.Quantity);
    }
    
    public async Task ClearAsync()
    {
        var cart = await GetOrCreateCartAsync();
        cart.Items.Clear();
        await _repo.SaveCartAsync(cart);
    }
    
    private async Task<Cart> GetOrCreateCartAsync()
    {
        var cart = await _repo.GetByUserIdAsync(_user.UserId?.ToString() ?? "guest");
        return cart ?? new Cart { UserId = _user.UserId?.ToString() ?? "guest" };
    }
}

public partial class CartViewModel : ObservableObject
{
    private readonly CartService _cart;
    private readonly IEventBus _bus;
    
    [ObservableProperty] private ObservableCollection<CartItemViewModel> _items = new();
    [ObservableProperty] private decimal _subtotal;
    [ObservableProperty] private decimal _shippingFee = 50;
    [ObservableProperty] private decimal _discount;
    [ObservableProperty] private decimal _total;
    [ObservableProperty] private string? _couponCode;
    [ObservableProperty] private bool _isLoading;
    
    public bool IsEmpty => !Items.Any();
    
    public CartViewModel(CartService cart, IEventBus bus)
    {
        _cart = cart;
        _bus = bus;
        
        // React to cart changes
        _bus.Subscribe<CartUpdatedEvent>(this, _ => _ = LoadCartAsync());
    }
    
    [RelayCommand]
    private async Task LoadCartAsync()
    {
        IsLoading = true;
        var cartData = await _cart.GetCartAsync();
        
        Items = new ObservableCollection<CartItemViewModel>(
            cartData.Items.Select(i => new CartItemViewModel(i, _cart, _bus)));
        
        CalculateTotals();
        IsLoading = false;
        OnPropertyChanged(nameof(IsEmpty));
    }
    
    [RelayCommand]
    private async Task ApplyCouponAsync()
    {
        if (string.IsNullOrWhiteSpace(CouponCode)) return;
        
        // Validate coupon
        // var result = await _couponService.ValidateAsync(CouponCode);
        // Discount = result.Amount;
        CalculateTotals();
    }
    
    [RelayCommand]
    private async Task CheckoutAsync()
    {
        if (!Items.Any()) return;
        await Shell.Current.GoToAsync("checkout");
    }
    
    private void CalculateTotals()
    {
        Subtotal = Items.Sum(i => i.LineTotal);
        Total = Subtotal + ShippingFee - Discount;
    }
}

public partial class CartItemViewModel : ObservableObject
{
    private readonly CartService _cart;
    private readonly IEventBus _bus;
    
    [ObservableProperty] private int _quantity;
    
    public CartItem Item { get; }
    public decimal LineTotal => Item.Price * Quantity;
    
    public CartItemViewModel(CartItem item, CartService cart, IEventBus bus)
    {
        Item = item;
        _cart = cart;
        _bus = bus;
        Quantity = item.Quantity;
    }
    
    [RelayCommand]
    private async Task IncreaseAsync()
    {
        await _cart.UpdateQuantityAsync(Item.ProductId, Quantity + 1);
        Quantity++;
        OnPropertyChanged(nameof(LineTotal));
    }
    
    [RelayCommand]
    private async Task DecreaseAsync()
    {
        if (Quantity <= 1) { await RemoveAsync(); return; }
        await _cart.UpdateQuantityAsync(Item.ProductId, Quantity - 1);
        Quantity--;
        OnPropertyChanged(nameof(LineTotal));
    }
    
    [RelayCommand]
    private async Task RemoveAsync()
    {
        await _cart.RemoveItemAsync(Item.ProductId);
        _bus.Publish(new CartItemRemovedEvent(Item.ProductId));
    }
}

public record CartItemRemovedEvent(int ProductId);
```

---

## Step 346: Checkout & Order

```csharp
// ============================================
// Checkout Flow
// ============================================

public partial class CheckoutViewModel : ObservableObject
{
    private readonly CartService _cart;
    private readonly IOrderApiService _orderApi;
    private readonly ICurrentUserService _user;
    
    [ObservableProperty] private CheckoutStep _currentStep = CheckoutStep.Address;
    [ObservableProperty] private AddressInfo _shippingAddress = new();
    [ObservableProperty] private PaymentMethod _selectedPayment = PaymentMethod.CreditCard;
    [ObservableProperty] private List<CartItem> _orderItems = new();
    [ObservableProperty] private decimal _total;
    [ObservableProperty] private bool _isPlacingOrder;
    
    public CheckoutViewModel(CartService cart, IOrderApiService orderApi, ICurrentUserService user)
    {
        _cart = cart;
        _orderApi = orderApi;
        _user = user;
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        var cartData = await _cart.GetCartAsync();
        OrderItems = cartData.Items;
        Total = cartData.Items.Sum(i => i.Price * i.Quantity) + 50; // + shipping
    }
    
    [RelayCommand]
    private void NextStep()
    {
        if (CurrentStep < CheckoutStep.Confirm)
            CurrentStep++;
    }
    
    [RelayCommand]
    private void PrevStep()
    {
        if (CurrentStep > CheckoutStep.Address)
            CurrentStep--;
    }
    
    [RelayCommand]
    private async Task PlaceOrderAsync()
    {
        IsPlacingOrder = true;
        try
        {
            var order = new CreateOrderRequest
            {
                UserId = _user.UserId ?? 0,
                Items = OrderItems.Select(i => new OrderItemDto(i.ProductId, i.Quantity, i.Price)).ToList(),
                ShippingAddress = ShippingAddress,
                PaymentMethod = SelectedPayment,
                Total = Total
            };
            
            var result = await _orderApi.CreateAsync(order);
            await _cart.ClearAsync();
            
            await Shell.Current.GoToAsync($"//orders/order-detail?id={result.OrderId}");
            await Toast.Make("สั่งซื้อสำเร็จ! 🎉", ToastDuration.Long).Show();
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("เกิดข้อผิดพลาด", ex.Message, "ตกลง");
        }
        finally
        {
            IsPlacingOrder = false;
        }
    }
}

public enum CheckoutStep { Address, Payment, Confirm }
public enum PaymentMethod { CreditCard, DebitCard, Promptpay, COD }

public record AddressInfo
{
    public string Name { get; set; } = string.Empty;
    public string Phone { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;
    public string District { get; set; } = string.Empty;
    public string Province { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
}
```

---

## Step 347: Product Detail

```csharp
// ============================================
// Product Detail ViewModel
// ============================================

public partial class ProductDetailViewModel : PageViewModel, IQueryAttributable
{
    private readonly IProductApiService _api;
    private readonly CartService _cart;
    private readonly IEventBus _bus;
    
    [ObservableProperty] private Product? _product;
    [ObservableProperty] private List<Product> _relatedProducts = new();
    [ObservableProperty] private int _quantity = 1;
    [ObservableProperty] private bool _isFavorite;
    [ObservableProperty] private int _selectedImageIndex;
    [ObservableProperty] private string? _selectedVariant;
    
    private int _productId;
    
    public ProductDetailViewModel(IProductApiService api, CartService cart, IEventBus bus)
    {
        _api = api;
        _cart = cart;
        _bus = bus;
    }
    
    public void ApplyQueryAttributes(IDictionary<string, object> query)
    {
        if (query.TryGetValue("id", out var id))
            _productId = int.Parse(id.ToString()!);
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        await ExecuteAsync(async () =>
        {
            Product = await _api.GetByIdAsync(_productId);
            
            if (Product != null)
            {
                RelatedProducts = await _api.GetRelatedAsync(_productId, limit: 4);
                IsFavorite = await CheckFavoriteAsync(_productId);
            }
        });
    }
    
    [RelayCommand]
    private async Task AddToCartAsync()
    {
        if (Product == null) return;
        
        await ExecuteAsync(async () =>
        {
            await _cart.AddItemAsync(Product.Id, Quantity);
            
            _bus.Publish(new CartUpdatedEvent(
                await _cart.GetTotalCountAsync(),
                await _cart.GetTotalPriceAsync()));
            
            await Toast.Make($"เพิ่ม {Product.Name} x{Quantity} ในตะกร้า", ToastDuration.Short).Show();
        });
    }
    
    [RelayCommand]
    private async Task BuyNowAsync()
    {
        await AddToCartAsync();
        await Shell.Current.GoToAsync("//cart");
    }
    
    [RelayCommand]
    private void IncreaseQuantity()
    {
        if (Product != null && Quantity < Product.StockQuantity)
            Quantity++;
    }
    
    [RelayCommand]
    private void DecreaseQuantity()
    {
        if (Quantity > 1) Quantity--;
    }
    
    [RelayCommand]
    private async Task ShareAsync()
    {
        if (Product == null) return;
        await Share.Default.RequestAsync(new ShareTextRequest
        {
            Title = Product.Name,
            Text = $"ดูสินค้า {Product.Name} ราคา {Product.Price:C}\n{Product.ShopUrl}"
        });
    }
    
    [RelayCommand]
    private async Task ToggleFavoriteAsync()
    {
        IsFavorite = !IsFavorite;
        // Save to favorites list
    }
    
    private Task<bool> CheckFavoriteAsync(int productId)
        => Task.FromResult(false); // Stub
}
```

---

## Step 348: Order History

```csharp
// ============================================
// Order History & Tracking
// ============================================

public partial class OrderListViewModel : ObservableObject
{
    private readonly IOrderApiService _api;
    
    [ObservableProperty] private ObservableCollection<OrderSummary> _orders = new();
    [ObservableProperty] private bool _isLoading;
    [ObservableProperty] private OrderStatusFilter _selectedFilter = OrderStatusFilter.All;
    
    public OrderListViewModel(IOrderApiService api) => _api = api;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        var allOrders = await _api.GetMyOrdersAsync();
        
        var filtered = SelectedFilter == OrderStatusFilter.All
            ? allOrders
            : allOrders.Where(o => o.Status.ToString() == SelectedFilter.ToString()).ToList();
        
        Orders = new ObservableCollection<OrderSummary>(filtered);
        IsLoading = false;
    }
    
    [RelayCommand]
    private void FilterBy(OrderStatusFilter filter)
    {
        SelectedFilter = filter;
        _ = LoadAsync();
    }
    
    [RelayCommand]
    private async Task ViewOrderAsync(int orderId)
    {
        await Shell.Current.GoToAsync($"order-detail?id={orderId}");
    }
    
    [RelayCommand]
    private async Task ReorderAsync(int orderId)
    {
        var order = await _api.GetByIdAsync(orderId);
        if (order == null) return;
        
        // Add all items from previous order to cart
        // foreach (var item in order.Items)
        //     await _cart.AddItemAsync(item.ProductId, item.Quantity);
        
        await Shell.Current.GoToAsync("//cart");
    }
}

public enum OrderStatusFilter { All, Pending, Processing, Shipped, Delivered, Cancelled }

public record OrderSummary(
    int Id,
    string OrderNumber,
    DateTime OrderDate,
    OrderStatus Status,
    decimal Total,
    int ItemCount,
    string? TrackingNumber
);
```

---

## Step 349: User Profile

```csharp
// ============================================
// User Profile & Settings
// ============================================

public partial class ProfileViewModel : ObservableObject
{
    private readonly ICurrentUserService _user;
    private readonly AuthService _auth;
    private readonly IUserProfileRepository _profile;
    
    [ObservableProperty] private UserProfile? _userProfile;
    [ObservableProperty] private bool _isDarkMode;
    [ObservableProperty] private bool _notificationsEnabled;
    [ObservableProperty] private string _selectedLanguage = "th";
    [ObservableProperty] private bool _isLoading;
    
    public ProfileViewModel(
        ICurrentUserService user,
        AuthService auth,
        IUserProfileRepository profile)
    {
        _user = user;
        _auth = auth;
        _profile = profile;
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        UserProfile = await _profile.GetCurrentUserProfileAsync();
        IsDarkMode = Preferences.Default.Get("dark_mode", false);
        NotificationsEnabled = Preferences.Default.Get("notifications", true);
        SelectedLanguage = Preferences.Default.Get("language", "th");
        IsLoading = false;
    }
    
    [RelayCommand]
    private async Task UpdateAvatarAsync()
    {
        var photo = await MediaPicker.Default.PickPhotoAsync();
        if (photo == null) return;
        
        // Upload and update profile photo
    }
    
    [RelayCommand]
    private async Task EditProfileAsync()
    {
        await Shell.Current.GoToAsync("edit-profile");
    }
    
    [RelayCommand]
    private async Task ChangePasswordAsync()
    {
        await Shell.Current.GoToAsync("change-password");
    }
    
    partial void OnIsDarkModeChanged(bool value)
    {
        Application.Current!.UserAppTheme = value ? AppTheme.Dark : AppTheme.Light;
        Preferences.Default.Set("dark_mode", value);
    }
    
    partial void OnNotificationsEnabledChanged(bool value)
    {
        Preferences.Default.Set("notifications", value);
    }
    
    [RelayCommand]
    private async Task LogoutAsync()
    {
        var confirmed = await Shell.Current.DisplayAlert(
            "ออกจากระบบ", "คุณต้องการออกจากระบบใช่หรือไม่?", "ออกจากระบบ", "ยกเลิก");
        
        if (!confirmed) return;
        
        await _auth.LogoutAsync();
        await Shell.Current.GoToAsync("//login");
    }
    
    [RelayCommand]
    private async Task DeleteAccountAsync()
    {
        var confirmed = await Shell.Current.DisplayAlert(
            "ลบบัญชี", "การลบบัญชีไม่สามารถย้อนกลับได้ คุณแน่ใจหรือไม่?",
            "ลบบัญชี", "ยกเลิก");
        
        if (!confirmed) return;
        
        // Delete account
    }
}
```

---

## Step 350: App Launch Optimization

```csharp
// ============================================
// App Launch Optimization
// ============================================

public partial class App : Application
{
    private readonly IServiceProvider _services;
    private readonly TokenManager _tokens;
    
    public App(IServiceProvider services, TokenManager tokens)
    {
        _services = services;
        _tokens = tokens;
        InitializeComponent();
    }
    
    protected override Window CreateWindow(IActivationState? activationState)
    {
        var window = base.CreateWindow(activationState);
        window.Page = new SplashPage(); // Show splash immediately
        
        // Determine initial page asynchronously
        _ = InitializeAsync(window);
        
        return window;
    }
    
    private async Task InitializeAsync(Window window)
    {
        // Parallel initialization
        var tasks = new[]
        {
            PreloadDatabaseAsync(),
            CheckAuthStatusAsync()
        };
        
        var results = await Task.WhenAll(tasks);
        var isAuthenticated = results[1];
        
        MainThread.BeginInvokeOnMainThread(() =>
        {
            window.Page = new AppShell();
            
            var startRoute = isAuthenticated ? "//products" : "//login";
            _ = Shell.Current.GoToAsync(startRoute);
        });
    }
    
    private async Task<bool> PreloadDatabaseAsync()
    {
        var db = _services.GetRequiredService<SQLiteAsyncConnection>();
        await db.ExecuteAsync("PRAGMA wal_checkpoint"); // Flush WAL
        return true;
    }
    
    private async Task<bool> CheckAuthStatusAsync()
    {
        return await _tokens.IsAuthenticatedAsync();
    }
}

// Splash Page
public class SplashPage : ContentPage
{
    public SplashPage()
    {
        BackgroundColor = Color.FromArgb("#2196F3");
        Content = new VerticalStackLayout
        {
            VerticalOptions = LayoutOptions.Center,
            HorizontalOptions = LayoutOptions.Center,
            Children =
            {
                new Image { Source = "app_logo.png", WidthRequest = 120 },
                new ActivityIndicator { IsRunning = true, Color = Colors.White },
                new Label { Text = "ShopMAUI", TextColor = Colors.White, FontSize = 24 }
            }
        };
    }
}
```

---

## สรุป Part 35

ใน Part 35 เราได้สร้าง E-Commerce App ครบวงจร:

1. **Project Architecture** - Feature-based structure
2. **MauiProgram.cs** - Complete DI setup
3. **AppShell** - Navigation with auth guards
4. **Product List** - Search, filter, paginate
5. **Cart Service** - Add/remove/update items
6. **Cart ViewModel** - Coupon, totals
7. **Checkout Flow** - Multi-step checkout
8. **Product Detail** - Gallery, variants, related
9. **Order History** - Status filter, reorder
10. **Profile** - Settings, auth, dark mode
11. **App Launch** - Parallel init, splash screen

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 35 | Steps 341-350*
