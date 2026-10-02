# Part 29: Background Services & Workers
## Steps 281-290: Background Processing

---

## Step 281: Background Task Overview

```csharp
// ============================================
// Background Processing ใน .NET MAUI
// ============================================

/*
 * Background Processing มีหลายแบบ:
 *
 * 1. Background Task (short-lived)
 *    - ทำงานขณะ app อยู่ background
 *    - iOS: ~30 วินาที, Android: ขึ้นอยู่กับ OS
 *    - ใช้สำหรับ: sync data, finish upload
 *
 * 2. Background Fetch / WorkManager
 *    - Periodic background execution
 *    - ไม่แน่นอนว่าจะ run เมื่อไหร่ (OS controls)
 *    - ใช้สำหรับ: update content, sync
 *
 * 3. Foreground Service
 *    - Android: Service with notification
 *    - ทำงานต่อเนื่องแม้ app ถูก minimize
 *    - ใช้สำหรับ: music player, GPS tracking, file download
 *
 * 4. Push Notification Wake
 *    - Wake app เมื่อมี push notification
 *    - iOS: Silent Push, Android: FCM Data Message
 */
```

---

## Step 282: Background Sync Service

```csharp
// ============================================
// Periodic Background Sync
// ============================================

public interface IBackgroundSyncService
{
    Task SyncAsync(CancellationToken ct);
    Task ScheduleSync(TimeSpan interval);
    void CancelSync();
}

public class BackgroundSyncService : IBackgroundSyncService
{
    private readonly IProductRepository _repository;
    private readonly IProductApiService _api;
    private readonly IConnectivity _connectivity;
    private CancellationTokenSource? _cts;
    
    public BackgroundSyncService(
        IProductRepository repository,
        IProductApiService api,
        IConnectivity connectivity)
    {
        _repository = repository;
        _api = api;
        _connectivity = connectivity;
    }
    
    public async Task SyncAsync(CancellationToken ct)
    {
        if (_connectivity.NetworkAccess != NetworkAccess.Internet)
            return;
        
        try
        {
            // Get items that need to be synced
            var pending = await _repository.GetPendingSyncAsync();
            
            foreach (var item in pending)
            {
                ct.ThrowIfCancellationRequested();
                
                try
                {
                    if (item.IsNew)
                        await _api.CreateAsync(item);
                    else if (item.IsModified)
                        await _api.UpdateAsync(item.Id, item);
                    else if (item.IsDeleted)
                        await _api.DeleteAsync(item.Id);
                    
                    await _repository.MarkSyncedAsync(item.Id);
                }
                catch (Exception ex) when (ex is not OperationCanceledException)
                {
                    // Log error, continue with next item
                    await _repository.MarkSyncFailedAsync(item.Id, ex.Message);
                }
            }
            
            // Pull new data from server
            var lastSync = Preferences.Default.Get("last_sync", DateTime.MinValue.ToString("O"));
            var since = DateTime.Parse(lastSync);
            
            var serverData = await _api.GetUpdatedSinceAsync(since, ct);
            
            foreach (var item in serverData)
            {
                ct.ThrowIfCancellationRequested();
                await _repository.UpsertAsync(item);
            }
            
            Preferences.Default.Set("last_sync", DateTime.UtcNow.ToString("O"));
        }
        catch (OperationCanceledException)
        {
            // Graceful cancellation
        }
    }
    
    public async Task ScheduleSync(TimeSpan interval)
    {
        CancelSync();
        _cts = new CancellationTokenSource();
        
        await Task.Run(async () =>
        {
            while (!_cts.Token.IsCancellationRequested)
            {
                try
                {
                    await SyncAsync(_cts.Token);
                    await Task.Delay(interval, _cts.Token);
                }
                catch (OperationCanceledException) { break; }
                catch { await Task.Delay(TimeSpan.FromMinutes(5), _cts.Token); }
            }
        }, _cts.Token);
    }
    
    public void CancelSync() => _cts?.Cancel();
}
```

---

## Step 283: App Lifecycle Integration

```csharp
// ============================================
// Background Task in App Lifecycle
// ============================================

public partial class App : Application
{
    private readonly IBackgroundSyncService _syncService;
    private CancellationTokenSource? _backgroundTaskCts;
    
    public App(IBackgroundSyncService syncService)
    {
        _syncService = syncService;
        InitializeComponent();
        MainPage = new AppShell();
    }
    
    protected override Window CreateWindow(IActivationState? activationState)
    {
        var window = base.CreateWindow(activationState);
        
        window.Stopped += OnAppStopped;
        window.Resumed += OnAppResumed;
        window.Destroying += OnAppDestroying;
        window.Activated += OnAppActivated;
        window.Deactivated += OnAppDeactivated;
        
        return window;
    }
    
    private async void OnAppActivated(object? sender, EventArgs e)
    {
        // App came to foreground - start sync
        await _syncService.ScheduleSync(TimeSpan.FromMinutes(5));
    }
    
    private void OnAppDeactivated(object? sender, EventArgs e)
    {
        // App going to background - request background task
        _backgroundTaskCts = new CancellationTokenSource();
        RequestBackgroundTask();
    }
    
    private async void OnAppStopped(object? sender, EventArgs e)
    {
        // Do a final sync before suspending
        if (_backgroundTaskCts != null)
        {
            _backgroundTaskCts.CancelAfter(TimeSpan.FromSeconds(25));
            await _syncService.SyncAsync(_backgroundTaskCts.Token);
        }
    }
    
    private async void OnAppResumed(object? sender, EventArgs e)
    {
        // App resumed - immediate sync
        await _syncService.SyncAsync(CancellationToken.None);
        await _syncService.ScheduleSync(TimeSpan.FromMinutes(5));
    }
    
    private void OnAppDestroying(object? sender, EventArgs e)
    {
        _syncService.CancelSync();
    }
    
    private void RequestBackgroundTask()
    {
        // Platform-specific: Register background task
        // iOS: BGTaskScheduler
        // Android: WorkManager
    }
}
```

---

## Step 284: WorkManager (Android)

```csharp
// ============================================
// Android WorkManager Integration
// ============================================

// In Android project: Platforms/Android/Workers/SyncWorker.cs
// using AndroidX.Work;

/*
[assembly: Preserve]
public class SyncWorker : Worker
{
    public SyncWorker(Context context, WorkerParameters parameters) 
        : base(context, parameters) { }
    
    public override Result DoWork()
    {
        try
        {
            // Get services from MAUI DI
            var syncService = IPlatformApplication.Current?.Services
                .GetService<IBackgroundSyncService>();
            
            if (syncService == null) return Result.InvokeFailure();
            
            // Use synchronous wrapper (Worker is not async)
            var cts = new CancellationTokenSource(TimeSpan.FromSeconds(25));
            syncService.SyncAsync(cts.Token).GetAwaiter().GetResult();
            
            return Result.InvokeSuccess();
        }
        catch
        {
            return Result.InvokeRetry();
        }
    }
}
*/

// Schedule worker from MauiProgram.cs or App.cs
public static class WorkManagerExtensions
{
    public static void SchedulePeriodicSync(TimeSpan interval)
    {
        // Android-specific code
        if (DeviceInfo.Current.Platform != DevicePlatform.Android) return;
        
        // This would use AndroidX WorkManager via platform channel
        // Actual implementation requires AndroidX.Work NuGet
    }
}
```

---

## Step 285: Download Manager

```csharp
// ============================================
// Background Download Manager
// ============================================

public class DownloadManager
{
    private readonly HttpClient _http;
    private readonly ConcurrentDictionary<string, DownloadTask> _activeTasks = new();
    
    public DownloadManager(HttpClient http) => _http = http;
    
    public DownloadTask StartDownload(string url, string fileName)
    {
        if (_activeTasks.TryGetValue(url, out var existing))
            return existing;
        
        var task = new DownloadTask(url, fileName);
        _activeTasks[url] = task;
        
        _ = ExecuteDownloadAsync(task);
        return task;
    }
    
    public void CancelDownload(string url)
    {
        if (_activeTasks.TryGetValue(url, out var task))
            task.Cancel();
    }
    
    private async Task ExecuteDownloadAsync(DownloadTask task)
    {
        try
        {
            task.Status = DownloadStatus.Downloading;
            
            using var response = await _http.GetAsync(task.Url, 
                HttpCompletionOption.ResponseHeadersRead, task.CancellationToken);
            
            response.EnsureSuccessStatusCode();
            
            var totalBytes = response.Content.Headers.ContentLength ?? -1;
            
            var savePath = Path.Combine(FileSystem.AppDataDirectory, "downloads", task.FileName);
            Directory.CreateDirectory(Path.GetDirectoryName(savePath)!);
            
            using var sourceStream = await response.Content.ReadAsStreamAsync(task.CancellationToken);
            using var destStream = File.Create(savePath);
            
            var buffer = new byte[8192];
            long downloadedBytes = 0;
            int bytesRead;
            
            while ((bytesRead = await sourceStream.ReadAsync(buffer, task.CancellationToken)) > 0)
            {
                await destStream.WriteAsync(buffer.AsMemory(0, bytesRead), task.CancellationToken);
                downloadedBytes += bytesRead;
                
                if (totalBytes > 0)
                    task.Progress = (double)downloadedBytes / totalBytes;
            }
            
            task.FilePath = savePath;
            task.Status = DownloadStatus.Completed;
        }
        catch (OperationCanceledException)
        {
            task.Status = DownloadStatus.Cancelled;
        }
        catch (Exception ex)
        {
            task.ErrorMessage = ex.Message;
            task.Status = DownloadStatus.Failed;
        }
        finally
        {
            _activeTasks.TryRemove(task.Url, out _);
        }
    }
}

public class DownloadTask : ObservableObject
{
    private double _progress;
    private DownloadStatus _status = DownloadStatus.Pending;
    private readonly CancellationTokenSource _cts = new();
    
    public string Url { get; }
    public string FileName { get; }
    public string? FilePath { get; set; }
    public string? ErrorMessage { get; set; }
    
    public CancellationToken CancellationToken => _cts.Token;
    
    public double Progress
    {
        get => _progress;
        set => SetProperty(ref _progress, value);
    }
    
    public DownloadStatus Status
    {
        get => _status;
        set => SetProperty(ref _status, value);
    }
    
    public string ProgressText => Status switch
    {
        DownloadStatus.Downloading => $"{Progress:P0}",
        DownloadStatus.Completed => "เสร็จแล้ว",
        DownloadStatus.Failed => $"ผิดพลาด: {ErrorMessage}",
        DownloadStatus.Cancelled => "ยกเลิก",
        _ => "รอ..."
    };
    
    public DownloadTask(string url, string fileName) { Url = url; FileName = fileName; }
    
    public void Cancel() => _cts.Cancel();
}

public enum DownloadStatus { Pending, Downloading, Completed, Failed, Cancelled }
```

---

## Step 286: Upload Queue

```csharp
// ============================================
// Persistent Upload Queue
// ============================================

public class UploadQueueService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly HttpClient _http;
    private readonly SemaphoreSlim _lock = new(1, 1);
    private bool _isProcessing;
    
    public UploadQueueService(SQLiteAsyncConnection db, HttpClient http)
    {
        _db = db;
        _http = http;
    }
    
    public async Task EnqueueAsync(UploadQueueItem item)
    {
        await _db.InsertAsync(item);
        _ = ProcessQueueAsync();
    }
    
    public async Task ProcessQueueAsync()
    {
        await _lock.WaitAsync();
        if (_isProcessing) { _lock.Release(); return; }
        _isProcessing = true;
        _lock.Release();
        
        try
        {
            while (true)
            {
                var item = await _db.Table<UploadQueueItem>()
                    .Where(x => x.Status == UploadStatus2.Pending || 
                                x.Status == UploadStatus2.Failed)
                    .OrderBy(x => x.CreatedAt)
                    .FirstOrDefaultAsync();
                
                if (item == null) break;
                
                await ProcessItemAsync(item);
            }
        }
        finally
        {
            _isProcessing = false;
        }
    }
    
    private async Task ProcessItemAsync(UploadQueueItem item)
    {
        item.Status = UploadStatus2.Uploading;
        item.AttemptCount++;
        item.LastAttemptAt = DateTime.UtcNow;
        await _db.UpdateAsync(item);
        
        try
        {
            if (!File.Exists(item.FilePath))
            {
                item.Status = UploadStatus2.Failed;
                item.ErrorMessage = "File not found";
                await _db.UpdateAsync(item);
                return;
            }
            
            using var content = new MultipartFormDataContent();
            using var fs = File.OpenRead(item.FilePath);
            content.Add(new StreamContent(fs), "file", Path.GetFileName(item.FilePath));
            
            foreach (var field in item.AdditionalFields ?? new())
                content.Add(new StringContent(field.Value), field.Key);
            
            var response = await _http.PostAsync(item.UploadUrl, content);
            response.EnsureSuccessStatusCode();
            
            item.Status = UploadStatus2.Completed;
            item.CompletedAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            item.ErrorMessage = ex.Message;
            item.Status = item.AttemptCount >= 3 
                ? UploadStatus2.Abandoned 
                : UploadStatus2.Failed;
        }
        
        await _db.UpdateAsync(item);
    }
    
    public async Task<List<UploadQueueItem>> GetQueueAsync()
        => await _db.Table<UploadQueueItem>()
            .Where(x => x.Status != UploadStatus2.Completed)
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync();
}

[SQLite.Table("upload_queue")]
public class UploadQueueItem
{
    [SQLite.PrimaryKey, SQLite.AutoIncrement] public int Id { get; set; }
    public string FilePath { get; set; } = string.Empty;
    public string UploadUrl { get; set; } = string.Empty;
    public UploadStatus2 Status { get; set; } = UploadStatus2.Pending;
    public int AttemptCount { get; set; }
    public string? ErrorMessage { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastAttemptAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    
    [SQLite.Ignore]
    public Dictionary<string, string>? AdditionalFields { get; set; }
}

public enum UploadStatus2 { Pending, Uploading, Completed, Failed, Abandoned }
```

---

## Step 287: Scheduled Notifications

```csharp
// ============================================
// Scheduled Local Notifications
// ============================================

public class ScheduledNotificationService
{
    // Schedule reminder notification
    public async Task ScheduleReminderAsync(
        int id,
        string title,
        string message,
        DateTimeOffset scheduledTime,
        bool repeating = false)
    {
        var request = new NotificationRequest
        {
            NotificationId = id,
            Title = title,
            Description = message,
            Schedule = new NotificationRequestSchedule
            {
                NotifyTime = scheduledTime.LocalDateTime,
                RepeatType = repeating ? NotificationRepeat.Daily : NotificationRepeat.No
            }
        };
        
        await LocalNotificationCenter.Current.Show(request);
    }
    
    // Schedule daily reminder
    public async Task ScheduleDailyReminderAsync(
        string title, string message, TimeSpan dailyTime)
    {
        var now = DateTime.Now;
        var scheduleTime = now.Date + dailyTime;
        if (scheduleTime <= now) scheduleTime = scheduleTime.AddDays(1);
        
        await ScheduleReminderAsync(
            id: 1001,
            title: title,
            message: message,
            scheduledTime: scheduleTime,
            repeating: true);
    }
    
    // Workout/medication reminders
    public async Task ScheduleMedicationRemindersAsync(
        string medicationName, List<TimeSpan> times)
    {
        var baseId = 2000;
        
        for (var i = 0; i < times.Count; i++)
        {
            var time = times[i];
            var scheduleTime = DateTime.Now.Date + time;
            if (scheduleTime <= DateTime.Now) scheduleTime = scheduleTime.AddDays(1);
            
            await ScheduleReminderAsync(
                id: baseId + i,
                title: $"ถึงเวลาทานยา: {medicationName}",
                message: $"กรุณาทาน {medicationName} ตามที่กำหนด",
                scheduledTime: scheduleTime,
                repeating: true);
        }
    }
    
    public async Task CancelAllAsync() 
        => await LocalNotificationCenter.Current.CancelAll();
    
    public async Task CancelAsync(int id) 
        => await LocalNotificationCenter.Current.Cancel(id);
}
```

---

## Step 288: Resource Monitoring

```csharp
// ============================================
// Battery & Resource Monitoring
// ============================================

public class ResourceMonitor
{
    private System.Timers.Timer? _monitorTimer;
    
    public event Action<ResourceState>? ResourceStateChanged;
    
    public void StartMonitoring(TimeSpan interval)
    {
        Battery.Default.BatteryInfoChanged += OnBatteryChanged;
        
        _monitorTimer = new System.Timers.Timer(interval.TotalMilliseconds);
        _monitorTimer.Elapsed += (_, _) => CheckResources();
        _monitorTimer.Start();
    }
    
    public void StopMonitoring()
    {
        Battery.Default.BatteryInfoChanged -= OnBatteryChanged;
        _monitorTimer?.Stop();
        _monitorTimer?.Dispose();
    }
    
    private void OnBatteryChanged(object? sender, BatteryInfoChangedEventArgs e)
    {
        var state = GetCurrentState();
        ResourceStateChanged?.Invoke(state);
    }
    
    private void CheckResources()
    {
        var state = GetCurrentState();
        ResourceStateChanged?.Invoke(state);
    }
    
    private ResourceState GetCurrentState()
    {
        return new ResourceState
        {
            BatteryLevel = Battery.Default.ChargeLevel,
            BatteryState = Battery.Default.State,
            IsCharging = Battery.Default.State == BatteryState.Charging,
            IsLowBattery = Battery.Default.ChargeLevel < 0.2,
            NetworkType = Connectivity.Current.ConnectionProfiles.Contains(ConnectionProfile.WiFi)
                ? NetworkType.WiFi
                : Connectivity.Current.ConnectionProfiles.Contains(ConnectionProfile.Cellular)
                    ? NetworkType.Cellular
                    : NetworkType.None,
            HasInternet = Connectivity.Current.NetworkAccess == NetworkAccess.Internet
        };
    }
}

public record ResourceState
{
    public double BatteryLevel { get; init; }
    public BatteryState BatteryState { get; init; }
    public bool IsCharging { get; init; }
    public bool IsLowBattery { get; init; }
    public NetworkType NetworkType { get; init; }
    public bool HasInternet { get; init; }
    
    // Should we defer expensive background tasks?
    public bool ShouldDeferBackgroundWork => IsLowBattery && !IsCharging;
    
    // Should we use cellular data?
    public bool AllowCellularData => NetworkType == NetworkType.WiFi || !IsLowBattery;
}

public enum NetworkType { None, WiFi, Cellular }
```

---

## Step 289: Task Scheduler

```csharp
// ============================================
// In-App Task Scheduler
// ============================================

public class AppTaskScheduler
{
    private readonly List<ScheduledTask> _tasks = new();
    private readonly CancellationTokenSource _cts = new();
    
    public void Schedule(string name, Func<CancellationToken, Task> work, 
        TimeSpan interval, bool runImmediately = false)
    {
        var task = new ScheduledTask(name, work, interval);
        _tasks.Add(task);
        
        var ct = _cts.Token;
        _ = RunScheduledTaskAsync(task, runImmediately, ct);
    }
    
    private async Task RunScheduledTaskAsync(
        ScheduledTask task, bool runImmediately, CancellationToken ct)
    {
        if (!runImmediately)
            await Task.Delay(task.Interval, ct);
        
        while (!ct.IsCancellationRequested)
        {
            task.LastRunAt = DateTime.UtcNow;
            task.RunCount++;
            
            try
            {
                await task.Work(ct);
                task.LastSuccess = DateTime.UtcNow;
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                task.LastError = ex.Message;
                task.ErrorCount++;
            }
            
            await Task.Delay(task.Interval, ct);
        }
    }
    
    public IReadOnlyList<ScheduledTask> GetTasks() => _tasks.AsReadOnly();
    
    public void Stop() => _cts.Cancel();
}

public class ScheduledTask
{
    public string Name { get; }
    public Func<CancellationToken, Task> Work { get; }
    public TimeSpan Interval { get; }
    public DateTime? LastRunAt { get; set; }
    public DateTime? LastSuccess { get; set; }
    public string? LastError { get; set; }
    public int RunCount { get; set; }
    public int ErrorCount { get; set; }
    
    public ScheduledTask(string name, Func<CancellationToken, Task> work, TimeSpan interval)
    {
        Name = name;
        Work = work;
        Interval = interval;
    }
}

// Usage:
// var scheduler = new AppTaskScheduler();
// scheduler.Schedule("sync", ct => syncService.SyncAsync(ct), TimeSpan.FromMinutes(15), true);
// scheduler.Schedule("cleanup", ct => cacheService.CleanOldCacheAsync(ct), TimeSpan.FromHours(24));
```

---

## Step 290: Background Service Registration

```csharp
// ============================================
// DI Registration for Background Services
// ============================================

public static class BackgroundServiceExtensions
{
    public static MauiAppBuilder ConfigureBackgroundServices(this MauiAppBuilder builder)
    {
        builder.Services.AddSingleton<IBackgroundSyncService, BackgroundSyncService>();
        builder.Services.AddSingleton<DownloadManager>();
        builder.Services.AddSingleton<UploadQueueService>();
        builder.Services.AddSingleton<ScheduledNotificationService>();
        builder.Services.AddSingleton<ResourceMonitor>();
        builder.Services.AddSingleton<AppTaskScheduler>();
        
        return builder;
    }
    
    // Start background services after app launches
    public static async Task StartBackgroundServicesAsync(this IServiceProvider services)
    {
        var scheduler = services.GetRequiredService<AppTaskScheduler>();
        var syncService = services.GetRequiredService<IBackgroundSyncService>();
        var resourceMonitor = services.GetRequiredService<ResourceMonitor>();
        
        // Start resource monitoring
        resourceMonitor.StartMonitoring(TimeSpan.FromMinutes(5));
        
        // Schedule periodic sync
        scheduler.Schedule("data-sync", 
            ct => syncService.SyncAsync(ct),
            TimeSpan.FromMinutes(15),
            runImmediately: true);
        
        // Process any pending uploads
        var uploadQueue = services.GetRequiredService<UploadQueueService>();
        await uploadQueue.ProcessQueueAsync();
    }
}

// In MauiProgram.cs:
// builder.ConfigureBackgroundServices();
// 
// In App.cs:
// protected override async void OnStart()
// {
//     await Handler.MauiContext!.Services.StartBackgroundServicesAsync();
// }
```

---

## สรุป Part 29

ใน Part 29 เราได้เรียนรู้:

1. **Background Processing Overview** - ประเภทต่างๆ
2. **Background Sync** - Periodic sync service
3. **App Lifecycle** - Integrate with window events
4. **WorkManager** - Android background jobs
5. **Download Manager** - Concurrent downloads with progress
6. **Upload Queue** - Persistent queue with retry
7. **Scheduled Notifications** - Daily/recurring reminders
8. **Resource Monitoring** - Battery, network state
9. **Task Scheduler** - In-app periodic tasks
10. **DI Registration** - Configure all background services

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 29 | Steps 281-290*
