# Part 92: Advanced Database Patterns
## Steps 911-920: Query Optimization, Full-Text Search, Migrations, Encryption

---

## Step 911: Advanced SQLite Queries

```csharp
// ============================================
// Complex Queries with Raw SQL
// ============================================

public class AdvancedOrderRepository
{
    private readonly SQLiteAsyncConnection _db;
    
    public AdvancedOrderRepository(SQLiteAsyncConnection db) => _db = db;
    
    // Window functions for revenue analytics
    public async Task<List<RevenueTrend>> GetRevenueTrendAsync(int days = 30)
    {
        return await _db.QueryAsync<RevenueTrend>(@"
            SELECT
                date(CreatedAt) as Date,
                COUNT(*) as OrderCount,
                SUM(Total) as Revenue,
                AVG(Total) as AverageOrderValue,
                SUM(SUM(Total)) OVER (
                    ORDER BY date(CreatedAt)
                    ROWS UNBOUNDED PRECEDING
                ) as CumulativeRevenue
            FROM Orders
            WHERE CreatedAt >= date('now', ?)
              AND Status = 'Delivered'
            GROUP BY date(CreatedAt)
            ORDER BY Date", $"-{days} days");
    }
    
    // Cohort analysis: retention by signup week
    public async Task<List<CohortRow>> GetCohortAnalysisAsync()
    {
        return await _db.QueryAsync<CohortRow>(@"
            WITH UserFirstOrder AS (
                SELECT UserId, MIN(date(CreatedAt)) as FirstOrderDate
                FROM Orders WHERE Status = 'Delivered'
                GROUP BY UserId
            ),
            Cohorts AS (
                SELECT 
                    ufo.UserId,
                    strftime('%Y-W%W', ufo.FirstOrderDate) as Cohort,
                    CAST(
                        julianday(date(o.CreatedAt)) - julianday(ufo.FirstOrderDate)
                    AS INTEGER) / 7 as WeekOffset
                FROM UserFirstOrder ufo
                JOIN Orders o ON o.UserId = ufo.UserId AND o.Status = 'Delivered'
            )
            SELECT 
                Cohort,
                WeekOffset,
                COUNT(DISTINCT UserId) as RetainedUsers
            FROM Cohorts
            GROUP BY Cohort, WeekOffset
            ORDER BY Cohort, WeekOffset");
    }
    
    // RFM: Recency, Frequency, Monetary scoring
    public async Task<List<RfmScore>> CalculateRfmAsync()
    {
        return await _db.QueryAsync<RfmScore>(@"
            WITH UserStats AS (
                SELECT
                    UserId,
                    CAST(julianday('now') - julianday(MAX(CreatedAt)) AS INTEGER) as DaysSinceLastOrder,
                    COUNT(*) as OrderCount,
                    SUM(Total) as TotalSpend
                FROM Orders WHERE Status = 'Delivered'
                GROUP BY UserId
            )
            SELECT
                UserId,
                DaysSinceLastOrder,
                OrderCount,
                TotalSpend,
                CASE WHEN DaysSinceLastOrder <= 7 THEN 5
                     WHEN DaysSinceLastOrder <= 14 THEN 4
                     WHEN DaysSinceLastOrder <= 30 THEN 3
                     WHEN DaysSinceLastOrder <= 60 THEN 2
                     ELSE 1 END as RecencyScore,
                CASE WHEN OrderCount >= 20 THEN 5
                     WHEN OrderCount >= 10 THEN 4
                     WHEN OrderCount >= 5 THEN 3
                     WHEN OrderCount >= 2 THEN 2
                     ELSE 1 END as FrequencyScore,
                CASE WHEN TotalSpend >= 5000 THEN 5
                     WHEN TotalSpend >= 2000 THEN 4
                     WHEN TotalSpend >= 1000 THEN 3
                     WHEN TotalSpend >= 500 THEN 2
                     ELSE 1 END as MonetaryScore
            FROM UserStats");
    }
}

public record RevenueTrend(string Date, int OrderCount, decimal Revenue, decimal AverageOrderValue, decimal CumulativeRevenue);
public record CohortRow(string Cohort, int WeekOffset, int RetainedUsers);
public record RfmScore(string UserId, int DaysSinceLastOrder, int OrderCount, decimal TotalSpend, int RecencyScore, int FrequencyScore, int MonetaryScore);
```

---

## Step 912: Full-Text Search (FTS5)

```csharp
// ============================================
// SQLite FTS5 Full-Text Search
// ============================================

public class FtsSearchService
{
    private readonly SQLiteAsyncConnection _db;
    
    public FtsSearchService(SQLiteAsyncConnection db) => _db = db;
    
    public async Task CreateFtsIndexAsync()
    {
        // Create FTS5 virtual table
        await _db.ExecuteAsync(@"
            CREATE VIRTUAL TABLE IF NOT EXISTS MenuItemsFts USING fts5(
                Id UNINDEXED,
                Name,
                Description,
                Tags,
                content=MenuItems,
                content_rowid=rowid,
                tokenize='unicode61 categories remove_diacritics 2'
            )");
        
        // Populate from existing data
        await _db.ExecuteAsync(@"
            INSERT INTO MenuItemsFts(rowid, Id, Name, Description, Tags)
            SELECT rowid, Id, Name, Description, Tags FROM MenuItems");
        
        // Triggers to keep FTS in sync
        await _db.ExecuteAsync(@"
            CREATE TRIGGER IF NOT EXISTS MenuItems_ai AFTER INSERT ON MenuItems BEGIN
                INSERT INTO MenuItemsFts(rowid, Id, Name, Description, Tags)
                VALUES (new.rowid, new.Id, new.Name, new.Description, new.Tags);
            END;");
        
        await _db.ExecuteAsync(@"
            CREATE TRIGGER IF NOT EXISTS MenuItems_ad AFTER DELETE ON MenuItems BEGIN
                INSERT INTO MenuItemsFts(MenuItemsFts, rowid, Id, Name, Description, Tags)
                VALUES ('delete', old.rowid, old.Id, old.Name, old.Description, old.Tags);
            END;");
    }
    
    public async Task<List<FtsResult>> SearchAsync(string query, int limit = 20)
    {
        // Use FTS5 MATCH syntax; highlight matched terms
        var ftsQuery = string.Join(" OR ", query.Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Select(term => $"{term}*"));
        
        return await _db.QueryAsync<FtsResult>(@"
            SELECT 
                Id,
                highlight(MenuItemsFts, 1, '<b>', '</b>') as Name,
                highlight(MenuItemsFts, 2, '<b>', '</b>') as Description,
                bm25(MenuItemsFts) as Rank
            FROM MenuItemsFts
            WHERE MenuItemsFts MATCH ?
            ORDER BY Rank
            LIMIT ?", ftsQuery, limit);
    }
    
    public async Task<List<string>> GetSuggestionsAsync(string prefix)
    {
        return (await _db.QueryAsync<SuggestionRow>(@"
            SELECT DISTINCT Name
            FROM MenuItemsFts
            WHERE Name MATCH ?
            LIMIT 5", $"{prefix}*"))
            .Select(r => r.Name).ToList();
    }
    
    private record SuggestionRow(string Name);
}

public record FtsResult(string Id, string Name, string Description, double Rank);
```

---

## Step 913: Database Migrations

```csharp
// ============================================
// Versioned Database Migration Runner
// ============================================

public interface IMigration
{
    int Version { get; }
    Task UpAsync(SQLiteAsyncConnection db);
    Task DownAsync(SQLiteAsyncConnection db);
}

public class MigrationRunner
{
    private readonly SQLiteAsyncConnection _db;
    private readonly IEnumerable<IMigration> _migrations;
    
    public MigrationRunner(SQLiteAsyncConnection db, IEnumerable<IMigration> migrations)
    {
        _db = db;
        _migrations = migrations.OrderBy(m => m.Version);
    }
    
    public async Task MigrateAsync()
    {
        await CreateMigrationsTableAsync();
        
        var currentVersion = await GetCurrentVersionAsync();
        
        foreach (var migration in _migrations.Where(m => m.Version > currentVersion))
        {
            await _db.RunInTransactionAsync(async conn =>
            {
                await migration.UpAsync(_db);
                
                conn.Execute(
                    "INSERT INTO Migrations (Version, AppliedAt) VALUES (?, ?)",
                    migration.Version, DateTime.UtcNow);
            });
        }
    }
    
    public async Task RollbackAsync(int targetVersion)
    {
        var currentVersion = await GetCurrentVersionAsync();
        
        foreach (var migration in _migrations
            .Where(m => m.Version > targetVersion && m.Version <= currentVersion)
            .OrderByDescending(m => m.Version))
        {
            await _db.RunInTransactionAsync(async conn =>
            {
                await migration.DownAsync(_db);
                conn.Execute("DELETE FROM Migrations WHERE Version = ?", migration.Version);
            });
        }
    }
    
    private Task CreateMigrationsTableAsync()
        => _db.ExecuteAsync(@"
            CREATE TABLE IF NOT EXISTS Migrations (
                Version INTEGER PRIMARY KEY,
                AppliedAt TEXT NOT NULL
            )");
    
    private async Task<int> GetCurrentVersionAsync()
    {
        var result = await _db.QueryAsync<VersionRow>("SELECT MAX(Version) as V FROM Migrations");
        return result?.FirstOrDefault()?.V ?? 0;
    }
    
    private record VersionRow(int V);
}

// Concrete migrations
public class Migration001CreateOrders : IMigration
{
    public int Version => 1;
    
    public Task UpAsync(SQLiteAsyncConnection db) => db.ExecuteAsync(@"
        CREATE TABLE IF NOT EXISTS Orders (
            Id TEXT PRIMARY KEY,
            UserId TEXT NOT NULL,
            RestaurantId TEXT NOT NULL,
            Total REAL NOT NULL,
            Status TEXT NOT NULL DEFAULT 'Pending',
            CreatedAt TEXT NOT NULL,
            UpdatedAt TEXT
        );
        CREATE INDEX IF NOT EXISTS IX_Orders_UserId ON Orders(UserId);
        CREATE INDEX IF NOT EXISTS IX_Orders_RestaurantId ON Orders(RestaurantId);");
    
    public Task DownAsync(SQLiteAsyncConnection db) => db.ExecuteAsync("DROP TABLE IF EXISTS Orders");
}

public class Migration002AddDeliveryAddress : IMigration
{
    public int Version => 2;
    
    public Task UpAsync(SQLiteAsyncConnection db) => db.ExecuteAsync(@"
        ALTER TABLE Orders ADD COLUMN DeliveryAddress TEXT;
        ALTER TABLE Orders ADD COLUMN DeliveryLat REAL;
        ALTER TABLE Orders ADD COLUMN DeliveryLng REAL;");
    
    public Task DownAsync(SQLiteAsyncConnection db)
        => Task.CompletedTask; // SQLite doesn't support DROP COLUMN easily
}
```

---

## Step 914: Database Encryption

```csharp
// ============================================
// SQLCipher Encrypted Database
// ============================================

// NuGet: SQLitePCLRaw.bundle_e_sqlcipher (for encrypted SQLite)
// Or use MAUI's platform keychain to store DB key

public class EncryptedDatabaseFactory
{
    private readonly SecureStorageService _secureStorage;
    
    public EncryptedDatabaseFactory(SecureStorageService secureStorage)
        => _secureStorage = secureStorage;
    
    public async Task<SQLiteAsyncConnection> CreateAsync(string dbPath)
    {
        var key = await GetOrCreateKeyAsync();
        
        // SQLCipher connection string with PRAGMA key
        var connection = new SQLiteAsyncConnection(dbPath);
        
        // Set encryption key (SQLCipher)
        await connection.ExecuteAsync($"PRAGMA key = '{key}'");
        await connection.ExecuteAsync("PRAGMA cipher_page_size = 4096");
        await connection.ExecuteAsync("PRAGMA kdf_iter = 64000");
        
        return connection;
    }
    
    private async Task<string> GetOrCreateKeyAsync()
    {
        var existing = await _secureStorage.GetAsync("db_encryption_key");
        if (existing != null) return existing;
        
        var key = Convert.ToBase64String(EncryptionService.GenerateKey());
        await _secureStorage.SetAsync("db_encryption_key", key);
        return key;
    }
    
    // Re-key the database (rotate encryption key)
    public async Task RekeyAsync(SQLiteAsyncConnection db, string newKey)
    {
        await db.ExecuteAsync($"PRAGMA rekey = '{newKey}'");
        await _secureStorage.SetAsync("db_encryption_key", newKey);
    }
}
```

---

## Step 915: Query Performance Analysis

```csharp
// ============================================
// EXPLAIN QUERY PLAN analysis
// ============================================

public class QueryPerformanceAnalyzer
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ILogger<QueryPerformanceAnalyzer> _logger;
    
    public QueryPerformanceAnalyzer(SQLiteAsyncConnection db, ILogger<QueryPerformanceAnalyzer> logger)
    {
        _db = db;
        _logger = logger;
    }
    
    public async Task<QueryPlan> ExplainAsync(string sql, params object[] args)
    {
        var rows = await _db.QueryAsync<ExplainRow>(
            $"EXPLAIN QUERY PLAN {sql}", args);
        
        var plan = new QueryPlan(sql, rows);
        
        if (plan.UsesFullTableScan)
            _logger.LogWarning("Full table scan detected: {Sql}", sql);
        
        return plan;
    }
    
    public async Task AnalyzeAllQueriesAsync()
    {
        // Common queries to analyze
        var queries = new Dictionary<string, (string sql, object[] args)>
        {
            ["OrdersByUser"] = ("SELECT * FROM Orders WHERE UserId = ?", new object[] { "test" }),
            ["MenuItemsByRestaurant"] = ("SELECT * FROM MenuItems WHERE RestaurantId = ?", new object[] { "test" }),
            ["RecentOrders"] = ("SELECT * FROM Orders ORDER BY CreatedAt DESC LIMIT 20", Array.Empty<object>())
        };
        
        foreach (var (name, (sql, args)) in queries)
        {
            var plan = await ExplainAsync(sql, args);
            _logger.LogInformation("Query {Name}: {Plan}", name, plan.Summary);
        }
    }
}

public class QueryPlan
{
    public string Sql { get; }
    public List<ExplainRow> Rows { get; }
    public bool UsesFullTableScan => Rows.Any(r => r.Detail.Contains("SCAN") && !r.Detail.Contains("COVERING"));
    public string Summary => string.Join(" → ", Rows.Select(r => r.Detail));
    
    public QueryPlan(string sql, List<ExplainRow> rows) { Sql = sql; Rows = rows; }
}

public record ExplainRow(int Id, int Parent, int NotUsed, string Detail);
```

---

## Step 916: Database Connection Pooling

```csharp
// ============================================
// Connection Pool for Multi-threaded Access
// ============================================

public class DatabasePool : IDisposable
{
    private readonly ConcurrentQueue<SQLiteAsyncConnection> _pool = new();
    private readonly SemaphoreSlim _semaphore;
    private readonly string _dbPath;
    private const int PoolSize = 5;
    
    public DatabasePool(string dbPath)
    {
        _dbPath = dbPath;
        _semaphore = new SemaphoreSlim(PoolSize, PoolSize);
        
        for (int i = 0; i < PoolSize; i++)
            _pool.Enqueue(new SQLiteAsyncConnection(_dbPath, SQLiteOpenFlags.ReadWrite | SQLiteOpenFlags.SharedCache));
    }
    
    public async Task<T> WithConnectionAsync<T>(Func<SQLiteAsyncConnection, Task<T>> work)
    {
        await _semaphore.WaitAsync();
        
        if (!_pool.TryDequeue(out var conn))
            conn = new SQLiteAsyncConnection(_dbPath);
        
        try
        {
            return await work(conn);
        }
        finally
        {
            _pool.Enqueue(conn);
            _semaphore.Release();
        }
    }
    
    public void Dispose()
    {
        while (_pool.TryDequeue(out var conn))
            conn.CloseAsync().GetAwaiter().GetResult();
        
        _semaphore.Dispose();
    }
}
```

---

## Step 917: Batch Operations

```csharp
// ============================================
// High-Performance Batch Insert/Update
// ============================================

public class BatchDatabaseService
{
    private readonly SQLiteAsyncConnection _db;
    
    public BatchDatabaseService(SQLiteAsyncConnection db) => _db = db;
    
    public async Task<int> BulkInsertAsync<T>(IEnumerable<T> items, int chunkSize = 500)
    {
        var list = items.ToList();
        int total = 0;
        
        for (int i = 0; i < list.Count; i += chunkSize)
        {
            var chunk = list.Skip(i).Take(chunkSize).ToList();
            
            await _db.RunInTransactionAsync(conn =>
            {
                foreach (var item in chunk)
                    conn.Insert(item);
            });
            
            total += chunk.Count;
        }
        
        return total;
    }
    
    public async Task<int> BulkUpdateAsync<T>(IEnumerable<T> items, int chunkSize = 500)
    {
        var list = items.ToList();
        int total = 0;
        
        for (int i = 0; i < list.Count; i += chunkSize)
        {
            var chunk = list.Skip(i).Take(chunkSize).ToList();
            
            await _db.RunInTransactionAsync(conn =>
            {
                foreach (var item in chunk)
                    conn.Update(item);
            });
            
            total += chunk.Count;
        }
        
        return total;
    }
    
    // Upsert with conflict resolution
    public async Task BulkUpsertAsync<T>(IEnumerable<T> items) where T : class
    {
        await _db.RunInTransactionAsync(conn =>
        {
            foreach (var item in items)
                conn.InsertOrReplace(item);
        });
    }
    
    // Delete by IDs in bulk
    public async Task BulkDeleteAsync(string tableName, IEnumerable<string> ids)
    {
        var idList = ids.ToList();
        for (int i = 0; i < idList.Count; i += 999) // SQLite limit
        {
            var batch = idList.Skip(i).Take(999).ToList();
            var placeholders = string.Join(",", Enumerable.Repeat("?", batch.Count));
            await _db.ExecuteAsync(
                $"DELETE FROM {tableName} WHERE Id IN ({placeholders})",
                batch.Cast<object>().ToArray());
        }
    }
}
```

---

## Step 918: Database Health Monitor

```csharp
// ============================================
// Database Health & Maintenance
// ============================================

public class DatabaseMaintenanceService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly ILogger<DatabaseMaintenanceService> _logger;
    
    public DatabaseMaintenanceService(SQLiteAsyncConnection db, ILogger<DatabaseMaintenanceService> logger)
    {
        _db = db;
        _logger = logger;
    }
    
    public async Task<DatabaseHealth> GetHealthAsync()
    {
        var integrity = await CheckIntegrityAsync();
        var sizeBytes = GetDatabaseSize();
        var tableStats = await GetTableStatsAsync();
        
        return new DatabaseHealth(integrity, sizeBytes, tableStats);
    }
    
    private async Task<bool> CheckIntegrityAsync()
    {
        var result = await _db.QueryAsync<IntegrityRow>("PRAGMA integrity_check");
        var passed = result?.FirstOrDefault()?.Value == "ok";
        
        if (!passed)
            _logger.LogCritical("Database integrity check FAILED");
        
        return passed;
    }
    
    private long GetDatabaseSize()
    {
        var dbPath = _db.DatabasePath;
        return File.Exists(dbPath) ? new FileInfo(dbPath).Length : 0;
    }
    
    private async Task<List<TableStat>> GetTableStatsAsync()
    {
        return await _db.QueryAsync<TableStat>(@"
            SELECT name as TableName, 0 as RowCount FROM sqlite_master WHERE type='table'");
    }
    
    public async Task VacuumAsync()
    {
        _logger.LogInformation("Starting VACUUM");
        var sw = Stopwatch.StartNew();
        await _db.ExecuteAsync("VACUUM");
        _logger.LogInformation("VACUUM completed in {Ms}ms", sw.ElapsedMilliseconds);
    }
    
    public async Task AnalyzeAsync()
        => await _db.ExecuteAsync("ANALYZE");
    
    // Run maintenance on app startup (if last run > 7 days ago)
    public async Task MaintenanceIfNeededAsync()
    {
        var lastRun = Preferences.Get("db_maintenance_last", DateTime.MinValue.Ticks);
        var lastRunDate = new DateTime(lastRun);
        
        if ((DateTime.UtcNow - lastRunDate).TotalDays < 7) return;
        
        await AnalyzeAsync();
        
        var health = await GetHealthAsync();
        if (health.SizeBytes > 50_000_000) // > 50 MB
            await VacuumAsync();
        
        Preferences.Set("db_maintenance_last", DateTime.UtcNow.Ticks);
    }
    
    private record IntegrityRow(string Value);
}

public record DatabaseHealth(bool IsIntact, long SizeBytes, List<TableStat> Tables);
public record TableStat(string TableName, int RowCount);
```

---

## Step 919: Reactive Database Queries

```csharp
// ============================================
// Reactive Database with Live Queries
// ============================================

public class ReactiveOrderService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly Subject<string> _changedTable = new();
    
    public IObservable<List<OrderReadModel>> WatchUserOrdersAsync(string userId)
    {
        return Observable.Interval(TimeSpan.FromSeconds(5))
            .StartWith(-1)
            .SelectMany(_ => Observable.FromAsync(async () =>
                await _db.Table<OrderReadModel>()
                    .Where(o => o.CustomerId == userId)
                    .OrderByDescending(o => o.CreatedAt)
                    .Take(20)
                    .ToListAsync()))
            .DistinctUntilChanged(new OrderListComparer());
    }
    
    // Notify on write
    public async Task UpdateOrderAsync(OrderReadModel order)
    {
        await _db.UpdateAsync(order);
        _changedTable.OnNext("Orders");
    }
    
    private class OrderListComparer : IEqualityComparer<List<OrderReadModel>>
    {
        public bool Equals(List<OrderReadModel>? x, List<OrderReadModel>? y)
        {
            if (x == null || y == null) return false;
            if (x.Count != y.Count) return false;
            return x.Zip(y).All(pair => pair.First.Id == pair.Second.Id &&
                pair.First.Status == pair.Second.Status);
        }
        
        public int GetHashCode(List<OrderReadModel> obj) => obj.Count.GetHashCode();
    }
}
```

---

## Step 920: Database Tests

```csharp
// ============================================
// Database Layer Tests
// ============================================

[TestFixture]
public class DatabaseTests
{
    private SQLiteAsyncConnection _db = null!;
    
    [SetUp]
    public async Task SetUp()
    {
        _db = new SQLiteAsyncConnection(":memory:");
        await new MigrationRunner(_db, new IMigration[]
        {
            new Migration001CreateOrders(),
            new Migration002AddDeliveryAddress()
        }).MigrateAsync();
    }
    
    [TearDown]
    public async Task TearDown()
        => await _db.CloseAsync();
    
    [Test]
    public async Task Migration_RunsSuccessfully_CreatesOrders()
    {
        var tables = await _db.QueryAsync<TableRow>("SELECT name FROM sqlite_master WHERE type='table'");
        Assert.That(tables.Select(t => t.Name), Contains.Item("Orders"));
        Assert.That(tables.Select(t => t.Name), Contains.Item("Migrations"));
    }
    
    [Test]
    public async Task BulkInsert_1000Rows_UnderOneSecond()
    {
        await _db.CreateTableAsync<OrderEntity>();
        
        var orders = Enumerable.Range(1, 1000).Select(i => new OrderEntity
        {
            Id = Guid.NewGuid().ToString(),
            UserId = "user-1",
            RestaurantId = "rest-1",
            Total = 100 + i,
            Status = "Pending",
            CreatedAt = DateTime.UtcNow
        });
        
        var svc = new BatchDatabaseService(_db);
        var sw = Stopwatch.StartNew();
        await svc.BulkInsertAsync(orders);
        sw.Stop();
        
        Assert.That(sw.ElapsedMilliseconds, Is.LessThan(1000),
            $"Bulk insert took {sw.ElapsedMilliseconds}ms");
        
        var count = await _db.Table<OrderEntity>().CountAsync();
        Assert.That(count, Is.EqualTo(1000));
    }
    
    [Test]
    public async Task DatabaseHealth_IntegrityCheck_Passes()
    {
        var svc = new DatabaseMaintenanceService(_db, null!);
        var health = await svc.GetHealthAsync();
        Assert.That(health.IsIntact, Is.True);
    }
    
    private record TableRow(string Name);
}
```

---

## สรุป Part 92

ใน Part 92 เราได้เรียนรู้:

1. **Advanced SQL** - Window functions, cohort analysis, RFM scoring
2. **FTS5 Full-Text Search** - Virtual tables, BM25 ranking, trigger sync
3. **Migrations** - Versioned up/down runner, transaction per migration
4. **Database Encryption** - SQLCipher PRAGMA key, rekey rotation
5. **EXPLAIN QUERY PLAN** - Full table scan detection, index verification
6. **Connection Pooling** - ConcurrentQueue pool, semaphore guard
7. **Batch Operations** - Chunked insert/update/delete in transactions
8. **Health Monitor** - Integrity check, VACUUM, ANALYZE, size check
9. **Reactive Queries** - Observable.Interval live query, DistinctUntilChanged
10. **Database Tests** - Migration, bulk insert < 1s, integrity check

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 92 | Steps 911-920*
