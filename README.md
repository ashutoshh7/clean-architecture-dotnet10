# 🏗️ Clean Architecture in .NET 10 with PostgreSQL

> A structured, hands-on learning course for full-stack .NET developers.

This repository is your step-by-step guide to learning and implementing **Clean Architecture** in a real ASP.NET Core .NET 10 Web API project backed by **PostgreSQL**.

Each module builds on the previous one — from understanding the theory to writing production-quality code.

---

## 📚 Course Modules

| Module | Topic | Description |
|--------|-------|-------------|
| [Module 01](./modules/01-what-is-clean-architecture.md) | What is Clean Architecture? | Core concepts, layers, dependency rule |
| [Module 02](./modules/02-solution-structure.md) | Setting Up the Solution | Create projects, folder structure, project references |
| [Module 03](./modules/03-domain-layer.md) | Domain Layer | Entities, value objects, domain rules |
| [Module 04](./modules/04-application-layer.md) | Application Layer | Use cases, interfaces, DTOs, service contracts |
| [Module 05](./modules/05-infrastructure-layer.md) | Infrastructure Layer | EF Core, PostgreSQL, repository implementations |
| [Module 06](./modules/06-api-layer.md) | API Layer | Controllers, DI wiring, Program.cs |
| [Module 07](./modules/07-migrations-and-database.md) | Migrations & Database | EF Core migrations, applying to PostgreSQL |
| [Module 08](./modules/08-repository-pattern.md) | Repository Pattern | Generic repo, unit of work |
| [Module 09](./modules/09-cqrs-and-mediatr.md) | CQRS + MediatR | Commands, queries, handlers |
| [Module 10](./modules/10-validation-and-errors.md) | Validation & Error Handling | FluentValidation, global exception middleware |

---

## 🧰 Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)
- [PostgreSQL](https://www.postgresql.org/download/) (or Docker: `docker run --name pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres`)
- [Visual Studio 2022/2026](https://visualstudio.microsoft.com/) or [VS Code](https://code.visualstudio.com/) or [Rider](https://www.jetbrains.com/rider/)
- Basic C# knowledge

---

## 🚀 Quick Start

```bash
git clone https://github.com/ashutoshh7/clean-architecture-dotnet10.git
cd clean-architecture-dotnet10
```

Then open `modules/01-what-is-clean-architecture.md` and follow along!

---

## 🗂️ Final Project Structure

```
YourApp/
├── src/
│   ├── YourApp.Domain/
│   ├── YourApp.Application/
│   ├── YourApp.Infrastructure/
│   └── YourApp.Api/
└── YourApp.sln
```

---

> Made with ❤️ for developers learning Clean Architecture the right way.
