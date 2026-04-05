# 6. Architecture Patterns

## 6.1 Clean Architecture

The dependency rule: all dependencies point inward. Domain is the centre — it knows nothing about infrastructure, HTTP, or databases.

```
Api  →  Application  →  Domain
         ↑
   Infrastructure
```

### Domain layer (no dependencies)

```csharp
// Aggregate root
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _events = [];
    public IReadOnlyList<IDomainEvent> DomainEvents => _events;

    protected void AddEvent(IDomainEvent @event) => _events.Add(@event);
    public void ClearEvents() => _events.Clear();
}

// Rich domain entity
public class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = [];

    private Order() { } // EF

    public Guid         Id         { get; private set; }
    public Guid         CustomerId { get; private set; }
    public OrderStatus  Status     { get; private set; }
    public DateTime     CreatedAt  { get; private set; }
    public decimal      Total      => _lines.Sum(l => l.LineTotal);
    public IReadOnlyList<OrderLine> Lines => _lines;

    public static Order Create(Guid customerId)
    {
        var order = new Order
        {
            Id         = Guid.NewGuid(),
            CustomerId = customerId,
            Status     = OrderStatus.Pending,
            CreatedAt  = DateTime.UtcNow
        };
        order.AddEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddLine(Guid productId, string name, int qty, decimal unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot add lines to a non-pending order.");

        _lines.Add(new OrderLine(productId, name, qty, unitPrice));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Order is not in pending status.");
        if (_lines.Count == 0)
            throw new DomainException("Cannot confirm an empty order.");

        Status = OrderStatus.Confirmed;
        AddEvent(new OrderConfirmedEvent(Id, Total));
    }
}

// Domain service (logic that spans multiple aggregates)
public class PricingService
{
    public decimal CalculateDiscount(Customer customer, Order order)
    {
        if (customer.IsPremium && order.Total > 500)
            return order.Total * 0.10m;
        return 0;
    }
}

// Repository interface (defined in domain, implemented in infrastructure)
public interface IOrderRepository
{
    Task<Order?>              GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default);
    void Add(Order order);
    Task SaveChangesAsync(CancellationToken ct = default);
}
```

### Application layer (orchestrates domain)

```csharp
// MediatR command
public record CreateOrderCommand(Guid CustomerId, List<OrderLineDto> Lines)
    : IRequest<Result<Guid>>;

// Handler
public class CreateOrderHandler(
    IOrderRepository orders,
    ICustomerRepository customers,
    IPublisher publisher) : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateOrderCommand cmd, CancellationToken ct)
    {
        var customer = await customers.GetByIdAsync(cmd.CustomerId, ct);
        if (customer is null)
            return Result.Failure<Guid>(Errors.Customer.NotFound(cmd.CustomerId));

        var order = Order.Create(customer.Id);

        foreach (var line in cmd.Lines)
            order.AddLine(line.ProductId, line.ProductName, line.Quantity, line.UnitPrice);

        orders.Add(order);
        await orders.SaveChangesAsync(ct);

        // Publish domain events
        foreach (var @event in order.DomainEvents)
            await publisher.Publish(@event, ct);
        order.ClearEvents();

        return Result.Success(order.Id);
    }
}

// Validator
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty().WithMessage("Order must have at least one line");
        RuleForEach(x => x.Lines).SetValidator(new OrderLineDtoValidator());
    }
}
```

---

## 6.2 CQRS with MediatR

Separate reads from writes. Commands mutate state; queries return data.

```csharp
// Install: dotnet add package MediatR

// Register
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<CreateOrderCommand>());

// Add pipeline behaviors (ordering matters)
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));

// Validation pipeline behavior
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Logging behavior
public class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var name = typeof(TRequest).Name;
        logger.LogInformation("Handling {Request}", name);
        var sw = Stopwatch.StartNew();

        var response = await next();

        logger.LogInformation("Handled {Request} in {Elapsed}ms", name, sw.ElapsedMilliseconds);
        return response;
    }
}

// Query
public record GetOrderQuery(Guid OrderId) : IRequest<Result<OrderResponse>>;

public class GetOrderHandler(IOrderReadRepository readRepo)
    : IRequestHandler<GetOrderQuery, Result<OrderResponse>>
{
    public async Task<Result<OrderResponse>> Handle(GetOrderQuery query, CancellationToken ct)
    {
        var order = await readRepo.GetByIdAsync(query.OrderId, ct);
        return order is null
            ? Result.Failure<OrderResponse>(Errors.Order.NotFound(query.OrderId))
            : Result.Success(OrderResponse.From(order));
    }
}

// In endpoint
app.MapGet("/orders/{id:guid}", async (Guid id, IMediator mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new GetOrderQuery(id), ct);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound();
});
```

---

## 6.3 Result pattern (no exceptions for control flow)

```csharp
// Result type
public class Result<T>
{
    public bool     IsSuccess { get; }
    public bool     IsFailure => !IsSuccess;
    public T?       Value     { get; }
    public Error    Error     { get; }

    private Result(T value)           { IsSuccess = true;  Value = value; }
    private Result(Error error)       { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value)  => new(value);
    public static Result<T> Failure(Error e)  => new(e);

    public Result<TOut> Map<TOut>(Func<T, TOut> mapper)
        => IsSuccess ? Result<TOut>.Success(mapper(Value!)) : Result<TOut>.Failure(Error);

    public async Task<Result<TOut>> MapAsync<TOut>(Func<T, Task<TOut>> mapper)
        => IsSuccess
            ? Result<TOut>.Success(await mapper(Value!))
            : Result<TOut>.Failure(Error);
}

public record Error(string Code, string Message)
{
    public static Error None        => new("", "");
    public static Error NotFound(string entity, object id)
        => new($"{entity}.NotFound", $"{entity} with id '{id}' was not found.");
    public static Error Unauthorized()
        => new("Auth.Unauthorized", "You are not authorised to perform this action.");
    public static Error Conflict(string message)
        => new("Conflict", message);
}

// Usage
public async Task<Result<Order>> ConfirmOrderAsync(Guid orderId, CancellationToken ct)
{
    var order = await repo.GetByIdAsync(orderId, ct);
    if (order is null)
        return Result<Order>.Failure(Error.NotFound("Order", orderId));

    try
    {
        order.Confirm();
        await repo.SaveChangesAsync(ct);
        return Result<Order>.Success(order);
    }
    catch (DomainException ex)
    {
        return Result<Order>.Failure(new Error("Domain", ex.Message));
    }
}

// Map Result to IResult in endpoints
public static IResult ToHttpResult<T>(this Result<T> result)
    => result.IsSuccess
        ? Results.Ok(result.Value)
        : result.Error.Code.EndsWith("NotFound")
            ? Results.NotFound(result.Error)
            : Results.Problem(result.Error.Message);
```

---

## 6.4 Domain events

```csharp
// Domain event interface
public interface IDomainEvent : INotification { }

// Events
public record OrderCreatedEvent(Guid OrderId, Guid CustomerId) : IDomainEvent;
public record OrderConfirmedEvent(Guid OrderId, decimal Total)  : IDomainEvent;

// Event handlers (MediatR INotificationHandler)
public class SendOrderConfirmationEmail(IEmailService email, ICustomerRepository customers)
    : INotificationHandler<OrderConfirmedEvent>
{
    public async Task Handle(OrderConfirmedEvent @event, CancellationToken ct)
    {
        var customer = await customers.GetByOrderIdAsync(@event.OrderId, ct);
        if (customer is null) return;

        await email.SendAsync(new EmailMessage
        {
            To      = customer.Email,
            Subject = "Your order has been confirmed",
            Body    = $"Order total: {@event.Total:C}"
        }, ct);
    }
}

// Dispatch events after saving (in Unit of Work or SaveChanges interceptor)
public class DomainEventDispatcher(IPublisher publisher)
{
    public async Task DispatchAsync(IEnumerable<AggregateRoot> aggregates, CancellationToken ct)
    {
        var events = aggregates
            .SelectMany(a => a.DomainEvents)
            .ToList();

        foreach (var aggregate in aggregates)
            aggregate.ClearEvents();

        foreach (var @event in events)
            await publisher.Publish(@event, ct);
    }
}
```

---

## 6.5 Vertical slice architecture

Instead of grouping by layer, group by feature. All code for a feature (endpoint, handler, validator, query, response) lives in one folder.

```
Features/
└── Orders/
    ├── Create/
    │   ├── Endpoint.cs       — MapPost("/orders")
    │   ├── Command.cs        — CreateOrderCommand record
    │   ├── Handler.cs        — IRequestHandler
    │   ├── Validator.cs      — AbstractValidator
    │   └── Response.cs       — OrderCreatedResponse record
    ├── GetById/
    │   ├── Endpoint.cs
    │   ├── Query.cs
    │   ├── Handler.cs
    │   └── Response.cs
    └── Cancel/
        ├── Endpoint.cs
        ├── Command.cs
        └── Handler.cs
```

```csharp
// Features/Orders/Create/Handler.cs
public static class CreateOrder
{
    public record Command(Guid CustomerId, List<LineItem> Lines) : IRequest<Result<Guid>>;
    public record LineItem(Guid ProductId, int Quantity, decimal UnitPrice);
    public record Response(Guid OrderId);

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.CustomerId).NotEmpty();
            RuleFor(x => x.Lines).NotEmpty();
        }
    }

    public class Handler(IOrderRepository repo) : IRequestHandler<Command, Result<Guid>>
    {
        public async Task<Result<Guid>> Handle(Command cmd, CancellationToken ct)
        {
            var order = Order.Create(cmd.CustomerId);
            foreach (var line in cmd.Lines)
                order.AddLine(line.ProductId, "Product", line.Quantity, line.UnitPrice);
            repo.Add(order);
            await repo.SaveChangesAsync(ct);
            return Result.Success(order.Id);
        }
    }

    public static void Map(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (Command cmd, IMediator mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", new Response(result.Value))
                : result.ToHttpResult();
        })
        .WithTags("Orders")
        .RequireAuthorization();
    }
}
```