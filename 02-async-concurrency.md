# 2. Async & Concurrency

## 2.1 The async/await model

C# async/await is built on the `Task`-based Asynchronous Pattern (TAP). The compiler transforms `async` methods into state machines — no threads are blocked while waiting for I/O.

```csharp
// Synchronous (blocks thread)
public string FetchData(string url)
{
    var response = httpClient.GetString(url); // blocks
    return response;
}

// Asynchronous (releases thread while waiting)
public async Task<string> FetchDataAsync(string url, CancellationToken ct = default)
{
    var response = await httpClient.GetStringAsync(url, ct); // releases thread
    return response;
}
```

### Rules you must follow

```csharp
// 1. Never use .Result or .Wait() — causes deadlocks and blocks threads
var data = service.GetAsync().Result;    // WRONG
var data = await service.GetAsync();     // RIGHT

// 2. async void is forbidden except for event handlers
public async void DoWork() { }          // WRONG — exceptions are unobservable
public async Task DoWorkAsync() { }     // RIGHT

// 3. Always accept CancellationToken
public async Task<User> GetUserAsync(int id)                           // ok but incomplete
public async Task<User> GetUserAsync(int id, CancellationToken ct = default)  // RIGHT

// 4. Propagate ct to every awaited call
public async Task ProcessAsync(CancellationToken ct = default)
{
    var data = await repo.GetAsync(ct);         // pass ct
    await processor.RunAsync(data, ct);         // pass ct
    await db.SaveChangesAsync(ct);              // pass ct
}
```

---

## 2.2 Task and ValueTask

### Task\<T\>

```csharp
// Returns a Task — always allocates a heap object
public async Task<string> GetNameAsync(int id, CancellationToken ct = default)
{
    var user = await db.Users.FindAsync([id], ct);
    return user?.Name ?? throw new NotFoundException(id);
}

// Task.FromResult — already-completed task (no async overhead)
public Task<int> GetConstantAsync() => Task.FromResult(42);

// Task.CompletedTask — void-returning already-completed
public Task NoOpAsync() => Task.CompletedTask;
```

### ValueTask\<T\>

Use when the result is frequently available synchronously (cache hits, in-memory data):

```csharp
public ValueTask<User?> GetUserAsync(int id, CancellationToken ct = default)
{
    // Hot path: return synchronously with zero allocation
    if (_cache.TryGetValue(id, out var cached))
        return ValueTask.FromResult<User?>(cached);

    // Cold path: async database call
    return new ValueTask<User?>(FetchFromDbAsync(id, ct));
}

private async Task<User?> FetchFromDbAsync(int id, CancellationToken ct)
{
    var user = await db.Users.FindAsync([id], ct);
    if (user is not null) _cache[id] = user;
    return user;
}
```

---

## 2.3 Concurrent operations

### Task.WhenAll — run all, wait for all

```csharp
public async Task<DashboardData> GetDashboardAsync(int userId, CancellationToken ct = default)
{
    // These three calls run concurrently — not sequentially
    var ordersTask  = orderRepo.GetByUserAsync(userId, ct);
    var profileTask = userRepo.GetProfileAsync(userId, ct);
    var statsTask   = analyticsService.GetStatsAsync(userId, ct);

    await Task.WhenAll(ordersTask, profileTask, statsTask);

    return new DashboardData(
        Orders:  ordersTask.Result,
        Profile: profileTask.Result,
        Stats:   statsTask.Result
    );
}
```

### Task.WhenAny — first one wins

```csharp
// Useful for timeouts or racing two data sources
var fastest = await Task.WhenAny(
    FetchFromPrimaryAsync(ct),
    FetchFromSecondaryAsync(ct)
);
var result = await fastest; // unwrap the winner
```

### Parallel.ForEachAsync — bounded async parallelism

```csharp
// Process items in parallel with a max degree of parallelism
await Parallel.ForEachAsync(
    productIds,
    new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
    async (id, token) =>
    {
        var product = await productService.GetAsync(id, token);
        await indexService.IndexAsync(product, token);
    });
```

---

## 2.4 CancellationToken

Every async method should accept and propagate a `CancellationToken`. This lets callers (HTTP requests, background jobs) cancel in-flight work.

```csharp
// ASP.NET Core automatically cancels when the client disconnects
app.MapGet("/slow", async (CancellationToken ct) =>
{
    var data = await heavyService.ComputeAsync(ct);
    return Results.Ok(data);
});

// Creating your own token with a timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    var result = await service.GetAsync(cts.Token);
}
catch (OperationCanceledException)
{
    logger.LogWarning("Operation timed out after 5 seconds");
}

// Linked tokens — cancel if either fires
using var linked = CancellationTokenSource.CreateLinkedTokenSource(
    requestCancellationToken,
    timeoutToken
);
```

---

## 2.5 Async streams (IAsyncEnumerable\<T\>)

Use for streaming large datasets without loading everything into memory:

```csharp
// Producer
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    DateTime since,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var batch in db.Orders
        .Where(o => o.CreatedAt > since)
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return batch;
    }
}

// Consumer
await foreach (var order in service.StreamOrdersAsync(since, ct))
{
    await processor.HandleAsync(order, ct);
}

// Minimal API streaming response (SSE / NDJSON)
app.MapGet("/stream", (OrderService svc, CancellationToken ct) =>
    svc.StreamOrdersAsync(DateTime.UtcNow.AddDays(-7), ct));
```

---

## 2.6 Channels

`System.Threading.Channels` is the modern producer-consumer primitive — faster than `BlockingCollection<T>` and fully async:

```csharp
// Bounded channel — backpressure when full
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// Producer
public async Task ProduceAsync(CancellationToken ct)
{
    await foreach (var item in GetItemsAsync(ct))
    {
        await channel.Writer.WriteAsync(item, ct);
    }
    channel.Writer.Complete();
}

// Consumer
public async Task ConsumeAsync(CancellationToken ct)
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item, ct);
    }
}
```

---

## 2.7 Thread safety

```csharp
// lock — mutual exclusion (use System.Threading.Lock in C# 13)
private readonly Lock _lock = new();

public void Increment()
{
    using (_lock.EnterScope())
        _count++;
}

// Interlocked — atomic operations without lock
private int _count = 0;
Interlocked.Increment(ref _count);
Interlocked.Add(ref _total, amount);
var snapshot = Interlocked.Read(ref _count);

// SemaphoreSlim — async-compatible mutual exclusion
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<T> GetWithLockAsync<T>(Func<Task<T>> operation, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        return await operation();
    }
    finally
    {
        _semaphore.Release();
    }
}

// ReaderWriterLockSlim — many readers, one writer
private readonly ReaderWriterLockSlim _rwLock = new();

public string Read()
{
    _rwLock.EnterReadLock();
    try { return _data; }
    finally { _rwLock.ExitReadLock(); }
}

public void Write(string data)
{
    _rwLock.EnterWriteLock();
    try { _data = data; }
    finally { _rwLock.ExitWriteLock(); }
}

// ConcurrentDictionary — lock-free concurrent reads/writes
private readonly ConcurrentDictionary<int, User> _cache = new();
_cache.TryAdd(id, user);
_cache.GetOrAdd(id, _ => FetchUser(id));
_cache.AddOrUpdate(id, user, (_, _) => user);
```

---

## 2.8 Background services

```csharp
// IHostedService — runs for the lifetime of the app
public class OrderExpiryService(IServiceScopeFactory scopeFactory, ILogger<OrderExpiryService> logger)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Order expiry service started");

        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }

    private async Task DoWorkAsync(CancellationToken ct)
    {
        // Must create a scope — background services are singletons
        // but DbContext/repositories are scoped
        using var scope = scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();

        var expiredOrders = await repo.GetExpiredAsync(ct);
        foreach (var order in expiredOrders)
        {
            order.Cancel("Expired");
            logger.LogInformation("Cancelled expired order {OrderId}", order.Id);
        }
        await repo.SaveChangesAsync(ct);
    }
}

// Register
builder.Services.AddHostedService<OrderExpiryService>();
```