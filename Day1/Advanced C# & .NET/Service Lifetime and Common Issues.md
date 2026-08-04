# .NET Service Lifetimes (Singleton, Scoped, Transient)

## Introduction

In the .NET Dependency Injection (DI) container, every registered service has a **lifetime**. A service lifetime determines:

- **When an instance is created**
- **How long the instance lives**
- **Who shares the instance**
- **When the instance is disposed**

Choosing the correct lifetime is critical for application performance, memory usage, thread safety, and correctness.

---

# Three Service Lifetimes

The built-in DI container supports three lifetimes:

1. Singleton
2. Scoped
3. Transient

```text
                    DI Container
                         │
      ┌──────────────────┼──────────────────┐
      ▼                  ▼                  ▼
 Singleton           Scoped            Transient
 One Instance      One Per Scope      New Every Time
```

---

# 1. Singleton Lifetime

## Definition

A singleton service is created **only once** during the application's lifetime.

Every request receives the **same instance**.

Registration:

```csharp
services.AddSingleton<ICacheService, CacheService>();
```

---

## Lifecycle

```text
Application Starts
        │
        ▼
Create Singleton
        │
        ▼
Store In Root Container
        │
        ▼
Reuse For Every Request
        │
        ▼
Dispose When Application Stops
```

---

## Example

```csharp
public class CacheService
{
    public Guid Id { get; } = Guid.NewGuid();
}
```

Registration:

```csharp
services.AddSingleton<CacheService>();
```

Request 1:

```
Id = A123
```

Request 2:

```
Id = A123
```

Request 100:

```
Id = A123
```

Same instance every time.

---

## Memory Representation

```text
Root Service Provider
        │
        ▼
+---------------------+
| CacheService        |
| Id = A123           |
+---------------------+

Every request uses this instance.
```

---

## Best Use Cases

✅ Logging

✅ Configuration

✅ Caching

✅ Static reference data

✅ API clients (HttpClientFactory)

✅ Feature flags

---

## Avoid

Never store:

- User-specific data
- Request-specific state
- Mutable shared state without synchronization

Example (Bad):

```csharp
public class UserContext
{
    public string Username { get; set; }
}
```

If registered as Singleton:

```csharp
services.AddSingleton<UserContext>();
```

Multiple users would overwrite the same object.

---

# 2. Scoped Lifetime

## Definition

A scoped service is created **once per scope**.

In ASP.NET Core, one HTTP request equals one scope.

Registration:

```csharp
services.AddScoped<IOrderService, OrderService>();
```

---

## Lifecycle

```text
HTTP Request
      │
      ▼
Create Scoped Instance
      │
      ▼
Reuse During Request
      │
      ▼
Dispose At End Of Request
```

---

## Example

```csharp
public class OrderService
{
    public Guid Id { get; } = Guid.NewGuid();
}
```

Request 1:

```
Id = X123
```

Multiple resolutions in the same request:

```
Id = X123
```

Request 2:

```
Id = Y456
```

Each request gets its own instance.

---

## Memory Representation

```text
HTTP Request 1

OrderService
Id = X123

-----------------------

HTTP Request 2

OrderService
Id = Y456
```

---

## Best Use Cases

✅ Entity Framework DbContext

✅ Repository pattern

✅ Unit of Work

✅ Business services

✅ Request context

✅ Current user information

---

## Why DbContext Is Scoped

Each request should have:

- One database transaction
- One change tracker
- One identity map

Sharing DbContext across requests would cause severe concurrency issues.

---

# 3. Transient Lifetime

## Definition

A transient service creates a **new instance every time it is requested**.

Registration:

```csharp
services.AddTransient<IEmailSender, EmailSender>();
```

---

## Lifecycle

```text
Resolve Service
      │
      ▼
Create Object

Resolve Again
      │
      ▼
Create Another Object
```

---

## Example

```csharp
public class EmailSender
{
    public Guid Id { get; } = Guid.NewGuid();
}
```

First resolution:

```
Id = A111
```

Second resolution:

```
Id = B222
```

Third resolution:

```
Id = C333
```

Every resolution gets a new instance.

---

## Best Use Cases

✅ Lightweight services

✅ Stateless utilities

✅ Email sender

✅ PDF generator

✅ Validators

✅ Mapping services

---

# Lifetime Comparison

| Feature | Singleton | Scoped | Transient |
|----------|-----------|---------|-----------|
| Instances | One | One per request | New every time |
| Shared Between Requests | Yes | No | No |
| Memory Usage | Lowest | Medium | Highest |
| Thread Safety Required | Yes | Usually No | No |
| Disposal | App shutdown | End of request | End of scope |

---

# Example Timeline

```text
Application Starts

Singleton Created

│

├──────────── Request 1 ─────────────

Scoped Created

Transient A

Transient B

Scoped Disposed

│

├──────────── Request 2 ─────────────

Scoped Created

Transient C

Transient D

Scoped Disposed

│

Application Stops

Singleton Disposed
```

---

# Common Lifetime Issues

## Issue 1: Scoped Service Inside Singleton

Example:

```csharp
public class CacheService
{
    private readonly DbContext _dbContext;

    public CacheService(DbContext dbContext)
    {
        _dbContext = dbContext;
    }
}
```

Registration:

```csharp
services.AddSingleton<CacheService>();
services.AddScoped<AppDbContext>();
```

Result:

```
InvalidOperationException

Cannot consume scoped service
from singleton.
```

### Why?

```text
Singleton

Lives Forever

↓

DbContext

Lives One Request

↓

Request Ends

↓

DbContext Disposed

↓

Singleton Still Holds Reference
```

The singleton would reference a disposed object.

---

## Solution

Inject `IServiceScopeFactory`.

```csharp
public class CacheService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public CacheService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public void Execute()
    {
        using var scope = _scopeFactory.CreateScope();

        var db = scope.ServiceProvider
                      .GetRequiredService<AppDbContext>();
    }
}
```

---

## Issue 2: Singleton Holding Mutable State

Bad example:

```csharp
public class Counter
{
    public int Count;
}
```

Registration:

```csharp
services.AddSingleton<Counter>();
```

Requests:

```
Request 1

Count = 5

Request 2

Count = 6

Request 3

Count = 7
```

All users share the same data.

### Solution

Keep singleton services:

- Immutable
- Thread-safe
- Stateless (when possible)

---

## Issue 3: Transient Captured by Singleton

```csharp
services.AddTransient<IEmailSender, EmailSender>();
services.AddSingleton<NotificationManager>();
```

```csharp
public class NotificationManager
{
    public NotificationManager(IEmailSender sender)
    {
    }
}
```

Although `IEmailSender` is transient, it is created **once** when the singleton is constructed and reused thereafter.

This defeats the purpose of a transient lifetime.

---

## Issue 4: Expensive Transient Objects

Bad example:

```csharp
services.AddTransient<HeavyModel>();
```

If `HeavyModel`:

- Loads large files
- Creates expensive connections
- Allocates significant memory

A new object is created on every resolution, leading to poor performance.

### Solution

Use Scoped or Singleton when appropriate.

---

## Issue 5: Thread Safety in Singleton

Example:

```csharp
public class Counter
{
    public int Value;

    public void Increment()
    {
        Value++;
    }
}
```

Multiple requests:

```
Thread 1

Read Value = 10

Thread 2

Read Value = 10

Thread 1

Write 11

Thread 2

Write 11
```

Expected:

```
12
```

Actual:

```
11
```

This is a race condition.

### Solution

Use synchronization primitives:

```csharp
lock (_lock)
{
    Value++;
}
```

Or use thread-safe collections such as:

- ConcurrentDictionary
- ConcurrentQueue
- ConcurrentBag

---

## Issue 6: Registering DbContext as Singleton

Incorrect:

```csharp
services.AddSingleton<AppDbContext>();
```

Problems:

- Shared change tracker
- Concurrency exceptions
- Memory growth
- Stale entity tracking

Correct:

```csharp
services.AddDbContext<AppDbContext>();
```

This registers the `DbContext` as **Scoped** by default.

---

# Choosing the Right Lifetime

| Scenario | Recommended Lifetime |
|----------|----------------------|
| Configuration | Singleton |
| Logging | Singleton |
| Caching | Singleton |
| HttpClient (via IHttpClientFactory) | Singleton Factory |
| Entity Framework DbContext | Scoped |
| Repository | Scoped |
| Business Service | Scoped |
| Unit of Work | Scoped |
| Validator | Transient |
| Email Sender | Transient |
| PDF Generator | Transient |
| Object Mapper | Singleton or Transient (depending on implementation) |

---

# Decision Flow

```text
Does the service maintain request-specific state?

        │
       Yes
        │
        ▼
     Scoped

        │
       No
        │
        ▼

Is the service expensive to create?

        │
      Yes
        │
        ▼
    Singleton

        │
      No
        │
        ▼

Is the service stateless?

        │
      Yes
        │
        ▼
    Transient
```

---

# Interview Questions

### Q1. What is the difference between Singleton, Scoped, and Transient?

- **Singleton:** One instance for the entire application.
- **Scoped:** One instance per HTTP request (scope).
- **Transient:** A new instance every time it is requested.

---

### Q2. Why is `DbContext` registered as Scoped?

Because each HTTP request should have its own:

- Database transaction
- Change tracker
- Entity state

Sharing a `DbContext` across requests causes concurrency and data consistency issues.

---

### Q3. Why can't a Singleton depend on a Scoped service?

A singleton lives for the application's lifetime, while a scoped service is disposed at the end of a request. Holding a reference to a disposed scoped service leads to runtime errors and invalid state.

---

### Q4. When should you use Transient?

Use Transient for:

- Lightweight
- Stateless
- Short-lived services

Examples:

- Validators
- Email senders
- Formatters
- Utility services

---

# Key Takeaways

- **Singleton**: One instance for the application's lifetime. Must be thread-safe.
- **Scoped**: One instance per request. Ideal for business logic and database access.
- **Transient**: New instance every resolution. Best for lightweight, stateless services.
- Never inject a **Scoped** service directly into a **Singleton**.
- Be mindful of performance when registering expensive services as **Transient**.
- Always consider thread safety when designing **Singleton** services.