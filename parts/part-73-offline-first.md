# Part 73: Offline-First Architecture
## Steps 721-730: SQLite Sync, Conflict Resolution, Background Sync, Delta Queries

---

## Step 721: Offline-First Data Layer

```csharp
// ============================================
// Offline-First Repository Pattern
// ============================================

public interface IOfflineRepository<T> where T : IEntity
{
    Task<T?> GetByIdAsync(int id);
    Task<IReadOnlyList<T>> GetAllAsync();
    Task SaveAsync(T entity);
    Task DeleteAsync(int id);
    Task<IReadOnlyList<T>> GetPendingSyncAsync();
    Task MarkSyncedAsync(int id, string serverVersion);
}

public enum SyncStatus { Synced, PendingCreate, PendingUpdate, PendingDelete }

public interface IEntity
{
    int Id { get; }
    string LocalId { get; } // GUID for offline-created items
    SyncStatus SyncStatus { get; set; }
    DateTime UpdatedAt { get; set; }
    string? ServerVersion { get; set; } // ETag or timestamp from server
}

// Base entity with sync metadata
public abstract class SyncableEntity : IEntity
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    [Indexed] public string LocalId { get; set; } = Guid.NewGuid().ToString();
    public SyncStatus SyncStatus { get; set; } = SyncStatus.PendingCreate;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public string? ServerVersion { get; set; }
}

// Cart item with offline support
public class CartItemEntity : SyncableEntity
{
    public string MenuItemId { get; set; } = "";
    public string MenuItemName { get; set; } = "";
    public decimal Price { get; set; }
    public int Quantity { get; set; }
    public string? Notes { get; set; }
}

// Implementation
public class OfflineCartRepository : IOfflineRepository<CartItemEntity>
{
    private readonly SQLiteAsyncConnection _db;
    
    public OfflineCartRepository(SQLiteAsyncConnection db)
    {
        _db = db;
    }
    
    public async Task<IReadOnlyList<CartItemEntity>> GetAllAsync()
    {
        var items = await _db.Table<CartItemEntity>()
            .Where(x => x.SyncStatus != SyncStatus.PendingDelete)
            .ToListAsync();
        return items.AsReadOnly();
    }
    
    public async Task SaveAsync(CartItemEntity entity)
    {
        entity.UpdatedAt = DateTime.UtcNow;
        
        if (entity.Id == 0)
        {
            entity.SyncStatus = SyncStatus.PendingCreate;
            await _db.InsertAsync(entity);
        }
        else
        {
            if (entity.SyncStatus == SyncStatus.Synced)
                entity.SyncStatus = SyncStatus.PendingUpdate;
            await _db.UpdateAsync(entity);
        }
    }
    
    public async Task DeleteAsync(int id)
    {
        var item = await _db.GetAsync<CartItemEntity>(id);
        if (item.SyncStatus == SyncStatus.PendingCreate)
        {
            await _db.DeleteAsync(item);
        }
        else
        {
            item.SyncStatus = SyncStatus.PendingDelete;
            item.UpdatedAt = DateTime.UtcNow;
            await _db.UpdateAsync(item);
        }
    }
    
    public async Task<IReadOnlyList<CartItemEntity>> GetPendingSyncAsync()
    {
        var items = await _db.Table<CartItemEntity>()
            .Where(x => x.SyncStatus != SyncStatus.Synced)
            .ToListAsync();
        return items.AsReadOnly();
    }
    
    public async Task MarkSyncedAsync(int id, string serverVersion)
    {
        var item = await _db.GetAsync<CartItemEntity>(id);
        item.SyncStatus = SyncStatus.Synced;
        item.ServerVersion = serverVersion;
        await _db.UpdateAsync(item);
    }
    
    public Task<CartItemEntity?> GetByIdAsync(int id)
        => _db.FindAsync<CartItemEntity>(id)!;
}
```

---

## Step 722: Sync Engine

```csharp
// ============================================
// Background Sync Engine
// ============================================

public class SyncEngine
{
    private readonly ICartApi _api;
    private readonly OfflineCartRepository _local;
    private readonly SyncConflictResolver _resolver;
    private readonly IConnectivity _connectivity;
    
    public event EventHandler<SyncResult>? SyncCompleted;
    
    public SyncEngine(ICartApi api, OfflineCartRepository local,
        SyncConflictResolver resolver, IConnectivity connectivity)
    {
        _api = api;
        _local = local;
        _resolver = resolver;
        _connectivity = connectivity;
        
        _connectivity.ConnectivityChanged += OnConnectivityChanged;
    }
    
    private void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
    {
        if (e.NetworkAccess == NetworkAccess.Internet)
            _ = SyncAsync();
    }
    
    public async Task<SyncResult> SyncAsync(CancellationToken ct = default)
    {
        if (_connectivity.NetworkAccess != NetworkAccess.Internet)
            return SyncResult.NoNetwork;
        
        var result = new SyncResult();
        
        try
        {
            // 1. Push local changes
            await PushChangesAsync(result, ct);
            
            // 2. Pull server changes
            await PullChangesAsync(result, ct);
            
            result.Success = true;
            result.SyncedAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            result.Error = ex.Message;
        }
        
        SyncCompleted?.Invoke(this, result);
        return result;
    }
    
    private async Task PushChangesAsync(SyncResult result, CancellationToken ct)
    {
        var pending = await _local.GetPendingSyncAsync();
        
        foreach (var item in pending)
        {
            ct.ThrowIfCancellationRequested();
            
            try
            {
                var dto = MapToDto(item);
                
                string serverVersion;
                switch (item.SyncStatus)
                {
                    case SyncStatus.PendingCreate:
                        var created = await _api.CreateAsync(dto, ct);
                        serverVersion = created.Version;
                        break;
                    
                    case SyncStatus.PendingUpdate:
                        var updated = await _api.UpdateAsync(item.LocalId, dto, item.ServerVersion!, ct);
                        serverVersion = updated.Version;
                        break;
                    
                    case SyncStatus.PendingDelete:
                        await _api.DeleteAsync(item.LocalId, ct);
                        await _local.DeleteAsync(item.Id); // hard delete
                        result.Deleted++;
                        continue;
                    
                    default: continue;
                }
                
                await _local.MarkSyncedAsync(item.Id, serverVersion);
                result.Pushed++;
            }
            catch (ConflictException)
            {
                await _resolver.ResolveAsync(item);
                result.Conflicts++;
            }
        }
    }
    
    private async Task PullChangesAsync(SyncResult result, CancellationToken ct)
    {
        var lastSync = Preferences.Get("last_sync", DateTime.MinValue.ToString());
        var since = DateTime.Parse(lastSync);
        
        var changes = await _api.GetChangesSinceAsync(since, ct);
        
        foreach (var change in changes)
        {
            var local = await FindLocalByServerIdAsync(change.Id);
            
            if (local == null)
            {
                // New from server
                await _local.SaveAsync(MapFromDto(change));
                result.Pulled++;
            }
            else if (local.SyncStatus == SyncStatus.Synced)
            {
                // Update local with server version
                UpdateFromDto(local, change);
                local.SyncStatus = SyncStatus.Synced;
                await _local.SaveAsync(local);
                result.Pulled++;
            }
            // else: conflict, handled in push phase
        }
        
        Preferences.Set("last_sync", DateTime.UtcNow.ToString());
    }
    
    private CartItemEntity MapFromDto(CartItemDto dto) => new()
    {
        LocalId = dto.Id,
        MenuItemId = dto.MenuItemId,
        MenuItemName = dto.MenuItemName,
        Price = dto.Price,
        Quantity = dto.Quantity,
        SyncStatus = SyncStatus.Synced,
        ServerVersion = dto.Version
    };
    
    private CartItemDto MapToDto(CartItemEntity e) => new()
    {
        Id = e.LocalId,
        MenuItemId = e.MenuItemId,
        MenuItemName = e.MenuItemName,
        Price = e.Price,
        Quantity = e.Quantity
    };
    
    private void UpdateFromDto(CartItemEntity local, CartItemDto dto)
    {
        local.Quantity = dto.Quantity;
        local.Notes = dto.Notes;
        local.ServerVersion = dto.Version;
        local.UpdatedAt = DateTime.UtcNow;
    }
    
    private Task<CartItemEntity?> FindLocalByServerIdAsync(string serverId)
        => _local.GetByIdAsync(0); // simplified
}

public class SyncResult
{
    public bool Success { get; set; }
    public int Pushed { get; set; }
    public int Pulled { get; set; }
    public int Deleted { get; set; }
    public int Conflicts { get; set; }
    public string? Error { get; set; }
    public DateTime SyncedAt { get; set; }
    
    public static readonly SyncResult NoNetwork = new() { Error = "No network" };
}
```

---

## Step 723: Conflict Resolution

```csharp
// ============================================
// Conflict Resolution Strategies
// ============================================

public enum ConflictStrategy
{
    ServerWins,       // Server version always wins
    ClientWins,       // Local version always wins
    LastWriteWins,    // Most recent UpdatedAt wins
    UserDecides,      // Present both versions to user
    Merge             // Attempt field-level merge
}

public class SyncConflictResolver
{
    private readonly ConflictStrategy _strategy;
    private readonly IOfflineRepository<CartItemEntity> _repository;
    
    public SyncConflictResolver(ConflictStrategy strategy,
        IOfflineRepository<CartItemEntity> repository)
    {
        _strategy = strategy;
        _repository = repository;
    }
    
    public async Task ResolveAsync(CartItemEntity local)
    {
        switch (_strategy)
        {
            case ConflictStrategy.ServerWins:
                await ResolveServerWinsAsync(local);
                break;
            
            case ConflictStrategy.ClientWins:
                await ResolveClientWinsAsync(local);
                break;
            
            case ConflictStrategy.LastWriteWins:
                await ResolveLastWriteWinsAsync(local);
                break;
            
            case ConflictStrategy.UserDecides:
                await ResolveUserDecidesAsync(local);
                break;
        }
    }
    
    private async Task ResolveServerWinsAsync(CartItemEntity local)
    {
        // Fetch server version and overwrite local
        // local.SyncStatus = SyncStatus.Synced;
        await _repository.SaveAsync(local);
    }
    
    private async Task ResolveClientWinsAsync(CartItemEntity local)
    {
        // Force push by removing server version (bypass optimistic concurrency)
        local.ServerVersion = null;
        await _repository.SaveAsync(local);
    }
    
    private async Task ResolveLastWriteWinsAsync(CartItemEntity local)
    {
        // Compare timestamps and pick winner
        // This requires fetching the server's UpdatedAt
        await Task.CompletedTask;
    }
    
    private async Task ResolveUserDecidesAsync(CartItemEntity local)
    {
        WeakReferenceMessenger.Default.Send(new ConflictDetectedMessage(local));
        await Task.CompletedTask;
    }
}

public record ConflictDetectedMessage(CartItemEntity LocalVersion);
```

---

## Step 724: Delta Sync Protocol

```csharp
// ============================================
// Delta Sync (Only Changed Records)
// ============================================

public class DeltaSyncService
{
    private readonly HttpClient _http;
    private const string LastSyncKey = "delta_sync_cursor";
    
    public DeltaSyncService(HttpClient http)
    {
        _http = http;
    }
    
    public async Task<DeltaPage<T>> FetchDeltaAsync<T>(
        string endpoint, CancellationToken ct = default)
    {
        var cursor = Preferences.Get(LastSyncKey, string.Empty);
        
        var url = string.IsNullOrEmpty(cursor)
            ? endpoint
            : $"{endpoint}?cursor={Uri.EscapeDataString(cursor)}";
        
        var response = await _http.GetFromJsonAsync<DeltaPage<T>>(url, ct)
            ?? throw new InvalidOperationException("Delta sync failed");
        
        if (!string.IsNullOrEmpty(response.NextCursor))
            Preferences.Set(LastSyncKey, response.NextCursor);
        
        return response;
    }
    
    public void ResetCursor()
        => Preferences.Remove(LastSyncKey);
}

public class DeltaPage<T>
{
    public List<T> Created { get; init; } = new();
    public List<T> Updated { get; init; } = new();
    public List<string> DeletedIds { get; init; } = new();
    public string? NextCursor { get; init; }
    public bool HasMore { get; init; }
}

// Apply delta to local database
public class DeltaApplier<T> where T : SyncableEntity, new()
{
    private readonly IOfflineRepository<T> _repo;
    
    public DeltaApplier(IOfflineRepository<T> repo)
    {
        _repo = repo;
    }
    
    public async Task ApplyAsync(DeltaPage<T> delta)
    {
        foreach (var item in delta.Created)
        {
            item.SyncStatus = SyncStatus.Synced;
            await _repo.SaveAsync(item);
        }
        
        foreach (var item in delta.Updated)
        {
            var local = await _repo.GetByIdAsync(item.Id);
            if (local?.SyncStatus == SyncStatus.Synced)
            {
                item.SyncStatus = SyncStatus.Synced;
                await _repo.SaveAsync(item);
            }
            // If local has pending changes: leave for conflict resolver
        }
        
        foreach (var deletedId in delta.DeletedIds)
        {
            // Map server ID to local ID and hard-delete
        }
    }
}
```

---

## Step 725: Optimistic UI Updates

```csharp
// ============================================
// Optimistic UI (Update UI Before Server Confirms)
// ============================================

public partial class OptimisticCartViewModel : ObservableObject
{
    private readonly IOfflineRepository<CartItemEntity> _repo;
    private readonly SyncEngine _sync;
    
    [ObservableProperty] private ObservableCollection<CartItemEntity> _items = new();
    [ObservableProperty] private decimal _total;
    [ObservableProperty] private bool _syncPending;
    
    public OptimisticCartViewModel(IOfflineRepository<CartItemEntity> repo, SyncEngine sync)
    {
        _repo = repo;
        _sync = sync;
        _sync.SyncCompleted += OnSyncCompleted;
    }
    
    [RelayCommand]
    private async Task AddItemAsync(CartItemEntity item)
    {
        // 1. Update UI immediately (optimistic)
        Items.Add(item);
        RecalculateTotal();
        SyncPending = true;
        
        // 2. Persist locally
        await _repo.SaveAsync(item);
        
        // 3. Try to sync in background (fire and forget)
        _ = _sync.SyncAsync();
    }
    
    [RelayCommand]
    private async Task RemoveItemAsync(CartItemEntity item)
    {
        // Optimistic: remove from UI
        Items.Remove(item);
        RecalculateTotal();
        SyncPending = true;
        
        // Queue for deletion
        await _repo.DeleteAsync(item.Id);
        _ = _sync.SyncAsync();
    }
    
    [RelayCommand]
    private async Task UpdateQuantityAsync((CartItemEntity item, int qty) args)
    {
        var (item, qty) = args;
        
        // Optimistic update
        var idx = Items.IndexOf(item);
        if (idx >= 0)
        {
            Items[idx].Quantity = qty;
            RecalculateTotal();
        }
        
        item.Quantity = qty;
        await _repo.SaveAsync(item);
        _ = _sync.SyncAsync();
    }
    
    private void OnSyncCompleted(object? sender, SyncResult result)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            SyncPending = result.Pushed > 0 || !result.Success;
            if (result.Conflicts > 0)
                ReloadFromLocalAsync().ConfigureAwait(false);
        });
    }
    
    private async Task ReloadFromLocalAsync()
    {
        var items = await _repo.GetAllAsync();
        Items = new ObservableCollection<CartItemEntity>(items);
        RecalculateTotal();
    }
    
    private void RecalculateTotal()
        => Total = Items.Sum(x => x.Price * x.Quantity);
}
```

---

## Step 726: Background Sync Scheduling

```csharp
// ============================================
// Periodic Background Sync
// ============================================

public class BackgroundSyncService
{
    private readonly SyncEngine _syncEngine;
    private CancellationTokenSource? _cts;
    private Task? _syncTask;
    
    public BackgroundSyncService(SyncEngine syncEngine)
    {
        _syncEngine = syncEngine;
    }
    
    public void Start(TimeSpan interval)
    {
        if (_syncTask != null) return;
        
        _cts = new CancellationTokenSource();
        _syncTask = SyncLoopAsync(interval, _cts.Token);
    }
    
    public async Task StopAsync()
    {
        _cts?.Cancel();
        if (_syncTask != null)
        {
            try { await _syncTask; }
            catch (OperationCanceledException) { }
        }
        _syncTask = null;
    }
    
    private async Task SyncLoopAsync(TimeSpan interval, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await _syncEngine.SyncAsync(ct);
                await Task.Delay(interval, ct);
            }
            catch (OperationCanceledException) { break; }
            catch
            {
                // Back off on error
                await Task.Delay(TimeSpan.FromMinutes(1), ct);
            }
        }
    }
}

// Register in MauiProgram.cs
// builder.Services.AddSingleton<BackgroundSyncService>();
// In App.xaml.cs OnStart:
// _syncService.Start(TimeSpan.FromMinutes(5));
```

---

## Step 727: Offline Queue for API Calls

```csharp
// ============================================
// Offline Queue – Defer API Calls When Offline
// ============================================

public class OfflineQueue
{
    private readonly SQLiteAsyncConnection _db;
    private readonly IConnectivity _connectivity;
    
    public OfflineQueue(SQLiteAsyncConnection db, IConnectivity connectivity)
    {
        _db = db;
        _connectivity = connectivity;
        _db.CreateTableAsync<QueuedAction>().Wait();
        
        _connectivity.ConnectivityChanged += async (_, e) =>
        {
            if (e.NetworkAccess == NetworkAccess.Internet)
                await FlushAsync();
        };
    }
    
    public async Task EnqueueAsync(string actionType, string payload)
    {
        await _db.InsertAsync(new QueuedAction
        {
            ActionType = actionType,
            Payload = payload,
            CreatedAt = DateTime.UtcNow,
            RetryCount = 0
        });
    }
    
    public async Task<bool> ExecuteOrQueueAsync(
        string actionType, string payload,
        Func<string, Task> executeFunc)
    {
        if (_connectivity.NetworkAccess == NetworkAccess.Internet)
        {
            try
            {
                await executeFunc(payload);
                return true;
            }
            catch
            {
                await EnqueueAsync(actionType, payload);
                return false;
            }
        }
        
        await EnqueueAsync(actionType, payload);
        return false;
    }
    
    public async Task FlushAsync()
    {
        var pending = await _db.Table<QueuedAction>()
            .Where(x => x.RetryCount < 5)
            .OrderBy(x => x.CreatedAt)
            .ToListAsync();
        
        foreach (var action in pending)
        {
            try
            {
                await ExecuteActionAsync(action);
                await _db.DeleteAsync(action);
            }
            catch
            {
                action.RetryCount++;
                action.LastAttempt = DateTime.UtcNow;
                await _db.UpdateAsync(action);
            }
        }
    }
    
    private Task ExecuteActionAsync(QueuedAction action)
    {
        // Dispatch to registered handlers
        return action.ActionType switch
        {
            "PlaceOrder" => ExecutePlaceOrderAsync(action.Payload),
            "RateRestaurant" => ExecuteRatingAsync(action.Payload),
            _ => Task.CompletedTask
        };
    }
    
    private Task ExecutePlaceOrderAsync(string payload) => Task.CompletedTask;
    private Task ExecuteRatingAsync(string payload) => Task.CompletedTask;
}

[Table("queued_actions")]
public class QueuedAction
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string ActionType { get; set; } = "";
    public string Payload { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public int RetryCount { get; set; }
    public DateTime? LastAttempt { get; set; }
}
```

---

## Step 728: Network-Aware Caching

```csharp
// ============================================
// Network-Aware Repository with Cache-First Strategy
// ============================================

public class NetworkAwareMenuRepository
{
    private readonly IMenuApi _api;
    private readonly SQLiteAsyncConnection _db;
    private readonly IConnectivity _connectivity;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromHours(1);
    
    public NetworkAwareMenuRepository(IMenuApi api, SQLiteAsyncConnection db,
        IConnectivity connectivity)
    {
        _api = api;
        _db = db;
        _connectivity = connectivity;
    }
    
    public async Task<IReadOnlyList<MenuItem>> GetMenuAsync(string restaurantId)
    {
        // Always try cache first
        var cached = await GetFromCacheAsync(restaurantId);
        
        if (_connectivity.NetworkAccess != NetworkAccess.Internet)
        {
            // Offline: return cache (possibly stale)
            return cached ?? new List<MenuItem>().AsReadOnly();
        }
        
        // Online: check if cache is fresh
        if (cached != null && await IsCacheFreshAsync(restaurantId))
            return cached;
        
        // Fetch from network and update cache
        var fresh = await _api.GetMenuAsync(restaurantId);
        await SaveToCacheAsync(restaurantId, fresh);
        return fresh;
    }
    
    private async Task<IReadOnlyList<MenuItem>?> GetFromCacheAsync(string restaurantId)
    {
        var rows = await _db.Table<MenuItemEntity>()
            .Where(x => x.RestaurantId == restaurantId)
            .ToListAsync();
        return rows.Count == 0 ? null
            : rows.Select(r => new MenuItem(r.Id, r.Name, r.Price, r.Description)).ToList().AsReadOnly();
    }
    
    private async Task<bool> IsCacheFreshAsync(string restaurantId)
    {
        var meta = await _db.FindAsync<CacheMeta>(restaurantId);
        return meta != null && (DateTime.UtcNow - meta.CachedAt) < CacheDuration;
    }
    
    private async Task SaveToCacheAsync(string restaurantId, IReadOnlyList<MenuItem> items)
    {
        await _db.RunInTransactionAsync(conn =>
        {
            conn.Execute("DELETE FROM MenuItemEntity WHERE RestaurantId = ?", restaurantId);
            foreach (var item in items)
                conn.Insert(new MenuItemEntity
                {
                    RestaurantId = restaurantId,
                    Id = item.Id, Name = item.Name,
                    Price = item.Price, Description = item.Description
                });
        });
        
        await _db.InsertOrReplaceAsync(new CacheMeta
        {
            Key = restaurantId, CachedAt = DateTime.UtcNow
        });
    }
}

[Table("cache_meta")]
public class CacheMeta
{
    [PrimaryKey] public string Key { get; set; } = "";
    public DateTime CachedAt { get; set; }
}
```

---

## Step 729: Offline Indicator UI

```xml
<!-- Offline Banner -->
<ContentPage x:Class="MyApp.OfflineAwarePage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">
    
    <Grid RowDefinitions="Auto,*">
        
        <!-- Offline Banner -->
        <Frame Grid.Row="0"
               BackgroundColor="#FF9500"
               Padding="8,4"
               IsVisible="{Binding IsOffline}"
               x:Name="OfflineBanner">
            <Label Text="คุณอยู่ในโหมดออฟไลน์ – ข้อมูลอาจไม่ล่าสุด"
                   TextColor="White" FontSize="13"
                   HorizontalOptions="Center" />
        </Frame>
        
        <!-- Sync Pending Indicator -->
        <Frame Grid.Row="0"
               BackgroundColor="#34C759"
               Padding="8,4"
               IsVisible="{Binding SyncInProgress}">
            <HorizontalStackLayout HorizontalOptions="Center" Spacing="6">
                <ActivityIndicator IsRunning="True" Color="White" Scale="0.7" />
                <Label Text="กำลังซิงค์..." TextColor="White" FontSize="13" />
            </HorizontalStackLayout>
        </Frame>
        
        <!-- Main Content -->
        <ScrollView Grid.Row="1">
            <!-- content -->
        </ScrollView>
        
    </Grid>
</ContentPage>
```

```csharp
public partial class OfflineAwareViewModel : ObservableObject
{
    [ObservableProperty] private bool _isOffline;
    [ObservableProperty] private bool _syncInProgress;
    [ObservableProperty] private DateTime? _lastSyncedAt;
    
    public OfflineAwareViewModel(IConnectivity connectivity, SyncEngine sync)
    {
        IsOffline = connectivity.NetworkAccess != NetworkAccess.Internet;
        connectivity.ConnectivityChanged += (_, e) =>
        {
            IsOffline = e.NetworkAccess != NetworkAccess.Internet;
        };
        
        sync.SyncCompleted += (_, result) =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                SyncInProgress = false;
                if (result.Success) LastSyncedAt = result.SyncedAt;
            });
        };
    }
    
    public string LastSyncText => LastSyncedAt.HasValue
        ? $"ซิงค์ล่าสุด: {ThaiFormatter.RelativeTime(LastSyncedAt.Value)}"
        : "ยังไม่ได้ซิงค์";
}
```

---

## Step 730: Offline Tests

```csharp
// ============================================
// Offline Tests
// ============================================

[TestFixture]
public class OfflineSyncTests
{
    private SQLiteAsyncConnection _db = null!;
    private OfflineCartRepository _repo = null!;
    
    [SetUp]
    public async Task SetUp()
    {
        _db = new SQLiteAsyncConnection(":memory:");
        await _db.CreateTableAsync<CartItemEntity>();
        _repo = new OfflineCartRepository(_db);
    }
    
    [Test]
    public async Task AddItem_WhenOffline_SavesAsPendingCreate()
    {
        var item = new CartItemEntity { MenuItemName = "ข้าวผัด", Price = 80, Quantity = 1 };
        
        await _repo.SaveAsync(item);
        
        var pending = await _repo.GetPendingSyncAsync();
        Assert.That(pending.Count, Is.EqualTo(1));
        Assert.That(pending[0].SyncStatus, Is.EqualTo(SyncStatus.PendingCreate));
    }
    
    [Test]
    public async Task DeleteUnsyncedItem_HardDeletes()
    {
        var item = new CartItemEntity { MenuItemName = "ผัดไทย", Price = 90, Quantity = 1 };
        await _repo.SaveAsync(item);
        
        await _repo.DeleteAsync(item.Id);
        
        var all = await _repo.GetAllAsync();
        Assert.That(all, Is.Empty);
    }
    
    [Test]
    public async Task DeleteSyncedItem_MarksAsPendingDelete()
    {
        var item = new CartItemEntity
        {
            MenuItemName = "ต้มยำ", Price = 120, Quantity = 1,
            SyncStatus = SyncStatus.Synced, ServerVersion = "v1"
        };
        await _db.InsertAsync(item);
        
        await _repo.DeleteAsync(item.Id);
        
        var pending = await _repo.GetPendingSyncAsync();
        Assert.That(pending.Any(p => p.SyncStatus == SyncStatus.PendingDelete), Is.True);
    }
    
    [Test]
    public async Task MarkSynced_UpdatesStatus()
    {
        var item = new CartItemEntity { MenuItemName = "ข้าวสวย", Price = 20, Quantity = 2 };
        await _repo.SaveAsync(item);
        
        await _repo.MarkSyncedAsync(item.Id, "server-v1");
        
        var result = await _repo.GetByIdAsync(item.Id);
        Assert.That(result!.SyncStatus, Is.EqualTo(SyncStatus.Synced));
        Assert.That(result.ServerVersion, Is.EqualTo("server-v1"));
    }
    
    [Test]
    public async Task GetAll_ExcludesPendingDeletes()
    {
        await _db.InsertAsync(new CartItemEntity
        {
            MenuItemName = "ลบแล้ว", Price = 50, Quantity = 1,
            SyncStatus = SyncStatus.PendingDelete
        });
        await _db.InsertAsync(new CartItemEntity
        {
            MenuItemName = "ยังอยู่", Price = 70, Quantity = 1,
            SyncStatus = SyncStatus.PendingCreate
        });
        
        var items = await _repo.GetAllAsync();
        Assert.That(items.Count, Is.EqualTo(1));
        Assert.That(items[0].MenuItemName, Is.EqualTo("ยังอยู่"));
    }
}
```

---

## สรุป Part 73

ใน Part 73 เราได้เรียนรู้:

1. **Offline Repository** - SyncStatus enum, IEntity, SyncableEntity base class
2. **Sync Engine** - Push + Pull phases, ConnectivityChanged trigger
3. **Conflict Resolution** - 4 strategies: ServerWins, ClientWins, LastWriteWins, UserDecides
4. **Delta Sync** - Cursor-based incremental sync, DeltaPage with Created/Updated/DeletedIds
5. **Optimistic UI** - Update UI first, sync in background, reload on conflict
6. **Background Sync** - Periodic loop, exponential backoff on error
7. **Offline Queue** - Queue API calls when offline, flush on reconnect
8. **Network-Aware Cache** - Cache-first with staleness check, offline fallback
9. **Offline Indicator UI** - Banner, sync progress, last sync time
10. **Offline Tests** - PendingCreate, hard delete, PendingDelete, sync marking

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 73 | Steps 721-730*
