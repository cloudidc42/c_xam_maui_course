# Part 20: Navigation และ Shell ใน .NET MAUI
## Steps 191-200: การนำทางในแอพ

---

## Step 191: Shell Navigation Overview

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- AppShell.xaml -->
<!-- ============================================ -->
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:MyApp.Views"
       x:Class="MyApp.AppShell"
       Shell.NavBarHasShadow="False">

    <!-- Global Shell resources -->
    <Shell.Resources>
        <Style TargetType="TabBar">
            <Setter Property="Shell.TabBarBackgroundColor" Value="{AppThemeBinding Light=White, Dark=#1C1C1E}" />
            <Setter Property="Shell.TabBarTitleColor" Value="{AppThemeBinding Light=#512BD4, Dark=#AC99EA}" />
            <Setter Property="Shell.TabBarUnselectedColor" Value="Gray" />
        </Style>
        <Style TargetType="Shell">
            <Setter Property="Shell.BackgroundColor" Value="{AppThemeBinding Light=#512BD4, Dark=#1C1C1E}" />
            <Setter Property="Shell.ForegroundColor" Value="White" />
            <Setter Property="Shell.TitleColor" Value="White" />
        </Style>
    </Shell.Resources>

    <!-- Tab Bar Navigation -->
    <TabBar>
        <Tab Title="หน้าหลัก" Icon="home.png">
            <ShellContent Route="home"
                          ContentTemplate="{DataTemplate views:HomePage2}" />
        </Tab>
        <Tab Title="สินค้า" Icon="products.png">
            <ShellContent Route="products"
                          ContentTemplate="{DataTemplate views:ProductsPage3}" />
        </Tab>
        <Tab Title="ตะกร้า" Icon="cart.png">
            <ShellContent Route="cart"
                          ContentTemplate="{DataTemplate views:CartPage}" />
        </Tab>
        <Tab Title="โปรไฟล์" Icon="profile.png">
            <ShellContent Route="profile"
                          ContentTemplate="{DataTemplate views:ProfilePage2}" />
        </Tab>
    </TabBar>

</Shell>
```

---

## Step 192: Registering Routes

```csharp
// ============================================
// Route Registration in AppShell.xaml.cs
// ============================================

public partial class AppShell3 : Shell
{
    public AppShell3()
    {
        InitializeComponent();
        RegisterRoutes();
    }
    
    private static void RegisterRoutes()
    {
        // Detail pages (not in tab bar)
        Routing.RegisterRoute("product-detail", typeof(ProductDetailPage2));
        Routing.RegisterRoute("checkout", typeof(CheckoutPage));
        Routing.RegisterRoute("order-confirmation", typeof(OrderConfirmationPage));
        Routing.RegisterRoute("edit-profile", typeof(EditProfilePage));
        Routing.RegisterRoute("settings", typeof(SettingsPage4));
        Routing.RegisterRoute("login", typeof(LoginPage));
        Routing.RegisterRoute("register", typeof(RegisterPage));
        
        // Nested routes (relative to parent)
        Routing.RegisterRoute("products/filter", typeof(FilterPage));
        Routing.RegisterRoute("cart/address", typeof(AddressPage));
    }
}

// Placeholder classes
public class HomePage2 : ContentPage { }
public class CartPage : ContentPage { }
public class ProfilePage2 : ContentPage { }
public class ProductDetailPage2 : ContentPage { }
public class CheckoutPage : ContentPage { }
public class OrderConfirmationPage : ContentPage { }
public class EditProfilePage : ContentPage { }
public class SettingsPage4 : ContentPage { }
public class LoginPage : ContentPage { }
public class RegisterPage : ContentPage { }
public class FilterPage : ContentPage { }
public class AddressPage : ContentPage { }
```

---

## Step 193: Programmatic Navigation

```csharp
// ============================================
// Navigation Patterns
// ============================================

public partial class NavigationViewModel : ObservableObject
{
    // ----------------------------------------
    // Absolute routes (reset navigation stack)
    // ----------------------------------------
    
    [RelayCommand]
    private Task GoHomeAsync()
        => Shell.Current.GoToAsync("//home");
    
    [RelayCommand]
    private Task GoProductsAsync()
        => Shell.Current.GoToAsync("//products");
    
    // ----------------------------------------
    // Relative routes (push onto stack)
    // ----------------------------------------
    
    [RelayCommand]
    private Task GoToProductDetailAsync(int productId)
        => Shell.Current.GoToAsync($"product-detail?id={productId}");
    
    // With dictionary parameters (for complex objects)
    [RelayCommand]
    private async Task GoToProductDetailWithObjectAsync(ProductEntity product)
    {
        await Shell.Current.GoToAsync("product-detail", new Dictionary<string, object>
        {
            { "product", product }
        });
    }
    
    // ----------------------------------------
    // Back navigation
    // ----------------------------------------
    
    [RelayCommand]
    private Task GoBackAsync()
        => Shell.Current.GoToAsync("..");
    
    [RelayCommand]
    private Task GoBack2LevelsAsync()
        => Shell.Current.GoToAsync("../..");
    
    // Back with result (using messaging)
    [RelayCommand]
    private async Task GoBackWithResultAsync(string result)
    {
        WeakReferenceMessenger.Default.Send(new ValueChangedMessage<string>(result));
        await Shell.Current.GoToAsync("..");
    }
    
    // ----------------------------------------
    // Navigate to Login (replace stack)
    // ----------------------------------------
    
    [RelayCommand]
    private Task GoToLoginAsync()
        => Shell.Current.GoToAsync("//login");
    
    // Navigate and clear history
    [RelayCommand]
    private async Task GoToHomeAfterLoginAsync()
    {
        // Reset to main tab bar
        await Shell.Current.GoToAsync("//home");
    }
}
```

---

## Step 194: Passing Parameters

```csharp
// ============================================
// QueryProperty - รับ parameters
// ============================================

// Simple type parameter
[QueryProperty(nameof(ProductId), "id")]
public partial class ProductDetailViewModel2 : ObservableObject
{
    private readonly DatabaseService _db;
    
    [ObservableProperty]
    private int _productId;
    
    [ObservableProperty]
    private ProductEntity? _product;
    
    [ObservableProperty]
    private bool _isLoading;
    
    public ProductDetailViewModel2(DatabaseService db) => _db = db;
    
    partial void OnProductIdChanged(int value)
    {
        if (value > 0)
            _ = LoadProductAsync(value);
    }
    
    private async Task LoadProductAsync(int id)
    {
        IsLoading = true;
        Product = await _db.GetProductByIdAsync(id);
        IsLoading = false;
    }
    
    [RelayCommand]
    private async Task AddToCartAsync()
    {
        if (Product == null) return;
        
        WeakReferenceMessenger.Default.Send(new ProductAddedMessage(
            new ProductItem(Product.Id, Product.Name, Product.Category ?? "", Product.Price, Product.Stock)));
        
        await Shell.Current.GoToAsync("..");
    }
}

// Object parameter
[QueryProperty(nameof(Order), "order")]
public partial class OrderDetailViewModel : ObservableObject
{
    [ObservableProperty]
    private OrderEntity? _order;
    
    partial void OnOrderChanged(OrderEntity? value)
    {
        // Loaded
    }
}

// ============================================
// Multiple parameters
// ============================================

[QueryProperty(nameof(FilterCategory), "category")]
[QueryProperty(nameof(MaxPrice), "maxprice")]
[QueryProperty(nameof(MinPrice), "minprice")]
public partial class FilteredProductsViewModel : ObservableObject
{
    [ObservableProperty]
    private string _filterCategory = string.Empty;
    
    [ObservableProperty]
    private decimal _maxPrice = 100000;
    
    [ObservableProperty]
    private decimal _minPrice = 0;
    
    partial void OnFilterCategoryChanged(string value) => LoadProductsAsync().FireAndForget();
    partial void OnMaxPriceChanged(decimal value) => LoadProductsAsync().FireAndForget();
    
    private async Task LoadProductsAsync()
    {
        await Task.Delay(100);
        // Filter logic here
    }
}
```

---

## Step 195: Modal Navigation

```csharp
// ============================================
// Modal Pages (Push on Modal Stack)
// ============================================

public partial class ModalNavigationViewModel : ObservableObject
{
    // Push modal
    [RelayCommand]
    private async Task ShowModalAsync()
    {
        var page = new FilterPage2();
        await Application.Current!.MainPage!.Navigation.PushModalAsync(page);
    }
    
    // Pop modal
    [RelayCommand]
    private async Task DismissModalAsync()
    {
        await Application.Current!.MainPage!.Navigation.PopModalAsync();
    }
    
    // Pop all modals
    [RelayCommand]
    private async Task DismissAllModalsAsync()
    {
        while (Application.Current!.MainPage!.Navigation.ModalStack.Count > 0)
            await Application.Current!.MainPage!.Navigation.PopModalAsync();
    }
    
    // Show as bottom sheet (using Shell)
    [RelayCommand]
    private async Task ShowBottomSheetAsync()
    {
        await Shell.Current.GoToAsync("filter", animate: true);
    }
}

// ============================================
// NavigationPage (legacy but still useful)
// ============================================

public class AppWithNavigation : Application
{
    public AppWithNavigation()
    {
        var mainPage = new NavigationPage(new HomePage2())
        {
            BarBackgroundColor = Color.FromArgb("#512BD4"),
            BarTextColor = Colors.White
        };
        MainPage = mainPage;
    }
}

// Navigation from code-behind
public class SomePage : ContentPage
{
    private async void OnButtonClicked(object sender, EventArgs e)
    {
        await Navigation.PushAsync(new ProductDetailPage2());
    }
    
    private async void OnBackButtonClicked(object sender, EventArgs e)
    {
        await Navigation.PopAsync();
    }
    
    private async void OnGoRootClicked(object sender, EventArgs e)
    {
        await Navigation.PopToRootAsync();
    }
}

public class FilterPage2 : ContentPage { }
```

---

## Step 196: Navigation Guard / Auth Check

```csharp
// ============================================
// Navigation Guard
// ============================================

public class AuthGuardBehavior : Behavior<Shell>
{
    private readonly SecureStorageService _secure;
    
    public AuthGuardBehavior(SecureStorageService secure) => _secure = secure;
    
    protected override void OnAttachedTo(Shell bindable)
    {
        base.OnAttachedTo(bindable);
        bindable.Navigating += OnShellNavigating;
    }
    
    protected override void OnDetachingFrom(Shell bindable)
    {
        bindable.Navigating -= OnShellNavigating;
        base.OnDetachingFrom(bindable);
    }
    
    private void OnShellNavigating(object? sender, ShellNavigatingEventArgs e)
    {
        // Pages that require authentication
        var protectedRoutes = new[] { "profile", "orders", "checkout" };
        
        bool isProtected = protectedRoutes.Any(r => 
            e.Target.Location.OriginalString.Contains(r));
        
        if (!isProtected) return;
        
        var token = _secure.GetTokenAsync().GetAwaiter().GetResult();
        
        if (string.IsNullOrEmpty(token))
        {
            e.Cancel();
            Shell.Current.GoToAsync("//login").FireAndForget();
        }
    }
}

// ============================================
// Back Button Override
// ============================================

public class UnsavedChangesPage : ContentPage
{
    [ObservableProperty]
    private bool _hasUnsavedChanges;
    
    protected override bool OnBackButtonPressed()
    {
        if (_hasUnsavedChanges)
        {
            Dispatcher.Dispatch(async () =>
            {
                bool confirmed = await DisplayAlert(
                    "ยืนยัน",
                    "มีการแก้ไขที่ยังไม่บันทึก ต้องการออกหรือไม่?",
                    "ออก", "ยกเลิก");
                
                if (confirmed)
                    await Navigation.PopAsync();
            });
            return true; // Prevent default back
        }
        
        return false; // Allow default back
    }
}
```

---

## Step 197: Deep Linking

```csharp
// ============================================
// Deep Linking (App Links / Universal Links)
// ============================================

// AndroidManifest.xml additions:
/*
<activity android:name="MainActivity" ...>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="products" />
        <!-- myapp://products/123 -->
    </intent-filter>
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="www.myapp.com" android:pathPrefix="/products" />
        <!-- https://www.myapp.com/products/123 -->
    </intent-filter>
</activity>
*/

// App.xaml.cs
public partial class App2 : Application
{
    protected override void OnAppLinkRequestReceived(Uri uri)
    {
        base.OnAppLinkRequestReceived(uri);
        HandleDeepLink(uri);
    }
    
    private void HandleDeepLink(Uri uri)
    {
        // myapp://products/123
        // https://myapp.com/products/123
        
        var path = uri.AbsolutePath.TrimStart('/');
        var segments = path.Split('/');
        
        switch (segments[0])
        {
            case "products" when segments.Length > 1 && int.TryParse(segments[1], out int productId):
                Shell.Current.GoToAsync($"//products/product-detail?id={productId}").FireAndForget();
                break;
                
            case "orders" when segments.Length > 1:
                Shell.Current.GoToAsync($"//orders?orderId={segments[1]}").FireAndForget();
                break;
                
            default:
                Shell.Current.GoToAsync("//home").FireAndForget();
                break;
        }
    }
}
```

---

## Step 198: Navigation Animation

```csharp
// ============================================
// Custom Navigation Animation
// ============================================

public class SlideFromBottomPageTransition : IPageTransition
{
    public async Task AnimateAsync(NavigationTransitionState navTransitionState, CancellationToken token)
    {
        var animation = navTransitionState.IsNavigationForward
            ? new Animation(v => navTransitionState.DestinationPage.TranslationY = v, 
                            1000, 0)
            : new Animation(v => navTransitionState.OriginPage.TranslationY = v, 
                            0, 1000);
        
        var tcs = new TaskCompletionSource<bool>();
        animation.Commit(navTransitionState.DestinationPage, "SlideTransition", 
                         rate: 16, length: 350, 
                         easing: Easing.CubicOut,
                         finished: (_, _) => tcs.SetResult(true));
        
        await tcs.Task;
    }
}

// Apply in Shell
/*
<Shell ...>
    <Shell.ItemTemplate>
        <ControlTemplate>
            <!-- custom tab bar -->
        </ControlTemplate>
    </Shell.ItemTemplate>
</Shell>
*/

// ============================================
// Page lifecycle events
// ============================================

public class LifecyclePage : ContentPage
{
    protected override void OnAppearing()
    {
        base.OnAppearing();
        // Page is visible, reload data if needed
        Console.WriteLine("OnAppearing");
    }
    
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        // Page is hidden, save state
        Console.WriteLine("OnDisappearing");
    }
    
    protected override void OnNavigatedTo(NavigatedToEventArgs args)
    {
        base.OnNavigatedTo(args);
        // Came to this page (from navigation)
        Console.WriteLine($"NavigatedTo: {args.PreviousPage?.GetType().Name}");
    }
    
    protected override void OnNavigatedFrom(NavigatedFromEventArgs args)
    {
        base.OnNavigatedFrom(args);
        // Left this page
        Console.WriteLine($"NavigatedFrom: {args.DestinationPage?.GetType().Name}");
    }
}
```

---

## Step 199: Tab Badge

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- Tab Badge - Cart Count -->
<!-- ============================================ -->
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       x:Class="MyApp.AppShell4">
    
    <TabBar>
        <!-- Simple tab -->
        <Tab Title="หน้าหลัก" Icon="home.png" Route="home">
            <ShellContent />
        </Tab>
        
        <!-- Tab with badge -->
        <Tab Title="ตะกร้า" Icon="cart.png" Route="cart"
             Shell.TabBarUnselectedColor="Gray">
            <!-- Badge from code: -->
            <!-- Shell.SetTabBarBadge(tab, "5"); -->
            <ShellContent />
        </Tab>
        
        <!-- Tab with notification dot -->
        <Tab Title="แจ้งเตือน" Icon="bell.png" Route="notifications">
            <ShellContent />
        </Tab>
    </TabBar>
    
</Shell>
```

```csharp
// Set badge programmatically
public partial class CartViewModel2 : ObservableObject
{
    [ObservableProperty]
    private int _cartCount;
    
    partial void OnCartCountChanged(int value)
    {
        // Update tab badge
        var shell = Shell.Current;
        if (shell?.CurrentItem is Tab tab)
        {
            Shell.SetTabBarBadge(tab, value > 0 ? value.ToString() : null);
        }
    }
}
```

---

## Step 200: Complete Navigation App

```csharp
// ============================================
// Navigation Service (Interface)
// ============================================

public interface INavService
{
    Task GoToAsync(string route);
    Task GoToAsync(string route, IDictionary<string, object> parameters);
    Task GoBackAsync();
    Task GoToRootAsync(string route);
    Task ShowAlertAsync(string title, string message, string cancel = "ตกลง");
    Task<bool> ShowConfirmAsync(string title, string message, string accept = "ใช่", string cancel = "ไม่");
    Task<string?> ShowActionSheetAsync(string title, string cancel, params string[] buttons);
}

public class NavService : INavService
{
    public Task GoToAsync(string route)
        => Shell.Current.GoToAsync(route);
    
    public Task GoToAsync(string route, IDictionary<string, object> parameters)
        => Shell.Current.GoToAsync(route, parameters);
    
    public Task GoBackAsync()
        => Shell.Current.GoToAsync("..");
    
    public Task GoToRootAsync(string route)
        => Shell.Current.GoToAsync("//" + route.TrimStart('/'));
    
    public Task ShowAlertAsync(string title, string message, string cancel = "ตกลง")
        => Shell.Current.DisplayAlert(title, message, cancel);
    
    public Task<bool> ShowConfirmAsync(string title, string message, 
        string accept = "ใช่", string cancel = "ไม่")
        => Shell.Current.DisplayAlert(title, message, accept, cancel);
    
    public Task<string?> ShowActionSheetAsync(string title, string cancel, params string[] buttons)
        => Shell.Current.DisplayActionSheet(title, cancel, null, buttons);
}

// ============================================
// Using in ViewModel
// ============================================

public partial class ProductsViewModel4 : ObservableObject
{
    private readonly INavService _nav;
    private readonly DatabaseService _db;
    
    [ObservableProperty]
    private List<ProductEntity> _products = new();
    
    public ProductsViewModel4(INavService nav, DatabaseService db)
    {
        _nav = nav;
        _db = db;
    }
    
    [RelayCommand]
    private async Task ViewProductAsync(ProductEntity product)
    {
        await _nav.GoToAsync("product-detail", new Dictionary<string, object>
        {
            { "product", product }
        });
    }
    
    [RelayCommand]
    private async Task DeleteProductAsync(ProductEntity product)
    {
        bool confirmed = await _nav.ShowConfirmAsync(
            "ยืนยันการลบ",
            $"ต้องการลบสินค้า '{product.Name}' หรือไม่?");
        
        if (confirmed)
        {
            await _db.DeleteProductAsync(product.Id);
            Products = Products.Where(p => p.Id != product.Id).ToList();
        }
    }
    
    [RelayCommand]
    private async Task SortProductsAsync()
    {
        var action = await _nav.ShowActionSheetAsync(
            "เรียงลำดับ",
            "ยกเลิก",
            "ชื่อ A-Z", "ราคาน้อย→มาก", "ราคามาก→น้อย", "ใหม่ล่าสุด");
        
        Products = action switch
        {
            "ชื่อ A-Z" => Products.OrderBy(p => p.Name).ToList(),
            "ราคาน้อย→มาก" => Products.OrderBy(p => p.Price).ToList(),
            "ราคามาก→น้อย" => Products.OrderByDescending(p => p.Price).ToList(),
            "ใหม่ล่าสุด" => Products.OrderByDescending(p => p.CreatedAt).ToList(),
            _ => Products
        };
    }
}
```

---

## สรุป Part 20

ใน Part 20 เราได้เรียนรู้:

1. **Shell Overview** - TabBar, FlyoutItem, ShellContent
2. **Route Registration** - Routing.RegisterRoute
3. **Programmatic Navigation** - GoToAsync absolute/relative
4. **Passing Parameters** - URL params, Dictionary, QueryProperty
5. **Modal Navigation** - PushModalAsync, PopModalAsync
6. **Navigation Guard** - Auth check ก่อน navigate
7. **Back Button Override** - OnBackButtonPressed
8. **Deep Linking** - App Links, URL schemes
9. **Page Lifecycle** - OnAppearing, OnNavigatedTo, etc.
10. **Navigation Service** - Interface สำหรับ testable navigation

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 20 | Steps 191-200*
