# Part 59: Advanced MVVM Patterns
## Steps 581-590: Navigation, Dialogs, Validation, MVVM Toolkit

---

## Step 581: Navigation Service

```csharp
// ============================================
// Strongly-Typed Navigation Service
// ============================================

public interface INavigationService
{
    Task GoToAsync<TPage>(object? parameter = null) where TPage : Page;
    Task GoBackAsync();
    Task GoBackToRootAsync();
    Task PushModalAsync<TPage>(object? parameter = null) where TPage : Page;
    Task PopModalAsync();
    string CurrentRoute { get; }
}

public class MauiNavigationService : INavigationService
{
    private readonly Dictionary<Type, string> _routes = new();
    private readonly IServiceProvider _services;
    
    public MauiNavigationService(IServiceProvider services) => _services = services;
    
    public string CurrentRoute
        => Shell.Current?.CurrentState.Location.ToString() ?? "/";
    
    public void RegisterRoute<TPage>() where TPage : Page
    {
        var route = typeof(TPage).Name.Replace("Page", "");
        _routes[typeof(TPage)] = route;
        Routing.RegisterRoute(route, typeof(TPage));
    }
    
    public async Task GoToAsync<TPage>(object? parameter = null) where TPage : Page
    {
        if (!_routes.TryGetValue(typeof(TPage), out var route))
            throw new InvalidOperationException($"Route not registered: {typeof(TPage).Name}");
        
        if (parameter != null)
            await Shell.Current.GoToAsync(route,
                new Dictionary<string, object> { ["parameter"] = parameter });
        else
            await Shell.Current.GoToAsync(route);
    }
    
    public Task GoBackAsync() => Shell.Current.GoToAsync("..");
    
    public Task GoBackToRootAsync()
        => Shell.Current.GoToAsync("//Main");
    
    public async Task PushModalAsync<TPage>(object? parameter = null) where TPage : Page
    {
        var page = _services.GetRequiredService<TPage>();
        if (parameter != null && page.BindingContext is IQueryAttributable qp)
            qp.ApplyQueryAttributes(new Dictionary<string, object> { ["parameter"] = parameter });
        
        await Application.Current!.MainPage!.Navigation.PushModalAsync(page);
    }
    
    public Task PopModalAsync()
        => Application.Current!.MainPage!.Navigation.PopModalAsync();
}

// ViewModel receives navigation parameter
public partial class ProductDetailViewModel : ObservableObject, IQueryAttributable
{
    private readonly IProductRepository _repo;
    
    [ObservableProperty] private ProductDto? _product;
    [ObservableProperty] private bool _isLoading;
    
    public ProductDetailViewModel(IProductRepository repo) => _repo = repo;
    
    public void ApplyQueryAttributes(IDictionary<string, object> query)
    {
        if (query.TryGetValue("parameter", out var param) && param is int id)
            _ = LoadProductAsync(id);
    }
    
    private async Task LoadProductAsync(int id)
    {
        IsLoading = true;
        Product = await _repo.GetByIdAsync(id);
        IsLoading = false;
    }
}
```

---

## Step 582: Dialog Service

```csharp
// ============================================
// Dialog & Alert Service
// ============================================

public interface IDialogService
{
    Task ShowAlertAsync(string title, string message, string cancel = "ตกลง");
    Task<bool> ShowConfirmAsync(string title, string message,
        string accept = "ยืนยัน", string cancel = "ยกเลิก");
    Task<string?> ShowInputAsync(string title, string prompt, string? placeholder = null);
    Task<string?> ShowActionSheetAsync(string title, string cancel, params string[] buttons);
    Task ShowToastAsync(string message, int durationMs = 2000);
    Task ShowLoadingAsync(string message = "กำลังโหลด...");
    void HideLoading();
}

public class MauiDialogService : IDialogService
{
    private Page MainPage => Application.Current!.MainPage!;
    private LoadingOverlay? _loadingOverlay;
    
    public Task ShowAlertAsync(string title, string message, string cancel = "ตกลง")
        => MainPage.DisplayAlert(title, message, cancel);
    
    public Task<bool> ShowConfirmAsync(string title, string message,
        string accept = "ยืนยัน", string cancel = "ยกเลิก")
        => MainPage.DisplayAlert(title, message, accept, cancel);
    
    public Task<string?> ShowInputAsync(string title, string prompt, string? placeholder = null)
        => MainPage.DisplayPromptAsync(title, prompt, placeholder: placeholder);
    
    public Task<string?> ShowActionSheetAsync(string title, string cancel, params string[] buttons)
        => MainPage.DisplayActionSheet(title, cancel, null, buttons);
    
    public async Task ShowToastAsync(string message, int durationMs = 2000)
    {
        var toast = CommunityToolkit.Maui.Alerts.Toast.Make(message);
        await toast.Show();
    }
    
    public async Task ShowLoadingAsync(string message = "กำลังโหลด...")
    {
        _loadingOverlay = new LoadingOverlay(message);
        await MainPage.Navigation.PushModalAsync(_loadingOverlay, false);
    }
    
    public void HideLoading()
    {
        if (_loadingOverlay != null)
        {
            MainPage.Navigation.PopModalAsync(false);
            _loadingOverlay = null;
        }
    }
}

// Usage in ViewModel
public partial class DeleteProductViewModel : ObservableObject
{
    private readonly IDialogService _dialogs;
    private readonly IProductRepository _repo;
    private readonly INavigationService _nav;
    
    public DeleteProductViewModel(
        IDialogService dialogs,
        IProductRepository repo,
        INavigationService nav)
    {
        _dialogs = dialogs;
        _repo = repo;
        _nav = nav;
    }
    
    [RelayCommand]
    private async Task DeleteAsync(int productId)
    {
        var confirmed = await _dialogs.ShowConfirmAsync(
            "ยืนยันการลบ",
            "คุณต้องการลบสินค้านี้หรือไม่?");
        
        if (!confirmed) return;
        
        await _dialogs.ShowLoadingAsync("กำลังลบ...");
        
        try
        {
            await _repo.DeleteAsync(productId);
            _dialogs.HideLoading();
            await _dialogs.ShowToastAsync("ลบสินค้าเรียบร้อยแล้ว");
            await _nav.GoBackAsync();
        }
        catch
        {
            _dialogs.HideLoading();
            await _dialogs.ShowAlertAsync("เกิดข้อผิดพลาด", "ไม่สามารถลบสินค้าได้");
        }
    }
}
```

---

## Step 583: Form Validation

```csharp
// ============================================
// Reactive Form Validation
// ============================================

public class ValidationRule<T>
{
    private readonly Func<T, bool> _check;
    private readonly string _errorMessage;
    
    public ValidationRule(Func<T, bool> check, string errorMessage)
    {
        _check = check;
        _errorMessage = errorMessage;
    }
    
    public string? Validate(T value) => _check(value) ? null : _errorMessage;
}

public class ValidatableProperty<T> : ObservableObject
{
    private T _value;
    private List<string> _errors = new();
    private bool _isDirty;
    
    [ObservableProperty] private List<string> errors = new();
    [ObservableProperty] private bool hasErrors;
    [ObservableProperty] private bool isValid;
    
    private readonly List<ValidationRule<T>> _rules = new();
    
    public T Value
    {
        get => _value;
        set
        {
            SetProperty(ref _value, value);
            _isDirty = true;
            Validate();
        }
    }
    
    public ValidatableProperty(T initial = default!) => _value = initial;
    
    public void AddRule(Func<T, bool> check, string error)
        => _rules.Add(new ValidationRule<T>(check, error));
    
    public bool Validate()
    {
        if (!_isDirty) return IsValid;
        
        var errs = _rules
            .Select(r => r.Validate(_value))
            .Where(e => e != null)
            .Cast<string>()
            .ToList();
        
        Errors = errs;
        HasErrors = errs.Count > 0;
        IsValid = errs.Count == 0;
        return IsValid;
    }
}

// Form ViewModel
public partial class RegisterViewModel : ObservableObject
{
    private readonly IAuthService _auth;
    private readonly IDialogService _dialogs;
    private readonly INavigationService _nav;
    
    public ValidatableProperty<string> Email { get; } = new(string.Empty);
    public ValidatableProperty<string> Password { get; } = new(string.Empty);
    public ValidatableProperty<string> ConfirmPassword { get; } = new(string.Empty);
    public ValidatableProperty<string> Name { get; } = new(string.Empty);
    
    [ObservableProperty] private bool _isSubmitting;
    
    public bool IsFormValid =>
        Email.IsValid && Password.IsValid && ConfirmPassword.IsValid && Name.IsValid;
    
    public RegisterViewModel(
        IAuthService auth, IDialogService dialogs, INavigationService nav)
    {
        _auth = auth;
        _dialogs = dialogs;
        _nav = nav;
        
        SetupValidation();
    }
    
    private void SetupValidation()
    {
        Email.AddRule(v => !string.IsNullOrWhiteSpace(v), "กรุณากรอกอีเมล");
        Email.AddRule(v => v.Contains('@') && v.Contains('.'), "รูปแบบอีเมลไม่ถูกต้อง");
        
        Password.AddRule(v => !string.IsNullOrWhiteSpace(v), "กรุณากรอกรหัสผ่าน");
        Password.AddRule(v => v.Length >= 8, "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร");
        Password.AddRule(v => v.Any(char.IsUpper), "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว");
        Password.AddRule(v => v.Any(char.IsDigit), "ต้องมีตัวเลขอย่างน้อย 1 ตัว");
        
        ConfirmPassword.AddRule(v => v == Password.Value, "รหัสผ่านไม่ตรงกัน");
        
        Name.AddRule(v => !string.IsNullOrWhiteSpace(v), "กรุณากรอกชื่อ");
        Name.AddRule(v => v.Length >= 2, "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร");
    }
    
    [RelayCommand]
    private async Task RegisterAsync()
    {
        var valid = new[] { Email, Password, ConfirmPassword, Name }
            .All(p => p.Validate());
        
        if (!valid) return;
        
        IsSubmitting = true;
        
        try
        {
            await _auth.RegisterAsync(Email.Value, Password.Value, Name.Value);
            await _dialogs.ShowToastAsync("สมัครสมาชิกสำเร็จ!");
            await _nav.GoBackToRootAsync();
        }
        catch (Exception ex)
        {
            await _dialogs.ShowAlertAsync("เกิดข้อผิดพลาด", ex.Message);
        }
        finally { IsSubmitting = false; }
    }
}
```

---

## Step 584: ViewModel Messaging

```csharp
// ============================================
// WeakReferenceMessenger (CommunityToolkit)
// ============================================

// Messages
public record CartUpdatedMessage(int ItemCount, decimal Total);
public record UserLoggedInMessage(int UserId, string Name);
public record UserLoggedOutMessage;
public record ThemeChangedMessage(AppTheme Theme);
public record RefreshRequiredMessage(string EntityType);

// Sender (CartViewModel)
public partial class CartViewModelV2 : ObservableObject
{
    [ObservableProperty] private int _itemCount;
    [ObservableProperty] private decimal _total;
    
    private void NotifyCartUpdated()
        => WeakReferenceMessenger.Default.Send(new CartUpdatedMessage(ItemCount, Total));
    
    [RelayCommand]
    private void AddItem(CartItem item)
    {
        ItemCount++;
        Total += item.UnitPrice;
        NotifyCartUpdated();
    }
}

// Receiver (Tab bar badge)
public partial class TabBarViewModel : ObservableObject,
    IRecipient<CartUpdatedMessage>,
    IRecipient<UserLoggedInMessage>
{
    [ObservableProperty] private int _cartBadge;
    [ObservableProperty] private string _userName = "Guest";
    
    public TabBarViewModel()
    {
        WeakReferenceMessenger.Default.RegisterAll(this);
    }
    
    public void Receive(CartUpdatedMessage message)
        => MainThread.BeginInvokeOnMainThread(
            () => CartBadge = message.ItemCount);
    
    public void Receive(UserLoggedInMessage message)
        => MainThread.BeginInvokeOnMainThread(
            () => UserName = message.Name);
    
    ~TabBarViewModel()
    {
        WeakReferenceMessenger.Default.UnregisterAll(this);
    }
}
```

---

## Step 585: MVVM Source Generator Patterns

```csharp
// ============================================
// CommunityToolkit.Mvvm Source Generators
// ============================================

// ObservableObject + RelayCommand + ObservableProperty
[ObservableObject]
public partial class ProductListViewModel
{
    private readonly IProductRepository _repo;
    private readonly INavigationService _nav;
    
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(DeleteSelectedCommand))]
    [NotifyPropertyChangedFor(nameof(HasSelection))]
    private ProductDto? _selectedProduct;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FilteredProducts))]
    private string _searchText = string.Empty;
    
    [ObservableProperty] private ObservableCollection<ProductDto> _products = new();
    [ObservableProperty] private bool _isLoading;
    [ObservableProperty] private bool _isEmpty;
    
    public bool HasSelection => SelectedProduct != null;
    
    public IEnumerable<ProductDto> FilteredProducts =>
        string.IsNullOrWhiteSpace(SearchText)
            ? Products
            : Products.Where(p =>
                p.Name.Contains(SearchText, StringComparison.OrdinalIgnoreCase) ||
                p.Category.Contains(SearchText, StringComparison.OrdinalIgnoreCase));
    
    public ProductListViewModel(IProductRepository repo, INavigationService nav)
    {
        _repo = repo;
        _nav = nav;
    }
    
    [RelayCommand]
    private async Task LoadAsync(CancellationToken ct)
    {
        IsLoading = true;
        try
        {
            var items = await _repo.GetAllAsync(ct);
            Products = new ObservableCollection<ProductDto>(items);
            IsEmpty = Products.Count == 0;
        }
        finally { IsLoading = false; }
    }
    
    [RelayCommand]
    private Task NavigateToDetailAsync(int id)
        => _nav.GoToAsync<ProductDetailPage>(id);
    
    [RelayCommand(CanExecute = nameof(HasSelection))]
    private async Task DeleteSelectedAsync()
    {
        if (SelectedProduct == null) return;
        await _repo.DeleteAsync(SelectedProduct.Id);
        Products.Remove(SelectedProduct);
        SelectedProduct = null;
        IsEmpty = Products.Count == 0;
    }
}
```

---

## Step 586: ViewModel Lifecycle

```csharp
// ============================================
// ViewModel Lifecycle Management
// ============================================

public interface IViewModelLifecycle
{
    Task OnAppearingAsync();
    Task OnDisappearingAsync();
}

public abstract class PageViewModel : ObservableObject, IViewModelLifecycle
{
    public virtual Task OnAppearingAsync() => Task.CompletedTask;
    public virtual Task OnDisappearingAsync() => Task.CompletedTask;
}

// Base page that hooks lifecycle
public abstract class BasePage<TViewModel> : ContentPage
    where TViewModel : PageViewModel
{
    protected TViewModel ViewModel { get; }
    
    protected BasePage(TViewModel vm)
    {
        ViewModel = vm;
        BindingContext = vm;
    }
    
    protected override void OnAppearing()
    {
        base.OnAppearing();
        _ = ViewModel.OnAppearingAsync();
    }
    
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _ = ViewModel.OnDisappearingAsync();
    }
}

// ViewModel with lifecycle
public class OrderListViewModel : PageViewModel
{
    private readonly IOrderRepository _repo;
    private bool _isFirstLoad = true;
    
    [ObservableProperty] private List<OrderDto> _orders = new();
    
    public OrderListViewModel(IOrderRepository repo) => _repo = repo;
    
    public override async Task OnAppearingAsync()
    {
        // Reload on every appearance (catches new orders placed from detail)
        await LoadOrdersAsync();
    }
    
    public override Task OnDisappearingAsync()
    {
        // Save scroll position, clear selection etc.
        return Task.CompletedTask;
    }
    
    private async Task LoadOrdersAsync()
    {
        Orders = await _repo.GetUserOrdersAsync();
    }
}
```

---

## Step 587: ViewModel Factory

```csharp
// ============================================
// ViewModel Factory for Dynamic Creation
// ============================================

public interface IViewModelFactory<TViewModel>
{
    TViewModel Create(object? parameter = null);
}

// Dynamic product card VM
public class ProductCardViewModel : ObservableObject
{
    [ObservableProperty] private ProductDto _product;
    [ObservableProperty] private bool _isInCart;
    [ObservableProperty] private int _cartQuantity;
    
    private readonly Store<AppState> _store;
    private readonly IDisposable _subscription;
    
    public ProductCardViewModel(ProductDto product, Store<AppState> store)
    {
        _product = product;
        _store = store;
        _subscription = store.Subscribe(OnStateChanged);
        OnStateChanged(store.State);
    }
    
    private void OnStateChanged(AppState state)
    {
        var item = state.Cart.Items.FirstOrDefault(i => i.ProductId == _product.Id);
        IsInCart = item != null;
        CartQuantity = item?.Quantity ?? 0;
    }
    
    [RelayCommand]
    private async Task AddToCartAsync()
        => await _store.DispatchAsync(
            new AddToCartAction(_product.Id, _product.Name, _product.Price));
    
    public void Dispose() => _subscription.Dispose();
}

// Factory
public class ProductCardViewModelFactory : IViewModelFactory<ProductCardViewModel>
{
    private readonly Store<AppState> _store;
    
    public ProductCardViewModelFactory(Store<AppState> store) => _store = store;
    
    public ProductCardViewModel Create(object? parameter = null)
    {
        var product = parameter as ProductDto
            ?? throw new ArgumentException("ProductDto required");
        return new ProductCardViewModel(product, _store);
    }
}
```

---

## Step 588: Composite ViewModel

```csharp
// ============================================
// Composite ViewModel (Shell + children)
// ============================================

public partial class HomeViewModel : ObservableObject
{
    // Composed child ViewModels
    public FeaturedBannerViewModel Banner { get; }
    public CategoryListViewModel Categories { get; }
    public FlashSaleViewModel FlashSale { get; }
    public RecommendedViewModel Recommended { get; }
    
    [ObservableProperty] private bool _isRefreshing;
    
    public HomeViewModel(
        FeaturedBannerViewModel banner,
        CategoryListViewModel categories,
        FlashSaleViewModel flashSale,
        RecommendedViewModel recommended)
    {
        Banner = banner;
        Categories = categories;
        FlashSale = flashSale;
        Recommended = recommended;
    }
    
    [RelayCommand]
    private async Task RefreshAsync()
    {
        IsRefreshing = true;
        
        await Task.WhenAll(
            Banner.LoadAsync(),
            Categories.LoadAsync(),
            FlashSale.LoadAsync(),
            Recommended.LoadAsync());
        
        IsRefreshing = false;
    }
}

// Child ViewModel
public partial class FlashSaleViewModel : ObservableObject
{
    private readonly IFlashSaleService _service;
    private System.Timers.Timer? _countdown;
    
    [ObservableProperty] private List<FlashSaleItem> _items = new();
    [ObservableProperty] private TimeSpan _timeRemaining;
    [ObservableProperty] private bool _isActive;
    
    public FlashSaleViewModel(IFlashSaleService service) => _service = service;
    
    public async Task LoadAsync()
    {
        var sale = await _service.GetCurrentSaleAsync();
        if (sale == null) { IsActive = false; return; }
        
        Items = sale.Items;
        IsActive = true;
        StartCountdown(sale.EndsAt);
    }
    
    private void StartCountdown(DateTime endsAt)
    {
        _countdown?.Dispose();
        _countdown = new System.Timers.Timer(1000);
        _countdown.Elapsed += (_, _) =>
        {
            var remaining = endsAt - DateTime.Now;
            TimeRemaining = remaining > TimeSpan.Zero ? remaining : TimeSpan.Zero;
            
            if (TimeRemaining == TimeSpan.Zero)
            {
                _countdown?.Dispose();
                IsActive = false;
            }
        };
        _countdown.Start();
    }
}
```

---

## Step 589: ViewModel Error Pipeline

```csharp
// ============================================
// Centralized Error Handling Pipeline
// ============================================

public abstract class BaseViewModelV2 : ObservableObject
{
    [ObservableProperty] private bool _isBusy;
    [ObservableProperty] private string? _errorMessage;
    [ObservableProperty] private bool _hasError;
    
    protected readonly IDialogService Dialogs;
    
    protected BaseViewModelV2(IDialogService dialogs) => Dialogs = dialogs;
    
    protected async Task ExecuteAsync(Func<Task> action, string? loadingMessage = null)
    {
        if (IsBusy) return;
        IsBusy = true;
        ErrorMessage = null;
        HasError = false;
        
        if (loadingMessage != null)
            await Dialogs.ShowLoadingAsync(loadingMessage);
        
        try
        {
            await action();
        }
        catch (UnauthorizedAccessException)
        {
            ErrorMessage = "กรุณาเข้าสู่ระบบใหม่";
            HasError = true;
            WeakReferenceMessenger.Default.Send(new UserLoggedOutMessage());
        }
        catch (HttpRequestException)
        {
            ErrorMessage = "ไม่สามารถเชื่อมต่ออินเทอร์เน็ตได้";
            HasError = true;
        }
        catch (TaskCanceledException)
        {
            ErrorMessage = "การร้องขอหมดเวลา โปรดลองใหม่";
            HasError = true;
        }
        catch (DomainException ex)
        {
            ErrorMessage = ex.Message;
            HasError = true;
            await Dialogs.ShowAlertAsync("ข้อผิดพลาด", ex.Message);
        }
        catch (Exception ex)
        {
            ErrorMessage = "เกิดข้อผิดพลาดที่ไม่คาดคิด";
            HasError = true;
            System.Diagnostics.Debug.WriteLine($"Unhandled: {ex}");
        }
        finally
        {
            IsBusy = false;
            if (loadingMessage != null) Dialogs.HideLoading();
        }
    }
}

// DomainException for business rule violations
public class DomainException : Exception
{
    public string Code { get; }
    
    public DomainException(string code, string message) : base(message)
        => Code = code;
}
```

---

## Step 590: ViewModel Testing

```csharp
// ============================================
// ViewModel Unit Tests
// ============================================

public class ProductListViewModelTests
{
    private readonly Mock<IProductRepository> _repo = new();
    private readonly Mock<INavigationService> _nav = new();
    
    [Fact]
    public async Task LoadCommand_PopulatesProducts()
    {
        _repo.Setup(r => r.GetAllAsync(default))
            .ReturnsAsync(new List<ProductDto>
            {
                new ProductDto(1, "สินค้า A", "หมวด 1", 100m),
                new ProductDto(2, "สินค้า B", "หมวด 2", 200m)
            });
        
        var vm = new ProductListViewModel(_repo.Object, _nav.Object);
        await vm.LoadCommand.ExecuteAsync(default);
        
        vm.Products.Should().HaveCount(2);
        vm.IsEmpty.Should().BeFalse();
        vm.IsLoading.Should().BeFalse();
    }
    
    [Fact]
    public async Task SearchText_FiltersProducts()
    {
        _repo.Setup(r => r.GetAllAsync(default))
            .ReturnsAsync(new List<ProductDto>
            {
                new(1, "เสื้อผ้า", "แฟชั่น", 100m),
                new(2, "รองเท้า", "แฟชั่น", 200m),
                new(3, "กระเป๋า", "กระเป๋า", 300m)
            });
        
        var vm = new ProductListViewModel(_repo.Object, _nav.Object);
        await vm.LoadCommand.ExecuteAsync(default);
        
        vm.SearchText = "แฟชั่น";
        
        vm.FilteredProducts.Should().HaveCount(2);
    }
    
    [Fact]
    public void DeleteSelectedCommand_CanExecute_OnlyWhenSelected()
    {
        var vm = new ProductListViewModel(_repo.Object, _nav.Object);
        
        vm.DeleteSelectedCommand.CanExecute(null).Should().BeFalse();
        
        vm.SelectedProduct = new ProductDto(1, "Test", "Cat", 100m);
        
        vm.DeleteSelectedCommand.CanExecute(null).Should().BeTrue();
    }
    
    [Fact]
    public async Task RegisterForm_InvalidEmail_ShowsError()
    {
        var auth = new Mock<IAuthService>();
        var dialogs = new Mock<IDialogService>();
        var nav = new Mock<INavigationService>();
        
        var vm = new RegisterViewModel(auth.Object, dialogs.Object, nav.Object);
        
        vm.Email.Value = "not-an-email";
        vm.Password.Value = "Valid1Pass";
        vm.ConfirmPassword.Value = "Valid1Pass";
        vm.Name.Value = "ชื่อ";
        
        await vm.RegisterCommand.ExecuteAsync(null);
        
        vm.Email.HasErrors.Should().BeTrue();
        vm.Email.Errors.Should().Contain("รูปแบบอีเมลไม่ถูกต้อง");
        auth.Verify(a => a.RegisterAsync(It.IsAny<string>(),
            It.IsAny<string>(), It.IsAny<string>()), Times.Never);
    }
}
```

---

## สรุป Part 59

ใน Part 59 เราได้เรียนรู้:

1. **Navigation Service** - Type-safe Shell navigation
2. **Dialog Service** - Alert, confirm, input, loading
3. **Form Validation** - ValidatableProperty<T>
4. **WeakReferenceMessenger** - Decoupled messaging
5. **MVVM Source Generators** - ObservableProperty, RelayCommand
6. **ViewModel Lifecycle** - OnAppearing/Disappearing
7. **ViewModel Factory** - Dynamic creation
8. **Composite ViewModel** - Shell + children
9. **Error Pipeline** - BaseViewModel error handling
10. **ViewModel Testing** - Unit, validation, commands

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 59 | Steps 581-590*
