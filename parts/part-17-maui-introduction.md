# Part 17: .NET MAUI Introduction
## Steps 161-170: เริ่มต้น Cross-platform Development

---

## Step 161: .NET MAUI คืออะไร

```
// ============================================
// .NET MAUI Overview
// ============================================

/*
 * MAUI = Multi-platform App UI
 * 
 * ทำงานบน:
 * - Android (4.4+)
 * - iOS (11+)  
 * - macOS (10.15+)
 * - Windows (10 build 1903+)
 * 
 * เขียนครั้งเดียว รันได้ทุก platform
 * 
 * เทียบกับ Xamarin.Forms:
 * - Performance ดีกว่า (single project)
 * - Hot Reload ดีกว่า
 * - .NET 6+ เท่านั้น
 * - API เดียวกัน แต่ปรับปรุงแล้ว
 */

/*
 * Architecture:
 * 
 * ┌─────────────────────────────────┐
 * │          .NET MAUI App          │  ← เขียนครั้งเดียว
 * │    (C# + XAML / C# only)        │
 * ├─────────────────────────────────┤
 * │         .NET MAUI Layer          │  ← Cross-platform APIs
 * ├──────────┬──────────┬───────────┤
 * │ Android  │   iOS    │  Windows  │  ← Platform-specific
 * │   .NET   │  .NET    │   .NET    │
 * └──────────┴──────────┴───────────┘
 */
```

---

## Step 162: ติดตั้งและสร้าง Project

```bash
# ============================================
# ติดตั้ง .NET MAUI Workload
# ============================================

# ติดตั้ง .NET 8 SDK
# https://dotnet.microsoft.com/download

# ติดตั้ง MAUI workload
dotnet workload install maui

# ตรวจสอบ dependencies
dotnet workload list

# สำหรับ Android: ต้องมี Java SDK
# สำหรับ iOS/macOS: ต้องมี Xcode (Mac เท่านั้น)

# ============================================
# สร้าง MAUI Project
# ============================================

# สร้าง project ใหม่
dotnet new maui -n MyApp

# สร้างตาม template ต่างๆ
dotnet new maui-blazor -n MyBlazorApp  # MAUI + Blazor
dotnet new maui-lib -n MyLibrary       # MAUI class library

# Build สำหรับ platform ต่างๆ
dotnet build -f net8.0-android
dotnet build -f net8.0-ios
dotnet build -f net8.0-windows10.0.19041.0
dotnet build -f net8.0-maccatalyst

# Run บน Android emulator
dotnet run -f net8.0-android

# Run บน Windows
dotnet run -f net8.0-windows10.0.19041.0
```

---

## Step 163: โครงสร้าง MAUI Project

```
// ============================================
// Project Structure
// ============================================

/*
 MyApp/
 ├── Platforms/                    ← Platform-specific code
 │   ├── Android/
 │   │   ├── AndroidManifest.xml
 │   │   ├── MainApplication.cs
 │   │   └── MainActivity.cs
 │   ├── iOS/
 │   │   ├── AppDelegate.cs
 │   │   ├── Info.plist
 │   │   └── Program.cs
 │   ├── MacCatalyst/
 │   └── Windows/
 │       └── App.xaml
 │
 ├── Resources/                   ← Shared resources
 │   ├── AppIcon/
 │   │   ├── appicon.svg          ← App icon (auto-resize)
 │   │   └── appiconfg.svg
 │   ├── Fonts/
 │   │   └── OpenSans-Regular.ttf
 │   ├── Images/
 │   │   └── dotnet_bot.png
 │   ├── Raw/                     ← Raw files (JSON, etc.)
 │   └── Styles/
 │       ├── Colors.xaml
 │       └── Styles.xaml
 │
 ├── Views/                       ← Pages (XAML + code-behind)
 │   └── MainPage.xaml
 │   └── MainPage.xaml.cs
 │
 ├── ViewModels/                  ← Business logic
 │   └── MainViewModel.cs
 │
 ├── Models/                      ← Data models
 │
 ├── Services/                    ← Services layer
 │
 ├── App.xaml                     ← App resources/styles
 ├── App.xaml.cs                  ← App lifecycle
 ├── AppShell.xaml                ← Navigation shell
 ├── AppShell.xaml.cs
 └── MauiProgram.cs               ← Startup/DI configuration
 */
```

---

## Step 164: MauiProgram.cs

```csharp
// ============================================
// MauiProgram.cs - Entry Point & DI Setup
// ============================================

using Microsoft.Extensions.Logging;
using Microsoft.Maui.Controls.Hosting;
using Microsoft.Maui.Hosting;
using CommunityToolkit.Maui;

namespace MyApp;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            // Community Toolkit
            .UseMauiCommunityToolkit()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
                fonts.AddFont("MaterialIcons-Regular.ttf", "MaterialIcons");
            })
            .ConfigureMauiHandlers(handlers =>
            {
                // Register custom handlers
                // handlers.AddHandler<MyControl, MyControlHandler>();
            });
        
        // Register Services
        builder.Services
            // HTTP
            .AddHttpClient<WeatherService>(client =>
            {
                client.BaseAddress = new Uri("https://api.weather.example.com/");
                client.Timeout = TimeSpan.FromSeconds(30);
            })
            
            // Singletons (1 instance for app lifetime)
            .AddSingleton<INavigationService, NavigationService>()
            .AddSingleton<ISettingsService, SettingsService>()
            .AddSingleton<IConnectivityService, ConnectivityService>()
            
            // Transients (new instance every time)
            .AddTransient<MainPage>()
            .AddTransient<MainViewModel>()
            .AddTransient<ProductsPage>()
            .AddTransient<ProductsViewModel>()
            .AddTransient<DetailPage>()
            .AddTransient<DetailViewModel>()
            
            // Database
            .AddSingleton<LocalDatabase>();
        
#if DEBUG
        builder.Logging.AddDebug();
#endif
        
        return builder.Build();
    }
}

// Placeholder classes (will be implemented in later parts)
public class App : Application
{
    public App(AppShell shell) => MainPage = shell;
}

public class AppShell : Shell
{
    public AppShell()
    {
        // Setup routes
        Routing.RegisterRoute("products", typeof(ProductsPage));
        Routing.RegisterRoute("product-detail", typeof(DetailPage));
    }
}

public class WeatherService { public WeatherService(HttpClient client) { } }
public class NavigationService : INavigationService { }
public class SettingsService : ISettingsService { }
public class ConnectivityService : IConnectivityService { }
public class LocalDatabase { }
public class ProductsPage : ContentPage { }
public class ProductsViewModel { }
public class DetailPage : ContentPage { }
public class DetailViewModel { }
public class MainPage : ContentPage { }
public class MainViewModel { }

public interface INavigationService { }
public interface ISettingsService { }
public interface IConnectivityService { }
```

---

## Step 165: App Lifecycle

```csharp
// ============================================
// App.xaml.cs - Application Lifecycle
// ============================================

namespace MyApp;

public partial class App2 : Application
{
    private readonly ISettingsService _settings;
    
    public App2(AppShell shell, ISettingsService settings) 
    {
        InitializeComponent();
        _settings = settings;
        MainPage = shell;
    }
    
    // App starting for the first time
    protected override void OnStart()
    {
        base.OnStart();
        Console.WriteLine("App started");
        // Load settings, check authentication, etc.
    }
    
    // App coming back to foreground
    protected override void OnResume()
    {
        base.OnResume();
        Console.WriteLine("App resumed");
        // Refresh data, restart timers
    }
    
    // App going to background
    protected override void OnSleep()
    {
        base.OnSleep();
        Console.WriteLine("App sleeping");
        // Save state, stop timers, cancel operations
    }
}

// ============================================
// Window Lifecycle (MAUI 8+)
// ============================================

/*
 * Lifecycle order:
 * 
 * Created → Activated → Deactivated → Stopped → Resumed → Stopped → Destroyed
 * 
 * Android: OnCreate → OnStart → OnResume → OnPause → OnStop → OnRestart
 * iOS:     ViewDidLoad → ViewWillAppear → ViewDidAppear → ViewWillDisappear
 */

// Register lifecycle events
public class LifecycleHandler
{
    public static void RegisterHandlers(MauiAppBuilder builder)
    {
        builder.ConfigureLifecycleEvents(events =>
        {
#if ANDROID
            events.AddAndroid(android => android
                .OnCreate((activity, bundle) => Console.WriteLine("Android Created"))
                .OnStart(activity => Console.WriteLine("Android Started"))
                .OnResume(activity => Console.WriteLine("Android Resumed"))
                .OnPause(activity => Console.WriteLine("Android Paused"))
                .OnStop(activity => Console.WriteLine("Android Stopped"))
                .OnDestroy(activity => Console.WriteLine("Android Destroyed"))
            );
#elif IOS
            events.AddiOS(ios => ios
                .FinishedLaunching((app, options) => { Console.WriteLine("iOS Launched"); return true; })
                .WillEnterForeground(app => Console.WriteLine("iOS Foreground"))
                .DidEnterBackground(app => Console.WriteLine("iOS Background"))
                .WillTerminate(app => Console.WriteLine("iOS Terminating"))
            );
#endif
        });
    }
}
```

---

## Step 166: XAML พื้นฐาน

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- MainPage.xaml - UI Layout -->
<!-- ============================================ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:Class="MyApp.Views.MainPage"
             Title="หน้าหลัก">

    <!-- BindingContext: ViewModel -->
    <ContentPage.BindingContext>
        <vm:MainViewModel2 />
    </ContentPage.BindingContext>

    <!-- เนื้อหา -->
    <ScrollView>
        <VerticalStackLayout Spacing="20" Padding="20">
            
            <!-- Label -->
            <Label Text="{Binding Title}"
                   FontSize="24"
                   FontAttributes="Bold"
                   HorizontalOptions="Center"
                   TextColor="{StaticResource Primary}" />

            <!-- Image -->
            <Image Source="dotnet_bot.png"
                   HeightRequest="200"
                   HorizontalOptions="Center" />

            <!-- Entry (Text Input) -->
            <Entry Placeholder="กรอกชื่อของคุณ"
                   Text="{Binding UserName}"
                   Keyboard="Text"
                   ReturnType="Done"
                   ReturnCommand="{Binding GreetCommand}" />

            <!-- Button -->
            <Button Text="{Binding ButtonText}"
                    Command="{Binding GreetCommand}"
                    BackgroundColor="{StaticResource Primary}"
                    TextColor="White"
                    CornerRadius="10"
                    HeightRequest="50" />

            <!-- ActivityIndicator -->
            <ActivityIndicator IsRunning="{Binding IsLoading}"
                               IsVisible="{Binding IsLoading}"
                               Color="{StaticResource Primary}" />

            <!-- Greeting message -->
            <Label Text="{Binding Greeting}"
                   FontSize="18"
                   HorizontalOptions="Center"
                   IsVisible="{Binding HasGreeting}" />

            <!-- Switch -->
            <HorizontalStackLayout Spacing="10">
                <Label Text="แจ้งเตือน:" VerticalOptions="Center" />
                <Switch IsToggled="{Binding NotificationsEnabled}" />
            </HorizontalStackLayout>

            <!-- Slider -->
            <VerticalStackLayout>
                <Label Text="{Binding FontSizeText}" />
                <Slider Value="{Binding FontSize}"
                        Minimum="10" Maximum="30"
                        MinimumTrackColor="{StaticResource Primary}" />
            </VerticalStackLayout>

            <!-- CollectionView -->
            <CollectionView ItemsSource="{Binding Items}"
                            HeightRequest="200">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <HorizontalStackLayout Padding="0,5" Spacing="10">
                            <Label Text="{Binding .}"
                                   FontSize="16"
                                   VerticalOptions="Center" />
                        </HorizontalStackLayout>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>

        </VerticalStackLayout>
    </ScrollView>

</ContentPage>
```

---

## Step 167: Code-Behind และ ViewModel

```csharp
// ============================================
// MainPage.xaml.cs - Code Behind
// ============================================

namespace MyApp.Views;

public partial class MainPage3 : ContentPage
{
    public MainPage3(MainViewModel2 viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
    
    protected override void OnAppearing()
    {
        base.OnAppearing();
        // Page is visible to user
        if (BindingContext is MainViewModel2 vm)
            vm.OnAppearing();
    }
    
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        // Page going off screen
    }
}

// ============================================
// MainViewModel.cs
// ============================================

using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Windows.Input;

namespace MyApp.ViewModels;

public class MainViewModel2 : INotifyPropertyChanged
{
    // ============================================
    // INotifyPropertyChanged Implementation
    // ============================================
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    
    protected bool SetField<T>(ref T field, T value, [CallerMemberName] string? name = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(name);
        return true;
    }
    
    // ============================================
    // Properties
    // ============================================
    
    private string _title = "สวัสดี MAUI!";
    public string Title
    {
        get => _title;
        set => SetField(ref _title, value);
    }
    
    private string _userName = string.Empty;
    public string UserName
    {
        get => _userName;
        set
        {
            if (SetField(ref _userName, value))
                OnPropertyChanged(nameof(ButtonText));
        }
    }
    
    private string _greeting = string.Empty;
    public string Greeting
    {
        get => _greeting;
        set
        {
            if (SetField(ref _greeting, value))
                OnPropertyChanged(nameof(HasGreeting));
        }
    }
    
    public bool HasGreeting => !string.IsNullOrEmpty(Greeting);
    
    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set => SetField(ref _isLoading, value);
    }
    
    private bool _notificationsEnabled = true;
    public bool NotificationsEnabled
    {
        get => _notificationsEnabled;
        set => SetField(ref _notificationsEnabled, value);
    }
    
    private double _fontSize = 16;
    public double FontSize
    {
        get => _fontSize;
        set
        {
            if (SetField(ref _fontSize, value))
                OnPropertyChanged(nameof(FontSizeText));
        }
    }
    
    public string FontSizeText => $"ขนาดตัวอักษร: {FontSize:F0}";
    
    public string ButtonText => string.IsNullOrEmpty(UserName) 
        ? "ทักทาย" : $"ทักทาย {UserName}";
    
    public ObservableCollection<string> Items { get; } = new()
    {
        "รายการที่ 1",
        "รายการที่ 2",
        "รายการที่ 3"
    };
    
    // ============================================
    // Commands
    // ============================================
    
    public ICommand GreetCommand { get; }
    public ICommand AddItemCommand { get; }
    public ICommand ClearCommand { get; }
    
    public MainViewModel2()
    {
        GreetCommand = new Command(async () => await GreetAsync());
        AddItemCommand = new Command(AddItem);
        ClearCommand = new Command(Clear, () => Items.Count > 0);
    }
    
    private async Task GreetAsync()
    {
        IsLoading = true;
        await Task.Delay(500); // simulate loading
        
        Greeting = string.IsNullOrWhiteSpace(UserName)
            ? "สวัสดีครับ!"
            : $"สวัสดีครับ คุณ{UserName}!";
        
        IsLoading = false;
    }
    
    private int _itemCount = 4;
    private void AddItem()
    {
        Items.Add($"รายการที่ {_itemCount++}");
        (ClearCommand as Command)?.ChangeCanExecute();
    }
    
    private void Clear()
    {
        Items.Clear();
        _itemCount = 1;
        (ClearCommand as Command)?.ChangeCanExecute();
    }
    
    public void OnAppearing()
    {
        // Called when page appears
    }
}
```

---

## Step 168: XAML Layouts

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- MAUI Layout Examples -->
<!-- ============================================ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.Views.LayoutsPage"
             Title="Layouts">
    
    <ScrollView>
        <VerticalStackLayout Spacing="20" Padding="20">
            
            <!-- ============================================ -->
            <!-- VerticalStackLayout -->
            <!-- ============================================ -->
            <Frame BackgroundColor="LightBlue" Padding="10">
                <VerticalStackLayout Spacing="5">
                    <Label Text="Vertical Stack" FontAttributes="Bold" />
                    <Label Text="Item 1" />
                    <Label Text="Item 2" />
                    <Label Text="Item 3" />
                </VerticalStackLayout>
            </Frame>
            
            <!-- ============================================ -->
            <!-- HorizontalStackLayout -->
            <!-- ============================================ -->
            <Frame BackgroundColor="LightGreen" Padding="10">
                <HorizontalStackLayout Spacing="10">
                    <Label Text="H-Stack:" FontAttributes="Bold" />
                    <Label Text="A" />
                    <Label Text="B" />
                    <Label Text="C" />
                </HorizontalStackLayout>
            </Frame>
            
            <!-- ============================================ -->
            <!-- Grid -->
            <!-- ============================================ -->
            <Frame BackgroundColor="LightYellow" Padding="10">
                <Grid RowDefinitions="Auto,Auto,Auto" 
                      ColumnDefinitions="*,*"
                      RowSpacing="5" ColumnSpacing="5">
                    
                    <Label Text="Grid" FontAttributes="Bold" 
                           Grid.Row="0" Grid.Column="0" Grid.ColumnSpan="2"
                           HorizontalOptions="Center" />
                    
                    <Label Text="[0,0]" Grid.Row="1" Grid.Column="0"
                           BackgroundColor="LightCoral" HorizontalOptions="Fill"
                           HorizontalTextAlignment="Center" />
                    <Label Text="[0,1]" Grid.Row="1" Grid.Column="1"
                           BackgroundColor="LightSalmon" HorizontalOptions="Fill"
                           HorizontalTextAlignment="Center" />
                    <Label Text="[1,0] colspan=2" Grid.Row="2" Grid.Column="0" 
                           Grid.ColumnSpan="2"
                           BackgroundColor="Khaki" HorizontalOptions="Fill"
                           HorizontalTextAlignment="Center" />
                </Grid>
            </Frame>
            
            <!-- ============================================ -->
            <!-- AbsoluteLayout -->
            <!-- ============================================ -->
            <Frame BackgroundColor="Lavender" Padding="10" HeightRequest="150">
                <AbsoluteLayout>
                    <BoxView Color="Red" 
                             AbsoluteLayout.LayoutBounds="0,0,100,50"
                             AbsoluteLayout.LayoutFlags="None" />
                    <BoxView Color="Blue"
                             AbsoluteLayout.LayoutBounds="0.5,0.5,80,80"
                             AbsoluteLayout.LayoutFlags="PositionProportional" />
                    <Label Text="Overlaid!"
                           AbsoluteLayout.LayoutBounds="0.5,0.5,-1,-1"
                           AbsoluteLayout.LayoutFlags="PositionProportional" />
                </AbsoluteLayout>
            </Frame>
            
            <!-- ============================================ -->
            <!-- FlexLayout -->
            <!-- ============================================ -->
            <Frame BackgroundColor="MistyRose" Padding="10">
                <FlexLayout Wrap="Wrap" JustifyContent="SpaceEvenly">
                    <Button Text="A" WidthRequest="70" />
                    <Button Text="B" WidthRequest="70" />
                    <Button Text="C" WidthRequest="70" />
                    <Button Text="D" WidthRequest="70" />
                    <Button Text="E" WidthRequest="70" />
                </FlexLayout>
            </Frame>
            
        </VerticalStackLayout>
    </ScrollView>
    
</ContentPage>
```

---

## Step 169: XAML Controls

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.Views.ControlsPage"
             Title="Controls">
    
    <ScrollView>
        <VerticalStackLayout Spacing="15" Padding="20">
            
            <!-- Text Controls -->
            <Label Text="Label: ป้ายข้อความ"
                   FontSize="16"
                   TextColor="DarkBlue"
                   FontAttributes="Bold,Italic" />
            
            <Entry Placeholder="Entry: กล่องข้อความ"
                   Keyboard="Email"
                   ClearButtonVisibility="WhileEditing" />
            
            <Editor Placeholder="Editor: กล่องข้อความหลายบรรทัด"
                    HeightRequest="100"
                    AutoSize="TextChanges" />
            
            <SearchBar Placeholder="SearchBar: ค้นหา..." />
            
            <!-- Buttons -->
            <Button Text="Button ปกติ"
                    BackgroundColor="DodgerBlue"
                    TextColor="White" />
            
            <ImageButton Source="dotnet_bot.png"
                         HeightRequest="50"
                         WidthRequest="50"
                         HorizontalOptions="Start" />
            
            <!-- Selection -->
            <Picker Title="เลือก Major" x:Name="MajorPicker">
                <Picker.ItemsSource>
                    <x:Array Type="{x:Type x:String}">
                        <x:String>Computer Science</x:String>
                        <x:String>Information Technology</x:String>
                        <x:String>Mathematics</x:String>
                    </x:Array>
                </Picker.ItemsSource>
            </Picker>
            
            <DatePicker Format="yyyy-MM-dd" Date="{x:Static sys:DateTime.Today}"
                        xmlns:sys="clr-namespace:System;assembly=mscorlib" />
            
            <TimePicker Format="HH:mm" />
            
            <!-- Toggles -->
            <HorizontalStackLayout Spacing="10">
                <Label Text="Dark Mode:" VerticalOptions="Center" />
                <Switch x:Name="DarkModeSwitch" />
            </HorizontalStackLayout>
            
            <CheckBox IsChecked="True" Color="Green" />
            
            <RadioButton Content="ตัวเลือก 1" GroupName="Options" />
            <RadioButton Content="ตัวเลือก 2" GroupName="Options" IsChecked="True" />
            
            <!-- Progress -->
            <Slider x:Name="VolumeSlider" 
                    Minimum="0" Maximum="100" Value="50"
                    ThumbColor="Purple" />
            
            <ProgressBar Progress="0.6"
                         ProgressColor="Green" />
            
            <Stepper Minimum="1" Maximum="10" Increment="1" Value="5" />
            
            <!-- Container -->
            <Frame BackgroundColor="AliceBlue" 
                   CornerRadius="10"
                   BorderColor="LightBlue"
                   Padding="15">
                <Label Text="Frame container" />
            </Frame>
            
            <Border StrokeShape="RoundRectangle 10"
                    BackgroundColor="Honeydew"
                    Padding="15">
                <Label Text="Border container (MAUI 7+)" />
            </Border>
            
            <!-- WebView -->
            <WebView Source="https://dotnet.microsoft.com" HeightRequest="200" />

        </VerticalStackLayout>
    </ScrollView>
    
</ContentPage>
```

---

## Step 170: Resources และ Styles

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- ============================================ -->
<!-- App.xaml - Global Resources -->
<!-- ============================================ -->
<Application xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.App">
    
    <Application.Resources>
        <ResourceDictionary>
            
            <!-- Merge other dictionaries -->
            <ResourceDictionary.MergedDictionaries>
                <ResourceDictionary Source="Resources/Styles/Colors.xaml" />
                <ResourceDictionary Source="Resources/Styles/Styles.xaml" />
            </ResourceDictionary.MergedDictionaries>
            
            <!-- App-wide Resources -->
            <x:Double x:Key="DefaultFontSize">16</x:Double>
            <x:Double x:Key="TitleFontSize">24</x:Double>
            <x:Double x:Key="HeaderFontSize">20</x:Double>
            
            <x:String x:Key="AppName">My MAUI App</x:String>
            
            <Thickness x:Key="PagePadding">20</Thickness>
            <Thickness x:Key="CardPadding">15,10</Thickness>
            
        </ResourceDictionary>
    </Application.Resources>
    
</Application>
```

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Resources/Styles/Colors.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">
    
    <!-- Brand Colors -->
    <Color x:Key="Primary">#512BD4</Color>
    <Color x:Key="PrimaryDark">#2B0B98</Color>
    <Color x:Key="PrimaryLight">#A07BF5</Color>
    <Color x:Key="Secondary">#DFD8F7</Color>
    <Color x:Key="SecondaryDark">#9880E5</Color>
    <Color x:Key="Tertiary">#2B0B98</Color>
    
    <!-- Semantic Colors -->
    <Color x:Key="Success">#28a745</Color>
    <Color x:Key="Warning">#ffc107</Color>
    <Color x:Key="Danger">#dc3545</Color>
    <Color x:Key="Info">#17a2b8</Color>
    
    <!-- Light Theme -->
    <Color x:Key="Background">White</Color>
    <Color x:Key="Surface">#F5F5F5</Color>
    <Color x:Key="TextPrimary">#212121</Color>
    <Color x:Key="TextSecondary">#757575</Color>
    
    <!-- Dark Theme (used automatically in dark mode) -->
    
</ResourceDictionary>
```

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Resources/Styles/Styles.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">
    
    <!-- Label Styles -->
    <Style x:Key="TitleStyle" TargetType="Label">
        <Setter Property="FontSize" Value="{StaticResource TitleFontSize}" />
        <Setter Property="FontAttributes" Value="Bold" />
        <Setter Property="TextColor" Value="{StaticResource Primary}" />
    </Style>
    
    <Style x:Key="SubtitleStyle" TargetType="Label">
        <Setter Property="FontSize" Value="{StaticResource HeaderFontSize}" />
        <Setter Property="TextColor" Value="{StaticResource TextSecondary}" />
    </Style>
    
    <Style x:Key="BodyStyle" TargetType="Label">
        <Setter Property="FontSize" Value="{StaticResource DefaultFontSize}" />
        <Setter Property="TextColor" Value="{StaticResource TextPrimary}" />
    </Style>
    
    <!-- Button Styles -->
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource Primary}" />
        <Setter Property="TextColor" Value="White" />
        <Setter Property="CornerRadius" Value="10" />
        <Setter Property="HeightRequest" Value="50" />
        <Setter Property="FontAttributes" Value="Bold" />
    </Style>
    
    <Style x:Key="OutlineButton" TargetType="Button">
        <Setter Property="BackgroundColor" Value="Transparent" />
        <Setter Property="TextColor" Value="{StaticResource Primary}" />
        <Setter Property="BorderColor" Value="{StaticResource Primary}" />
        <Setter Property="BorderWidth" Value="1.5" />
        <Setter Property="CornerRadius" Value="10" />
        <Setter Property="HeightRequest" Value="50" />
    </Style>
    
    <!-- Card Style -->
    <Style x:Key="CardFrame" TargetType="Frame">
        <Setter Property="BackgroundColor" Value="{StaticResource Surface}" />
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="BorderColor" Value="Transparent" />
        <Setter Property="Padding" Value="{StaticResource CardPadding}" />
        <Setter Property="HasShadow" Value="True" />
    </Style>
    
    <!-- Entry Style -->
    <Style x:Key="ModernEntry" TargetType="Entry">
        <Setter Property="BackgroundColor" Value="{StaticResource Surface}" />
        <Setter Property="TextColor" Value="{StaticResource TextPrimary}" />
        <Setter Property="PlaceholderColor" Value="{StaticResource TextSecondary}" />
        <Setter Property="HeightRequest" Value="50" />
    </Style>
    
</ResourceDictionary>
```

```csharp
// ============================================
// ใช้ Styles ใน XAML
// ============================================

/*
<Label Text="Title" Style="{StaticResource TitleStyle}" />
<Label Text="Subtitle" Style="{StaticResource SubtitleStyle}" />
<Label Text="Body text" Style="{StaticResource BodyStyle}" />

<Button Text="Primary" Style="{StaticResource PrimaryButton}" />
<Button Text="Outline" Style="{StaticResource OutlineButton}" />

<Frame Style="{StaticResource CardFrame}">
    <Label Text="Card content" />
</Frame>
*/
```

---

## สรุป Part 17

ใน Part 17 เราได้เรียนรู้:

1. **.NET MAUI Overview** - Cross-platform, Architecture
2. **Installation** - dotnet workload, project creation
3. **Project Structure** - Platforms, Resources, Views
4. **MauiProgram.cs** - DI setup, services registration
5. **App Lifecycle** - Start, Sleep, Resume
6. **XAML Basics** - Page structure, bindings
7. **ViewModel** - INotifyPropertyChanged, Commands
8. **Layouts** - VerticalStack, Grid, AbsoluteLayout, FlexLayout
9. **Controls** - Entry, Button, Picker, Switch, Slider
10. **Resources & Styles** - Colors, Styles, Themes

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 17 | Steps 161-170*
