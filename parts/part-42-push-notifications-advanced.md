# Part 42: Push Notifications Advanced
## Steps 411-420: FCM, APNs, Rich Notifications, Notification Actions

---

## Step 411: Firebase Cloud Messaging Setup

```csharp
// ============================================
// FCM Setup for Android
// ============================================

// NuGet: Plugin.Firebase.CloudMessaging (or native)

// AndroidManifest.xml additions:
/*
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<service android:name=".FirebaseMessagingService"
         android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
*/

// Platforms/Android/FirebaseMessagingService.cs
[Android.App.Service(Exported = false)]
[Android.App.IntentFilter(
    new[] { "com.google.firebase.MESSAGING_EVENT" },
    Priority = int.MaxValue)]
public class MyFirebaseMessagingService : 
    Firebase.Messaging.FirebaseMessagingService
{
    public override void OnNewToken(string token)
    {
        base.OnNewToken(token);
        // Save token and send to server
        var service = MauiApplication.Current.Services
            .GetRequiredService<IPushNotificationService>();
        service.UpdateTokenAsync(token).GetAwaiter().GetResult();
    }
    
    public override void OnMessageReceived(
        Firebase.Messaging.RemoteMessage message)
    {
        base.OnMessageReceived(message);
        
        var notification = new PushNotification
        {
            Title = message.GetNotification()?.Title ?? message.Data.GetValueOrDefault("title"),
            Body = message.GetNotification()?.Body ?? message.Data.GetValueOrDefault("body"),
            Data = message.Data.ToDictionary(kv => kv.Key, kv => (string)kv.Value),
            ImageUrl = message.Data.GetValueOrDefault("imageUrl"),
            Action = message.Data.GetValueOrDefault("action")
        };
        
        var service = MauiApplication.Current.Services
            .GetRequiredService<IPushNotificationService>();
        service.HandleNotificationReceivedAsync(notification).GetAwaiter().GetResult();
    }
}
```

---

## Step 412: Push Notification Service

```csharp
// ============================================
// Cross-platform Push Notification Service
// ============================================

public class PushNotification
{
    public string? Title { get; set; }
    public string? Body { get; set; }
    public Dictionary<string, string> Data { get; set; } = new();
    public string? ImageUrl { get; set; }
    public string? Action { get; set; }
    public string? NotificationId { get; set; }
}

public interface IPushNotificationService
{
    event EventHandler<PushNotification> NotificationReceived;
    event EventHandler<PushNotification> NotificationTapped;
    
    Task<string?> GetTokenAsync();
    Task UpdateTokenAsync(string token);
    Task HandleNotificationReceivedAsync(PushNotification notification);
    Task HandleNotificationTappedAsync(PushNotification notification);
    Task RequestPermissionAsync();
    Task SubscribeToTopicAsync(string topic);
    Task UnsubscribeFromTopicAsync(string topic);
}

public class PushNotificationService : IPushNotificationService
{
    private readonly IServerApi _api;
    private readonly IPreferences _prefs;
    
    public event EventHandler<PushNotification>? NotificationReceived;
    public event EventHandler<PushNotification>? NotificationTapped;
    
    public PushNotificationService(IServerApi api, IPreferences prefs)
    {
        _api = api;
        _prefs = prefs;
    }
    
    public async Task<string?> GetTokenAsync()
    {
#if ANDROID
        return await Firebase.Messaging.FirebaseMessaging.Instance
            .GetToken().AsAsync<Java.Lang.String>() as string;
#elif IOS
        return await Task.FromResult(
            UIKit.UIDevice.CurrentDevice.IdentifierForVendor?.AsString());
#else
        return null;
#endif
    }
    
    public async Task UpdateTokenAsync(string token)
    {
        var prevToken = _prefs.Get("fcm_token", string.Empty);
        if (prevToken == token) return;
        
        _prefs.Set("fcm_token", token);
        await _api.RegisterDeviceTokenAsync(token);
    }
    
    public async Task HandleNotificationReceivedAsync(PushNotification notification)
    {
        // Show local notification if app is in foreground
        var appState = Application.Current?.Windows.FirstOrDefault()?.Page != null;
        if (appState)
            await ShowLocalNotificationAsync(notification);
        
        NotificationReceived?.Invoke(this, notification);
    }
    
    public async Task HandleNotificationTappedAsync(PushNotification notification)
    {
        NotificationTapped?.Invoke(this, notification);
        
        // Navigate based on action
        await NavigateFromNotificationAsync(notification);
    }
    
    private async Task NavigateFromNotificationAsync(PushNotification notification)
    {
        if (notification.Action == null) return;
        
        await MainThread.InvokeOnMainThreadAsync(async () =>
        {
            switch (notification.Action)
            {
                case "view_order":
                    var orderId = notification.Data.GetValueOrDefault("orderId");
                    await Shell.Current.GoToAsync($"//orders/{orderId}");
                    break;
                    
                case "view_product":
                    var productId = notification.Data.GetValueOrDefault("productId");
                    await Shell.Current.GoToAsync($"//products/{productId}");
                    break;
                    
                case "view_chat":
                    await Shell.Current.GoToAsync("//chat");
                    break;
            }
        });
    }
    
    private async Task ShowLocalNotificationAsync(PushNotification notification)
    {
        // Use Plugin.LocalNotification or similar
        await Task.CompletedTask;
    }
    
    public async Task RequestPermissionAsync()
    {
#if IOS
        var (status, _) = await UserNotifications.UNUserNotificationCenter.Current
            .RequestAuthorizationAsync(
                UserNotifications.UNAuthorizationOptions.Alert |
                UserNotifications.UNAuthorizationOptions.Badge |
                UserNotifications.UNAuthorizationOptions.Sound);
        
        if (status)
            UIKit.UIApplication.SharedApplication.RegisterForRemoteNotifications();
#elif ANDROID
        if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.Tiramisu)
        {
            var status = await Permissions.RequestAsync<Permissions.PostNotifications>();
        }
#endif
    }
    
    public async Task SubscribeToTopicAsync(string topic)
    {
#if ANDROID
        await Firebase.Messaging.FirebaseMessaging.Instance
            .SubscribeToTopic(topic).AsAsync();
#endif
    }
    
    public async Task UnsubscribeFromTopicAsync(string topic)
    {
#if ANDROID
        await Firebase.Messaging.FirebaseMessaging.Instance
            .UnsubscribeFromTopic(topic).AsAsync();
#endif
    }
}
```

---

## Step 413: Rich Notifications (Android)

```csharp
// ============================================
// Rich Notifications with Actions
// ============================================

// Platforms/Android/RichNotificationService.cs
public class AndroidNotificationService
{
    private const string ChannelId = "main_channel";
    private const string OrdersChannelId = "orders_channel";
    private const string ChatChannelId = "chat_channel";
    
    public void CreateNotificationChannels()
    {
        if (Android.OS.Build.VERSION.SdkInt < Android.OS.BuildVersionCodes.O) return;
        
        var manager = (Android.App.NotificationManager)Android.App.Application.Context
            .GetSystemService(Android.Content.Context.NotificationService)!;
        
        // Main channel
        var mainChannel = new Android.App.NotificationChannel(
            ChannelId, "ทั่วไป", Android.App.NotificationImportance.Default)
        {
            Description = "การแจ้งเตือนทั่วไป"
        };
        
        // Orders channel - high importance
        var ordersChannel = new Android.App.NotificationChannel(
            OrdersChannelId, "คำสั่งซื้อ", Android.App.NotificationImportance.High)
        {
            Description = "การอัปเดตสถานะคำสั่งซื้อ"
        };
        ordersChannel.EnableVibration(true);
        ordersChannel.EnableLights(true);
        ordersChannel.LightColor = Android.Graphics.Color.Blue;
        
        // Chat channel
        var chatChannel = new Android.App.NotificationChannel(
            ChatChannelId, "แชท", Android.App.NotificationImportance.High)
        {
            Description = "ข้อความแชท"
        };
        
        manager.CreateNotificationChannels(new[] { mainChannel, ordersChannel, chatChannel });
    }
    
    public void ShowOrderNotification(int orderId, string status, decimal total)
    {
        var context = Android.App.Application.Context;
        
        // Tap action - open order detail
        var tapIntent = new Android.Content.Intent(context, typeof(MainActivity));
        tapIntent.PutExtra("action", "view_order");
        tapIntent.PutExtra("orderId", orderId.ToString());
        tapIntent.SetFlags(Android.Content.ActivityFlags.SingleTop);
        
        var pendingIntent = Android.App.PendingIntent.GetActivity(
            context, orderId, tapIntent,
            Android.App.PendingIntentFlags.UpdateCurrent | Android.App.PendingIntentFlags.Immutable);
        
        // Action buttons
        var viewAction = new Android.App.Notification.Action.Builder(
            Android.Resource.Drawable.IcMenuView,
            "ดูคำสั่งซื้อ", pendingIntent).Build();
        
        var notification = new Android.App.Notification.Builder(context, OrdersChannelId)
            .SetContentTitle($"คำสั่งซื้อ #{orderId}")
            .SetContentText($"สถานะ: {status} | ยอด {total:C0}")
            .SetSmallIcon(Resource.Drawable.notification_icon)
            .SetAutoCancel(true)
            .SetContentIntent(pendingIntent)
            .AddAction(viewAction)
            .Build();
        
        var manager = Android.App.Application.Context
            .GetSystemService(Android.Content.Context.NotificationService) 
            as Android.App.NotificationManager;
        
        manager?.Notify(orderId, notification);
    }
    
    // Big picture notification
    public void ShowImageNotification(string title, string text, byte[] imageBytes)
    {
        var context = Android.App.Application.Context;
        var bitmap = Android.Graphics.BitmapFactory.DecodeByteArray(imageBytes, 0, imageBytes.Length);
        
        var bigPicture = new Android.App.Notification.BigPictureStyle()
            .BigPicture(bitmap)
            .SetSummaryText(text);
        
        var notification = new Android.App.Notification.Builder(context, ChannelId)
            .SetContentTitle(title)
            .SetContentText(text)
            .SetSmallIcon(Resource.Drawable.notification_icon)
            .SetStyle(bigPicture)
            .SetAutoCancel(true)
            .Build();
        
        var manager = Android.App.Application.Context
            .GetSystemService(Android.Content.Context.NotificationService)
            as Android.App.NotificationManager;
        
        manager?.Notify(new Random().Next(), notification);
    }
}
```

---

## Step 414: APNs (iOS) Setup

```csharp
// ============================================
// APNs Setup for iOS
// ============================================

// Platforms/iOS/AppDelegate.cs
public class AppDelegate : MauiUIApplicationDelegate
{
    protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();
    
    public override bool FinishedLaunching(
        UIKit.UIApplication application,
        Foundation.NSDictionary launchOptions)
    {
        // Handle notification tapped when app was killed
        if (launchOptions?.ObjectForKey(
            UIKit.UIApplication.LaunchOptionsRemoteNotificationKey) 
            is Foundation.NSDictionary notification)
        {
            HandleNotificationFromLaunch(notification);
        }
        
        return base.FinishedLaunching(application, launchOptions);
    }
    
    public override void RegisteredForRemoteNotifications(
        UIKit.UIApplication application, Foundation.NSData deviceToken)
    {
        // Convert token to string
        var token = deviceToken.Description
            .Trim('<', '>')
            .Replace(" ", "");
        
        var service = MauiUIApplication.Current.Services
            .GetRequiredService<IPushNotificationService>();
        service.UpdateTokenAsync(token).GetAwaiter().GetResult();
    }
    
    public override void ReceivedRemoteNotification(
        UIKit.UIApplication application,
        Foundation.NSDictionary userInfo)
    {
        var notification = ParseApnsPayload(userInfo);
        var service = MauiUIApplication.Current.Services
            .GetRequiredService<IPushNotificationService>();
        service.HandleNotificationReceivedAsync(notification).GetAwaiter().GetResult();
    }
    
    private static PushNotification ParseApnsPayload(Foundation.NSDictionary userInfo)
    {
        var aps = userInfo.ObjectForKey(new Foundation.NSString("aps")) as Foundation.NSDictionary;
        var alert = aps?.ObjectForKey(new Foundation.NSString("alert")) as Foundation.NSDictionary;
        
        return new PushNotification
        {
            Title = alert?.ObjectForKey(new Foundation.NSString("title"))?.ToString(),
            Body = alert?.ObjectForKey(new Foundation.NSString("body"))?.ToString(),
        };
    }
    
    private void HandleNotificationFromLaunch(Foundation.NSDictionary notification)
    {
        // App was opened from notification
    }
}
```

---

## Step 415: Local Notifications

```csharp
// ============================================
// Local Notifications
// ============================================

// NuGet: Plugin.LocalNotification

public class LocalNotificationService
{
    public async Task ScheduleAsync(LocalNotification notification)
    {
        var request = new NotificationRequest
        {
            NotificationId = notification.Id,
            Title = notification.Title,
            Description = notification.Body,
            BadgeNumber = notification.BadgeCount,
            Schedule = new NotificationRequestSchedule
            {
                NotifyTime = notification.ScheduledFor
            },
            Android = new AndroidOptions
            {
                ChannelId = "reminders",
                Priority = AndroidNotificationPriority.High,
                AutoCancel = true
            },
            iOS = new iOSOptions
            {
                HideForegroundAlert = false,
                PlayForegroundSound = true
            }
        };
        
        if (!string.IsNullOrEmpty(notification.ImagePath))
        {
            // Rich notification with image
            request.iOS = new iOSOptions
            {
                AttachmentPath = notification.ImagePath
            };
        }
        
        await LocalNotificationCenter.Current.Show(request);
    }
    
    public async Task CancelAsync(int notificationId)
        => await LocalNotificationCenter.Current.Cancel(notificationId);
    
    public async Task CancelAllAsync()
        => await LocalNotificationCenter.Current.CancelAll();
    
    // Schedule daily reminder
    public async Task ScheduleDailyReminderAsync(TimeSpan time, string message)
    {
        var next = DateTime.Today.Add(time);
        if (next < DateTime.Now) next = next.AddDays(1);
        
        await ScheduleAsync(new LocalNotification
        {
            Id = 9999,
            Title = "แจ้งเตือนประจำวัน",
            Body = message,
            ScheduledFor = next
        });
    }
}

public class LocalNotification
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public DateTime? ScheduledFor { get; set; }
    public int BadgeCount { get; set; }
    public string? ImagePath { get; set; }
}
```

---

## Step 416: Notification Preferences

```csharp
// ============================================
// User Notification Preferences
// ============================================

public class NotificationPreferences
{
    public bool OrderUpdates { get; set; } = true;
    public bool PromotionsDeals { get; set; } = true;
    public bool ChatMessages { get; set; } = true;
    public bool SystemAlerts { get; set; } = true;
    public bool DailyReminders { get; set; } = false;
    public TimeSpan? QuietHoursStart { get; set; }
    public TimeSpan? QuietHoursEnd { get; set; }
    public string Sound { get; set; } = "default";
}

public class NotificationPreferenceService
{
    private readonly IPreferences _prefs;
    
    public NotificationPreferenceService(IPreferences prefs) => _prefs = prefs;
    
    public NotificationPreferences Load()
    {
        return new NotificationPreferences
        {
            OrderUpdates = _prefs.Get("notif_orders", true),
            PromotionsDeals = _prefs.Get("notif_promo", true),
            ChatMessages = _prefs.Get("notif_chat", true),
            SystemAlerts = _prefs.Get("notif_system", true),
            DailyReminders = _prefs.Get("notif_daily", false),
        };
    }
    
    public void Save(NotificationPreferences prefs)
    {
        _prefs.Set("notif_orders", prefs.OrderUpdates);
        _prefs.Set("notif_promo", prefs.PromotionsDeals);
        _prefs.Set("notif_chat", prefs.ChatMessages);
        _prefs.Set("notif_system", prefs.SystemAlerts);
        _prefs.Set("notif_daily", prefs.DailyReminders);
    }
    
    public bool ShouldNotify(string category)
    {
        var prefs = Load();
        
        // Check quiet hours
        if (IsQuietHours(prefs)) return false;
        
        return category switch
        {
            "order" => prefs.OrderUpdates,
            "promo" => prefs.PromotionsDeals,
            "chat" => prefs.ChatMessages,
            "system" => prefs.SystemAlerts,
            _ => true
        };
    }
    
    private static bool IsQuietHours(NotificationPreferences prefs)
    {
        if (!prefs.QuietHoursStart.HasValue || !prefs.QuietHoursEnd.HasValue)
            return false;
        
        var now = DateTime.Now.TimeOfDay;
        var start = prefs.QuietHoursStart.Value;
        var end = prefs.QuietHoursEnd.Value;
        
        return start < end
            ? now >= start && now < end
            : now >= start || now < end; // crosses midnight
    }
}
```

---

## Step 417: Badge Count Management

```csharp
// ============================================
// App Badge & Notification Count
// ============================================

public class BadgeService
{
    private int _unreadCount;
    
    public int UnreadCount => _unreadCount;
    
    public void SetBadgeCount(int count)
    {
        _unreadCount = Math.Max(0, count);
        UpdatePlatformBadge(_unreadCount);
    }
    
    public void Increment() => SetBadgeCount(_unreadCount + 1);
    
    public void Decrement() => SetBadgeCount(_unreadCount - 1);
    
    public void Clear() => SetBadgeCount(0);
    
    private static void UpdatePlatformBadge(int count)
    {
#if IOS
        MainThread.BeginInvokeOnMainThread(() =>
        {
            UIKit.UIApplication.SharedApplication.ApplicationIconBadgeNumber = count;
        });
#elif ANDROID
        // Android badge requires ShortcutBadger or similar
        // ShortcutBadger.applyCount(context, count);
#endif
    }
}
```

---

## Step 418: Notification Analytics

```csharp
// ============================================
// Track Notification Engagement
// ============================================

public class NotificationAnalytics
{
    private readonly SQLiteConnection _db;
    
    public NotificationAnalytics(SQLiteConnection db)
    {
        _db = db;
        _db.CreateTable<NotificationEvent>();
    }
    
    public void TrackReceived(string notificationId, string category)
    {
        Insert(notificationId, category, "received");
    }
    
    public void TrackOpened(string notificationId, string category)
    {
        Insert(notificationId, category, "opened");
    }
    
    public void TrackDismissed(string notificationId, string category)
    {
        Insert(notificationId, category, "dismissed");
    }
    
    private void Insert(string notifId, string category, string action)
    {
        _db.Insert(new NotificationEvent
        {
            NotificationId = notifId,
            Category = category,
            Action = action,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public NotificationStats GetStats(DateTime from, DateTime to)
    {
        var events = _db.Table<NotificationEvent>()
            .Where(e => e.Timestamp >= from && e.Timestamp <= to)
            .ToList();
        
        var received = events.Count(e => e.Action == "received");
        var opened = events.Count(e => e.Action == "opened");
        var dismissed = events.Count(e => e.Action == "dismissed");
        
        return new NotificationStats(
            Received: received,
            Opened: opened,
            Dismissed: dismissed,
            OpenRate: received > 0 ? (double)opened / received : 0);
    }
}

public class NotificationEvent
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string NotificationId { get; set; } = string.Empty;
    public string Category { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
}

public record NotificationStats(int Received, int Opened, int Dismissed, double OpenRate);
```

---

## Step 419: Server-Side Token Management

```csharp
// ============================================
// Backend Integration for Push
// ============================================

public class DeviceTokenApi
{
    private readonly HttpClient _http;
    
    public DeviceTokenApi(HttpClient http) => _http = http;
    
    public async Task RegisterTokenAsync(RegisterTokenRequest request)
    {
        var response = await _http.PostAsJsonAsync("api/devices/register", request);
        response.EnsureSuccessStatusCode();
    }
    
    public async Task UpdateTokenAsync(string oldToken, string newToken)
    {
        var response = await _http.PostAsJsonAsync("api/devices/update-token",
            new { oldToken, newToken });
        response.EnsureSuccessStatusCode();
    }
    
    public async Task UnregisterTokenAsync(string token)
    {
        var response = await _http.DeleteAsync($"api/devices/{Uri.EscapeDataString(token)}");
        response.EnsureSuccessStatusCode();
    }
    
    public async Task UpdatePreferencesAsync(string token, NotificationPreferences prefs)
    {
        var response = await _http.PutAsJsonAsync($"api/devices/{Uri.EscapeDataString(token)}/preferences",
            new
            {
                orders = prefs.OrderUpdates,
                promotions = prefs.PromotionsDeals,
                chat = prefs.ChatMessages,
                system = prefs.SystemAlerts
            });
        response.EnsureSuccessStatusCode();
    }
}

public class RegisterTokenRequest
{
    public string Token { get; set; } = string.Empty;
    public string Platform { get; set; } = string.Empty; // "android" | "ios"
    public string AppVersion { get; set; } = string.Empty;
    public string DeviceModel { get; set; } = string.Empty;
    public string Language { get; set; } = string.Empty;
}
```

---

## Step 420: Notification Testing

```csharp
// ============================================
// Push Notification Testing
// ============================================

// Debug helper to send test notifications
public class NotificationTestHelper
{
    private readonly IPushNotificationService _pushService;
    private readonly LocalNotificationService _localService;
    
    public NotificationTestHelper(
        IPushNotificationService pushService,
        LocalNotificationService localService)
    {
        _pushService = pushService;
        _localService = localService;
    }
    
    // Simulate receiving a push
    public async Task SimulateOrderNotificationAsync(int orderId)
    {
        var notification = new PushNotification
        {
            Title = $"คำสั่งซื้อ #{orderId} อัปเดต",
            Body = "สินค้าของคุณถูกจัดส่งแล้ว",
            Action = "view_order",
            Data = new Dictionary<string, string>
            {
                { "orderId", orderId.ToString() },
                { "status", "shipped" }
            }
        };
        
        await _pushService.HandleNotificationReceivedAsync(notification);
    }
    
    public async Task SimulateChatNotificationAsync(string from, string message)
    {
        await _pushService.HandleNotificationReceivedAsync(new PushNotification
        {
            Title = from,
            Body = message,
            Action = "view_chat",
            Data = new Dictionary<string, string>
            {
                { "fromUserId", "123" },
                { "chatId", "456" }
            }
        });
    }
    
    // Schedule local test notification
    public async Task ScheduleTestAsync(int seconds = 5)
    {
        await _localService.ScheduleAsync(new LocalNotification
        {
            Id = 1,
            Title = "Test Notification",
            Body = $"นี่คือการทดสอบ scheduled at {DateTime.Now:HH:mm:ss}",
            ScheduledFor = DateTime.Now.AddSeconds(seconds)
        });
    }
    
    // Print token to console
    public async Task PrintTokenAsync()
    {
        var token = await _pushService.GetTokenAsync();
        Console.WriteLine($"=== Push Token ===\n{token}\n==================");
    }
}
```

---

## สรุป Part 42

ใน Part 42 เราได้เรียนรู้:

1. **FCM Setup** - Android Firebase messaging service
2. **Push Service** - Cross-platform abstraction
3. **Rich Notifications** - BigPicture, action buttons (Android)
4. **APNs Setup** - iOS AppDelegate integration
5. **Local Notifications** - Scheduled reminders
6. **User Preferences** - Quiet hours, categories, sounds
7. **Badge Count** - iOS/Android badge management
8. **Analytics** - Track open/dismiss rates
9. **Server Integration** - Token registration API
10. **Testing Helpers** - Simulate notifications in debug

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 42 | Steps 411-420*
