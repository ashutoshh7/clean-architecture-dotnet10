# Module 14 — End-to-End Request Flow

This module brings everything together and shows the complete picture from an HTTP request hitting your API to the read model being available in Cosmos DB.

---

## 1. Complete Solution Structure

After adding the CQRS + Cosmos DB + Service Bus pieces, your solution looks like this:

```
OrderManagement.sln
├── src/
│   ├── OrderManagement.Domain/
│   │   ├── Entities/Order.cs
│   │   ├── Entities/OrderItem.cs
│   │   └── Enums/OrderStatus.cs
│   │
│   ├── OrderManagement.Application/
│   │   ├── Features/
│   │   │   └── Orders/
│   │   │       ├── Commands/CreateOrderCommand.cs
│   │   │       ├── Commands/CreateOrderCommandHandler.cs
│   │   │       ├── Commands/CreateOrderValidator.cs
│   │   │       ├── Queries/GetOrderByIdQuery.cs
│   │   │       └── Queries/GetOrdersByCustomerQuery.cs
│   │   ├── Events/OrderCreatedEvent.cs
│   │   ├── Interfaces/
│   │   │   ├── IOrderWriteRepository.cs
│   │   │   ├── IOrderReadRepository.cs
│   │   │   ├── IEventPublisher.cs
│   │   │   └── IUnitOfWork.cs
│   │   └── ReadModels/OrderReadModel.cs
│   │
│   ├── OrderManagement.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs                  ← EF Core (write DB)
│   │   │   ├── OrderWriteRepository.cs
│   │   │   └── CosmosOrderReadRepository.cs     ← Cosmos DB (read DB)
│   │   ├── Messaging/
│   │   │   ├── ServiceBusEventPublisher.cs
│   │   │   └── OrderEventConsumer.cs
│   │   └── DependencyInjection.cs
│   │
│   └── OrderManagement.API/
│       ├── Controllers/OrdersController.cs
│       └── Program.cs
│
└── tests/
    ├── OrderManagement.Domain.Tests/
    ├── OrderManagement.Application.Tests/
    └── OrderManagement.Infrastructure.Tests/
```

---

## 2. Complete Dependency Flow

```mermaid
flowchart TD
    API["🌐 API Layer\nOrdersController"] --> APP
    APP["📋 Application Layer\nCommands · Queries · Handlers"] --> DOM
    APP --> IWRITE["IOrderWriteRepository"]
    APP --> IREAD["IOrderReadRepository"]
    APP --> IPUB["IEventPublisher"]
    DOM["🏛️ Domain Layer\nEntities · Value Objects"]
    INFRA["⚙️ Infrastructure Layer\nImplementations"]
    IWRITE -.->|implements| INFRA
    IREAD -.->|implements| INFRA
    IPUB -.->|implements| INFRA
    INFRA --> SQL[(SQL Server)]
    INFRA --> SB[Azure Service Bus]
    INFRA --> COSMOS[(Cosmos DB)]

    classDef layer fill:#0078D4,color:#fff,stroke:#005a9e
    classDef db fill:#CC5500,color:#fff,stroke:#993300
    classDef infra fill:#107C10,color:#fff,stroke:#0a5c0a
    classDef iface fill:#5C2D91,color:#fff,stroke:#3a1a6b
    class API,APP,DOM layer
    class SQL,COSMOS,SB db
    class INFRA infra
    class IWRITE,IREAD,IPUB iface
```

---

## 3. Full HTTP Request Lifecycle

```mermaid
flowchart TD
    REQ(["POST /api/orders"])
    REQ --> VAL{FluentValidation\nPipeline Behavior}
    VAL -->|Invalid| ERR(["400 Bad Request"])
    VAL -->|Valid| CMD[CreateOrderCommand]
    CMD --> HAND[CreateOrderCommandHandler]
    HAND --> SQL[(SQL Server\nWrite DB)]
    SQL --> SAVED{Save OK?}
    SAVED -->|Error| ERR2(["500 Internal Server Error"])
    SAVED -->|OK| PUB[Publish OrderCreatedEvent]
    PUB --> SB[Azure Service Bus Topic]
    SB --> RES(["201 Created — OrderId returned"])

    SB -.->|async consumer| CON[OrderEventConsumer\nBackgroundService]
    CON --> PROJ[Map to OrderReadModel]
    PROJ --> UPS[UpsertAsync]
    UPS --> CDB[(Cosmos DB\nRead Store)]

    QREQ(["GET /api/orders/{id}"])
    QREQ --> QHAND[GetOrderByIdQueryHandler]
    QHAND --> CDB
    CDB --> RESP(["200 OK — OrderReadModel JSON"])

    style REQ fill:#0078D4,color:#fff
    style QREQ fill:#0078D4,color:#fff
    style RES fill:#107C10,color:#fff
    style RESP fill:#107C10,color:#fff
    style ERR fill:#A80000,color:#fff
    style ERR2 fill:#A80000,color:#fff
    style SB fill:#0078D4,color:#fff
    style CDB fill:#0078D4,color:#fff
    style SQL fill:#CC5500,color:#fff
```

---

## 4. API Controller (Complete)

```csharp
// API/Controllers/OrdersController.cs
[ApiController]
[Route("api/[controller]")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _sender;
    public OrdersController(ISender sender) => _sender = sender;

    [HttpPost]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var command = new CreateOrderCommand(
            request.CustomerId,
            request.CustomerName,
            request.Items.Select(i => new OrderItemInput(i.ProductName, i.Quantity, i.UnitPrice)).ToList());

        var orderId = await _sender.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id = orderId, customerId = request.CustomerId }, orderId);
    }

    [HttpGet("{id}")]
    [ProducesResponseType(typeof(OrderReadModel), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(
        [FromRoute] string id,
        [FromQuery] string customerId,
        CancellationToken ct)
    {
        var result = await _sender.Send(new GetOrderByIdQuery(id, customerId), ct);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpGet("customer/{customerId}")]
    [ProducesResponseType(typeof(IReadOnlyList<OrderReadModel>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetByCustomer(
        [FromRoute] string customerId,
        CancellationToken ct)
    {
        var results = await _sender.Send(new GetOrdersByCustomerQuery(customerId), ct);
        return Ok(results);
    }
}
```

---

## 5. Program.cs Wiring

```csharp
// API/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Application layer
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<CreateOrderCommand>()
       .AddOpenBehavior(typeof(ValidationBehavior<,>)));

builder.Services.AddValidatorsFromAssemblyContaining<CreateOrderValidator>();

// Infrastructure layer
builder.Services
    .AddDbContext<AppDbContext>(opt =>
        opt.UseSqlServer(builder.Configuration.GetConnectionString("WriteDb")))
    .AddCosmosDb(builder.Configuration)
    .AddMessaging(builder.Configuration);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

---

## 6. Quick-Start Checklist

- [ ] Docker Desktop running
- [ ] `docker compose up -d` started (Service Bus emulator)
- [ ] Cosmos DB Emulator running (Windows installer or Docker)
- [ ] `dotnet ef migrations add InitialCreate --project Infrastructure --startup-project API`
- [ ] `dotnet ef database update --project Infrastructure --startup-project API`
- [ ] `dotnet run --project API`
- [ ] Open Swagger at `https://localhost:{port}/swagger`
- [ ] POST to `/api/orders` — verify 201 response
- [ ] Wait ~1 second, then GET `/api/orders/{id}?customerId={id}` — verify Cosmos read works
- [ ] Check Cosmos emulator Data Explorer at `https://localhost:8081/_explorer/index.html` to see your document

---

## 7. Further Learning Path

| Topic | Why It Matters |
|-------|---------------|
| **Outbox Pattern** | Guarantees no message is lost if Service Bus is temporarily unavailable |
| **Event Sourcing** | Store all state changes as events — Cosmos DB is a natural fit |
| **Saga / Process Manager** | Coordinate multi-step workflows across services |
| **Azure Cosmos DB Change Feed** | Alternative to Service Bus for projections — Cosmos pushes changes to consumers |
| **Azure AD B2C + JWT** | Secure the API — directly relevant to your existing experience |
| **Azure DevOps CI/CD** | Deploy the API to Azure App Service with pipeline |
