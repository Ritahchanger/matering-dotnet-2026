# 4. Data & Persistence

## 4.1 Entity Framework Core 9

EF Core is the primary ORM for .NET backends. It maps C# classes to database tables and generates SQL automatically.

### DbContext setup

```csharp
// AppDbContext.cs
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order>   Orders   => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Apply all IEntityTypeConfiguration<T> classes in the assembly
        builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }

    // Intercept SaveChanges to auto-set timestamps
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = DateTime.UtcNow;
            if (entry.State is EntityState.Added or EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return await base.SaveChangesAsync(ct);
    }
}

// Register
builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseNpgsql(
        builder.Configuration.GetConnectionString("Default"),
        npgsql => npgsql.CommandTimeout(30)));
```

### Entity configuration

```csharp
// Prefer IEntityTypeConfiguration over data annotations
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("products");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(p => p.Price)
            .HasPrecision(18, 2)
            .IsRequired();

        builder.Property(p => p.Category)
            .HasConversion<string>()  // store enum as string
            .HasMaxLength(50);

        // Owned type (value object stored in same table)
        builder.OwnsOne(p => p.Dimensions, d =>
        {
            d.Property(x => x.Width).HasColumnName("width_cm");
            d.Property(x => x.Height).HasColumnName("height_cm");
        });

        // Index
        builder.HasIndex(p => p.Sku).IsUnique();
        builder.HasIndex(p => new { p.Category, p.Price });

        // Relationships
        builder.HasMany(p => p.OrderLines)
            .WithOne(l => l.Product)
            .HasForeignKey(l => l.ProductId)
            .OnDelete(DeleteBehavior.Restrict);

        // Row-level soft delete filter
        builder.HasQueryFilter(p => !p.IsDeleted);
    }
}
```

### Querying

```csharp
public class ProductRepository(AppDbContext db) : IProductRepository
{
    // Basic find
    public Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
        => db.Products.FindAsync([id], ct).AsTask();

    // Query with includes
    public Task<Product?> GetWithDetailsAsync(int id, CancellationToken ct = default)
        => db.Products
            .Include(p => p.OrderLines)
                .ThenInclude(l => l.Order)
            .FirstOrDefaultAsync(p => p.Id == id, ct);

    // Filtered, sorted, paged — stays as IQueryable (DB-side)
    public async Task<PagedResult<Product>> SearchAsync(
        ProductFilter filter, CancellationToken ct = default)
    {
        var query = db.Products.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(filter.Search))
            query = query.Where(p => p.Name.Contains(filter.Search));

        if (filter.Category is not null)
            query = query.Where(p => p.Category == filter.Category);

        if (filter.MinPrice.HasValue)
            query = query.Where(p => p.Price >= filter.MinPrice.Value);

        var total = await query.CountAsync(ct);

        var items = await query
            .OrderBy(p => p.Name)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync(ct);

        return new PagedResult<Product>(items, total, filter.Page, filter.PageSize);
    }

    // Projection — only fetch needed columns
    public Task<List<ProductSummary>> GetSummariesAsync(CancellationToken ct = default)
        => db.Products
            .AsNoTracking()
            .Select(p => new ProductSummary(p.Id, p.Name, p.Price, p.Category))
            .ToListAsync(ct);

    // Async stream for bulk processing
    public IAsyncEnumerable<Product> StreamAllAsync(CancellationToken ct = default)
        => db.Products.AsNoTracking().AsAsyncEnumerable();

    // Write
    public async Task<Product> AddAsync(Product product, CancellationToken ct = default)
    {
        db.Products.Add(product);
        await db.SaveChangesAsync(ct);
        return product;
    }

    public Task SaveChangesAsync(CancellationToken ct = default)
        => db.SaveChangesAsync(ct);
}
```

### Migrations

```bash
# Create a migration
dotnet ef migrations add AddProductSkuIndex --project src/Infrastructure --startup-project src/Api

# Apply to database
dotnet ef database update --project src/Infrastructure --startup-project src/Api

# Generate SQL script (for production)
dotnet ef migrations script --idempotent -o migrations.sql

# Rollback
dotnet ef database update PreviousMigrationName
```

```csharp
// Auto-apply migrations at startup (development only)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
```

---

## 4.2 Dapper (micro-ORM)

Use Dapper for complex queries, stored procedures, or performance-critical reads.

```csharp
// Install: dotnet add package Dapper Npgsql

public class OrderReadRepository(IDbConnectionFactory connectionFactory)
{
    // Simple query
    public async Task<IReadOnlyList<OrderSummary>> GetRecentAsync(
        int customerId, CancellationToken ct = default)
    {
        await using var conn = await connectionFactory.OpenAsync(ct);

        var sql = """
            SELECT o.id, o.total, o.status, o.created_at,
                   COUNT(ol.id) AS line_count
            FROM orders o
            JOIN order_lines ol ON ol.order_id = o.id
            WHERE o.customer_id = @CustomerId
              AND o.created_at > NOW() - INTERVAL '30 days'
            GROUP BY o.id
            ORDER BY o.created_at DESC
            LIMIT 50
            """;

        var result = await conn.QueryAsync<OrderSummary>(
            sql,
            new { CustomerId = customerId });

        return result.AsList();
    }

    // Multi-mapping (JOIN into multiple objects)
    public async Task<IReadOnlyList<Order>> GetWithLinesAsync(int customerId)
    {
        await using var conn = await connectionFactory.OpenAsync();

        var sql = """
            SELECT o.*, ol.*
            FROM orders o
            JOIN order_lines ol ON ol.order_id = o.id
            WHERE o.customer_id = @CustomerId
            """;

        var orderDict = new Dictionary<int, Order>();

        await conn.QueryAsync<Order, OrderLine, Order>(
            sql,
            (order, line) =>
            {
                if (!orderDict.TryGetValue(order.Id, out var existing))
                    orderDict[order.Id] = existing = order;
                existing.Lines.Add(line);
                return existing;
            },
            new { CustomerId = customerId },
            splitOn: "order_line_id");

        return orderDict.Values.ToList();
    }

    // Stored procedure
    public async Task<SalesReport> GetSalesReportAsync(DateTime from, DateTime to)
    {
        await using var conn = await connectionFactory.OpenAsync();
        return await conn.QuerySingleAsync<SalesReport>(
            "get_sales_report",
            new { From = from, To = to },
            commandType: CommandType.StoredProcedure);
    }

    // Bulk insert
    public async Task BulkInsertAsync(IEnumerable<Product> products)
    {
        await using var conn = await connectionFactory.OpenAsync();
        await conn.ExecuteAsync(
            "INSERT INTO products (name, price) VALUES (@Name, @Price)",
            products);
    }
}

// Connection factory
public class NpgsqlConnectionFactory(string connectionString) : IDbConnectionFactory
{
    public async Task<IDbConnection> OpenAsync(CancellationToken ct = default)
    {
        var conn = new NpgsqlConnection(connectionString);
        await conn.OpenAsync(ct);
        return conn;
    }
}
```

---

## 4.3 Redis caching

```csharp
// Install: dotnet add package StackExchange.Redis Microsoft.Extensions.Caching.StackExchangeRedis

// Register
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = builder.Configuration.GetConnectionString("Redis");
    opts.InstanceName = "myapp:";
});

// Generic cache service
public class CacheService(IDistributedCache cache) : ICacheService
{
    private static readonly JsonSerializerOptions JsonOpts = new(JsonSerializerDefaults.Web);

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var bytes = await cache.GetAsync(key, ct);
        return bytes is null
            ? default
            : JsonSerializer.Deserialize<T>(bytes, JsonOpts);
    }

    public async Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? absoluteExpiry = null,
        TimeSpan? slidingExpiry = null,
        CancellationToken ct = default)
    {
        var opts = new DistributedCacheEntryOptions();
        if (absoluteExpiry.HasValue) opts.AbsoluteExpirationRelativeToNow = absoluteExpiry;
        if (slidingExpiry.HasValue)  opts.SlidingExpiration = slidingExpiry;

        var bytes = JsonSerializer.SerializeToUtf8Bytes(value, JsonOpts);
        await cache.SetAsync(key, bytes, opts, ct);
    }

    public Task RemoveAsync(string key, CancellationToken ct = default)
        => cache.RemoveAsync(key, ct);

    // Cache-aside pattern
    public async Task<T> GetOrSetAsync<T>(
        string key,
        Func<CancellationToken, Task<T>> factory,
        TimeSpan expiry,
        CancellationToken ct = default)
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await factory(ct);
        await SetAsync(key, value, absoluteExpiry: expiry, ct: ct);
        return value;
    }
}

// Usage
public class ProductService(IProductRepository repo, ICacheService cache)
{
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var cacheKey = $"product:{id}";
        return await cache.GetOrSetAsync(
            cacheKey,
            _ => repo.GetByIdAsync(id, ct),
            expiry: TimeSpan.FromMinutes(10),
            ct: ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        await repo.UpdateAsync(product, ct);
        await cache.RemoveAsync($"product:{product.Id}", ct); // invalidate
    }
}
```

---

## 4.4 Transactions

```csharp
// EF Core transaction (single DbContext)
public async Task TransferAsync(Guid fromId, Guid toId, decimal amount, CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    try
    {
        var from = await db.Accounts.FindAsync([fromId], ct)
            ?? throw new NotFoundException(fromId);
        var to = await db.Accounts.FindAsync([toId], ct)
            ?? throw new NotFoundException(toId);

        from.Debit(amount);
        to.Credit(amount);

        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
    catch
    {
        await tx.RollbackAsync(ct);
        throw;
    }
}

// Shared transaction between EF Core and Dapper
public async Task ComplexOperationAsync(CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    var conn = db.Database.GetDbConnection();

    // EF Core write
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);

    // Dapper read using same connection and transaction
    var stats = await conn.QuerySingleAsync<Stats>(
        "SELECT * FROM order_stats WHERE id = @Id",
        new { order.Id },
        transaction: tx.GetDbTransaction());

    await tx.CommitAsync(ct);
}
```

---

## 4.5 Repository pattern

```csharp
// Generic base repository
public abstract class Repository<T, TId>(AppDbContext db)
    where T : class, IEntity<TId>
{
    protected AppDbContext Db => db;

    public ValueTask<T?> GetByIdAsync(TId id, CancellationToken ct = default)
        => db.Set<T>().FindAsync([id!], ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await db.Set<T>().AsNoTracking().ToListAsync(ct);

    public void Add(T entity) => db.Set<T>().Add(entity);
    public void Update(T entity) => db.Set<T>().Update(entity);
    public void Remove(T entity) => db.Set<T>().Remove(entity);

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => db.SaveChangesAsync(ct);
}

// Concrete repository
public class ProductRepository(AppDbContext db)
    : Repository<Product, int>(db), IProductRepository
{
    public Task<List<Product>> GetByCategoryAsync(string category, CancellationToken ct = default)
        => Db.Products
            .AsNoTracking()
            .Where(p => p.Category == category)
            .OrderBy(p => p.Name)
            .ToListAsync(ct);
}
```