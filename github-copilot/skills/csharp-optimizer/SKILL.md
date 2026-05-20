---
name: csharp
description: >
  Expert C# and .NET development skill. Use this whenever the user asks to write, review,
  refactor, debug, or explain C# code — including ASP.NET Core APIs, Entity Framework,
  LINQ, async/await patterns, dependency injection, unit testing, Blazor, WPF, console
  apps, class libraries, NuGet packages, or any .NET-related task. Trigger on keywords
  like "C#", ".NET", "csharp", "dotnet", "ASP.NET", "Entity Framework", "LINQ",
  "Blazor", "xUnit", "NUnit", "Roslyn", or any file ending in .cs, .csproj, .sln.
  Also trigger when the user pastes C# code and asks for help, even without explicit
  language mention. Always prefer this skill over generic code generation for C# tasks.
---

# C# Skill

Comprehensive guidance for writing idiomatic, production-grade C# and .NET code.
Covers .NET 8/9 (LTS), modern language features through C# 13, and the full ecosystem.

---

## Quick reference: what to read first

| Task | Go to |
|------|-------|
| Starting a new project | [Project setup](#project-setup) |
| Writing a class or record | [Type design](#type-design) |
| Async code | [Async/await patterns](#asyncawait-patterns) |
| Database / EF Core | [Entity Framework Core](#entity-framework-core) |
| ASP.NET Core API | [ASP.NET Core](#aspnet-core) |
| Unit tests | [Testing](#testing) |
| LINQ queries | [LINQ](#linq) |
| DI / IoC | [Dependency injection](#dependency-injection) |
| Error handling | [Error handling](#error-handling) |
| Performance | [Performance](#performance) |

---

## Guiding principles

1. **Correctness first** — compile cleanly with no warnings (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`).
2. **Idiomatic C#** — use the language version the project targets; never downgrade syntax.
3. **Minimal surface area** — prefer `internal` over `public`; expose only what callers need.
4. **Nullable safe** — enable `<Nullable>enable</Nullable>` in every project; annotate all public APIs.
5. **Async all the way** — never block on async code (`Task.Result`, `.Wait()`) in library or web code.
6. **Immutability by default** — prefer `record`, `readonly struct`, `init`-only properties.

---

## Project setup

### Minimal .csproj (library)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

### Minimal .csproj (ASP.NET Core Web API)

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

### editorconfig baseline

Always include at project root to enforce code style:

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
dotnet_sort_system_directives_first = true
csharp_new_line_before_open_brace = all
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
```

---

## Type design

### Prefer records for data

```csharp
// Immutable data transfer object
public sealed record CreateOrderRequest(
    Guid CustomerId,
    IReadOnlyList<OrderLineItem> Items,
    string? CouponCode = null);

// Mutable record with init-only setters
public record Product
{
    public required Guid Id { get; init; }
    public required string Name { get; init; }
    public decimal Price { get; init; }
}
```

### Use `required` members (C# 11+)

```csharp
public class CustomerService
{
    public required ICustomerRepository Repository { get; init; }
    public required ILogger<CustomerService> Logger { get; init; }
}
```

### Readonly structs for value semantics

```csharp
public readonly struct Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch");
        return new(a.Amount + b.Amount, a.Currency);
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

### Sealed classes by default

Mark leaf classes `sealed` to prevent unintended inheritance and enable JIT devirtualisation:

```csharp
public sealed class OrderProcessor(IOrderRepository repo, IEventBus bus)
{
    // implementation
}
```

---

## Async/await patterns

### Core rules

- Return `Task` or `Task<T>` from public async methods (never `async void` except event handlers).
- Accept `CancellationToken` in every I/O method signature.
- Use `ConfigureAwait(false)` in libraries (not in ASP.NET Core or Blazor app code).
- Prefer `ValueTask<T>` for hot paths that often complete synchronously.

```csharp
// Library method — use ConfigureAwait(false)
public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct = default)
{
    await using var conn = await _connectionFactory.OpenAsync(ct).ConfigureAwait(false);
    return await conn.QuerySingleOrDefaultAsync<Order>(
        "SELECT * FROM Orders WHERE Id = @id", new { id }, cancellationToken: ct)
        .ConfigureAwait(false);
}

// App/controller method — omit ConfigureAwait
public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
{
    var order = await _orderService.GetOrderAsync(id, ct);
    return order is null ? NotFound() : Ok(order);
}
```

### Parallel async with bounded concurrency

```csharp
// Process up to 4 items concurrently
var semaphore = new SemaphoreSlim(4);
var tasks = items.Select(async item =>
{
    await semaphore.WaitAsync(ct);
    try { return await ProcessAsync(item, ct); }
    finally { semaphore.Release(); }
});
var results = await Task.WhenAll(tasks);
```

### Fire-and-forget background work (ASP.NET Core)

```csharp
// Inject IHostedService or use BackgroundService — never fire-forget raw tasks in web apps
public class EmailBackgroundService(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = scopeFactory.CreateScope();
            var sender = scope.ServiceProvider.GetRequiredService<IEmailSender>();
            await sender.SendAsync(job, stoppingToken);
        }
    }
}
```

---

## LINQ

### Prefer method syntax for complex queries

```csharp
var summary = orders
    .Where(o => o.Status == OrderStatus.Completed && o.CreatedAt >= cutoff)
    .GroupBy(o => o.CustomerId)
    .Select(g => new CustomerSummary(
        CustomerId: g.Key,
        OrderCount: g.Count(),
        TotalSpent: g.Sum(o => o.Total)))
    .OrderByDescending(s => s.TotalSpent)
    .Take(100)
    .ToList();
```

### Avoid multiple enumeration

```csharp
// BAD — enumerates twice
if (source.Any() && source.First().IsValid) { ... }

// GOOD — enumerate once
var first = source.FirstOrDefault();
if (first is { IsValid: true }) { ... }
```

### Use query syntax for joins

```csharp
var result =
    from order in orders
    join customer in customers on order.CustomerId equals customer.Id
    where order.Total > 1000m
    select new { order.Id, customer.Name, order.Total };
```

---

## Error handling

### Result pattern (avoid exceptions for control flow)

```csharp
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool ok, T? value, string? error) =>
        (IsSuccess, Value, Error) = (ok, value, error);

    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}

// Usage
public async Task<Result<Order>> PlaceOrderAsync(CreateOrderRequest req, CancellationToken ct)
{
    if (!req.Items.Any())
        return Result<Order>.Fail("Order must contain at least one item.");

    var order = await _repo.CreateAsync(req, ct);
    return Result<Order>.Ok(order);
}
```

### Custom exceptions (only for exceptional cases)

```csharp
public sealed class DomainException(string message) : Exception(message);
public sealed class NotFoundException(string entity, object key)
    : Exception($"{entity} with key '{key}' was not found.");
```

### Global exception handling in ASP.NET Core

```csharp
// Program.cs
app.UseExceptionHandler(exceptionApp =>
    exceptionApp.Run(async ctx =>
    {
        var feature = ctx.Features.Get<IExceptionHandlerFeature>();
        var ex = feature?.Error;

        ctx.Response.StatusCode = ex switch
        {
            NotFoundException => StatusCodes.Status404NotFound,
            DomainException    => StatusCodes.Status422UnprocessableEntity,
            _                  => StatusCodes.Status500InternalServerError
        };

        await ctx.Response.WriteAsJsonAsync(new { error = ex?.Message });
    }));
```

---

## Dependency injection

### Registration patterns

```csharp
// Extension method per feature (keeps Program.cs clean)
public static class OrderingServiceExtensions
{
    public static IServiceCollection AddOrdering(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IOrderService, OrderService>();
        services.Configure<OrderOptions>(config.GetSection("Ordering"));
        return services;
    }
}

// Program.cs
builder.Services.AddOrdering(builder.Configuration);
```

### Lifetime rules

| Lifetime | Use for |
|----------|---------|
| `Singleton` | Stateless services, caches, configuration wrappers |
| `Scoped` | EF Core `DbContext`, unit-of-work, per-request state |
| `Transient` | Lightweight stateless utilities |

> Never inject `Scoped` into `Singleton` — use `IServiceScopeFactory` instead.

### Options pattern

```csharp
public sealed class OrderOptions
{
    public const string Section = "Ordering";
    public int MaxItemsPerOrder { get; init; } = 50;
    public TimeSpan ReservationTimeout { get; init; } = TimeSpan.FromMinutes(15);
}

// Inject
public class OrderService(IOptions<OrderOptions> opts)
{
    private readonly OrderOptions _opts = opts.Value;
}
```

---

## Entity Framework Core

### DbContext setup

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Entity configuration (Fluent API)

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Total).HasPrecision(18, 2).IsRequired();
        builder.HasMany(o => o.Items).WithOne().OnDelete(DeleteBehavior.Cascade);
        builder.HasIndex(o => o.CustomerId);
    }
}
```

### Query best practices

```csharp
// Use AsNoTracking for read-only queries
var orders = await _ctx.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Include(o => o.Items)
    .ToListAsync(ct);

// Projection — never select entire entity when you need a subset
var names = await _ctx.Customers
    .Where(c => c.IsActive)
    .Select(c => new { c.Id, c.FullName })
    .ToListAsync(ct);

// Avoid N+1 — always Include related data or project
```

### Migrations

```bash
# Add migration
dotnet ef migrations add AddOrderStatusIndex --project src/Data --startup-project src/Api

# Apply to DB
dotnet ef database update --project src/Data --startup-project src/Api

# Generate SQL script for production
dotnet ef migrations script --idempotent -o migrations.sql
```

---

## ASP.NET Core

### Minimal API (preferred for .NET 8+)

```csharp
var app = builder.Build();

var orders = app.MapGroup("/api/orders").RequireAuthorization();

orders.MapGet("/{id:guid}", async (Guid id, IOrderService svc, CancellationToken ct) =>
{
    var order = await svc.GetOrderAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
})
.WithName("GetOrder")
.Produces<Order>()
.Produces(404);

orders.MapPost("/", async (
    [FromBody] CreateOrderRequest req,
    IOrderService svc,
    CancellationToken ct) =>
{
    var result = await svc.PlaceOrderAsync(req, ct);
    return result.IsSuccess
        ? Results.CreatedAtRoute("GetOrder", new { id = result.Value!.Id }, result.Value)
        : Results.UnprocessableEntity(result.Error);
})
.WithName("CreateOrder")
.Produces<Order>(201)
.Produces(422);
```

### Validation with FluentValidation

```csharp
public sealed class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().WithMessage("At least one item required.");
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.Quantity).GreaterThan(0);
            item.RuleFor(i => i.ProductId).NotEmpty();
        });
    }
}

// Register
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
```

### Program.cs structure

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. Services
builder.Services.AddOrdering(builder.Configuration);
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// 2. Middleware (order matters)
if (app.Environment.IsDevelopment()) app.UseSwagger().UseSwaggerUI();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseExceptionHandler(...);

// 3. Endpoints
app.MapOrderEndpoints();

app.Run();
```

---

## Testing

### xUnit + FluentAssertions + NSubstitute

```xml
<!-- test.csproj -->
<PackageReference Include="xunit" Version="2.*" />
<PackageReference Include="FluentAssertions" Version="6.*" />
<PackageReference Include="NSubstitute" Version="5.*" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
```

### Unit test structure (AAA)

```csharp
public sealed class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceTests() => _sut = new OrderService(_repo);

    [Fact]
    public async Task PlaceOrderAsync_WithEmptyItems_ReturnsFailure()
    {
        // Arrange
        var req = new CreateOrderRequest(Guid.NewGuid(), []);

        // Act
        var result = await _sut.PlaceOrderAsync(req, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("at least one item");
        await _repo.DidNotReceive().CreateAsync(Arg.Any<CreateOrderRequest>(), Arg.Any<CancellationToken>());
    }

    [Theory]
    [InlineData(1)]
    [InlineData(5)]
    [InlineData(50)]
    public async Task PlaceOrderAsync_WithValidItems_CreatesOrder(int itemCount)
    {
        // Arrange
        var items = Enumerable.Range(0, itemCount)
            .Select(_ => new OrderLineItem(Guid.NewGuid(), 1))
            .ToList();
        var req = new CreateOrderRequest(Guid.NewGuid(), items);
        var expected = new Order { Id = Guid.NewGuid() };
        _repo.CreateAsync(req, Arg.Any<CancellationToken>()).Returns(expected);

        // Act
        var result = await _sut.PlaceOrderAsync(req, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().BeEquivalentTo(expected);
    }
}
```

### Integration test with WebApplicationFactory

```csharp
public sealed class OrderApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task GET_order_returns_404_when_not_found()
    {
        var client = factory.CreateClient();
        var response = await client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Performance

### Span and Memory for zero-allocation parsing

```csharp
public static bool TryParseOrderRef(ReadOnlySpan<char> input, out Guid id)
{
    // ORD-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    if (input.Length != 36 + 4 || !input.StartsWith("ORD-"))
    {
        id = default;
        return false;
    }
    return Guid.TryParse(input[4..], out id);
}
```

### String building

```csharp
// For known small strings: string interpolation (compiler-optimised in C# 10+)
var label = $"Order #{order.Id} — {order.Total:C}";

// For loops / large concatenation: StringBuilder or Span<char>
var sb = new StringBuilder(capacity: 256);
foreach (var item in items) sb.Append(item.Name).Append(", ");
```

### ArrayPool for large temporary buffers

```csharp
var pool = ArrayPool<byte>.Shared;
var buffer = pool.Rent(4096);
try
{
    var read = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    Process(buffer.AsSpan(0, read));
}
finally { pool.Return(buffer); }
```

### Benchmark with BenchmarkDotNet

```csharp
[MemoryDiagnoser]
public class ParserBenchmarks
{
    private readonly string _input = "ORD-" + Guid.NewGuid().ToString();

    [Benchmark(Baseline = true)]
    public bool ParseString() => TryParseOrderRef(_input, out _);

    [Benchmark]
    public bool ParseSpan() => TryParseOrderRef(_input.AsSpan(), out _);
}
// Run: dotnet run -c Release --project benchmarks/
```

---

## Common patterns reference

### Builder pattern

```csharp
public sealed class QueryBuilder
{
    private string _table = string.Empty;
    private readonly List<string> _conditions = [];
    private int? _limit;

    public QueryBuilder From(string table) { _table = table; return this; }
    public QueryBuilder Where(string condition) { _conditions.Add(condition); return this; }
    public QueryBuilder Limit(int n) { _limit = n; return this; }
    public string Build() =>
        $"SELECT * FROM {_table}" +
        (_conditions.Count > 0 ? " WHERE " + string.Join(" AND ", _conditions) : "") +
        (_limit.HasValue ? $" LIMIT {_limit}" : "");
}
```

### Strategy pattern with DI

```csharp
public interface IPaymentStrategy { string Method { get; } Task<PaymentResult> PayAsync(decimal amount, CancellationToken ct); }

// Register all implementations
builder.Services.AddScoped<IPaymentStrategy, StripePaymentStrategy>();
builder.Services.AddScoped<IPaymentStrategy, PayPalPaymentStrategy>();

// Resolve by key
public class PaymentRouter(IEnumerable<IPaymentStrategy> strategies)
{
    public IPaymentStrategy For(string method) =>
        strategies.FirstOrDefault(s => s.Method == method)
        ?? throw new InvalidOperationException($"No payment strategy for '{method}'.");
}
```

---

## Naming conventions (quick reference)

| Element | Convention | Example |
|---------|-----------|---------|
| Class / Record / Struct | PascalCase | `OrderProcessor` |
| Interface | `I` + PascalCase | `IOrderRepository` |
| Enum | PascalCase (singular type, plural values only for flags) | `OrderStatus.Pending` |
| Async method | Suffix `Async` | `GetOrderAsync` |
| Private field | `_camelCase` | `_orderRepository` |
| Constant | PascalCase | `MaxRetryCount` |
| Local variable | camelCase | `orderCount` |
| Generic type param | `T` or descriptive `TEntity` | `TResult` |

---

## Misconfiguration checklist

Before generating or reviewing any C# code, verify:

- [ ] `<Nullable>enable</Nullable>` is set — no `?` annotations missing from public APIs
- [ ] `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` — no suppressed warnings
- [ ] No `.Result` or `.Wait()` on `Task` in library/web code
- [ ] `CancellationToken` flows through all async I/O calls
- [ ] EF Core queries use `AsNoTracking()` for read-only paths
- [ ] No magic strings — use constants, `nameof()`, or strongly-typed options
- [ ] Secrets are not hardcoded — reference `IConfiguration`, env vars, or secret manager
- [ ] `DbContext` is registered as `Scoped`, never `Singleton`
- [ ] No `catch (Exception ex)` without re-throw or structured logging
- [ ] Integration tests use a separate test database or in-memory provider

---

## Useful CLI commands

```bash
# New solution
dotnet new sln -n MyApp

# Add projects
dotnet new webapi -n MyApp.Api -o src/Api
dotnet new classlib -n MyApp.Core -o src/Core
dotnet new xunit -n MyApp.Tests -o tests/Unit
dotnet sln add src/Api src/Core tests/Unit

# Add package
dotnet add src/Api package Microsoft.EntityFrameworkCore.SqlServer

# Format
dotnet format

# Lint / analysis
dotnet build -warnaserror

# Test with coverage
dotnet test --collect:"XPlat Code Coverage"
```