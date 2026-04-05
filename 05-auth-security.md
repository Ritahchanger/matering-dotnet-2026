# 5. Authentication & Security

## 5.1 JWT authentication

```csharp
// Install: dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

// appsettings.json
{
  "Jwt": {
    "Secret": "your-256-bit-secret-key-here-keep-it-safe",
    "Issuer": "https://api.myapp.com",
    "Audience": "https://myapp.com",
    "ExpiryMinutes": 60
  }
}

// Register JWT authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        var config = builder.Configuration.GetSection("Jwt");
        var secret = config["Secret"]!;

        opts.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer              = config["Issuer"],
            ValidAudience            = config["Audience"],
            IssuerSigningKey         = new SymmetricSecurityKey(
                                           Encoding.UTF8.GetBytes(secret)),
            ClockSkew                = TimeSpan.Zero  // no tolerance
        };

        // Support tokens from SignalR query string
        opts.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var token = context.Request.Query["access_token"];
                var path  = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(token) && path.StartsWithSegments("/hubs"))
                    context.Token = token;
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();
```

### Token service

```csharp
public class TokenService(IOptions<JwtOptions> opts) : ITokenService
{
    private readonly JwtOptions _opts = opts.Value;

    public string GenerateToken(User user)
    {
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub,   user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti,   Guid.NewGuid().ToString()),
            new Claim(ClaimTypes.Role,               user.Role),
            new Claim("tenant_id",                   user.TenantId.ToString())
        };

        var key         = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_opts.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var expiry      = DateTime.UtcNow.AddMinutes(_opts.ExpiryMinutes);

        var token = new JwtSecurityToken(
            issuer:             _opts.Issuer,
            audience:           _opts.Audience,
            claims:             claims,
            expires:            expiry,
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var bytes = new byte[64];
        RandomNumberGenerator.Fill(bytes);
        return Convert.ToBase64String(bytes);
    }
}
```

### Auth endpoints

```csharp
public static class AuthEndpoints
{
    public static IEndpointRouteBuilder MapAuthEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/auth").WithTags("Auth");

        group.MapPost("/login", Login);
        group.MapPost("/refresh", Refresh);
        group.MapPost("/logout", Logout).RequireAuthorization();

        return app;
    }

    private static async Task<IResult> Login(
        LoginRequest req,
        IAuthService auth,
        CancellationToken ct)
    {
        var result = await auth.LoginAsync(req.Email, req.Password, ct);
        return result.IsSuccess
            ? Results.Ok(result.Value)
            : Results.Problem("Invalid credentials", statusCode: 401);
    }
}

public record LoginRequest(string Email, string Password);
public record AuthResponse(string AccessToken, string RefreshToken, DateTime ExpiresAt);
```

---

## 5.2 Authorization

```csharp
// Policy-based authorization
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    opts.AddPolicy("MinimumAge", policy =>
        policy.RequireClaim("age")
              .AddRequirements(new MinimumAgeRequirement(18)));

    opts.AddPolicy("SameTenant", policy =>
        policy.AddRequirements(new SameTenantRequirement()));
});

// Custom requirement + handler
public class SameTenantRequirement : IAuthorizationRequirement { }

public class SameTenantHandler(IHttpContextAccessor httpContext)
    : AuthorizationHandler<SameTenantRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameTenantRequirement requirement)
    {
        var userTenant    = context.User.FindFirst("tenant_id")?.Value;
        var routeTenantId = httpContext.HttpContext?.GetRouteValue("tenantId")?.ToString();

        if (userTenant == routeTenantId)
            context.Succeed(requirement);
        else
            context.Fail(new AuthorizationFailureReason(this, "Tenant mismatch"));

        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, SameTenantHandler>();

// Apply in endpoints
app.MapDelete("/products/{id}", DeleteProduct)
   .RequireAuthorization("AdminOnly");

app.MapGet("/tenants/{tenantId}/orders", GetOrders)
   .RequireAuthorization("SameTenant");
```

### Reading the current user

```csharp
// Abstraction for the current user (testable)
public interface ICurrentUser
{
    Guid   Id       { get; }
    string Email    { get; }
    string Role     { get; }
    Guid   TenantId { get; }
    bool   IsAdmin  { get; }
}

public class CurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    private ClaimsPrincipal User => accessor.HttpContext?.User
        ?? throw new InvalidOperationException("No HTTP context");

    public Guid   Id       => Guid.Parse(User.FindFirstValue(JwtRegisteredClaimNames.Sub)!);
    public string Email    => User.FindFirstValue(JwtRegisteredClaimNames.Email)!;
    public string Role     => User.FindFirstValue(ClaimTypes.Role)!;
    public Guid   TenantId => Guid.Parse(User.FindFirstValue("tenant_id")!);
    public bool   IsAdmin  => Role == "Admin";
}

// Register
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, CurrentUser>();

// Use in service
public class OrderService(IOrderRepository repo, ICurrentUser currentUser)
{
    public async Task<IReadOnlyList<Order>> GetMyOrdersAsync(CancellationToken ct)
        => await repo.GetByCustomerAsync(currentUser.Id, ct);
}
```

---

## 5.3 Password hashing

```csharp
// Never store plain-text passwords
// Use BCrypt (dotnet add package BCrypt.Net-Next)
public class PasswordHasher : IPasswordHasher
{
    private const int WorkFactor = 12;

    public string Hash(string password)
        => BCrypt.Net.BCrypt.HashPassword(password, WorkFactor);

    public bool Verify(string password, string hash)
        => BCrypt.Net.BCrypt.Verify(password, hash);
}

// Or use ASP.NET Core's built-in
public class AspPasswordHasher : IPasswordHasher
{
    private readonly PasswordHasher<object> _hasher = new();

    public string Hash(string password)
        => _hasher.HashPassword(null!, password);

    public bool Verify(string password, string hash)
        => _hasher.VerifyHashedPassword(null!, hash, password)
               != PasswordVerificationResult.Failed;
}
```

---

## 5.4 Security best practices

### Input validation

```csharp
// Always validate and sanitise input
public class CreateCommentValidator : AbstractValidator<CreateCommentRequest>
{
    public CreateCommentValidator()
    {
        RuleFor(x => x.Content)
            .NotEmpty()
            .MaximumLength(2000)
            .Must(NotContainScripts)
            .WithMessage("Content contains disallowed characters");
    }

    private bool NotContainScripts(string content)
        => !content.Contains("<script", StringComparison.OrdinalIgnoreCase);
}
```

### SQL injection prevention

```csharp
// EF Core parameterises automatically — safe
db.Users.Where(u => u.Email == email).FirstOrDefaultAsync(ct);

// Dapper with parameters — safe
conn.QueryAsync("SELECT * FROM users WHERE email = @Email", new { Email = email });

// Raw SQL with EF Core — use parameters
db.Users.FromSqlRaw("SELECT * FROM users WHERE email = {0}", email); // safe — parameterised
db.Users.FromSqlRaw($"SELECT * FROM users WHERE email = '{email}'"); // DANGEROUS — SQL injection
```

### Rate limiting

```csharp
// dotnet add package Microsoft.AspNetCore.RateLimiting (built into .NET 8+)
builder.Services.AddRateLimiter(opts =>
{
    // Global fixed window
    opts.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));

    // Named policy for login — stricter
    opts.AddFixedWindowLimiter("login", opts =>
    {
        opts.PermitLimit = 5;
        opts.Window      = TimeSpan.FromMinutes(15);
        opts.QueueLimit  = 0;
    });

    opts.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = 429;
        await ctx.HttpContext.Response.WriteAsJsonAsync(
            new { error = "Too many requests. Please try again later." }, ct);
    };
});

app.UseRateLimiter();

// Apply to specific endpoint
app.MapPost("/auth/login", LoginHandler).RequireRateLimiting("login");
```

### CORS

```csharp
builder.Services.AddCors(opts =>
{
    opts.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com", "https://staging.myapp.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials());

    // Development only — never in production
    if (builder.Environment.IsDevelopment())
        opts.AddPolicy("AllowAll", policy =>
            policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());
});

app.UseCors(app.Environment.IsDevelopment() ? "AllowAll" : "AllowFrontend");
```

### Secrets management

```csharp
// Development: dotnet user-secrets
// dotnet user-secrets set "Jwt:Secret" "my-dev-secret"

// Production: environment variables, Azure Key Vault, AWS Secrets Manager
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables()          // overrides json
    .AddUserSecrets<Program>(optional: true); // development only

// Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```