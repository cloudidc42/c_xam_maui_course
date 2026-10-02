# Part 60: Blazor Hybrid
## Steps 591-600: BlazorWebView, Component Sharing, JS Interop

---

## Step 591: BlazorWebView Setup

```csharp
// ============================================
// Blazor Hybrid Setup
// ============================================

// MauiProgram.cs
builder.Services
    .AddMauiBlazorWebView();

#if DEBUG
builder.Services
    .AddBlazorWebViewDeveloperTools();
#endif

// Register Blazor services
builder.Services
    .AddSingleton<WeatherService>()
    .AddSingleton<ProductBlazorService>();
```

```xml
<!-- MainPage.xaml -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:b="clr-namespace:Microsoft.AspNetCore.Components.WebView.Maui;assembly=Microsoft.AspNetCore.Components.WebView.Maui">
    
    <b:BlazorWebView HostPageUrl="wwwroot/index.html">
        <b:BlazorWebView.RootComponents>
            <b:RootComponent Selector="#app" ComponentType="{x:Type local:App}" />
        </b:BlazorWebView.RootComponents>
    </b:BlazorWebView>
</ContentPage>
```

```html
<!-- wwwroot/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <link href="css/app.css" rel="stylesheet"/>
    <link href="MyApp.styles.css" rel="stylesheet"/>
</head>
<body>
    <div id="app">
        <div class="loading-screen">กำลังโหลด...</div>
    </div>
    <div id="blazor-error-ui">
        เกิดข้อผิดพลาดที่ไม่คาดคิด
        <a href="" class="reload">โหลดใหม่</a>
        <a class="dismiss">❌</a>
    </div>
    <script src="_framework/blazor.webview.js" autostart="false"></script>
    <script>
        Blazor.start();
    </script>
</body>
</html>
```

---

## Step 592: Blazor Component

```csharp
// ============================================
// Shared Blazor Component
// ============================================

// Components/ProductCard.razor
@using MyApp.Shared.Models

<div class="product-card @(IsSelected ? "selected" : "")">
    <div class="product-image">
        @if (!string.IsNullOrEmpty(Product.ImageUrl))
        {
            <img src="@Product.ImageUrl" alt="@Product.Name" loading="lazy" />
        }
        else
        {
            <div class="placeholder-image">
                <span>📦</span>
            </div>
        }
    </div>
    
    <div class="product-info">
        <h3>@Product.Name</h3>
        <span class="category">@Product.Category</span>
        <div class="price-row">
            <span class="price">฿@Product.Price.ToString("N0")</span>
            @if (Product.OriginalPrice.HasValue)
            {
                <span class="original-price">฿@Product.OriginalPrice.Value.ToString("N0")</span>
                <span class="discount-badge">
                    -@((int)((1 - Product.Price / Product.OriginalPrice.Value) * 100))%
                </span>
            }
        </div>
    </div>
    
    <div class="product-actions">
        <button class="btn-cart" @onclick="AddToCart" disabled="@IsAdding">
            @if (IsAdding) { <span class="spinner"></span> }
            else { <span>🛒 เพิ่มในตะกร้า</span> }
        </button>
        <button class="btn-detail" @onclick="ViewDetail">ดูรายละเอียด</button>
    </div>
</div>

@code {
    [Parameter] public ProductDto Product { get; set; } = null!;
    [Parameter] public bool IsSelected { get; set; }
    [Parameter] public EventCallback<ProductDto> OnAddToCart { get; set; }
    [Parameter] public EventCallback<int> OnViewDetail { get; set; }
    
    private bool IsAdding;
    
    private async Task AddToCart()
    {
        IsAdding = true;
        await OnAddToCart.InvokeAsync(Product);
        IsAdding = false;
    }
    
    private Task ViewDetail()
        => OnViewDetail.InvokeAsync(Product.Id);
}
```

---

## Step 593: Blazor Page with Data

```csharp
// ============================================
// Blazor Page (Product List)
// ============================================

// Pages/Products.razor
@page "/products"
@inject IProductRepository ProductRepository
@inject NavigationManager NavManager
@inject IJSRuntime JS

<PageTitle>สินค้าทั้งหมด</PageTitle>

<div class="page-header">
    <h1>สินค้าทั้งหมด</h1>
    <div class="search-bar">
        <input type="text" placeholder="ค้นหาสินค้า..." 
               @bind="SearchText" @bind:event="oninput"
               @onkeyup="OnSearchKeyUp" />
        @if (!string.IsNullOrEmpty(SearchText))
        {
            <button @onclick="() => SearchText = string.Empty">✕</button>
        }
    </div>
</div>

<div class="category-filter">
    @foreach (var cat in Categories)
    {
        <button class="chip @(SelectedCategory == cat ? "active" : "")"
                @onclick="() => SelectCategory(cat)">
            @cat
        </button>
    }
</div>

@if (IsLoading)
{
    <div class="loading-grid">
        @for (int i = 0; i < 6; i++)
        {
            <div class="skeleton-card"></div>
        }
    </div>
}
else if (!FilteredProducts.Any())
{
    <div class="empty-state">
        <span>🔍</span>
        <p>ไม่พบสินค้า "@SearchText"</p>
    </div>
}
else
{
    <div class="product-grid">
        @foreach (var product in FilteredProducts)
        {
            <ProductCard Product="@product"
                         OnAddToCart="HandleAddToCart"
                         OnViewDetail="NavigateToDetail" />
        }
    </div>
    
    @if (HasMoreProducts)
    {
        <button class="btn-load-more" @onclick="LoadMoreAsync">
            @if (IsLoadingMore) { <span>กำลังโหลด...</span> }
            else { <span>โหลดเพิ่มเติม</span> }
        </button>
    }
}

@code {
    private List<ProductDto> Products = new();
    private List<string> Categories = new();
    private string SearchText = string.Empty;
    private string SelectedCategory = "ทั้งหมด";
    private bool IsLoading = true;
    private bool IsLoadingMore;
    private bool HasMoreProducts;
    private int CurrentPage = 1;
    private const int PageSize = 20;
    
    private IEnumerable<ProductDto> FilteredProducts => Products
        .Where(p => (SelectedCategory == "ทั้งหมด" || p.Category == SelectedCategory)
            && (string.IsNullOrEmpty(SearchText) ||
                p.Name.Contains(SearchText, StringComparison.OrdinalIgnoreCase)));
    
    protected override async Task OnInitializedAsync()
    {
        await LoadProductsAsync();
        Categories = new[] { "ทั้งหมด" }.Concat(
            Products.Select(p => p.Category).Distinct().Order()).ToList();
    }
    
    private async Task LoadProductsAsync()
    {
        IsLoading = true;
        var result = await ProductRepository.GetPageAsync(1, PageSize);
        Products = result.Items;
        HasMoreProducts = result.TotalCount > PageSize;
        IsLoading = false;
    }
    
    private async Task LoadMoreAsync()
    {
        IsLoadingMore = true;
        CurrentPage++;
        var result = await ProductRepository.GetPageAsync(CurrentPage, PageSize);
        Products.AddRange(result.Items);
        HasMoreProducts = Products.Count < result.TotalCount;
        IsLoadingMore = false;
    }
    
    private void SelectCategory(string category)
        => SelectedCategory = category;
    
    private void OnSearchKeyUp(KeyboardEventArgs e) { }
    
    private async Task HandleAddToCart(ProductDto product)
    {
        await JS.InvokeVoidAsync("showToast", $"เพิ่ม {product.Name} ในตะกร้าแล้ว");
    }
    
    private void NavigateToDetail(int id)
        => NavManager.NavigateTo($"/products/{id}");
}
```

---

## Step 594: JS Interop

```csharp
// ============================================
// JavaScript Interop
// ============================================

// wwwroot/js/app.js
window.showToast = (message) => {
    const toast = document.createElement('div');
    toast.className = 'toast-notification';
    toast.textContent = message;
    document.body.appendChild(toast);
    
    setTimeout(() => toast.classList.add('show'), 100);
    setTimeout(() => {
        toast.classList.remove('show');
        setTimeout(() => document.body.removeChild(toast), 300);
    }, 2000);
};

window.scrollToTop = () => window.scrollTo({ top: 0, behavior: 'smooth' });

window.copyToClipboard = async (text) => {
    await navigator.clipboard.writeText(text);
    return true;
};

window.getScrollPosition = () => window.scrollY;

window.setTheme = (theme) => {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
};

// Blazor JS service wrapper
public class JsInteropService
{
    private readonly IJSRuntime _js;
    
    public JsInteropService(IJSRuntime js) => _js = js;
    
    public ValueTask ShowToastAsync(string message)
        => _js.InvokeVoidAsync("showToast", message);
    
    public ValueTask ScrollToTopAsync()
        => _js.InvokeVoidAsync("scrollToTop");
    
    public ValueTask<bool> CopyToClipboardAsync(string text)
        => _js.InvokeAsync<bool>("copyToClipboard", text);
    
    public ValueTask<int> GetScrollPositionAsync()
        => _js.InvokeAsync<int>("getScrollPosition");
    
    public ValueTask SetThemeAsync(string theme)
        => _js.InvokeVoidAsync("setTheme", theme);
}
```

---

## Step 595: MAUI ↔ Blazor Communication

```csharp
// ============================================
// MAUI ↔ Blazor State Sharing
// ============================================

// Shared service accessible from both MAUI and Blazor
public class SharedAppState
{
    public event Action? StateChanged;
    
    private AppTheme _theme = AppTheme.Light;
    private UserInfo? _currentUser;
    private int _cartItemCount;
    
    public AppTheme Theme
    {
        get => _theme;
        set { _theme = value; StateChanged?.Invoke(); }
    }
    
    public UserInfo? CurrentUser
    {
        get => _currentUser;
        set { _currentUser = value; StateChanged?.Invoke(); }
    }
    
    public int CartItemCount
    {
        get => _cartItemCount;
        set { _cartItemCount = value; StateChanged?.Invoke(); }
    }
    
    public bool IsAuthenticated => CurrentUser != null;
}

public record UserInfo(int Id, string Name, string Email, string Role);

// MAUI ViewModel using SharedAppState
public partial class MainViewModel : ObservableObject
{
    private readonly SharedAppState _state;
    
    [ObservableProperty] private string _welcomeText = "ยินดีต้อนรับ";
    [ObservableProperty] private int _cartBadge;
    
    public MainViewModel(SharedAppState state)
    {
        _state = state;
        _state.StateChanged += OnStateChanged;
        UpdateFromState();
    }
    
    private void OnStateChanged()
        => MainThread.BeginInvokeOnMainThread(UpdateFromState);
    
    private void UpdateFromState()
    {
        WelcomeText = _state.IsAuthenticated
            ? $"สวัสดี {_state.CurrentUser!.Name}"
            : "ยินดีต้อนรับ";
        CartBadge = _state.CartItemCount;
    }
}

// Blazor component using SharedAppState
// Components/Header.razor
@inject SharedAppState State
@implements IDisposable

<header class="app-header @(State.Theme == AppTheme.Dark ? "dark" : "")">
    <div class="brand">MyShop</div>
    <nav>
        <a href="/products">สินค้า</a>
        <a href="/categories">หมวดหมู่</a>
        @if (State.IsAuthenticated)
        {
            <a href="/orders">คำสั่งซื้อ</a>
        }
    </nav>
    <div class="header-actions">
        <a href="/cart" class="cart-icon">
            🛒 @if (State.CartItemCount > 0) {
                <span class="badge">@State.CartItemCount</span>
            }
        </a>
        @if (State.IsAuthenticated)
        {
            <span>@State.CurrentUser!.Name</span>
        }
        else
        {
            <a href="/login">เข้าสู่ระบบ</a>
        }
    </div>
</header>

@code {
    protected override void OnInitialized()
        => State.StateChanged += OnStateChanged;
    
    private void OnStateChanged()
        => InvokeAsync(StateHasChanged);
    
    public void Dispose()
        => State.StateChanged -= OnStateChanged;
}
```

---

## Step 596: Blazor Charts

```csharp
// ============================================
// Blazor Chart Components
// ============================================

// Components/Charts/LineChart.razor
@using System.Text.Json

<div class="chart-container" style="height: @Height">
    <canvas @ref="_canvas"></canvas>
</div>

@code {
    private ElementReference _canvas;
    
    [Parameter] public string Height { get; set; } = "300px";
    [Parameter] public List<DataPoint> Data { get; set; } = new();
    [Parameter] public string Label { get; set; } = "ข้อมูล";
    [Parameter] public string Color { get; set; } = "#007AFF";
    
    [Inject] private IJSRuntime JS { get; set; } = null!;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            await RenderChartAsync();
    }
    
    protected override async Task OnParametersSetAsync()
    {
        if (_canvas.Id != null)
            await UpdateChartAsync();
    }
    
    private async Task RenderChartAsync()
    {
        var config = new
        {
            type = "line",
            data = new
            {
                labels = Data.Select(d => d.Label).ToArray(),
                datasets = new[]
                {
                    new
                    {
                        label = Label,
                        data = Data.Select(d => d.Value).ToArray(),
                        borderColor = Color,
                        backgroundColor = Color + "20",
                        fill = true,
                        tension = 0.4
                    }
                }
            },
            options = new
            {
                responsive = true,
                maintainAspectRatio = false,
                plugins = new { legend = new { display = false } }
            }
        };
        
        await JS.InvokeVoidAsync("createChart", _canvas, JsonSerializer.Serialize(config));
    }
    
    private async Task UpdateChartAsync()
    {
        var data = new
        {
            labels = Data.Select(d => d.Label).ToArray(),
            values = Data.Select(d => d.Value).ToArray()
        };
        await JS.InvokeVoidAsync("updateChart", _canvas, JsonSerializer.Serialize(data));
    }
}

public record DataPoint(string Label, double Value);
```

---

## Step 597: Blazor Forms

```csharp
// ============================================
// Blazor Form with Validation
// ============================================

// Pages/Checkout.razor
@page "/checkout"
@inject IOrderService OrderService
@inject NavigationManager NavManager

<h1>สรุปคำสั่งซื้อ</h1>

<EditForm Model="@_model" OnValidSubmit="HandleValidSubmit">
    <DataAnnotationsValidator />
    
    <div class="form-section">
        <h2>ที่อยู่จัดส่ง</h2>
        
        <div class="form-group">
            <label>ชื่อ-นามสกุล</label>
            <InputText @bind-Value="_model.RecipientName" class="form-control" />
            <ValidationMessage For="@(() => _model.RecipientName)" />
        </div>
        
        <div class="form-group">
            <label>เบอร์โทรศัพท์</label>
            <InputText @bind-Value="_model.Phone" class="form-control" />
            <ValidationMessage For="@(() => _model.Phone)" />
        </div>
        
        <div class="form-group">
            <label>ที่อยู่</label>
            <InputTextArea @bind-Value="_model.Address" class="form-control" rows="3" />
            <ValidationMessage For="@(() => _model.Address)" />
        </div>
        
        <div class="form-row">
            <div class="form-group">
                <label>จังหวัด</label>
                <InputSelect @bind-Value="_model.Province" class="form-control">
                    <option value="">-- เลือกจังหวัด --</option>
                    @foreach (var province in _provinces)
                    {
                        <option value="@province">@province</option>
                    }
                </InputSelect>
                <ValidationMessage For="@(() => _model.Province)" />
            </div>
            <div class="form-group">
                <label>รหัสไปรษณีย์</label>
                <InputText @bind-Value="_model.PostalCode" class="form-control" />
                <ValidationMessage For="@(() => _model.PostalCode)" />
            </div>
        </div>
    </div>
    
    <div class="form-section">
        <h2>ช่องทางชำระเงิน</h2>
        
        @foreach (var method in _paymentMethods)
        {
            <label class="payment-option @(_model.PaymentMethod == method.Id ? "selected" : "")">
                <InputRadio @bind-Value="_model.PaymentMethod" Value="@method.Id" />
                <span class="icon">@method.Icon</span>
                <span>@method.Name</span>
            </label>
        }
    </div>
    
    <div class="order-summary">
        <div class="row"><span>ยอดสินค้า</span><span>฿@_subtotal.ToString("N0")</span></div>
        <div class="row"><span>ค่าจัดส่ง</span><span>฿@_shippingFee.ToString("N0")</span></div>
        <div class="row total"><span>รวมทั้งสิ้น</span><span>฿@(_subtotal + _shippingFee).ToString("N0")</span></div>
    </div>
    
    <button type="submit" class="btn-confirm" disabled="@_isSubmitting">
        @if (_isSubmitting)
        {
            <span>กำลังดำเนินการ...</span>
        }
        else
        {
            <span>ยืนยันคำสั่งซื้อ</span>
        }
    </button>
</EditForm>

@code {
    private CheckoutModel _model = new();
    private bool _isSubmitting;
    private decimal _subtotal = 2599;
    private decimal _shippingFee = 50;
    
    private readonly List<string> _provinces = new()
    {
        "กรุงเทพมหานคร", "เชียงใหม่", "ภูเก็ต", "ขอนแก่น", "นครราชสีมา"
    };
    
    private readonly List<(string Id, string Name, string Icon)> _paymentMethods = new()
    {
        ("credit_card", "บัตรเครดิต/เดบิต", "💳"),
        ("promptpay", "พร้อมเพย์", "📱"),
        ("bank_transfer", "โอนเงิน", "🏦"),
        ("cod", "เก็บเงินปลายทาง", "💵")
    };
    
    private async Task HandleValidSubmit()
    {
        _isSubmitting = true;
        try
        {
            var orderId = await OrderService.PlaceOrderAsync(_model);
            NavManager.NavigateTo($"/orders/{orderId}/success");
        }
        catch (Exception)
        {
            _isSubmitting = false;
        }
    }
}

public class CheckoutModel
{
    [Required(ErrorMessage = "กรุณากรอกชื่อ-นามสกุล")]
    public string RecipientName { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณากรอกเบอร์โทรศัพท์")]
    [RegularExpression(@"^0[0-9]{9}$", ErrorMessage = "รูปแบบเบอร์โทรไม่ถูกต้อง")]
    public string Phone { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณากรอกที่อยู่")]
    public string Address { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณาเลือกจังหวัด")]
    public string Province { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณากรอกรหัสไปรษณีย์")]
    [RegularExpression(@"^\d{5}$", ErrorMessage = "รหัสไปรษณีย์ต้องเป็นตัวเลข 5 หลัก")]
    public string PostalCode { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณาเลือกช่องทางชำระเงิน")]
    public string PaymentMethod { get; set; } = string.Empty;
}
```

---

## Step 598: Blazor Layout & CSS

```css
/* wwwroot/css/app.css */

:root {
    --primary: #007AFF;
    --secondary: #5856D6;
    --success: #34C759;
    --danger: #FF3B30;
    --warning: #FF9500;
    --surface: #FFFFFF;
    --surface-elevated: #F2F2F7;
    --text-primary: #1C1C1E;
    --text-secondary: #6C6C70;
    --border: #D1D1D6;
    --radius: 12px;
    --shadow: 0 4px 20px rgba(0,0,0,0.08);
}

[data-theme="dark"] {
    --surface: #1C1C1E;
    --surface-elevated: #2C2C2E;
    --text-primary: #FFFFFF;
    --text-secondary: #8E8E93;
    --border: #3A3A3C;
}

.product-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
    gap: 16px;
    padding: 16px;
}

.product-card {
    background: var(--surface);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    overflow: hidden;
    transition: transform 0.2s;
}

.product-card:hover { transform: translateY(-4px); }
.product-card.selected { outline: 2px solid var(--primary); }

.skeleton-card {
    background: linear-gradient(90deg, 
        var(--surface-elevated) 25%, 
        var(--border) 50%, 
        var(--surface-elevated) 75%);
    background-size: 400% 100%;
    animation: shimmer 1.5s infinite;
    border-radius: var(--radius);
    height: 240px;
}

@keyframes shimmer {
    0% { background-position: 100% 0; }
    100% { background-position: -100% 0; }
}

.toast-notification {
    position: fixed;
    bottom: 80px;
    left: 50%;
    transform: translateX(-50%) translateY(20px);
    background: rgba(0,0,0,0.85);
    color: white;
    padding: 12px 24px;
    border-radius: 24px;
    font-size: 14px;
    opacity: 0;
    transition: all 0.3s ease;
    z-index: 9999;
    white-space: nowrap;
}

.toast-notification.show {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
}
```

---

## Step 599: Blazor + Native Bridge

```csharp
// ============================================
// Blazor ↔ Native MAUI Bridge
// ============================================

// Call native MAUI features from Blazor
public class NativeBridge
{
    public event Func<Task>? TakePhotoRequested;
    public event Func<Task>? ShareRequested;
    public event Func<string, Task>? OpenUrlRequested;
    
    private string? _capturedPhotoPath;
    private TaskCompletionSource<string?>? _photoTcs;
    
    public async Task<string?> TakePhotoAsync()
    {
        _photoTcs = new TaskCompletionSource<string?>();
        TakePhotoRequested?.Invoke();
        return await _photoTcs.Task;
    }
    
    public void PhotoCaptured(string? path)
        => _photoTcs?.SetResult(path);
    
    public Task ShareAsync() 
    { 
        ShareRequested?.Invoke(); 
        return Task.CompletedTask; 
    }
}

// MAUI side listens to bridge
public partial class MainPage : ContentPage
{
    private readonly NativeBridge _bridge;
    
    public MainPage(NativeBridge bridge)
    {
        _bridge = bridge;
        
        _bridge.TakePhotoRequested += async () =>
        {
            try
            {
                var photo = await MediaPicker.Default.CapturePhotoAsync();
                if (photo != null)
                {
                    var path = Path.Combine(FileSystem.CacheDirectory, photo.FileName);
                    using var stream = await photo.OpenReadAsync();
                    using var file = File.Create(path);
                    await stream.CopyToAsync(file);
                    _bridge.PhotoCaptured(path);
                }
                else
                {
                    _bridge.PhotoCaptured(null);
                }
            }
            catch { _bridge.PhotoCaptured(null); }
        };
        
        _bridge.ShareRequested += async () =>
        {
            await Share.Default.RequestAsync(new ShareTextRequest
            {
                Text = "ลองดู MyShop แอปช้อปปิ้งที่ดีที่สุด!",
                Title = "แชร์ MyShop"
            });
        };
        
        InitializeComponent();
    }
}

// Blazor component using bridge
// Components/ProductActions.razor
@inject NativeBridge Bridge

<div class="product-actions">
    <button @onclick="CaptureAndUpload">📷 อัปโหลดรูปภาพ</button>
    <button @onclick="() => Bridge.ShareAsync()">📤 แชร์</button>
</div>

@if (!string.IsNullOrEmpty(_uploadedImagePath))
{
    <img src="@_uploadedImagePath" class="uploaded-preview" />
}

@code {
    private string? _uploadedImagePath;
    
    private async Task CaptureAndUpload()
    {
        var path = await Bridge.TakePhotoAsync();
        if (path != null)
            _uploadedImagePath = $"file://{path}";
    }
}
```

---

## Step 600: Production Blazor Hybrid

```csharp
// ============================================
// Production Checklist: Blazor Hybrid
// ============================================

/*
 * ✅ Blazor Hybrid Production Checklist
 *
 * Performance:
 *  □ Lazy loading Blazor assemblies
 *  □ Virtualize<T> for large lists
 *  □ OnAfterRenderAsync(firstRender) for JS init
 *  □ StateHasChanged() only when needed
 *
 * Security:
 *  □ Sanitize all user inputs
 *  □ CORS not applicable (embedded)
 *  □ No sensitive data in JS scope
 *
 * Platform:
 *  □ BlazorWebViewDeveloperTools only in DEBUG
 *  □ Test on all target platforms
 *  □ Handle app lifecycle (background/foreground)
 *
 * Data:
 *  □ Use DI-injected services (not static)
 *  □ Dispose IDisposable components
 *  □ Error boundaries for graceful failures
 */

// Error boundary component
// Components/AppErrorBoundary.razor
@inherits ErrorBoundaryBase

@if (CurrentException != null)
{
    <div class="error-boundary">
        <div class="error-icon">⚠️</div>
        <h2>เกิดข้อผิดพลาด</h2>
        <p>ขอโทษ เกิดข้อผิดพลาดที่ไม่คาดคิด</p>
        <button @onclick="Recover">ลองใหม่อีกครั้ง</button>
        
        @if (ShowDetails)
        {
            <details>
                <summary>รายละเอียด</summary>
                <pre>@CurrentException.Message</pre>
            </details>
        }
    </div>
}
else
{
    @ChildContent
}

@code {
    [Parameter] public bool ShowDetails { get; set; }
    [Parameter] public RenderFragment? ChildContent { get; set; }
    
    protected override Task OnErrorAsync(Exception exception)
    {
        System.Diagnostics.Debug.WriteLine($"Blazor error: {exception}");
        return Task.CompletedTask;
    }
}

// App.razor wrapping with error boundary
// App.razor
<AppErrorBoundary>
    <Router AppAssembly="@typeof(App).Assembly">
        <Found Context="routeData">
            <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
            <FocusOnNavigate RouteData="@routeData" Selector="h1" />
        </Found>
        <NotFound>
            <PageTitle>ไม่พบหน้านี้</PageTitle>
            <LayoutView Layout="@typeof(MainLayout)">
                <div class="not-found">
                    <h1>404 - ไม่พบหน้าที่ต้องการ</h1>
                    <a href="/">กลับหน้าหลัก</a>
                </div>
            </LayoutView>
        </NotFound>
    </Router>
</AppErrorBoundary>
```

---

## สรุป Part 60

ใน Part 60 เราได้เรียนรู้:

1. **BlazorWebView Setup** - MauiProgram, index.html
2. **Blazor Components** - ProductCard with parameters
3. **Blazor Pages** - Data binding, search, pagination
4. **JS Interop** - Toast, clipboard, theme
5. **MAUI ↔ Blazor State** - SharedAppState
6. **Blazor Charts** - Chart.js integration
7. **Blazor Forms** - EditForm, DataAnnotations
8. **CSS Design System** - Variables, dark mode, animations
9. **Native Bridge** - Photo capture, share from Blazor
10. **Production Checklist** - Error boundary, performance

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 60 | Steps 591-600*
