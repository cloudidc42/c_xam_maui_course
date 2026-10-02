# Part 22: Platform Features ใน .NET MAUI
## Steps 211-220: Camera, GPS, Notifications และ Sensors

---

## Step 211: Camera และ MediaPicker

```csharp
// ============================================
// Camera และ Gallery
// ============================================

public partial class CameraViewModel : ObservableObject
{
    [ObservableProperty]
    private ImageSource? _photo;
    
    [ObservableProperty]
    private bool _hasPhoto;
    
    [ObservableProperty]
    private bool _isLoading;
    
    // Take photo with camera
    [RelayCommand]
    private async Task TakePhotoAsync()
    {
        if (!MediaPicker.Default.IsCaptureSupported)
        {
            await Shell.Current.DisplayAlert("แจ้งเตือน", "กล้องไม่รองรับ", "ตกลง");
            return;
        }
        
        try
        {
            var photo = await MediaPicker.Default.CapturePhotoAsync(new MediaPickerOptions
            {
                Title = "ถ่ายรูป"
            });
            
            if (photo != null)
            {
                await ProcessPhotoAsync(photo);
            }
        }
        catch (PermissionException)
        {
            await Shell.Current.DisplayAlert("ขอสิทธิ์", "กรุณาอนุญาตให้ใช้กล้อง", "ตกลง");
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("ผิดพลาด", ex.Message, "ตกลง");
        }
    }
    
    // Pick from gallery
    [RelayCommand]
    private async Task PickPhotoAsync()
    {
        try
        {
            var result = await MediaPicker.Default.PickPhotoAsync(new MediaPickerOptions
            {
                Title = "เลือกรูปภาพ"
            });
            
            if (result != null)
                await ProcessPhotoAsync(result);
        }
        catch (PermissionException)
        {
            await Shell.Current.DisplayAlert("ขอสิทธิ์", "กรุณาอนุญาตให้เข้าถึง Gallery", "ตกลง");
        }
    }
    
    // Pick multiple photos
    [RelayCommand]
    private async Task PickMultiplePhotosAsync()
    {
        var results = await MediaPicker.Default.PickMultipleAsync(new MediaPickerOptions
        {
            Title = "เลือกรูปภาพ (สูงสุด 5 รูป)"
        });
        
        foreach (var result in results.Take(5))
        {
            await ProcessPhotoAsync(result);
        }
    }
    
    // Record video
    [RelayCommand]
    private async Task RecordVideoAsync()
    {
        var video = await MediaPicker.Default.CaptureVideoAsync();
        if (video != null)
        {
            Console.WriteLine($"Video: {video.FileName}, {video.ContentType}");
        }
    }
    
    private async Task ProcessPhotoAsync(FileResult photo)
    {
        IsLoading = true;
        
        // Copy to app data dir (source file may be temp)
        var localPath = Path.Combine(FileSystem.AppDataDirectory, photo.FileName);
        
        using var sourceStream = await photo.OpenReadAsync();
        using var destStream = File.OpenWrite(localPath);
        await sourceStream.CopyToAsync(destStream);
        
        Photo = ImageSource.FromFile(localPath);
        HasPhoto = true;
        
        IsLoading = false;
    }
    
    [RelayCommand]
    private void RemovePhoto()
    {
        Photo = null;
        HasPhoto = false;
    }
}
```

---

## Step 212: Geolocation (GPS)

```csharp
// ============================================
// Geolocation
// ============================================

public partial class LocationViewModel : ObservableObject
{
    [ObservableProperty]
    private double _latitude;
    
    [ObservableProperty]
    private double _longitude;
    
    [ObservableProperty]
    private double _altitude;
    
    [ObservableProperty]
    private double _accuracy;
    
    [ObservableProperty]
    private string _address = string.Empty;
    
    [ObservableProperty]
    private bool _isLocating;
    
    [ObservableProperty]
    private string _statusMessage = string.Empty;
    
    [RelayCommand]
    private async Task GetLocationAsync()
    {
        IsLocating = true;
        StatusMessage = "กำลังหาตำแหน่ง...";
        
        try
        {
            var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();
            
            if (status != PermissionStatus.Granted)
            {
                status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
                
                if (status != PermissionStatus.Granted)
                {
                    StatusMessage = "ไม่ได้รับอนุญาตให้เข้าถึงตำแหน่ง";
                    return;
                }
            }
            
            var request = new GeolocationRequest(GeolocationAccuracy.Medium)
            {
                Timeout = TimeSpan.FromSeconds(10)
            };
            
            var location = await Geolocation.Default.GetLocationAsync(request);
            
            if (location != null)
            {
                Latitude = location.Latitude;
                Longitude = location.Longitude;
                Altitude = location.Altitude ?? 0;
                Accuracy = location.Accuracy ?? 0;
                
                StatusMessage = $"ตำแหน่ง: {Latitude:F6}, {Longitude:F6}";
                
                // Reverse geocoding
                await GetAddressAsync(location.Latitude, location.Longitude);
            }
            else
            {
                StatusMessage = "ไม่พบตำแหน่ง";
            }
        }
        catch (FeatureNotSupportedException)
        {
            StatusMessage = "GPS ไม่รองรับในอุปกรณ์นี้";
        }
        catch (FeatureNotEnabledException)
        {
            StatusMessage = "กรุณาเปิด GPS";
        }
        catch (PermissionException)
        {
            StatusMessage = "ไม่มีสิทธิ์เข้าถึงตำแหน่ง";
        }
        finally
        {
            IsLocating = false;
        }
    }
    
    private async Task GetAddressAsync(double lat, double lon)
    {
        try
        {
            var placemarks = await Geocoding.Default.GetPlacemarksAsync(lat, lon);
            var placemark = placemarks?.FirstOrDefault();
            
            if (placemark != null)
            {
                Address = string.Join(", ", new[]
                {
                    placemark.SubThoroughfare,
                    placemark.Thoroughfare,
                    placemark.SubLocality,
                    placemark.Locality,
                    placemark.AdminArea,
                    placemark.CountryName
                }.Where(s => !string.IsNullOrEmpty(s)));
            }
        }
        catch
        {
            Address = "ไม่สามารถหาที่อยู่ได้";
        }
    }
    
    // Geocoding (address → coordinates)
    [RelayCommand]
    private async Task SearchAddressAsync(string address)
    {
        var locations = await Geocoding.Default.GetLocationsAsync(address);
        var location = locations?.FirstOrDefault();
        
        if (location != null)
        {
            Latitude = location.Latitude;
            Longitude = location.Longitude;
        }
    }
    
    // Open map app
    [RelayCommand]
    private async Task OpenMapAsync()
    {
        var location = new Location(Latitude, Longitude);
        var options = new MapLaunchOptions { Name = "ตำแหน่งของฉัน" };
        
        try
        {
            await Map.Default.OpenAsync(location, options);
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("ผิดพลาด", ex.Message, "ตกลง");
        }
    }
}
```

---

## Step 213: Local Notifications

```csharp
// ============================================
// Local Notifications (Plugin.LocalNotification)
// ============================================

// NuGet: Plugin.LocalNotification

using Plugin.LocalNotification;
using Plugin.LocalNotification.AndroidOption;
using Plugin.LocalNotification.iOSOption;

public class NotificationService
{
    private int _nextId = 1;
    
    // Simple notification
    public async Task ShowNotificationAsync(string title, string message)
    {
        var notification = new NotificationRequest
        {
            NotificationId = _nextId++,
            Title = title,
            Description = message,
            BadgeNumber = 1,
            Android = new AndroidOptions
            {
                ChannelId = "general",
                IconSmallName = new AndroidIcon("notification_icon")
            }
        };
        
        await LocalNotificationCenter.Current.Show(notification);
    }
    
    // Scheduled notification
    public async Task ScheduleNotificationAsync(string title, string message, 
        DateTime when)
    {
        var notification = new NotificationRequest
        {
            NotificationId = _nextId++,
            Title = title,
            Description = message,
            Schedule = new NotificationRequestSchedule
            {
                NotifyTime = when,
                RepeatType = NotificationRepeat.No
            },
            Android = new AndroidOptions
            {
                ChannelId = "reminder",
                Priority = AndroidPriority.High
            }
        };
        
        await LocalNotificationCenter.Current.Show(notification);
    }
    
    // Recurring notification
    public async Task ScheduleRecurringAsync(string title, string message, 
        TimeSpan dailyTime)
    {
        var today = DateTime.Today.Add(dailyTime);
        if (today < DateTime.Now)
            today = today.AddDays(1);
        
        var notification = new NotificationRequest
        {
            NotificationId = _nextId++,
            Title = title,
            Description = message,
            Schedule = new NotificationRequestSchedule
            {
                NotifyTime = today,
                RepeatType = NotificationRepeat.Daily
            }
        };
        
        await LocalNotificationCenter.Current.Show(notification);
    }
    
    // Cancel notification
    public void CancelNotification(int id)
        => LocalNotificationCenter.Current.Cancel(id);
    
    public void CancelAll()
        => LocalNotificationCenter.Current.CancelAll();
}

// ============================================
// Push Notifications (Firebase)
// ============================================

// NuGet: Plugin.Firebase.CloudMessaging
// Requires google-services.json (Android) and GoogleService-Info.plist (iOS)

public class PushNotificationService
{
    public async Task<string> GetTokenAsync()
    {
        // Firebase Token
        // return await CrossFirebaseCloudMessaging.Current.GetTokenAsync();
        return await Task.FromResult("mock-firebase-token");
    }
    
    public void OnTokenRefreshed(string newToken)
    {
        // Send new token to server
        Console.WriteLine($"FCM Token: {newToken}");
    }
    
    public void HandleNotification(string title, string body, 
        Dictionary<string, string> data)
    {
        Console.WriteLine($"Notification: {title} - {body}");
        
        // Navigate based on data
        if (data.TryGetValue("route", out var route))
        {
            MainThread.BeginInvokeOnMainThread(async () =>
            {
                await Shell.Current.GoToAsync(route);
            });
        }
    }
}
```

---

## Step 214: Permissions

```csharp
// ============================================
// Permissions Management
// ============================================

public class PermissionService
{
    public async Task<bool> RequestCameraAsync()
        => await CheckAndRequestAsync<Permissions.Camera>();
    
    public async Task<bool> RequestGalleryAsync()
        => await CheckAndRequestAsync<Permissions.Photos>();
    
    public async Task<bool> RequestLocationAsync()
        => await CheckAndRequestAsync<Permissions.LocationWhenInUse>();
    
    public async Task<bool> RequestLocationAlwaysAsync()
        => await CheckAndRequestAsync<Permissions.LocationAlways>();
    
    public async Task<bool> RequestMicrophoneAsync()
        => await CheckAndRequestAsync<Permissions.Microphone>();
    
    public async Task<bool> RequestStorageAsync()
        => await CheckAndRequestAsync<Permissions.StorageRead>();
    
    public async Task<bool> RequestNotificationsAsync()
        => await CheckAndRequestAsync<Permissions.PostNotifications>();
    
    private static async Task<bool> CheckAndRequestAsync<T>()
        where T : Permissions.BasePermission, new()
    {
        var status = await Permissions.CheckStatusAsync<T>();
        
        if (status == PermissionStatus.Granted)
            return true;
        
        if (status == PermissionStatus.Denied && 
            DeviceInfo.Current.Platform == DevicePlatform.iOS)
        {
            // On iOS, denied means user must go to Settings
            return false;
        }
        
        status = await Permissions.RequestAsync<T>();
        return status == PermissionStatus.Granted;
    }
    
    // Check if should show rationale
    public async Task<bool> ShouldShowRationaleAsync<T>()
        where T : Permissions.BasePermission, new()
    {
        if (DeviceInfo.Current.Platform != DevicePlatform.Android)
            return false;
        
        var status = await Permissions.CheckStatusAsync<T>();
        return status == PermissionStatus.Denied;
    }
    
    // Open app settings
    public void OpenSettings()
        => AppInfo.Current.ShowSettingsUI();
}
```

---

## Step 215: Device Info และ Platform Detection

```csharp
// ============================================
// Device Info
// ============================================

public class DeviceInfoService
{
    public static void LogDeviceInfo()
    {
        var info = DeviceInfo.Current;
        
        Console.WriteLine($"Model: {info.Model}");
        Console.WriteLine($"Manufacturer: {info.Manufacturer}");
        Console.WriteLine($"Name: {info.Name}");
        Console.WriteLine($"Version: {info.VersionString}");
        Console.WriteLine($"Platform: {info.Platform}");
        Console.WriteLine($"Idiom: {info.Idiom}"); // Phone, Tablet, Desktop, Watch, TV
        Console.WriteLine($"DeviceType: {info.DeviceType}"); // Physical, Virtual
    }
    
    // Check platform
    public static bool IsAndroid => DeviceInfo.Current.Platform == DevicePlatform.Android;
    public static bool IsIOS => DeviceInfo.Current.Platform == DevicePlatform.iOS;
    public static bool IsWindows => DeviceInfo.Current.Platform == DevicePlatform.WinUI;
    public static bool IsMacCatalyst => DeviceInfo.Current.Platform == DevicePlatform.MacCatalyst;
    
    // Check idiom
    public static bool IsPhone => DeviceInfo.Current.Idiom == DeviceIdiom.Phone;
    public static bool IsTablet => DeviceInfo.Current.Idiom == DeviceIdiom.Tablet;
    public static bool IsDesktop => DeviceInfo.Current.Idiom == DeviceIdiom.Desktop;
    
    // Battery
    public static void CheckBattery()
    {
        var battery = Battery.Default;
        Console.WriteLine($"Level: {battery.ChargeLevel:P0}");
        Console.WriteLine($"State: {battery.State}"); // Charging, Discharging, Full, NotCharging
        Console.WriteLine($"Source: {battery.PowerSource}"); // Battery, AC, USB, Wireless
        
        Battery.Default.BatteryInfoChanged += (sender, e) =>
        {
            Console.WriteLine($"Battery changed: {e.ChargeLevel:P0}");
        };
    }
    
    // Display info
    public static void CheckDisplay()
    {
        var display = DeviceDisplay.Current.MainDisplayInfo;
        Console.WriteLine($"Width: {display.Width}");
        Console.WriteLine($"Height: {display.Height}");
        Console.WriteLine($"Density: {display.Density}");
        Console.WriteLine($"Orientation: {display.Orientation}");
    }
    
    // Connectivity
    public static void CheckConnectivity()
    {
        var access = Connectivity.Current.NetworkAccess;
        var profiles = Connectivity.Current.ConnectionProfiles;
        
        Console.WriteLine($"Network: {access}");
        Console.WriteLine($"Profiles: {string.Join(", ", profiles)}");
        
        Connectivity.Current.ConnectivityChanged += (sender, e) =>
        {
            Console.WriteLine($"Connectivity: {e.NetworkAccess}");
        };
    }
}
```

---

## Step 216: Sensors (Accelerometer, Gyroscope)

```csharp
// ============================================
// Device Sensors
// ============================================

public partial class SensorViewModel : ObservableObject
{
    [ObservableProperty]
    private string _accelerometerData = string.Empty;
    
    [ObservableProperty]
    private string _gyroscopeData = string.Empty;
    
    [ObservableProperty]
    private string _magnetometerData = string.Empty;
    
    [ObservableProperty]
    private string _compassData = string.Empty;
    
    [ObservableProperty]
    private string _shakeStatus = string.Empty;
    
    public SensorViewModel()
    {
        Accelerometer.Default.ReadingChanged += OnAccelerometerChanged;
        Gyroscope.Default.ReadingChanged += OnGyroscopeChanged;
        Magnetometer.Default.ReadingChanged += OnMagnetometerChanged;
        Compass.Default.ReadingChanged += OnCompassChanged;
        Accelerometer.Default.ShakeDetected += OnShakeDetected;
    }
    
    [RelayCommand]
    private void ToggleAccelerometer()
    {
        if (Accelerometer.Default.IsMonitoring)
            Accelerometer.Default.Stop();
        else
            Accelerometer.Default.Start(SensorSpeed.UI);
    }
    
    [RelayCommand]
    private void ToggleGyroscope()
    {
        if (Gyroscope.Default.IsMonitoring)
            Gyroscope.Default.Stop();
        else
            Gyroscope.Default.Start(SensorSpeed.UI);
    }
    
    [RelayCommand]
    private void ToggleCompass()
    {
        if (Compass.Default.IsMonitoring)
            Compass.Default.Stop();
        else
            Compass.Default.Start(SensorSpeed.UI);
    }
    
    private void OnAccelerometerChanged(object? sender, AccelerometerChangedEventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            var data = e.Reading;
            AccelerometerData = $"X: {data.Acceleration.X:F2}, Y: {data.Acceleration.Y:F2}, Z: {data.Acceleration.Z:F2}";
        });
    }
    
    private void OnGyroscopeChanged(object? sender, GyroscopeChangedEventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            var data = e.Reading;
            GyroscopeData = $"X: {data.AngularVelocity.X:F2}, Y: {data.AngularVelocity.Y:F2}, Z: {data.AngularVelocity.Z:F2}";
        });
    }
    
    private void OnMagnetometerChanged(object? sender, MagnetometerChangedEventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            var data = e.Reading;
            MagnetometerData = $"X: {data.MagneticField.X:F2}, Y: {data.MagneticField.Y:F2}, Z: {data.MagneticField.Z:F2}";
        });
    }
    
    private void OnCompassChanged(object? sender, CompassChangedEventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            var heading = e.Reading.HeadingMagneticNorth;
            var direction = heading switch
            {
                < 22.5 or >= 337.5 => "เหนือ ↑",
                < 67.5 => "ตะวันออกเฉียงเหนือ ↗",
                < 112.5 => "ตะวันออก →",
                < 157.5 => "ตะวันออกเฉียงใต้ ↘",
                < 202.5 => "ใต้ ↓",
                < 247.5 => "ตะวันตกเฉียงใต้ ↙",
                < 292.5 => "ตะวันตก ←",
                _ => "ตะวันตกเฉียงเหนือ ↖"
            };
            CompassData = $"{heading:F0}° {direction}";
        });
    }
    
    private void OnShakeDetected(object? sender, EventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            ShakeStatus = "เขย่าแล้ว! " + DateTime.Now.ToString("HH:mm:ss");
        });
    }
}
```

---

## Step 217: Haptic Feedback และ Vibration

```csharp
// ============================================
// Haptic Feedback
// ============================================

public class HapticService
{
    // Simple vibration
    public static void Vibrate(double seconds = 0.5)
    {
        try
        {
            Vibration.Default.Vibrate(TimeSpan.FromSeconds(seconds));
        }
        catch (FeatureNotSupportedException)
        {
            // Not supported
        }
    }
    
    // Cancel vibration
    public static void StopVibration()
    {
        try
        {
            Vibration.Default.Cancel();
        }
        catch { }
    }
    
    // Haptic Feedback
    public static void LightImpact()
    {
        try
        {
            HapticFeedback.Default.Perform(HapticFeedbackType.Click);
        }
        catch (FeatureNotSupportedException)
        {
            // Not supported on this device
        }
    }
    
    public static void MediumImpact()
    {
        try
        {
            HapticFeedback.Default.Perform(HapticFeedbackType.LongPress);
        }
        catch (FeatureNotSupportedException) { }
    }
    
    // Success/Error patterns
    public static async Task SuccessPatternAsync()
    {
        Vibrate(0.05);
        await Task.Delay(50);
        Vibrate(0.05);
    }
    
    public static async Task ErrorPatternAsync()
    {
        for (int i = 0; i < 3; i++)
        {
            Vibrate(0.1);
            await Task.Delay(150);
        }
    }
}
```

---

## Step 218: Share และ Clipboard

```csharp
// ============================================
// Share
// ============================================

public class SharingService
{
    // Share text
    public Task ShareTextAsync(string text, string? title = null)
        => Share.Default.RequestAsync(new ShareTextRequest
        {
            Text = text,
            Title = title ?? "แชร์"
        });
    
    // Share URL
    public Task ShareUrlAsync(string url, string? title = null)
        => Share.Default.RequestAsync(new ShareTextRequest
        {
            Text = url,
            Title = title ?? "แชร์ลิงก์"
        });
    
    // Share file
    public Task ShareFileAsync(string filePath, string? title = null)
        => Share.Default.RequestAsync(new ShareFileRequest
        {
            Title = title ?? "แชร์ไฟล์",
            File = new ShareFile(filePath)
        });
    
    // Share multiple files
    public Task ShareMultipleFilesAsync(IEnumerable<string> filePaths)
    {
        var files = filePaths.Select(p => new ShareFile(p)).ToList();
        return Share.Default.RequestAsync(new ShareMultipleFilesRequest
        {
            Title = "แชร์ไฟล์",
            Files = files
        });
    }
    
    // Clipboard
    public Task CopyToClipboardAsync(string text)
        => Clipboard.Default.SetTextAsync(text);
    
    public async Task<string?> PasteFromClipboardAsync()
    {
        if (!Clipboard.Default.HasText) return null;
        return await Clipboard.Default.GetTextAsync();
    }
    
    // Open URL
    public Task OpenUrlAsync(string url)
        => Browser.Default.OpenAsync(url, BrowserLaunchMode.SystemPreferred);
    
    // Open URL in app
    public Task OpenUrlInAppAsync(string url)
        => Browser.Default.OpenAsync(url, new BrowserLaunchOptions
        {
            LaunchMode = BrowserLaunchMode.InApp,
            TitleMode = BrowserTitleMode.Show,
            PreferredToolbarColor = Color.FromArgb("#512BD4"),
            PreferredControlColor = Colors.White
        });
    
    // Send email
    public Task SendEmailAsync(string to, string subject, string body)
        => Email.Default.ComposeAsync(new EmailMessage
        {
            Subject = subject,
            Body = body,
            To = new List<string> { to }
        });
    
    // Make phone call
    public Task CallPhoneAsync(string number)
        => PhoneDialer.Default.Open(number);
    
    // Send SMS
    public Task SendSmsAsync(string to, string message)
        => Sms.Default.ComposeAsync(new SmsMessage(message, new[] { to }));
}
```

---

## Step 219: Barcode Scanner

```csharp
// ============================================
// Barcode / QR Code Scanner
// ============================================

// NuGet: ZXing.Net.Maui.Controls

// In MauiProgram.cs:
// builder.UseBarcodeReader();

// XAML:
/*
<zxing:CameraBarcodeReaderView
    x:Name="barcodeView"
    BarcodeDetectionFrameRate="5"
    IsDetecting="True"
    BarcodesDetected="OnBarcodesDetected">
    <zxing:CameraBarcodeReaderView.Options>
        <zxing:BarcodeReaderOptions
            Formats="QrCode,Ean13,Code128"
            AutoRotate="True" />
    </zxing:CameraBarcodeReaderView.Options>
</zxing:CameraBarcodeReaderView>
*/

public partial class BarcodeViewModel : ObservableObject
{
    [ObservableProperty]
    private bool _isDetecting = true;
    
    [ObservableProperty]
    private string _lastResult = string.Empty;
    
    [ObservableProperty]
    private string _barcodeFormat = string.Empty;
    
    public void OnBarcodesDetected(BarcodeDetectionEventArgs e)
    {
        IsDetecting = false; // Stop scanning
        
        var barcode = e.Results.FirstOrDefault();
        if (barcode != null)
        {
            MainThread.BeginInvokeOnMainThread(async () =>
            {
                LastResult = barcode.Value;
                BarcodeFormat = barcode.Format.ToString();
                
                await ProcessBarcodeAsync(barcode.Value);
                
                IsDetecting = true; // Resume scanning
            });
        }
    }
    
    private async Task ProcessBarcodeAsync(string value)
    {
        // Handle different barcode types
        if (value.StartsWith("http://") || value.StartsWith("https://"))
        {
            bool open = await Shell.Current.DisplayAlert(
                "QR Code", $"เปิดลิงก์: {value}?", "เปิด", "ยกเลิก");
            if (open)
                await Browser.Default.OpenAsync(value);
        }
        else if (long.TryParse(value, out _))
        {
            // Product barcode - look up in database
            LastResult = $"EAN: {value}";
        }
        else
        {
            await Shell.Current.DisplayAlert("QR Code", value, "ตกลง");
        }
    }
    
    // Generate QR Code
    public ImageSource GenerateQrCode(string content)
    {
        // Using ZXing writer
        // var writer = new BarcodeWriter<SkiaSharp.SKBitmap>
        // {
        //     Format = BarcodeFormat.QR_CODE,
        //     Options = new QrCodeEncodingOptions { Width = 300, Height = 300 }
        // };
        // var bitmap = writer.Write(content);
        // var image = bitmap.Encode(SkiaSharp.SKEncodedImageFormat.Png, 100).ToArray();
        // return ImageSource.FromStream(() => new MemoryStream(image));
        
        return "placeholder.png"; // Replace with actual implementation
    }
}

// Placeholder types (in real project, from ZXing.Net.Maui.Controls)
public class BarcodeDetectionEventArgs : EventArgs
{
    public IEnumerable<BarcodeResult> Results { get; } = new List<BarcodeResult>();
}
public class BarcodeResult
{
    public string Value { get; } = string.Empty;
    public BarcodeFormats Format { get; } = BarcodeFormats.QrCode;
}
public enum BarcodeFormats { QrCode, Ean13, Code128 }
```

---

## Step 220: Biometric Authentication

```csharp
// ============================================
// Biometric Authentication (Fingerprint / Face ID)
// ============================================

// NuGet: Plugin.Fingerprint

using Plugin.Fingerprint;
using Plugin.Fingerprint.Abstractions;

public class BiometricService
{
    public async Task<bool> IsBiometricAvailableAsync()
    {
        var availability = await CrossFingerprint.Current.GetAvailabilityAsync();
        return availability == FingerprintAvailability.Available;
    }
    
    public async Task<string> GetBiometricTypeAsync()
    {
        var availability = await CrossFingerprint.Current.GetAvailabilityAsync();
        return availability switch
        {
            FingerprintAvailability.Available => "Biometric",
            FingerprintAvailability.NoPermission => "ไม่มีสิทธิ์",
            FingerprintAvailability.NoApi => "ไม่รองรับ",
            FingerprintAvailability.NoFingerprint => "ไม่มี fingerprint ลงทะเบียน",
            FingerprintAvailability.NoImplementation => "ไม่รองรับ",
            _ => "ไม่ทราบ"
        };
    }
    
    public async Task<bool> AuthenticateAsync(string reason = "ยืนยันตัวตน")
    {
        var isAvailable = await IsBiometricAvailableAsync();
        if (!isAvailable) return false;
        
        var authConfig = new AuthenticationRequestConfiguration(
            "Authentication Required", reason)
        {
            AllowAlternativeAuthentication = true,
            CancelTitle = "ยกเลิก",
            FallbackTitle = "ใช้รหัสผ่าน"
        };
        
        var result = await CrossFingerprint.Current.AuthenticateAsync(authConfig);
        
        return result.Authenticated;
    }
    
    public async Task<bool> AuthenticateWithCallbackAsync(
        string reason,
        Action onSuccess,
        Action<string> onFailed)
    {
        var result = await AuthenticateAsync(reason);
        
        if (result)
            onSuccess();
        else
            onFailed("การยืนยันตัวตนไม่สำเร็จ");
        
        return result;
    }
}

// Usage in ViewModel
public partial class SecurePageViewModel : ObservableObject
{
    private readonly BiometricService _biometric;
    
    [ObservableProperty]
    private bool _isAuthenticated;
    
    public SecurePageViewModel(BiometricService biometric) => _biometric = biometric;
    
    [RelayCommand]
    private async Task AuthenticateAsync()
    {
        bool success = await _biometric.AuthenticateAsync("ยืนยันตัวตนเพื่อเข้าถึงข้อมูล");
        
        if (success)
        {
            IsAuthenticated = true;
            // Load sensitive data
        }
        else
        {
            await Shell.Current.DisplayAlert("ผิดพลาด", "การยืนยันตัวตนไม่สำเร็จ", "ตกลง");
        }
    }
}
```

---

## สรุป Part 22

ใน Part 22 เราได้เรียนรู้:

1. **Camera/MediaPicker** - ถ่ายรูป, เลือก gallery, บันทึกวิดีโอ
2. **Geolocation** - GPS coordinates, reverse geocoding, open map
3. **Local Notifications** - Push, scheduled, recurring
4. **Push Notifications** - Firebase Cloud Messaging (FCM)
5. **Permissions** - Check, request, rationale
6. **Device Info** - Platform, model, battery, display
7. **Sensors** - Accelerometer, Gyroscope, Compass
8. **Haptic Feedback** - Vibration patterns
9. **Share/Clipboard** - Share content, copy/paste
10. **Barcode Scanner** - QR code, barcodes
11. **Biometric Auth** - Fingerprint, Face ID

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 22 | Steps 211-220*
