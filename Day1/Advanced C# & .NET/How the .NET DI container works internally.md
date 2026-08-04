# How the .NET Dependency Injection (DI) Container Works Internally

## Introduction

Dependency Injection (DI) is a design pattern that allows classes to receive their dependencies from an external source instead of creating them directly. In ASP.NET Core, the built-in DI container manages the creation, lifetime, and disposal of objects.

Instead of writing:

```csharp
public class OrderService
{
    private readonly PaymentService _paymentService;

    public OrderService()
    {
        _paymentService = new PaymentService();
    }
}
```

You write:

```csharp
public class OrderService
{
    private readonly IPaymentService _paymentService;

    public OrderService(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
}
```

The DI container automatically creates and injects the required dependencies.

---

# High-Level Architecture

```text
                Program.cs
                     │
                     ▼
          IServiceCollection
        (Service Registrations)
                     │
                     ▼
      BuildServiceProvider()
                     │
                     ▼
            ServiceProvider
                     │
                     ▼
      Resolve Requested Service
                     │
                     ▼
       Create Dependency Graph
                     │
                     ▼
      Constructor Injection
                     │
                     ▼
           Return Object
```

---

# Step 1: Service Registration

Every service is registered in `IServiceCollection`.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<ICacheService, CacheService>();
builder.Services.AddTransient<IEmailService, EmailService>();
```

At this stage:

- No objects are created.
- Only metadata about each service is stored.

Internally it looks like this:

| Service Type | Implementation | Lifetime |
|-------------|---------------|----------|
| IOrderService | OrderService | Scoped |
| ICacheService | CacheService | Singleton |
| IEmailService | EmailService | Transient |

---

# Step 2: ServiceDescriptor

Each registration becomes a `ServiceDescriptor`.

Simplified version:

```csharp
public class ServiceDescriptor
{
    public Type ServiceType;

    public Type ImplementationType;

    public ServiceLifetime Lifetime;

    public Func<IServiceProvider, object> Factory;

    public object ImplementationInstance;
}
```

Example:

```csharp
services.AddScoped<IOrderService, OrderService>();
```

Internally:

```
ServiceDescriptor
------------------------------
ServiceType        : IOrderService
ImplementationType : OrderService
Lifetime           : Scoped
```

---

# Step 3: Building the Service Provider

```csharp
var provider = services.BuildServiceProvider();
```

This creates the actual DI container.

Internally:

```text
ServiceCollection
        │
        ▼
Validate Registrations
        │
        ▼
Create CallSiteFactory
        │
        ▼
Create Root ServiceProvider
        │
        ▼
Ready to Resolve Services
```

---

# Step 4: CallSiteFactory

The **CallSiteFactory** analyzes all registrations and creates a dependency graph describing how each service should be constructed.

Example:

```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IPaymentService, PaymentService>();
services.AddSingleton<ILogger, Logger>();
```

Dependency graph:

```text
IOrderService
      │
      ▼
 OrderService
      │
      ▼
IPaymentService
      │
      ▼
PaymentService
      │
      ▼
ILogger
      │
      ▼
Logger
```

This graph is called a **Call Site**.

---

# Step 5: ServiceProviderEngine

The `ServiceProvider` delegates object creation to the `ServiceProviderEngine`.

```text
ServiceProvider
        │
        ▼
ServiceProviderEngine
        │
        ▼
Create Object
```

Responsibilities:

- Resolve dependencies
- Create object graphs
- Cache reusable instances
- Optimize performance

Modern .NET versions compile object creation into delegates instead of repeatedly using reflection.

---

# Step 6: Constructor Selection

Consider this class:

```csharp
public class OrderService
{
    public OrderService(
        IPaymentService paymentService,
        ILogger<OrderService> logger)
    {
    }
}
```

The container performs:

```text
OrderService
      │
      ▼
Inspect Constructor
      │
      ├──────────────┐
      ▼              ▼
Resolve        Resolve
IPayment       ILogger
Service
      │              │
      ▼              ▼
Create         Create
Objects        Objects
      │              │
      └──────┬───────┘
             ▼
Invoke Constructor
```

---

# Step 7: Recursive Dependency Resolution

Imagine this dependency chain:

```text
OrderController
        │
        ▼
OrderService
        │
        ▼
PaymentService
        │
        ▼
Repository
        │
        ▼
DbContext
```

Resolution order:

1. Resolve `OrderController`
2. Needs `OrderService`
3. Needs `PaymentService`
4. Needs `Repository`
5. Needs `DbContext`
6. Create `DbContext`
7. Create `Repository`
8. Create `PaymentService`
9. Create `OrderService`
10. Create `OrderController`

The container builds the object graph from the deepest dependency upward.

---

# Step 8: Lifetime Management

## Singleton

Created once for the application's lifetime.

```text
Application Starts
        │
        ▼
Create Singleton
        │
        ▼
Cache Instance
        │
        ▼
Reuse Everywhere
        │
        ▼
Dispose When App Stops
```

Registration:

```csharp
services.AddSingleton<ICacheService, CacheService>();
```

---

## Scoped

One instance per scope (typically one HTTP request).

```text
HTTP Request
      │
      ▼
Create Scoped Instance
      │
      ▼
Reuse Within Request
      │
      ▼
Dispose At End Of Request
```

Registration:

```csharp
services.AddScoped<IOrderService, OrderService>();
```

---

## Transient

New instance every time.

```text
Resolve
   │
   ▼
New Object

Resolve Again
   │
   ▼
Another New Object
```

Registration:

```csharp
services.AddTransient<IEmailService, EmailService>();
```

---

# Step 9: Caching Object Factories

Instead of using reflection every time, the container compiles object creation into delegates.

Conceptually:

```text
Reflection
      │
      ▼
Compile Factory Delegate
      │
      ▼
Cache Delegate
      │
      ▼
Reuse Delegate
```

Equivalent concept:

```csharp
Func<IServiceProvider, object> factory;
```

This significantly improves performance.

---

# Step 10: IServiceScope

Every HTTP request creates a new scope.

```text
Root ServiceProvider
          │
          ▼
Create Scope
          │
          ▼
Resolve Scoped Services
          │
          ▼
Dispose Scope
```

Example:

```csharp
using var scope = provider.CreateScope();

var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
```

---

# Step 11: Circular Dependency Detection

Example:

```text
Service A
    │
    ▼
Service B
    │
    ▼
Service C
    │
    ▼
Service A
```

The container detects this cycle and throws an exception.

Example message:

```
A circular dependency was detected for the service.
```

---

# Step 12: Disposal

Services implementing `IDisposable` or `IAsyncDisposable` are disposed automatically.

| Lifetime | Disposal Time |
|-----------|---------------|
| Singleton | Application shutdown |
| Scoped | End of scope/request |
| Transient | End of scope (if created by the container) |

Example:

```csharp
public class Repository : IDisposable
{
    public void Dispose()
    {
        // Release resources
    }
}
```

---

# Complete Resolution Flow

```text
Application Starts
        │
        ▼
Register Services
        │
        ▼
ServiceCollection
        │
        ▼
BuildServiceProvider()
        │
        ▼
CallSiteFactory
        │
        ▼
Build Dependency Graph
        │
        ▼
Compile Factory Delegates
        │
        ▼
Application Request
        │
        ▼
Resolve Requested Service
        │
        ▼
Resolve Constructor Dependencies
        │
        ▼
Create Dependency Graph
        │
        ▼
Inject Dependencies
        │
        ▼
Return Object
        │
        ▼
Dispose According To Lifetime
```

---

# Internal Components

| Component | Responsibility |
|-----------|----------------|
| IServiceCollection | Stores service registrations |
| ServiceDescriptor | Stores registration metadata |
| BuildServiceProvider() | Builds the DI container |
| ServiceProvider | Resolves requested services |
| CallSiteFactory | Creates dependency graphs |
| ServiceProviderEngine | Executes object creation |
| IServiceScope | Manages scoped lifetimes |
| ActivatorUtilities | Creates objects using constructor injection |
| Constructor Resolver | Selects the constructor to use |
| Lifetime Cache | Stores singleton and scoped instances |

---

# Example Walkthrough

Registration:

```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IPaymentService, PaymentService>();
services.AddScoped<IRepository, Repository>();
services.AddDbContext<AppDbContext>();
```

Dependency hierarchy:

```text
OrderService
      │
      ▼
PaymentService
      │
      ▼
Repository
      │
      ▼
AppDbContext
```

Execution steps:

1. Request `IOrderService`.
2. Locate `OrderService` registration.
3. Inspect constructor.
4. Resolve `IPaymentService`.
5. Resolve `IRepository`.
6. Resolve `AppDbContext`.
7. Create `AppDbContext`.
8. Create `Repository`.
9. Create `PaymentService`.
10. Create `OrderService`.
11. Return the fully initialized object.

---

# Key Interview Points

- `IServiceCollection` stores registrations; it does not create objects.
- Every registration becomes a `ServiceDescriptor`.
- `BuildServiceProvider()` creates the runtime DI container.
- `CallSiteFactory` builds dependency graphs (call sites).
- Dependencies are resolved recursively using constructor injection.
- Compiled delegates are cached for better performance.
- Singleton services are created once and reused.
- Scoped services are created once per request or scope.
- Transient services are created every time they are requested.
- Circular dependencies are detected automatically.
- Disposable services are disposed according to their lifetime.
- The built-in .NET DI container supports constructor injection, open generics, scopes, and factory registrations, while remaining intentionally lightweight.
