# Part 63: Production App Foundation – Food Delivery
## Steps 621-630: Complete Production App Architecture

---

## Step 621: App Overview & Solution Structure

```
MyFoodDelivery/
├── src/
│   ├── Domain/                     # Entities, Value Objects, Events
│   │   ├── Orders/
│   │   │   ├── Order.cs
│   │   │   ├── OrderLine.cs
│   │   │   └── Events/
│   │   ├── Restaurants/
│   │   │   ├── Restaurant.cs
│   │   │   └── MenuItem.cs
│   │   ├── Riders/
│   │   │   └── Rider.cs
│   │   └── Shared/
│   │       ├── Money.cs
│   │       └── Address.cs
│   │
│   ├── Application/                # Commands, Queries, Handlers
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   ├── TrackOrder/
│   │   │   └── CancelOrder/
│   │   └── Restaurants/
│   │       ├── SearchRestaurants/
│   │       └── GetMenu/
│   │
│   ├── Infrastructure/             # SQLite, HTTP, SignalR
│   │   ├── Persistence/
│   │   ├── Http/
│   │   └── RealTime/
│   │
│   └── UI.MAUI/                    # Pages, ViewModels
│       ├── Features/
│       │   ├── Home/
│       │   ├── Restaurant/
│       │   ├── Cart/
│       │   ├── Checkout/
│       │   ├── OrderTracking/
│       │   └── Profile/
│       └── Shared/
│           ├── Controls/
│           └── Services/
│
└── tests/
    ├── Domain.Tests/
    ├── Application.Tests/
    └── UI.Tests/
```

---

## Step 622: Domain Model

```csharp
// ============================================
// Food Delivery Domain Model
// ============================================

// Value Objects
public record Money(decimal Amount, string Currency = "THB")
{
    public static Money Zero => new(0);
    public Money Add(Money other) => new(Amount + other.Amount);
    public Money Multiply(int factor) => new(Amount * factor);
    public override string ToString() => $"฿{Amount:N0}";
}

public record Address(
    string Street, string District, string Province,
    string PostalCode, double Latitude, double Longitude);

public record Location(double Lat, double Lng)
{
    public double DistanceTo(Location other)
    {
        const double R = 6371; // km
        var dLat = (other.Lat - Lat) * Math.PI / 180;
        var dLng = (other.Lng - Lng) * Math.PI / 180;
        var a = Math.Sin(dLat/2) * Math.Sin(dLat/2) +
            Math.Cos(Lat * Math.PI / 180) * Math.Cos(other.Lat * Math.PI / 180) *
            Math.Sin(dLng/2) * Math.Sin(dLng/2);
        return R * 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
    }
}

// Restaurant aggregate
public class Restaurant : AggregateRoot
{
    private readonly List<MenuItem> _menu = new();
    
    public string Name { get; private set; } = string.Empty;
    public string Category { get; private set; } = string.Empty;
    public Address Address { get; private set; } = null!;
    public Location Location { get; private set; } = null!;
    public double Rating { get; private set; }
    public int DeliveryTimeMinutes { get; private set; }
    public Money DeliveryFee { get; private set; } = Money.Zero;
    public Money MinimumOrder { get; private set; } = Money.Zero;
    public bool IsOpen { get; private set; }
    public string? ImageUrl { get; private set; }
    
    public IReadOnlyList<MenuItem> Menu => _menu.AsReadOnly();
    
    public static Restaurant Create(
        string name, string category, Address address,
        Location location, int deliveryMinutes,
        Money deliveryFee, Money minimumOrder)
    {
        var restaurant = new Restaurant
        {
            Name = name, Category = category,
            Address = address, Location = location,
            DeliveryTimeMinutes = deliveryMinutes,
            DeliveryFee = deliveryFee, MinimumOrder = minimumOrder,
            IsOpen = true, Rating = 0
        };
        restaurant.AddDomainEvent(new RestaurantRegisteredEvent(
            Guid.NewGuid(), DateTime.UtcNow, restaurant.Id, name));
        return restaurant;
    }
    
    public void AddMenuItem(string name, string description,
        Money price, string category, string? imageUrl = null)
    {
        var item = new MenuItem(Guid.NewGuid(), Id, name, description,
            price, category, imageUrl, true);
        _menu.Add(item);
    }
    
    public void UpdateRating(double newRating)
    {
        if (newRating < 0 || newRating > 5)
            throw new DomainException("INVALID_RATING", "คะแนนต้องอยู่ระหว่าง 0-5");
        Rating = newRating;
    }
    
    public void Open() { IsOpen = true; }
    public void Close() { IsOpen = false; }
}

// Menu item
public class MenuItem
{
    public Guid Id { get; }
    public int RestaurantId { get; }
    public string Name { get; }
    public string Description { get; }
    public Money Price { get; private set; }
    public string Category { get; }
    public string? ImageUrl { get; }
    public bool IsAvailable { get; private set; }
    
    public MenuItem(Guid id, int restaurantId, string name, string description,
        Money price, string category, string? imageUrl, bool isAvailable)
    {
        Id = id; RestaurantId = restaurantId;
        Name = name; Description = description;
        Price = price; Category = category;
        ImageUrl = imageUrl; IsAvailable = isAvailable;
    }
    
    public void UpdatePrice(Money newPrice) => Price = newPrice;
    public void SetAvailable(bool available) => IsAvailable = available;
}
```

---

## Step 623: Order Aggregate

```csharp
// ============================================
// Order Aggregate (Complete)
// ============================================

public class FoodOrder : AggregateRoot
{
    private readonly List<FoodOrderLine> _lines = new();
    
    public int CustomerId { get; private set; }
    public int RestaurantId { get; private set; }
    public string RestaurantName { get; private set; } = string.Empty;
    public int? RiderId { get; private set; }
    public FoodOrderStatus Status { get; private set; }
    public Address DeliveryAddress { get; private set; } = null!;
    public Money Subtotal { get; private set; } = Money.Zero;
    public Money DeliveryFee { get; private set; } = Money.Zero;
    public Money Total { get; private set; } = Money.Zero;
    public string PaymentMethod { get; private set; } = string.Empty;
    public string? SpecialInstructions { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? AcceptedAt { get; private set; }
    public DateTime? PickedUpAt { get; private set; }
    public DateTime? DeliveredAt { get; private set; }
    public DateTime? CancelledAt { get; private set; }
    public string? CancellationReason { get; private set; }
    public int EstimatedDeliveryMinutes { get; private set; }
    
    public IReadOnlyList<FoodOrderLine> Lines => _lines.AsReadOnly();
    
    public static FoodOrder Place(
        int customerId, int restaurantId, string restaurantName,
        List<FoodOrderLine> lines, Address deliveryAddress,
        Money deliveryFee, string paymentMethod,
        string? specialInstructions = null)
    {
        if (lines.Count == 0)
            throw new DomainException("EMPTY_ORDER", "กรุณาเพิ่มรายการอาหาร");
        
        var subtotal = lines.Aggregate(Money.Zero, (sum, l) => sum.Add(l.Subtotal));
        
        var order = new FoodOrder
        {
            CustomerId = customerId,
            RestaurantId = restaurantId,
            RestaurantName = restaurantName,
            DeliveryAddress = deliveryAddress,
            DeliveryFee = deliveryFee,
            Subtotal = subtotal,
            Total = subtotal.Add(deliveryFee),
            PaymentMethod = paymentMethod,
            SpecialInstructions = specialInstructions,
            Status = FoodOrderStatus.PendingPayment,
            CreatedAt = DateTime.UtcNow,
            EstimatedDeliveryMinutes = 30
        };
        
        order._lines.AddRange(lines);
        order.AddDomainEvent(new OrderPlacedEvent(
            Guid.NewGuid(), DateTime.UtcNow, order.Id,
            customerId, restaurantId, order.Total.Amount));
        
        return order;
    }
    
    public void ConfirmPayment()
    {
        if (Status != FoodOrderStatus.PendingPayment)
            throw new DomainException("INVALID_STATUS", "ไม่สามารถยืนยันการชำระเงินได้");
        
        Status = FoodOrderStatus.WaitingRestaurant;
        AddDomainEvent(new PaymentConfirmedEvent(
            Guid.NewGuid(), DateTime.UtcNow, Id, Total.Amount));
    }
    
    public void AcceptByRestaurant(int cookingMinutes)
    {
        if (Status != FoodOrderStatus.WaitingRestaurant)
            throw new DomainException("INVALID_STATUS", "ไม่สามารถรับออร์เดอร์ได้");
        
        Status = FoodOrderStatus.Preparing;
        AcceptedAt = DateTime.UtcNow;
        EstimatedDeliveryMinutes = cookingMinutes + 15; // +15 for delivery
        AddDomainEvent(new OrderAcceptedEvent(
            Guid.NewGuid(), DateTime.UtcNow, Id, cookingMinutes));
    }
    
    public void AssignRider(int riderId)
    {
        if (Status != FoodOrderStatus.ReadyForPickup)
            throw new DomainException("INVALID_STATUS", "ออร์เดอร์ยังไม่พร้อมส่ง");
        
        RiderId = riderId;
        Status = FoodOrderStatus.PickingUp;
    }
    
    public void MarkReadyForPickup()
    {
        if (Status != FoodOrderStatus.Preparing)
            throw new DomainException("INVALID_STATUS", "ยังไม่พร้อมส่ง");
        Status = FoodOrderStatus.ReadyForPickup;
    }
    
    public void MarkPickedUp()
    {
        if (Status != FoodOrderStatus.PickingUp)
            throw new DomainException("INVALID_STATUS", "ไรเดอร์ยังไม่รับออร์เดอร์");
        
        Status = FoodOrderStatus.OnTheWay;
        PickedUpAt = DateTime.UtcNow;
        AddDomainEvent(new OrderPickedUpEvent(
            Guid.NewGuid(), DateTime.UtcNow, Id, RiderId!.Value));
    }
    
    public void MarkDelivered()
    {
        if (Status != FoodOrderStatus.OnTheWay)
            throw new DomainException("INVALID_STATUS", "ออร์เดอร์ยังไม่ได้รับ");
        
        Status = FoodOrderStatus.Delivered;
        DeliveredAt = DateTime.UtcNow;
        AddDomainEvent(new OrderDeliveredEvent(
            Guid.NewGuid(), DateTime.UtcNow, Id, CustomerId));
    }
    
    public void Cancel(string reason)
    {
        if (Status is FoodOrderStatus.OnTheWay or FoodOrderStatus.Delivered)
            throw new DomainException("CANNOT_CANCEL", "ไม่สามารถยกเลิกออร์เดอร์ที่กำลังส่งหรือส่งแล้ว");
        
        Status = FoodOrderStatus.Cancelled;
        CancelledAt = DateTime.UtcNow;
        CancellationReason = reason;
        AddDomainEvent(new OrderCancelledEvent(
            Guid.NewGuid(), DateTime.UtcNow, Id, reason));
    }
}

public class FoodOrderLine
{
    public Guid MenuItemId { get; }
    public string ItemName { get; }
    public Money UnitPrice { get; }
    public int Quantity { get; }
    public string? Notes { get; }
    public Money Subtotal => UnitPrice.Multiply(Quantity);
    
    public FoodOrderLine(Guid menuItemId, string name, Money price, int qty, string? notes)
    {
        MenuItemId = menuItemId; ItemName = name;
        UnitPrice = price; Quantity = qty; Notes = notes;
    }
}

public enum FoodOrderStatus
{
    PendingPayment, WaitingRestaurant, Preparing,
    ReadyForPickup, PickingUp, OnTheWay, Delivered, Cancelled
}
```

---

## Step 624: App Navigation Structure

```csharp
// ============================================
// Shell Navigation Structure
// ============================================

// AppShell.xaml
/*
<Shell x:Class="MyFoodDelivery.AppShell"
       xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       Shell.NavBarIsVisible="False">
    
    <!-- Tab Bar -->
    <TabBar>
        <ShellContent Title="หน้าหลัก" Icon="home.png"
                      ContentTemplate="{DataTemplate home:HomePage}" Route="Home" />
        <ShellContent Title="ค้นหา" Icon="search.png"
                      ContentTemplate="{DataTemplate search:SearchPage}" Route="Search" />
        <ShellContent Title="คำสั่งซื้อ" Icon="orders.png"
                      ContentTemplate="{DataTemplate orders:OrdersPage}" Route="Orders" />
        <ShellContent Title="บัญชี" Icon="profile.png"
                      ContentTemplate="{DataTemplate profile:ProfilePage}" Route="Profile" />
    </TabBar>
</Shell>
*/

// Route registration
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        RegisterRoutes();
    }
    
    private static void RegisterRoutes()
    {
        Routing.RegisterRoute("Restaurant", typeof(RestaurantPage));
        Routing.RegisterRoute("Restaurant/Menu", typeof(MenuPage));
        Routing.RegisterRoute("Cart", typeof(CartPage));
        Routing.RegisterRoute("Checkout", typeof(CheckoutPage));
        Routing.RegisterRoute("Payment", typeof(PaymentPage));
        Routing.RegisterRoute("OrderTracking", typeof(OrderTrackingPage));
        Routing.RegisterRoute("OrderSuccess", typeof(OrderSuccessPage));
        Routing.RegisterRoute("OrderHistory", typeof(OrderHistoryPage));
        Routing.RegisterRoute("OrderDetail", typeof(OrderDetailPage));
        Routing.RegisterRoute("Profile/Edit", typeof(EditProfilePage));
        Routing.RegisterRoute("Profile/Addresses", typeof(AddressesPage));
        Routing.RegisterRoute("Profile/Payments", typeof(PaymentMethodsPage));
        Routing.RegisterRoute("Auth/Login", typeof(LoginPage));
        Routing.RegisterRoute("Auth/Register", typeof(RegisterPage));
    }
}
```

---

## Step 625: Home Page ViewModel

```csharp
// ============================================
// Home Page ViewModel
// ============================================

public partial class HomeViewModel : PageViewModel
{
    private readonly IMediator _mediator;
    private readonly ITenantContext _tenant;
    private readonly ILocationService _location;
    
    [ObservableProperty] private List<BannerItem> _banners = new();
    [ObservableProperty] private List<CategoryItem> _categories = new();
    [ObservableProperty] private List<RestaurantSummary> _nearbyRestaurants = new();
    [ObservableProperty] private List<RestaurantSummary> _popularRestaurants = new();
    [ObservableProperty] private bool _isLoading;
    [ObservableProperty] private Location? _userLocation;
    [ObservableProperty] private string _locationText = "กำลังค้นหาตำแหน่ง...";
    [ObservableProperty] private string _greetingText = "สวัสดี";
    [ObservableProperty] private bool _isRefreshing;
    
    public HomeViewModel(
        IMediator mediator, ITenantContext tenant, ILocationService location)
    {
        _mediator = mediator;
        _tenant = tenant;
        _location = location;
    }
    
    public override async Task OnAppearingAsync()
    {
        if (!NearbyRestaurants.Any())
            await LoadAsync();
        
        UpdateGreeting();
    }
    
    private void UpdateGreeting()
    {
        var hour = DateTime.Now.Hour;
        GreetingText = hour switch
        {
            < 12 => "สวัสดีตอนเช้า",
            < 17 => "สวัสดีตอนบ่าย",
            < 20 => "สวัสดีตอนเย็น",
            _ => "สวัสดีตอนกลางคืน"
        };
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        
        try
        {
            // Get location
            UserLocation = await _location.GetCurrentLocationAsync();
            if (UserLocation != null)
                LocationText = await _location.GetAddressAsync(UserLocation);
            
            // Load all home data in parallel
            var (banners, categories, nearby, popular) = await (
                _mediator.Send(new GetBannersQuery()),
                _mediator.Send(new GetCategoriesQuery()),
                _mediator.Send(new GetNearbyRestaurantsQuery(UserLocation, 5)),
                _mediator.Send(new GetPopularRestaurantsQuery(10))
            ).WhenAll();
            
            Banners = banners;
            Categories = categories;
            NearbyRestaurants = nearby;
            PopularRestaurants = popular;
        }
        finally { IsLoading = false; }
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        IsRefreshing = true;
        await LoadAsync();
        IsRefreshing = false;
    }
    
    [RelayCommand]
    private Task NavigateToRestaurantAsync(int restaurantId)
        => Shell.Current.GoToAsync($"Restaurant?id={restaurantId}");
    
    [RelayCommand]
    private Task SearchAsync(string query)
        => Shell.Current.GoToAsync($"Search?q={Uri.EscapeDataString(query)}");
}
```

---

## Step 626: Restaurant Search

```csharp
// ============================================
// Restaurant Search Feature
// ============================================

public record SearchRestaurantsQuery(
    string? Keyword = null,
    string? Category = null,
    Location? UserLocation = null,
    double? MaxDistanceKm = null,
    SortBy Sort = SortBy.Relevance,
    int Page = 1, int PageSize = 20) : IRequest<PagedResult<RestaurantSummary>>;

public enum SortBy { Relevance, Rating, Distance, DeliveryTime, Price }

public record RestaurantSummary(
    int Id, string Name, string Category, double Rating,
    int DeliveryTimeMinutes, Money DeliveryFee, Money MinimumOrder,
    string? ImageUrl, double? DistanceKm, bool IsOpen, int ReviewCount);

public class SearchRestaurantsHandler :
    IRequestHandler<SearchRestaurantsQuery, PagedResult<RestaurantSummary>>
{
    private readonly IRestaurantRepository _repo;
    
    public SearchRestaurantsHandler(IRestaurantRepository repo) => _repo = repo;
    
    public async Task<PagedResult<RestaurantSummary>> Handle(
        SearchRestaurantsQuery query, CancellationToken ct)
    {
        var restaurants = await _repo.SearchAsync(query, ct);
        
        // Calculate distance if location provided
        var summaries = restaurants.Select(r =>
        {
            double? dist = null;
            if (query.UserLocation != null)
            {
                var rLoc = new Location(r.LocationLat, r.LocationLng);
                dist = Math.Round(query.UserLocation.DistanceTo(rLoc), 1);
            }
            
            return new RestaurantSummary(
                r.Id, r.Name, r.Category, r.Rating,
                r.DeliveryTimeMinutes,
                new Money(r.DeliveryFeeAmount),
                new Money(r.MinimumOrderAmount),
                r.ImageUrl, dist, r.IsOpen, r.ReviewCount);
        });
        
        // Sort
        summaries = query.Sort switch
        {
            SortBy.Rating => summaries.OrderByDescending(s => s.Rating),
            SortBy.Distance => summaries.OrderBy(s => s.DistanceKm ?? double.MaxValue),
            SortBy.DeliveryTime => summaries.OrderBy(s => s.DeliveryTimeMinutes),
            SortBy.Price => summaries.OrderBy(s => s.MinimumOrder.Amount),
            _ => summaries.OrderByDescending(s => s.Rating)
        };
        
        var list = summaries.ToList();
        var total = list.Count;
        var paged = list
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .ToList();
        
        return new PagedResult<RestaurantSummary>(paged, total, query.Page, query.PageSize);
    }
}
```

---

## Step 627: Cart Management

```csharp
// ============================================
// Cart ViewModel
// ============================================

public partial class CartViewModel : PageViewModel
{
    private readonly ICartService _cartService;
    private readonly INavigationService _nav;
    private readonly IDialogService _dialogs;
    private readonly PricingService _pricing;
    
    [ObservableProperty] private List<CartLineViewModel> _lines = new();
    [ObservableProperty] private bool _isEmpty;
    [ObservableProperty] private string _restaurantName = string.Empty;
    [ObservableProperty] private Money _subtotal = Money.Zero;
    [ObservableProperty] private Money _deliveryFee = Money.Zero;
    [ObservableProperty] private Money _total = Money.Zero;
    [ObservableProperty] private Money _discount = Money.Zero;
    [ObservableProperty] private string? _couponCode;
    [ObservableProperty] private bool _hasCoupon;
    [ObservableProperty] private bool _freeDelivery;
    [ObservableProperty] private string _deliveryMessage = string.Empty;
    
    public CartViewModel(
        ICartService cartService, INavigationService nav,
        IDialogService dialogs, PricingService pricing)
    {
        _cartService = cartService;
        _nav = nav;
        _dialogs = dialogs;
        _pricing = pricing;
    }
    
    public override async Task OnAppearingAsync()
        => await RefreshCartAsync();
    
    private async Task RefreshCartAsync()
    {
        var cart = await _cartService.GetCurrentCartAsync();
        
        if (cart == null || cart.Lines.Count == 0)
        {
            Lines = new();
            IsEmpty = true;
            return;
        }
        
        RestaurantName = cart.RestaurantName;
        Lines = cart.Lines.Select(l => new CartLineViewModel(l, this)).ToList();
        IsEmpty = false;
        
        await RecalculateAsync();
    }
    
    private async Task RecalculateAsync()
    {
        var cart = await _cartService.GetCurrentCartAsync();
        if (cart == null) return;
        
        Subtotal = new Money(Lines.Sum(l => l.LineTotal.Amount));
        
        // Free delivery if subtotal >= 300
        FreeDelivery = Subtotal.Amount >= 300;
        DeliveryFee = FreeDelivery
            ? Money.Zero
            : new Money(cart.DeliveryFee);
        
        var remaining = 300 - Subtotal.Amount;
        DeliveryMessage = FreeDelivery
            ? "ได้รับฟรีค่าจัดส่ง!"
            : $"เพิ่มอีก ฿{remaining:N0} รับฟรีค่าจัดส่ง";
        
        Discount = HasCoupon
            ? new Money(Subtotal.Amount * 0.1m) // 10% off
            : Money.Zero;
        
        Total = new Money(Subtotal.Amount + DeliveryFee.Amount - Discount.Amount);
    }
    
    [RelayCommand]
    private async Task ApplyCouponAsync()
    {
        var code = await _dialogs.ShowInputAsync("ใส่โค้ดส่วนลด", "โค้ดของคุณ", "เช่น SAVE10");
        if (string.IsNullOrEmpty(code)) return;
        
        // Validate coupon
        var valid = await _cartService.ValidateCouponAsync(code);
        if (valid)
        {
            CouponCode = code;
            HasCoupon = true;
            await RecalculateAsync();
            await _dialogs.ShowToastAsync("ใช้โค้ดส่วนลดสำเร็จ!");
        }
        else
        {
            await _dialogs.ShowAlertAsync("โค้ดไม่ถูกต้อง", "โปรดตรวจสอบโค้ดของคุณอีกครั้ง");
        }
    }
    
    [RelayCommand]
    private Task ProceedToCheckoutAsync()
        => _nav.GoToAsync<CheckoutPage>();
}

public partial class CartLineViewModel : ObservableObject
{
    private readonly CartViewModel _parent;
    
    public string Name { get; }
    public Money UnitPrice { get; }
    [ObservableProperty] private int _quantity;
    public Money LineTotal => new(UnitPrice.Amount * Quantity);
    public string? Notes { get; }
    public string? ImageUrl { get; }
    
    public CartLineViewModel(CartLine line, CartViewModel parent)
    {
        _parent = parent;
        Name = line.ItemName;
        UnitPrice = new Money(line.UnitPrice);
        _quantity = line.Quantity;
        Notes = line.Notes;
        ImageUrl = line.ImageUrl;
    }
    
    [RelayCommand]
    private async Task IncreaseAsync()
    {
        Quantity++;
        OnPropertyChanged(nameof(LineTotal));
        await _parent.RecalculateCommand.ExecuteAsync(null);
    }
    
    [RelayCommand]
    private async Task DecreaseAsync()
    {
        if (Quantity <= 1) return;
        Quantity--;
        OnPropertyChanged(nameof(LineTotal));
        await _parent.RecalculateCommand.ExecuteAsync(null);
    }
}
```

---

## Step 628: Checkout ViewModel

```csharp
// ============================================
// Checkout ViewModel
// ============================================

public partial class CheckoutViewModel : PageViewModel
{
    private readonly IOrderService _orders;
    private readonly ICartService _cart;
    private readonly IPaymentService _payment;
    private readonly INavigationService _nav;
    private readonly IDialogService _dialogs;
    
    [ObservableProperty] private List<SavedAddress> _savedAddresses = new();
    [ObservableProperty] private SavedAddress? _selectedAddress;
    [ObservableProperty] private List<PaymentMethod> _paymentMethods = new();
    [ObservableProperty] private PaymentMethod? _selectedPayment;
    [ObservableProperty] private Money _total;
    [ObservableProperty] private int _estimatedMinutes = 30;
    [ObservableProperty] private string? _specialInstructions;
    [ObservableProperty] private bool _isPlacingOrder;
    
    public bool CanPlaceOrder => SelectedAddress != null && SelectedPayment != null;
    
    public CheckoutViewModel(
        IOrderService orders, ICartService cart, IPaymentService payment,
        INavigationService nav, IDialogService dialogs)
    {
        _orders = orders;
        _cart = cart;
        _payment = payment;
        _nav = nav;
        _dialogs = dialogs;
    }
    
    public override async Task OnAppearingAsync()
    {
        await Task.WhenAll(
            LoadAddressesAsync(),
            LoadPaymentMethodsAsync());
        
        var cart = await _cart.GetCurrentCartAsync();
        if (cart != null) Total = new Money(cart.Total);
    }
    
    private async Task LoadAddressesAsync()
    {
        SavedAddresses = await _orders.GetSavedAddressesAsync();
        SelectedAddress = SavedAddresses.FirstOrDefault(a => a.IsDefault);
    }
    
    private async Task LoadPaymentMethodsAsync()
    {
        PaymentMethods = new List<PaymentMethod>
        {
            new("credit_card", "บัตรเครดิต/เดบิต", "💳", true),
            new("promptpay", "พร้อมเพย์", "📱", true),
            new("cash", "ชำระเงินสด", "💵", true),
            new("wallet", "กระเป๋าเงิน (฿124.50)", "👛", true)
        };
        SelectedPayment = PaymentMethods.First();
    }
    
    [RelayCommand]
    private async Task AddNewAddressAsync()
    {
        await _nav.GoToAsync<AddAddressPage>();
        await LoadAddressesAsync();
    }
    
    [RelayCommand(CanExecute = nameof(CanPlaceOrder))]
    private async Task PlaceOrderAsync()
    {
        if (SelectedAddress == null || SelectedPayment == null) return;
        
        IsPlacingOrder = true;
        
        try
        {
            // Process payment if needed
            if (SelectedPayment.Id != "cash")
            {
                var paymentResult = await _payment.InitiateAsync(
                    Total, SelectedPayment.Id);
                
                if (!paymentResult.RequiresRedirect && !paymentResult.Success)
                {
                    await _dialogs.ShowAlertAsync("ชำระเงินไม่สำเร็จ", paymentResult.Error ?? "ลองใหม่อีกครั้ง");
                    return;
                }
            }
            
            var orderId = await _orders.PlaceOrderAsync(new PlaceOrderCommand(
                AddressId: SelectedAddress.Id,
                PaymentMethod: SelectedPayment.Id,
                SpecialInstructions: SpecialInstructions));
            
            await _cart.ClearAsync();
            await _nav.GoToAsync<OrderSuccessPage>(orderId);
        }
        catch (DomainException ex)
        {
            await _dialogs.ShowAlertAsync("เกิดข้อผิดพลาด", ex.Message);
        }
        finally { IsPlacingOrder = false; }
    }
}

public record SavedAddress(int Id, string Name, string FullAddress, bool IsDefault);
public record PaymentMethod(string Id, string Name, string Icon, bool IsAvailable);
```

---

## Step 629: Order Tracking

```csharp
// ============================================
// Order Tracking (Real-Time)
// ============================================

public partial class OrderTrackingViewModel : PageViewModel, IDisposable
{
    private readonly ISignalRConnection _signalR;
    private readonly IOrderRepository _orders;
    private readonly CompositeDisposable _disposables = new();
    
    [ObservableProperty] private FoodOrderDto? _order;
    [ObservableProperty] private Location? _riderLocation;
    [ObservableProperty] private int _progressPercent;
    [ObservableProperty] private string _statusText = string.Empty;
    [ObservableProperty] private string _etaText = string.Empty;
    [ObservableProperty] private bool _isDelivered;
    
    public OrderTrackingViewModel(
        ISignalRConnection signalR, IOrderRepository orders)
    {
        _signalR = signalR;
        _orders = orders;
    }
    
    public async Task LoadOrderAsync(int orderId)
    {
        var data = await _orders.GetByIdAsync(orderId);
        if (data == null) return;
        
        UpdateOrderState(data);
        
        // Subscribe to real-time updates
        await _signalR.JoinGroupAsync($"order_{orderId}");
        
        _signalR.On<FoodOrderDto>("OrderStatusChanged", order =>
            MainThread.BeginInvokeOnMainThread(() => UpdateOrderState(order)));
        
        _signalR.On<Location>("RiderLocationUpdated", loc =>
            MainThread.BeginInvokeOnMainThread(() => RiderLocation = loc));
    }
    
    private void UpdateOrderState(FoodOrderDto order)
    {
        Order = order;
        ProgressPercent = order.Status switch
        {
            FoodOrderStatus.PendingPayment => 5,
            FoodOrderStatus.WaitingRestaurant => 15,
            FoodOrderStatus.Preparing => 35,
            FoodOrderStatus.ReadyForPickup => 55,
            FoodOrderStatus.PickingUp => 65,
            FoodOrderStatus.OnTheWay => 85,
            FoodOrderStatus.Delivered => 100,
            _ => 0
        };
        
        StatusText = order.Status switch
        {
            FoodOrderStatus.PendingPayment => "รอยืนยันการชำระเงิน",
            FoodOrderStatus.WaitingRestaurant => "รอร้านรับออร์เดอร์",
            FoodOrderStatus.Preparing => "ร้านกำลังเตรียมอาหาร",
            FoodOrderStatus.ReadyForPickup => "อาหารพร้อมรอส่ง",
            FoodOrderStatus.PickingUp => "ไรเดอร์กำลังรับอาหาร",
            FoodOrderStatus.OnTheWay => "ไรเดอร์กำลังเดินทาง",
            FoodOrderStatus.Delivered => "ส่งอาหารเรียบร้อยแล้ว",
            FoodOrderStatus.Cancelled => "ออร์เดอร์ถูกยกเลิก",
            _ => "กำลังดำเนินการ"
        };
        
        if (order.EstimatedDeliveryAt.HasValue && order.Status != FoodOrderStatus.Delivered)
        {
            var remaining = order.EstimatedDeliveryAt.Value - DateTime.Now;
            EtaText = remaining.TotalMinutes > 0
                ? $"ประมาณ {(int)remaining.TotalMinutes} นาที"
                : "เดี๋ยวนี้!";
        }
        
        IsDelivered = order.Status == FoodOrderStatus.Delivered;
    }
    
    public void Dispose()
    {
        _signalR.LeaveGroupAsync($"order_{Order?.Id}");
        _disposables.Dispose();
    }
}
```

---

## Step 630: DI Registration

```csharp
// ============================================
// Dependency Injection Registration
// ============================================

// MauiProgram.cs
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder.UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("NotoSansThai-Regular.ttf", "NotoSans");
                fonts.AddFont("NotoSansThai-Bold.ttf", "NotoSansBold");
            });
        
        // Infrastructure
        builder.Services
            .AddSingleton(_ => new SQLiteConnection(
                Path.Combine(FileSystem.AppDataDirectory, "food_delivery.db")))
            .AddSingleton<IOutbox, SQLiteOutbox>()
            .AddSingleton<EventBus>()
            .AddSingleton<JobQueue>()
            .AddSingleton<DataSyncService>();
        
        // Domain
        builder.Services
            .AddSingleton<PricingService>()
            .AddSingleton<ExperimentService>();
        
        // Application
        builder.Services
            .AddTransient<IMediator, AppMediator>()
            .AddTransient<SearchRestaurantsHandler>()
            .AddTransient<PlaceOrderHandler>()
            .AddTransient<GetOrderTrackingHandler>();
        
        // HTTP
        builder.Services
            .AddHttpClient<IRestaurantApiClient, RestaurantApiClient>(c =>
                c.BaseAddress = new Uri(AppConfig.ApiBaseUrl))
            .AddTransientHttpErrorPolicy(b =>
                b.WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i))));
        
        // Platform
        builder.Services
            .AddSingleton(Connectivity.Current)
            .AddSingleton(Geolocation.Default)
            .AddSingleton(Preferences.Default)
            .AddSingleton<IFeatureFlagService, FeatureFlagService>()
            .AddSingleton<SharedAppState>()
            .AddSingleton<ITenantContext, TenantContext>();
        
        // SignalR
        builder.Services
            .AddSingleton<ISignalRConnection>(sp =>
                new SignalRConnection(AppConfig.HubUrl, sp.GetRequiredService<IAuthTokenService>()));
        
        // Navigation
        builder.Services
            .AddSingleton<INavigationService, MauiNavigationService>()
            .AddSingleton<IDialogService, MauiDialogService>();
        
        // ViewModels
        builder.Services
            .AddTransient<HomeViewModel>()
            .AddTransient<RestaurantViewModel>()
            .AddTransient<CartViewModel>()
            .AddTransient<CheckoutViewModel>()
            .AddTransient<OrderTrackingViewModel>()
            .AddTransient<ProfileViewModel>();
        
        // Pages
        builder.Services
            .AddTransient<HomePage>()
            .AddTransient<RestaurantPage>()
            .AddTransient<CartPage>()
            .AddTransient<CheckoutPage>()
            .AddTransient<OrderTrackingPage>()
            .AddTransient<ProfilePage>();
        
        return builder.Build();
    }
}
```

---

## สรุป Part 63

ใน Part 63 เราได้เรียนรู้:

1. **Solution Structure** - 4-layer food delivery app
2. **Domain Model** - Restaurant, MenuItem, Location
3. **Order Aggregate** - Full order lifecycle states
4. **Shell Navigation** - Tab bar + route registration
5. **Home ViewModel** - Parallel data load, greeting
6. **Restaurant Search** - Query + sort + distance
7. **Cart Management** - Line items, coupon, free delivery
8. **Checkout Flow** - Address, payment, place order
9. **Order Tracking** - Real-time SignalR updates
10. **DI Registration** - MauiProgram full setup

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 63 | Steps 621-630*
