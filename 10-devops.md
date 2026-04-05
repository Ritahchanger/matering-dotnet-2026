# 10. DevOps & Deployment

## 10.1 Dockerfile (optimised multi-stage)

```dockerfile
# Stage 1 — restore (cached unless .csproj files change)
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS restore
WORKDIR /src
COPY ["src/MyApp.Api/MyApp.Api.csproj",             "src/MyApp.Api/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj", "src/MyApp.Application/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj",         "src/MyApp.Domain/"]
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Stage 2 — build
FROM restore AS build
COPY . .
RUN dotnet build "src/MyApp.Api/MyApp.Api.csproj" \
    -c Release --no-restore -o /app/build

# Stage 3 — publish
FROM build AS publish
RUN dotnet publish "src/MyApp.Api/MyApp.Api.csproj" \
    -c Release --no-build -o /app/publish \
    /p:UseAppHost=false

# Stage 4 — runtime image (no SDK)
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

# Run as non-root
RUN addgroup --system --gid 1001 appgroup && \
    adduser  --system --uid 1001 --ingroup appgroup appuser
USER appuser

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### docker-compose.yml

```yaml
services:
  api:
    build:
      context: .
      target: final
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Host=postgres;Database=myapp;Username=postgres;Password=secret
      - ConnectionStrings__Redis=redis:6379
      - Jwt__Secret=${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       myapp
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass "${REDIS_PASSWORD}"
    volumes:
      - redisdata:/data

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"
    environment:
      - ACCEPT_EULA=Y

volumes:
  pgdata:
  redisdata:
```

---

## 10.2 GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '9.0.x'
  DOTNET_NOLOGO: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB:       testdb
          POSTGRES_USER:     postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: dotnet test --no-build -c Release \
          --logger "trx;LogFileName=results.trx" \
          --collect:"XPlat Code Coverage"
        env:
          ConnectionStrings__Default: "Host=localhost;Database=testdb;Username=postgres;Password=postgres"

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: '**/*.trx'
          reporter: dotnet-trx

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  publish:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest,ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to:   type=gha,mode=max
```

---

## 10.3 Environment configuration

```csharp
// appsettings hierarchy — later overrides earlier
// appsettings.json           — base (committed)
// appsettings.Production.json — prod-specific (do NOT commit secrets)
// Environment variables       — CI/CD, container, k8s secrets
// User secrets               — developer local (never committed)

// Validate configuration at startup
public static class ConfigurationExtensions
{
    public static IServiceCollection AddAndValidateOptions<T>(
        this IServiceCollection services,
        IConfiguration configuration,
        string sectionName) where T : class
    {
        services.AddOptions<T>()
            .Bind(configuration.GetSection(sectionName))
            .ValidateDataAnnotations()
            .ValidateOnStart();
        return services;
    }
}
```

---

## 10.4 Application startup validation

```csharp
// Fail fast — detect misconfigurations before serving traffic
var app = builder.Build();

// Run migrations on startup
if (app.Environment.IsProduction())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var pendingMigrations = await db.Database.GetPendingMigrationsAsync();
    if (pendingMigrations.Any())
    {
        app.Logger.LogInformation("Applying {Count} pending migrations", pendingMigrations.Count());
        await db.Database.MigrateAsync();
    }
}

// Verify external dependencies
if (app.Environment.IsProduction())
{
    using var scope = app.Services.CreateScope();
    var healthCheck = scope.ServiceProvider.GetRequiredService<HealthCheckService>();
    var result = await healthCheck.CheckHealthAsync(r => r.Tags.Contains("startup"));
    if (result.Status != HealthStatus.Healthy)
    {
        app.Logger.LogCritical("Startup health check failed: {Result}", result);
        Environment.Exit(1);
    }
}
```

---

## 10.5 Graceful shutdown

```csharp
// ASP.NET Core handles SIGTERM gracefully by default
// Configure the shutdown timeout
builder.Services.Configure<HostOptions>(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// In-flight requests complete, new requests are rejected
// Background services receive the CancellationToken — respond to it!
public class MyBackgroundService(ILogger<MyBackgroundService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                logger.LogInformation("Background service stopping gracefully");
                break;
            }
        }
    }
}
```

---

## 10.6 Native AOT (ahead-of-time compilation)

```xml
<!-- MyApp.Api.csproj -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

```bash
# Publish AOT binary
dotnet publish -c Release -r linux-x64

# Result: single native binary, no .NET runtime required
# ~5ms startup vs ~200ms for JIT
# ~30MB image vs ~200MB for framework-dependent
```

```csharp
// AOT constraints:
// - No runtime reflection (use source generators instead)
// - No dynamic code generation
// - JsonSerializerContext required for System.Text.Json

[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(List<Product>))]
[JsonSerializable(typeof(CreateProductRequest))]
public partial class AppJsonContext : JsonSerializerContext { }

// Register
builder.Services.ConfigureHttpJsonOptions(opts =>
    opts.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```