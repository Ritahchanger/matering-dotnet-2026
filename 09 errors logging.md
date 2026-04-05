# 9. Error Handling & Logging

## 9.1 Exception handling in ASP.NET Core

```csharp
// Global exception handler (preferred over try/catch everywhere)
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

app.UseExceptionHandler();

// Handler implementation
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        var (statusCode, title) = exception switch
        {
            ValidationException ve  => (400, "Validation failed"),
            NotFoundException       => (404, "Resource not found"),
            ConflictException       => (409, "Conflict"),
            UnauthorizedException   => (401, "Unauthorised"),
            ForbiddenException      => (403, "Forbidden"),
            DomainException         => (422, "Business rule violation"),
            OperationCanceledException => (499, "Request cancelled"),
            _                       => (500, "An unexpected error occurred")
        };

        if (statusCode == 500)
            logger.LogError(exception, "Unhandled exception");
        else
            logger.LogWarning(exception, "Handled exception: {Type}", exception.GetType().Name);

        context.Response.StatusCode = statusCode;

        var problem = new ProblemDetails
        {
            Status   = statusCode,
            Title    = title,
            Detail   = exception is ValidationException ve2
                           ? string.Join("; ", ve2.Errors.Select(e => e.ErrorMessage))
                           : exception.Message,
            Instance = context.Request.Path
        };

        // Add validation errors as extensions
        if (exception is ValidationException validationEx)
            problem.Extensions["errors"] = validationEx.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

        await context.Response.WriteAsJsonAsync(problem, ct);
        return true;
    }
}
```

### Custom exceptions

```csharp
public class NotFoundException(string entity, object id)
    : Exception($"{entity} with id '{id}' was not found.");

public class ConflictException(string message) : Exception(message);

public class DomainException(string message) : Exception(message);

public class ForbiddenException(string message = "You do not have permission.") : Exception(message);
```

---

## 9.2 Structured logging with Serilog

```csharp
// dotnet add package Serilog.AspNetCore Serilog.Sinks.Console Serilog.Sinks.Seq

// Program.cs
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Seq("http://localhost:5341"));  // structured log UI

// appsettings.json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
      }
    }
  }
}

// Request logging middleware (Serilog built-in)
app.UseSerilogRequestLogging(opts =>
{
    opts.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    opts.EnrichDiagnosticContext = (ctx, http) =>
    {
        ctx.Set("RequestHost", http.Request.Host.Value);
        ctx.Set("UserId", http.User.FindFirstValue(JwtRegisteredClaimNames.Sub) ?? "anonymous");
    };
});
```

### Logging best practices

```csharp
public class OrderService(ILogger<OrderService> logger)
{
    public async Task<Order> CreateAsync(CreateOrderCommand cmd, CancellationToken ct)
    {
        // Use message templates — not string interpolation
        logger.LogInformation("Creating order for customer {CustomerId}", cmd.CustomerId);

        // Log structured data
        logger.LogInformation(
            "Order created {@Order}",          // @ = destructure the object
            new { Id = order.Id, cmd.CustomerId, Total = order.Total });

        // Log exceptions with context
        try
        {
            await repo.SaveChangesAsync(ct);
        }
        catch (Exception ex)
        {
            logger.LogError(ex,
                "Failed to save order for customer {CustomerId}",
                cmd.CustomerId);
            throw;
        }

        // Use log scopes for correlated operations
        using (logger.BeginScope(new { OrderId = order.Id }))
        {
            logger.LogInformation("Sending confirmation email");
            await emailService.SendAsync(order, ct);
            logger.LogInformation("Email sent");
        }

        return order;
    }
}

// Log levels guide:
// Trace   — very detailed, hot path tracing (disabled in prod)
// Debug   — diagnostic information for developers
// Information — normal application events (order created, user logged in)
// Warning — unexpected but recoverable (cache miss, retry)
// Error   — failure that needs attention (DB timeout, external API error)
// Critical — application is about to crash or has crashed
```

---

## 9.3 OpenTelemetry observability

```csharp
// dotnet add package OpenTelemetry.Extensions.Hosting
// dotnet add package OpenTelemetry.Instrumentation.AspNetCore
// dotnet add package OpenTelemetry.Instrumentation.Http
// dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
// dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("my-api", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("MyApp.*")               // custom ActivitySource
        .AddOtlpExporter(opts =>
            opts.Endpoint = new Uri("http://localhost:4317")))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("MyApp.*")                // custom meters
        .AddOtlpExporter())
    .WithLogging(logging => logging
        .AddOtlpExporter());

// Custom traces
public class OrderService
{
    private static readonly ActivitySource ActivitySource = new("MyApp.Orders");

    public async Task<Order> CreateAsync(CreateOrderCommand cmd, CancellationToken ct)
    {
        using var activity = ActivitySource.StartActivity("CreateOrder");
        activity?.SetTag("customer.id", cmd.CustomerId.ToString());

        var order = Order.Create(cmd.CustomerId);
        activity?.SetTag("order.id", order.Id.ToString());

        await repo.SaveChangesAsync(ct);

        activity?.SetTag("order.total", order.Total);
        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    }
}

// Custom metrics
public class OrderMetrics
{
    private static readonly Meter Meter = new("MyApp.Orders");
    private static readonly Counter<long>   OrdersCreated  = Meter.CreateCounter<long>("orders.created");
    private static readonly Histogram<double> OrderTotal   = Meter.CreateHistogram<double>("orders.total", "USD");

    public static void RecordOrderCreated(Order order)
    {
        OrdersCreated.Add(1, new KeyValuePair<string, object?>("status", "created"));
        OrderTotal.Record((double)order.Total);
    }
}
```

---

## 9.4 Health checks

```csharp
// Install: dotnet add package AspNetCore.HealthChecks.NpgSql AspNetCore.HealthChecks.Redis

builder.Services
    .AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Default")!, name: "postgres")
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!,     name: "redis")
    .AddCheck<ExternalApiHealthCheck>("payment-gateway");

// Custom health check
public class ExternalApiHealthCheck(IPaymentClient paymentClient) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            await paymentClient.PingAsync(ct);
            return HealthCheckResult.Healthy("Payment gateway is reachable");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Payment gateway is unreachable", ex);
        }
    }
}

// Map endpoints
app.MapHealthChecks("/health/live",  new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```