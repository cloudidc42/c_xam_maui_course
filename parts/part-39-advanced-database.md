# Part 39: Advanced Database Patterns
## Steps 381-390: SQLite Migrations, Queries, Sync

---

## Step 381: Database Migrations

```csharp
// ============================================
// SQLite Migration System
// ============================================

public interface IMigration
{
    int Version { get; }
    string Description { get; }
    void Up(SQLiteConnection db);
    void Down(SQLiteConnection db);
}

public class MigrationRunner
{
    private readonly SQLiteConnection _db;
    private readonly List<IMigration> _migrations;
    
    public MigrationRunner(SQLiteConnection db)
    {
        _db = db;
        _migrations = DiscoverMigrations();
        EnsureMigrationsTable();
    }
    
    private void EnsureMigrationsTable()
    {
        _db.Execute(@"
            CREATE TABLE IF NOT EXISTS __Migrations (
                Version INTEGER PRIMARY KEY,
                Description TEXT NOT NULL,
                AppliedAt TEXT NOT NULL
            )");
    }
    
    private static List<IMigration> DiscoverMigrations()
    {
        return typeof(MigrationRunner).Assembly
            .GetTypes()
            .Where(t => typeof(IMigration).IsAssignableFrom(t) && !t.IsInterface && !t.IsAbstract)
            .Select(t => (IMigration)Activator.CreateInstance(t)!)
            .OrderBy(m => m.Version)
            .ToList();
    }
    
    public void MigrateToLatest()
    {
        var currentVersion = GetCurrentVersion();
        var pending = _migrations.Where(m => m.Version > currentVersion).ToList();
        
        foreach (var migration in pending)
        {
            _db.BeginTransaction();
            try
            {
                migration.Up(_db);
                _db.Execute(@"
                    INSERT INTO __Migrations (Version, Description, AppliedAt)
                    VALUES (?, ?, ?)",
                    migration.Version,
                    migration.Description,
                    DateTime.UtcNow.ToString("O"));
                _db.Commit();
            }
            catch
            {
                _db.Rollback();
                throw;
            }
        }
    }
    
    private int GetCurrentVersion()
    {
        try
        {
            return _db.ExecuteScalar<int>("SELECT MAX(Version) FROM __Migrations");
        }
        catch { return 0; }
    }
}

// Migration 001
public class M001_CreateInitialSchema : IMigration
{
    public int Version => 1;
    public string Description => "Create initial schema";
    
    public void Up(SQLiteConnection db)
    {
        db.Execute(@"
            CREATE TABLE IF NOT EXISTS Products (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Price REAL NOT NULL,
                Stock INTEGER NOT NULL DEFAULT 0,
                CreatedAt TEXT NOT NULL
            )");
    }
    
    public void Down(SQLiteConnection db)
    {
        db.Execute("DROP TABLE IF EXISTS Products");
    }
}

// Migration 002
public class M002_AddProductCategory : IMigration
{
    public int Version => 2;
    public string Description => "Add category to products";
    
    public void Up(SQLiteConnection db)
    {
        db.Execute("ALTER TABLE Products ADD COLUMN CategoryId INTEGER");
        db.Execute(@"
            CREATE TABLE IF NOT EXISTS Categories (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL UNIQUE
            )");
        db.Execute("CREATE INDEX IF NOT EXISTS IX_Products_CategoryId ON Products(CategoryId)");
    }
    
    public void Down(SQLiteConnection db)
    {
        // SQLite doesn't support DROP COLUMN easily, workaround:
        db.Execute("DROP TABLE IF EXISTS Categories");
    }
}
```

---

## Step 382: Advanced Queries

```csharp
// ============================================
// Complex SQLite Queries
// ============================================

public class ProductQueryService
{
    private readonly SQLiteConnection _db;
    
    public ProductQueryService(SQLiteConnection db) => _db = db;
    
    // Full-text search
    public List<Product> SearchFullText(string query)
    {
        // Enable FTS5 extension
        _db.Execute(@"
            CREATE VIRTUAL TABLE IF NOT EXISTS products_fts 
            USING fts5(name, description, content='Products', content_rowid='Id')");
        
        return _db.Query<Product>(@"
            SELECT p.* FROM Products p
            JOIN products_fts fts ON p.Id = fts.rowid
            WHERE products_fts MATCH ?
            ORDER BY rank
            LIMIT 50",
            query);
    }
    
    // Pagination with total count (single query)
    public (List<Product> Items, int Total) GetPaged(
        int page, int size, string? category = null, string? sort = null)
    {
        var whereClause = category != null ? "WHERE c.Name = ?" : "";
        var sortClause = sort switch
        {
            "price_asc" => "ORDER BY p.Price ASC",
            "price_desc" => "ORDER BY p.Price DESC",
            "newest" => "ORDER BY p.CreatedAt DESC",
            _ => "ORDER BY p.Name ASC"
        };
        
        var sql = $@"
            SELECT p.*, c.Name as CategoryName,
                   COUNT(*) OVER() as TotalCount
            FROM Products p
            LEFT JOIN Categories c ON p.CategoryId = c.Id
            {whereClause}
            {sortClause}
            LIMIT ? OFFSET ?";
        
        var args = category != null
            ? new object[] { category, size, (page - 1) * size }
            : new object[] { size, (page - 1) * size };
        
        var rows = _db.Query<ProductWithMeta>(sql, args);
        var total = rows.FirstOrDefault()?.TotalCount ?? 0;
        
        return (rows.Select(r => r.ToProduct()).ToList(), total);
    }
    
    // Aggregations
    public CategoryStats GetCategoryStats()
    {
        return _db.Query<CategoryStats>(@"
            SELECT 
                c.Name as Category,
                COUNT(p.Id) as ProductCount,
                AVG(p.Price) as AveragePrice,
                MIN(p.Price) as MinPrice,
                MAX(p.Price) as MaxPrice,
                SUM(p.Stock) as TotalStock
            FROM Categories c
            LEFT JOIN Products p ON c.Id = p.CategoryId
            GROUP BY c.Id, c.Name
            ORDER BY ProductCount DESC")
            .First();
    }
    
    // Recursive CTE for category hierarchy
    public List<CategoryNode> GetCategoryTree()
    {
        return _db.Query<CategoryNode>(@"
            WITH RECURSIVE category_tree AS (
                SELECT Id, Name, ParentId, 0 AS Level, Name AS Path
                FROM Categories
                WHERE ParentId IS NULL
                
                UNION ALL
                
                SELECT c.Id, c.Name, c.ParentId, ct.Level + 1, 
                       ct.Path || ' > ' || c.Name
                FROM Categories c
                JOIN category_tree ct ON c.ParentId = ct.Id
            )
            SELECT * FROM category_tree ORDER BY Path");
    }
    
    // Window functions
    public List<ProductRank> GetProductRankings()
    {
        return _db.Query<ProductRank>(@"
            SELECT 
                p.Id, p.Name, p.Price, c.Name as Category,
                RANK() OVER (ORDER BY p.Price DESC) as GlobalRank,
                RANK() OVER (PARTITION BY p.CategoryId ORDER BY p.Price DESC) as CategoryRank,
                LAG(p.Price) OVER (ORDER BY p.CreatedAt) as PreviousPrice,
                p.Price - LAG(p.Price) OVER (ORDER BY p.CreatedAt) as PriceDelta
            FROM Products p
            LEFT JOIN Categories c ON p.CategoryId = c.Id
            ORDER BY GlobalRank");
    }
}

public record CategoryStats(
    string Category, int ProductCount, double AveragePrice,
    double MinPrice, double MaxPrice, int TotalStock);

public record ProductRank(
    int Id, string Name, decimal Price, string Category,
    int GlobalRank, int CategoryRank, decimal? PreviousPrice, decimal? PriceDelta);
```

---

## Step 383: Repository Pattern with Specification

```csharp
// ============================================
// Generic Repository with Specification
// ============================================

public class Specification<T>
{
    public Expression<Func<T, bool>>? Criteria { get; private set; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }
    public Expression<Func<T, object>>? OrderByDescending { get; private set; }
    public int? Take { get; private set; }
    public int? Skip { get; private set; }
    
    protected void AddCriteria(Expression<Func<T, bool>> criteria)
        => Criteria = criteria;
    
    protected void AddInclude(Expression<Func<T, object>> include)
        => Includes.Add(include);
    
    protected void ApplyOrderBy(Expression<Func<T, object>> orderBy)
        => OrderBy = orderBy;
    
    protected void ApplyOrderByDescending(Expression<Func<T, object>> orderByDesc)
        => OrderByDescending = orderByDesc;
    
    protected void ApplyPaging(int page, int size)
    {
        Skip = (page - 1) * size;
        Take = size;
    }
}

// Concrete specification
public class ActiveProductsByCategory : Specification<Product>
{
    public ActiveProductsByCategory(string category, int page = 1, int size = 20)
    {
        AddCriteria(p => p.CategoryName == category && p.IsActive && p.Stock > 0);
        ApplyOrderBy(p => p.Name);
        ApplyPaging(page, size);
    }
}

public class LowStockProducts : Specification<Product>
{
    public LowStockProducts(int threshold = 5)
    {
        AddCriteria(p => p.Stock <= threshold && p.IsActive);
        ApplyOrderBy(p => p.Stock);
    }
}

// SQLite Repository with Specification
public class SQLiteRepository<T> where T : class, new()
{
    private readonly SQLiteConnection _db;
    
    public SQLiteRepository(SQLiteConnection db) => _db = db;
    
    public List<T> Find(Specification<T> spec)
    {
        var query = _db.Table<T>();
        
        if (spec.Criteria != null)
            query = query.Where(spec.Criteria);
        
        if (spec.OrderBy != null)
        {
            // SQLite.NET has limited expression support
            // For complex cases, use raw SQL
        }
        
        if (spec.Skip.HasValue)
            query = query.Skip(spec.Skip.Value);
        
        if (spec.Take.HasValue)
            query = query.Take(spec.Take.Value);
        
        return query.ToList();
    }
    
    public int Count(Specification<T> spec)
    {
        var query = _db.Table<T>();
        if (spec.Criteria != null)
            query = query.Where(spec.Criteria);
        return query.Count();
    }
}
```

---

## Step 384: Unit of Work

```csharp
// ============================================
// Unit of Work Pattern
// ============================================

public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    IOrderRepository Orders { get; }
    
    void BeginTransaction();
    void Commit();
    void Rollback();
    Task<int> SaveChangesAsync();
}

public class SQLiteUnitOfWork : IUnitOfWork
{
    private readonly SQLiteConnection _db;
    private bool _transactionStarted;
    
    private IProductRepository? _products;
    private ICategoryRepository? _categories;
    private IOrderRepository? _orders;
    
    public SQLiteUnitOfWork(SQLiteConnection db) => _db = db;
    
    public IProductRepository Products
        => _products ??= new ProductRepository(_db);
    
    public ICategoryRepository Categories
        => _categories ??= new CategoryRepository(_db);
    
    public IOrderRepository Orders
        => _orders ??= new OrderRepository(_db);
    
    public void BeginTransaction()
    {
        if (!_transactionStarted)
        {
            _db.BeginTransaction();
            _transactionStarted = true;
        }
    }
    
    public void Commit()
    {
        if (_transactionStarted)
        {
            _db.Commit();
            _transactionStarted = false;
        }
    }
    
    public void Rollback()
    {
        if (_transactionStarted)
        {
            _db.Rollback();
            _transactionStarted = false;
        }
    }
    
    public Task<int> SaveChangesAsync()
    {
        // SQLite.NET auto-saves, but we can track pending operations
        return Task.FromResult(0);
    }
    
    public void Dispose()
    {
        if (_transactionStarted) Rollback();
    }
}

// Usage in service
public class OrderService
{
    private readonly IUnitOfWork _uow;
    
    public OrderService(IUnitOfWork uow) => _uow = uow;
    
    public async Task PlaceOrderAsync(Cart cart, Address shippingAddress)
    {
        _uow.BeginTransaction();
        try
        {
            // Create order
            var order = new Order
            {
                UserId = cart.UserId,
                Status = OrderStatus.Pending,
                ShippingAddress = shippingAddress,
                CreatedAt = DateTime.UtcNow
            };
            
            await Task.Run(() => _uow.Orders.Insert(order));
            
            // Create order items & update stock
            foreach (var item in cart.Items)
            {
                var product = await Task.Run(() => _uow.Products.GetById(item.ProductId));
                if (product == null || product.Stock < item.Quantity)
                    throw new InvalidOperationException($"สินค้า {item.ProductName} ไม่เพียงพอ");
                
                product.Stock -= item.Quantity;
                _uow.Products.Update(product);
                
                _uow.Orders.InsertItem(new OrderItem
                {
                    OrderId = order.Id,
                    ProductId = item.ProductId,
                    Quantity = item.Quantity,
                    UnitPrice = item.UnitPrice
                });
            }
            
            _uow.Commit();
        }
        catch
        {
            _uow.Rollback();
            throw;
        }
    }
}
```

---

## Step 385: Database Encryption

```csharp
// ============================================
// SQLCipher - Encrypted SQLite
// ============================================

// NuGet: SQLitePCLRaw.bundle_sqlcipher

public class EncryptedDatabaseService
{
    private SQLiteConnection? _db;
    
    public void Open(string dbPath, string password)
    {
        var connectionString = new SQLiteConnectionString(
            databasePath: dbPath,
            storeDateTimeAsTicks: true,
            key: password);
        
        _db = new SQLiteConnection(connectionString);
    }
    
    public void ChangePassword(string oldPassword, string newPassword)
    {
        if (_db == null) return;
        _db.Execute($"PRAGMA rekey = '{newPassword}';");
    }
    
    public void Export(string exportPath, string exportPassword)
    {
        if (_db == null) return;
        _db.Execute($"ATTACH DATABASE '{exportPath}' AS export KEY '{exportPassword}';");
        _db.Execute("SELECT sqlcipher_export('export');");
        _db.Execute("DETACH DATABASE export;");
    }
    
    // Derive key from biometric/device ID
    public static string DeriveEncryptionKey()
    {
        var deviceId = DeviceInfo.Current.Idiom.ToString() + DeviceInfo.Current.Manufacturer;
        using var sha256 = System.Security.Cryptography.SHA256.Create();
        var hash = sha256.ComputeHash(System.Text.Encoding.UTF8.GetBytes(deviceId));
        return Convert.ToHexString(hash).ToLower();
    }
}
```

---

## Step 386: Database Versioning & Backup

```csharp
// ============================================
// Database Backup & Restore
// ============================================

public class DatabaseBackupService
{
    private readonly string _dbPath;
    private readonly string _backupDir;
    
    public DatabaseBackupService(string dbPath)
    {
        _dbPath = dbPath;
        _backupDir = Path.Combine(FileSystem.AppDataDirectory, "backups");
        Directory.CreateDirectory(_backupDir);
    }
    
    public async Task<string> CreateBackupAsync()
    {
        var timestamp = DateTime.Now.ToString("yyyyMMdd_HHmmss");
        var backupPath = Path.Combine(_backupDir, $"backup_{timestamp}.db");
        
        await Task.Run(() => File.Copy(_dbPath, backupPath));
        
        // Keep only last 5 backups
        var backups = Directory.GetFiles(_backupDir, "backup_*.db")
            .OrderByDescending(f => f)
            .Skip(5)
            .ToList();
        
        foreach (var old in backups) File.Delete(old);
        
        return backupPath;
    }
    
    public async Task RestoreFromBackupAsync(string backupPath)
    {
        if (!File.Exists(backupPath))
            throw new FileNotFoundException("ไม่พบไฟล์ backup");
        
        var tempPath = _dbPath + ".tmp";
        
        // Atomic restore via temp file
        await Task.Run(() =>
        {
            File.Copy(_dbPath, tempPath); // backup current
            try
            {
                File.Copy(backupPath, _dbPath, overwrite: true);
            }
            catch
            {
                // Restore original if failed
                File.Copy(tempPath, _dbPath, overwrite: true);
                throw;
            }
            finally
            {
                File.Delete(tempPath);
            }
        });
    }
    
    public List<BackupInfo> GetBackups()
    {
        return Directory.GetFiles(_backupDir, "backup_*.db")
            .Select(f => new BackupInfo(
                Path: f,
                Name: Path.GetFileNameWithoutExtension(f),
                Size: new FileInfo(f).Length,
                CreatedAt: File.GetCreationTime(f)))
            .OrderByDescending(b => b.CreatedAt)
            .ToList();
    }
}

public record BackupInfo(string Path, string Name, long Size, DateTime CreatedAt);
```

---

## Step 387: Query Builder

```csharp
// ============================================
// Fluent Query Builder
// ============================================

public class SqliteQueryBuilder<T> where T : class, new()
{
    private readonly SQLiteConnection _db;
    private readonly List<string> _whereClauses = new();
    private readonly List<object> _parameters = new();
    private string? _orderBy;
    private bool _orderDesc;
    private int? _limit;
    private int? _offset;
    
    public SqliteQueryBuilder(SQLiteConnection db) => _db = db;
    
    public SqliteQueryBuilder<T> Where(string column, object value)
    {
        _whereClauses.Add($"{column} = ?");
        _parameters.Add(value);
        return this;
    }
    
    public SqliteQueryBuilder<T> WhereLike(string column, string pattern)
    {
        _whereClauses.Add($"{column} LIKE ?");
        _parameters.Add(pattern);
        return this;
    }
    
    public SqliteQueryBuilder<T> WhereIn<TVal>(string column, IEnumerable<TVal> values)
    {
        var list = values.ToList();
        var placeholders = string.Join(",", Enumerable.Repeat("?", list.Count));
        _whereClauses.Add($"{column} IN ({placeholders})");
        _parameters.AddRange(list.Cast<object>());
        return this;
    }
    
    public SqliteQueryBuilder<T> WhereGreaterThan(string column, object value)
    {
        _whereClauses.Add($"{column} > ?");
        _parameters.Add(value);
        return this;
    }
    
    public SqliteQueryBuilder<T> WhereBetween(string column, object min, object max)
    {
        _whereClauses.Add($"{column} BETWEEN ? AND ?");
        _parameters.Add(min);
        _parameters.Add(max);
        return this;
    }
    
    public SqliteQueryBuilder<T> OrderBy(string column, bool descending = false)
    {
        _orderBy = column;
        _orderDesc = descending;
        return this;
    }
    
    public SqliteQueryBuilder<T> Paginate(int page, int size)
    {
        _limit = size;
        _offset = (page - 1) * size;
        return this;
    }
    
    public List<T> ToList()
    {
        var tableName = typeof(T).Name + "s"; // simple convention
        var sql = BuildSql(tableName);
        return _db.Query<T>(sql, _parameters.ToArray());
    }
    
    private string BuildSql(string tableName)
    {
        var sb = new System.Text.StringBuilder($"SELECT * FROM {tableName}");
        
        if (_whereClauses.Count > 0)
            sb.Append(" WHERE ").Append(string.Join(" AND ", _whereClauses));
        
        if (_orderBy != null)
            sb.Append($" ORDER BY {_orderBy} {(_orderDesc ? "DESC" : "ASC")}");
        
        if (_limit.HasValue) sb.Append($" LIMIT {_limit}");
        if (_offset.HasValue) sb.Append($" OFFSET {_offset}");
        
        return sb.ToString();
    }
}

// Usage
public class ProductRepository
{
    private readonly SQLiteConnection _db;
    
    public ProductRepository(SQLiteConnection db) => _db = db;
    
    public List<Product> Search(string? keyword, decimal? minPrice, decimal? maxPrice,
        string? category, int page = 1, int size = 20)
    {
        var query = new SqliteQueryBuilder<Product>(_db);
        
        if (!string.IsNullOrEmpty(keyword))
            query.WhereLike("Name", $"%{keyword}%");
        
        if (minPrice.HasValue && maxPrice.HasValue)
            query.WhereBetween("Price", minPrice.Value, maxPrice.Value);
        else if (minPrice.HasValue)
            query.WhereGreaterThan("Price", minPrice.Value);
        
        if (!string.IsNullOrEmpty(category))
            query.Where("CategoryName", category);
        
        return query
            .OrderBy("CreatedAt", descending: true)
            .Paginate(page, size)
            .ToList();
    }
}
```

---

## Step 388: Bulk Operations

```csharp
// ============================================
// Optimized Bulk Database Operations
// ============================================

public static class BulkOperationExtensions
{
    public static int BulkInsert<T>(this SQLiteConnection db, IEnumerable<T> items)
    {
        var list = items.ToList();
        if (list.Count == 0) return 0;
        
        int inserted = 0;
        db.BeginTransaction();
        try
        {
            foreach (var batch in list.Chunk(500))
            {
                foreach (var item in batch)
                    db.Insert(item);
                inserted += batch.Length;
            }
            db.Commit();
        }
        catch
        {
            db.Rollback();
            throw;
        }
        return inserted;
    }
    
    public static int BulkUpsert<T>(this SQLiteConnection db, IEnumerable<T> items)
    {
        var list = items.ToList();
        if (list.Count == 0) return 0;
        
        int affected = 0;
        db.BeginTransaction();
        try
        {
            foreach (var batch in list.Chunk(500))
            {
                foreach (var item in batch)
                    affected += db.InsertOrReplace(item);
            }
            db.Commit();
        }
        catch
        {
            db.Rollback();
            throw;
        }
        return affected;
    }
    
    public static int BulkDelete<T>(this SQLiteConnection db, 
        string column, IEnumerable<object> values)
    {
        var list = values.ToList();
        if (list.Count == 0) return 0;
        
        int deleted = 0;
        var tableName = typeof(T).Name + "s";
        
        db.BeginTransaction();
        try
        {
            foreach (var batch in list.Chunk(999)) // SQLite max vars
            {
                var placeholders = string.Join(",", Enumerable.Repeat("?", batch.Length));
                deleted += db.Execute(
                    $"DELETE FROM {tableName} WHERE {column} IN ({placeholders})",
                    batch);
            }
            db.Commit();
        }
        catch
        {
            db.Rollback();
            throw;
        }
        return deleted;
    }
}
```

---

## Step 389: Change Tracking

```csharp
// ============================================
// Automatic Change Tracking
// ============================================

public class TrackedEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
    public int? CreatedByUserId { get; set; }
    public int? UpdatedByUserId { get; set; }
}

public class TrackingRepository<T> where T : TrackedEntity, new()
{
    private readonly SQLiteConnection _db;
    private readonly int? _currentUserId;
    
    public TrackingRepository(SQLiteConnection db, int? currentUserId = null)
    {
        _db = db;
        _currentUserId = currentUserId;
    }
    
    public int Insert(T entity)
    {
        entity.CreatedAt = DateTime.UtcNow;
        entity.CreatedByUserId = _currentUserId;
        return _db.Insert(entity);
    }
    
    public int Update(T entity)
    {
        entity.UpdatedAt = DateTime.UtcNow;
        entity.UpdatedByUserId = _currentUserId;
        return _db.Update(entity);
    }
    
    public int SoftDelete(int id)
    {
        var entity = _db.Get<T>(id);
        if (entity == null) return 0;
        
        entity.IsDeleted = true;
        entity.UpdatedAt = DateTime.UtcNow;
        entity.UpdatedByUserId = _currentUserId;
        return _db.Update(entity);
    }
    
    public List<T> GetActive()
        => _db.Table<T>().Where(e => !e.IsDeleted).ToList();
    
    public T? GetById(int id)
        => _db.Table<T>().FirstOrDefault(e => e.Id == id && !e.IsDeleted);
}
```

---

## Step 390: Multi-database Support

```csharp
// ============================================
// Multiple Databases
// ============================================

public class DatabaseContext
{
    private readonly Dictionary<string, SQLiteConnection> _connections = new();
    
    public SQLiteConnection Main => GetOrCreate("main", "main.db");
    public SQLiteConnection Cache => GetOrCreate("cache", "cache.db");
    public SQLiteConnection Analytics => GetOrCreate("analytics", "analytics.db");
    
    private SQLiteConnection GetOrCreate(string name, string fileName)
    {
        if (!_connections.ContainsKey(name))
        {
            var path = Path.Combine(FileSystem.AppDataDirectory, fileName);
            var conn = new SQLiteConnection(path);
            conn.Pragma("journal_mode", "WAL");
            conn.Pragma("cache_size", -64000); // 64MB
            _connections[name] = conn;
        }
        return _connections[name];
    }
    
    // Cross-database query via ATTACH
    public List<ProductWithAnalytics> GetProductsWithAnalytics()
    {
        var analyticsPath = Path.Combine(FileSystem.AppDataDirectory, "analytics.db");
        Main.Execute($"ATTACH DATABASE '{analyticsPath}' AS analytics");
        
        try
        {
            return Main.Query<ProductWithAnalytics>(@"
                SELECT p.*, a.ViewCount, a.PurchaseCount
                FROM Products p
                LEFT JOIN analytics.ProductEvents a ON p.Id = a.ProductId");
        }
        finally
        {
            Main.Execute("DETACH DATABASE analytics");
        }
    }
    
    public void CloseAll()
    {
        foreach (var conn in _connections.Values)
            conn.Close();
        _connections.Clear();
    }
}
```

---

## สรุป Part 39

ใน Part 39 เราได้เรียนรู้:

1. **Migrations** - Version-based schema evolution
2. **Advanced Queries** - FTS5, Pagination, CTEs, Window Functions
3. **Specification Pattern** - Type-safe query composition
4. **Unit of Work** - Transaction management across repos
5. **Database Encryption** - SQLCipher
6. **Backup & Restore** - Atomic backup system
7. **Query Builder** - Fluent SQL builder
8. **Bulk Operations** - Efficient batch insert/delete
9. **Change Tracking** - Auto timestamps, soft delete
10. **Multi-database** - ATTACH for cross-db queries

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 39 | Steps 381-390*
