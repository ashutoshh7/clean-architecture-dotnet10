# Module 01 — What is Clean Architecture?

> **Goal:** Understand the *why* behind Clean Architecture before writing any code.

---

## 🧠 Concept: The Problem with Layered Spaghetti

In most beginner .NET projects, everything ends up in one big project:
- Controllers call Entity Framework directly
- Business logic is scattered across services, controllers, and even models
- Changing the database means touching 20 files

**Clean Architecture** solves this by enforcing a strict **separation of concerns** through layers — and one golden rule: **dependencies always point inward**.

---

## 📐 The Four Layers

```
         ┌─────────────────────────────┐
         │         API / UI            │  ← Outermost: HTTP, controllers
         ├─────────────────────────────┤
         │      Infrastructure         │  ← DB, EF Core, external services
         ├─────────────────────────────┤
         │        Application          │  ← Use cases, interfaces, DTOs
         ├─────────────────────────────┤
         │          Domain             │  ← Entities, business rules (innermost)
         └─────────────────────────────┘
```

| Layer | Knows About | Does NOT Know About |
|-------|-------------|---------------------|
| Domain | Nothing external | EF Core, ASP.NET, PostgreSQL |
| Application | Domain | EF Core, PostgreSQL |
| Infrastructure | Application + Domain | ASP.NET, HTTP |
| API | All (via DI) | — |

---

## 🔑 The Dependency Rule

> **Source code dependencies must point inward.** Outer layers depend on inner layers. Inner layers never depend on outer layers.

This means:
- `Domain` has zero external package references
- `Application` references `Domain` only
- `Infrastructure` references `Application` and `Domain`
- `Api` wires everything together through Dependency Injection

---

## 💡 Why PostgreSQL Only Lives in Infrastructure

Your `Domain` entity `Product` does not know it is stored in PostgreSQL. If you swap PostgreSQL for SQL Server tomorrow, you only change the `Infrastructure` layer — your business rules stay untouched.

---

## ✅ Key Takeaways

- [ ] Clean Architecture is about **dependency direction**, not folder names
- [ ] The **Domain** is the heart — it never depends on anything external
- [ ] **Infrastructure** is a plugin — it can be swapped without touching business logic
- [ ] **Dependency Injection** is how the outer layers connect to the inner layers

---

## 📖 Further Reading

- [Clean Architecture by Robert C. Martin (summary)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [NDepend: Clean Architecture for ASP.NET Core](https://blog.ndepend.com/clean-architecture-for-asp-net-core-solution/)

---

➡️ **Next:** [Module 02 — Setting Up the Solution](./02-solution-structure.md)
