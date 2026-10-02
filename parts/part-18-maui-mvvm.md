# Part 18: MVVM Pattern ใน .NET MAUI
## Steps 171-180: Model-View-ViewModel Pattern

---

## Step 171: MVVM Overview

```csharp
// ============================================
// MVVM Pattern
// ============================================

/*
 * MVVM = Model - View - ViewModel
 * 
 * Model     : ข้อมูลและ business logic
 * View      : UI (XAML)
 * ViewModel : สื่อกลาง View ↔ Model
 * 
 * ┌──────────────┐    Data Binding     ┌──────────────┐
 * │     View     │ ←────────────────→  │  ViewModel   │
 * │   (XAML)     │    Commands         │  (C# class)  │
 * └──────────────┘                     └──────┬───────┘
 *                                             │ calls/observes
 *                                      ┌──────▼───────┐
 *                                      │    Model      │
 *                                      │  (data/logic) │
 *                                      └──────────────┘
 *
 * ประโยชน์:
 * - Test ViewModel ได้โดยไม่ต้องมี UI
 * - Separation of Concerns
 * - Reuse ViewModel ได้
 */
```

---

## Step 172: CommunityToolkit.Mvvm

```csharp
// ============================================
// CommunityToolkit.Mvvm - เขียน MVVM น้อยลง
// ============================================

// NuGet: CommunityToolkit.Mvvm
// ใน MauiProgram.cs:
// builder.UseMauiCommunityToolkit();

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

// ============================================
// ObservableObject Base Class
// ============================================

public partial class ProductViewModel : ObservableObject
{
    // [ObservableProperty] generates property + notification
    [ObservableProperty]
    private string _name = string.Empty;
    
    [ObservableProperty]
    private decimal _price;
    
    [ObservableProperty]
    private int _stock;
    
    [ObservableProperty]
    private bool _isLoading;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(StockStatus))]
    private int _quantity;
    
    // Computed property
    public string StockStatus => _quantity switch
    {
        0 => "หมดสต็อก",
        <= 5 => "เหลือน้อย",
        _ => "มีสินค้า"
    };
    
    // [ObservableProperty] with validation
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกชื่อสินค้า")]
    [MinLength(2, ErrorMessage = "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร")]
    private string _productName = string.Empty;
}

// ============================================
// Commands with [RelayCommand]
// ============================================

public partial class MainViewModel3 : ObservableObject
{
    [ObservableProperty]
    private string _message = string.Empty;
    
    [ObservableProperty]
    private bool _isLoading;
    
    // Synchronous command
    [RelayCommand]
    private void Clear()
    {
        Message = string.Empty;
    }
    
    // Async command
    [RelayCommand]
    private async Task LoadDataAsync(CancellationToken ct)
    {
        IsLoading = true;
        try
        {
            await Task.Delay(1000, ct); // simulate loading
            Message = "ข้อมูลโหลดสำเร็จ!";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    // Command with parameter
    [RelayCommand]
    private void SelectItem(string item)
    {
        Message = $"เลือก: {item}";
    }
    
    // Command with CanExecute
    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync()
    {
        await Task.Delay(500);
        Message = "บันทึกสำเร็จ!";
    }
    
    private bool CanSave() => !string.IsNullOrEmpty(Message);
    
    // Cancel running command
    [RelayCommand]
    private async Task LongOperationAsync(CancellationToken ct)
    {
        for (int i = 0; i < 100; i++)
        {
            await Task.Delay(100, ct);
            Message = $"Progress: {i}%";
        }
    }
    
    // Call CancelLongOperationCommand.Execute(null) to cancel
}
```

---

## Step 173: Data Binding ขั้นสูง

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- Advanced Data Binding -->
<!-- ============================================ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:converters="clr-namespace:MyApp.Converters"
             x:Class="MyApp.Views.BindingDemoPage">
    
    <ContentPage.Resources>
        <!-- Converters -->
        <converters:BoolToStringConverter x:Key="BoolToStatus"
            TrueText="ใช้งาน" FalseText="ปิดใช้งาน" />
        <converters:InverseBoolConverter x:Key="InverseBool" />
        <converters:NullToBoolConverter x:Key="NullToVisible" />
        
        <!-- Formatting -->
        <x:String x:Key="CurrencyFormat">{0:C}</x:String>
        <x:String x:Key="PercentFormat">{0:P1}</x:String>
    </ContentPage.Resources>
    
    <VerticalStackLayout Spacing="15" Padding="20">
        
        <!-- TwoWay binding (default for Entry) -->
        <Entry Text="{Binding Name, Mode=TwoWay}" Placeholder="ชื่อ" />
        
        <!-- OneWay binding (read-only) -->
        <Label Text="{Binding Name, Mode=OneWay}" />
        
        <!-- OneWayToSource (write-only) -->
        <Slider Value="{Binding Volume, Mode=OneWayToSource}" 
                Minimum="0" Maximum="100" />
        
        <!-- OneTime binding (set once) -->
        <Label Text="{Binding AppVersion, Mode=OneTime}" />
        
        <!-- StringFormat -->
        <Label Text="{Binding Price, StringFormat='ราคา: {0:C}'}" />
        <Label Text="{Binding Score, StringFormat='คะแนน: {0:F2}%'}" />
        <Label Text="{Binding Date, StringFormat='วันที่: {0:dd/MM/yyyy}'}" />
        
        <!-- Converter -->
        <Label Text="{Binding IsActive, Converter={StaticResource BoolToStatus}}" />
        <Button IsEnabled="{Binding IsLoading, Converter={StaticResource InverseBool}}"
                Text="ดำเนินการ" />
        
        <!-- FallbackValue -->
        <Label Text="{Binding Nickname, FallbackValue='ไม่มีชื่อเล่น'}" />
        
        <!-- TargetNullValue -->
        <Label Text="{Binding Email, TargetNullValue='ไม่ได้ระบุอีเมล'}" />
        
        <!-- Multi binding -->
        <Label>
            <Label.Text>
                <MultiBinding StringFormat="{}{0} {1} ({2})">
                    <Binding Path="FirstName" />
                    <Binding Path="LastName" />
                    <Binding Path="Age" />
                </MultiBinding>
            </Label.Text>
        </Label>
        
        <!-- Binding to non-BindingContext element -->
        <Slider x:Name="FontSlider" Minimum="10" Maximum="30" Value="16" />
        <Label Text="Sample text" 
               FontSize="{Binding Source={x:Reference FontSlider}, Path=Value}" />
        
        <!-- Static binding -->
        <Label Text="{x:Static sys:DateTime.Now}" 
               xmlns:sys="clr-namespace:System;assembly=mscorlib" />
        
    </VerticalStackLayout>
    
</ContentPage>
```

---

## Step 174: Value Converters

```csharp
// ============================================
// IValueConverter
// ============================================

using Microsoft.Maui.Controls;

public class BoolToStringConverter : IValueConverter
{
    public string TrueText { get; set; } = "Yes";
    public string FalseText { get; set; } = "No";
    
    public object Convert(object? value, Type targetType, object? parameter, 
        System.Globalization.CultureInfo culture)
        => (bool?)value == true ? TrueText : FalseText;
    
    public object ConvertBack(object? value, Type targetType, object? parameter, 
        System.Globalization.CultureInfo culture)
        => (string?)value == TrueText;
}

public class InverseBoolConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter, 
        System.Globalization.CultureInfo culture)
        => !(bool?)value ?? true;
    
    public object ConvertBack(object? value, Type targetType, object? parameter, 
        System.Globalization.CultureInfo culture)
        => !(bool?)value ?? true;
}

public class NullToBoolConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
        => value != null;
    
    public object ConvertBack(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
        => throw new NotImplementedException();
}

public class BytesToImageConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
    {
        if (value is byte[] bytes && bytes.Length > 0)
            return ImageSource.FromStream(() => new MemoryStream(bytes));
        return "placeholder.png";
    }
    
    public object ConvertBack(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
        => throw new NotImplementedException();
}

public class StockToColorConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
    {
        int stock = (int?)value ?? 0;
        return stock switch
        {
            0 => Colors.Red,
            <= 5 => Colors.Orange,
            _ => Colors.Green
        };
    }
    
    public object ConvertBack(object? value, Type targetType, object? parameter,
        System.Globalization.CultureInfo culture)
        => throw new NotImplementedException();
}

// ============================================
// Multi-value Converter
// ============================================

public class AllTrueConverter : IMultiValueConverter
{
    public object Convert(object[] values, Type targetType, object parameter,
        System.Globalization.CultureInfo culture)
        => values.All(v => v is bool b && b);
    
    public object[] ConvertBack(object value, Type[] targetTypes, object parameter,
        System.Globalization.CultureInfo culture)
        => throw new NotImplementedException();
}
```

---

## Step 175: ObservableCollection

```csharp
// ============================================
// ObservableCollection - Collection ที่ notify UI
// ============================================

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class ProductsViewModel2 : ObservableObject
{
    [ObservableProperty]
    private ObservableCollection<ProductItem> _products = new();
    
    [ObservableProperty]
    private ProductItem? _selectedProduct;
    
    [ObservableProperty]
    private bool _isRefreshing;
    
    [ObservableProperty]
    private string _searchText = string.Empty;
    
    [ObservableProperty]
    private ObservableCollection<ProductItem> _filteredProducts = new();
    
    partial void OnSearchTextChanged(string value)
    {
        FilterProducts(value);
    }
    
    private void FilterProducts(string search)
    {
        if (string.IsNullOrWhiteSpace(search))
        {
            FilteredProducts = new ObservableCollection<ProductItem>(Products);
            return;
        }
        
        var filtered = Products
            .Where(p => p.Name.Contains(search, StringComparison.OrdinalIgnoreCase) ||
                        p.Category.Contains(search, StringComparison.OrdinalIgnoreCase));
        
        FilteredProducts = new ObservableCollection<ProductItem>(filtered);
    }
    
    [RelayCommand]
    private async Task LoadProductsAsync()
    {
        IsRefreshing = true;
        try
        {
            await Task.Delay(1000); // simulate API call
            
            var items = new[]
            {
                new ProductItem(1, "iPhone 15", "Electronics", 35000, 100),
                new ProductItem(2, "Samsung S24", "Electronics", 30000, 150),
                new ProductItem(3, "Nike Air Max", "Shoes", 4500, 200),
                new ProductItem(4, "iPad Pro", "Electronics", 42000, 50),
                new ProductItem(5, "Adidas Ultra", "Shoes", 3800, 180),
            };
            
            Products.Clear();
            foreach (var item in items)
                Products.Add(item);
            
            FilterProducts(SearchText);
        }
        finally
        {
            IsRefreshing = false;
        }
    }
    
    [RelayCommand]
    private async Task AddProductAsync()
    {
        // Navigate to add product page
        await Shell.Current.GoToAsync("add-product");
    }
    
    [RelayCommand]
    private async Task SelectProductAsync(ProductItem product)
    {
        SelectedProduct = product;
        await Shell.Current.GoToAsync("product-detail", new Dictionary<string, object>
        {
            { "product", product }
        });
    }
    
    [RelayCommand]
    private void DeleteProduct(ProductItem product)
    {
        Products.Remove(product);
        FilterProducts(SearchText);
    }
}

public partial class ProductItem : ObservableObject
{
    public int Id { get; }
    
    [ObservableProperty]
    private string _name;
    
    [ObservableProperty]
    private string _category;
    
    [ObservableProperty]
    private decimal _price;
    
    [ObservableProperty]
    private int _stock;
    
    public ProductItem(int id, string name, string category, decimal price, int stock)
    {
        Id = id;
        _name = name;
        _category = category;
        _price = price;
        _stock = stock;
    }
    
    public bool IsLowStock => Stock <= 5;
    public bool IsOutOfStock => Stock == 0;
    public string StockText => Stock == 0 ? "หมดสต็อก" : $"{Stock} ชิ้น";
}
```

---

## Step 176: Navigation ด้วย Shell

```csharp
// ============================================
// Shell Navigation
// ============================================

// AppShell.xaml.cs
public partial class AppShell2 : Shell
{
    public AppShell2()
    {
        InitializeComponent();
        
        // Register routes
        Routing.RegisterRoute("products", typeof(ProductsPage2));
        Routing.RegisterRoute("product-detail", typeof(ProductDetailPage));
        Routing.RegisterRoute("add-product", typeof(AddProductPage));
        Routing.RegisterRoute("profile", typeof(ProfilePage));
        Routing.RegisterRoute("settings", typeof(SettingsPage2));
    }
}

// AppShell.xaml
/*
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:MyApp.Views"
       x:Class="MyApp.AppShell2"
       FlyoutBehavior="Flyout"
       Title="MyApp">

    <!-- Tab Bar -->
    <TabBar>
        <ShellContent Title="หน้าหลัก" Icon="home.png"
                      ContentTemplate="{DataTemplate views:HomePage}" />
        <ShellContent Title="สินค้า" Icon="products.png"
                      ContentTemplate="{DataTemplate views:ProductsPage2}" />
        <ShellContent Title="โปรไฟล์" Icon="profile.png"
                      ContentTemplate="{DataTemplate views:ProfilePage}" />
    </TabBar>

    <!-- Flyout (Hamburger menu) -->
    <FlyoutItem Title="หน้าหลัก" Icon="home.png">
        <ShellContent ContentTemplate="{DataTemplate views:HomePage}" />
    </FlyoutItem>
    <FlyoutItem Title="ตั้งค่า" Icon="settings.png">
        <ShellContent ContentTemplate="{DataTemplate views:SettingsPage2}" />
    </FlyoutItem>

</Shell>
*/

// ============================================
// Navigation Service
// ============================================

public interface INavigationService2
{
    Task GoToAsync(string route);
    Task GoToAsync(string route, Dictionary<string, object> parameters);
    Task GoBackAsync();
    Task GoToRootAsync();
}

public class NavigationService2 : INavigationService2
{
    public Task GoToAsync(string route)
        => Shell.Current.GoToAsync(route);
    
    public Task GoToAsync(string route, Dictionary<string, object> parameters)
        => Shell.Current.GoToAsync(route, parameters);
    
    public Task GoBackAsync()
        => Shell.Current.GoToAsync("..");
    
    public Task GoToRootAsync()
        => Shell.Current.GoToAsync("//main");
}

// ============================================
// ViewModel รับ navigation parameters
// ============================================

[QueryProperty(nameof(ProductId), "id")]
[QueryProperty(nameof(Product), "product")]
public partial class ProductDetailViewModel : ObservableObject
{
    [ObservableProperty]
    private int _productId;
    
    [ObservableProperty]
    private ProductItem? _product;
    
    partial void OnProductIdChanged(int value)
    {
        LoadProductAsync(value).FireAndForget();
    }
    
    private async Task LoadProductAsync(int id)
    {
        // Load from service
        await Task.Delay(100);
        // Product = await _productService.GetByIdAsync(id);
    }
}

// Extension method
public static class TaskExtensions
{
    public static void FireAndForget(this Task task, Action<Exception>? onError = null)
    {
        _ = task.ContinueWith(t =>
        {
            if (t.IsFaulted)
                onError?.Invoke(t.Exception!.GetBaseException());
        });
    }
}

// Placeholder page classes
public class ProductsPage2 : ContentPage { }
public class ProductDetailPage : ContentPage { }
public class AddProductPage : ContentPage { }
public class ProfilePage : ContentPage { }
public class SettingsPage2 : ContentPage { }
public class HomePage : ContentPage { }
```

---

## Step 177: CollectionView

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- CollectionView Examples -->
<!-- ============================================ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:Class="MyApp.Views.ProductsPage2"
             Title="สินค้า">
    
    <ContentPage.BindingContext>
        <vm:ProductsViewModel2 />
    </ContentPage.BindingContext>
    
    <Grid RowDefinitions="Auto,*">
        
        <!-- Search bar -->
        <SearchBar Grid.Row="0"
                   Text="{Binding SearchText}"
                   Placeholder="ค้นหาสินค้า..." />
        
        <!-- Products list -->
        <RefreshView Grid.Row="1"
                     Command="{Binding LoadProductsCommand}"
                     IsRefreshing="{Binding IsRefreshing}">
            
            <CollectionView ItemsSource="{Binding FilteredProducts}"
                            SelectionMode="None"
                            ItemsUpdatingScrollMode="MaintainScrollOffset">
                
                <!-- Header -->
                <CollectionView.Header>
                    <Label Text="{Binding FilteredProducts.Count, StringFormat='สินค้า {0} รายการ'}"
                           Padding="15,10"
                           TextColor="Gray" />
                </CollectionView.Header>
                
                <!-- Empty state -->
                <CollectionView.EmptyView>
                    <VerticalStackLayout HorizontalOptions="Center" VerticalOptions="Center"
                                        Spacing="10" Margin="0,50">
                        <Image Source="empty.png" HeightRequest="100" />
                        <Label Text="ไม่พบสินค้า" HorizontalOptions="Center" 
                               FontSize="18" TextColor="Gray" />
                    </VerticalStackLayout>
                </CollectionView.EmptyView>
                
                <!-- Item Template -->
                <CollectionView.ItemTemplate>
                    <DataTemplate x:DataType="vm:ProductItem">
                        <Grid Padding="15,8">
                            <Frame CornerRadius="12" Padding="12" HasShadow="True">
                                <Grid ColumnDefinitions="80,*,Auto" ColumnSpacing="12">
                                    
                                    <!-- Image -->
                                    <Image Grid.Column="0"
                                           Source="product_placeholder.png"
                                           HeightRequest="70"
                                           Aspect="AspectFit" />
                                    
                                    <!-- Info -->
                                    <VerticalStackLayout Grid.Column="1" VerticalOptions="Center" Spacing="4">
                                        <Label Text="{Binding Name}"
                                               FontSize="16"
                                               FontAttributes="Bold" />
                                        <Label Text="{Binding Category}"
                                               FontSize="13"
                                               TextColor="Gray" />
                                        <Label Text="{Binding Price, StringFormat='฿{0:N0}'}"
                                               FontSize="15"
                                               TextColor="#512BD4" />
                                    </VerticalStackLayout>
                                    
                                    <!-- Actions -->
                                    <VerticalStackLayout Grid.Column="2" VerticalOptions="Center" Spacing="4">
                                        <Label Text="{Binding StockText}"
                                               FontSize="12"
                                               TextColor="{Binding Stock, Converter={StaticResource StockColor}}" />
                                        <Button Text="ลบ"
                                                FontSize="12"
                                                HeightRequest="35"
                                                Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.DeleteProductCommand}"
                                                CommandParameter="{Binding .}" />
                                    </VerticalStackLayout>
                                    
                                </Grid>
                            </Frame>
                            
                            <!-- Tap gesture -->
                            <Grid.GestureRecognizers>
                                <TapGestureRecognizer 
                                    Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.SelectProductCommand}"
                                    CommandParameter="{Binding .}" />
                            </Grid.GestureRecognizers>
                        </Grid>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
                
                <!-- Grid layout -->
                <CollectionView.ItemsLayout>
                    <LinearItemsLayout Orientation="Vertical" ItemSpacing="0" />
                    <!-- หรือ Grid layout:
                    <GridItemsLayout Orientation="Vertical" Span="2" VerticalItemSpacing="10" HorizontalItemSpacing="10" />
                    -->
                </CollectionView.ItemsLayout>
                
            </CollectionView>
        </RefreshView>
        
    </Grid>
    
</ContentPage>
```

---

## Step 178: Forms และ Validation

```csharp
// ============================================
// Form Validation
// ============================================

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.ComponentModel.DataAnnotations;

public partial class RegistrationViewModel : ObservableValidator
{
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกชื่อ")]
    [MinLength(2, ErrorMessage = "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร")]
    private string _firstName = string.Empty;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกนามสกุล")]
    private string _lastName = string.Empty;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกอีเมล")]
    [EmailAddress(ErrorMessage = "รูปแบบอีเมลไม่ถูกต้อง")]
    private string _email = string.Empty;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณากรอกรหัสผ่าน")]
    [MinLength(8, ErrorMessage = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")]
    private string _password = string.Empty;
    
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "กรุณายืนยันรหัสผ่าน")]
    [CustomValidation(typeof(RegistrationViewModel), nameof(ValidatePasswordConfirm))]
    private string _confirmPassword = string.Empty;
    
    [ObservableProperty]
    private bool _isLoading;
    
    [ObservableProperty]
    private string _errorMessage = string.Empty;
    
    // Custom validation
    public static ValidationResult? ValidatePasswordConfirm(string confirm, 
        ValidationContext context)
    {
        var vm = (RegistrationViewModel)context.ObjectInstance;
        if (confirm != vm.Password)
            return new ValidationResult("รหัสผ่านไม่ตรงกัน");
        return ValidationResult.Success;
    }
    
    public bool IsValid => !HasErrors;
    
    [RelayCommand]
    private async Task RegisterAsync()
    {
        // Trigger all validations
        ValidateAllProperties();
        
        if (HasErrors)
        {
            ErrorMessage = "กรุณาแก้ไขข้อผิดพลาดก่อนดำเนินการ";
            return;
        }
        
        IsLoading = true;
        ErrorMessage = string.Empty;
        
        try
        {
            await Task.Delay(1500); // simulate API call
            
            // Navigate on success
            await Shell.Current.GoToAsync("//main");
        }
        catch (Exception ex)
        {
            ErrorMessage = $"เกิดข้อผิดพลาด: {ex.Message}";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    // Helper to get first error for a property
    public string GetFirstError(string propertyName)
    {
        var errors = GetErrors(propertyName);
        return errors.FirstOrDefault()?.ErrorMessage ?? string.Empty;
    }
}
```

---

## Step 179: Messaging

```csharp
// ============================================
// WeakReferenceMessenger (CommunityToolkit)
// ============================================

using CommunityToolkit.Mvvm.Messaging;
using CommunityToolkit.Mvvm.Messaging.Messages;

// Messages
public record ProductAddedMessage(ProductItem Product);
public record ProductDeletedMessage(int ProductId);
public record CartUpdatedMessage(int ItemCount);
public class RefreshProductsMessage { }

// Sender
public partial class AddProductViewModel : ObservableObject
{
    private readonly IMessenger _messenger;
    
    public AddProductViewModel(IMessenger messenger) => _messenger = messenger;
    
    [RelayCommand]
    private async Task SaveProductAsync()
    {
        var product = new ProductItem(0, "New Product", "Electronics", 1000, 50);
        
        // Notify other ViewModels
        _messenger.Send(new ProductAddedMessage(product));
        
        await Shell.Current.GoToAsync("..");
    }
}

// Receiver
public partial class ProductsViewModel3 : ObservableObject,
    IRecipient<ProductAddedMessage>,
    IRecipient<ProductDeletedMessage>
{
    private readonly IMessenger _messenger;
    
    [ObservableProperty]
    private ObservableCollection<ProductItem> _products = new();
    
    public ProductsViewModel3(IMessenger messenger)
    {
        _messenger = messenger;
        messenger.RegisterAll(this); // register all IRecipient<T> interfaces
    }
    
    public void Receive(ProductAddedMessage message)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            Products.Add(message.Product);
        });
    }
    
    public void Receive(ProductDeletedMessage message)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            var product = Products.FirstOrDefault(p => p.Id == message.ProductId);
            if (product != null)
                Products.Remove(product);
        });
    }
    
    // Unsubscribe when done
    public void Dispose()
    {
        _messenger.UnregisterAll(this);
    }
}

// ============================================
// ValueChangedMessage
// ============================================

public partial class ThemeViewModel : ObservableObject
{
    private readonly IMessenger _messenger;
    
    [ObservableProperty]
    private bool _isDarkMode;
    
    public ThemeViewModel(IMessenger messenger) => _messenger = messenger;
    
    partial void OnIsDarkModeChanged(bool value)
    {
        _messenger.Send(new ValueChangedMessage<bool>(value));
        Application.Current!.UserAppTheme = value ? AppTheme.Dark : AppTheme.Light;
    }
}

public class AnyViewModel : IRecipient<ValueChangedMessage<bool>>
{
    public AnyViewModel(IMessenger messenger)
    {
        messenger.Register<AnyViewModel, ValueChangedMessage<bool>>(this, 
            (recipient, message) =>
            {
                Console.WriteLine($"Dark mode: {message.Value}");
            });
    }
    
    public void Receive(ValueChangedMessage<bool> message)
        => Console.WriteLine($"Theme changed: {message.Value}");
}
```

---

## Step 180: Complete MVVM Example

```csharp
// ============================================
// Complete: Todo App ViewModel
// ============================================

public record TodoItem(int Id, string Title, bool IsCompleted, DateTime CreatedAt);

public partial class TodoViewModel : ObservableObject
{
    [ObservableProperty]
    private ObservableCollection<TodoItem> _todos = new();
    
    [ObservableProperty]
    private string _newTodoTitle = string.Empty;
    
    [ObservableProperty]
    private bool _showCompleted = true;
    
    [ObservableProperty]
    private bool _isLoading;
    
    private int _nextId = 1;
    private List<TodoItem> _allTodos = new();
    
    public int TotalCount => _allTodos.Count;
    public int CompletedCount => _allTodos.Count(t => t.IsCompleted);
    public int PendingCount => _allTodos.Count(t => !t.IsCompleted);
    
    partial void OnShowCompletedChanged(bool value) => RefreshTodos();
    
    [RelayCommand]
    private async Task LoadTodosAsync()
    {
        IsLoading = true;
        await Task.Delay(500);
        
        _allTodos = new List<TodoItem>
        {
            new(_nextId++, "เรียน C#", true, DateTime.Now.AddDays(-3)),
            new(_nextId++, "เรียน MAUI", false, DateTime.Now.AddDays(-2)),
            new(_nextId++, "สร้าง App แรก", false, DateTime.Now.AddDays(-1)),
            new(_nextId++, "ทำ Unit Tests", false, DateTime.Now),
        };
        
        RefreshTodos();
        NotifyStats();
        IsLoading = false;
    }
    
    [RelayCommand(CanExecute = nameof(CanAddTodo))]
    private void AddTodo()
    {
        if (string.IsNullOrWhiteSpace(NewTodoTitle)) return;
        
        var todo = new TodoItem(_nextId++, NewTodoTitle.Trim(), false, DateTime.Now);
        _allTodos.Add(todo);
        NewTodoTitle = string.Empty;
        RefreshTodos();
        NotifyStats();
    }
    
    private bool CanAddTodo() => !string.IsNullOrWhiteSpace(NewTodoTitle);
    
    partial void OnNewTodoTitleChanged(string value)
        => AddTodoCommand.NotifyCanExecuteChanged();
    
    [RelayCommand]
    private void ToggleTodo(TodoItem todo)
    {
        int idx = _allTodos.FindIndex(t => t.Id == todo.Id);
        if (idx >= 0)
        {
            _allTodos[idx] = todo with { IsCompleted = !todo.IsCompleted };
            RefreshTodos();
            NotifyStats();
        }
    }
    
    [RelayCommand]
    private void DeleteTodo(TodoItem todo)
    {
        _allTodos.RemoveAll(t => t.Id == todo.Id);
        RefreshTodos();
        NotifyStats();
    }
    
    [RelayCommand]
    private void ClearCompleted()
    {
        _allTodos.RemoveAll(t => t.IsCompleted);
        RefreshTodos();
        NotifyStats();
    }
    
    private void RefreshTodos()
    {
        var filtered = ShowCompleted
            ? _allTodos
            : _allTodos.Where(t => !t.IsCompleted);
        
        Todos = new ObservableCollection<TodoItem>(
            filtered.OrderBy(t => t.IsCompleted).ThenByDescending(t => t.CreatedAt));
    }
    
    private void NotifyStats()
    {
        OnPropertyChanged(nameof(TotalCount));
        OnPropertyChanged(nameof(CompletedCount));
        OnPropertyChanged(nameof(PendingCount));
    }
}
```

---

## สรุป Part 18

ใน Part 18 เราได้เรียนรู้:

1. **MVVM Pattern** - Model, View, ViewModel roles
2. **CommunityToolkit.Mvvm** - [ObservableProperty], [RelayCommand]
3. **Data Binding** - Modes, StringFormat, Converters
4. **Value Converters** - IValueConverter, Multi binding
5. **ObservableCollection** - Collection binding
6. **Shell Navigation** - Routes, Parameters
7. **CollectionView** - Lists, Grid, Empty state
8. **Forms Validation** - DataAnnotations, ObservableValidator
9. **Messaging** - WeakReferenceMessenger
10. **Complete Todo App** - Full MVVM example

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 18 | Steps 171-180*
