# 8. Performance

## 8.1 Span\<T\> and Memory\<T\>

`Span<T>` is a stack-allocated view over contiguous memory — no heap allocation, zero-copy slicing.

```csharp
// Parse a CSV line without allocating substrings
public static IEnumerable<string> ParseCsvLine(string line)
{
    var span = line.AsSpan();
    int start = 0;

    for (int i = 0; i < span.Length; i++)
    {
        if (span[i] == ',')
        {
            yield return span[start..i].ToString(); // only allocates on yield
            start = i + 1;
        }
    }
    yield return span[start..].ToString();
}

// Stack-allocated buffer for small operations
Span<byte> buffer = stackalloc byte[256]; // no heap allocation
int written = Encoding.UTF8.GetBytes("Hello, World!", buffer);
var result  = buffer[..written];

// Memory<T> — heap version of Span (can be stored, used in async)
public async Task ProcessAsync(Memory<byte> data, CancellationToken ct)
{
    await stream.WriteAsync(data, ct); // Memory<T> works with async
}

// ArrayPool — reuse arrays instead of allocating new ones
var pool   = ArrayPool<byte>.Shared;
var buffer = pool.Rent(4096); // may return larger array
try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    Process(buffer.AsSpan(0, read));
}
finally
{
    pool.Return(buffer); // return to pool
}
```

---

## 8.2 String performance

```csharp
// Avoid string concatenation in loops — O(n²)
// BAD
string result = "";
foreach (var item in items)
    result += item + ",";  // allocates a new string each time

// GOOD — StringBuilder is O(n)
var sb = new StringBuilder();
foreach (var item in items)
    sb.Append(item).Append(',');
string result = sb.ToString();

// BEST for known patterns — string.Join
string result = string.Join(",", items);

// String.Create — allocate once, write into it
public static string FormatId(int prefix, Guid id)
    => string.Create(40, (prefix, id), (span, state) =>
    {
        state.prefix.TryFormat(span, out var written);
        span[written] = '-';
        state.id.TryFormat(span[(written + 1)..], out _);
    });

// Interpolated string handlers (C# 10+) — no intermediate allocations
// These are already optimised by the compiler for Console.WriteLine and logging
logger.LogInformation("Processing order {OrderId} for customer {CustomerId}", orderId, customerId);
// The string is NOT formatted unless the log level is enabled
```

---

## 8.3 Avoiding allocations

```csharp
// Use structs for small, short-lived data
public readonly struct Point(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;
    public double DistanceTo(Point other)
        => Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}

// Use record structs (C# 10+) — value semantics + equality
public record struct Money(decimal Amount, string Currency);

// Avoid LINQ in hot paths — use for loops
// Allocation-heavy:
var total = orders.Where(o => o.IsActive).Sum(o => o.Total);

// Zero-allocation:
decimal total = 0;
foreach (var o in orders)
    if (o.IsActive) total += o.Total;

// Use IEnumerable<T> return types for deferred execution
// Materialise only when needed
public IEnumerable<Product> GetCheap(decimal maxPrice)
{
    foreach (var p in _products)
        if (p.Price <= maxPrice)
            yield return p; // no intermediate list allocated
}
```

---

## 8.4 EF Core query optimisation

```csharp
// Use AsNoTracking for read-only queries
var products = await db.Products.AsNoTracking().ToListAsync(ct);

// Project to DTOs — only SELECT needed columns
var summaries = await db.Orders
    .AsNoTracking()
    .Select(o => new OrderSummary(o.Id, o.Total, o.Status))
    .ToListAsync(ct);

// Avoid N+1 queries — always use Include
// BAD: N+1
var orders = await db.Orders.ToListAsync(ct);
foreach (var order in orders)
    var customer = await db.Customers.FindAsync(order.CustomerId, ct); // N extra queries

// GOOD: single JOIN
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync(ct);

// Split queries for large collections
var orders = await db.Orders
    .Include(o => o.Lines)
    .AsSplitQuery()         // avoids Cartesian explosion
    .ToListAsync(ct);

// Compiled queries — avoid query translation overhead on hot paths
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Products.FirstOrDefault(p => p.Id == id));

// Usage
var product = await GetProductById(db, id);

// Bulk operations (EF Core 7+) — no entity tracking
await db.Products
    .Where(p => p.Category == "Discontinued")
    .ExecuteDeleteAsync(ct);

await db.Products
    .Where(p => p.Stock == 0)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsAvailable, false), ct);
```

---

## 8.5 Caching strategies

```csharp
// In-memory cache (single instance only)
builder.Services.AddMemoryCache();

public class ProductCache(IMemoryCache cache, IProductRepository repo)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        var key = $"product:{id}";
        if (cache.TryGetValue(key, out Product? product))
            return product;

        product = await repo.GetByIdAsync(id, ct);
        if (product is not null)
        {
            cache.Set(key, product, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration               = TimeSpan.FromMinutes(2),
                Size                            = 1  // if SizeLimit is set
            });
        }
        return product;
    }
}

// Output caching (ASP.NET Core 7+) — cache entire HTTP responses
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(b => b.Cache().Expire(TimeSpan.FromSeconds(30)));

    opts.AddPolicy("Products", b =>
        b.Cache()
         .Expire(TimeSpan.FromMinutes(5))
         .VaryByQuery("category", "page"));
});

app.UseOutputCache();

app.MapGet("/products", GetProducts).CacheOutput("Products");
app.MapGet("/products/{id}", GetProduct).CacheOutput(b => b.Expire(TimeSpan.FromMinutes(10)));

// Invalidate output cache by tag
app.MapPost("/products", async (CreateProductRequest req, IOutputCacheStore store, CancellationToken ct) =>
{
    // ... create product
    await store.EvictByTagAsync("Products", ct);
});
```

---

## 8.6 Benchmarking with BenchmarkDotNet

```csharp
// dotnet add package BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class StringBenchmarks
{
    private const int N = 1000;
    private readonly string[] _items = Enumerable.Range(0, N).Select(i => $"item{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        string result = "";
        foreach (var item in _items)
            result += item;
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in _items)
            sb.Append(item);
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin() => string.Join("", _items);
}

// Run
// dotnet run -c Release
BenchmarkRunner.Run<StringBenchmarks>();
```