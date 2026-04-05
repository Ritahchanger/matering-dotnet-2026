# 7. Testing

## 7.1 Unit tests with xUnit

```csharp
// Install: dotnet add package xunit xunit.runner.visualstudio FluentAssertions

public class OrderTests
{
    // Fact — single scenario
    [Fact]
    public void Create_ShouldProducePendingOrder()
    {
        var customerId = Guid.NewGuid();
        var order = Order.Create(customerId);

        order.Status.Should().Be(OrderStatus.Pending);
        order.CustomerId.Should().Be(customerId);
        order.Lines.Should().BeEmpty();
        order.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderCreatedEvent>();
    }

    // Theory — parameterised scenarios
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public void AddLine_WithInvalidQuantity_ShouldThrow(int qty)
    {
        var order = Order.Create(Guid.NewGuid());
        var act = () => order.AddLine(Guid.NewGuid(), "Widget", qty, 9.99m);
        act.Should().Throw<ArgumentOutOfRangeException>();
    }

    [Fact]
    public void Confirm_WhenEmpty_ShouldThrowDomainException()
    {
        var order = Order.Create(Guid.NewGuid());
        var act = () => order.Confirm();
        act.Should().Throw<DomainException>()
           .WithMessage("*empty*");
    }

    [Fact]
    public void Confirm_WithLines_ShouldChangeStatus()
    {
        var order = GivenOrderWithLines();
        order.Confirm();
        order.Status.Should().Be(OrderStatus.Confirmed);
    }

    // Private helpers (arrange shortcuts)
    private static Order GivenOrderWithLines()
    {
        var order = Order.Create(Guid.NewGuid());
        order.AddLine(Guid.NewGuid(), "Widget", 2, 49.99m);
        return order;
    }
}
```

### Class fixtures (shared setup)

```csharp
// Shared across all tests in the class
public class ProductServiceTests : IClassFixture<ProductServiceFixture>
{
    private readonly ProductServiceFixture _fixture;

    public ProductServiceTests(ProductServiceFixture fixture)
        => _fixture = fixture;

    [Fact]
    public async Task GetByIdAsync_WhenExists_ReturnsProduct()
    {
        var product = await _fixture.Service.GetByIdAsync(1);
        product.Should().NotBeNull();
        product!.Name.Should().Be("Widget");
    }
}

public class ProductServiceFixture
{
    public IProductService Service { get; }

    public ProductServiceFixture()
    {
        var repo = new FakeProductRepository();
        repo.Add(new Product(1, "Widget", 9.99m));
        Service = new ProductService(repo, NullLogger<ProductService>.Instance);
    }
}
```

---

## 7.2 Mocking with NSubstitute

```csharp
// Install: dotnet add package NSubstitute

public class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly IEmailService    _email = Substitute.For<IEmailService>();
    private readonly OrderService     _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repo, _email, NullLogger<OrderService>.Instance);
    }

    [Fact]
    public async Task CreateAsync_ShouldSaveAndReturnOrder()
    {
        // Arrange
        var cmd = new CreateOrderCommand(Guid.NewGuid(), []);
        _repo.SaveChangesAsync(Arg.Any<CancellationToken>()).Returns(Task.CompletedTask);

        // Act
        var result = await _sut.CreateAsync(cmd);

        // Assert
        result.IsSuccess.Should().BeTrue();
        _repo.Received(1).Add(Arg.Any<Order>());
        await _repo.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task GetByIdAsync_WhenNotFound_ShouldReturnFailure()
    {
        // Arrange
        var id = Guid.NewGuid();
        _repo.GetByIdAsync(id, Arg.Any<CancellationToken>()).Returns((Order?)null);

        // Act
        var result = await _sut.GetByIdAsync(id);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Contain("NotFound");
    }

    [Fact]
    public async Task ConfirmAsync_ShouldSendConfirmationEmail()
    {
        // Arrange
        var order = Order.Create(Guid.NewGuid());
        order.AddLine(Guid.NewGuid(), "Item", 1, 10m);
        _repo.GetByIdAsync(order.Id, Arg.Any<CancellationToken>()).Returns(order);

        // Act
        await _sut.ConfirmAsync(order.Id);

        // Assert
        await _email.Received(1).SendAsync(
            Arg.Is<EmailMessage>(m => m.Subject.Contains("confirmed")),
            Arg.Any<CancellationToken>());
    }
}
```

---

## 7.3 Integration tests with WebApplicationFactory

```csharp
// Install: dotnet add package Microsoft.AspNetCore.Mvc.Testing

public class OrdersApiTests : IClassFixture<TestWebAppFactory>, IAsyncLifetime
{
    private readonly TestWebAppFactory _factory;
    private readonly HttpClient _client;

    public OrdersApiTests(TestWebAppFactory factory)
    {
        _factory = factory;
        _client  = factory.CreateClient();
    }

    public async Task InitializeAsync() => await _factory.ResetDatabaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task POST_orders_ShouldReturn201()
    {
        // Arrange
        var customerId = await _factory.SeedCustomerAsync();
        var request = new CreateOrderCommand(customerId, [
            new(ProductId: Guid.NewGuid(), Quantity: 2, UnitPrice: 49.99m)
        ]);

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<CreateOrder.Response>();
        body!.OrderId.Should().NotBeEmpty();
    }

    [Fact]
    public async Task GET_orders_id_WhenNotFound_ShouldReturn404()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}

// Test factory
public class TestWebAppFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace real DB with test container
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(opts =>
                opts.UseNpgsql(_postgres.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task ResetDatabaseAsync()
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Orders.ExecuteDeleteAsync();
        await db.Customers.ExecuteDeleteAsync();
    }

    public async Task<Guid> SeedCustomerAsync()
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var customer = Customer.Create("Test User", "test@example.com");
        db.Customers.Add(customer);
        await db.SaveChangesAsync();
        return customer.Id;
    }

    public new async Task DisposeAsync() => await _postgres.DisposeAsync();
}
```

---

## 7.4 Authenticated requests in tests

```csharp
// Extension method for adding auth headers
public static class HttpClientExtensions
{
    public static HttpClient AsUser(this HttpClient client, string role = "User")
    {
        var token = GenerateTestToken(role);
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
        return client;
    }

    private static string GenerateTestToken(string role)
    {
        var claims = new[] { new Claim(ClaimTypes.Role, role) };
        var key    = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("test-secret-key-256-bits!!!!!!!"));
        var creds  = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token  = new JwtSecurityToken(claims: claims, expires: DateTime.UtcNow.AddHours(1), signingCredentials: creds);
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}

// Usage in tests
var response = await _client.AsUser("Admin").DeleteAsync($"/api/products/{id}");
```

---

## 7.5 Test patterns and organisation

```csharp
// Arrange-Act-Assert (AAA) — always follow this structure
[Fact]
public async Task Example()
{
    // Arrange — set up state
    var order = Order.Create(Guid.NewGuid());

    // Act — do the thing
    order.Confirm();  // NOTE: this will throw because order is empty — intentional

    // Assert — verify outcome
    order.Status.Should().Be(OrderStatus.Confirmed);
}

// Builder pattern for complex test data
public class OrderBuilder
{
    private Guid _customerId = Guid.NewGuid();
    private readonly List<(Guid, int, decimal)> _lines = [];

    public OrderBuilder ForCustomer(Guid customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithLine(Guid productId, int qty, decimal price)
    {
        _lines.Add((productId, qty, price));
        return this;
    }

    public OrderBuilder WithLines(int count = 1)
    {
        for (int i = 0; i < count; i++)
            _lines.Add((Guid.NewGuid(), i + 1, (i + 1) * 10m));
        return this;
    }

    public Order Build()
    {
        var order = Order.Create(_customerId);
        foreach (var (pid, qty, price) in _lines)
            order.AddLine(pid, $"Product {pid}", qty, price);
        return order;
    }

    public static OrderBuilder Default() => new OrderBuilder().WithLines();
}

// Usage
var order = new OrderBuilder()
    .ForCustomer(customerId)
    .WithLine(productId, qty: 2, price: 49.99m)
    .Build();
```

---

## 7.6 Testing tips

| Tip | Why |
|-----|-----|
| Test behaviour, not implementation | Tests survive refactoring |
| One assert concept per test | Failures are pinpointed |
| Name tests as `Method_Scenario_ExpectedResult` | Tests document the system |
| Use `FluentAssertions` over raw `Assert` | Better failure messages |
| Prefer `NSubstitute` over `Moq` | Simpler, fewer surprises |
| Use `TestContainers` for DB tests | Tests against real engine |
| Seed minimal data per test | Tests stay independent |
| Run tests in parallel with `[Collection]` | Isolate shared state |