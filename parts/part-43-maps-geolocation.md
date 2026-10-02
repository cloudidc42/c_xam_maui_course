# Part 43: Maps & Geolocation Advanced
## Steps 421-430: Google Maps, Location Tracking, Geofencing

---

## Step 421: Maps Integration

```csharp
// ============================================
// .NET MAUI Maps
// ============================================

// NuGet: Microsoft.Maui.Controls.Maps

// MauiProgram.cs
builder.UseMauiMaps();

// XAML
/*
<ContentPage xmlns:maps="clr-namespace:Microsoft.Maui.Controls.Maps;assembly=Microsoft.Maui.Controls.Maps">
    <maps:Map x:Name="MainMap"
              MapType="Street"
              IsShowingUser="True"
              IsScrollEnabled="True"
              IsZoomEnabled="True"
              IsTrafficEnabled="True" />
</ContentPage>
*/

// Code-behind
public partial class MapPage : ContentPage
{
    public MapPage()
    {
        InitializeComponent();
        SetupMap();
    }
    
    private void SetupMap()
    {
        // Center on Bangkok
        var bangkok = new Location(13.7563, 100.5018);
        MainMap.MoveToRegion(MapSpan.FromCenterAndRadius(
            bangkok, Distance.FromKilometers(5)));
        
        // Add pins
        AddStorePin(bangkok, "สาขาหลัก", "เปิด 9:00-22:00");
        AddStorePin(new Location(13.7660, 100.5380), "สาขาทองหล่อ", "เปิด 10:00-21:00");
    }
    
    private void AddStorePin(Location location, string name, string address)
    {
        var pin = new Pin
        {
            Location = location,
            Label = name,
            Address = address,
            Type = PinType.Place
        };
        
        pin.MarkerClicked += async (s, e) =>
        {
            e.HideInfoWindow = false;
            await DisplayAlert(name, address, "ตกลง");
        };
        
        MainMap.Pins.Add(pin);
    }
}
```

---

## Step 422: Custom Map Pins

```csharp
// ============================================
// Custom Map Pins & Overlays
// ============================================

// Custom pin with data
public class StorePin : Pin
{
    public int StoreId { get; set; }
    public bool IsOpen { get; set; }
    public int Distance { get; set; }
    public string? PhoneNumber { get; set; }
}

// Map with clustering and custom icons
public class StoreMapViewModel : ObservableObject
{
    private readonly IStoreService _stores;
    private readonly IGeolocationService _geo;
    
    public ObservableCollection<StorePin> StorePins { get; } = new();
    
    public StoreMapViewModel(IStoreService stores, IGeolocationService geo)
    {
        _stores = stores;
        _geo = geo;
    }
    
    public async Task LoadStoresAsync(Location? centerLocation = null)
    {
        var location = centerLocation ?? await _geo.GetCurrentLocationAsync();
        if (location == null) return;
        
        var stores = await _stores.GetNearbyAsync(
            location.Latitude, location.Longitude, radiusKm: 10);
        
        StorePins.Clear();
        foreach (var store in stores)
        {
            StorePins.Add(new StorePin
            {
                StoreId = store.Id,
                Label = store.Name,
                Address = store.Address,
                Location = new Location(store.Latitude, store.Longitude),
                IsOpen = store.IsCurrentlyOpen,
                Distance = (int)store.DistanceKm
            });
        }
    }
    
    // Draw service area circle
    public MapCircle CreateServiceAreaCircle(Location center, double radiusKm)
    {
        return new MapCircle(center)
        {
            Radius = Distance.FromKilometers(radiusKm),
            FillColor = Color.FromArgb("#330000FF"),
            StrokeColor = Colors.Blue,
            StrokeWidth = 2
        };
    }
    
    // Draw delivery route polyline
    public MapPolyline CreateRoutePolyline(List<Location> waypoints)
    {
        var polyline = new MapPolyline
        {
            StrokeColor = Colors.Blue,
            StrokeWidth = 4
        };
        
        foreach (var point in waypoints)
            polyline.Geopath.Add(point);
        
        return polyline;
    }
}
```

---

## Step 423: Location Services

```csharp
// ============================================
// Location Service
// ============================================

public interface IGeolocationService
{
    Task<Location?> GetCurrentLocationAsync(GeolocationAccuracy accuracy = GeolocationAccuracy.Medium);
    Task<Location?> GetLastKnownLocationAsync();
    IObservable<Location> GetLocationStream(TimeSpan interval, double distanceMeters = 50);
    Task<bool> RequestPermissionAsync();
    Task<IEnumerable<Placemark>> GetPlacemarksAsync(Location location);
    Task<Location?> GeocodeAsync(string address);
}

public class GeolocationService : IGeolocationService
{
    public async Task<Location?> GetCurrentLocationAsync(
        GeolocationAccuracy accuracy = GeolocationAccuracy.Medium)
    {
        try
        {
            var request = new GeolocationRequest(accuracy, TimeSpan.FromSeconds(10));
            return await Geolocation.Default.GetLocationAsync(request);
        }
        catch (FeatureNotSupportedException)
        {
            return null; // GPS not available
        }
        catch (PermissionException)
        {
            return null; // Permission denied
        }
        catch (Exception)
        {
            return await GetLastKnownLocationAsync();
        }
    }
    
    public async Task<Location?> GetLastKnownLocationAsync()
    {
        try { return await Geolocation.Default.GetLastKnownLocationAsync(); }
        catch { return null; }
    }
    
    public IObservable<Location> GetLocationStream(
        TimeSpan interval, double distanceMeters = 50)
    {
        return System.Reactive.Linq.Observable.Create<Location>(async (observer, ct) =>
        {
            while (!ct.IsCancellationRequested)
            {
                var location = await GetCurrentLocationAsync(GeolocationAccuracy.High);
                if (location != null)
                    observer.OnNext(location);
                
                await Task.Delay(interval, ct);
            }
        });
    }
    
    public async Task<bool> RequestPermissionAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();
        
        if (status == PermissionStatus.Granted) return true;
        
        status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
        return status == PermissionStatus.Granted;
    }
    
    public async Task<IEnumerable<Placemark>> GetPlacemarksAsync(Location location)
    {
        return await Geocoding.Default.GetPlacemarksAsync(
            location.Latitude, location.Longitude);
    }
    
    public async Task<Location?> GeocodeAsync(string address)
    {
        var locations = await Geocoding.Default.GetLocationsAsync(address);
        return locations?.FirstOrDefault();
    }
    
    // Calculate distance
    public static double CalculateDistance(Location from, Location to)
    {
        return Location.CalculateDistance(from, to, DistanceUnits.Kilometers);
    }
    
    // Format distance Thai style
    public static string FormatDistance(double km)
    {
        if (km < 1) return $"{(int)(km * 1000)} เมตร";
        return km < 10 ? $"{km:F1} กม." : $"{(int)km} กม.";
    }
}
```

---

## Step 424: Geofencing

```csharp
// ============================================
// Geofencing
// ============================================

public class GeofenceRegion
{
    public string Id { get; set; } = string.Empty;
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public double RadiusMeters { get; set; }
    public string? Label { get; set; }
    public bool NotifyOnEntry { get; set; } = true;
    public bool NotifyOnExit { get; set; } = true;
    public bool NotifyOnDwell { get; set; }
    public TimeSpan DwellTime { get; set; } = TimeSpan.FromMinutes(5);
}

public class GeofenceService
{
    private readonly Dictionary<string, GeofenceRegion> _regions = new();
    private readonly IGeolocationService _geo;
    private readonly LocalNotificationService _notifications;
    private Location? _lastLocation;
    private readonly Dictionary<string, DateTime> _entryTimes = new();
    
    public event EventHandler<GeofenceEvent>? GeofenceTriggered;
    
    public GeofenceService(IGeolocationService geo, LocalNotificationService notifications)
    {
        _geo = geo;
        _notifications = notifications;
    }
    
    public void AddRegion(GeofenceRegion region)
        => _regions[region.Id] = region;
    
    public void RemoveRegion(string regionId)
        => _regions.Remove(regionId);
    
    // Called periodically or on location update
    public async Task CheckGeofencesAsync(Location location)
    {
        foreach (var region in _regions.Values)
        {
            var distance = Location.CalculateDistance(
                new Location(region.Latitude, region.Longitude),
                location,
                DistanceUnits.Kilometers) * 1000; // to meters
            
            bool isInside = distance <= region.RadiusMeters;
            bool wasInside = _lastLocation != null &&
                Location.CalculateDistance(
                    new Location(region.Latitude, region.Longitude),
                    _lastLocation,
                    DistanceUnits.Kilometers) * 1000 <= region.RadiusMeters;
            
            // Entry
            if (isInside && !wasInside && region.NotifyOnEntry)
            {
                _entryTimes[region.Id] = DateTime.UtcNow;
                await TriggerEventAsync(region, GeofenceEventType.Entry);
            }
            
            // Exit
            if (!isInside && wasInside && region.NotifyOnExit)
            {
                _entryTimes.Remove(region.Id);
                await TriggerEventAsync(region, GeofenceEventType.Exit);
            }
            
            // Dwell
            if (isInside && region.NotifyOnDwell &&
                _entryTimes.TryGetValue(region.Id, out var entryTime) &&
                DateTime.UtcNow - entryTime >= region.DwellTime)
            {
                _entryTimes[region.Id] = DateTime.UtcNow; // reset dwell
                await TriggerEventAsync(region, GeofenceEventType.Dwell);
            }
        }
        
        _lastLocation = location;
    }
    
    private async Task TriggerEventAsync(GeofenceRegion region, GeofenceEventType eventType)
    {
        var geofenceEvent = new GeofenceEvent(region, eventType, DateTime.UtcNow);
        GeofenceTriggered?.Invoke(this, geofenceEvent);
        
        // Send local notification
        var message = eventType switch
        {
            GeofenceEventType.Entry => $"คุณเข้ามาใกล้ {region.Label}",
            GeofenceEventType.Exit => $"คุณออกจากพื้นที่ {region.Label}",
            GeofenceEventType.Dwell => $"คุณอยู่ใน {region.Label} นานกว่า {region.DwellTime.TotalMinutes} นาที",
            _ => ""
        };
        
        if (!string.IsNullOrEmpty(message))
        {
            await _notifications.ScheduleAsync(new LocalNotification
            {
                Id = region.Id.GetHashCode(),
                Title = "แจ้งเตือนพื้นที่",
                Body = message,
                ScheduledFor = DateTime.Now
            });
        }
    }
}

public enum GeofenceEventType { Entry, Exit, Dwell }
public record GeofenceEvent(GeofenceRegion Region, GeofenceEventType Type, DateTime Timestamp);
```

---

## Step 425: Delivery Tracking

```csharp
// ============================================
// Real-time Delivery Tracking
// ============================================

public class DeliveryTrackingViewModel : ObservableObject, IDisposable
{
    private readonly IGeolocationService _geo;
    private readonly IDeliveryApi _deliveryApi;
    private readonly System.Reactive.Disposables.CompositeDisposable _disposables = new();
    
    public Pin? DeliveryPin { get; private set; }
    public MapSpan? CurrentRegion { get; private set; }
    public string StatusText { get; private set; } = string.Empty;
    public string EstimatedArrival { get; private set; } = string.Empty;
    public List<Location> RouteWaypoints { get; private set; } = new();
    
    public DeliveryTrackingViewModel(
        IGeolocationService geo, IDeliveryApi deliveryApi)
    {
        _geo = geo;
        _deliveryApi = deliveryApi;
    }
    
    public async Task StartTrackingAsync(string orderId)
    {
        // Fetch initial delivery info
        var delivery = await _deliveryApi.GetDeliveryStatusAsync(orderId);
        UpdateDeliveryInfo(delivery);
        
        // Poll for updates every 30 seconds
        System.Reactive.Linq.Observable.Interval(TimeSpan.FromSeconds(30))
            .SelectMany(_ => System.Reactive.Linq.Observable.FromAsync(
                () => _deliveryApi.GetDeliveryStatusAsync(orderId)))
            .ObserveOn(System.Reactive.Concurrency.TaskPoolScheduler.Default)
            .Subscribe(UpdateDeliveryInfo)
            .DisposeWith(_disposables);
    }
    
    private void UpdateDeliveryInfo(DeliveryStatus delivery)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            StatusText = delivery.Status;
            EstimatedArrival = $"ถึงภายใน {delivery.EstimatedMinutes} นาที";
            
            DeliveryPin = new Pin
            {
                Location = new Location(delivery.CourierLatitude, delivery.CourierLongitude),
                Label = "พนักงานส่งของ",
                Type = PinType.SearchResult
            };
            
            RouteWaypoints = delivery.RoutePoints
                .Select(p => new Location(p.Lat, p.Lng))
                .ToList();
            
            // Center map
            CurrentRegion = MapSpan.FromCenterAndRadius(
                DeliveryPin.Location,
                Distance.FromKilometers(2));
            
            OnPropertyChanged(nameof(DeliveryPin));
            OnPropertyChanged(nameof(StatusText));
            OnPropertyChanged(nameof(EstimatedArrival));
            OnPropertyChanged(nameof(RouteWaypoints));
            OnPropertyChanged(nameof(CurrentRegion));
        });
    }
    
    public void Dispose() => _disposables.Dispose();
}
```

---

## Step 426: Store Locator

```csharp
// ============================================
// Store Locator with Search
// ============================================

public partial class StoreLocatorViewModel : ObservableObject
{
    private readonly IGeolocationService _geo;
    private readonly IStoreService _stores;
    
    [ObservableProperty] private ObservableCollection<StoreItemViewModel> _nearbyStores = new();
    [ObservableProperty] private ObservableCollection<Pin> _mapPins = new();
    [ObservableProperty] private string _searchQuery = string.Empty;
    [ObservableProperty] private bool _isLoading;
    [ObservableProperty] private Location? _userLocation;
    [ObservableProperty] private MapSpan? _mapRegion;
    
    public StoreLocatorViewModel(IGeolocationService geo, IStoreService stores)
    {
        _geo = geo;
        _stores = stores;
    }
    
    [RelayCommand]
    private async Task LoadNearbyAsync()
    {
        IsLoading = true;
        
        UserLocation = await _geo.GetCurrentLocationAsync(GeolocationAccuracy.Medium);
        
        if (UserLocation == null)
        {
            IsLoading = false;
            return;
        }
        
        MapRegion = MapSpan.FromCenterAndRadius(UserLocation, Distance.FromKilometers(5));
        
        var stores = await _stores.GetNearbyAsync(
            UserLocation.Latitude, UserLocation.Longitude, 10);
        
        NearbyStores.Clear();
        MapPins.Clear();
        
        foreach (var store in stores.OrderBy(s => s.DistanceKm))
        {
            NearbyStores.Add(new StoreItemViewModel(store, UserLocation));
            MapPins.Add(new Pin
            {
                Label = store.Name,
                Address = $"{GeolocationService.FormatDistance(store.DistanceKm)} | {(store.IsOpen ? "เปิด" : "ปิด")}",
                Location = new Location(store.Latitude, store.Longitude)
            });
        }
        
        IsLoading = false;
    }
    
    [RelayCommand]
    private async Task NavigateToStoreAsync(StoreItemViewModel store)
    {
        // Open navigation app
        await Map.Default.OpenAsync(new Location(store.Latitude, store.Longitude),
            new MapLaunchOptions
            {
                Name = store.Name,
                NavigationMode = NavigationMode.Driving
            });
    }
    
    [RelayCommand]
    private async Task SearchAsync()
    {
        if (string.IsNullOrWhiteSpace(SearchQuery)) return;
        
        // Geocode search query
        var location = await _geo.GeocodeAsync(SearchQuery);
        if (location == null) return;
        
        UserLocation = location;
        MapRegion = MapSpan.FromCenterAndRadius(location, Distance.FromKilometers(5));
        await LoadNearbyCommand.ExecuteAsync(null);
    }
}

public class StoreItemViewModel
{
    public int Id { get; }
    public string Name { get; }
    public string Address { get; }
    public string DistanceText { get; }
    public double Latitude { get; }
    public double Longitude { get; }
    public bool IsOpen { get; }
    public string Hours { get; }
    
    public StoreItemViewModel(Store store, Location userLocation)
    {
        Id = store.Id;
        Name = store.Name;
        Address = store.Address;
        Latitude = store.Latitude;
        Longitude = store.Longitude;
        IsOpen = store.IsCurrentlyOpen;
        Hours = store.OpenHours;
        DistanceText = GeolocationService.FormatDistance(store.DistanceKm);
    }
}
```

---

## Step 427: Background Location (iOS)

```csharp
// ============================================
// Background Location Tracking (iOS)
// ============================================

// Info.plist additions:
/*
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งเพื่อติดตามการส่งของ</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งเพื่อค้นหาร้านค้าใกล้เคียง</string>
<key>UIBackgroundModes</key>
<array>
    <string>location</string>
</array>
*/

// Platforms/iOS/Services/BackgroundLocationService.cs
public class BackgroundLocationService : NSObject, ICLLocationManagerDelegate
{
    private readonly CLLocationManager _locationManager;
    private Action<Location>? _onLocationUpdated;
    
    public BackgroundLocationService()
    {
        _locationManager = new CLLocationManager
        {
            Delegate = this,
            DesiredAccuracy = CLLocation.AccuracyBest,
            DistanceFilter = 50, // meters
            AllowsBackgroundLocationUpdates = true,
            PausesLocationUpdatesAutomatically = false
        };
    }
    
    public void StartTracking(Action<Location> onUpdate)
    {
        _onLocationUpdated = onUpdate;
        
        if (CLLocationManager.Status == CLAuthorizationStatus.AuthorizedAlways)
        {
            _locationManager.StartUpdatingLocation();
        }
        else
        {
            _locationManager.RequestAlwaysAuthorization();
        }
    }
    
    public void StopTracking()
    {
        _locationManager.StopUpdatingLocation();
        _onLocationUpdated = null;
    }
    
    [Export("locationManager:didUpdateLocations:")]
    public void LocationsUpdated(CLLocationManager manager, CLLocation[] locations)
    {
        var latest = locations.LastOrDefault();
        if (latest == null) return;
        
        var location = new Location(latest.Coordinate.Latitude, latest.Coordinate.Longitude)
        {
            Accuracy = latest.HorizontalAccuracy,
            Speed = latest.Speed,
            Course = latest.Course
        };
        
        MainThread.InvokeOnMainThreadAsync(() => _onLocationUpdated?.Invoke(location));
    }
    
    [Export("locationManager:didFailWithError:")]
    public void Failed(CLLocationManager manager, Foundation.NSError error)
    {
        Console.WriteLine($"Location error: {error.LocalizedDescription}");
    }
}
```

---

## Step 428: Distance Matrix

```csharp
// ============================================
// Distance & Routing
// ============================================

public class RoutingService
{
    private readonly HttpClient _http;
    private const string GoogleDirectionsUrl = 
        "https://maps.googleapis.com/maps/api/directions/json";
    
    public RoutingService(HttpClient http) => _http = http;
    
    public async Task<RouteInfo?> GetRouteAsync(
        Location origin, Location destination, 
        TravelMode mode = TravelMode.Driving)
    {
        var modeStr = mode switch
        {
            TravelMode.Walking => "walking",
            TravelMode.Bicycling => "bicycling",
            TravelMode.Transit => "transit",
            _ => "driving"
        };
        
        var url = $"{GoogleDirectionsUrl}?" +
            $"origin={origin.Latitude},{origin.Longitude}&" +
            $"destination={destination.Latitude},{destination.Longitude}&" +
            $"mode={modeStr}&language=th&" +
            $"key={AppConfig.GoogleMapsKey}";
        
        var response = await _http.GetFromJsonAsync<GoogleDirectionsResponse>(url);
        
        if (response?.Routes?.Count > 0)
        {
            var route = response.Routes[0];
            var leg = route.Legs[0];
            
            return new RouteInfo(
                Distance: leg.Distance.Value,
                DistanceText: leg.Distance.Text,
                Duration: TimeSpan.FromSeconds(leg.Duration.Value),
                DurationText: leg.Duration.Text,
                Steps: leg.Steps.Select(s => new RouteStep(
                    s.HtmlInstructions, s.Distance.Text, s.Duration.Text)).ToList(),
                Polyline: DecodePolyline(route.OverviewPolyline.Points));
        }
        
        return null;
    }
    
    private static List<Location> DecodePolyline(string encoded)
    {
        var points = new List<Location>();
        int index = 0, lat = 0, lng = 0;
        
        while (index < encoded.Length)
        {
            int b, shift = 0, result = 0;
            
            do
            {
                b = encoded[index++] - 63;
                result |= (b & 0x1f) << shift;
                shift += 5;
            } while (b >= 32);
            
            lat += (result & 1) != 0 ? ~(result >> 1) : result >> 1;
            
            shift = 0; result = 0;
            do
            {
                b = encoded[index++] - 63;
                result |= (b & 0x1f) << shift;
                shift += 5;
            } while (b >= 32);
            
            lng += (result & 1) != 0 ? ~(result >> 1) : result >> 1;
            
            points.Add(new Location(lat / 1e5, lng / 1e5));
        }
        
        return points;
    }
}

public enum TravelMode { Driving, Walking, Bicycling, Transit }
public record RouteInfo(
    int Distance, string DistanceText,
    TimeSpan Duration, string DurationText,
    List<RouteStep> Steps, List<Location> Polyline);
public record RouteStep(string Instruction, string Distance, string Duration);
```

---

## Step 429: Map Clustering

```csharp
// ============================================
// Pin Clustering Algorithm
// ============================================

public class MarkerClusterer
{
    private readonly double _clusterRadiusKm;
    
    public MarkerClusterer(double clusterRadiusKm = 0.5)
        => _clusterRadiusKm = clusterRadiusKm;
    
    public List<MapCluster> Cluster(IEnumerable<Pin> pins, double zoomLevel)
    {
        // Adjust cluster radius based on zoom
        var radius = _clusterRadiusKm * Math.Pow(2, 15 - zoomLevel);
        
        var pinList = pins.ToList();
        var clusters = new List<MapCluster>();
        var processed = new HashSet<int>();
        
        for (int i = 0; i < pinList.Count; i++)
        {
            if (processed.Contains(i)) continue;
            
            var cluster = new MapCluster();
            cluster.AddPin(pinList[i]);
            processed.Add(i);
            
            for (int j = i + 1; j < pinList.Count; j++)
            {
                if (processed.Contains(j)) continue;
                
                var distance = Location.CalculateDistance(
                    pinList[i].Location, pinList[j].Location,
                    DistanceUnits.Kilometers);
                
                if (distance <= radius)
                {
                    cluster.AddPin(pinList[j]);
                    processed.Add(j);
                }
            }
            
            clusters.Add(cluster);
        }
        
        return clusters;
    }
}

public class MapCluster
{
    private readonly List<Pin> _pins = new();
    
    public int Count => _pins.Count;
    
    public Location Center
    {
        get
        {
            var lat = _pins.Average(p => p.Location.Latitude);
            var lng = _pins.Average(p => p.Location.Longitude);
            return new Location(lat, lng);
        }
    }
    
    public Pin ClusterPin => Count == 1
        ? _pins[0]
        : new Pin
        {
            Location = Center,
            Label = $"{Count} ร้านค้า",
            Type = PinType.Place
        };
    
    public void AddPin(Pin pin) => _pins.Add(pin);
    public IReadOnlyList<Pin> Pins => _pins.AsReadOnly();
}
```

---

## Step 430: Address Autocomplete

```csharp
// ============================================
// Address Autocomplete (Places API)
// ============================================

public class PlacesAutocompleteService
{
    private readonly HttpClient _http;
    private const string AutocompleteUrl = 
        "https://maps.googleapis.com/maps/api/place/autocomplete/json";
    private const string DetailsUrl =
        "https://maps.googleapis.com/maps/api/place/details/json";
    
    public PlacesAutocompleteService(HttpClient http) => _http = http;
    
    public async Task<List<PlacePrediction>> SearchAsync(
        string input, string? sessionToken = null, CancellationToken ct = default)
    {
        if (input.Length < 3) return [];
        
        var url = $"{AutocompleteUrl}?" +
            $"input={Uri.EscapeDataString(input)}&" +
            $"language=th&components=country:th&" +
            $"key={AppConfig.GoogleMapsKey}";
        
        if (sessionToken != null)
            url += $"&sessiontoken={sessionToken}";
        
        var response = await _http.GetFromJsonAsync<PlacesAutocompleteResponse>(url, ct);
        return response?.Predictions ?? [];
    }
    
    public async Task<PlaceDetails?> GetDetailsAsync(
        string placeId, CancellationToken ct = default)
    {
        var url = $"{DetailsUrl}?" +
            $"place_id={placeId}&" +
            $"fields=name,formatted_address,geometry&" +
            $"language=th&key={AppConfig.GoogleMapsKey}";
        
        var response = await _http.GetFromJsonAsync<PlaceDetailsResponse>(url, ct);
        return response?.Result;
    }
}

// ViewModel with debounced search
public partial class AddressSearchViewModel : ObservableObject
{
    private readonly PlacesAutocompleteService _places;
    private readonly string _sessionToken = Guid.NewGuid().ToString();
    private CancellationTokenSource? _cts;
    
    [ObservableProperty] private string _searchText = string.Empty;
    [ObservableProperty] private ObservableCollection<PlacePrediction> _predictions = new();
    [ObservableProperty] private bool _isSearching;
    
    public AddressSearchViewModel(PlacesAutocompleteService places) => _places = places;
    
    partial void OnSearchTextChanged(string value)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        _ = SearchDebounced(value, _cts.Token);
    }
    
    private async Task SearchDebounced(string query, CancellationToken ct)
    {
        try
        {
            await Task.Delay(300, ct);
            
            IsSearching = true;
            var results = await _places.SearchAsync(query, _sessionToken, ct);
            
            Predictions.Clear();
            foreach (var p in results) Predictions.Add(p);
        }
        catch (OperationCanceledException) { }
        finally { IsSearching = false; }
    }
    
    [RelayCommand]
    private async Task SelectPredictionAsync(PlacePrediction prediction)
    {
        var details = await _places.GetDetailsAsync(prediction.PlaceId);
        if (details == null) return;
        
        // Clear and set selected address
        Predictions.Clear();
        SearchText = details.FormattedAddress ?? prediction.Description;
    }
}
```

---

## สรุป Part 43

ใน Part 43 เราได้เรียนรู้:

1. **Maps Integration** - .NET MAUI Maps, pins, tap events
2. **Custom Pins** - Store pins with data
3. **Location Services** - Current, last known, streaming
4. **Geofencing** - Entry/Exit/Dwell detection
5. **Delivery Tracking** - Real-time courier location
6. **Store Locator** - Nearby search, navigate
7. **Background Location** - iOS background tracking
8. **Routing** - Google Directions API, polyline decode
9. **Clustering** - Group nearby pins by zoom
10. **Address Autocomplete** - Google Places API

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 43 | Steps 421-430*
