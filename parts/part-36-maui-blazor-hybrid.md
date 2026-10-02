# Part 36: .NET MAUI Blazor Hybrid
## Steps 351-360: Web + Native Combined

---

## Step 351: Blazor Hybrid Overview

```csharp
// ============================================
// .NET MAUI Blazor Hybrid
// ============================================

/*
 * Blazor Hybrid คืออะไร?
 * - รวม Blazor (web UI framework) กับ .NET MAUI
 * - ใช้ Razor Components ใน native app
 * - เข้าถึง native APIs ได้จาก Blazor
 * - ใช้โค้ดร่วมกันระหว่าง web app กับ mobile app
 *
 * เมื่อไหร่ควรใช้?
 * - มี existing Blazor web app
 * - ต้องการ share UI components กับ web
 * - Team มีความเชี่ยวชาญด้าน web
 *
 * Setup:
 * dotnet new maui-blazor -n MyHybridApp
 */

// MauiProgram.cs
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
            });
        
        // Add Blazor WebView
        builder.Services.AddMauiBlazorWebView();
        
#if DEBUG
        builder.Services.AddBlazorWebViewDeveloperTools();
#endif
        
        // Register services accessible from Blazor
        builder.Services.AddSingleton<WeatherService>();
        builder.Services.AddSingleton<INativeService, NativeService>();
        
        return builder.Build();
    }
}
```

---

## Step 352: BlazorWebView in XAML

```xml
<!-- MainPage.xaml -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:blazor="clr-namespace:Microsoft.AspNetCore.Components.WebView.Maui;assembly=Microsoft.AspNetCore.Components.WebView.Maui"
             x:Class="MyHybridApp.MainPage">
    
    <blazor:BlazorWebView HostPage="wwwroot/index.html">
        <blazor:BlazorWebView.RootComponents>
            <blazor:RootComponent Selector="#app" ComponentType="{x:Type local:Routes}" />
        </blazor:BlazorWebView.RootComponents>
    </blazor:BlazorWebView>
    
</ContentPage>
```

```html
<!-- wwwroot/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover" />
    <link rel="stylesheet" href="css/app.css" />
    <link rel="stylesheet" href="MyHybridApp.styles.css" />
</head>
<body>
    <div id="app">กำลังโหลด...</div>
    <div id="blazor-error-ui">
        เกิดข้อผิดพลาด <a href="" class="reload">โหลดใหม่</a>
    </div>
    <script src="_framework/blazor.webview.js" autostart="false"></script>
    <script>
        Blazor.start();
    </script>
</body>
</html>
```

---

## Step 353: Razor Components

```razor
@* Routes.razor *@
<Router AppAssembly="@typeof(Routes).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
        <FocusOnNavigate RouteData="@routeData" Selector="h1" />
    </Found>
    <NotFound>
        <p>ไม่พบหน้าที่ต้องการ</p>
    </NotFound>
</Router>
```

```razor
@* Layouts/MainLayout.razor *@
@inherits LayoutComponentBase

<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>
    <main>
        <div class="top-row">
            <a href="https://docs.microsoft.com/aspnet/core/" target="_blank">เกี่ยวกับ</a>
        </div>
        <article class="content px-4">
            @Body
        </article>
    </main>
</div>
```

```razor
@* Pages/ProductList.razor *@
@page "/products"
@inject IProductApiService ProductApi
@inject NavigationManager Nav

<h1>รายการสินค้า</h1>

@if (_loading)
{
    <p>กำลังโหลด...</p>
}
else if (_products.Any())
{
    <div class="product-grid">
        @foreach (var product in _products)
        {
            <div class="product-card" @onclick="() => ViewProduct(product.Id)">
                <img src="@product.ImageUrl" alt="@product.Name" />
                <h3>@product.Name</h3>
                <p>@product.Price.ToString("C")</p>
                <button @onclick="() => AddToCart(product)" 
                        @onclick:stopPropagation="true">
                    เพิ่มในตะกร้า
                </button>
            </div>
        }
    </div>
}
else
{
    <p>ไม่พบสินค้า</p>
}

@code {
    private List<Product> _products = new();
    private bool _loading = true;
    
    protected override async Task OnInitializedAsync()
    {
        _products = await ProductApi.GetAllAsync();
        _loading = false;
    }
    
    private void ViewProduct(int id)
        => Nav.NavigateTo($"/product/{id}");
    
    private async Task AddToCart(Product product)
    {
        // Add to cart logic
    }
}
```

---

## Step 354: Native API from Blazor

```csharp
// ============================================
// Accessing Native APIs from Blazor
// ============================================

// Interface (shared)
public interface INativeService
{
    Task<string?> TakePhotoAsync();
    Task<Location?> GetLocationAsync();
    Task ShareTextAsync(string text);
    void HapticFeedback(HapticFeedbackType type);
    string DeviceModel { get; }
    string OsVersion { get; }
}

// Implementation (MAUI)
public class NativeService : INativeService
{
    public async Task<string?> TakePhotoAsync()
    {
        var photo = await MediaPicker.Default.CapturePhotoAsync();
        if (photo == null) return null;
        
        using var stream = await photo.OpenReadAsync();
        using var ms = new MemoryStream();
        await stream.CopyToAsync(ms);
        return Convert.ToBase64String(ms.ToArray());
    }
    
    public async Task<Location?> GetLocationAsync()
    {
        var location = await Geolocation.Default.GetLastKnownLocationAsync();
        return location;
    }
    
    public async Task ShareTextAsync(string text)
    {
        await Share.Default.RequestAsync(new ShareTextRequest
        {
            Text = text,
            Title = "แชร์"
        });
    }
    
    public void HapticFeedback(HapticFeedbackType type)
    {
        Microsoft.Maui.Devices.HapticFeedback.Default.Perform(type);
    }
    
    public string DeviceModel => DeviceInfo.Current.Model;
    public string OsVersion => DeviceInfo.Current.VersionString;
}

// Usage in Blazor component
@inject INativeService Native

<button @onclick="TakePhoto">ถ่ายรูป</button>
@if (!string.IsNullOrEmpty(_photoBase64))
{
    <img src="data:image/jpeg;base64,@_photoBase64" />
}

@code {
    private string? _photoBase64;
    
    private async Task TakePhoto()
    {
        _photoBase64 = await Native.TakePhotoAsync();
        StateHasChanged();
    }
}
```

---

## Step 355: JavaScript Interop

```csharp
// ============================================
// JS Interop in Blazor Hybrid
// ============================================

// Calling JS from C#
public class ChartService
{
    private readonly IJSRuntime _js;
    
    public ChartService(IJSRuntime js) => _js = js;
    
    public async Task InitChartAsync(string elementId, ChartConfig config)
    {
        await _js.InvokeVoidAsync("initChart", elementId, config);
    }
    
    public async Task UpdateChartAsync(string elementId, List<double> data)
    {
        await _js.InvokeVoidAsync("updateChart", elementId, data);
    }
    
    public async Task<string> GetBase64ImageAsync(string elementId)
    {
        return await _js.InvokeAsync<string>("getChartImage", elementId);
    }
}

// wwwroot/js/chart.js
/*
window.initChart = (elementId, config) => {
    const ctx = document.getElementById(elementId).getContext('2d');
    window.charts = window.charts || {};
    window.charts[elementId] = new Chart(ctx, config);
};

window.updateChart = (elementId, data) => {
    if (window.charts?.[elementId]) {
        window.charts[elementId].data.datasets[0].data = data;
        window.charts[elementId].update();
    }
};

// Calling C# from JS
window.notifyDotNet = (message) => {
    DotNet.invokeMethodAsync('MyHybridApp', 'OnNotification', message);
};
*/

// Static .NET method callable from JS
public static class BlazorJsBridge
{
    private static Action<string>? _notificationCallback;
    
    public static void SetNotificationCallback(Action<string> callback)
        => _notificationCallback = callback;
    
    [Microsoft.JSInterop.JSInvokable]
    public static void OnNotification(string message)
        => _notificationCallback?.Invoke(message);
}
```

---

## Step 356: State Management in Blazor

```csharp
// ============================================
// State Management
// ============================================

// Singleton state service
public class AppState
{
    private readonly List<Action> _listeners = new();
    
    private UserProfile? _user;
    private int _cartCount;
    private AppTheme _theme = AppTheme.System;
    
    public UserProfile? User
    {
        get => _user;
        set { _user = value; NotifyStateChanged(); }
    }
    
    public int CartCount
    {
        get => _cartCount;
        set { _cartCount = value; NotifyStateChanged(); }
    }
    
    public AppTheme Theme
    {
        get => _theme;
        set
        {
            _theme = value;
            Application.Current!.UserAppTheme = value;
            NotifyStateChanged();
        }
    }
    
    public bool IsAuthenticated => User != null;
    
    public void Subscribe(Action listener) => _listeners.Add(listener);
    public void Unsubscribe(Action listener) => _listeners.Remove(listener);
    
    private void NotifyStateChanged()
    {
        foreach (var listener in _listeners) listener();
    }
}

// Usage in components
@inject AppState AppState
@implements IDisposable

<p>สวัสดี @AppState.User?.Name</p>
<span>ตะกร้า: @AppState.CartCount</span>

@code {
    protected override void OnInitialized()
    {
        AppState.Subscribe(StateHasChanged);
    }
    
    public void Dispose()
    {
        AppState.Unsubscribe(StateHasChanged);
    }
}
```

---

## Step 357: Blazor Forms & Validation

```razor
@* Forms/ProductForm.razor *@
@page "/products/create"
@inject IProductApiService ProductApi
@inject NavigationManager Nav

<EditForm Model="_model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    
    <div class="form-group">
        <label>ชื่อสินค้า</label>
        <InputText class="form-control" @bind-Value="_model.Name" />
        <ValidationMessage For="() => _model.Name" />
    </div>
    
    <div class="form-group">
        <label>ราคา</label>
        <InputNumber class="form-control" @bind-Value="_model.Price" />
        <ValidationMessage For="() => _model.Price" />
    </div>
    
    <div class="form-group">
        <label>หมวดหมู่</label>
        <InputSelect class="form-control" @bind-Value="_model.Category">
            <option value="">เลือกหมวดหมู่</option>
            @foreach (var cat in _categories)
            {
                <option value="@cat">@cat</option>
            }
        </InputSelect>
        <ValidationMessage For="() => _model.Category" />
    </div>
    
    <div class="form-group">
        <label>รูปภาพ</label>
        <InputFile OnChange="HandleFileSelected" accept="image/*" />
        @if (_imagePreview != null)
        {
            <img src="@_imagePreview" class="img-preview" />
        }
    </div>
    
    <button type="submit" disabled="@_saving" class="btn btn-primary">
        @(_saving ? "กำลังบันทึก..." : "บันทึก")
    </button>
    <button type="button" @onclick="Cancel" class="btn btn-secondary">ยกเลิก</button>
</EditForm>

@code {
    private ProductFormModel _model = new();
    private List<string> _categories = new() { "อิเล็กทรอนิกส์", "เสื้อผ้า", "อาหาร" };
    private string? _imagePreview;
    private bool _saving;
    
    private async Task HandleFileSelected(InputFileChangeEventArgs e)
    {
        var file = e.File;
        if (file.Size > 5 * 1024 * 1024) // 5MB limit
        {
            // Show error
            return;
        }
        
        using var stream = file.OpenReadStream(maxAllowedSize: 5 * 1024 * 1024);
        using var ms = new MemoryStream();
        await stream.CopyToAsync(ms);
        
        _imagePreview = $"data:{file.ContentType};base64,{Convert.ToBase64String(ms.ToArray())}";
        _model.ImageData = ms.ToArray();
    }
    
    private async Task HandleSubmit()
    {
        _saving = true;
        try
        {
            await ProductApi.CreateAsync(_model.ToProduct());
            Nav.NavigateTo("/products");
        }
        finally
        {
            _saving = false;
        }
    }
    
    private void Cancel() => Nav.NavigateTo("/products");
}
```

---

## Step 358: Blazor + MAUI Navigation

```csharp
// ============================================
// Navigation between Blazor and MAUI
// ============================================

// Navigate from Blazor to MAUI page
public class BlazorNavigationService
{
    private readonly NavigationManager _nav;
    
    public BlazorNavigationService(NavigationManager nav) => _nav = nav;
    
    // Navigate within Blazor
    public void NavigateTo(string url) => _nav.NavigateTo(url);
    
    // Navigate to native MAUI page
    public static async Task NavigateToNativeAsync(string route)
    {
        await MainThread.InvokeOnMainThreadAsync(async () =>
            await Shell.Current.GoToAsync(route));
    }
    
    // Go back
    public static void GoBack()
    {
        MainThread.BeginInvokeOnMainThread(() => 
            Shell.Current.Navigation.PopAsync());
    }
}

// Open Blazor in modal from MAUI
public partial class NativePage : ContentPage
{
    [RelayCommand]
    private async Task OpenBlazorModalAsync()
    {
        var blazorPage = new ContentPage
        {
            Content = new BlazorWebView
            {
                HostPage = "wwwroot/index.html",
                RootComponents =
                {
                    new RootComponent
                    {
                        Selector = "#app",
                        ComponentType = typeof(ProductPickerComponent)
                    }
                }
            }
        };
        
        await Navigation.PushModalAsync(blazorPage);
    }
}
```

---

## Step 359: Shared Component Library

```razor
@* Shared/ProductCard.razor (usable in both MAUI Blazor and ASP.NET Core Blazor) *@
<div class="product-card @(IsHighlighted ? "highlighted" : "")">
    @if (!string.IsNullOrEmpty(ImageUrl))
    {
        <img src="@ImageUrl" alt="@ProductName" class="product-image" />
    }
    
    <div class="product-info">
        <h3 class="product-name">@ProductName</h3>
        
        @if (!string.IsNullOrEmpty(Category))
        {
            <span class="category-badge">@Category</span>
        }
        
        <div class="price-row">
            @if (OriginalPrice > Price)
            {
                <span class="original-price">@OriginalPrice.ToString("C")</span>
                <span class="discount">-@DiscountPercent%</span>
            }
            <span class="price">@Price.ToString("C")</span>
        </div>
        
        <div class="action-row">
            <button class="btn-cart" @onclick="OnAddToCart">
                🛒 เพิ่มในตะกร้า
            </button>
            
            @if (ShowFavorite)
            {
                <button class="btn-favorite @(IsFavorite ? "active" : "")" 
                        @onclick="OnToggleFavorite">
                    @(IsFavorite ? "❤️" : "🤍")
                </button>
            }
        </div>
    </div>
</div>

@code {
    [Parameter] public string ProductName { get; set; } = string.Empty;
    [Parameter] public string? ImageUrl { get; set; }
    [Parameter] public string? Category { get; set; }
    [Parameter] public decimal Price { get; set; }
    [Parameter] public decimal OriginalPrice { get; set; }
    [Parameter] public bool IsFavorite { get; set; }
    [Parameter] public bool IsHighlighted { get; set; }
    [Parameter] public bool ShowFavorite { get; set; } = true;
    
    [Parameter] public EventCallback OnAddToCart { get; set; }
    [Parameter] public EventCallback OnToggleFavorite { get; set; }
    
    private int DiscountPercent => OriginalPrice > 0
        ? (int)((OriginalPrice - Price) / OriginalPrice * 100)
        : 0;
}
```

---

## Step 360: Testing Blazor Components

```csharp
// ============================================
// Testing Blazor Components with bUnit
// ============================================

// Install: bunit
// using Bunit;

public class ProductCardTests : TestContext
{
    [Fact]
    public void ProductCard_RendersName()
    {
        // Arrange & Act
        var cut = RenderComponent<ProductCard>(parameters => parameters
            .Add(p => p.ProductName, "MacBook Pro")
            .Add(p => p.Price, 59900m));
        
        // Assert
        cut.Find(".product-name").TextContent.Should().Be("MacBook Pro");
        cut.Find(".price").TextContent.Should().Contain("59,900");
    }
    
    [Fact]
    public void ProductCard_ShowsDiscount_WhenOriginalPriceHigher()
    {
        var cut = RenderComponent<ProductCard>(p => p
            .Add(c => c.ProductName, "Product")
            .Add(c => c.Price, 800m)
            .Add(c => c.OriginalPrice, 1000m));
        
        cut.Find(".discount").TextContent.Should().Contain("20%");
        cut.Find(".original-price").Should().NotBeNull();
    }
    
    [Fact]
    public async Task ProductCard_AddToCart_InvokesCallback()
    {
        var callbackInvoked = false;
        
        var cut = RenderComponent<ProductCard>(p => p
            .Add(c => c.ProductName, "Product")
            .Add(c => c.Price, 100m)
            .Add(c => c.OnAddToCart, EventCallback.Factory.Create(this, 
                () => callbackInvoked = true)));
        
        await cut.Find(".btn-cart").ClickAsync(new MouseEventArgs());
        
        callbackInvoked.Should().BeTrue();
    }
}
```

---

## สรุป Part 36

ใน Part 36 เราได้เรียนรู้:

1. **Blazor Hybrid Overview** - When to use
2. **BlazorWebView** - Setup in XAML
3. **Razor Components** - Pages, layouts, routes
4. **Native APIs from Blazor** - INativeService
5. **JavaScript Interop** - JS to C# and back
6. **State Management** - AppState service
7. **Blazor Forms** - Validation, file upload
8. **Navigation** - Blazor ↔ MAUI
9. **Shared Components** - Reusable across platforms
10. **Testing** - bUnit for Blazor components

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 36 | Steps 351-360*
