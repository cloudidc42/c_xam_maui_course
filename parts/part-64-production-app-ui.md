# Part 64: Production App UI – Food Delivery
## Steps 631-640: XAML Pages, Animations, Maps, Reviews

---

## Step 631: Home Page XAML

```xml
<!-- Features/Home/HomePage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyFoodDelivery.Features.Home.HomePage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:MyFoodDelivery.Features.Home"
             xmlns:controls="clr-namespace:MyFoodDelivery.Shared.Controls"
             Shell.NavBarIsVisible="False"
             BackgroundColor="{AppThemeBinding Light=#F8F8F8, Dark=#1C1C1E}">
    
    <RefreshView Command="{Binding RefreshCommand}" IsRefreshing="{Binding IsRefreshing}">
        <ScrollView>
            <VerticalStackLayout>
                
                <!-- Header -->
                <Grid ColumnDefinitions="*,Auto" Padding="16,50,16,16"
                      BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}">
                    <VerticalStackLayout Grid.Column="0">
                        <Label Text="{Binding GreetingText}" FontSize="14"
                               TextColor="{AppThemeBinding Light=#8E8E93, Dark=#AEAEB2}" />
                        <Label Text="{Binding LocationText}" FontSize="16" FontAttributes="Bold"
                               TextColor="{AppThemeBinding Light=#1C1C1E, Dark=White}"
                               MaxLines="1" LineBreakMode="TailTruncation" />
                    </VerticalStackLayout>
                    <ImageButton Grid.Column="1" Source="notification.png"
                                 HeightRequest="44" WidthRequest="44"
                                 BackgroundColor="Transparent" />
                </Grid>
                
                <!-- Search Bar -->
                <Frame Margin="16,12" Padding="12,0"
                       BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#3A3A3C}"
                       CornerRadius="12" HasShadow="False" BorderColor="Transparent">
                    <Grid ColumnDefinitions="Auto,*,Auto">
                        <Image Source="search.png" HeightRequest="20" WidthRequest="20"
                               Margin="0,0,8,0" VerticalOptions="Center" />
                        <Entry Grid.Column="1" Placeholder="ค้นหาร้านอาหาร..."
                               ReturnCommand="{Binding SearchCommand}"
                               ReturnCommandParameter="{Binding Source={RelativeSource Self}, Path=Text}"
                               BackgroundColor="Transparent" />
                    </Grid>
                </Frame>
                
                <!-- Banner Carousel -->
                <CarouselView ItemsSource="{Binding Banners}" HeightRequest="160"
                              Margin="0,8" IndicatorView="BannerIndicator">
                    <CarouselView.ItemTemplate>
                        <DataTemplate>
                            <Frame Margin="16,0" CornerRadius="16" Padding="0" ClipsToBounds="True">
                                <Image Source="{Binding ImageUrl}" Aspect="AspectFill" />
                            </Frame>
                        </DataTemplate>
                    </CarouselView.ItemTemplate>
                </CarouselView>
                <IndicatorView x:Name="BannerIndicator"
                               IndicatorColor="#D1D1D6"
                               SelectedIndicatorColor="#007AFF"
                               Margin="0,8" />
                
                <!-- Categories -->
                <Label Text="หมวดหมู่" FontSize="18" FontAttributes="Bold"
                       Margin="16,8,16,0" />
                <CollectionView ItemsSource="{Binding Categories}"
                                Margin="0,8" HeightRequest="80">
                    <CollectionView.ItemsLayout>
                        <LinearItemsLayout Orientation="Horizontal" ItemSpacing="8" />
                    </CollectionView.ItemsLayout>
                    <CollectionView.ItemTemplate>
                        <DataTemplate>
                            <VerticalStackLayout WidthRequest="72"
                                                 Margin="{Binding IsFirst, Converter={StaticResource FirstItemMarginConverter}, ConverterParameter=16}">
                                <Frame WidthRequest="56" HeightRequest="56"
                                       CornerRadius="28" Padding="0"
                                       BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#3A3A3C}">
                                    <Label Text="{Binding Icon}" FontSize="28"
                                           HorizontalOptions="Center" VerticalOptions="Center" />
                                </Frame>
                                <Label Text="{Binding Name}" FontSize="11"
                                       HorizontalOptions="Center" MaxLines="1" />
                            </VerticalStackLayout>
                        </DataTemplate>
                    </CollectionView.ItemTemplate>
                </CollectionView>
                
                <!-- Nearby Restaurants -->
                <Grid Margin="16,8" ColumnDefinitions="*,Auto">
                    <Label Text="ร้านใกล้คุณ" FontSize="18" FontAttributes="Bold" />
                    <Label Grid.Column="1" Text="ดูทั้งหมด" FontSize="14"
                           TextColor="#007AFF" VerticalOptions="End" />
                </Grid>
                
                <CollectionView ItemsSource="{Binding NearbyRestaurants}" Margin="0,0,0,8"
                                HeightRequest="220">
                    <CollectionView.ItemsLayout>
                        <LinearItemsLayout Orientation="Horizontal" ItemSpacing="12" />
                    </CollectionView.ItemsLayout>
                    <CollectionView.ItemTemplate>
                        <DataTemplate>
                            <controls:RestaurantCard
                                Width="200"
                                Margin="{Binding IsFirst, Converter={StaticResource FirstItemMarginConverter}, ConverterParameter=16}"
                                Command="{Binding Source={RelativeSource AncestorType={x:Type vm:HomeViewModel}}, Path=NavigateToRestaurantCommand}"
                                CommandParameter="{Binding Id}" />
                        </DataTemplate>
                    </CollectionView.ItemTemplate>
                </CollectionView>
                
                <!-- Popular Restaurants -->
                <Label Text="ยอดนิยม" Margin="16,8,16,0"
                       FontSize="18" FontAttributes="Bold" />
                <CollectionView ItemsSource="{Binding PopularRestaurants}" Margin="0,8,0,80">
                    <CollectionView.ItemTemplate>
                        <DataTemplate>
                            <controls:RestaurantListItem
                                Margin="16,0,16,8"
                                Command="{Binding Source={RelativeSource AncestorType={x:Type vm:HomeViewModel}}, Path=NavigateToRestaurantCommand}"
                                CommandParameter="{Binding Id}" />
                        </DataTemplate>
                    </CollectionView.ItemTemplate>
                </CollectionView>
                
            </VerticalStackLayout>
        </ScrollView>
    </RefreshView>
</ContentPage>
```

---

## Step 632: Restaurant Page

```xml
<!-- Features/Restaurant/RestaurantPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyFoodDelivery.Features.Restaurant.RestaurantPage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             Shell.NavBarIsVisible="False">
    
    <Grid RowDefinitions="Auto,*,Auto">
        
        <!-- Hero image with overlay -->
        <Grid Grid.Row="0" HeightRequest="240">
            <Image Source="{Binding Restaurant.ImageUrl}"
                   Aspect="AspectFill" />
            <!-- Gradient overlay -->
            <BoxView>
                <BoxView.Background>
                    <LinearGradientBrush EndPoint="0,1">
                        <GradientStop Color="Transparent" Offset="0.5" />
                        <GradientStop Color="#CC000000" Offset="1.0" />
                    </LinearGradientBrush>
                </BoxView.Background>
            </BoxView>
            <!-- Back button -->
            <ImageButton Source="back.png" BackgroundColor="Transparent"
                         Margin="16,50" WidthRequest="44" HeightRequest="44"
                         HorizontalOptions="Start" VerticalOptions="Start"
                         Command="{Binding GoBackCommand}" />
            <!-- Restaurant info overlay -->
            <VerticalStackLayout VerticalOptions="End" Margin="16,16">
                <Label Text="{Binding Restaurant.Name}"
                       TextColor="White" FontSize="24" FontAttributes="Bold" />
                <HorizontalStackLayout Spacing="8">
                    <Label Text="{Binding Restaurant.Category}"
                           TextColor="#CCFFFFFF" FontSize="14" />
                    <Label Text="•" TextColor="#CCFFFFFF" FontSize="14" />
                    <Image Source="star.png" HeightRequest="14" WidthRequest="14" />
                    <Label Text="{Binding Restaurant.Rating, StringFormat='{0:F1}'}"
                           TextColor="White" FontSize="14" />
                </HorizontalStackLayout>
            </VerticalStackLayout>
        </Grid>
        
        <!-- Delivery info strip -->
        <Grid Grid.Row="1" RowDefinitions="Auto,*">
            <HorizontalStackLayout Grid.Row="0" Spacing="24" Padding="16,12"
                                   BackgroundColor="White">
                <VerticalStackLayout HorizontalOptions="Center">
                    <Label Text="{Binding Restaurant.DeliveryTimeMinutes, StringFormat='{0} นาที'}"
                           FontSize="16" FontAttributes="Bold" HorizontalOptions="Center" />
                    <Label Text="เวลาส่ง" FontSize="12" TextColor="#8E8E93" HorizontalOptions="Center" />
                </VerticalStackLayout>
                <VerticalStackLayout HorizontalOptions="Center">
                    <Label Text="{Binding Restaurant.DeliveryFee}" 
                           FontSize="16" FontAttributes="Bold" HorizontalOptions="Center" />
                    <Label Text="ค่าส่ง" FontSize="12" TextColor="#8E8E93" HorizontalOptions="Center" />
                </VerticalStackLayout>
                <VerticalStackLayout HorizontalOptions="Center">
                    <Label Text="{Binding Restaurant.MinimumOrder}"
                           FontSize="16" FontAttributes="Bold" HorizontalOptions="Center" />
                    <Label Text="สั่งขั้นต่ำ" FontSize="12" TextColor="#8E8E93" HorizontalOptions="Center" />
                </VerticalStackLayout>
            </HorizontalStackLayout>
            
            <!-- Menu Items by Category -->
            <CollectionView Grid.Row="1" ItemsSource="{Binding MenuGroups}">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <VerticalStackLayout>
                            <!-- Category Header -->
                            <Label Text="{Binding Category}"
                                   FontSize="18" FontAttributes="Bold"
                                   Margin="16,16,16,8" />
                            <!-- Items -->
                            <BindableLayout.ItemsSource>
                                <BindableLayout ItemsSource="{Binding Items}">
                                    <BindableLayout.ItemTemplate>
                                        <DataTemplate>
                                            <Grid ColumnDefinitions="*,100" Padding="16,8"
                                                  BackgroundColor="White" Margin="0,0,0,1">
                                                <VerticalStackLayout Grid.Column="0" Spacing="4">
                                                    <Label Text="{Binding Name}"
                                                           FontSize="15" FontAttributes="Bold" />
                                                    <Label Text="{Binding Description}"
                                                           FontSize="13" TextColor="#8E8E93"
                                                           MaxLines="2" LineBreakMode="TailTruncation" />
                                                    <Label Text="{Binding Price}"
                                                           FontSize="15" TextColor="#007AFF"
                                                           FontAttributes="Bold" />
                                                </VerticalStackLayout>
                                                <Frame Grid.Column="1" Padding="0"
                                                       CornerRadius="8" ClipsToBounds="True">
                                                    <Image Source="{Binding ImageUrl}"
                                                           Aspect="AspectFill" />
                                                    <!-- Add button -->
                                                    <Frame.GestureRecognizers>
                                                        <TapGestureRecognizer
                                                            Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, Path=BindingContext.AddToCartCommand}"
                                                            CommandParameter="{Binding .}" />
                                                    </Frame.GestureRecognizers>
                                                </Frame>
                                            </Grid>
                                        </DataTemplate>
                                    </BindableLayout.ItemTemplate>
                                </BindableLayout>
                            </BindableLayout.ItemsSource>
                        </VerticalStackLayout>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>
        </Grid>
        
        <!-- Cart Summary Bar -->
        <Frame Grid.Row="2" Margin="16,8,16,34" Padding="20,16"
               BackgroundColor="#007AFF" CornerRadius="16" HasShadow="True"
               IsVisible="{Binding CartHasItems}">
            <Grid ColumnDefinitions="*,Auto">
                <VerticalStackLayout>
                    <Label Text="{Binding CartItemCount, StringFormat='{0} รายการ'}"
                           TextColor="White" FontSize="13" />
                    <Label Text="{Binding CartTotal}" TextColor="White"
                           FontSize="18" FontAttributes="Bold" />
                </VerticalStackLayout>
                <Label Grid.Column="1" Text="ดูตะกร้า →"
                       TextColor="White" FontSize="16" FontAttributes="Bold"
                       VerticalOptions="Center" />
            </Grid>
            <Frame.GestureRecognizers>
                <TapGestureRecognizer Command="{Binding GoToCartCommand}" />
            </Frame.GestureRecognizers>
        </Frame>
        
    </Grid>
</ContentPage>
```

---

## Step 633: Order Tracking Page

```xml
<!-- Features/OrderTracking/OrderTrackingPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyFoodDelivery.Features.OrderTracking.OrderTrackingPage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:maps="clr-namespace:Microsoft.Maui.Controls.Maps;assembly=Microsoft.Maui.Controls.Maps"
             Title="ติดตามออร์เดอร์">
    
    <Grid RowDefinitions="260,*">
        
        <!-- Map -->
        <maps:Map x:Name="TrackingMap" Grid.Row="0"
                  MapType="Street"
                  IsShowingUser="True" />
        
        <!-- Order Details Panel -->
        <ScrollView Grid.Row="1">
            <VerticalStackLayout Padding="16">
                
                <!-- Status -->
                <Frame BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                       CornerRadius="16" Padding="16" Margin="0,0,0,12">
                    <VerticalStackLayout Spacing="12">
                        
                        <Label Text="{Binding StatusText}"
                               FontSize="20" FontAttributes="Bold"
                               HorizontalOptions="Center" />
                        
                        <Label Text="{Binding EtaText}"
                               FontSize="14" TextColor="#8E8E93"
                               HorizontalOptions="Center"
                               IsVisible="{Binding IsDelivered, Converter={StaticResource InvertBoolConverter}}" />
                        
                        <!-- Progress Steps -->
                        <Grid ColumnDefinitions="*,*,*,*,*,*" HeightRequest="60">
                            <!-- Step indicators with connecting lines -->
                            <BoxView Grid.ColumnSpan="6" HeightRequest="2"
                                     VerticalOptions="Center" Color="#E5E5EA" />
                            <BoxView Grid.ColumnSpan="6" HeightRequest="2"
                                     VerticalOptions="Center" Color="#007AFF"
                                     WidthRequest="{Binding ProgressWidth}" HorizontalOptions="Start" />
                        </Grid>
                        
                        <!-- Progress bar -->
                        <ProgressBar Progress="{Binding ProgressPercent, Converter={StaticResource PercentToProgressConverter}}"
                                     ProgressColor="#007AFF"
                                     HeightRequest="6" />
                        
                    </VerticalStackLayout>
                </Frame>
                
                <!-- Rider Info (when assigned) -->
                <Frame IsVisible="{Binding HasRider}"
                       BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                       CornerRadius="16" Padding="16" Margin="0,0,0,12">
                    <Grid ColumnDefinitions="56,*,Auto" ColumnSpacing="12">
                        <Frame Grid.Column="0" Padding="0" CornerRadius="28"
                               WidthRequest="56" HeightRequest="56">
                            <Image Source="{Binding Rider.AvatarUrl}" Aspect="AspectFill" />
                        </Frame>
                        <VerticalStackLayout Grid.Column="1" VerticalOptions="Center">
                            <Label Text="{Binding Rider.Name}" FontSize="16" FontAttributes="Bold" />
                            <Label Text="{Binding Rider.VehicleInfo}" FontSize="13" TextColor="#8E8E93" />
                        </VerticalStackLayout>
                        <HorizontalStackLayout Grid.Column="2" Spacing="8" VerticalOptions="Center">
                            <ImageButton Source="call.png" HeightRequest="44" WidthRequest="44"
                                         BackgroundColor="#F2F2F7" CornerRadius="22"
                                         Command="{Binding CallRiderCommand}" />
                            <ImageButton Source="chat.png" HeightRequest="44" WidthRequest="44"
                                         BackgroundColor="#F2F2F7" CornerRadius="22"
                                         Command="{Binding ChatRiderCommand}" />
                        </HorizontalStackLayout>
                    </Grid>
                </Frame>
                
                <!-- Order Items -->
                <Frame BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                       CornerRadius="16" Padding="16" Margin="0,0,0,12">
                    <VerticalStackLayout Spacing="12">
                        <Label Text="รายการอาหาร" FontSize="16" FontAttributes="Bold" />
                        <BindableLayout.ItemsSource>
                            <BindableLayout ItemsSource="{Binding Order.Lines}">
                                <BindableLayout.ItemTemplate>
                                    <DataTemplate>
                                        <Grid ColumnDefinitions="Auto,*,Auto">
                                            <Label Grid.Column="0" Text="{Binding Quantity, StringFormat='x{0}'}"
                                                   TextColor="#8E8E93" WidthRequest="32" />
                                            <Label Grid.Column="1" Text="{Binding ItemName}" />
                                            <Label Grid.Column="2" Text="{Binding Subtotal}" TextColor="#007AFF" />
                                        </Grid>
                                    </DataTemplate>
                                </BindableLayout.ItemTemplate>
                            </BindableLayout>
                        </BindableLayout.ItemsSource>
                        
                        <BoxView HeightRequest="1" Color="#E5E5EA" />
                        
                        <Grid ColumnDefinitions="*,Auto">
                            <Label Text="รวมทั้งสิ้น" FontAttributes="Bold" />
                            <Label Grid.Column="1" Text="{Binding Order.Total}"
                                   FontAttributes="Bold" TextColor="#007AFF" />
                        </Grid>
                    </VerticalStackLayout>
                </Frame>
                
                <!-- Rate order (after delivery) -->
                <Button Text="ให้คะแนนออร์เดอร์"
                        BackgroundColor="#007AFF" TextColor="White"
                        CornerRadius="12" HeightRequest="50"
                        Command="{Binding RateOrderCommand}"
                        IsVisible="{Binding IsDelivered}"
                        Margin="0,0,0,16" />
                
            </VerticalStackLayout>
        </ScrollView>
        
    </Grid>
</ContentPage>
```

---

## Step 634: Order Tracking Code-Behind + Map

```csharp
// ============================================
// Order Tracking with Map
// ============================================

public partial class OrderTrackingPage : ContentPage
{
    private readonly OrderTrackingViewModel _vm;
    private Pin? _restaurantPin;
    private Pin? _riderPin;
    private Pin? _destinationPin;
    
    public OrderTrackingPage(OrderTrackingViewModel vm)
    {
        InitializeComponent();
        BindingContext = _vm = vm;
    }
    
    protected override async void OnAppearing()
    {
        base.OnAppearing();
        
        // Get orderId from shell navigation
        if (Shell.Current.CurrentState.Location.ToString().Contains("id="))
        {
            // Parse ID
            var orderId = 1; // from query
            await _vm.LoadOrderAsync(orderId);
            SetupMap();
        }
        
        _vm.PropertyChanged += OnViewModelPropertyChanged;
    }
    
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _vm.PropertyChanged -= OnViewModelPropertyChanged;
    }
    
    private void SetupMap()
    {
        if (_vm.Order == null) return;
        
        // Restaurant pin
        _restaurantPin = new Pin
        {
            Label = _vm.Order.RestaurantName,
            Type = PinType.Place,
            Location = new Microsoft.Maui.Devices.Sensors.Location(
                _vm.Order.RestaurantLat, _vm.Order.RestaurantLng)
        };
        TrackingMap.Pins.Add(_restaurantPin);
        
        // Destination pin
        _destinationPin = new Pin
        {
            Label = "ที่อยู่ของคุณ",
            Type = PinType.SavedPin,
            Location = new Microsoft.Maui.Devices.Sensors.Location(
                _vm.Order.DeliveryLat, _vm.Order.DeliveryLng)
        };
        TrackingMap.Pins.Add(_destinationPin);
        
        // Center map
        TrackingMap.MoveToRegion(MapSpan.FromCenterAndRadius(
            _restaurantPin.Location,
            Distance.FromKilometers(3)));
    }
    
    private void OnViewModelPropertyChanged(object? sender, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == nameof(OrderTrackingViewModel.RiderLocation))
            UpdateRiderOnMap();
    }
    
    private void UpdateRiderOnMap()
    {
        if (_vm.RiderLocation == null) return;
        
        var loc = new Microsoft.Maui.Devices.Sensors.Location(
            _vm.RiderLocation.Lat, _vm.RiderLocation.Lng);
        
        if (_riderPin == null)
        {
            _riderPin = new Pin
            {
                Label = "ไรเดอร์",
                Type = PinType.SearchResult,
                Location = loc
            };
            TrackingMap.Pins.Add(_riderPin);
        }
        else
        {
            _riderPin.Location = loc;
        }
        
        // Follow rider
        TrackingMap.MoveToRegion(MapSpan.FromCenterAndRadius(loc, Distance.FromKilometers(1)));
    }
}
```

---

## Step 635: Rating & Review UI

```xml
<!-- Components/RatingBottomSheet.xaml -->
<ContentView xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyFoodDelivery.Shared.Controls.RatingBottomSheet">
    
    <Frame BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
           CornerRadius="24" Padding="24" HasShadow="True">
        <VerticalStackLayout Spacing="20">
            
            <Label Text="คุณพอใจกับออร์เดอร์นี้แค่ไหน?"
                   FontSize="20" FontAttributes="Bold" HorizontalOptions="Center" />
            
            <!-- Star Rating -->
            <HorizontalStackLayout HorizontalOptions="Center" Spacing="8">
                <Label x:Name="Star1" Text="⭐" FontSize="40"
                       Opacity="{Binding Rating, Converter={StaticResource StarOpacityConverter}, ConverterParameter=1}">
                    <Label.GestureRecognizers>
                        <TapGestureRecognizer Command="{Binding SetRatingCommand}" CommandParameter="1" />
                    </Label.GestureRecognizers>
                </Label>
                <Label Text="⭐" FontSize="40"
                       Opacity="{Binding Rating, Converter={StaticResource StarOpacityConverter}, ConverterParameter=2}">
                    <Label.GestureRecognizers>
                        <TapGestureRecognizer Command="{Binding SetRatingCommand}" CommandParameter="2" />
                    </Label.GestureRecognizers>
                </Label>
                <Label Text="⭐" FontSize="40"
                       Opacity="{Binding Rating, Converter={StaticResource StarOpacityConverter}, ConverterParameter=3}">
                    <Label.GestureRecognizers>
                        <TapGestureRecognizer Command="{Binding SetRatingCommand}" CommandParameter="3" />
                    </Label.GestureRecognizers>
                </Label>
                <Label Text="⭐" FontSize="40"
                       Opacity="{Binding Rating, Converter={StaticResource StarOpacityConverter}, ConverterParameter=4}">
                    <Label.GestureRecognizers>
                        <TapGestureRecognizer Command="{Binding SetRatingCommand}" CommandParameter="4" />
                    </Label.GestureRecognizers>
                </Label>
                <Label Text="⭐" FontSize="40"
                       Opacity="{Binding Rating, Converter={StaticResource StarOpacityConverter}, ConverterParameter=5}">
                    <Label.GestureRecognizers>
                        <TapGestureRecognizer Command="{Binding SetRatingCommand}" CommandParameter="5" />
                    </Label.GestureRecognizers>
                </Label>
            </HorizontalStackLayout>
            
            <Label Text="{Binding RatingLabel}" FontSize="16"
                   HorizontalOptions="Center" TextColor="#8E8E93" />
            
            <!-- Quick tags -->
            <FlexLayout Direction="Row" Wrap="Wrap" JustifyContent="Center"
                        AlignContent="Center">
                <BindableLayout.ItemsSource>
                    <Binding Path="QuickTags" />
                </BindableLayout.ItemsSource>
                <BindableLayout.ItemTemplate>
                    <DataTemplate>
                        <Frame Margin="4"
                               BackgroundColor="{Binding IsSelected, Converter={StaticResource SelectedTagColorConverter}}"
                               CornerRadius="16" Padding="12,6"
                               HasShadow="False">
                            <Label Text="{Binding Text}" FontSize="14"
                                   TextColor="{Binding IsSelected, Converter={StaticResource SelectedTagTextColorConverter}}" />
                            <Frame.GestureRecognizers>
                                <TapGestureRecognizer Command="{Binding ToggleCommand}" />
                            </Frame.GestureRecognizers>
                        </Frame>
                    </DataTemplate>
                </BindableLayout.ItemTemplate>
            </FlexLayout>
            
            <!-- Comment -->
            <Frame BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#3A3A3C}"
                   CornerRadius="12" HasShadow="False" Padding="12">
                <Editor Placeholder="บอกเราเพิ่มเติมได้เลย (ไม่บังคับ)"
                        Text="{Binding Comment}"
                        HeightRequest="80" MaxLength="200" />
            </Frame>
            
            <!-- Submit -->
            <Button Text="ส่งรีวิว"
                    BackgroundColor="#007AFF" TextColor="White"
                    CornerRadius="12" HeightRequest="50"
                    Command="{Binding SubmitRatingCommand}"
                    IsEnabled="{Binding Rating, Converter={StaticResource GreaterThanZeroConverter}}" />
            
        </VerticalStackLayout>
    </Frame>
</ContentView>
```

---

## Step 636: Animations

```csharp
// ============================================
// Page Animations
// ============================================

// Fade-in animation on page appear
public static class PageAnimations
{
    public static async Task FadeInAsync(VisualElement element)
    {
        element.Opacity = 0;
        element.TranslationY = 20;
        
        await Task.WhenAll(
            element.FadeTo(1, 300, Easing.CubicOut),
            element.TranslateTo(0, 0, 300, Easing.CubicOut));
    }
    
    public static async Task SlideUpAsync(VisualElement element, uint duration = 350)
    {
        element.TranslationY = element.Height;
        await element.TranslateTo(0, 0, duration, Easing.CubicOut);
    }
    
    public static async Task BounceAsync(VisualElement element)
    {
        await element.ScaleTo(1.15, 100);
        await element.ScaleTo(0.95, 80);
        await element.ScaleTo(1.0, 60);
    }
    
    public static async Task ShakeAsync(VisualElement element)
    {
        for (int i = 0; i < 3; i++)
        {
            await element.TranslateTo(-8, 0, 60);
            await element.TranslateTo(8, 0, 60);
        }
        await element.TranslateTo(0, 0, 60);
    }
    
    public static async Task PulseAsync(VisualElement element, Color pulseColor, Color originalColor)
    {
        if (element is Frame frame)
        {
            frame.BackgroundColor = pulseColor;
            await Task.Delay(200);
            await frame.ColorTo(originalColor, 400);
        }
    }
}

// Add-to-cart animation
public class AddToCartAnimator
{
    private readonly AbsoluteLayout _overlay;
    
    public AddToCartAnimator(AbsoluteLayout overlay) => _overlay = overlay;
    
    public async Task AnimateAsync(View sourceView, View cartIcon)
    {
        // Get positions
        var sourceBounds = sourceView.GetAbsoluteBounds();
        var cartBounds = cartIcon.GetAbsoluteBounds();
        
        // Create flying item
        var flyingItem = new Frame
        {
            WidthRequest = 40, HeightRequest = 40,
            BackgroundColor = Colors.OrangeRed,
            CornerRadius = 20
        };
        
        AbsoluteLayout.SetLayoutBounds(flyingItem, new Rect(
            sourceBounds.X, sourceBounds.Y, 40, 40));
        _overlay.Children.Add(flyingItem);
        
        // Animate to cart
        await flyingItem.TranslateTo(
            cartBounds.X - sourceBounds.X,
            cartBounds.Y - sourceBounds.Y, 500, Easing.CubicIn);
        
        await flyingItem.ScaleTo(0, 200);
        _overlay.Children.Remove(flyingItem);
        
        // Bounce cart icon
        await PageAnimations.BounceAsync(cartIcon);
    }
}
```

---

## Step 637: Profile Page

```csharp
// ============================================
// Profile ViewModel
// ============================================

public partial class ProfileViewModel : PageViewModel
{
    private readonly IUserService _users;
    private readonly IAuthService _auth;
    private readonly INavigationService _nav;
    private readonly IDialogService _dialogs;
    
    [ObservableProperty] private UserProfile? _profile;
    [ObservableProperty] private List<ProfileMenuItem> _menuItems = new();
    [ObservableProperty] private bool _isLoading;
    
    public ProfileViewModel(
        IUserService users, IAuthService auth,
        INavigationService nav, IDialogService dialogs)
    {
        _users = users;
        _auth = auth;
        _nav = nav;
        _dialogs = dialogs;
        
        BuildMenu();
    }
    
    public override async Task OnAppearingAsync()
        => await LoadProfileAsync();
    
    private async Task LoadProfileAsync()
    {
        IsLoading = true;
        Profile = await _users.GetProfileAsync();
        IsLoading = false;
    }
    
    private void BuildMenu()
    {
        MenuItems = new List<ProfileMenuItem>
        {
            new("ข้อมูลส่วนตัว", "person.png", NavigateToEditProfileCommand),
            new("ที่อยู่จัดส่ง", "location.png", NavigateToAddressesCommand),
            new("วิธีชำระเงิน", "creditcard.png", NavigateToPaymentsCommand),
            new("ประวัติคำสั่งซื้อ", "receipt.png", NavigateToOrderHistoryCommand),
            new("คูปองและส่วนลด", "gift.png", NavigateToCouponsCommand),
            new("การแจ้งเตือน", "bell.png", NavigateToNotificationsCommand),
            new("ความเป็นส่วนตัว", "lock.png", NavigateToPrivacyCommand),
            new("ติดต่อเรา", "chat.png", NavigateToSupportCommand),
        };
    }
    
    [RelayCommand]
    private Task NavigateToEditProfileAsync()
        => _nav.GoToAsync<EditProfilePage>();
    
    [RelayCommand]
    private Task NavigateToAddressesAsync()
        => _nav.GoToAsync<AddressesPage>();
    
    [RelayCommand]
    private Task NavigateToPaymentsAsync()
        => _nav.GoToAsync<PaymentMethodsPage>();
    
    [RelayCommand]
    private Task NavigateToOrderHistoryAsync()
        => _nav.GoToAsync<OrderHistoryPage>();
    
    [RelayCommand]
    private Task NavigateToCouponsAsync()
        => _nav.GoToAsync<CouponsPage>();
    
    [RelayCommand]
    private Task NavigateToNotificationsAsync()
        => _nav.GoToAsync<NotificationsPage>();
    
    [RelayCommand]
    private Task NavigateToPrivacyAsync()
        => _nav.GoToAsync<PrivacyPage>();
    
    [RelayCommand]
    private Task NavigateToSupportAsync()
        => _nav.GoToAsync<SupportPage>();
    
    [RelayCommand]
    private async Task LogoutAsync()
    {
        var confirmed = await _dialogs.ShowConfirmAsync(
            "ออกจากระบบ", "คุณต้องการออกจากระบบหรือไม่?",
            "ออก", "ยกเลิก");
        
        if (!confirmed) return;
        
        await _auth.LogoutAsync();
        WeakReferenceMessenger.Default.Send(new UserLoggedOutMessage());
        await _nav.GoBackToRootAsync();
    }
}

public record ProfileMenuItem(string Title, string Icon, IAsyncRelayCommand Command);
```

---

## Step 638: Skeleton Loading

```csharp
// ============================================
// Skeleton Loading for Content
// ============================================

// RestaurantSkeleton.xaml
public class RestaurantSkeletonView : ContentView
{
    public RestaurantSkeletonView()
    {
        Content = new VerticalStackLayout
        {
            Spacing = 12,
            Children =
            {
                // Hero image skeleton
                new SkeletonView { HeightRequest = 180, CornerRadius = 16 },
                
                new VerticalStackLayout
                {
                    Margin = new Thickness(16, 0),
                    Spacing = 8,
                    Children =
                    {
                        // Name
                        new SkeletonView { HeightRequest = 22, WidthRequest = 200 },
                        // Category
                        new SkeletonView { HeightRequest = 16, WidthRequest = 120 },
                        // Stats row
                        new HorizontalStackLayout
                        {
                            Spacing = 12,
                            Children =
                            {
                                new SkeletonView { HeightRequest = 32, WidthRequest = 80 },
                                new SkeletonView { HeightRequest = 32, WidthRequest = 80 },
                                new SkeletonView { HeightRequest = 32, WidthRequest = 80 },
                            }
                        }
                    }
                }
            }
        };
    }
}

// Home page skeleton
public class HomeSkeletonView : ContentView
{
    public HomeSkeletonView()
    {
        Content = new VerticalStackLayout
        {
            Padding = new Thickness(16),
            Spacing = 16,
            Children =
            {
                // Banner skeleton
                new SkeletonView { HeightRequest = 160, CornerRadius = 16 },
                // Category row
                new HorizontalStackLayout
                {
                    Spacing = 12,
                    Children = Enumerable.Range(0, 5).Select(_ =>
                        new SkeletonView { HeightRequest = 80, WidthRequest = 72, CornerRadius = 8 } as View).ToList()
                },
                // Section header
                new SkeletonView { HeightRequest = 24, WidthRequest = 160 },
                // Restaurant cards row
                new HorizontalStackLayout
                {
                    Spacing = 12,
                    Children = Enumerable.Range(0, 3).Select(_ =>
                        new SkeletonView { HeightRequest = 200, WidthRequest = 180, CornerRadius = 12 } as View).ToList()
                }
            }
        };
    }
}
```

---

## Step 639: Push Notifications

```csharp
// ============================================
// Push Notification Service
// ============================================

public interface IPushNotificationService
{
    Task<string?> GetTokenAsync();
    Task RegisterAsync(string token);
    void HandleNotification(NotificationPayload payload);
}

public class FirebasePushService : IPushNotificationService
{
    private readonly INavigationService _nav;
    private readonly IOrderRepository _orders;
    
    public FirebasePushService(INavigationService nav, IOrderRepository orders)
    {
        _nav = nav;
        _orders = orders;
    }
    
    public async Task<string?> GetTokenAsync()
    {
#if ANDROID
        return await Firebase.Messaging.FirebaseMessaging.Instance.GetTokenAsync();
#elif IOS
        return await GetApnsTokenAsync();
#else
        return null;
#endif
    }
    
    public async Task RegisterAsync(string token)
    {
        // Send token to server
        using var http = new HttpClient();
        await http.PostAsJsonAsync(
            $"{AppConfig.ApiBaseUrl}/notifications/register",
            new { Token = token, Platform = DeviceInfo.Platform.ToString() });
    }
    
    public void HandleNotification(NotificationPayload payload)
    {
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            switch (payload.Type)
            {
                case "order_status_changed":
                    var orderId = int.Parse(payload.Data["orderId"]);
                    await _nav.GoToAsync<OrderTrackingPage>(orderId);
                    break;
                
                case "promotion":
                    await Shell.Current.GoToAsync("//Home");
                    break;
                
                case "chat_message":
                    // Show in-app notification
                    WeakReferenceMessenger.Default.Send(
                        new IncomingChatMessage(payload.Data["message"]));
                    break;
            }
        });
    }
    
    private Task<string?> GetApnsTokenAsync() => Task.FromResult<string?>(null);
}

public record NotificationPayload(string Type, Dictionary<string, string> Data, string? Title, string? Body);
public record IncomingChatMessage(string Message);
```

---

## Step 640: App Startup Flow

```csharp
// ============================================
// App Startup & Onboarding
// ============================================

public partial class App : Application
{
    private readonly IAuthTokenService _auth;
    private readonly IPushNotificationService _push;
    private readonly IFeatureFlagService _flags;
    
    public App(
        IAuthTokenService auth,
        IPushNotificationService push,
        IFeatureFlagService flags)
    {
        _auth = auth;
        _push = push;
        _flags = flags;
        
        InitializeComponent();
        SetupMessaging();
    }
    
    protected override async void OnStart()
    {
        base.OnStart();
        
        // Show splash
        MainPage = new SplashPage();
        
        await Task.WhenAll(
            InitializeAuthAsync(),
            InitializePushAsync(),
            InitializeFeatureFlagsAsync());
        
        // Navigate to appropriate start page
        MainPage = await DetermineStartPage();
    }
    
    private async Task InitializeAuthAsync()
    {
        try
        {
            if (await _auth.IsAuthenticatedAsync())
                await _auth.RefreshIfNeededAsync();
        }
        catch { /* Ignore - will redirect to login */ }
    }
    
    private async Task InitializePushAsync()
    {
        var token = await _push.GetTokenAsync();
        if (token != null) await _push.RegisterAsync(token);
    }
    
    private async Task InitializeFeatureFlagsAsync()
    {
        // Pre-fetch feature flags
        await Task.WhenAll(
            Enum.GetValues<FeatureFlag>()
                .Select(f => _flags.IsEnabledAsync(f)));
    }
    
    private async Task<Page> DetermineStartPage()
    {
        var isAuthenticated = await _auth.IsAuthenticatedAsync();
        
        if (!isAuthenticated)
        {
            // First time: show onboarding
            if (!Preferences.Default.ContainsKey("onboarding_done"))
                return new OnboardingPage();
            
            return new LoginPage();
        }
        
        return new AppShell();
    }
    
    private void SetupMessaging()
    {
        WeakReferenceMessenger.Default.Register<UserLoggedOutMessage>(this, (_, _) =>
        {
            MainThread.BeginInvokeOnMainThread(async () =>
            {
                await Task.Delay(500);
                MainPage = new LoginPage();
            });
        });
    }
}
```

---

## สรุป Part 64

ใน Part 64 เราได้เรียนรู้:

1. **Home Page XAML** - Header, carousel, categories, restaurant lists
2. **Restaurant Page** - Hero image, menu items, cart bar
3. **Order Tracking XAML** - Map, status, rider info
4. **Map Integration** - Pin management, follow rider
5. **Rating & Review** - Star rating, quick tags, comment
6. **Page Animations** - Fade, slide, bounce, shake
7. **Skeleton Loading** - Restaurant & home skeletons
8. **Push Notifications** - Firebase token, handle tap
9. **Profile Page** - Menu items, logout
10. **App Startup Flow** - Auth check, feature flags, routing

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 64 | Steps 631-640*
