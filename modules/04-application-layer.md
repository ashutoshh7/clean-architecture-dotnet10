# Module 04 — Application Layer

> **Goal:** Define *what your system can do* — use cases, interfaces, and DTOs — without knowing how data is stored.

---

## 🧠 Concept: Application Layer = Use Cases

The Application layer answers: *"What can users do in this system?"*

It contains:
- **Interfaces (Contracts)** — e.g., `IProductRepository` (defined here, implemented in Infrastructure)
- **Use Case / Service classes** — orchestrate domain logic
- **DTOs (Data Transfer Objects)** — what goes in and out of use cases
- **Mapping logic** — convert between domain entities and DTOs

It does **NOT** contain:
- EF Core, SQL, or database code
- HTTP concepts (status codes, `HttpContext`)
- Specific framework classes

---

## 🛠️ Step 1: Define the Repository Interface

```
src/CleanApp.Application/
└── Contracts/
    └── IProductRepository.cs
```

```csharp
// src/CleanApp.Application/Contracts/IProductRepository.cs
using CleanApp.Domain.Entities;

namespace CleanApp.Application.Contracts;

public interface IProductRepository
{
    Task<List<Product>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<Product?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(Product product, CancellationToken cancellationToken = default);
    Task UpdateAsync(Product product, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}
```

> ⚠️ Notice: this interface is in Application, but there is no EF Core here. The *implementation* will be in Infrastructure.

---

## 🛠️ Step 2: Define DTOs

```
src/CleanApp.Application/
└── Products/
    ├── ProductDto.cs
    ├── CreateProductRequest.cs
    └── UpdatePriceRequest.cs
```

```csharp
// src/CleanApp.Application/Products/ProductDto.cs
namespace CleanApp.Application.Products;

public record ProductDto(
    Guid Id,
    string Name,
    decimal Price,
    int StockQuantity,
    DateTime CreatedAt
);
```

```csharp
// src/CleanApp.Application/Products/CreateProductRequest.cs
namespace CleanApp.Application.Products;

public record CreateProductRequest(
    string Name,
    decimal Price,
    int StockQuantity
);
```

---

## 🛠️ Step 3: Create the Product Service

```
src/CleanApp.Application/
└── Products/
    └── ProductService.cs
```

```csharp
// src/CleanApp.Application/Products/ProductService.cs
using CleanApp.Application.Contracts;
using CleanApp.Domain.Entities;

namespace CleanApp.Application.Products;

public class ProductService
{
    private readonly IProductRepository _repository;

    public ProductService(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<List<ProductDto>> GetAllAsync(CancellationToken ct = default)
    {
        var products = await _repository.GetAllAsync(ct);
        return products.Select(p => new ProductDto(
            p.Id, p.Name, p.Price, p.StockQuantity, p.CreatedAt
        )).ToList();
    }

    public async Task<ProductDto> CreateAsync(CreateProductRequest request, CancellationToken ct = default)
    {
        var product = Product.Create(request.Name, request.Price, request.StockQuantity);
        await _repository.AddAsync(product, ct);
        return new ProductDto(product.Id, product.Name, product.Price, product.StockQuantity, product.CreatedAt);
    }

    public async Task ReduceStockAsync(Guid id, int quantity, CancellationToken ct = default)
    {
        var product = await _repository.GetByIdAsync(id, ct)
            ?? throw new KeyNotFoundException($"Product {id} not found.");

        product.ReduceStock(quantity);
        await _repository.UpdateAsync(product, ct);
    }
}
```

---

## 🧠 Concept: Why Records for DTOs?

C# `record` types are perfect for DTOs because:
- They are **immutable by default** — data flows one-way
- They have **value equality** built-in — two DTOs with same values are equal
- They are **concise** — no need to write boilerplate constructors

---

## ✅ Key Takeaways

- [ ] Application defines **interfaces** for repositories (not implementations)
- [ ] **DTOs** are simple data containers — use `record` types
- [ ] **Services** orchestrate domain logic and use repository interfaces
- [ ] Application depends only on **Domain** — never Infrastructure

---

➡️ **Next:** [Module 05 — Infrastructure Layer](./05-infrastructure-layer.md)
