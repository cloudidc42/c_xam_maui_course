# Part 89: Push Notifications & In-App Messaging
## Steps 881-890: FCM, APNs, Local Notifications, Deep Links, Notification Center

---

## Step 881: Push Notification Setup

```csharp
// ============================================
// Firebase Cloud Messaging (FCM) Setup
// ============================================

// MauiProgram.cs additions:
// builder.Services.AddSingleton<IPushNotificationService, FcmPushNotificationService>();

// Android: google-services.json in project root
// iOS: GoogleService-Info.plist in project root
// NuGet: Plugin.Firebase.CloudMessaging

public interface IPushNotificationService
{
    event EventHandler<PushNotificationReceivedArgs> NotificationReceived;
    event EventHandler<PushNotificationTappedArgs> NotificationTapped;
    
    Task<string?> GetTokenAsync();
    Task<bool> RequestPermissionAsync();
    Task SubscribeToTopicAsync(string topic);
    Task UnsubscribeFromTopicAsync(string topic);
}

public record PushNotificationReceivedArgs(
    string Title, string Body, Dictionary<string, string> Data);

public record PushNotificationTappedArgs(
    string? Action, Dictionary<string, string> Data);

public class FcmPushNotificationService : IPushNotificationService
{
    private readonly ILogger<FcmPushNotificationService> _logger;
    
    public event EventHandler<PushNotificationReceivedArgs>? NotificationReceived;
    public event EventHandler<PushNotificationTappedArgs>? NotificationTapped;
    
    public FcmPushNotificationService(ILogger<FcmPushNotificationService> logger)
    {
        _logger = logger;
        SetupFcmHandlers();
    }
    
    private void SetupFcmHandlers()
    {
        // Firebase CloudMessaging handlers (using Plugin.Firebase)
        // CrossFirebaseCloudMessaging.Current.NotificationReceived += ...
        // CrossFirebaseCloudMessaging.Current.NotificationTapped += ...
    }
    
    public async Task<string?> GetTokenAsync()
    {
        try
        {
            // return await CrossFirebaseCloudMessaging.Current.GetTokenAsync();
            return await Task.FromResult("mock-fcm-token"); // placeholder
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get FCM token");
            return null;
        }
    }
    
    public async Task<bool> RequestPermissionAsync()
    {
#if IOS
        var result = await Permissions.RequestAsync<Permissions.PostNotifications>();
        return result == PermissionStatus.Granted;
#else
        if (OperatingSystem.IsAndroidVersionAtLeast(33))
        {
            var result = await Permissions.RequestAsync<Permissions.PostNotifications>();
            return result == PermissionStatus.Granted;
        }
        return true;
#endif
    }
    
    public async Task SubscribeToTopicAsync(string topic)
    {
        // await CrossFirebaseCloudMessaging.Current.SubscribeToTopicAsync(topic);
        _logger.LogInformation("Subscribed to topic: {Topic}", topic);
        await Task.CompletedTask;
    }
    
    public async Task UnsubscribeFromTopicAsync(string topic)
    {
        // await CrossFirebaseCloudMessaging.Current.UnsubscribeFromTopicAsync(topic);
        await Task.CompletedTask;
    }
}
```

---

## Step 882: Device Token Registration

```csharp
// ============================================
// Register Device Token with Backend
// ============================================

public class DeviceRegistrationService
{
    private readonly HttpClient _http;
    private readonly IPushNotificationService _push;
    
    public DeviceRegistrationService(HttpClient http, IPushNotificationService push)
    {
        _http = http;
        _push = push;
    }
    
    public async Task RegisterAsync(string userId)
    {
        var token = await _push.GetTokenAsync();
        if (string.IsNullOrEmpty(token)) return;
        
        var registration = new DeviceRegistration
        {
            UserId = userId,
            PushToken = token,
            Platform = DeviceInfo.Platform.ToString().ToLower(),
            DeviceId = DeviceInfo.Name,
            AppVersion = AppInfo.VersionString,
            Locale = System.Globalization.CultureInfo.CurrentCulture.Name,
            Timezone = TimeZoneInfo.Local.Id
        };
        
        await _http.PostAsJsonAsync("/api/devices/register", registration);
        
        // Save locally for deregistration on logout
        Preferences.Set("registered_push_token", token);
        
        // Subscribe to user-specific topic
        await _push.SubscribeToTopicAsync($"user_{userId}");
        
        // Subscribe to regional promotions
        await _push.SubscribeToTopicAsync("th_promotions");
    }
    
    public async Task UnregisterAsync(string userId)
    {
        var token = Preferences.Get("registered_push_token", "");
        if (string.IsNullOrEmpty(token)) return;
        
        await _http.DeleteAsync($"/api/devices/{Uri.EscapeDataString(token)}");
        await _push.UnsubscribeFromTopicAsync($"user_{userId}");
        
        Preferences.Remove("registered_push_token");
    }
}

public class DeviceRegistration
{
    public string UserId { get; set; } = "";
    public string PushToken { get; set; } = "";
    public string Platform { get; set; } = "";
    public string DeviceId { get; set; } = "";
    public string AppVersion { get; set; } = "";
    public string Locale { get; set; } = "";
    public string Timezone { get; set; } = "";
}
```

---

## Step 883: Local Notifications

```csharp
// ============================================
// Local Notification Service
// ============================================

// NuGet: Plugin.LocalNotification

public interface ILocalNotificationService
{
    Task SendAsync(string title, string body,
        NotificationPriority priority = NotificationPriority.Default,
        string? deepLink = null,
        Dictionary<string, string>? data = null);
    
    Task ScheduleAsync(string title, string body, DateTime fireAt, string? deepLink = null);
    Task CancelAsync(int id);
    Task CancelAllAsync();
}

public enum NotificationPriority { Low, Default, High }

public class LocalNotificationService : ILocalNotificationService
{
    private int _nextId = 1000;
    
    public async Task SendAsync(string title, string body,
        NotificationPriority priority = NotificationPriority.Default,
        string? deepLink = null,
        Dictionary<string, string>? data = null)
    {
        var id = _nextId++;
        
        // Plugin.LocalNotification usage (simplified):
        // LocalNotificationCenter.Current.Show(new NotificationRequest {
        //     NotificationId = id,
        //     Title = title,
        //     Description = body,
        //     ...
        // });
        
        await Task.CompletedTask;
    }
    
    public async Task ScheduleAsync(string title, string body, DateTime fireAt, string? deepLink = null)
    {
        var id = _nextId++;
        
        // LocalNotificationCenter.Current.Show(new NotificationRequest {
        //     NotificationId = id,
        //     Schedule = new NotificationRequestSchedule { NotifyTime = fireAt }
        //     ...
        // });
        
        await Task.CompletedTask;
    }
    
    public Task CancelAsync(int id)
    {
        // LocalNotificationCenter.Current.Cancel(id);
        return Task.CompletedTask;
    }
    
    public Task CancelAllAsync()
    {
        // LocalNotificationCenter.Current.CancelAll();
        return Task.CompletedTask;
    }
}
```

---

## Step 884: Deep Link Navigation

```csharp
// ============================================
// Deep Link & Universal Link Handler
// ============================================

// AndroidManifest.xml:
// <intent-filter android:autoVerify="true">
//   <action android:name="android.intent.action.VIEW" />
//   <category android:name="android.intent.category.DEFAULT" />
//   <category android:name="android.intent.category.BROWSABLE" />
//   <data android:scheme="https" android:host="fooddelivery.th" />
// </intent-filter>
//
// iOS Info.plist: CFBundleURLSchemes: ["fooddelivery"]
// Associated Domains: applinks:fooddelivery.th

public class DeepLinkService
{
    private readonly ILogger<DeepLinkService> _logger;
    
    public DeepLinkService(ILogger<DeepLinkService> logger)
    {
        _logger = logger;
        // Hook into MAUI app lifecycle
        App.Current!.HandlerChanged += OnHandlerChanged;
    }
    
    private void OnHandlerChanged(object? sender, EventArgs e)
    {
#if ANDROID
        var activity = Microsoft.Maui.ApplicationModel.Platform.CurrentActivity;
        if (activity?.Intent?.Data != null)
            HandleDeepLinkAsync(activity.Intent.Data.ToString()!).GetAwaiter().GetResult();
#elif IOS
        // Handled via AppDelegate.OpenUrl / ContinueUserActivity
#endif
    }
    
    public async Task HandleDeepLinkAsync(string url)
    {
        _logger.LogInformation("Deep link: {Url}", url);
        
        // Parse: fooddelivery://restaurant/abc123
        //        https://fooddelivery.th/order/xyz456
        
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri)) return;
        
        var route = ParseRoute(uri);
        
        if (route != null)
            await Shell.Current.GoToAsync(route);
    }
    
    private string? ParseRoute(Uri uri)
    {
        var path = uri.AbsolutePath.TrimStart('/');
        var segments = path.Split('/');
        
        return (segments[0], segments.Length) switch
        {
            ("restaurant", 2) => $"//restaurant-detail?id={segments[1]}",
            ("order", 2) => $"//order-tracking?orderId={segments[1]}",
            ("promo", 2) => $"//promotions?code={segments[1]}",
            ("menu-item", 2) => $"//menu-item-detail?id={segments[1]}",
            ("profile", _) => "//profile",
            _ => null
        };
    }
}
```

---

## Step 885: Notification Preferences

```csharp
// ============================================
// User Notification Preferences
// ============================================

public class NotificationPreferences
{
    public bool OrderStatusUpdates { get; set; } = true;
    public bool PromotionAlerts { get; set; } = true;
    public bool NewRestaurants { get; set; } = false;
    public bool LoyaltyPointUpdates { get; set; } = true;
    public bool DeliveryReminders { get; set; } = true;
    public bool WeeklyDigest { get; set; } = false;
    
    // Quiet hours
    public TimeSpan QuietFrom { get; set; } = new(22, 0, 0);
    public TimeSpan QuietTo { get; set; } = new(8, 0, 0);
    
    public bool IsQuietHour(DateTime time)
    {
        var t = time.TimeOfDay;
        if (QuietFrom < QuietTo)
            return t >= QuietFrom && t <= QuietTo;
        return t >= QuietFrom || t <= QuietTo; // wraps midnight
    }
}

public class NotificationPreferencesViewModel : BaseViewModel
{
    private readonly HttpClient _http;
    private readonly IPushNotificationService _push;
    
    [ObservableProperty] private NotificationPreferences _prefs = new();
    
    public NotificationPreferencesViewModel(HttpClient http, IPushNotificationService push)
    {
        _http = http;
        _push = push;
    }
    
    public async Task LoadAsync()
    {
        var result = await _http.GetFromJsonAsync<NotificationPreferences>("/api/notifications/prefs");
        if (result != null) Prefs = result;
    }
    
    public async Task SaveAsync()
    {
        await _http.PutAsJsonAsync("/api/notifications/prefs", Prefs);
        
        // Sync topic subscriptions
        if (Prefs.PromotionAlerts)
            await _push.SubscribeToTopicAsync("th_promotions");
        else
            await _push.UnsubscribeFromTopicAsync("th_promotions");
    }
}
```

---

## Step 886: In-App Notification Center

```csharp
// ============================================
// In-App Notification Center
// ============================================

public class NotificationCenterViewModel : BaseViewModel
{
    private readonly SQLiteAsyncConnection _db;
    private readonly IPushNotificationService _push;
    
    [ObservableProperty] private ObservableCollection<InAppNotification> _notifications = new();
    [ObservableProperty] private int _unreadCount;
    
    public NotificationCenterViewModel(SQLiteAsyncConnection db, IPushNotificationService push)
    {
        _db = db;
        _push = push;
        _db.CreateTableAsync<InAppNotification>().Wait();
        
        _push.NotificationReceived += OnPushReceived;
    }
    
    public async Task LoadAsync()
    {
        var items = await _db.Table<InAppNotification>()
            .OrderByDescending(n => n.ReceivedAt)
            .Take(50)
            .ToListAsync();
        
        Notifications = new ObservableCollection<InAppNotification>(items);
        UnreadCount = Notifications.Count(n => !n.IsRead);
    }
    
    public async Task MarkReadAsync(string notificationId)
    {
        var n = Notifications.FirstOrDefault(x => x.Id == notificationId);
        if (n == null || n.IsRead) return;
        
        n.IsRead = true;
        await _db.UpdateAsync(n);
        UnreadCount = Notifications.Count(x => !x.IsRead);
    }
    
    public async Task MarkAllReadAsync()
    {
        foreach (var n in Notifications.Where(x => !x.IsRead))
        {
            n.IsRead = true;
            await _db.UpdateAsync(n);
        }
        UnreadCount = 0;
    }
    
    public async Task DeleteAsync(string notificationId)
    {
        var n = Notifications.FirstOrDefault(x => x.Id == notificationId);
        if (n == null) return;
        
        await _db.DeleteAsync(n);
        Notifications.Remove(n);
        UnreadCount = Notifications.Count(x => !x.IsRead);
    }
    
    private async void OnPushReceived(object? _, PushNotificationReceivedArgs args)
    {
        var notification = new InAppNotification
        {
            Id = Guid.NewGuid().ToString(),
            Title = args.Title,
            Body = args.Body,
            Type = args.Data.GetValueOrDefault("type", "general"),
            DeepLink = args.Data.GetValueOrDefault("deep_link"),
            IsRead = false,
            ReceivedAt = DateTime.UtcNow
        };
        
        await _db.InsertAsync(notification);
        
        MainThread.BeginInvokeOnMainThread(() =>
        {
            Notifications.Insert(0, notification);
            UnreadCount++;
        });
    }
}

public class InAppNotification
{
    [PrimaryKey] public string Id { get; set; } = "";
    public string Title { get; set; } = "";
    public string Body { get; set; } = "";
    public string Type { get; set; } = ""; // order_update, promo, loyalty, general
    public string? DeepLink { get; set; }
    public bool IsRead { get; set; }
    public DateTime ReceivedAt { get; set; }
}
```

---

## Step 887: Smart Notification Timing

```csharp
// ============================================
// AI-Powered Notification Timing
// ============================================

public class SmartNotificationScheduler
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ILocalNotificationService _notifications;
    
    public SmartNotificationScheduler(SQLiteAsyncConnection db, ILocalNotificationService notifications)
    {
        _db = db;
        _notifications = notifications;
    }
    
    public async Task ScheduleLunchReminderAsync(string userId)
    {
        // Predict best time based on order history
        var bestHour = await PredictBestHourAsync(userId, "lunch");
        
        var today = DateTime.Today.AddHours(bestHour);
        if (today < DateTime.Now) today = today.AddDays(1);
        
        await _notifications.ScheduleAsync(
            "ถึงเวลาสั่งข้าวกลางวันแล้ว! 🍱",
            "ร้านโปรดของคุณพร้อมรับออเดอร์แล้ว",
            today);
    }
    
    private async Task<int> PredictBestHourAsync(string userId, string mealType)
    {
        // Most common order hour for this meal type
        var result = await _db.QueryAsync<HourCount>(
            @"SELECT strftime('%H', CreatedAt) as Hour, COUNT(*) as Count
              FROM Orders WHERE UserId = ?
              AND strftime('%H', CreatedAt) BETWEEN ? AND ?
              GROUP BY Hour ORDER BY Count DESC LIMIT 1",
            userId,
            mealType == "lunch" ? "11" : "17",
            mealType == "lunch" ? "14" : "21");
        
        return int.TryParse(result?.FirstOrDefault()?.Hour, out var hour) ? hour : 12;
    }
    
    private record HourCount(string Hour, int Count);
}
```

---

## Step 888: Notification Templates

```csharp
// ============================================
// Notification Template Engine
// ============================================

public class NotificationTemplates
{
    // Order status templates
    public static (string Title, string Body) OrderConfirmed(string restaurantName) =>
        ("ออเดอร์ได้รับการยืนยันแล้ว ✅",
         $"{restaurantName} ได้รับออเดอร์ของคุณและกำลังเตรียมอาหาร");
    
    public static (string Title, string Body) OrderPickedUp(string riderName) =>
        ("ไรเดอร์รับออเดอร์แล้ว 🛵",
         $"{riderName} กำลังนำอาหารมาส่งคุณ");
    
    public static (string Title, string Body) OrderDelivered() =>
        ("อาหารมาถึงแล้ว! 🎉",
         "รับอาหารได้เลย ขอบคุณที่ใช้บริการ");
    
    // Promo templates
    public static (string Title, string Body) FlashSale(string discount, string expiry) =>
        ($"Flash Sale {discount}! ⚡",
         $"ส่วนลดพิเศษ หมดเวลา {expiry} รีบใช้งานด่วน");
    
    public static (string Title, string Body) LoyaltyPoints(int points, int balance) =>
        ("คุณได้รับคะแนนสะสม 🌟",
         $"ได้รับ {points} คะแนน ยอดรวม {balance} คะแนน");
    
    // Rich notification (with image + action buttons)
    public static NotificationPayload BuildRichOrder(string orderId, string imagUrl) =>
        new NotificationPayload(
            Title: "ออเดอร์ #" + orderId[..6].ToUpper(),
            Body: "แตะเพื่อติดตามออเดอร์",
            ImageUrl: imagUrl,
            Actions: new[]
            {
                new NotificationAction("track", "ติดตาม", $"fooddelivery://order/{orderId}"),
                new NotificationAction("cancel", "ยกเลิก", $"fooddelivery://cancel/{orderId}")
            });
}

public record NotificationPayload(
    string Title, string Body, string? ImageUrl,
    NotificationAction[] Actions);

public record NotificationAction(string Id, string Label, string DeepLink);
```

---

## Step 889: Notification Analytics

```csharp
// ============================================
// Notification Delivery & Engagement Analytics
// ============================================

public class NotificationAnalytics
{
    private readonly IAnalytics _analytics;
    
    public NotificationAnalytics(IAnalytics analytics) => _analytics = analytics;
    
    public void TrackSent(string type, string userId)
        => _analytics.Track("notification_sent", new Dictionary<string, object>
        {
            ["type"] = type,
            ["user_id"] = userId
        });
    
    public void TrackDelivered(string notificationId)
        => _analytics.Track("notification_delivered", new Dictionary<string, object>
        {
            ["notification_id"] = notificationId
        });
    
    public void TrackOpened(string notificationId, string? deepLink)
        => _analytics.Track("notification_opened", new Dictionary<string, object>
        {
            ["notification_id"] = notificationId,
            ["has_deep_link"] = deepLink != null
        });
    
    public void TrackDismissed(string notificationId)
        => _analytics.Track("notification_dismissed", new Dictionary<string, object>
        {
            ["notification_id"] = notificationId
        });
    
    // CTR = opened / delivered
    public double CalculateCtr(int opened, int delivered)
        => delivered > 0 ? (double)opened / delivered : 0;
}
```

---

## Step 890: Notification Tests

```csharp
// ============================================
// Push Notification Tests
// ============================================

[TestFixture]
public class NotificationTests
{
    [Test]
    public void NotificationPreferences_QuietHour_WrapsAcrossMidnight()
    {
        var prefs = new NotificationPreferences
        {
            QuietFrom = new TimeSpan(22, 0, 0),
            QuietTo = new TimeSpan(8, 0, 0)
        };
        
        Assert.That(prefs.IsQuietHour(DateTime.Today.AddHours(23)), Is.True);
        Assert.That(prefs.IsQuietHour(DateTime.Today.AddHours(7)), Is.True);
        Assert.That(prefs.IsQuietHour(DateTime.Today.AddHours(12)), Is.False);
    }
    
    [Test]
    public void DeepLinkService_OrderUrl_ParsesCorrectly()
    {
        var svc = new DeepLinkService(null!);
        // Access private ParseRoute via reflection
        var method = typeof(DeepLinkService).GetMethod("ParseRoute",
            System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
        
        var uri = new Uri("https://fooddelivery.th/order/abc-123");
        var route = (string?)method?.Invoke(svc, new object[] { uri });
        
        Assert.That(route, Does.Contain("order-tracking"));
        Assert.That(route, Does.Contain("abc-123"));
    }
    
    [Test]
    public void Templates_OrderConfirmed_ContainsRestaurantName()
    {
        var (title, body) = NotificationTemplates.OrderConfirmed("ร้านครัวไทย");
        Assert.That(body, Contains.Substring("ร้านครัวไทย"));
    }
    
    [Test]
    public async Task NotificationCenter_MarkRead_DecreasesUnreadCount()
    {
        var db = new SQLiteAsyncConnection(":memory:");
        var center = new NotificationCenterViewModel(db, new FakePushService());
        
        var n = new InAppNotification
        {
            Id = "n1", Title = "Test", Body = "Test",
            Type = "general", IsRead = false, ReceivedAt = DateTime.UtcNow
        };
        
        await db.CreateTableAsync<InAppNotification>();
        await db.InsertAsync(n);
        await center.LoadAsync();
        
        Assert.That(center.UnreadCount, Is.EqualTo(1));
        
        await center.MarkReadAsync("n1");
        
        Assert.That(center.UnreadCount, Is.EqualTo(0));
    }
}

public class FakePushService : IPushNotificationService
{
    public event EventHandler<PushNotificationReceivedArgs>? NotificationReceived;
    public event EventHandler<PushNotificationTappedArgs>? NotificationTapped;
    
    public Task<string?> GetTokenAsync() => Task.FromResult<string?>("fake-token");
    public Task<bool> RequestPermissionAsync() => Task.FromResult(true);
    public Task SubscribeToTopicAsync(string topic) => Task.CompletedTask;
    public Task UnsubscribeFromTopicAsync(string topic) => Task.CompletedTask;
}
```

---

## สรุป Part 89

ใน Part 89 เราได้เรียนรู้:

1. **FCM Setup** - Plugin.Firebase integration, token retrieval, topic subscribe
2. **Device Registration** - Token sync to backend, locale & timezone
3. **Local Notifications** - Plugin.LocalNotification, schedule, cancel
4. **Deep Links** - Universal links, custom scheme, route parsing
5. **Notification Preferences** - Quiet hours, topic opt-in/out
6. **In-App Notification Center** - SQLite storage, unread badge, mark read
7. **Smart Timing** - SQL-based best-hour prediction, meal type
8. **Rich Templates** - Title/body factories, action buttons
9. **Notification Analytics** - CTR tracking, sent/delivered/opened
10. **Notification Tests** - Quiet hour wrap, deep link parse, unread count

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 89 | Steps 881-890*
