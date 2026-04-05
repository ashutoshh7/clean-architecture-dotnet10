# Module 09 — CQRS + MediatR

> **Goal:** Separate read operations (Queries) from write operations (Commands) using the CQRS pattern and MediatR.

---

## 🧠 Concept: What is CQRS?

**CQRS** (Command Query Responsibility Segregation) means you use different models for reading and writing data.

- **Command** — changes state, returns nothing (or just an ID): `CreateProductCommand`
- **Query** — reads state, returns data: `GetAllProductsQuery`

This replaces the fat `ProductService` class with small, focused **handler classes** — one class per use case. This is much easier to test and maintain.

---

## 🛠️ Step 1: Install MediatR

```bash
dotnet add src/CleanApp.Application/CleanApp.Application.csproj \
  package MediatR

dotnet add src/CleanApp.Api/CleanApp.Api.csproj \
  package MediatR.Extensions.Microsoft.DependencyInjection
```

---

## 🛠️ Step 2: Create a Query

```
src/CleanApp.Application/
└── Products/
    └── Queries/
        ├── GetAllProductsQuery.cs
        └── GetAllProductsHandler.cs
```

```csharp
// src/CleanApp.Application/Products/Queries/GetAllProductsQuery.cs
using MediatR;

namespace CleanApp.Application.Products.Queries;

public record GetAllProductsQuery : IRequest<List<ProductDto>>;
```

```csharp
// src/CleanApp.Application/Products/Queries/GetAllProductsHandler.cs
using CleanApp.Application.Contracts;
using MediatR;

namespace CleanApp.Application.Products.Queries;

public class GetAllProductsHandler : IRequestHandler<GetAllProductsQuery, List<ProductDto>>
{
    private readonly IProductRepository _repository;

    public GetAllProductsHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<List<ProductDto>> Handle(GetAllProductsQuery request, CancellationToken ct)
    {
        var products = await _repository.GetAllAsync(ct);
        return products.Select(p =>
            new ProductDto(p.Id, p.Name, p.Price, p.StockQuantity, p.CreatedAt)
        ).ToList();
    }
}
```

---

## 🛠️ Step 3: Create a Command

```
src/CleanApp.Application/
└── Products/
    └── Commands/
        ├── CreateProductCommand.cs
        └── CreateProductHandler.cs
```

```csharp
// src/CleanApp.Application/Products/Commands/CreateProductCommand.cs
using MediatR;

namespace CleanApp.Application.Products.Commands;

public record CreateProductCommand(
    string Name,
    decimal Price,
    int StockQuantity
) : IRequest<ProductDto>;
```

```csharp
// src/CleanApp.Application/Products/Commands/CreateProductHandler.cs
using CleanApp.Application.Contracts;
using CleanApp.Domain.Entities;
using MediatR;

namespace CleanApp.Application.Products.Commands;

public class CreateProductHandler : IRequestHandler<CreateProductCommand, ProductDto>
{
    private readonly IProductRepository _repository;

    public CreateProductHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<ProductDto> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = Product.Create(request.Name, request.Price, request.StockQuantity);
        await _repository.AddAsync(product, ct);
        return new ProductDto(product.Id, product.Name, product.Price, product.StockQuantity, product.CreatedAt);
    }
}
```

---

## 🛠️ Step 4: Update the Controller to Use MediatR

```csharp
// src/CleanApp.Api/Controllers/ProductsController.cs
using CleanApp.Application.Products;
using CleanApp.Application.Products.Commands;
using CleanApp.Application.Products.Queries;
using MediatR;
using Microsoft.AspNetCore.Mvc;

namespace CleanApp.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken ct) =>
        Ok(await _mediator.Send(new GetAllProductsQuery(), ct));

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductCommand command, CancellationToken ct)
    {
        var result = await _mediator.Send(command, ct);
        return CreatedAtAction(nameof(GetAll), new { id = result.Id }, result);
    }
}
```

---

## 🛠️ Step 5: Register MediatR in Program.cs

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(
        typeof(CleanApp.Application.Products.Queries.GetAllProductsQuery).Assembly));
```

---

## 🧠 Concept: MediatR Pipeline Behaviors

MediatR supports **Pipeline Behaviors** — middleware that runs before/after every handler. Common uses:
- Logging every command/query
- Validating input with FluentValidation
- Wrapping operations in transactions

---

## ✅ Key Takeaways

- [ ] **Commands** change state; **Queries** read state
- [ ] Each use case is a **separate handler class** — easy to test in isolation
- [ ] The controller only sends requests through `IMediator` — it doesn't know the handler
- [ ] MediatR Pipeline Behaviors add cross-cutting concerns cleanly

---

➡️ **Next:** [Module 10 — Validation & Error Handling](./10-validation-and-errors.md)
