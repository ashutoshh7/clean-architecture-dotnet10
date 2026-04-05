# Module 08 — Repository Pattern & Unit of Work

> **Goal:** Understand and implement a Generic Repository and Unit of Work pattern for reusable, testable data access.

---

## 🧠 Concept: Why a Generic Repository?

Instead of writing the same CRUD methods for every entity, a **Generic Repository** extracts the common operations (`GetById`, `GetAll`, `Add`, `Update`, `Delete`) into a reusable base class.

The **Unit of Work** pattern wraps multiple repository operations into a single transaction — either all succeed or all fail.

---

## 🛠️ Step 1: Generic Repository Interface (Application Layer)

```csharp
// src/CleanApp.Application/Contracts/IRepository.cs
namespace CleanApp.Application.Contracts;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Delete(T entity);
}
```

---

## 🛠️ Step 2: Unit of Work Interface (Application Layer)

```csharp
// src/CleanApp.Application/Contracts/IUnitOfWork.cs
namespace CleanApp.Application.Contracts;

public interface IUnitOfWork : IDisposable
{
    IRepository<T> Repository<T>() where T : class;
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

---

## 🛠️ Step 3: Generic Repository Implementation (Infrastructure Layer)

```csharp
// src/CleanApp.Infrastructure/Repositories/Repository.cs
using CleanApp.Application.Contracts;
using CleanApp.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

namespace CleanApp.Infrastructure.Repositories;

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _db;
    protected readonly DbSet<T> _dbSet;

    public Repository(AppDbContext db)
    {
        _db = db;
        _dbSet = db.Set<T>();
    }

    public Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        _dbSet.FindAsync([id], ct).AsTask();

    public Task<List<T>> GetAllAsync(CancellationToken ct = default) =>
        _dbSet.AsNoTracking().ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default) =>
        await _dbSet.AddAsync(entity, ct);

    public void Update(T entity) =>
        _dbSet.Update(entity);

    public void Delete(T entity) =>
        _dbSet.Remove(entity);
}
```

---

## 🛠️ Step 4: Unit of Work Implementation (Infrastructure Layer)

```csharp
// src/CleanApp.Infrastructure/Repositories/UnitOfWork.cs
using CleanApp.Application.Contracts;
using CleanApp.Infrastructure.Persistence;

namespace CleanApp.Infrastructure.Repositories;

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;
    private readonly Dictionary<Type, object> _repositories = new();

    public UnitOfWork(AppDbContext db)
    {
        _db = db;
    }

    public IRepository<T> Repository<T>() where T : class
    {
        var type = typeof(T);
        if (!_repositories.ContainsKey(type))
            _repositories[type] = new Repository<T>(_db);

        return (IRepository<T>)_repositories[type];
    }

    public Task<int> SaveChangesAsync(CancellationToken ct = default) =>
        _db.SaveChangesAsync(ct);

    public void Dispose() => _db.Dispose();
}
```

---

## 🛠️ Step 5: Register in DI

In `DependencyInjection.cs`:
```csharp
services.AddScoped<IUnitOfWork, UnitOfWork>();
```

---

## 🧠 Concept: When to Use Unit of Work?

Use Unit of Work when a **single operation touches multiple repositories**. For example, creating an Order and reducing Product stock at the same time — both DB writes must succeed together or both must fail.

```csharp
// Example service method using Unit of Work
var order = Order.Create(customerId);
await _uow.Repository<Order>().AddAsync(order, ct);

var product = await _uow.Repository<Product>().GetByIdAsync(productId, ct);
product.ReduceStock(quantity);
_uow.Repository<Product>().Update(product);

await _uow.SaveChangesAsync(ct); // single transaction
```

---

## ✅ Key Takeaways

- [ ] Generic Repository removes CRUD boilerplate for every entity
- [ ] Unit of Work wraps multiple DB operations in a single transaction
- [ ] Interfaces live in **Application**, implementations in **Infrastructure**
- [ ] Register `IUnitOfWork` as **Scoped** (tied to HTTP request lifetime)

---

➡️ **Next:** [Module 09 — CQRS + MediatR](./09-cqrs-and-mediatr.md)
