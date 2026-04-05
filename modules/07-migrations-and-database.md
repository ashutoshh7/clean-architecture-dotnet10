# Module 07 — Migrations & Database Setup

> **Goal:** Use EF Core migrations to create and evolve the PostgreSQL database schema.

---

## 🧠 Concept: What are EF Core Migrations?

EF Core **migrations** are C# files that describe how to evolve your database schema over time. When you change an entity (add a column, rename a property), you create a new migration — EF Core generates the SQL to update the database.

**Why migrations and not raw SQL?**
- Schema changes are tracked in version control
- Migrations are reversible (Up / Down)
- The team always has the same schema

---

## 🛠️ Step 1: Ensure PostgreSQL is Running

Using Docker (easiest):
```bash
docker run --name cleanapp-pg \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=cleanappdb \
  -p 5432:5432 \
  -d postgres:16
```

Or make sure your local PostgreSQL is running and the credentials in `appsettings.json` are correct.

---

## 🛠️ Step 2: Install EF Core CLI Tool

```bash
dotnet tool install --global dotnet-ef
```

Verify:
```bash
dotnet ef --version
```

---

## 🛠️ Step 3: Create the Initial Migration

You must specify `--project` (where the DbContext lives) and `--startup-project` (the runnable API project that has the connection string config):

```bash
dotnet ef migrations add InitialCreate \
  --project src/CleanApp.Infrastructure \
  --startup-project src/CleanApp.Api
```

This creates:
```
src/CleanApp.Infrastructure/
└── Migrations/
    ├── 20260405000000_InitialCreate.cs
    └── AppDbContextModelSnapshot.cs
```

---

## 🛠️ Step 4: Apply the Migration to PostgreSQL

```bash
dotnet ef database update \
  --project src/CleanApp.Infrastructure \
  --startup-project src/CleanApp.Api
```

This runs the migration SQL against your PostgreSQL database and creates the `products` table.

---

## 🛠️ Step 5: Verify in PostgreSQL

Connect to PostgreSQL and check:

```sql
\c cleanappdb
\dt
SELECT * FROM products;
```

Or use **pgAdmin**, **DBeaver**, or **TablePlus**.

---

## 🧠 Concept: Future Migrations

Every time you change a domain entity (add a property, rename a column), you:
1. Update the entity class
2. Update the EF configuration
3. Run `dotnet ef migrations add <MigrationName>`
4. Run `dotnet ef database update`

Never manually edit the database — always go through migrations.

---

## ✅ Key Takeaways

- [ ] Install `dotnet-ef` CLI tool globally
- [ ] Always specify `--project` and `--startup-project` for migration commands
- [ ] Migrations live in `Infrastructure`, not `Api`
- [ ] Never manually edit the database schema — use migrations
- [ ] Commit migrations to source control

---

➡️ **Next:** [Module 08 — Repository Pattern](./08-repository-pattern.md)
