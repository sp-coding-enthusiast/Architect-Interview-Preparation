# Open Generic Registrations in .NET Dependency Injection

## Introduction

One of the powerful features of the .NET Dependency Injection (DI) container is **Open Generic Registration**.

Instead of registering every generic type individually, you can register a generic type definition once, and the DI container will automatically create the appropriate closed generic type when requested.

This greatly reduces boilerplate code and makes applications more scalable and maintainable.

---

# What is a Generic Type?

A generic type is a class or interface that works with different data types.

Example:

```csharp
public interface IRepository<T>
{
    T GetById(int id);
    void Add(T entity);
}
```

Implementation:

```csharp
public class Repository<T> : IRepository<T>
{
    public T GetById(int id)
    {
        throw new NotImplementedException();
    }

    public void Add(T entity)
    {
    }
}
```

Here:

- `IRepository<T>` is an **open generic interface**
- `Repository<T>` is an **open generic implementation**

No concrete type (`Customer`, `Product`, etc.) has been specified yet.

---

# Open Generic vs Closed Generic

## Open Generic

A generic type without specifying its type parameter.

```csharp
IRepository<T>

Repository<T>

List<T>

Dictionary<TKey, TValue>
```

These are templates.

---

## Closed Generic

A generic type with actual type arguments.

```csharp
IRepository<Customer>

Repository<Customer>

List<string>

Dictionary<int, string>
```

These are concrete types that can be instantiated.

---

# Without Open Generic Registration

Suppose you have three entities.

```csharp
public class Customer
{
}

public class Product
{
}

public class Order
{
}
```

Without open generics, you must register every repository individually.

```csharp
services.AddScoped<IRepository<Customer>, Repository<Customer>>();

services.AddScoped<IRepository<Product>, Repository<Product>>();

services.AddScoped<IRepository<Order>, Repository<Order>>();
```

As your application grows:

- 10 entities → 10 registrations
- 100 entities → 100 registrations
- 500 entities → 500 registrations

This quickly becomes difficult to maintain.

---

# Open Generic Registration

Instead of registering each type separately, register the generic type definition once.

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Notice the use of:

```csharp
typeof(IRepository<>)

typeof(Repository<>)
```

The `<>` indicates an **open generic type definition**.

---

# How the DI Container Resolves It

Registration:

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Application requests:

```csharp
IRepository<Customer>
```

Internally:

```text
Requested Type

IRepository<Customer>

        │

        ▼

Find Registration

IRepository<>

        │

        ▼

Repository<>

        │

        ▼

Close Generic

Repository<Customer>

        │

        ▼

Create Object
```

The container automatically substitutes `T` with `Customer`.

---

# Example

Registration:

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Constructor:

```csharp
public class CustomerService
{
    private readonly IRepository<Customer> _repository;

    public CustomerService(IRepository<Customer> repository)
    {
        _repository = repository;
    }
}
```

Resolution flow:

```text
CustomerService

        │

Needs IRepository<Customer>

        │

DI Container Finds

IRepository<>

        │

Uses

Repository<>

        │

Creates

Repository<Customer>

        │

Injects Into

CustomerService
```

No additional registration is required.

---

# Another Example

Suppose you request:

```csharp
IRepository<Product>
```

The container creates:

```text
Repository<Product>
```

If later you request:

```csharp
IRepository<Order>
```

The container creates:

```text
Repository<Order>
```

One registration supports every entity type.

---

# Internal Resolution Process

When resolving:

```csharp
IRepository<Customer>
```

The container performs:

```text
Requested Type

IRepository<Customer>

        │

        ▼

Is Exact Match Registered?

        │

       No

        │

        ▼

Search Open Generic

IRepository<>

        │

        ▼

Found

Repository<>

        │

        ▼

Create Closed Generic

Repository<Customer>

        │

        ▼

Resolve Constructor

        │

        ▼

Return Object
```

---

# Generic Constraints

Suppose:

```csharp
public class Repository<T>
    where T : class
{
}
```

Registration remains the same.

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

The container validates constraints at runtime.

Valid:

```csharp
IRepository<Customer>
```

Invalid:

```csharp
IRepository<int>
```

Result:

```
ArgumentException

Generic arguments do not satisfy
the constraints.
```

---

# Multiple Generic Parameters

Open generics also work with multiple type parameters.

Example:

```csharp
public interface IRepository<TKey, TValue>
{
}
```

Implementation:

```csharp
public class Repository<TKey, TValue>
    : IRepository<TKey, TValue>
{
}
```

Registration:

```csharp
services.AddScoped(
    typeof(IRepository<,>),
    typeof(Repository<,>));
```

Notice:

```text
<,>
```

indicates two generic parameters.

Examples:

```csharp
IRepository<int, Customer>

IRepository<Guid, Product>
```

The DI container creates the corresponding implementation automatically.

---

# Lifetime with Open Generics

Open generic registrations support all standard lifetimes.

Singleton:

```csharp
services.AddSingleton(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Scoped:

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Transient:

```csharp
services.AddTransient(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Each closed generic follows the configured lifetime.

---

# Constructor Dependency Resolution

Suppose:

```csharp
public class Repository<T>
{
    public Repository(AppDbContext dbContext)
    {
    }
}
```

Resolution:

```text
Repository<Customer>

        │

Needs AppDbContext

        │

Resolve DbContext

        │

Create Repository<Customer>
```

The generic type participates in dependency injection like any other service.

---

# Typical Use Cases

## Generic Repository

```csharp
IRepository<T>

Repository<T>
```

---

## Generic Cache

```csharp
ICache<T>

MemoryCache<T>
```

---

## Generic Validator

```csharp
IValidator<T>

Validator<T>
```

---

## Generic Event Handler

```csharp
IEventHandler<TEvent>

EventHandler<TEvent>
```

---

## Generic Service

```csharp
IService<T>

Service<T>
```

---

## Generic Factory

```csharp
IFactory<T>

Factory<T>
```

---

# Advantages

- Eliminates repetitive registrations.
- Automatically supports new entity types.
- Improves maintainability.
- Reduces startup configuration.
- Encourages reusable generic implementations.
- Works seamlessly with constructor injection.

---

# Limitations

The built-in DI container:

- Resolves open generic registrations automatically.
- Supports generic constraints.
- Supports multiple generic parameters.

However, it does **not** support:

- Conditional generic registrations.
- Named generic registrations.
- Key-based generic registrations (prior to .NET 8 keyed services).
- Automatic selection between multiple generic implementations based on runtime conditions.

For advanced scenarios, containers such as **Autofac**, **Lamar**, or **Simple Injector** provide additional capabilities.

---

# Real-World Example (Repository Pattern)

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task AddAsync(T entity);
}

public class Repository<T> : IRepository<T>
    where T : class
{
    private readonly AppDbContext _dbContext;

    public Repository(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbContext.Set<T>().FindAsync(id);
    }

    public async Task AddAsync(T entity)
    {
        await _dbContext.Set<T>().AddAsync(entity);
    }
}
```

Registration:

```csharp
services.AddDbContext<AppDbContext>();

services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Usage:

```csharp
public class ProductService
{
    private readonly IRepository<Product> _repository;

    public ProductService(IRepository<Product> repository)
    {
        _repository = repository;
    }
}
```

No explicit registration for `Product` is required.

---

# Interview Questions

## Q1. What is an open generic registration?

An open generic registration registers a generic type definition (for example, `IRepository<>`) so the DI container can automatically create closed generic instances such as `IRepository<Customer>` or `IRepository<Order>` when needed.

---

## Q2. Why use open generic registrations?

They eliminate repetitive registrations, reduce configuration, improve maintainability, and automatically support new generic types.

---

## Q3. How do you register an open generic service?

```csharp
services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

---

## Q4. Can open generics have multiple type parameters?

Yes.

Example:

```csharp
services.AddScoped(
    typeof(IRepository<,>),
    typeof(Repository<,>));
```

---

## Q5. What happens when `IRepository<Customer>` is requested?

The DI container:

1. Finds the open generic registration (`IRepository<>`).
2. Creates the closed generic implementation (`Repository<Customer>`).
3. Resolves its constructor dependencies.
4. Returns the fully initialized instance.

---

# Key Takeaways

- An **open generic** is a generic type definition without concrete type arguments.
- A **closed generic** specifies actual type arguments.
- Register open generics using `typeof(IRepository<>)` and `typeof(Repository<>)`.
- One registration can support unlimited entity types.
- Open generic registrations work with **Singleton**, **Scoped**, and **Transient** lifetimes.
- The DI container automatically closes the generic type and performs constructor injection during resolution.
- Open generics are widely used in repository, caching, validation, messaging, and service abstractions.