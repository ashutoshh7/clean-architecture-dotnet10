# Module 05 — Infrastructure Layer

> **Goal:** Implement the repository interfaces using EF Core and PostgreSQL.

---

## 🧠 Concept: Infrastructure as a Plugin

Infrastructure is the layer that talks to the outside world — databases, file systems, email providers, HTTP clients. Because it only implements interfaces defined in Application, swapping PostgreSQL for SQL Server means changing *only this layer*.

---

## 🛠️ Step 1: Create the DbContext

```
src/CleanApp.Infrastructure/
└── Persistence/
    ├── AppDbContext.cs
    └── Configurations/
        └── ProductConfiguration.cs
```

```csharp
// src/CleanApp.Infrastructure/Persistence/AppDbContext.cs
using CleanApp.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace CleanApp.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Automatically apply all IEntityTypeConfiguration classes in this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

## 🛠️ Step 2: EF Core Entity Configuration

Instead of polluting your domain entity with EF Core annotations, use a separate configuration class:

```csharp
// src/CleanApp.Infrastructure/Persistence/Configurations/ProductConfiguration.cs
using CleanApp.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace CleanApp.Infrastructure.Persistence.Configurations;

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("products");

        builder.HasKey(x => x.Id);

        builder.Property(x => x.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.Price)
            .HasColumnType("numeric(12,2)")
            .IsRequired();

        builder.Property(x => x.StockQuantity)
            .IsRequired();

        builder.Property(x => x.CreatedAt)
            .IsRequired();
    }
}
```

> ✅ This keeps EF Core config completely out of your Domain entity.

---

## 🛠️ Step 3: Implement the Repository

```
src/CleanApp.Infrastructure/
└── Repositories/
    └── ProductRepository.cs
```

```csharp
// src/CleanApp.Infrastructure/Repositories/ProductRepository.cs
using CleanApp.Application.Contracts;
using CleanApp.Domain.Entities;
using CleanApp.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

namespace CleanApp.Infrastructure.Repositories;

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db)
    {
        _db = db;
    }

    public Task<List<Product>> GetAllAsync(CancellationToken ct = default) =>
        _db.Products.AsNoTracking().ToListAsync(ct);

    public Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        _db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task AddAsync(Product product, CancellationToken ct = default)
    {
        await _db.Products.AddAsync(product, ct);
        await _db.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        _db.Products.Update(product);
        await _db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct = default)
    {
        var product = await _db.Products.FindAsync([id], ct);
        if (product is not null)
        {
            _db.Products.Remove(product);
            await _db.SaveChangesAsync(ct);
        }
    }
}
```

---

## 🛠️ Step 4: DI Extension Method (Best Practice)

Create an extension method so `Program.cs` stays clean:

```
src/CleanApp.Infrastructure/
└── DependencyInjection.cs
```

```csharp
// src/CleanApp.Infrastructure/DependencyInjection.cs
using CleanApp.Application.Contracts;
using CleanApp.Infrastructure.Persistence;
using CleanApp.Infrastructure.Repositories;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace CleanApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IProductRepository, ProductRepository>();

        return services;
    }
}
```

---

## 🧠 Concept: `AsNoTracking()` for Read Queries

When fetching data you won't modify, always use `.AsNoTracking()`. EF Core won't track changes on those entities, which improves performance significantly for read-heavy operations.

---

## ✅ Key Takeaways

- [ ] `AppDbContext` lives in Infrastructure — never in Application or Domain
- [ ] Use `IEntityTypeConfiguration<T>` classes to keep EF config out of Domain
- [ ] Repository implements the interface from Application
- [ ] Use `AsNoTracking()` for read-only queries
- [ ] DI extension method (`AddInfrastructure`) keeps `Program.cs` clean

---

➡️ **Next:** [Module 06 — API Layer](./06-api-layer.md)
