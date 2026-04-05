# Module 02 — Setting Up the Solution Structure

> **Goal:** Create the .NET 10 solution with four projects and wire their references correctly.

---

## 🧠 Concept: Why Four Separate Projects?

Each project is a compilation boundary. This enforces the Dependency Rule at the **compiler level** — if `Domain` accidentally tries to reference EF Core, the build will fail. This is better than relying on developer discipline.

---

## 🛠️ Step 1: Create the Solution and Projects

Open a terminal and run:

```bash
# Create solution folder
mkdir CleanApp && cd CleanApp

# Create solution file
dotnet new sln -n CleanApp

# Create the four projects
dotnet new classlib -n CleanApp.Domain       -o src/CleanApp.Domain
dotnet new classlib -n CleanApp.Application  -o src/CleanApp.Application
dotnet new classlib -n CleanApp.Infrastructure -o src/CleanApp.Infrastructure
dotnet new webapi   -n CleanApp.Api          -o src/CleanApp.Api

# Add projects to solution
dotnet sln add src/CleanApp.Domain/CleanApp.Domain.csproj
dotnet sln add src/CleanApp.Application/CleanApp.Application.csproj
dotnet sln add src/CleanApp.Infrastructure/CleanApp.Infrastructure.csproj
dotnet sln add src/CleanApp.Api/CleanApp.Api.csproj
```

---

## 🛠️ Step 2: Add Project References (Dependency Direction)

```bash
# Application depends on Domain
dotnet add src/CleanApp.Application/CleanApp.Application.csproj \
  reference src/CleanApp.Domain/CleanApp.Domain.csproj

# Infrastructure depends on Application and Domain
dotnet add src/CleanApp.Infrastructure/CleanApp.Infrastructure.csproj \
  reference src/CleanApp.Application/CleanApp.Application.csproj
dotnet add src/CleanApp.Infrastructure/CleanApp.Infrastructure.csproj \
  reference src/CleanApp.Domain/CleanApp.Domain.csproj

# Api depends on Application and Infrastructure
dotnet add src/CleanApp.Api/CleanApp.Api.csproj \
  reference src/CleanApp.Application/CleanApp.Application.csproj
dotnet add src/CleanApp.Api/CleanApp.Api.csproj \
  reference src/CleanApp.Infrastructure/CleanApp.Infrastructure.csproj
```

---

## 🛠️ Step 3: Add Required NuGet Packages

```bash
# EF Core + PostgreSQL provider — Infrastructure only
dotnet add src/CleanApp.Infrastructure/CleanApp.Infrastructure.csproj \
  package Npgsql.EntityFrameworkCore.PostgreSQL

# EF Core Design tools — Api project (for migrations CLI)
dotnet add src/CleanApp.Api/CleanApp.Api.csproj \
  package Microsoft.EntityFrameworkCore.Design
```

---

## 📁 Expected Folder Structure

```
CleanApp/
├── src/
│   ├── CleanApp.Domain/
│   │   └── CleanApp.Domain.csproj
│   ├── CleanApp.Application/
│   │   └── CleanApp.Application.csproj
│   ├── CleanApp.Infrastructure/
│   │   └── CleanApp.Infrastructure.csproj
│   └── CleanApp.Api/
│       ├── Program.cs
│       └── CleanApp.Api.csproj
└── CleanApp.sln
```

---

## ✅ Key Takeaways

- [ ] Four projects: Domain, Application, Infrastructure, Api
- [ ] References go **inward only** — Domain has zero references
- [ ] `Npgsql.EntityFrameworkCore.PostgreSQL` goes in **Infrastructure only**
- [ ] `Microsoft.EntityFrameworkCore.Design` goes in **Api** for migration commands

---

➡️ **Next:** [Module 03 — Domain Layer](./03-domain-layer.md)
