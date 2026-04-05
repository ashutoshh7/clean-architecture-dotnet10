# Module 10 — Validation & Error Handling

> **Goal:** Add input validation with FluentValidation and a global exception handler for clean, consistent API error responses.

---

## 🧠 Concept: Where Should Validation Live?

There are two kinds of validation:
- **Input validation** — is the request well-formed? (Name not empty, Price > 0) → **Application Layer**
- **Business rule validation** — does the operation make sense? (Is stock available?) → **Domain Layer**

Never put validation logic in controllers.

---

## 🛠️ Step 1: Install FluentValidation

```bash
dotnet add src/CleanApp.Application/CleanApp.Application.csproj \
  package FluentValidation

dotnet add src/CleanApp.Api/CleanApp.Api.csproj \
  package FluentValidation.AspNetCore
```

---

## 🛠️ Step 2: Create a Validator

```
src/CleanApp.Application/
└── Products/
    └── Commands/
        └── CreateProductCommandValidator.cs
```

```csharp
// src/CleanApp.Application/Products/Commands/CreateProductCommandValidator.cs
using FluentValidation;

namespace CleanApp.Application.Products.Commands;

public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Product name is required.")
            .MaximumLength(200).WithMessage("Name cannot exceed 200 characters.");

        RuleFor(x => x.Price)
            .GreaterThanOrEqualTo(0).WithMessage("Price must be zero or greater.");

        RuleFor(x => x.StockQuantity)
            .GreaterThanOrEqualTo(0).WithMessage("Stock quantity must be zero or greater.");
    }
}
```

---

## 🛠️ Step 3: MediatR Validation Pipeline Behavior

Add a behavior that runs every validator before the handler:

```csharp
// src/CleanApp.Application/Behaviors/ValidationBehavior.cs
using FluentValidation;
using MediatR;

namespace CleanApp.Application.Behaviors;

public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);

        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

---

## 🛠️ Step 4: Global Exception Middleware

```
src/CleanApp.Api/
└── Middleware/
    └── ExceptionHandlingMiddleware.cs
```

```csharp
// src/CleanApp.Api/Middleware/ExceptionHandlingMiddleware.cs
using FluentValidation;
using System.Net;
using System.Text.Json;

namespace CleanApp.Api.Middleware;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = (int)HttpStatusCode.BadRequest;
            context.Response.ContentType = "application/json";
            var errors = ex.Errors.Select(e => new { e.PropertyName, e.ErrorMessage });
            await context.Response.WriteAsync(JsonSerializer.Serialize(new { errors }));
        }
        catch (KeyNotFoundException ex)
        {
            context.Response.StatusCode = (int)HttpStatusCode.NotFound;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(JsonSerializer.Serialize(new { error = ex.Message }));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(JsonSerializer.Serialize(new { error = "An unexpected error occurred." }));
        }
    }
}
```

---

## 🛠️ Step 5: Register Everything in Program.cs

```csharp
// In Program.cs, add:
using CleanApp.Application.Behaviors;
using CleanApp.Api.Middleware;
using FluentValidation;

// Register validators from Application assembly
builder.Services.AddValidatorsFromAssembly(
    typeof(CleanApp.Application.Products.Commands.CreateProductCommandValidator).Assembly);

// Register validation pipeline behavior
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

// ... app.Build() ...

// Register global exception handler BEFORE other middleware
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

---

## 🧠 Concept: Middleware Order Matters

In ASP.NET Core, middleware runs in the order it is registered. The exception handling middleware must be registered **first** (before `UseRouting`, `UseAuthorization`, etc.) so it can catch exceptions from all subsequent middleware.

---

## ✅ Key Takeaways

- [ ] Use **FluentValidation** for input validation in the Application layer
- [ ] **ValidationBehavior** runs validators automatically before every MediatR handler
- [ ] **Global exception middleware** catches all unhandled exceptions and returns consistent JSON errors
- [ ] Exception middleware must be the **first** middleware registered
- [ ] Domain exceptions express **business rule violations** — they bubble up cleanly through this setup

---

## 🎉 You've Completed the Course!

You now know how to build a full Clean Architecture .NET 10 API with:
- ✅ Strict layer separation (Domain → Application → Infrastructure → Api)
- ✅ PostgreSQL + EF Core with Npgsql
- ✅ Repository Pattern + Unit of Work
- ✅ CQRS with MediatR
- ✅ FluentValidation with Pipeline Behaviors
- ✅ Global exception handling

**What to learn next:**
- 🔐 Authentication with Azure AD B2C or JWT Bearer tokens
- 📊 Adding read models / projections for complex queries
- 🧪 Unit Testing with xUnit + Moq
- 🐳 Dockerizing the API + PostgreSQL with `docker-compose`
- ⚙️ Azure deployment with App Service + Azure Database for PostgreSQL
