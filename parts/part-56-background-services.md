# Part 56: Background Services & Workers
## Steps 551-560: Background Tasks, Sync, Scheduled Jobs

---

## Step 551: Background Service Base

```csharp
// ============================================
// Background Service Pattern
// ============================================

public abstract class BackgroundService : IDisposable
{
    private Task? _executingTask;
    private CancellationTokenSource? _cts;
    private bool _disposed;
    
    public bool IsRunning => _executingTask is { IsCompleted: false };
    
    public void Start()
    {
        if (IsRunning) return;
        
        _cts = new CancellationTokenSource();
        _executingTask = ExecuteAsync(_cts.Token);
    }
    
    public async Task StopAsync()
    {
        if (_executingTask == null) return;
        
        _cts?.Cancel();
        
        try
        {
            await Task.WhenAny(_executingTask, Task.Delay(5000));
        }
        catch (OperationCanceledException) { }
    }
    
    protected abstract Task ExecuteAsync(CancellationToken ct);
    
    public void Dispose()
    {
        if (_disposed) return;
        _cts?.Cancel();
        _cts?.Dispose();
        _disposed = true;
    }
}

// Periodic background service
public abstract class PeriodicBackgroundService : BackgroundService
{
    protected abstract TimeSpan Interval { get; }
    protected abstract Task RunAsync(CancellationToken ct);
    
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await RunAsync(ct);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                // Log but continue
                System.Diagnostics.Debug.WriteLine($"Background service error: {ex}");
            }
            
            try
            {
                await Task.Delay(Interval, ct);
            }
            catch (OperationCanceledException) { break; }
        }
    }
}
```

---

## Step 552: Data Sync Background Service

```csharp
// ============================================
// Background Sync Service
// ============================================

public class DataSyncService : PeriodicBackgroundService
{
    private readonly IDeltaSyncEngine _sync;
    private readonly IConnectivity _connectivity;
    
    protected override TimeSpan Interval => TimeSpan.FromMinutes(5);
    
    public DataSyncService(IDeltaSyncEngine sync, IConnectivity connectivity)
    {
        _sync = sync;
        _connectivity = connectivity;
    }
    
    protected override async Task RunAsync(CancellationToken ct)
    {
        if (_connectivity.NetworkAccess != NetworkAccess.Internet) return;
        
        await _sync.PushChangesAsync(ct);
        await _sync.PullChangesAsync(ct);
    }
}

// Cache cleanup service
public class CacheCleanupService : PeriodicBackgroundService
{
    private readonly ImageCacheService _imageCache;
    private readonly SQLiteConnection _db;
    
    protected override TimeSpan Interval => TimeSpan.FromHours(1);
    
    public CacheCleanupService(ImageCacheService imageCache, SQLiteConnection db)
    {
        _imageCache = imageCache;
        _db = db;
    }
    
    protected override async Task RunAsync(CancellationToken ct)
    {
        // Clean old images (older than 7 days)
        await _imageCache.CleanOldFilesAsync(TimeSpan.FromDays(7));
        
        // Clean old analytics events (already sent)
        var cutoff = DateTime.UtcNow.AddDays(-30);
        _db.Execute(
            "DELETE FROM AnalyticsEvent WHERE Sent = 1 AND Timestamp < ?",
            cutoff);
        
        // Clean old audit logs (older than 90 days)
        var auditCutoff = DateTime.UtcNow.AddDays(-90);
        _db.Execute(
            "DELETE FROM AuditEntry WHERE Timestamp < ?",
            auditCutoff);
    }
}
```

---

## Step 553: Android Foreground Service

```csharp
// ============================================
// Android Foreground Service (long-running)
// ============================================

#if ANDROID
[Android.App.Service(ForegroundServiceType = Android.Content.PM.ForegroundService.TypeDataSync)]
public class AndroidSyncService : Android.App.Service
{
    private CancellationTokenSource? _cts;
    private static readonly int NotificationId = 1001;
    
    public override Android.OS.IBinder? OnBind(Android.Content.Intent? intent) => null;
    
    public override Android.App.StartCommandResult OnStartCommand(
        Android.Content.Intent? intent, Android.App.StartCommandFlags flags, int startId)
    {
        StartForeground(NotificationId, BuildNotification("กำลังซิงค์ข้อมูล..."));
        
        _cts = new CancellationTokenSource();
        Task.Run(() => RunSyncAsync(_cts.Token));
        
        return Android.App.StartCommandResult.Sticky;
    }
    
    private async Task RunSyncAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                UpdateNotification("กำลังอัปโหลดข้อมูล...");
                // await _sync.PushChangesAsync(ct);
                
                UpdateNotification("กำลังดาวน์โหลดข้อมูล...");
                // await _sync.PullChangesAsync(ct);
                
                UpdateNotification("ซิงค์เสร็จแล้ว ✓");
                await Task.Delay(TimeSpan.FromMinutes(15), ct);
            }
            catch (OperationCanceledException) { break; }
        }
        
        StopForeground(Android.App.StopForegroundFlags.Remove);
        StopSelf();
    }
    
    private Android.App.Notification BuildNotification(string text)
    {
        var channelId = "sync_channel";
        
        var channel = new Android.App.NotificationChannel(
            channelId, "Sync Service",
            Android.App.NotificationImportance.Low);
        
        var manager = (Android.App.NotificationManager)
            GetSystemService(NotificationService)!;
        manager.CreateNotificationChannel(channel);
        
        return new Android.App.Notification.Builder(this, channelId)
            .SetContentTitle("MyShop")
            .SetContentText(text)
            .SetSmallIcon(Resource.Drawable.ic_sync)
            .SetOngoing(true)
            .Build();
    }
    
    private void UpdateNotification(string text)
    {
        var manager = (Android.App.NotificationManager)
            GetSystemService(NotificationService)!;
        manager.Notify(NotificationId, BuildNotification(text));
    }
    
    public override void OnDestroy()
    {
        _cts?.Cancel();
        base.OnDestroy();
    }
}
#endif
```

---

## Step 554: iOS Background Fetch

```csharp
// ============================================
// iOS Background Fetch
// ============================================

#if IOS
[Foundation.Register("AppDelegate")]
public partial class AppDelegate : MauiUIApplicationDelegate
{
    protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();
    
    // Called by iOS when background fetch is triggered
    public override async void PerformFetch(
        UIKit.UIApplication application,
        Action<UIKit.UIBackgroundFetchResult> completionHandler)
    {
        try
        {
            var services = IPlatformApplication.Current!.Services;
            var sync = services.GetRequiredService<IDeltaSyncEngine>();
            
            await sync.IncrementalSyncAsync();
            
            completionHandler(UIKit.UIBackgroundFetchResult.NewData);
        }
        catch
        {
            completionHandler(UIKit.UIBackgroundFetchResult.Failed);
        }
    }
}

// Enable background fetch in Info.plist:
// <key>UIBackgroundModes</key>
// <array>
//   <string>fetch</string>
//   <string>remote-notification</string>
// </array>

// Enable background fetch interval
public class iOSBackgroundFetchSetup
{
    public static void Enable()
    {
        UIKit.UIApplication.SharedApplication.SetMinimumBackgroundFetchInterval(
            UIKit.UIApplication.BackgroundFetchIntervalMinimum);
    }
}
#endif
```

---

## Step 555: Job Queue

```csharp
// ============================================
// Persistent Job Queue
// ============================================

public interface IJob
{
    string JobType { get; }
    Task ExecuteAsync(CancellationToken ct = default);
}

public class JobQueue
{
    private readonly SQLiteConnection _db;
    private readonly IServiceProvider _services;
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public JobQueue(SQLiteConnection db, IServiceProvider services)
    {
        _db = db;
        _services = services;
        _db.CreateTable<JobRecord>();
    }
    
    public void Enqueue<TJob>(object? payload = null) where TJob : IJob
    {
        _db.Insert(new JobRecord
        {
            JobType = typeof(TJob).FullName!,
            Payload = payload != null
                ? System.Text.Json.JsonSerializer.Serialize(payload) : null,
            Status = JobStatus.Pending,
            CreatedAt = DateTime.UtcNow,
            MaxAttempts = 3
        });
    }
    
    public async Task ProcessPendingAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var pending = _db.Table<JobRecord>()
                .Where(j => j.Status == JobStatus.Pending && j.Attempts < j.MaxAttempts)
                .OrderBy(j => j.CreatedAt)
                .Take(10)
                .ToList();
            
            foreach (var record in pending)
            {
                if (ct.IsCancellationRequested) break;
                await ExecuteJobAsync(record, ct);
            }
        }
        finally { _semaphore.Release(); }
    }
    
    private async Task ExecuteJobAsync(JobRecord record, CancellationToken ct)
    {
        record.Status = JobStatus.Running;
        record.Attempts++;
        record.LastAttemptAt = DateTime.UtcNow;
        _db.Update(record);
        
        try
        {
            var jobType = Type.GetType(record.JobType)
                ?? throw new InvalidOperationException($"Unknown job: {record.JobType}");
            
            var job = (IJob)_services.GetRequiredService(jobType);
            await job.ExecuteAsync(ct);
            
            record.Status = JobStatus.Completed;
            record.CompletedAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            record.Status = record.Attempts >= record.MaxAttempts
                ? JobStatus.Failed : JobStatus.Pending;
            record.LastError = ex.Message;
        }
        
        _db.Update(record);
    }
}

public class JobRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string JobType { get; set; } = string.Empty;
    public string? Payload { get; set; }
    public JobStatus Status { get; set; }
    public int Attempts { get; set; }
    public int MaxAttempts { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastAttemptAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public string? LastError { get; set; }
}

public enum JobStatus { Pending, Running, Completed, Failed }

// Example jobs
public class SendEmailJob : IJob
{
    public string JobType => nameof(SendEmailJob);
    private readonly IEmailService _email;
    
    public SendEmailJob(IEmailService email) => _email = email;
    
    public async Task ExecuteAsync(CancellationToken ct = default)
    {
        // Get payload from DI or static context
        await _email.SendAsync("test@email.com", "Subject", "Body");
    }
}
```

---

## Step 556: WorkManager-style Scheduler

```csharp
// ============================================
// Task Scheduler
// ============================================

public class TaskScheduler
{
    private readonly Dictionary<string, (TimeSpan Interval, Func<CancellationToken, Task> Task)> 
        _tasks = new();
    private readonly CancellationTokenSource _cts = new();
    private readonly List<Task> _running = new();
    
    public void Register(string name, TimeSpan interval, Func<CancellationToken, Task> task)
    {
        _tasks[name] = (interval, task);
    }
    
    public void Start()
    {
        foreach (var (name, (interval, task)) in _tasks)
        {
            _running.Add(RunTaskAsync(name, interval, task, _cts.Token));
        }
    }
    
    private async Task RunTaskAsync(
        string name, TimeSpan interval,
        Func<CancellationToken, Task> task, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await task(ct);
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Task {name} failed: {ex.Message}");
            }
            
            try { await Task.Delay(interval, ct); }
            catch (OperationCanceledException) { break; }
        }
    }
    
    public async Task StopAsync()
    {
        _cts.Cancel();
        await Task.WhenAll(_running);
    }
}

// Usage in App startup
public partial class App : Application
{
    private readonly TaskScheduler _scheduler;
    
    public App(
        DataSyncService sync,
        CacheCleanupService cleanup,
        AnalyticsService analytics)
    {
        var scheduler = new TaskScheduler();
        
        scheduler.Register("sync", TimeSpan.FromMinutes(5),
            ct => sync.RunAsync(ct));
        
        scheduler.Register("cleanup", TimeSpan.FromHours(1),
            ct => cleanup.RunAsync(ct));
        
        scheduler.Register("flush-analytics", TimeSpan.FromMinutes(2),
            ct => analytics.FlushAsync(null!, ct));
        
        scheduler.Start();
        _scheduler = scheduler;
        
        InitializeComponent();
        MainPage = new AppShell();
    }
}
```

---

## Step 557: Network Queue

```csharp
// ============================================
// Offline Request Queue
// ============================================

public class OfflineRequestQueue
{
    private readonly SQLiteConnection _db;
    private readonly HttpClient _http;
    private readonly IConnectivity _connectivity;
    
    public OfflineRequestQueue(SQLiteConnection db, HttpClient http, IConnectivity connectivity)
    {
        _db = db;
        _http = http;
        _connectivity = connectivity;
        _db.CreateTable<QueuedRequest>();
        
        // Process when online
        _connectivity.ConnectivityChanged += OnConnectivityChanged;
    }
    
    public void Queue(string method, string url, string? body = null, int priority = 0)
    {
        _db.Insert(new QueuedRequest
        {
            Method = method,
            Url = url,
            Body = body,
            Priority = priority,
            CreatedAt = DateTime.UtcNow
        });
    }
    
    private async void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
    {
        if (e.NetworkAccess == NetworkAccess.Internet)
            await ProcessQueueAsync();
    }
    
    public async Task ProcessQueueAsync(CancellationToken ct = default)
    {
        var pending = _db.Table<QueuedRequest>()
            .Where(r => !r.Processed)
            .OrderByDescending(r => r.Priority)
            .ThenBy(r => r.CreatedAt)
            .ToList();
        
        foreach (var request in pending)
        {
            if (ct.IsCancellationRequested) break;
            
            try
            {
                var message = new HttpRequestMessage(
                    new HttpMethod(request.Method), request.Url);
                
                if (request.Body != null)
                    message.Content = new StringContent(request.Body,
                        System.Text.Encoding.UTF8, "application/json");
                
                var response = await _http.SendAsync(message, ct);
                
                if (response.IsSuccessStatusCode)
                {
                    request.Processed = true;
                    request.ProcessedAt = DateTime.UtcNow;
                    _db.Update(request);
                }
                else if ((int)response.StatusCode >= 400 && (int)response.StatusCode < 500)
                {
                    // Client error - won't be fixed by retry
                    request.Processed = true;
                    request.Error = $"HTTP {(int)response.StatusCode}";
                    _db.Update(request);
                }
            }
            catch when (!ct.IsCancellationRequested) { /* retry later */ }
        }
    }
}

public class QueuedRequest
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Method { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;
    public string? Body { get; set; }
    public int Priority { get; set; }
    public bool Processed { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
}
```

---

## Step 558: Background Download Manager

```csharp
// ============================================
// Background Download Manager
// ============================================

public class DownloadManager
{
    private readonly SQLiteConnection _db;
    private readonly HttpClient _http;
    private readonly Dictionary<int, CancellationTokenSource> _activeCts = new();
    
    public event Action<DownloadProgress>? ProgressUpdated;
    
    public DownloadManager(SQLiteConnection db, HttpClient http)
    {
        _db = db;
        _http = http;
        _db.CreateTable<DownloadRecord>();
    }
    
    public int Enqueue(string url, string savePath, string? displayName = null)
    {
        var record = new DownloadRecord
        {
            Url = url,
            SavePath = savePath,
            DisplayName = displayName ?? Path.GetFileName(savePath),
            Status = DownloadStatus.Queued,
            CreatedAt = DateTime.UtcNow
        };
        _db.Insert(record);
        
        _ = StartDownloadAsync(record.Id);
        return record.Id;
    }
    
    public void Cancel(int downloadId)
    {
        if (_activeCts.TryGetValue(downloadId, out var cts))
        {
            cts.Cancel();
            _activeCts.Remove(downloadId);
        }
        
        var record = _db.Find<DownloadRecord>(downloadId);
        if (record != null)
        {
            record.Status = DownloadStatus.Cancelled;
            _db.Update(record);
        }
    }
    
    private async Task StartDownloadAsync(int downloadId)
    {
        var record = _db.Find<DownloadRecord>(downloadId);
        if (record == null) return;
        
        var cts = new CancellationTokenSource();
        _activeCts[downloadId] = cts;
        
        record.Status = DownloadStatus.Downloading;
        _db.Update(record);
        
        try
        {
            using var response = await _http.GetAsync(
                record.Url, HttpCompletionOption.ResponseHeadersRead, cts.Token);
            
            response.EnsureSuccessStatusCode();
            
            var totalBytes = response.Content.Headers.ContentLength ?? -1;
            var downloaded = 0L;
            
            Directory.CreateDirectory(Path.GetDirectoryName(record.SavePath)!);
            
            using var stream = await response.Content.ReadAsStreamAsync(cts.Token);
            using var file = File.Create(record.SavePath);
            
            var buffer = new byte[8192];
            int bytesRead;
            
            while ((bytesRead = await stream.ReadAsync(buffer, cts.Token)) > 0)
            {
                await file.WriteAsync(buffer.AsMemory(0, bytesRead), cts.Token);
                downloaded += bytesRead;
                
                var percent = totalBytes > 0 ? (int)(downloaded * 100 / totalBytes) : -1;
                ProgressUpdated?.Invoke(new DownloadProgress(downloadId, percent, downloaded, totalBytes));
            }
            
            record.Status = DownloadStatus.Completed;
            record.CompletedAt = DateTime.UtcNow;
            record.BytesDownloaded = downloaded;
        }
        catch (OperationCanceledException)
        {
            record.Status = DownloadStatus.Cancelled;
        }
        catch (Exception ex)
        {
            record.Status = DownloadStatus.Failed;
            record.Error = ex.Message;
        }
        finally
        {
            _activeCts.Remove(downloadId);
            _db.Update(record);
        }
    }
}

public class DownloadRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string Url { get; set; } = string.Empty;
    public string SavePath { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public DownloadStatus Status { get; set; }
    public long BytesDownloaded { get; set; }
    public string? Error { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}

public record DownloadProgress(int Id, int Percent, long Downloaded, long Total);
public enum DownloadStatus { Queued, Downloading, Completed, Failed, Cancelled }
```

---

## Step 559: Background Location (Android)

```csharp
// ============================================
// Background Location Tracking
// ============================================

#if ANDROID
[Android.App.Service(ForegroundServiceType = 
    Android.Content.PM.ForegroundService.TypeLocation)]
public class LocationTrackingService : Android.App.Service
{
    private Android.Locations.LocationManager? _locationManager;
    private LocationListener? _listener;
    
    public override Android.OS.IBinder? OnBind(Android.Content.Intent? intent) => null;
    
    public override Android.App.StartCommandResult OnStartCommand(
        Android.Content.Intent? intent, Android.App.StartCommandFlags flags, int startId)
    {
        StartForeground(2001, BuildNotification());
        StartTracking();
        return Android.App.StartCommandResult.Sticky;
    }
    
    private void StartTracking()
    {
        _locationManager = (Android.Locations.LocationManager)
            GetSystemService(Android.Content.Context.LocationService)!;
        
        _listener = new LocationListener(OnLocationUpdate);
        
        _locationManager.RequestLocationUpdates(
            Android.Locations.LocationManager.GpsProvider,
            5000, // min time ms
            10,   // min distance meters
            _listener);
    }
    
    private void OnLocationUpdate(Android.Locations.Location location)
    {
        var lat = location.Latitude;
        var lng = location.Longitude;
        
        // Save to database or broadcast
        Android.Content.Intent intent = new("com.myapp.LOCATION_UPDATE");
        intent.PutExtra("lat", lat);
        intent.PutExtra("lng", lng);
        SendBroadcast(intent);
    }
    
    private Android.App.Notification BuildNotification()
    {
        var channelId = "location_channel";
        var channel = new Android.App.NotificationChannel(
            channelId, "Location Tracking",
            Android.App.NotificationImportance.Low);
        
        ((Android.App.NotificationManager)GetSystemService(NotificationService)!)
            .CreateNotificationChannel(channel);
        
        return new Android.App.Notification.Builder(this, channelId)
            .SetContentTitle("กำลังติดตามตำแหน่ง")
            .SetContentText("แอปกำลังบันทึกเส้นทาง")
            .SetSmallIcon(Resource.Drawable.ic_location)
            .SetOngoing(true)
            .Build();
    }
    
    public override void OnDestroy()
    {
        _locationManager?.RemoveUpdates(_listener!);
        base.OnDestroy();
    }
    
    class LocationListener : Java.Lang.Object, Android.Locations.ILocationListener
    {
        private readonly Action<Android.Locations.Location> _callback;
        public LocationListener(Action<Android.Locations.Location> callback) => _callback = callback;
        public void OnLocationChanged(Android.Locations.Location location) => _callback(location);
        public void OnProviderDisabled(string provider) { }
        public void OnProviderEnabled(string provider) { }
        public void OnStatusChanged(string? provider, Android.Locations.Availability status, Android.OS.Bundle? extras) { }
    }
}
#endif
```

---

## Step 560: Background Processing Best Practices

```csharp
// ============================================
// Best Practices Summary
// ============================================

/*
 * ✅ Background Processing Rules:
 * 
 * 1. Lifetime Management
 *    □ Always use CancellationToken
 *    □ Implement IDisposable/IAsyncDisposable
 *    □ Stop services in App.OnSleep
 * 
 * 2. Battery Optimization
 *    □ Batch network calls
 *    □ Use exponential backoff
 *    □ Respect Doze mode (Android) / Background App Refresh (iOS)
 *    □ Minimum poll interval: 15 minutes
 * 
 * 3. Data Safety
 *    □ Use SQLite transactions
 *    □ Never lose data on crash (persistent job queue)
 *    □ Idempotent operations
 * 
 * 4. User Experience
 *    □ Show foreground notification if > few seconds
 *    □ Update UI on MainThread
 *    □ Clear user-facing progress when done
 * 
 * 5. Error Handling
 *    □ Log all background errors
 *    □ Retry with backoff
 *    □ Dead-letter queue for permanently failed jobs
 */

public class BackgroundServiceCoordinator
{
    private readonly List<BackgroundService> _services;
    
    public BackgroundServiceCoordinator(IEnumerable<BackgroundService> services)
        => _services = services.ToList();
    
    public void StartAll()
    {
        foreach (var service in _services)
            service.Start();
    }
    
    public async Task StopAllAsync()
    {
        await Task.WhenAll(_services.Select(s => s.StopAsync()));
    }
    
    // Hook into app lifecycle
    public void AttachToApp(Application app)
    {
        app.Resume += (_, _) => StartAll();
        app.Sleep += async (_, _) => await StopAllAsync();
    }
}
```

---

## สรุป Part 56

ใน Part 56 เราได้เรียนรู้:

1. **Background Service Base** - CancellationToken, periodic
2. **Data Sync Service** - Push/pull when online
3. **Android Foreground Service** - ForegroundServiceType
4. **iOS Background Fetch** - UIBackgroundModes
5. **Job Queue** - Persistent SQLite job queue
6. **Task Scheduler** - Multi-task coordinator
7. **Network Queue** - Offline request accumulation
8. **Download Manager** - Progress, cancel, resume
9. **Background Location** - GPS tracking service
10. **Best Practices** - Battery, safety, UX

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 56 | Steps 551-560*
