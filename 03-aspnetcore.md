# 3. ASP.NET Core

## 3.1 Application bootstrap

```csharp
// Program.cs — the modern .NET 9 entrypoint (no Startup class)
var builder = WebApplication.CreateBuilder(args);

// ── Services ──────────────────────────────────────────────────────────────
builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();

builder.Services
    .AddApplication()       // extension method in Application layer
    .AddInfrastructure(builder.Configuration); // extension method in Infrastructure

// ── Build ─────────────────────────────────────────────────────────────────
var app = builder.Build();

// ── Middleware pipeline (order matters) ───────────────────────────────────
if (app.Environment.IsDevelopment())
    app.MapOpenApi();

app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// ── Endpoints ─────────────────────────────────────────────────────────────
app.MapProductEndpoints();
app.MapOrderEndpoints();
app.MapHealthChecks("/health");

app.Run();
```

---

## 3.2 Dependency injection

ASP.NET Core has a built-in IoC container. Three lifetimes:

| Lifetime | Instance per | Use for |
|----------|-------------|---------|
| `Singleton` | App lifetime | Config, caches, HttpClient factories |
| `Scoped` | HTTP request | DbContext, repositories, services |
| `Transient` | Every injection | Lightweight, stateless utilities |

```csharp
// Registration
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddTransient<IEmailValidator, EmailValidator>();

// Register with factory
builder.Services.AddScoped<IPaymentGateway>(sp =>
{
    var config = sp.GetRequiredService<IOptions<PaymentOptions>>().Value;
    return new StripeGateway(config.ApiKey);
});

// Register all implementations of an interface
builder.Services.Scan(scan => scan
    .FromAssemblyOf<IOrderHandler>()
    .AddClasses(c => c.AssignableTo<IOrderHandler>())
    .AsImplementedInterfaces()
    .WithScopedLifetime());

// DI in classes — constructor injection (preferred)
public class OrderService(
    IOrderRepository orders,
    IPaymentGateway payment,
    ILogger<OrderService> logger)
{
    // parameters are captured as fields automatically (primary constructor)
}

// DI in minimal APIs — inject from route handler parameters
app.MapGet("/orders/{id}", async (
    Guid id,
    IOrderRepository repo,
    CancellationToken ct) =>
{
    var order = await repo.GetByIdAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
});
```

---

## 3.3 Configuration and options pattern

```csharp
// appsettings.json
{
  "Database": {
    "ConnectionString": "Host=localhost;Database=myapp",
    "MaxPoolSize": 20,
    "CommandTimeout": 30
  },
  "Email": {
    "SmtpHost": "smtp.example.com",
    "Port": 587,
    "FromAddress": "noreply@example.com"
  }
}

// Options class
public class DatabaseOptions
{
    public const string SectionName = "Database";

    [Required]
    public string ConnectionString { get; init; } = "";

    [Range(1, 100)]
    public int MaxPoolSize { get; init; } = 20;

    public int CommandTimeout { get; init; } = 30;
}

// Register with validation
builder.Services
    .AddOptions<DatabaseOptions>()
    .BindConfiguration(DatabaseOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // fail fast at startup if invalid

// Consume
public class DatabaseService(IOptions<DatabaseOptions> opts)
{
    private readonly DatabaseOptions _opts = opts.Value;

    public NpgsqlConnection CreateConnection()
        => new(_opts.ConnectionString);
}

// IOptionsSnapshot — re-read per request (useful for hot reload)
public class EmailService(IOptionsSnapshot<EmailOptions> opts)
{
    // opts.Value is refreshed each request
}

// IOptionsMonitor — live updates with callback
public class FeatureService(IOptionsMonitor<FeatureFlags> monitor)
{
    private FeatureFlags _flags = monitor.CurrentValue;

    public FeatureService(IOptionsMonitor<FeatureFlags> monitor)
    {
        _flags = monitor.CurrentValue;
        monitor.OnChange(flags => _flags = flags);
    }
}
```

---

## 3.4 Minimal APIs

```csharp
// Simple endpoint
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));

// With route parameter
app.MapGet("/products/{id:int}", async (int id, IProductService svc, CancellationToken ct) =>
    await svc.GetByIdAsync(id, ct) is { } product
        ? Results.Ok(product)
        : Results.NotFound());

// With query parameters (bound automatically)
app.MapGet("/products", async (
    [FromQuery] string? search,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    IProductService svc,
    CancellationToken ct) =>
{
    var result = await svc.SearchAsync(search, page, pageSize, ct);
    return Results.Ok(result);
});

// POST with body
app.MapPost("/products", async (
    CreateProductRequest req,
    IProductService svc,
    CancellationToken ct) =>
{
    var product = await svc.CreateAsync(req, ct);
    return Results.CreatedAtRoute("GetProduct", new { product.Id }, product);
})
.WithName("CreateProduct")
.Produces<Product>(201)
.ProducesValidationProblem();

// Route groups — organise related endpoints
var products = app.MapGroup("/api/products")
    .WithTags("Products")
    .RequireAuthorization();

products.MapGet("/",      GetAllProducts);
products.MapGet("/{id}",  GetProductById).WithName("GetProduct");
products.MapPost("/",     CreateProduct);
products.MapPut("/{id}",  UpdateProduct);
products.MapDelete("/{id}", DeleteProduct);
```

### Organising endpoints (recommended pattern)

```csharp
// ProductEndpoints.cs — extension method pattern
public static class ProductEndpoints
{
    public static IEndpointRouteBuilder MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products");

        group.MapGet("/", GetAll);
        group.MapGet("/{id:int}", GetById).WithName("GetProductById");
        group.MapPost("/", Create).RequireAuthorization();
        group.MapPut("/{id:int}", Update).RequireAuthorization();
        group.MapDelete("/{id:int}", Delete).RequireAuthorization("Admin");

        return app;
    }

    private static async Task<IResult> GetAll(
        IProductService svc, CancellationToken ct)
    {
        var products = await svc.GetAllAsync(ct);
        return Results.Ok(products);
    }

    private static async Task<IResult> GetById(
        int id, IProductService svc, CancellationToken ct)
    {
        return await svc.GetByIdAsync(id, ct) is { } product
            ? Results.Ok(product)
            : Results.NotFound();
    }

    private static async Task<IResult> Create(
        CreateProductRequest req,
        IValidator<CreateProductRequest> validator,
        IProductService svc,
        CancellationToken ct)
    {
        var validation = await validator.ValidateAsync(req, ct);
        if (!validation.IsValid)
            return Results.ValidationProblem(validation.ToDictionary());

        var product = await svc.CreateAsync(req, ct);
        return Results.CreatedAtRoute("GetProductById", new { product.Id }, product);
    }

    // ... Update, Delete
}
```

---

## 3.5 Middleware

```csharp
// Inline middleware
app.Use(async (context, next) =>
{
    // Before
    var sw = Stopwatch.StartNew();

    await next(context);

    // After
    sw.Stop();
    context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
});

// Class-based middleware
public class RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation(
            "HTTP {Method} {Path} started",
            context.Request.Method,
            context.Request.Path);

        var sw = Stopwatch.StartNew();
        try
        {
            await next(context);
            logger.LogInformation(
                "HTTP {Method} {Path} responded {StatusCode} in {Elapsed}ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "HTTP {Method} {Path} failed", context.Request.Method, context.Request.Path);
            throw;
        }
    }
}

// Extension method for clean registration
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder app)
        => app.UseMiddleware<RequestLoggingMiddleware>();
}

// Endpoint filter (scoped to specific routes)
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx,
        EndpointFilterDelegate next)
    {
        var arg = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (arg is not null)
        {
            var result = await validator.ValidateAsync(arg);
            if (!result.IsValid)
                return Results.ValidationProblem(result.ToDictionary());
        }
        return await next(ctx);
    }
}

// Apply to a group
products.MapPost("/", Create)
        .AddEndpointFilter<ValidationFilter<CreateProductRequest>>();
```

---

## 3.6 Model binding and validation

```csharp
// Request DTO with FluentValidation
public record CreateProductRequest(
    string Name,
    decimal Price,
    string Category,
    int Stock);

public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200);

        RuleFor(x => x.Price)
            .GreaterThan(0)
            .LessThan(1_000_000);

        RuleFor(x => x.Category)
            .NotEmpty()
            .Must(BeValidCategory)
            .WithMessage("Category must be one of: Electronics, Clothing, Food");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0);
    }

    private bool BeValidCategory(string category)
        => new[] { "Electronics", "Clothing", "Food" }.Contains(category);
}

// Register FluentValidation
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductRequestValidator>();

// Binding sources
app.MapGet("/items", (
    [FromRoute]  int id,
    [FromQuery]  string? filter,
    [FromHeader] string? authorization,
    [FromBody]   UpdateRequest body,
    [FromForm]   IFormFile? file) => { });
```

---

## 3.7 Returning responses

```csharp
// IResult / Results static helpers
Results.Ok(value)                          // 200
Results.Created("/resource/1", value)      // 201
Results.CreatedAtRoute("name", route, val) // 201 with Location header
Results.NoContent()                        // 204
Results.BadRequest("message")              // 400
Results.Unauthorized()                     // 401
Results.Forbid()                           // 403
Results.NotFound()                         // 404
Results.Conflict()                         // 409
Results.UnprocessableEntity()              // 422
Results.ValidationProblem(errors)          // 422 with RFC 9457 body
Results.Problem("detail", statusCode: 500) // ProblemDetails

// Typed results (best for OpenAPI accuracy)
async Task<Results<Ok<Product>, NotFound>> GetById(int id, IProductService svc, CancellationToken ct)
{
    var product = await svc.GetByIdAsync(id, ct);
    return product is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(product);
}

// Streaming file
Results.File(stream, "application/pdf", "report.pdf")
Results.Stream(async stream => await writer.WriteAsync(stream))
```

---

## 3.8 HttpClient and external APIs

```csharp
// Register a typed HttpClient
builder.Services.AddHttpClient<IPaymentClient, StripeClient>(client =>
{
    client.BaseAddress = new Uri("https://api.stripe.com/v1/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddStandardResilienceHandler(); // Polly retry/circuit breaker built-in .NET 8+

// Typed client
public class StripeClient(HttpClient client) : IPaymentClient
{
    public async Task<ChargeResponse> ChargeAsync(ChargeRequest req, CancellationToken ct = default)
    {
        var response = await client.PostAsJsonAsync("charges", req, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<ChargeResponse>(ct)
            ?? throw new InvalidOperationException("Empty response from Stripe");
    }
}

// Named client (alternative)
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
});

// Consume named client
public class GitHubService(IHttpClientFactory factory)
{
    public async Task<string> GetProfileAsync(string username, CancellationToken ct = default)
    {
        var client = factory.CreateClient("GitHub");
        return await client.GetStringAsync($"users/{username}", ct);
    }
}
```