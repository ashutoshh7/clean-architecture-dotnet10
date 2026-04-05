# Module 06 — API Layer

> **Goal:** Expose your use cases as HTTP endpoints and wire everything together through Dependency Injection.

---

## 🧠 Concept: API Layer = Composition Root

The API project is the **entry point** of your application. Its job is:
1. Register all services in the DI container
2. Configure the middleware pipeline
3. Expose HTTP endpoints (controllers or minimal API)

Controllers should be **thin** — they receive a request, pass it to a service, and return a response. No business logic here.

---

## 🛠️ Step 1: Configure `appsettings.json`

```json
// src/CleanApp.Api/appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=cleanappdb;Username=postgres;Password=postgres"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

## 🛠️ Step 2: Wire Up `Program.cs`

```csharp
// src/CleanApp.Api/Program.cs
using CleanApp.Application.Products;
using CleanApp.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// ── Services ──────────────────────────────────────────────────
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register Application services
builder.Services.AddScoped<ProductService>();

// Register Infrastructure (DbContext + Repositories)
builder.Services.AddInfrastructure(builder.Configuration);

// ── Middleware Pipeline ────────────────────────────────────────
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## 🛠️ Step 3: Create the Products Controller

```
src/CleanApp.Api/
└── Controllers/
    └── ProductsController.cs
```

```csharp
// src/CleanApp.Api/Controllers/ProductsController.cs
using CleanApp.Application.Products;
using Microsoft.AspNetCore.Mvc;

namespace CleanApp.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService _productService;

    public ProductsController(ProductService productService)
    {
        _productService = productService;
    }

    // GET /api/products
    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken ct)
    {
        var products = await _productService.GetAllAsync(ct);
        return Ok(products);
    }

    // POST /api/products
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request, CancellationToken ct)
    {
        var product = await _productService.CreateAsync(request, ct);
        return CreatedAtAction(nameof(GetAll), new { id = product.Id }, product);
    }

    // PATCH /api/products/{id}/reduce-stock
    [HttpPatch("{id:guid}/reduce-stock")]
    public async Task<IActionResult> ReduceStock(Guid id, [FromBody] ReduceStockRequest request, CancellationToken ct)
    {
        await _productService.ReduceStockAsync(id, request.Quantity, ct);
        return NoContent();
    }
}

public record ReduceStockRequest(int Quantity);
```

---

## 🧠 Concept: Why Thin Controllers?

A controller that contains business logic is hard to test, hard to reuse, and violates Single Responsibility Principle. The controller should only:
- Parse the HTTP request
- Call the application service
- Return an HTTP response

All decisions live in the Application or Domain layer.

---

## ✅ Key Takeaways

- [ ] `Program.cs` is the **Composition Root** — all DI registration happens here
- [ ] Use extension methods (`AddInfrastructure`) to keep `Program.cs` clean
- [ ] Controllers are **thin** — no business logic
- [ ] Always use `CancellationToken` in async controller actions for proper cancellation

---

➡️ **Next:** [Module 07 — Migrations & Database](./07-migrations-and-database.md)
