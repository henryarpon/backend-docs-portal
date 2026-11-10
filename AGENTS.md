You are an expert in C#, ASP.NET Core and scalable backend development. You write clean, maintainable, testable and secure code following .NET best practices, Clean Architecture and CQRS with MediatR.

## Tech Stack

- .NET 10 (or the solution's target framework), C# latest
- ASP.NET Core Web API (controllers)
- MediatR for CQRS (commands, queries, pipeline behaviors)
- FluentValidation for request validation
- Entity Framework Core with SQL Server
- xUnit for tests, NSubstitute for test doubles

## C# Best Practices

- `Nullable` is enabled; never suppress warnings with `!` unless the value is proven non-null
- Use file-scoped namespaces
- Use `record` for commands, queries and DTOs
- Mark classes `sealed` unless they are designed for inheritance
- Prefer `var` when the type is obvious from the right-hand side
- Use async all the way: every I/O method is `async` and accepts a `CancellationToken`
- Never use `.Result`, `.Wait()` or `async void`
- Use `TimeProvider` instead of `DateTime.Now` / `DateTime.UtcNow` so time can be faked in tests
- Store dates in UTC
- Use structured logging with `ILogger<T>` and message templates: `logger.LogInformation("Created customer {CustomerId}", id)`, never string interpolation

## Clean Architecture

The solution has four layers. Dependencies point inward only: Api → Application → Domain. Infrastructure implements interfaces defined in Application.

| Layer | Project | Contains | May reference | Must NOT |
| --- | --- | --- | --- | --- |
| Domain | `ProjectName.Domain` | Entities, value objects, enums, domain events, domain exceptions | Nothing | Reference EF Core, ASP.NET Core, MediatR or any other project |
| Application | `ProjectName.Application` | Commands, queries, handlers, validators, DTOs, interfaces, pipeline behaviors | Domain | Reference Infrastructure or Api, or call external services directly |
| Infrastructure | `ProjectName.Infrastructure` | `DbContext`, EF configurations, migrations, external service clients (Service Bus, Redis, Blob, email) | Application, Domain | Contain business rules |
| Api | `ProjectName.Api` | Controllers, middleware, `Program.cs`, DI composition | Application, Infrastructure (only for DI registration) | Contain business logic or use `DbContext` directly |

- An Azure Functions host, if added, is another presentation project like `ProjectName.Api`: it sends commands and queries through MediatR and holds no business logic.
- Each project exposes one `DependencyInjection.cs` with an extension method (`AddApplication()`, `AddInfrastructure(configuration)`) that `Program.cs` calls.

## Solution Structure

```
ProjectName.sln
├── src/
│   ├── ProjectName.Domain/
│   │   ├── Common/                      # BaseEntity, IAuditable, DomainEvent base
│   │   ├── Entities/                    # Customer.cs, Order.cs
│   │   ├── ValueObjects/                # Email.cs, Money.cs
│   │   ├── Enums/
│   │   ├── Events/                      # CustomerCreatedEvent.cs
│   │   └── Exceptions/                  # DomainException.cs
│   │
│   ├── ProjectName.Application/
│   │   ├── Common/
│   │   │   ├── Behaviors/               # ValidationBehavior, LoggingBehavior, PerformanceBehavior
│   │   │   ├── Exceptions/              # NotFoundException, ValidationException, ForbiddenException
│   │   │   ├── Interfaces/              # IApplicationDbContext, ICurrentUser, IEmailSender
│   │   │   └── Models/                  # PagedResult<T>
│   │   ├── Features/
│   │   │   └── Customers/
│   │   │       ├── Commands/
│   │   │       │   ├── CreateCustomer/
│   │   │       │   │   ├── CreateCustomerCommand.cs
│   │   │       │   │   ├── CreateCustomerCommandHandler.cs
│   │   │       │   │   └── CreateCustomerCommandValidator.cs
│   │   │       │   └── UpdateCustomer/
│   │   │       └── Queries/
│   │   │           ├── GetCustomerById/
│   │   │           │   ├── GetCustomerByIdQuery.cs
│   │   │           │   ├── GetCustomerByIdQueryHandler.cs
│   │   │           │   └── CustomerDto.cs
│   │   │           └── GetCustomers/
│   │   └── DependencyInjection.cs
│   │
│   ├── ProjectName.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── Configurations/          # CustomerConfiguration.cs (IEntityTypeConfiguration<T>)
│   │   │   ├── Interceptors/            # AuditableEntityInterceptor.cs
│   │   │   └── Migrations/
│   │   ├── Services/                    # EmailSender.cs, CurrentUser.cs
│   │   ├── Messaging/                   # Service Bus publishers
│   │   ├── Caching/                     # Redis cache service
│   │   └── DependencyInjection.cs
│   │
│   └── ProjectName.Api/
│       ├── Controllers/                 # CustomersController.cs
│       ├── Middleware/                  # GlobalExceptionHandler.cs
│       ├── appsettings.json
│       └── Program.cs
│
└── tests/
    ├── ProjectName.Domain.UnitTests/
    ├── ProjectName.Application.UnitTests/
    └── ProjectName.Api.IntegrationTests/
```

## Domain Layer

- Entities protect their own rules: private setters, changes only through methods
- Create entities with constructors or static factory methods that validate input
- Keep the domain free of EF Core attributes; mapping lives in Infrastructure configurations
- Use value objects for concepts with rules (email, money, date range)
- Throw `DomainException` (or a subclass) when a business rule is broken

```csharp
namespace ProjectName.Domain.Entities;

public sealed class Customer : BaseEntity
{
    private Customer() { } // for EF Core

    public string Name { get; private set; } = string.Empty;
    public Email Email { get; private set; } = null!;
    public bool IsActive { get; private set; }

    public static Customer Create(string name, Email email)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new DomainException("Customer name is required.");

        return new Customer { Name = name.Trim(), Email = email, IsActive = true };
    }

    public void Deactivate() => IsActive = false;
}
```

## CQRS with MediatR

Every use case is either a **command** (changes state) or a **query** (reads state). Never both.

### Commands

- Named `<Verb><Noun>Command`: `CreateCustomerCommand`, `DeactivateCustomerCommand`
- Return the new entity's id, `Unit`, or a small result record. Never return entities.
- Every command has a FluentValidation validator
- Load, change and save through domain methods; call `SaveChangesAsync` once per command

```csharp
namespace ProjectName.Application.Features.Customers.Commands.CreateCustomer;

public sealed record CreateCustomerCommand(string Name, string Email) : IRequest<Guid>;

public sealed class CreateCustomerCommandValidator : AbstractValidator<CreateCustomerCommand>
{
    public CreateCustomerCommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
    }
}

public sealed class CreateCustomerCommandHandler(IApplicationDbContext db)
    : IRequestHandler<CreateCustomerCommand, Guid>
{
    public async Task<Guid> Handle(CreateCustomerCommand request, CancellationToken cancellationToken)
    {
        var customer = Customer.Create(request.Name, Email.From(request.Email));

        db.Customers.Add(customer);
        await db.SaveChangesAsync(cancellationToken);

        return customer.Id;
    }
}
```

### Queries

- Named `Get<Noun>Query` / `Get<Noun>sQuery`: `GetCustomerByIdQuery`, `GetCustomersQuery`
- Read only: never call `SaveChangesAsync` or change entities
- Use `AsNoTracking()` and project straight to DTOs with `Select`; never load full entities just to map them
- Paged lists return `PagedResult<T>`; always apply a maximum page size
- Throw `NotFoundException` when a requested item does not exist

```csharp
namespace ProjectName.Application.Features.Customers.Queries.GetCustomerById;

public sealed record GetCustomerByIdQuery(Guid Id) : IRequest<CustomerDto>;

public sealed record CustomerDto(Guid Id, string Name, string Email, bool IsActive);

public sealed class GetCustomerByIdQueryHandler(IApplicationDbContext db)
    : IRequestHandler<GetCustomerByIdQuery, CustomerDto>
{
    public async Task<CustomerDto> Handle(GetCustomerByIdQuery request, CancellationToken cancellationToken)
    {
        return await db.Customers
            .AsNoTracking()
            .Where(c => c.Id == request.Id)
            .Select(c => new CustomerDto(c.Id, c.Name, c.Email.Value, c.IsActive))
            .FirstOrDefaultAsync(cancellationToken)
            ?? throw new NotFoundException(nameof(Customer), request.Id);
    }
}
```

### Handler rules

- One handler per request; command, handler, validator and DTO live in the same feature folder
- Handlers do NOT call other handlers through `ISender`. Share logic through domain methods or an application service instead
- Handlers depend on interfaces from `Application/Common/Interfaces`, never on Infrastructure classes
- Map with explicit code (`Select` projections, small static `ToDto()` methods); do NOT use reflection-based mapping libraries in new code

### Pipeline behaviors

Registered in `Application/DependencyInjection.cs`, in this order:

1. `LoggingBehavior`: logs request name and duration
2. `ValidationBehavior`: runs all validators for the request and throws `ValidationException` with every failure
3. `PerformanceBehavior`: logs a warning for requests slower than 500 ms

Cross-cutting concerns (validation, logging, transactions, caching) go in behaviors, not in handlers.

## Persistence (EF Core)

- `IApplicationDbContext` in Application exposes the `DbSet<T>` properties handlers need plus `SaveChangesAsync`; `ApplicationDbContext` in Infrastructure implements it
- Configure every entity in its own `IEntityTypeConfiguration<T>` class; no data annotations on entities
- Set explicit column lengths, required fields and indexes in configurations
- Audit fields (`CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`) are set by `AuditableEntityInterceptor`, not by handlers
- No lazy loading. Use `Include` only when a command needs the related data
- Name migrations by what they change: `dotnet ef migrations add AddCustomerEmailIndex -p src/ProjectName.Infrastructure -s src/ProjectName.Api`
- Never edit a migration that has been applied to a shared environment; add a new one

## API Layer

- Controllers are thin: map the HTTP request to a command or query, call `ISender.Send`, return the result
- No business logic, no `DbContext`, no repositories in controllers
- Every action is `async`, accepts `CancellationToken` and returns `ActionResult<T>` (or `IActionResult`)
- Use correct status codes: `201 Created` with a location for creates, `204 No Content` for updates and deletes
- Document every action with `[ProducesResponseType]`

```csharp
namespace ProjectName.Api.Controllers;

[ApiController]
[Route("api/customers")]
public sealed class CustomersController(ISender sender) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType<CustomerDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CustomerDto>> GetById(Guid id, CancellationToken cancellationToken)
        => Ok(await sender.Send(new GetCustomerByIdQuery(id), cancellationToken));

    [HttpPost]
    [ProducesResponseType<Guid>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<Guid>> Create(CreateCustomerCommand command, CancellationToken cancellationToken)
    {
        var id = await sender.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id }, id);
    }
}
```

## Error Handling

- One global exception handler (`IExceptionHandler`) converts exceptions to RFC 7807 `ProblemDetails`:

| Exception | Status |
| --- | --- |
| `ValidationException` | 400, with the field errors |
| `DomainException` | 422 |
| `NotFoundException` | 404 |
| `ForbiddenException` | 403 |
| Anything else | 500, logged, with no internal details in the response |

- Do NOT wrap handler code in try/catch just to log and rethrow
- Never return stack traces or exception messages from unexpected errors to clients

## Configuration and Security

- Bind settings with the options pattern (`IOptions<T>`) and validate them on startup with `ValidateOnStart()`
- Secrets never go in `appsettings.json` or source control: use user-secrets locally and Azure Key Vault in deployed environments
- Every endpoint requires authorization unless explicitly marked `[AllowAnonymous]`
- Get the current user through `ICurrentUser`, never by reading `HttpContext` in Application

## Naming Conventions

| Item | Name | Location |
| --- | --- | --- |
| Command | `CreateCustomerCommand` | `Features/Customers/Commands/CreateCustomer/` |
| Command handler | `CreateCustomerCommandHandler` | Same folder |
| Validator | `CreateCustomerCommandValidator` | Same folder |
| Query | `GetCustomerByIdQuery` | `Features/Customers/Queries/GetCustomerById/` |
| Query handler | `GetCustomerByIdQueryHandler` | Same folder |
| DTO | `CustomerDto` | Folder of the query that returns it |
| Entity | `Customer` | `Domain/Entities/` |
| EF configuration | `CustomerConfiguration` | `Infrastructure/Persistence/Configurations/` |
| Controller | `CustomersController` | `Api/Controllers/` |
| Interface | `IEmailSender` | `Application/Common/Interfaces/` |

- One public type per file; file name matches the type name
- Folder structure matches namespaces
- Feature folders are plural nouns (`Customers`, `Orders`)

## Testing

- **Domain unit tests**: entity methods and value objects, no mocks needed
- **Application unit tests**: handlers and validators, with test doubles for interfaces
- **API integration tests**: `WebApplicationFactory` against a real SQL Server test database (for example with Testcontainers), covering the HTTP contract and status codes
- Test names follow `Method_Scenario_ExpectedResult`: `Handle_EmailAlreadyExists_ThrowsValidationException`
- Every new command or query gets at least one handler test and one validator test

## Before Finishing a Task

- [ ] No business logic in controllers or Infrastructure
- [ ] Domain project references no other project or framework package
- [ ] Every new command has a validator
- [ ] Queries use `AsNoTracking()` and project to DTOs; no entity is returned from a handler
- [ ] Every async method accepts and passes a `CancellationToken`
- [ ] New entities have an `IEntityTypeConfiguration` and a migration
- [ ] No secrets in config files
- [ ] `dotnet build` has no warnings and `dotnet test` passes