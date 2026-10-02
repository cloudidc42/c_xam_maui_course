# Part 44: Offline-First & Sync Advanced
## Steps 431-440: Conflict Resolution, Delta Sync, CRDT

---

## Step 431: Offline-First Architecture

```csharp
// ============================================
// Advanced Offline-First Architecture
// ============================================

/*
 * ออกแบบระบบ Offline-First ที่สมบูรณ์:
 * 
 * 1. Local Database (SQLite) เป็น source of truth
 * 2. Queue การเปลี่ยนแปลงทั้งหมด
 * 3. Sync engine: push changes + pull updates
 * 4. Conflict resolution strategy
 * 5. Vector clock / logical timestamps
 * 6. CRDT (Conflict-free Replicated Data Types)
 */

public class ChangeRecord
{
    [PrimaryKey, AutoIncrement] public long Id { get; set; }
    public string EntityType { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string Operation { get; set; } = string.Empty; // Insert/Update/Delete
    public string Payload { get; set; } = string.Empty; // JSON
    public long VectorClock { get; set; }
    public string DeviceId { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public string Status { get; set; } = "Pending"; // Pending/Synced/Failed/Conflicted
}

public class SyncState
{
    [PrimaryKey] public string EntityType { get; set; } = string.Empty;
    public DateTime LastSyncedAt { get; set; }
    public long LastServerCursor { get; set; }
    public long LocalVectorClock { get; set; }
}
```

---

## Step 432: Vector Clock

```csharp
// ============================================
// Vector Clock for Causality Tracking
// ============================================

public class VectorClock
{
    private readonly Dictionary<string, long> _clock;
    
    public VectorClock()
    {
        _clock = new Dictionary<string, long>();
    }
    
    private VectorClock(Dictionary<string, long> clock)
    {
        _clock = new Dictionary<string, long>(clock);
    }
    
    public long Get(string nodeId)
        => _clock.GetValueOrDefault(nodeId, 0);
    
    public VectorClock Increment(string nodeId)
    {
        var next = new Dictionary<string, long>(_clock)
        {
            [nodeId] = Get(nodeId) + 1
        };
        return new VectorClock(next);
    }
    
    public VectorClock Merge(VectorClock other)
    {
        var merged = new Dictionary<string, long>(_clock);
        
        foreach (var (nodeId, time) in other._clock)
        {
            merged[nodeId] = Math.Max(
                merged.GetValueOrDefault(nodeId, 0), time);
        }
        
        return new VectorClock(merged);
    }
    
    // Happens-before relationship
    public bool HappensBefore(VectorClock other)
    {
        var allNodes = _clock.Keys.Concat(other._clock.Keys).Distinct();
        
        bool strictlyLess = false;
        foreach (var node in allNodes)
        {
            var myTime = Get(node);
            var otherTime = other.Get(node);
            
            if (myTime > otherTime) return false;
            if (myTime < otherTime) strictlyLess = true;
        }
        
        return strictlyLess;
    }
    
    // Concurrent (neither happens-before the other)
    public bool IsConcurrentWith(VectorClock other)
        => !HappensBefore(other) && !other.HappensBefore(this);
    
    public string Serialize()
        => System.Text.Json.JsonSerializer.Serialize(_clock);
    
    public static VectorClock Deserialize(string json)
    {
        var dict = System.Text.Json.JsonSerializer.Deserialize<Dictionary<string, long>>(json) ?? new();
        return new VectorClock(dict);
    }
}
```

---

## Step 433: Delta Sync Protocol

```csharp
// ============================================
// Delta Sync - Only sync what changed
// ============================================

public class DeltaSyncEngine
{
    private readonly SQLiteConnection _db;
    private readonly ISyncApi _api;
    private readonly string _deviceId;
    
    public DeltaSyncEngine(SQLiteConnection db, ISyncApi api)
    {
        _db = db;
        _api = api;
        _deviceId = GetOrCreateDeviceId();
    }
    
    public async Task<SyncReport> SyncAsync(
        string entityType, CancellationToken ct = default)
    {
        var report = new SyncReport { EntityType = entityType };
        
        // Step 1: Push local changes
        var pushResult = await PushChangesAsync(entityType, ct);
        report.Pushed = pushResult.Count;
        report.PushErrors = pushResult.Errors;
        
        // Step 2: Pull server changes since last cursor
        var syncState = GetSyncState(entityType);
        var pullResult = await PullChangesAsync(entityType, syncState.LastServerCursor, ct);
        
        // Step 3: Apply pulled changes (with conflict detection)
        foreach (var change in pullResult.Changes)
        {
            await ApplyServerChangeAsync(change, ct);
            report.Pulled++;
        }
        
        // Step 4: Update sync state
        UpdateSyncState(entityType, pullResult.NextCursor);
        
        return report;
    }
    
    private async Task<PushResult> PushChangesAsync(
        string entityType, CancellationToken ct)
    {
        var pending = _db.Table<ChangeRecord>()
            .Where(c => c.EntityType == entityType &&
                       c.Status == "Pending")
            .OrderBy(c => c.Id)
            .Take(100)
            .ToList();
        
        if (pending.Count == 0)
            return new PushResult([], 0, []);
        
        try
        {
            var response = await _api.PushChangesAsync(
                new PushRequest(pending.Select(MapToChangeDto).ToList()), ct);
            
            foreach (var change in pending)
            {
                var serverResult = response.Results
                    .FirstOrDefault(r => r.LocalId == change.Id.ToString());
                
                if (serverResult?.Success == true)
                {
                    change.Status = "Synced";
                }
                else if (serverResult?.IsConflict == true)
                {
                    change.Status = "Conflicted";
                }
                else
                {
                    change.Status = "Failed";
                }
                
                _db.Update(change);
            }
            
            return new PushResult(
                pending,
                pending.Count(c => c.Status == "Synced"),
                pending.Where(c => c.Status == "Failed").ToList());
        }
        catch (Exception ex)
        {
            return new PushResult(pending, 0, pending, ex.Message);
        }
    }
    
    private async Task<PullResult> PullChangesAsync(
        string entityType, long cursor, CancellationToken ct)
    {
        return await _api.PullChangesAsync(
            new PullRequest(entityType, cursor, _deviceId, pageSize: 100), ct);
    }
    
    private async Task ApplyServerChangeAsync(ChangeDto change, CancellationToken ct)
    {
        // Check for conflict
        var localChange = _db.Table<ChangeRecord>()
            .Where(c => c.EntityId == change.EntityId &&
                       c.EntityType == change.EntityType &&
                       c.Status == "Pending")
            .FirstOrDefault();
        
        if (localChange != null)
        {
            // Conflict: server and local both modified the same entity
            await ResolveConflictAsync(localChange, change, ct);
        }
        else
        {
            // No conflict: apply server change directly
            ApplyChange(change);
        }
    }
    
    private async Task ResolveConflictAsync(
        ChangeRecord local, ChangeDto server, CancellationToken ct)
    {
        // Last-write-wins strategy
        if (server.Timestamp > local.CreatedAt)
        {
            // Server wins: discard local, apply server
            _db.Delete(local);
            ApplyChange(server);
        }
        else
        {
            // Client wins: mark conflict resolved, keep local
            local.Status = "Synced";
            _db.Update(local);
        }
    }
    
    private void ApplyChange(ChangeDto change)
    {
        switch (change.Operation)
        {
            case "Insert":
            case "Update":
                _db.Execute(
                    $"INSERT OR REPLACE INTO {change.EntityType}s SELECT * FROM json({change.Payload})");
                break;
            case "Delete":
                _db.Execute(
                    $"DELETE FROM {change.EntityType}s WHERE Id = ?", change.EntityId);
                break;
        }
    }
    
    private SyncState GetSyncState(string entityType)
    {
        return _db.Table<SyncState>()
            .FirstOrDefault(s => s.EntityType == entityType)
            ?? new SyncState { EntityType = entityType };
    }
    
    private void UpdateSyncState(string entityType, long cursor)
    {
        _db.InsertOrReplace(new SyncState
        {
            EntityType = entityType,
            LastSyncedAt = DateTime.UtcNow,
            LastServerCursor = cursor
        });
    }
    
    private static string GetOrCreateDeviceId()
    {
        const string key = "device_sync_id";
        var existing = Preferences.Default.Get(key, string.Empty);
        
        if (!string.IsNullOrEmpty(existing)) return existing;
        
        var newId = Guid.NewGuid().ToString("N")[..12];
        Preferences.Default.Set(key, newId);
        return newId;
    }
    
    private ChangeDto MapToChangeDto(ChangeRecord record) => new(
        LocalId: record.Id.ToString(),
        EntityType: record.EntityType,
        EntityId: record.EntityId,
        Operation: record.Operation,
        Payload: record.Payload,
        Timestamp: record.CreatedAt,
        DeviceId: _deviceId);
}
```

---

## Step 434: CRDT Counter

```csharp
// ============================================
// CRDT - G-Counter (Grow-only Counter)
// ============================================

public class GCounter
{
    private readonly Dictionary<string, long> _counts;
    
    public GCounter()
    {
        _counts = new Dictionary<string, long>();
    }
    
    private GCounter(Dictionary<string, long> counts)
    {
        _counts = new Dictionary<string, long>(counts);
    }
    
    public long Value => _counts.Values.Sum();
    
    public GCounter Increment(string nodeId, long by = 1)
    {
        var next = new Dictionary<string, long>(_counts)
        {
            [nodeId] = _counts.GetValueOrDefault(nodeId, 0) + by
        };
        return new GCounter(next);
    }
    
    // Merge two counters (take max per node)
    public GCounter Merge(GCounter other)
    {
        var merged = new Dictionary<string, long>(_counts);
        
        foreach (var (nodeId, count) in other._counts)
        {
            merged[nodeId] = Math.Max(
                merged.GetValueOrDefault(nodeId, 0), count);
        }
        
        return new GCounter(merged);
    }
    
    public string Serialize()
        => System.Text.Json.JsonSerializer.Serialize(_counts);
    
    public static GCounter Deserialize(string json)
    {
        var dict = System.Text.Json.JsonSerializer
            .Deserialize<Dictionary<string, long>>(json) ?? new();
        return new GCounter(dict);
    }
}

// PN-Counter (can increment and decrement)
public class PNCounter
{
    private readonly GCounter _positive;
    private readonly GCounter _negative;
    
    public PNCounter()
    {
        _positive = new GCounter();
        _negative = new GCounter();
    }
    
    private PNCounter(GCounter pos, GCounter neg)
    {
        _positive = pos;
        _negative = neg;
    }
    
    public long Value => _positive.Value - _negative.Value;
    
    public PNCounter Increment(string nodeId, long by = 1)
        => new(_positive.Increment(nodeId, by), _negative);
    
    public PNCounter Decrement(string nodeId, long by = 1)
        => new(_positive, _negative.Increment(nodeId, by));
    
    public PNCounter Merge(PNCounter other)
        => new(_positive.Merge(other._positive), _negative.Merge(other._negative));
}

// Use for: likes, views, cart quantities that can be modified offline
public class CartQuantitySync
{
    private readonly SQLiteConnection _db;
    private readonly string _deviceId;
    
    public CartQuantitySync(SQLiteConnection db, string deviceId)
    {
        _db = db;
        _deviceId = deviceId;
    }
    
    public void AddToCart(string productId, int qty)
    {
        var entry = GetOrCreate(productId);
        var counter = PNCounter.Deserialize(entry.CounterData);
        var updated = counter.Increment(_deviceId, qty);
        
        entry.CounterData = updated.Serialize();
        entry.DisplayQuantity = (int)updated.Value;
        _db.InsertOrReplace(entry);
    }
    
    public void RemoveFromCart(string productId, int qty)
    {
        var entry = GetOrCreate(productId);
        var counter = PNCounter.Deserialize(entry.CounterData);
        var updated = counter.Decrement(_deviceId, qty);
        
        if (updated.Value <= 0)
        {
            _db.Delete(entry);
        }
        else
        {
            entry.CounterData = updated.Serialize();
            entry.DisplayQuantity = (int)updated.Value;
            _db.InsertOrReplace(entry);
        }
    }
    
    private CartQuantityEntry GetOrCreate(string productId)
    {
        return _db.Table<CartQuantityEntry>()
            .FirstOrDefault(e => e.ProductId == productId)
            ?? new CartQuantityEntry
            {
                ProductId = productId,
                CounterData = new PNCounter().Serialize()
            };
    }
}
```

---

## Step 435: Last-Write-Wins Register

```csharp
// ============================================
// CRDT LWW-Register
// ============================================

public class LwwRegister<T>
{
    public T? Value { get; private set; }
    public DateTime Timestamp { get; private set; }
    public string NodeId { get; private set; } = string.Empty;
    
    public LwwRegister<T> Assign(T value, string nodeId)
    {
        return new LwwRegister<T>
        {
            Value = value,
            Timestamp = DateTime.UtcNow,
            NodeId = nodeId
        };
    }
    
    public LwwRegister<T> Merge(LwwRegister<T> other)
    {
        if (other.Timestamp > Timestamp ||
            (other.Timestamp == Timestamp && other.NodeId > NodeId))
        {
            return other;
        }
        return this;
    }
}

// Observed-Remove Set (OR-Set) - supports add and remove
public class OrSet<T> where T : notnull
{
    // Each element has a set of unique tags
    private readonly Dictionary<T, HashSet<string>> _elements = new();
    private readonly Dictionary<T, HashSet<string>> _tombstones = new();
    
    public void Add(T element, string tag)
    {
        if (!_elements.ContainsKey(element))
            _elements[element] = new HashSet<string>();
        _elements[element].Add(tag);
    }
    
    public void Remove(T element)
    {
        if (_elements.TryGetValue(element, out var tags))
        {
            if (!_tombstones.ContainsKey(element))
                _tombstones[element] = new HashSet<string>();
            
            foreach (var tag in tags)
                _tombstones[element].Add(tag);
        }
    }
    
    public bool Contains(T element)
    {
        if (!_elements.TryGetValue(element, out var tags)) return false;
        
        var tombstoned = _tombstones.TryGetValue(element, out var ts) ? ts : new HashSet<string>();
        return tags.Except(tombstoned).Any();
    }
    
    public IEnumerable<T> GetAll()
        => _elements.Keys.Where(Contains);
    
    public OrSet<T> Merge(OrSet<T> other)
    {
        var merged = new OrSet<T>();
        
        // Merge elements
        var allKeys = _elements.Keys.Concat(other._elements.Keys).Distinct();
        foreach (var key in allKeys)
        {
            var tags1 = _elements.TryGetValue(key, out var t1) ? t1 : new HashSet<string>();
            var tags2 = other._elements.TryGetValue(key, out var t2) ? t2 : new HashSet<string>();
            merged._elements[key] = new HashSet<string>(tags1.Union(tags2));
        }
        
        // Merge tombstones
        var tombKeys = _tombstones.Keys.Concat(other._tombstones.Keys).Distinct();
        foreach (var key in tombKeys)
        {
            var t1 = _tombstones.TryGetValue(key, out var ts1) ? ts1 : new HashSet<string>();
            var t2 = other._tombstones.TryGetValue(key, out var ts2) ? ts2 : new HashSet<string>();
            merged._tombstones[key] = new HashSet<string>(t1.Union(t2));
        }
        
        return merged;
    }
}
```

---

## Step 436: Sync Queue UI

```csharp
// ============================================
// Sync Status & Queue UI
// ============================================

public partial class SyncStatusViewModel : ObservableObject
{
    private readonly DeltaSyncEngine _syncEngine;
    private readonly SQLiteConnection _db;
    
    [ObservableProperty] private int _pendingChanges;
    [ObservableProperty] private int _failedChanges;
    [ObservableProperty] private bool _isSyncing;
    [ObservableProperty] private DateTime? _lastSyncedAt;
    [ObservableProperty] private string _syncStatusText = "ไม่เคย sync";
    [ObservableProperty] private bool _isOnline;
    
    public SyncStatusViewModel(DeltaSyncEngine syncEngine, SQLiteConnection db)
    {
        _syncEngine = syncEngine;
        _db = db;
        
        // Monitor connectivity
        Connectivity.ConnectivityChanged += (s, e) =>
        {
            IsOnline = e.NetworkAccess == NetworkAccess.Internet;
            if (IsOnline) _ = SyncCommand.ExecuteAsync(null);
        };
        
        IsOnline = Connectivity.Current.NetworkAccess == NetworkAccess.Internet;
        RefreshStats();
    }
    
    [RelayCommand]
    private async Task SyncAsync()
    {
        if (!IsOnline || IsSyncing) return;
        
        IsSyncing = true;
        SyncStatusText = "กำลัง sync...";
        
        try
        {
            var report = await _syncEngine.SyncAsync("Product");
            
            LastSyncedAt = DateTime.Now;
            SyncStatusText = report.Pushed > 0 || report.Pulled > 0
                ? $"Sync เสร็จ: อัปโหลด {report.Pushed}, ดาวน์โหลด {report.Pulled}"
                : "ข้อมูลเป็นปัจจุบัน";
        }
        catch (Exception ex)
        {
            SyncStatusText = $"Sync ล้มเหลว: {ex.Message}";
        }
        finally
        {
            IsSyncing = false;
            RefreshStats();
        }
    }
    
    private void RefreshStats()
    {
        PendingChanges = _db.Table<ChangeRecord>().Count(c => c.Status == "Pending");
        FailedChanges = _db.Table<ChangeRecord>().Count(c => c.Status == "Failed");
        
        var state = _db.Table<SyncState>().FirstOrDefault();
        LastSyncedAt = state?.LastSyncedAt;
        
        if (LastSyncedAt.HasValue)
        {
            var elapsed = DateTime.Now - LastSyncedAt.Value;
            SyncStatusText = elapsed.TotalMinutes < 1
                ? "Sync เมื่อสักครู่"
                : elapsed.TotalHours < 1
                    ? $"Sync เมื่อ {(int)elapsed.TotalMinutes} นาทีที่แล้ว"
                    : $"Sync เมื่อ {(int)elapsed.TotalHours} ชั่วโมงที่แล้ว";
        }
    }
}
```

---

## Step 437: Sync Conflict UI

```csharp
// ============================================
// Conflict Resolution UI
// ============================================

public partial class ConflictResolutionViewModel : ObservableObject
{
    private readonly SQLiteConnection _db;
    private readonly ISyncApi _api;
    
    [ObservableProperty] 
    private ObservableCollection<ConflictItem> _conflicts = new();
    
    public ConflictResolutionViewModel(SQLiteConnection db, ISyncApi api)
    {
        _db = db;
        _api = api;
        LoadConflicts();
    }
    
    private void LoadConflicts()
    {
        var conflicted = _db.Table<ChangeRecord>()
            .Where(c => c.Status == "Conflicted")
            .ToList();
        
        Conflicts = new ObservableCollection<ConflictItem>(
            conflicted.Select(c => new ConflictItem(c)));
    }
    
    [RelayCommand]
    private async Task ResolveWithLocalAsync(ConflictItem conflict)
    {
        // Keep local, mark as synced
        conflict.Record.Status = "Synced";
        _db.Update(conflict.Record);
        
        // Send local version to server
        await _api.ForceUpdateAsync(conflict.Record.EntityType,
            conflict.Record.EntityId, conflict.Record.Payload);
        
        Conflicts.Remove(conflict);
    }
    
    [RelayCommand]
    private async Task ResolveWithServerAsync(ConflictItem conflict)
    {
        // Discard local, fetch server version
        _db.Delete(conflict.Record);
        
        var serverData = await _api.GetEntityAsync(
            conflict.Record.EntityType, conflict.Record.EntityId);
        
        if (serverData != null)
        {
            _db.Execute(
                $"INSERT OR REPLACE INTO {conflict.Record.EntityType}s VALUES (?)",
                serverData);
        }
        
        Conflicts.Remove(conflict);
    }
    
    [RelayCommand]
    private void MergeManually(ConflictItem conflict)
    {
        // Open manual merge UI
    }
}

public class ConflictItem
{
    public ChangeRecord Record { get; }
    public string EntityType => Record.EntityType;
    public string EntityId => Record.EntityId;
    public string LocalData => Record.Payload;
    public DateTime ConflictedAt => Record.CreatedAt;
    
    public ConflictItem(ChangeRecord record) => Record = record;
}
```

---

## Step 438: Sync Testing

```csharp
// ============================================
// Testing Sync Engine
// ============================================

public class SyncEngineTests
{
    [Fact]
    public async Task PushChanges_SendsPendingToServer()
    {
        // Arrange
        var db = new SQLiteConnection(":memory:");
        db.CreateTable<ChangeRecord>();
        db.CreateTable<SyncState>();
        
        var mockApi = new Mock<ISyncApi>();
        mockApi.Setup(a => a.PushChangesAsync(It.IsAny<PushRequest>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PushResponse([new PushResultItem("1", true, null)]));
        
        mockApi.Setup(a => a.PullChangesAsync(It.IsAny<PullRequest>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PullResult([], 100));
        
        db.Insert(new ChangeRecord
        {
            EntityType = "Product",
            EntityId = "p1",
            Operation = "Update",
            Payload = """{"id":"p1","name":"Test"}""",
            Status = "Pending",
            CreatedAt = DateTime.UtcNow
        });
        
        var engine = new DeltaSyncEngine(db, mockApi.Object);
        
        // Act
        var report = await engine.SyncAsync("Product");
        
        // Assert
        report.Pushed.Should().Be(1);
        
        var updatedRecord = db.Table<ChangeRecord>().First();
        updatedRecord.Status.Should().Be("Synced");
    }
    
    [Fact]
    public void VectorClock_HappensBefore_CorrectOrdering()
    {
        var clock1 = new VectorClock().Increment("A");
        var clock2 = clock1.Increment("A");
        
        clock1.HappensBefore(clock2).Should().BeTrue();
        clock2.HappensBefore(clock1).Should().BeFalse();
    }
    
    [Fact]
    public void VectorClock_Concurrent_DetectsParallelUpdates()
    {
        var initial = new VectorClock();
        var clock1 = initial.Increment("A");
        var clock2 = initial.Increment("B");
        
        clock1.IsConcurrentWith(clock2).Should().BeTrue();
    }
    
    [Fact]
    public void GCounter_Merge_TakesMax()
    {
        var c1 = new GCounter().Increment("device1", 5);
        var c2 = new GCounter().Increment("device2", 3);
        
        var merged = c1.Merge(c2);
        merged.Value.Should().Be(8);
    }
}
```

---

## Step 439: Incremental Sync

```csharp
// ============================================
// Incremental Sync with Cursor Pagination
// ============================================

public class IncrementalSyncService
{
    private readonly SQLiteConnection _db;
    private readonly ISyncApi _api;
    
    public IncrementalSyncService(SQLiteConnection db, ISyncApi api)
    {
        _db = db;
        _api = api;
    }
    
    public async Task FullSyncAsync(string entityType, CancellationToken ct = default)
    {
        // Reset cursor for full sync
        var state = new SyncState { EntityType = entityType, LastServerCursor = 0 };
        _db.InsertOrReplace(state);
        
        await IncrementalSyncAsync(entityType, ct);
    }
    
    public async Task IncrementalSyncAsync(string entityType, CancellationToken ct = default)
    {
        var state = _db.Table<SyncState>()
            .FirstOrDefault(s => s.EntityType == entityType)
            ?? new SyncState { EntityType = entityType };
        
        var cursor = state.LastServerCursor;
        
        while (true)
        {
            ct.ThrowIfCancellationRequested();
            
            var page = await _api.PullChangesAsync(
                new PullRequest(entityType, cursor, "device1", 200), ct);
            
            if (page.Changes.Count == 0) break;
            
            // Apply changes in transaction
            _db.BeginTransaction();
            try
            {
                foreach (var change in page.Changes)
                    ApplyServerChange(change);
                
                cursor = page.NextCursor;
                _db.InsertOrReplace(new SyncState
                {
                    EntityType = entityType,
                    LastSyncedAt = DateTime.UtcNow,
                    LastServerCursor = cursor
                });
                
                _db.Commit();
            }
            catch
            {
                _db.Rollback();
                throw;
            }
            
            if (!page.HasMore) break;
        }
    }
    
    private void ApplyServerChange(ChangeDto change)
    {
        var tableName = change.EntityType + "s";
        
        switch (change.Operation)
        {
            case "Delete":
                _db.Execute($"DELETE FROM {tableName} WHERE Id = ?", change.EntityId);
                break;
            default:
                _db.Execute(
                    $"INSERT OR REPLACE INTO {tableName} SELECT * FROM json_each(?)",
                    change.Payload);
                break;
        }
    }
}
```

---

## Step 440: Sync Dashboard

```csharp
// ============================================
// Sync Health Dashboard
// ============================================

public class SyncHealthDashboard
{
    private readonly SQLiteConnection _db;
    
    public SyncHealthDashboard(SQLiteConnection db) => _db = db;
    
    public SyncHealth GetHealth()
    {
        var pending = _db.Table<ChangeRecord>().Count(c => c.Status == "Pending");
        var failed = _db.Table<ChangeRecord>().Count(c => c.Status == "Failed");
        var conflicted = _db.Table<ChangeRecord>().Count(c => c.Status == "Conflicted");
        var syncedToday = _db.Table<ChangeRecord>()
            .Count(c => c.Status == "Synced" && c.CreatedAt >= DateTime.Today);
        
        var oldestPending = _db.Table<ChangeRecord>()
            .Where(c => c.Status == "Pending")
            .OrderBy(c => c.CreatedAt)
            .FirstOrDefault()?.CreatedAt;
        
        return new SyncHealth(
            PendingCount: pending,
            FailedCount: failed,
            ConflictedCount: conflicted,
            SyncedTodayCount: syncedToday,
            OldestPendingAge: oldestPending.HasValue
                ? DateTime.UtcNow - oldestPending.Value
                : null,
            IsHealthy: failed == 0 && conflicted == 0 && pending < 100,
            Status: DetermineStatus(pending, failed, conflicted));
    }
    
    private static string DetermineStatus(int pending, int failed, int conflicted)
    {
        if (failed > 0 || conflicted > 0) return "มีปัญหา";
        if (pending > 50) return "รอ sync จำนวนมาก";
        if (pending > 0) return "กำลังรอ sync";
        return "ปกติ";
    }
}

public record SyncHealth(
    int PendingCount,
    int FailedCount,
    int ConflictedCount,
    int SyncedTodayCount,
    TimeSpan? OldestPendingAge,
    bool IsHealthy,
    string Status
);
```

---

## สรุป Part 44

ใน Part 44 เราได้เรียนรู้:

1. **Offline-First Architecture** - ChangeRecord, VectorClock
2. **Vector Clock** - HappensBefore, concurrent detection
3. **Delta Sync** - Push/Pull with cursor pagination
4. **CRDT G-Counter** - Grow-only, conflict-free merging
5. **PNCounter** - Increment & decrement CRDT
6. **LWW-Register & OR-Set** - More CRDT types
7. **Sync Queue UI** - Status, progress, online detection
8. **Conflict Resolution UI** - Local wins, server wins, manual
9. **Testing Sync** - Unit tests for sync engine and CRDTs
10. **Incremental Sync** - Paginated delta pulls

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 44 | Steps 431-440*
