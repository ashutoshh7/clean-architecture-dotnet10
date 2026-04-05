# Module 03 — Domain Layer

> **Goal:** Build the core of your application — entities and business rules with zero external dependencies.

---

## 🧠 Concept: What Goes in Domain?

The Domain layer represents your **business world** in code. It answers: *"What does this system do?"* — not *"how does it store data?"* or *"what HTTP endpoints exist?"*

**Domain contains:**
- **Entities** — objects with identity (e.g., `Product`, `Order`, `Customer`)
- **Value Objects** — objects defined by their value, not identity (e.g., `Money`, `Address`)
- **Domain Events** — things that happened (e.g., `OrderPlacedEvent`)
- **Enums** — domain-level enumerations (e.g., `OrderStatus`)
- **Domain Exceptions** — business rule violations (e.g., `InsufficientStockException`)

**Domain does NOT contain:**
- EF Core attributes like `[Column]`, `[Table]` (configure in Infrastructure instead)
- Any `using Microsoft.EntityFrameworkCore;`
- HTTP-related code

---

## 🛠️ Step 1: Create the Product Entity

Delete the default `Class1.cs` and create this folder structure:

```
src/CleanApp.Domain/
└── Entities/
    └── Product.cs
```

```csharp
// src/CleanApp.Domain/Entities/Product.cs
namespace CleanApp.Domain.Entities;

public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public decimal Price { get; private set; }
    public int StockQuantity { get; private set; }
    public DateTime CreatedAt { get; private set; }

    // EF Core needs a parameterless constructor (can be private/protected)
    private Product() { }

    // Factory method — enforces business rules on creation
    public static Product Create(string name, decimal price, int stockQuantity)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Product name cannot be empty.", nameof(name));

        if (price < 0)
            throw new ArgumentException("Price cannot be negative.", nameof(price));

        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Price = price,
            StockQuantity = stockQuantity,
            CreatedAt = DateTime.UtcNow
        };
    }

    // Domain behavior — business logic lives on the entity
    public void ReduceStock(int quantity)
    {
        if (quantity > StockQuantity)
            throw new InvalidOperationException($"Insufficient stock. Available: {StockQuantity}");

        StockQuantity -= quantity;
    }

    public void UpdatePrice(decimal newPrice)
    {
        if (newPrice < 0)
            throw new ArgumentException("Price cannot be negative.");

        Price = newPrice;
    }
}
```

---

## 🧠 Concept: Private Setters + Factory Methods

Using **private setters** and a **factory method** (`Product.Create(...)`) enforces that a `Product` can only be created in a valid state. You can never do `new Product { Price = -100 }` by accident.

This is called **Encapsulation** and it's a core principle of Domain-Driven Design (DDD).

---

## 🛠️ Step 2: Add a Base Entity (Optional but Recommended)

```
src/CleanApp.Domain/
└── Common/
    └── BaseEntity.cs
```

```csharp
// src/CleanApp.Domain/Common/BaseEntity.cs
namespace CleanApp.Domain.Common;

public abstract class BaseEntity
{
    public Guid Id { get; protected set; }
    public DateTime CreatedAt { get; protected set; }
    public DateTime? UpdatedAt { get; protected set; }
}
```

Now `Product` can inherit from `BaseEntity` and you don't repeat `Id` and timestamps in every entity.

---

## ✅ Key Takeaways

- [ ] Domain entities encapsulate **business behavior**, not just data bags
- [ ] Use **private setters** to prevent invalid state
- [ ] Use **factory methods** to enforce creation rules
- [ ] Domain has **zero NuGet dependencies** — only .NET BCL
- [ ] EF Core mapping config goes in Infrastructure, not here

---

➡️ **Next:** [Module 04 — Application Layer](./04-application-layer.md)
