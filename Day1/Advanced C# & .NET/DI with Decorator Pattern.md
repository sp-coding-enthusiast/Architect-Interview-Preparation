# Decorator Pattern with Dependency Injection (.NET)

## Introduction

The **Decorator Pattern** is a structural design pattern that allows you to **add new behavior to an object without modifying its original implementation**.

Instead of changing an existing class, you wrap it inside another class (called a **Decorator**) that implements the same interface and delegates the original work while adding additional behavior.

With **Dependency Injection (DI)**, decorators are automatically composed by the DI container, making it easy to add cross-cutting concerns such as:

- Logging
- Caching
- Validation
- Authorization
- Metrics
- Retry
- Circuit Breaker

---

# Why Use the Decorator Pattern?

Suppose we have an order service.

```csharp
public interface IOrderService
{
    Task PlaceOrderAsync(Order order);
}
```

Basic implementation:

```csharp
public class OrderService : IOrderService
{
    public Task PlaceOrderAsync(Order order)
    {
        Console.WriteLine("Order placed");

        return Task.CompletedTask;
    }
}
```

Now the business requires:

- Logging
- Validation
- Caching
- Metrics

A common mistake is adding all of these inside `OrderService`.

```text
OrderService

↓

Validate

↓

Log

↓

Cache

↓

Business Logic

↓

Metrics
```

Problems:

- Violates Single Responsibility Principle
- Difficult to maintain
- Hard to test
- Difficult to reuse

Decorator Pattern solves this.

---

# High-Level Architecture

```text
Application

        │

        ▼

LoggingDecorator

        │

        ▼

ValidationDecorator

        │

        ▼

CachingDecorator

        │

        ▼

OrderService
```

Each decorator adds one responsibility.

---

# Step 1: Create the Interface

```csharp
public interface IOrderService
{
    Task PlaceOrderAsync(Order order);
}
```

---

# Step 2: Create the Original Service

```csharp
public class OrderService : IOrderService
{
    public async Task PlaceOrderAsync(Order order)
    {
        Console.WriteLine("Saving order...");

        await Task.CompletedTask;
    }
}
```

This class only contains business logic.

---

# Step 3: Create a Logging Decorator

```csharp
public class LoggingOrderService : IOrderService
{
    private readonly IOrderService _inner;

    public LoggingOrderService(IOrderService inner)
    {
        _inner = inner;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        Console.WriteLine("Logging Started");

        await _inner.PlaceOrderAsync(order);

        Console.WriteLine("Logging Finished");
    }
}
```

Notice:

- Implements the same interface
- Wraps another `IOrderService`
- Delegates work to the wrapped service

---

# Execution Flow

```text
Client

        │

        ▼

Logging Decorator

        │

Log Before

        │

        ▼

Order Service

        │

Business Logic

        │

        ▼

Logging Decorator

        │

Log After
```

---

# Step 4: Validation Decorator

```csharp
public class ValidationOrderService : IOrderService
{
    private readonly IOrderService _inner;

    public ValidationOrderService(IOrderService inner)
    {
        _inner = inner;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        if (order == null)
            throw new ArgumentNullException(nameof(order));

        await _inner.PlaceOrderAsync(order);
    }
}
```

---

# Multiple Decorators

Decorators can be stacked.

```text
Logging

      │

      ▼

Validation

      │

      ▼

Caching

      │

      ▼

Authorization

      │

      ▼

OrderService
```

Each decorator performs one task.

---

# Runtime Flow

```text
Application

        │

        ▼

LoggingDecorator

        │

Before

        ▼

ValidationDecorator

        │

Validate

        ▼

CachingDecorator

        │

Check Cache

        ▼

OrderService

        │

Business Logic

        ▼

Return Result
```

---

# Manual Composition

Without a DI container:

```csharp
IOrderService service =
    new LoggingOrderService(
        new ValidationOrderService(
            new OrderService()));
```

Although it works, it becomes difficult to maintain.

---

# Decorator with Dependency Injection

Register the original service.

```csharp
builder.Services.AddScoped<OrderService>();
```

Then register decorators.

Using **Scrutor** (recommended):

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();

builder.Services.Decorate<IOrderService, ValidationOrderService>();

builder.Services.Decorate<IOrderService, LoggingOrderService>();
```

Resolution:

```text
IOrderService

        │

LoggingDecorator

        │

ValidationDecorator

        │

OrderService
```

The DI container automatically builds the decorator chain.

---

# Internal Resolution

Suppose:

```csharp
var service =
    provider.GetRequiredService<IOrderService>();
```

Internally:

```text
Resolve IOrderService

        │

Create OrderService

        │

Wrap With Validation

        │

Wrap With Logging

        │

Return Final Object
```

---

# Without Scrutor

The built-in DI container does **not** support decorators directly.

Manual registration:

```csharp
builder.Services.AddScoped<OrderService>();

builder.Services.AddScoped<IOrderService>(sp =>
{
    var service =
        sp.GetRequiredService<OrderService>();

    return new LoggingOrderService(service);
});
```

For multiple decorators:

```csharp
builder.Services.AddScoped<IOrderService>(sp =>
{
    IOrderService service =
        new OrderService();

    service =
        new ValidationOrderService(service);

    service =
        new LoggingOrderService(service);

    return service;
});
```

Scrutor makes this much cleaner.

---

# Real-World Example: Logging

Original service:

```text
OrderService

↓

Save To Database
```

Decorator:

```text
LoggingDecorator

↓

Start Timer

↓

Call Service

↓

Stop Timer

↓

Write Log
```

The business service remains unchanged.

---

# Real-World Example: Caching

```text
Client

↓

CachingDecorator

↓

Is Cached?

↓

Yes → Return Cache

↓

No

↓

OrderService

↓

Save Result

↓

Return
```

---

# Real-World Example: Retry

```text
RetryDecorator

↓

Call API

↓

Failed?

↓

Retry

↓

Retry

↓

Success
```

Business logic doesn't know retries exist.

---

# Common Use Cases

## Logging

```text
Request

↓

Log

↓

Service
```

---

## Validation

```text
Validate

↓

Service
```

---

## Authorization

```text
Authorize

↓

Service
```

---

## Metrics

```text
Start Timer

↓

Service

↓

Record Duration
```

---

## Caching

```text
Check Cache

↓

Service

↓

Store Cache
```

---

## Retry

```text
Try

↓

Service

↓

Retry
```

---

## Circuit Breaker

```text
Circuit Open?

↓

No

↓

Service

↓

Failure Count
```

---

# Decorator vs Inheritance

## Inheritance

```text
Base Service

↓

Derived Service
```

Behavior is fixed at compile time.

---

## Decorator

```text
Logging

↓

Validation

↓

Caching

↓

Service
```

Behavior can be composed dynamically.

---

# Decorator vs Middleware

Middleware:

```text
HTTP Request

↓

Middleware

↓

Controller
```

Decorator:

```text
Method Call

↓

Decorator

↓

Business Service
```

Middleware works at the HTTP pipeline level.

Decorators work at the object level.

---

# Advantages

- Follows Single Responsibility Principle
- Follows Open/Closed Principle
- Easy to extend
- Reusable behaviors
- Better testability
- Clean separation of concerns
- No modification to original classes

---

# Disadvantages

- More classes
- Longer object chains
- More registrations
- Slight increase in complexity

---

# Common Mistakes

## Putting Logging Inside Business Logic

Bad:

```csharp
public async Task Save()
{
    Console.WriteLine("Start");

    // Business Logic

    Console.WriteLine("End");
}
```

Logging should be in a decorator.

---

## Forgetting to Call the Inner Service

Bad:

```csharp
public async Task Save()
{
    Console.WriteLine("Logging");
}
```

Business logic never executes.

Correct:

```csharp
await _inner.Save();
```

---

## Changing Business Logic in the Decorator

Decorators should **extend** behavior, not replace core business logic.

---

# Interview Questions

## Q1. What is the Decorator Pattern?

A structural design pattern that dynamically adds responsibilities to an object by wrapping it with another object implementing the same interface.

---

## Q2. Why use Decorator instead of inheritance?

Because decorators allow behavior to be composed dynamically at runtime without modifying existing classes or creating deep inheritance hierarchies.

---

## Q3. What problems does the Decorator Pattern solve?

- Logging
- Validation
- Caching
- Retry
- Metrics
- Authorization
- Cross-cutting concerns

---

## Q4. Does the built-in .NET DI container support decorators?

Not directly.

You can:

- Register them manually
- Use the **Scrutor** library, which provides the `Decorate()` extension method.

---

## Q5. What is the difference between Middleware and Decorator?

| Middleware | Decorator |
|------------|-----------|
| Works on HTTP requests | Works on object method calls |
| Executes once per request | Executes for each decorated service method |
| Used in ASP.NET Core pipeline | Used in business/service layer |

---

# Key Takeaways

- The Decorator Pattern wraps an object to add new behavior without modifying the original implementation.
- Decorators implement the same interface as the wrapped service.
- Each decorator should have a **single responsibility**.
- Decorators are ideal for cross-cutting concerns like logging, caching, validation, retry, and metrics.
- The built-in DI container doesn't natively support decorators, but **Scrutor** makes decorator registration simple with `Decorate<TService, TDecorator>()`.
- Decorators keep business logic clean while allowing behavior to be extended in a composable and testable way.