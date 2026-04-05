# 1. Language Core

## 1.1 Type system

C# is a statically typed language. Every variable has a compile-time type.

### Value types vs reference types

| Category | Examples | Stored on | Null by default |
|----------|----------|-----------|-----------------|
| Value types | `int`, `bool`, `double`, `struct`, `enum` | Stack (usually) | No |
| Reference types | `class`, `string`, `array`, `record class` | Heap | Yes (without NRT) |
| Nullable value | `int?`, `bool?`, `DateTime?` | Stack + flag | Yes |

```csharp
int a = 42;           // value type — copy on assignment
int b = a;
b = 100;              // a is still 42

string s1 = "hello";  // reference type — points to heap object
string s2 = s1;       // both point to same object (strings are immutable so safe)
```

### Nullable reference types (NRT)

Enable in your `.csproj` — it is the default in .NET 6+:

```xml
<Nullable>enable</Nullable>
```

With NRT enabled, the compiler warns you about potential null dereferences:

```csharp
string name = null;    // warning: cannot assign null to non-nullable string
string? name = null;   // ok — explicitly nullable

void Greet(string? name)
{
    Console.WriteLine(name.Length);         // warning: possible null dereference
    Console.WriteLine(name?.Length ?? 0);   // safe
    if (name is not null)
        Console.WriteLine(name.Length);     // safe — compiler tracks nullability
}
```

---

## 1.2 Built-in types

```csharp
// Integers
byte    b  = 255;          // 0..255
short   s  = -32768;       // 16-bit signed
int     i  = 2_147_483_647;// 32-bit signed (most common)
long    l  = 9_000_000_000L;// 64-bit signed
nint    n  = 0;            // native int (platform-sized)

// Floating point
float   f  = 3.14f;        // 32-bit, ~7 digits precision
double  d  = 3.14159265;   // 64-bit, ~15 digits precision
decimal m  = 19.99m;       // 128-bit, exact — use for money

// Other
bool    ok = true;
char    c  = 'A';          // UTF-16 code unit
string  str = "hello";     // immutable sequence of chars

// Special
object  obj = anything;    // base type of everything
dynamic dyn = anything;    // resolved at runtime (avoid in backend code)
```

### String operations

```csharp
// Interpolation (preferred)
var msg = $"Hello, {name}! You have {count} messages.";

// Raw string literals (no escaping needed)
var json = """
    {
      "key": "value",
      "path": "C:\Users\alice"
    }
    """;

// Verbatim strings
var path = @"C:\Users\alice\Documents";

// Common methods
str.ToUpper()
str.Trim()
str.Split(',')
str.Contains("word")
str.StartsWith("prefix")
str.Replace("old", "new")
str.Substring(0, 5)        // prefer str[0..5] (range syntax)
str[^1]                    // last character
str[2..5]                  // slice

// StringBuilder for heavy concatenation
var sb = new StringBuilder();
sb.Append("Hello");
sb.AppendLine(", World!");
var result = sb.ToString();
```

---

## 1.3 Variables, constants, and inference

```csharp
// Type inference — compiler determines type
var count = 42;           // int
var name  = "Alice";      // string
var items = new List<int>();

// Constants — compile-time values
const int MaxRetries = 3;
const string DefaultRole = "user";

// Read-only fields — set once (runtime)
private readonly ILogger _logger;

// Static read-only
public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
```

---

## 1.4 Operators

```csharp
// Null operators
var x = value ?? "default";           // if value is null, use "default"
var y = obj?.Property;                // null if obj is null
obj?.Method();                        // no-op if obj is null
var z = obj?.Prop ?? throw new InvalidOperationException();

// Null-coalescing assignment
list ??= new List<string>();          // assign only if null

// Pattern matching operators
if (obj is string s)        { /* s is string */ }
if (obj is not null)        { /* obj is not null */ }
if (n is > 0 and < 100)     { /* range check */ }

// Range and index
var last  = arr[^1];         // last element
var slice = arr[1..4];       // elements 1, 2, 3
var rest  = arr[2..];        // from index 2 to end
```

---

## 1.5 Control flow

```csharp
// If / else
if (condition) { } else if (other) { } else { }

// Switch statement
switch (status)
{
    case "active":
        break;
    case "inactive":
    case "disabled":
        break;
    default:
        break;
}

// Switch expression (preferred — exhaustive, returns value)
var label = status switch
{
    "active"   => "Active user",
    "inactive" => "Inactive user",
    null       => throw new ArgumentNullException(nameof(status)),
    _          => $"Unknown: {status}"
};

// For loops
for (int i = 0; i < 10; i++) { }
foreach (var item in collection) { }
while (condition) { }
do { } while (condition);

// Loop control
break;     // exit loop
continue;  // skip to next iteration
return;    // exit method
```

---

## 1.6 Pattern matching

Pattern matching is one of C#'s most powerful features for backend code.

```csharp
// Type patterns
object obj = GetSomething();
if (obj is int n)
    Console.WriteLine($"Integer: {n}");

// Property patterns
if (user is { IsActive: true, Role: "Admin" })
    GrantAccess();

// Nested property patterns
if (order is { Customer: { Country: "KE" }, Total: > 1000 })
    ApplyDiscount();

// Positional patterns (for records/deconstructable types)
if (point is (0, 0))
    Console.WriteLine("Origin");

// List patterns (C# 11+)
int[] arr = [1, 2, 3];
if (arr is [1, ..])          Console.WriteLine("Starts with 1");
if (arr is [_, _, _])        Console.WriteLine("Exactly 3 elements");
if (arr is [var first, ..])  Console.WriteLine($"First: {first}");

// Switch with patterns
string Describe(object obj) => obj switch
{
    int n when n < 0         => "negative int",
    int n                    => $"int: {n}",
    string { Length: 0 }     => "empty string",
    string s                 => $"string: {s}",
    IEnumerable<int> { } col => $"collection of {col.Count()} ints",
    null                     => "null",
    _                        => "something else"
};
```

---

## 1.7 Classes

```csharp
// Full class
public class Order
{
    // Fields (private by convention)
    private readonly List<OrderLine> _lines = [];

    // Properties
    public Guid Id { get; private set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;

    // Read-only computed property
    public decimal Total => _lines.Sum(l => l.LineTotal);
    public IReadOnlyList<OrderLine> Lines => _lines;

    // Constructor
    public Order(CustomerId customerId)
    {
        CustomerId = customerId;
    }

    // Private EF constructor
    private Order() { }

    public CustomerId CustomerId { get; private set; }

    // Methods
    public void AddLine(Product product, int quantity)
    {
        ArgumentNullException.ThrowIfNull(product);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);

        _lines.Add(new OrderLine(product, quantity));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot confirm an order in {Status} status.");
        Status = OrderStatus.Confirmed;
    }
}
```

### Primary constructors (C# 12+)

```csharp
// Parameters are in scope throughout the class body
public class ProductService(IProductRepository repo, ILogger<ProductService> logger)
{
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        logger.LogInformation("Fetching product {Id}", id);
        return await repo.GetByIdAsync(id, ct);
    }
}
```

### Static classes and methods

```csharp
public static class Guard
{
    public static T NotNull<T>(T? value, string paramName) where T : class
        => value ?? throw new ArgumentNullException(paramName);

    public static string NotEmpty(string? value, string paramName)
        => string.IsNullOrWhiteSpace(value)
            ? throw new ArgumentException("Must not be empty.", paramName)
            : value;
}
```

---

## 1.8 Interfaces

```csharp
// Define the contract
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task SaveChangesAsync(CancellationToken ct = default);
}

// Default interface methods (use sparingly)
public interface IHasTimestamps
{
    DateTime CreatedAt { get; }
    DateTime? UpdatedAt { get; }

    bool IsRecent() => CreatedAt > DateTime.UtcNow.AddDays(-7);
}

// Implement
public class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default)
        => await db.Orders.Where(o => o.CustomerId == customerId).ToListAsync(ct);

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await db.Orders.AddAsync(order, ct);

    public Task SaveChangesAsync(CancellationToken ct = default)
        => db.SaveChangesAsync(ct);
}
```

---

## 1.9 Records

Records are ideal for DTOs, value objects, and immutable data carriers.

```csharp
// Positional record class (reference type, immutable by default)
public record CreateProductRequest(string Name, decimal Price, string Category);

// Positional record struct (value type — zero allocation for small data)
public record struct Coordinates(double Lat, double Lng);

// Record with extra members
public record Product(int Id, string Name, decimal Price)
{
    // Computed property
    public string DisplayPrice => $"${Price:F2}";

    // Validation in constructor
    public Product : this(Id, Name, Price)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(Name);
        ArgumentOutOfRangeException.ThrowIfNegative(Price);
    }
}

// Non-destructive mutation
var original = new Product(1, "Laptop", 999.99m);
var updated  = original with { Price = 1099.99m };

// Records have value-based equality
var a = new Coordinates(1.0, 2.0);
var b = new Coordinates(1.0, 2.0);
Console.WriteLine(a == b); // true
```

---

## 1.10 Generics

```csharp
// Generic class
public class Repository<T> where T : class, IEntity
{
    protected readonly DbContext _db;
    protected DbSet<T> Set => _db.Set<T>();

    public Repository(DbContext db) => _db = db;

    public Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => Set.FindAsync([id], ct).AsTask();

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await Set.AsNoTracking().ToListAsync(ct);
}

// Generic method
public static T Clamp<T>(T value, T min, T max) where T : IComparable<T>
{
    if (value.CompareTo(min) < 0) return min;
    if (value.CompareTo(max) > 0) return max;
    return value;
}

// Generic constraints
where T : class              // reference type
where T : struct             // value type
where T : new()              // has parameterless constructor
where T : IEntity            // implements interface
where T : BaseEntity         // inherits from class
where T : class, IEntity, new() // combine constraints
where T : notnull            // non-nullable
```

---

## 1.11 Collections

```csharp
// Array (fixed size, fast indexed access)
int[] nums = [1, 2, 3, 4, 5];
int[] zeros = new int[10];

// List<T> (dynamic size, indexed access)
var list = new List<string> { "a", "b", "c" };
list.Add("d");
list.Remove("a");
list.AddRange(["e", "f"]);

// Dictionary<TKey, TValue>
var dict = new Dictionary<string, int>
{
    ["alice"] = 1,
    ["bob"]   = 2
};
dict.TryGetValue("alice", out int id);
dict.GetValueOrDefault("charlie", 0);

// HashSet<T> (unique values, O(1) lookup)
var set = new HashSet<string> { "a", "b", "c" };
set.Add("d");
set.Contains("a"); // true

// Queue<T> and Stack<T>
var queue = new Queue<string>();
queue.Enqueue("first");
var item = queue.Dequeue();

var stack = new Stack<int>();
stack.Push(1);
var top = stack.Pop();

// ImmutableList (thread-safe reads, creates new on mutation)
using System.Collections.Immutable;
var immutable = ImmutableList.Create(1, 2, 3);
var withFour = immutable.Add(4); // original unchanged

// Collection expressions (C# 12+)
int[]        arr  = [1, 2, 3];
List<string> lst  = ["a", "b"];
int[]        both = [..arr, 4, 5];  // spread
```

---

## 1.12 LINQ

LINQ (Language Integrated Query) is central to C# backend development.

```csharp
var orders = GetOrders(); // IEnumerable<Order> or IQueryable<Order>

// Filtering
var active = orders.Where(o => o.Status == OrderStatus.Active);

// Projection
var summaries = orders.Select(o => new { o.Id, o.Total, o.CreatedAt });

// Ordering
var sorted = orders.OrderByDescending(o => o.CreatedAt)
                   .ThenBy(o => o.Total);

// Grouping
var byStatus = orders
    .GroupBy(o => o.Status)
    .ToDictionary(g => g.Key, g => g.ToList());

// Aggregation
var total    = orders.Sum(o => o.Total);
var average  = orders.Average(o => o.Total);
var count    = orders.Count(o => o.Total > 100);
var max      = orders.Max(o => o.Total);

// Existence checks
bool anyPending  = orders.Any(o => o.Status == OrderStatus.Pending);
bool allShipped  = orders.All(o => o.Status == OrderStatus.Shipped);

// Single-element retrieval
var order = orders.First(o => o.Id == id);            // throws if none
var order = orders.FirstOrDefault(o => o.Id == id);   // null if none
var order = orders.Single(o => o.Id == id);           // throws if != 1
var order = orders.SingleOrDefault(o => o.Id == id);  // null or throws if > 1

// Joining
var result = orders.Join(
    customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { Order = o, Customer = c });

// Flattening
var allLines = orders.SelectMany(o => o.Lines);

// Chaining
var report = orders
    .Where(o => o.CreatedAt > DateTime.UtcNow.AddMonths(-1))
    .OrderByDescending(o => o.Total)
    .Take(10)
    .Select(o => new OrderSummary(o.Id, o.Total, o.CreatedAt))
    .ToList();

// Deferred execution — query runs when iterated
var query = orders.Where(o => o.Total > 100);  // not yet executed
var list  = query.ToList();                     // executes now
```

### Query syntax (alternative, less common in backend code)

```csharp
var result =
    from o in orders
    where o.Status == OrderStatus.Active
    orderby o.Total descending
    select new { o.Id, o.Total };
```

---

## 1.13 Enums and flags

```csharp
// Basic enum
public enum OrderStatus
{
    Pending   = 0,
    Confirmed = 1,
    Shipped   = 2,
    Delivered = 3,
    Cancelled = 4
}

// Flags enum (bitwise combination)
[Flags]
public enum Permissions
{
    None    = 0,
    Read    = 1 << 0,  // 1
    Write   = 1 << 1,  // 2
    Delete  = 1 << 2,  // 4
    Admin   = Read | Write | Delete
}

var perms = Permissions.Read | Permissions.Write;
bool canRead  = perms.HasFlag(Permissions.Read);   // true
bool canDelete = perms.HasFlag(Permissions.Delete); // false

// Enum parsing
var status = Enum.Parse<OrderStatus>("Confirmed");
if (Enum.TryParse<OrderStatus>("invalid", out var s)) { }

// Extension methods on enums
public static class OrderStatusExtensions
{
    public static bool IsTerminal(this OrderStatus status)
        => status is OrderStatus.Delivered or OrderStatus.Cancelled;
}
```

---

## 1.14 Extension methods

```csharp
// Must be in a static class
public static class StringExtensions
{
    public static string ToKebabCase(this string value)
        => string.IsNullOrEmpty(value) ? value
            : string.Concat(value.Select((c, i) =>
                i > 0 && char.IsUpper(c) ? $"-{c}" : $"{c}")).ToLower();

    public static bool IsValidEmail(this string value)
        => !string.IsNullOrWhiteSpace(value) && value.Contains('@');
}

// Usage — reads like a method on string
"HelloWorld".ToKebabCase();  // "hello-world"
"user@test.com".IsValidEmail(); // true

// Extension methods on IQueryable (very common with EF Core)
public static class QueryableExtensions
{
    public static IQueryable<T> WhereIf<T>(
        this IQueryable<T> query,
        bool condition,
        Expression<Func<T, bool>> predicate)
        => condition ? query.Where(predicate) : query;

    public static async Task<PagedResult<T>> ToPagedAsync<T>(
        this IQueryable<T> query,
        int page, int pageSize,
        CancellationToken ct = default)
    {
        var total = await query.CountAsync(ct);
        var items = await query.Skip((page - 1) * pageSize).Take(pageSize).ToListAsync(ct);
        return new PagedResult<T>(items, total, page, pageSize);
    }
}
```

---

## 1.15 Delegates, Func, and Action

```csharp
// Func — returns a value
Func<int, int, int> add = (a, b) => a + b;
int result = add(3, 4); // 7

Func<string, bool> isValid = s => s.Length > 3;

// Action — void return
Action<string> log = msg => Console.WriteLine(msg);
Action<int, int> print = (a, b) => Console.WriteLine($"{a} + {b}");

// Predicate (Func<T, bool> shorthand)
Predicate<int> isEven = n => n % 2 == 0;

// Using in higher-order methods
public List<T> Filter<T>(List<T> items, Func<T, bool> predicate)
    => items.Where(predicate).ToList();

// Events (multicast delegates)
public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;

    protected virtual void OnOrderCreated(Order order)
        => OrderCreated?.Invoke(this, new OrderCreatedEventArgs(order));
}
```

---

## 1.16 Structs and value objects

```csharp
// Custom value type — good for strongly-typed IDs
public readonly struct CustomerId : IEquatable<CustomerId>
{
    public Guid Value { get; }

    public CustomerId(Guid value)
    {
        if (value == Guid.Empty)
            throw new ArgumentException("CustomerId cannot be empty.");
        Value = value;
    }

    public static CustomerId New() => new(Guid.NewGuid());
    public static CustomerId Parse(string value) => new(Guid.Parse(value));

    public bool Equals(CustomerId other) => Value == other.Value;
    public override bool Equals(object? obj) => obj is CustomerId other && Equals(other);
    public override int GetHashCode() => Value.GetHashCode();
    public override string ToString() => Value.ToString();

    public static bool operator ==(CustomerId left, CustomerId right) => left.Equals(right);
    public static bool operator !=(CustomerId left, CustomerId right) => !left.Equals(right);
}

// Simpler: use record struct
public record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
}
```