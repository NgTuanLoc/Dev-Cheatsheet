---
tags: [dotnet, intermediate, testing]
aliases: [xUnit, Moq, FluentAssertions, Integration Tests]
level: Intermediate
---

# Testing

> **One-liner**: **xUnit** is the dominant test framework, **Moq** (or **NSubstitute**) for mocks, **FluentAssertions** for readable expectations, **WebApplicationFactory** for in-process integration tests against ASP.NET Core.

---

## Quick Reference

| Tool | Purpose |
|------|---------|
| `xunit` | Test framework |
| `Microsoft.NET.Test.Sdk` | Test runner integration |
| `xunit.runner.visualstudio` | VS / `dotnet test` runner |
| `Moq` / `NSubstitute` | Mocking |
| `FluentAssertions` | `result.Should().Be(...)` |
| `Bogus` | Fake data |
| `Testcontainers` | Real DB/Redis in tests |
| `WebApplicationFactory<TStartup>` | In-process API tests |
| `Microsoft.EntityFrameworkCore.InMemory` | In-memory EF (limited) |

| Attribute | Use |
|-----------|-----|
| `[Fact]` | parameterless test |
| `[Theory]` + `[InlineData(...)]` | parameterized |
| `[ClassData]` / `[MemberData]` | custom data sources |
| `IClassFixture<T>` | per-class shared setup |
| `ICollectionFixture<T>` | cross-class shared setup |
| `[Skip]` (xUnit v3) | conditional skip |
| `[Trait("Category", "...")]` | filter |

---

## Core Concept

A unit test verifies one piece of code in isolation, fast and deterministic. It follows the **AAA pattern**: Arrange, Act, Assert.

In .NET, **xUnit** uses constructor-as-setup and `IDisposable` (or `IAsyncLifetime`) as teardown — no `[SetUp]`/`[TearDown]` attributes. A new test class instance is created per test, so fields are isolated automatically.

For integration tests of ASP.NET Core APIs, **`WebApplicationFactory<T>`** spins up the whole pipeline in memory and gives you an `HttpClient` that talks to it — no network, no Kestrel. Combine with **Testcontainers** for real Postgres/Redis instead of mocks.

---

## Syntax & API

### Project setup
```bash
dotnet new xunit -n Shop.Tests
dotnet add Shop.Tests reference Shop
dotnet add Shop.Tests package Moq
dotnet add Shop.Tests package FluentAssertions
dotnet add Shop.Tests package Microsoft.AspNetCore.Mvc.Testing
```

### Basic test
```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_TwoPositive_ReturnsSum()
    {
        // Arrange
        var calc = new Calculator();

        // Act
        var result = calc.Add(2, 3);

        // Assert
        result.Should().Be(5);
    }
}
```

### Theory (parameterized)
```csharp
public class ValidatorTests
{
    [Theory]
    [InlineData("",       false)]
    [InlineData("a@b",    false)]
    [InlineData("a@b.co", true)]
    public void IsValidEmail(string input, bool expected)
    {
        EmailValidator.IsValid(input).Should().Be(expected);
    }

    [Theory]
    [MemberData(nameof(BadInputs))]
    public void Reject(string input) =>
        Validator.Validate(input).Should().BeFalse();

    public static IEnumerable<object[]> BadInputs => new[]
    {
        new object[] { "" },
        new object[] { " " },
        new object[] { new string('a', 1000) },
    };
}
```

### Async tests
```csharp
[Fact]
public async Task GetUser_ReturnsUser()
{
    var svc = new UserService();
    var u = await svc.GetByIdAsync(1);
    u.Should().NotBeNull();
}
```

### Mocking with Moq
```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task Place_ChargesUser_AndSavesOrder()
    {
        var users = new Mock<IUserRepository>();
        users.Setup(r => r.GetByIdAsync(1))
             .ReturnsAsync(new User { Id = 1, Balance = 100 });

        var saved = false;
        var orders = new Mock<IOrderRepository>();
        orders.Setup(r => r.SaveAsync(It.IsAny<Order>()))
              .Callback<Order>(_ => saved = true)
              .Returns(Task.CompletedTask);

        var svc = new OrderService(users.Object, orders.Object);
        await svc.PlaceAsync(1, new OrderDto(50));

        saved.Should().BeTrue();
        users.Verify(r => r.GetByIdAsync(1), Times.Once);
        orders.Verify(r => r.SaveAsync(It.Is<Order>(o => o.Total == 50)), Times.Once);
    }
}
```

### NSubstitute (alternative — slightly nicer syntax)
```csharp
var users = Substitute.For<IUserRepository>();
users.GetByIdAsync(1).Returns(new User { Id = 1 });

await users.Received(1).GetByIdAsync(1);
```

### Shared setup with IClassFixture
```csharp
public class DbFixture : IAsyncLifetime
{
    public ShopContext Db { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        var opts = new DbContextOptionsBuilder<ShopContext>()
            .UseInMemoryDatabase("test").Options;
        Db = new ShopContext(opts);
        await Db.Database.EnsureCreatedAsync();
    }

    public Task DisposeAsync() => Db.DisposeAsync().AsTask();
}

public class UserRepoTests : IClassFixture<DbFixture>
{
    private readonly DbFixture _fix;
    public UserRepoTests(DbFixture fix) => _fix = fix;

    [Fact]
    public async Task Add_Persists() { /* ... */ }
}
```

### Integration test with WebApplicationFactory
```csharp
public class UsersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UsersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(b => b.ConfigureServices(s =>
        {
            // Replace real services with fakes
            var dbDescriptor = s.Single(d => d.ServiceType == typeof(DbContextOptions<ShopContext>));
            s.Remove(dbDescriptor);
            s.AddDbContext<ShopContext>(o => o.UseInMemoryDatabase("test"));
        })).CreateClient();
    }

    [Fact]
    public async Task Get_ReturnsUsers()
    {
        var resp = await _client.GetAsync("/api/v1/users");
        resp.StatusCode.Should().Be(HttpStatusCode.OK);
        var body = await resp.Content.ReadFromJsonAsync<List<User>>();
        body.Should().NotBeNull();
    }
}
```

### Testcontainers — real Postgres
```csharp
public class PostgresFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } =
        new PostgreSqlBuilder().WithImage("postgres:16").Build();

    public Task InitializeAsync() => Container.StartAsync();
    public Task DisposeAsync() => Container.DisposeAsync().AsTask();
}
```

### dotnet test
```bash
dotnet test
dotnet test --filter "FullyQualifiedName~Calculator"
dotnet test --filter "Category=Slow"
dotnet test --logger "console;verbosity=detailed"
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
```

---

## Common Patterns

```csharp
// Pattern: object mother for test data
public static class Fixtures
{
    public static User AUser(string? email = null) =>
        new() { Email = email ?? "alice@x.com", Name = "Alice" };

    public static Order AnOrder(decimal total = 100m, int userId = 1) =>
        new() { UserId = userId, Total = total };
}

[Fact]
public void Process()
{
    var order = Fixtures.AnOrder(total: 50);
    // ...
}
```

```csharp
// Pattern: dispose for per-test cleanup
public class TempFileTests : IDisposable
{
    private readonly string _file = Path.GetTempFileName();

    [Fact] public void WritesAndReads() { /* ... */ }

    public void Dispose() => File.Delete(_file);
}
```

```csharp
// Pattern: prevent test interdependencies — fresh DI per test
public sealed class TestApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder b) =>
        b.ConfigureServices(s => { /* swap real for fake */ });
}
```

---

## Gotchas & Tips

- **Tests must be independent** — no shared mutable state. xUnit creates a new class instance per test, but statics survive.
- **`InMemoryDatabase` differs from real SQL** — no FK enforcement, no transactions, no concurrency conflicts. Use Testcontainers for high-fidelity tests.
- **Don't test private methods directly** — test the public behavior. If the private logic is too rich to be covered indirectly, extract a class.
- **Mock interfaces, not concrete classes** — concrete mocking requires `virtual` (Moq) or runtime interception. Cleaner to depend on abstractions.
- **One assert per test is a guideline, not a rule** — multiple assertions on the same act are fine. Each test should verify one *behavior*.
- **Always use `Should()`** with FluentAssertions — failure messages are far better than `Assert.Equal`.
- **Async test methods returning `void` are silently skipped** — must return `Task` (xUnit detects this; some other runners don't).
- **Use `[Trait]` to tag slow/integration tests** — run unit tests on every save, integration tests in CI only.
- **Test naming**: `Method_Scenario_Expected` — `Withdraw_NegativeAmount_Throws`.

---

## See Also

- [[02 - Clean Architecture]]
- [[14 - ASP.NET Core Basics]]
- [[16 - Entity Framework Core]]
- [[01 - Design Patterns]]
