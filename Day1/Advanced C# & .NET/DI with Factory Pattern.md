# Factory Pattern with Dependency Injection (.NET)

## Introduction

The **Factory Pattern** is a creational design pattern that encapsulates object creation logic inside a factory instead of creating objects directly using `new`.

When combined with **Dependency Injection (DI)**, the factory itself is managed by the DI container. This allows:

- Runtime object selection
- Loose coupling
- Easy testing
- Better maintainability
- Support for multiple implementations

Instead of this:

```csharp
var payment = new CreditCardPaymentService();
```

You use:

```csharp
var payment = paymentFactory.Create("CreditCard");
```

The factory decides which implementation to return.

---

# Why Use Factory Pattern?

Suppose your application supports multiple payment providers.

Without a factory:

```csharp
if (paymentType == "Card")
{
    return new CreditCardPaymentService();
}
else
{
    return new PaypalPaymentService();
}
```

Problems:

- Tight coupling
- Difficult to test
- Hard to extend
- Violates Open/Closed Principle

Factory Pattern solves this.

---

# High-Level Architecture

```text
                Client
                  │
                  ▼
          IPaymentFactory
                  │
                  ▼
          PaymentFactory
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
CreditCardPayment      PaypalPayment
```

The client knows only about the factory.

---

# Step 1: Create the Interface

```csharp
public interface IPaymentService
{
    void Pay(decimal amount);
}
```

---

# Step 2: Create Implementations

```csharp
public class CreditCardPaymentService : IPaymentService
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Paid {amount} using Credit Card");
    }
}
```

```csharp
public class PaypalPaymentService : IPaymentService
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Paid {amount} using PayPal");
    }
}
```

---

# Step 3: Create the Factory Interface

```csharp
public interface IPaymentFactory
{
    IPaymentService Create(string paymentType);
}
```

---

# Step 4: Implement the Factory

The factory receives services from the DI container.

```csharp
public class PaymentFactory : IPaymentFactory
{
    private readonly IServiceProvider _serviceProvider;

    public PaymentFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IPaymentService Create(string paymentType)
    {
        return paymentType switch
        {
            "Card" =>
                _serviceProvider.GetRequiredService<CreditCardPaymentService>(),

            "Paypal" =>
                _serviceProvider.GetRequiredService<PaypalPaymentService>(),

            _ => throw new NotSupportedException()
        };
    }
}
```

Notice that the factory **does not create objects using `new`**.

Instead, it asks the DI container to resolve them.

---

# Step 5: Register Services

```csharp
builder.Services.AddScoped<CreditCardPaymentService>();

builder.Services.AddScoped<PaypalPaymentService>();

builder.Services.AddScoped<IPaymentFactory, PaymentFactory>();
```

---

# Step 6: Use the Factory

```csharp
public class CheckoutService
{
    private readonly IPaymentFactory _factory;

    public CheckoutService(IPaymentFactory factory)
    {
        _factory = factory;
    }

    public void Checkout(string type)
    {
        var payment = _factory.Create(type);

        payment.Pay(500);
    }
}
```

---

# Runtime Flow

```text
CheckoutService

        │

Requests

IPaymentFactory

        │

DI Creates

PaymentFactory

        │

Factory Receives

IServiceProvider

        │

Factory Calls

GetRequiredService()

        │

DI Creates

CreditCardPaymentService

        │

Returns Service
```

---

# Internal Working

Suppose:

```csharp
_factory.Create("Card");
```

Internally:

```text
Factory

        │

Checks "Card"

        │

Calls

GetRequiredService<CreditCardPaymentService>()

        │

DI Finds Registration

        │

Creates Dependencies

        │

Returns Instance
```

The factory delegates object creation to the DI container.

---

# Why Not Use `new`?

Bad:

```csharp
return new CreditCardPaymentService();
```

Problems:

- Dependencies are not injected.
- Lifetime management is bypassed.
- Harder to unit test.
- Configuration is ignored.

Good:

```csharp
_serviceProvider.GetRequiredService<CreditCardPaymentService>();
```

Benefits:

- Constructor dependencies are injected.
- Lifetime is respected.
- Easy to mock in tests.
- Centralized object creation.

---

# Factory Using IEnumerable<T>

A cleaner approach avoids `switch` statements.

Interface:

```csharp
public interface IPaymentService
{
    string Name { get; }

    void Pay(decimal amount);
}
```

Implementation:

```csharp
public class CreditCardPaymentService : IPaymentService
{
    public string Name => "Card";

    public void Pay(decimal amount)
    {
    }
}
```

```csharp
public class PaypalPaymentService : IPaymentService
{
    public string Name => "Paypal";

    public void Pay(decimal amount)
    {
    }
}
```

Factory:

```csharp
public class PaymentFactory : IPaymentFactory
{
    private readonly IEnumerable<IPaymentService> _services;

    public PaymentFactory(IEnumerable<IPaymentService> services)
    {
        _services = services;
    }

    public IPaymentService Create(string type)
    {
        return _services.First(x => x.Name == type);
    }
}
```

Registration:

```csharp
builder.Services.AddScoped<IPaymentService, CreditCardPaymentService>();

builder.Services.AddScoped<IPaymentService, PaypalPaymentService>();

builder.Services.AddScoped<IPaymentFactory, PaymentFactory>();
```

---

# Resolution Flow with IEnumerable

```text
DI Container

        │

Find All

IPaymentService

        │

Creates List

─────────────────────────

CreditCardPaymentService

PaypalPaymentService

─────────────────────────

        │

Inject List

        │

Factory Selects One
```

No switch statement is required.

---

# Factory with Func<T>

Sometimes only one object needs deferred creation.

Registration:

```csharp
builder.Services.AddScoped<MyService>();

builder.Services.AddScoped<Func<MyService>>(sp =>
{
    return () => sp.GetRequiredService<MyService>();
});
```

Usage:

```csharp
public class Demo
{
    private readonly Func<MyService> _factory;

    public Demo(Func<MyService> factory)
    {
        _factory = factory;
    }

    public void Execute()
    {
        var service = _factory();
    }
}
```

The object is created only when needed.

---

# Factory vs Direct DI

## Direct DI

```csharp
public class CheckoutService
{
    public CheckoutService(IPaymentService payment)
    {
    }
}
```

Best when only one implementation exists.

---

## Factory

```csharp
public class CheckoutService
{
    public CheckoutService(IPaymentFactory factory)
    {
    }
}
```

Best when multiple implementations exist and the implementation is chosen at runtime.

---

# Real-World Use Cases

## Payment Gateways

```text
Payment Type

      │

      ▼

Payment Factory

      │

───────────────

Stripe

PayPal

Razorpay

Adyen

───────────────
```

---

## Notification Providers

```text
Email

SMS

Push Notification

WhatsApp

Teams

Slack
```

---

## File Storage

```text
Local Storage

Azure Blob

Amazon S3

Google Cloud Storage
```

---

## Database Providers

```text
SQL Server

PostgreSQL

MySQL

Oracle
```

---

## AI Model Selection

```text
OpenAI

Azure OpenAI

Claude

Gemini

Llama
```

The factory chooses the correct implementation based on configuration or user input.

---

# Advantages

- Encapsulates object creation.
- Reduces coupling.
- Supports runtime implementation selection.
- Works seamlessly with DI.
- Improves testability.
- Respects service lifetimes.
- Easier to extend with new implementations.

---

# Common Mistakes

## Creating Objects with `new`

```csharp
return new PaypalPaymentService();
```

Bypasses DI and lifetime management.

---

## Injecting IServiceProvider Everywhere

Bad:

```csharp
public class OrderService
{
    public OrderService(IServiceProvider provider)
    {
    }
}
```

This is considered the **Service Locator anti-pattern** because the class hides its dependencies.

Use `IServiceProvider` only inside infrastructure components like factories.

---

## Large Switch Statements

A factory with dozens of `switch` cases becomes difficult to maintain.

Better alternatives:

- `IEnumerable<T>`
- Dictionary-based lookup
- Keyed Services (.NET 8+)

---

# Interview Questions

## Q1. Why use the Factory Pattern with DI?

To choose between multiple implementations at runtime while allowing the DI container to manage object creation and lifetimes.

---

## Q2. Why shouldn't the factory use `new`?

Using `new` bypasses dependency injection, ignores configured lifetimes, and makes testing more difficult.

---

## Q3. When should you use a factory?

Use a factory when:

- Multiple implementations exist.
- The implementation depends on runtime data.
- Object creation is complex.
- You want to hide creation logic from clients.

---

## Q4. Is using `IServiceProvider` inside a factory acceptable?

Yes. A factory is an infrastructure component responsible for creating objects, so using `IServiceProvider` there is appropriate. Injecting `IServiceProvider` into business services, however, is generally discouraged because it hides dependencies.

---

## Q5. What is the best approach for selecting implementations?

Options include:

- `switch` statement (simple scenarios)
- `IEnumerable<T>` (clean and extensible)
- Dictionary lookup (fast runtime selection)
- Keyed Services in .NET 8+ (recommended for many scenarios)

---

# Key Takeaways

- The Factory Pattern encapsulates object creation.
- With DI, factories **resolve** services instead of creating them with `new`.
- Factories are ideal when implementation selection happens at runtime.
- Prefer `IEnumerable<T>` or Keyed Services over large `switch` statements as the number of implementations grows.
- Use `IServiceProvider` inside factories sparingly and avoid using it in business logic.
- Factory Pattern and Dependency Injection complement each other to produce loosely coupled, testable, and maintainable applications.