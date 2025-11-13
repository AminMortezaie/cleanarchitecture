# CleanArchitecture Public API & Component Reference

## Overview
- **Solution layout** — Follows the Clean Architecture pattern with distinct `Domain`, `Application`, `Infrastructure`, and `Web` projects, plus optional Aspire `ServiceDefaults` and `AppHost` hosts.
- **Request handling flow** — HTTP requests hit `src/Web` minimal APIs, execute MediatR commands/queries from `src/Application`, which manipulate domain entities via `IApplicationDbContext` backed by EF Core in `src/Infrastructure`.
- **Authentication** — All bundled endpoints require an authenticated user. Some operations also check roles or policies (`Roles.Administrator`, `Policies.CanPurge`).
- **Conventions** — Every endpoint maps to a `EndpointGroupBase` subclass. All commands/queries are asynchronous and validated with FluentValidation before handlers run.

## HTTP API Reference

All endpoints are grouped under `/api/{GroupName}`; group names default to the endpoint class name. The middleware pipeline enforces HTTPS, Swagger UI at `/api`, exception handling, and (when enabled) health checks.

| Group | Method | Route | Request Body | Response | AuthZ |
|-------|--------|-------|--------------|----------|-------|
| `TodoLists` | GET | `/api/TodoLists` | – | `TodosVm` | User |
| `TodoLists` | POST | `/api/TodoLists` | `CreateTodoListCommand` | `201 Created` (id) | User |
| `TodoLists` | PUT | `/api/TodoLists/{id}` | `UpdateTodoListCommand` | `204 No Content` | User |
| `TodoLists` | DELETE | `/api/TodoLists/{id}` | – | `204 No Content` | User |
| `TodoItems` | GET | `/api/TodoItems` | – (query params) | `PaginatedList<TodoItemBriefDto>` | User |
| `TodoItems` | POST | `/api/TodoItems` | `CreateTodoItemCommand` | `201 Created` (id) | User |
| `TodoItems` | PUT | `/api/TodoItems/{id}` | `UpdateTodoItemCommand` | `204 No Content` | User |
| `TodoItems` | PUT | `/api/TodoItems/UpdateDetail/{id}` | `UpdateTodoItemDetailCommand` | `204 No Content` | User |
| `TodoItems` | DELETE | `/api/TodoItems/{id}` | – | `204 No Content` | User |
| `WeatherForecasts` | GET | `/api/WeatherForecasts` | – | `IEnumerable<WeatherForecast>` | User |
| `Users`* | Auto | `/api/Users/*` | Identity endpoints | Identity responses | – |

> \* `Users` routes compile only when the `UseApiOnly` symbol is defined. They expose ASP.NET Identity endpoints via `MapIdentityApi<ApplicationUser>()`.

### TodoLists Endpoints
  - **GET `/api/TodoLists`** — Returns all lists and metadata.
    - Response sample:

    ```5:10:src/Application/TodoLists/Queries/GetTodos/TodosVm.cs
    public class TodosVm
    {
        public IReadOnlyCollection<LookupDto> PriorityLevels { get; init; } = Array.Empty<LookupDto>();
        public IReadOnlyCollection<TodoListDto> Lists { get; init; } = Array.Empty<TodoListDto>();
    }
    ```

    Example response:
    ```json
    {
      "priorityLevels": [
        { "id": 0, "title": "None" },
        { "id": 1, "title": "Low" }
      ],
      "lists": [
        {
          "id": 1,
          "title": "Todo List",
          "colour": "#FFFFFF",
          "items": [
            { "id": 1, "listId": 1, "title": "Make a todo list 📃", "done": false, "priority": 0, "note": null }
          ]
        }
      ]
    }
    ```

- **POST `/api/TodoLists`** — Creates a list.
  - Request:
    ```http
    POST /api/TodoLists HTTP/1.1
    Authorization: Bearer {token}
    Content-Type: application/json

    { "title": "Backlog" }
    ```
  - Response: `201 Created`, body is integer id, `Location: /TodoLists/{id}`.

- **PUT `/api/TodoLists/{id}`** — Updates title; body must include matching `id`.
  - Request body:
    ```json
    { "id": 1, "title": "Renamed List" }
    ```
  - Validation enforces non-empty, max 200 chars, unique title per list.

- **DELETE `/api/TodoLists/{id}`** — Deletes the list if found; returns `204 No Content`.

### TodoItems Endpoints
- **GET `/api/TodoItems`** — Requires `ListId`, optional paging params. Returns paginated items.
  - Example request: `GET /api/TodoItems?ListId=1&PageNumber=1&PageSize=10`
  - Response sample:
    ```json
    {
      "items": [
        { "id": 1, "listId": 1, "title": "Make a todo list 📃", "done": false }
      ],
      "pageNumber": 1,
      "totalPages": 1,
      "totalCount": 1,
      "hasPreviousPage": false,
      "hasNextPage": false
    }
    ```

- **POST `/api/TodoItems`** — Creates an item in a list.
  - Body: `{ "listId": 1, "title": "Add docs" }`
  - Emits `TodoItemCreatedEvent`, returns new item id.

- **PUT `/api/TodoItems/{id}`** — Updates `Title` and `Done` flag.
  - Request body: `{ "id": 5, "title": "Compile report", "done": true }`
  - Rejects if body id != route id (returns `400 Bad Request`).

- **PUT `/api/TodoItems/UpdateDetail/{id}`** — Updates metadata (`ListId`, `Priority`, `Note`).

- **DELETE `/api/TodoItems/{id}`** — Removes item and publishes `TodoItemDeletedEvent`.

### WeatherForecasts Endpoint
- **GET `/api/WeatherForecasts`** — Returns five pseudo-random forecasts with Celsius/Fahrenheit conversions.
  - Example response:
    ```json
    [
      { "date": "2025-11-14T09:00:00Z", "temperatureC": 24, "temperatureF": 75, "summary": "Mild" }
    ]
    ```

### Users Endpoint Group (Conditional)
- Adds ASP.NET Identity API endpoints for registration, login, token refresh when `UseApiOnly` symbol is defined.
- Mounted at `/api/Users` with built-in swagger docs generated by NSwag.

## Application Layer (MediatR Requests)

All requests are `record` types implementing `IRequest`/`IRequest<T>`. Validators live alongside handlers. Register services via `builder.AddApplicationServices()` which wires AutoMapper, MediatR, and pipeline behaviours.

### TodoLists Commands & Queries
- `CreateTodoListCommand` → `int`
  - Sets list title, saves via `IApplicationDbContext.TodoLists`.
  - Usage:
    ```csharp
    var id = await mediator.Send(new CreateTodoListCommand { Title = "Sprint Backlog" });
    ```
- `UpdateTodoListCommand` → `Unit`
  - Requires existing id; updates `Title`.
- `DeleteTodoListCommand` → `Unit`
  - Removes the list. Throws `NotFoundException` if missing.
- `PurgeTodoListsCommand` → `Unit`
  - Deletes all lists. Decorated with `[Authorize(Roles = Roles.Administrator)]` and `[Authorize(Policy = Policies.CanPurge)]`.
- `GetTodosQuery` → `TodosVm`
  - Projects lists/items via AutoMapper, returns lookup for `PriorityLevel`.

### TodoItems Commands & Queries
- `CreateTodoItemCommand` → `int`
  - Sets `ListId`, `Title`, `Done = false`, and raises `TodoItemCreatedEvent`.
- `UpdateTodoItemCommand` → `Unit`
  - Updates `Title` + `Done`.
- `UpdateTodoItemDetailCommand` → `Unit`
  - Sets `ListId`, `Priority`, `Note`.
- `DeleteTodoItemCommand` → `Unit`
  - Removes item, raises `TodoItemDeletedEvent`.
- `GetTodoItemsWithPaginationQuery` → `PaginatedList<TodoItemBriefDto>`
  - Filters by `ListId`, orders by `Title`, returns AutoMapper projection.

### WeatherForecasts Query
- `GetWeatherForecastsQuery` → `IEnumerable<WeatherForecast>`
  - Returns deterministic count (5 entries); handler is synchronous by design.

### Pipeline Behaviours
- `LoggingBehaviour` — Logs request name, user id, optional user name before handler executes.
- `UnhandledExceptionBehaviour` — Logs and rethrows unexpected exceptions.
- `AuthorizationBehaviour` — Enforces `[Authorize]` attributes on requests using `IUser` and `IIdentityService`.
- `ValidationBehaviour` — Runs all `IValidator<T>` instances and throws `ValidationException` when failures exist.
- `PerformanceBehaviour` — Logs warnings when a request exceeds 500 ms.

## Domain Model

### Base Types
- `BaseEntity`
  - Properties: `Id`, `DomainEvents`.
  - Methods: `AddDomainEvent`, `RemoveDomainEvent`, `ClearDomainEvents`.
- `BaseAuditableEntity` — Extends `BaseEntity` with `Created`, `CreatedBy`, `LastModified`, `LastModifiedBy`.
- `BaseEvent` — MediatR `INotification`.
- `ValueObject` — Implements equality by components; helper operators ensure null-safe comparison.

### Entities
- `TodoList` — Title (required, 200 chars), `Colour` value object, collection of `TodoItem`.
- `TodoItem`
  - Properties: `ListId`, `Title`, `Note`, `Priority`, `Reminder`, `Done`, navigation `List`.
  - When `Done` transitions `false → true`, raises `TodoItemCompletedEvent`.

### Value Objects & Enums
- `Colour`
  - Factory `Colour.From("#RRGGBB")` ensures value is within supported palette.
  - Static presets (e.g., `Colour.White`, `Colour.Blue`).
  - Implicit conversion to string and explicit from string.
- `PriorityLevel` — Enum (`None`, `Low`, `Medium`, `High`).

### Domain Events
- `TodoItemCreatedEvent`, `TodoItemCompletedEvent`, `TodoItemDeletedEvent`
  - Payload: the affected `TodoItem`.
  - Propagated via `DispatchDomainEventsInterceptor`.

### Domain Exceptions & Constants
- `UnsupportedColourException` for invalid `Colour`.
- Role/policy constants (`Roles.Administrator`, `Policies.CanPurge`) used throughout authorization.

## Cross-Cutting Interfaces & Models

### Interfaces
- `IApplicationDbContext`
  - Exposes `DbSet<TodoList>`, `DbSet<TodoItem>`, `SaveChangesAsync`.
- `IIdentityService`
  - Account utilities: `GetUserNameAsync`, `IsInRoleAsync`, `AuthorizeAsync`, `CreateUserAsync`, `DeleteUserAsync`.
- `IUser`
  - Abstraction over current user identity: `Id`, `Roles`.

### Models
- `PaginatedList<T>`
  - Fields: `Items`, `PageNumber`, `TotalPages`, `TotalCount`, `HasPreviousPage`, `HasNextPage`.
  - Static `CreateAsync` builds instance from EF `IQueryable`.
- `Result`
  - Represents success/failure with error messages; helper factory methods `Success`, `Failure`.
- `LookupDto`, `TodoItemBriefDto`, `TodoItemDto`, `TodoListDto`, `TodosVm` — AutoMapper projection targets exposed via APIs.

### Mapping Helpers
- `MappingExtensions`
  - `PaginatedListAsync` — Combined projection and pagination for queryables.
  - `ProjectToListAsync` — Materializes AutoMapper projections to `List<T>`.

### Security Attributes
- `AuthorizeAttribute`
  - Decorate requests to demand authenticated, role-based, or policy-based access controls enforced by `AuthorizationBehaviour`.

### Exceptions
- `ValidationException`
  - Aggregates FluentValidation errors into `Errors` dictionary.
- `ForbiddenAccessException`
  - Thrown for authorization failures.

## Infrastructure Layer

### Entity Framework Core
- `ApplicationDbContext`
  - Inherits `IdentityDbContext<ApplicationUser>`, implements `IApplicationDbContext`.
  - Applies configurations from assembly (e.g., `TodoListConfiguration`, `TodoItemConfiguration`).
- `ApplicationDbContextInitialiser`
  - Extension `InitialiseDatabaseAsync()` seeds default roles/users and sample todo data.
  - `TrySeedAsync` ensures idempotent seeding.
- Configurations enforce title length and own the `Colour` value object.

### SaveChanges Interceptors
- `AuditableEntityInterceptor`
  - Injects user/time information on `BaseAuditableEntity` changes (handles owned entities).
- `DispatchDomainEventsInterceptor`
  - Collects and publishes domain events before save completes.

### Identity
- `ApplicationUser` — ASP.NET Identity user record.
- `IdentityService`
  - Implements `IIdentityService` using `UserManager`, `IAuthorizationService`, `IUserClaimsPrincipalFactory`.
- `IdentityResultExtensions` — Converts `IdentityResult` to application `Result`.

### Dependency Injection
- `AddInfrastructureServices(this IHostApplicationBuilder builder)`
  - Configures EF Core with interceptors and the selected provider (`SqlServer`, `Sqlite`, `Npgsql`).
  - Adds ASP.NET Identity and authorization policy.
  - Registers `ApplicationDbContext`, `ApplicationDbContextInitialiser`, `IIdentityService`, and `TimeProvider.System`.

## Web Layer Infrastructure

- `AddWebServices(this IHostApplicationBuilder builder)`
  - Registers `CurrentUser`, exception handler, health checks, swagger (NSwag), Razor Pages (when enabled), and configures API behavior.
- `AddKeyVaultIfConfigured` — Binds Azure Key Vault secrets when `AZURE_KEY_VAULT_ENDPOINT` is set.
- `CurrentUser` (`IUser` implementation)
  - Reads `ClaimTypes.NameIdentifier` & roles from HTTP context.
- `CustomExceptionHandler`
  - Maps known exceptions (`ValidationException`, `NotFoundException`, `UnauthorizedAccessException`, `ForbiddenAccessException`) to RFC-compliant responses.
- `EndpointGroupBase` & `MapEndpoints`
  - Automatically scan/export endpoint groups; each subclass declares a `Map(RouteGroupBuilder)` method.
- `IEndpointRouteBuilderExtensions`
  - Overloads for minimal API `MapGet/Post/Put/Delete` that enforce named delegates and set operation names for OpenAPI.
- `MethodInfoExtensions`
  - Guard extension ensuring anonymous delegates aren't used without explicit names.

## Service Defaults & App Host

- `AddServiceDefaults(this IHostApplicationBuilder builder)`
  - Enables OpenTelemetry tracing/metrics, service discovery, resilience policies, and health checks suitable for .NET Aspire apps.
- `MapDefaultEndpoints(this WebApplication app)`
  - Exposes `/health` & `/alive` endpoints in development builds.
- `AppHost` project
  - Composes distributed dependencies (SQL Server or PostgreSQL) and starts the `Web` project with required database resources.

## Usage Scenarios & Examples

### Bootstrapping the Application
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();           // optional Aspire defaults
builder.AddKeyVaultIfConfigured();
builder.AddApplicationServices();       // registers MediatR, validators, AutoMapper
builder.AddInfrastructureServices();    // EF Core, Identity, interceptors
builder.AddWebServices();               // current user, endpoints, Swagger
```

### Seeding & Migrating Data
```csharp
if (app.Environment.IsDevelopment())
{
    await app.InitialiseDatabaseAsync(); // drops, creates, seeds database
}
```

### Sending Commands/Queries
```csharp
public class TodoFacade
{
    private readonly ISender _sender;

    public TodoFacade(ISender sender) => _sender = sender;

    public async Task<int> AddItemAsync(int listId, string title)
    {
        return await _sender.Send(new CreateTodoItemCommand
        {
            ListId = listId,
            Title = title
        });
    }
}
```

### Handling Validation Errors
```http
POST /api/TodoItems HTTP/1.1
Content-Type: application/json

{ "listId": 1, "title": "" }
```

Response:
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation failures have occurred.",
  "status": 400,
  "errors": { "Title": [ "'Title' must not be empty." ] }
}
```

### Leveraging Authorization Policies
```csharp
[Authorize(Roles = Roles.Administrator)]
public record PurgeTodoListsCommand : IRequest;

// In tests or services:
await mediator.Send(new PurgeTodoListsCommand()); // Requires authenticated admin user
```

### Working with `PaginatedList<T>`
```csharp
var page = await _context.TodoItems
    .Where(x => x.ListId == listId)
    .OrderBy(x => x.Title)
    .Select(x => new TodoItemBriefDto { Id = x.Id, Title = x.Title })
    .PaginatedListAsync(1, 20, cancellationToken);
```

### Responding to Domain Events
- `DispatchDomainEventsInterceptor` publishes events before commit:
  ```csharp
  foreach (var domainEvent in domainEvents)
      await _mediator.Publish(domainEvent);
  ```
- Attach notification handlers (e.g., in `Application`) to react to item lifecycle changes.

## Testing Considerations
- Functional and unit test projects (`tests/`) exercise commands, queries, and repositories. When adding new public APIs, mirror them with tests to preserve coverage.
- Use `ApplicationDbContextInitialiser` or EF Core in-memory provider for integration tests.

## Extending the API
- Create a new endpoint by inheriting `EndpointGroupBase`, overriding `GroupName` if desired, and mapping handlers with authorization/validation.
- Add corresponding MediatR request, validator, and handler in the `Application` project to keep business logic isolated from transport concerns.
- Update AutoMapper profiles or DTOs to control payload shapes without exposing domain entities directly.

