# Options Pattern in .NET (IOptions, IOptionsSnapshot, IOptionsMonitor)

## Introduction

The **Options Pattern** is the recommended way to read configuration in ASP.NET Core.

Instead of reading configuration values directly from `IConfiguration`, the Options Pattern binds configuration sections to **strongly typed classes**, making the application:

- Type-safe
- Easier to test
- Easier to maintain
- Less error-prone
- Better aligned with Dependency Injection (DI)

Instead of writing:

```csharp
var connectionString = configuration["Database:ConnectionString"];
```

You write:

```csharp
public class DatabaseOptions
{
    public string ConnectionString { get; set; } = string.Empty;
}
```

And inject:

```csharp
public class OrderService
{
    private readonly DatabaseOptions _options;

    public OrderService(IOptions<DatabaseOptions> options)
    {
        _options = options.Value;
    }
}
```

---

# Why Use the Options Pattern?

Without the Options Pattern:

```csharp
public class OrderService
{
    private readonly IConfiguration _configuration;

    public OrderService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void Save()
    {
        var connection =
            _configuration["Database:ConnectionString"];
    }
}
```

Problems:

- String keys everywhere
- No compile-time checking
- Hard to unit test
- Easy to introduce typos

Example:

```csharp
_configuration["Databse:ConnectionString"]
```

This typo is only detected at runtime.

---

# Using Strongly Typed Configuration

appsettings.json

```json
{
  "Database": {
    "ConnectionString": "Server=.;Database=ShopDb;",
    "Timeout": 30
  }
}
```

Options class:

```csharp
public class DatabaseOptions
{
    public string ConnectionString { get; set; } = string.Empty;

    public int Timeout { get; set; }
}
```

Registration:

```csharp
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));
```

Injection:

```csharp
public class OrderService
{
    private readonly DatabaseOptions _options;

    public OrderService(IOptions<DatabaseOptions> options)
    {
        _options = options.Value;
    }
}
```

---

# Internal Working

```text
appsettings.json

        │

        ▼

Configuration Provider

        │

        ▼

Configuration Binder

        │

        ▼

DatabaseOptions Object

        │

        ▼

IOptions<T>

        │

        ▼

Injected Into Service
```

---

# What Happens Internally?

During startup:

```csharp
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));
```

The DI container stores:

```text
Database Section

↓

Configuration Binder

↓

DatabaseOptions

↓

Options Factory

↓

IOptions<DatabaseOptions>
```

The object is created when first requested.

---

# IOptions<T>

## Definition

`IOptions<T>` provides a **singleton** configuration object.

Registration:

```csharp
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));
```

Usage:

```csharp
public class OrderService
{
    private readonly DatabaseOptions _options;

    public OrderService(IOptions<DatabaseOptions> options)
    {
        _options = options.Value;
    }
}
```

---

## Lifecycle

```text
Application Starts

        │

Read Configuration

        │

Create Options Object

        │

Cache Object

        │

Reuse Everywhere
```

Configuration is read once.

---

## Characteristics

- Singleton
- Fast
- Configuration does **not** change while the application is running
- Best for static settings

---

## Best Use Cases

- Connection strings
- API endpoints
- Application settings
- Feature flags that rarely change

---

# IOptionsSnapshot<T>

## Definition

Creates a **new options object for every scope**.

In ASP.NET Core:

One HTTP request = One scope.

Usage:

```csharp
public class OrderService
{
    public OrderService(
        IOptionsSnapshot<DatabaseOptions> options)
    {
    }
}
```

---

## Lifecycle

```text
Request 1

↓

Read Configuration

↓

Create Options

↓

Dispose

-------------------------

Request 2

↓

Read Configuration Again

↓

Create New Options
```

Each request gets its own instance.

---

## Characteristics

- Scoped
- Reads updated configuration on the next request
- Only works inside request scopes
- Cannot be injected into Singleton services

---

## Example

Configuration:

```json
{
  "Discount": 10
}
```

Request 1:

```
Discount = 10
```

Change configuration:

```json
{
  "Discount": 20
}
```

Request 2:

```
Discount = 20
```

Request 1 still sees:

```
10
```

---

# IOptionsMonitor<T>

## Definition

Supports **live configuration changes**.

Usage:

```csharp
public class OrderService
{
    private readonly IOptionsMonitor<DatabaseOptions> _monitor;

    public OrderService(
        IOptionsMonitor<DatabaseOptions> monitor)
    {
        _monitor = monitor;
    }

    public void Execute()
    {
        var options = _monitor.CurrentValue;
    }
}
```

---

## Lifecycle

```text
Configuration File

        │

        ▼

File Watcher

        │

Configuration Changed

        │

        ▼

Rebind Options

        │

        ▼

CurrentValue Updated
```

No application restart required.

---

## Example

Configuration:

```json
{
  "Discount": 10
}
```

Application running:

```
CurrentValue = 10
```

Modify file:

```json
{
  "Discount": 20
}
```

Immediately:

```
CurrentValue = 20
```

---

# Change Notifications

`IOptionsMonitor` supports callbacks.

```csharp
public class NotificationService
{
    public NotificationService(
        IOptionsMonitor<DatabaseOptions> monitor)
    {
        monitor.OnChange(options =>
        {
            Console.WriteLine("Configuration Changed");
        });
    }
}
```

Every configuration change triggers the callback.

---

# Internal Working of IOptionsMonitor

```text
appsettings.json

        │

        ▼

Configuration Provider

        │

File Watcher

        │

Configuration Changed

        │

        ▼

Options Monitor

        │

Update CurrentValue

        │

Notify Subscribers
```

---

# Comparison

| Feature | IOptions | IOptionsSnapshot | IOptionsMonitor |
|----------|----------|------------------|-----------------|
| Lifetime | Singleton | Scoped | Singleton |
| Reads Configuration | Once | Per Request | Live Updates |
| Supports Reload | No | Yes (Next Request) | Yes (Immediate) |
| Can Be Injected into Singleton | Yes | No | Yes |
| CurrentValue Property | No | No | Yes |
| OnChange Callback | No | No | Yes |

---

# Lifetime Diagram

```text
                Application

                     │

────────────────────────────────

IOptions

One Instance

────────────────────────────────

IOptionsSnapshot

Request 1

Request 2

Request 3

────────────────────────────────

IOptionsMonitor

One Instance

Configuration Reloads

CurrentValue Changes
```

---

# Options Validation

Options can be validated during startup.

```csharp
public class DatabaseOptions
{
    public string ConnectionString { get; set; } = string.Empty;
}
```

Registration:

```csharp
builder.Services
    .AddOptions<DatabaseOptions>()
    .Bind(builder.Configuration.GetSection("Database"))
    .Validate(o => !string.IsNullOrWhiteSpace(o.ConnectionString),
              "ConnectionString is required.")
    .ValidateOnStart();
```

If validation fails, the application throws an exception during startup instead of failing later.

---

# Named Options

Useful when multiple configurations use the same options class.

Configuration:

```json
{
  "SqlDatabase": {
    "ConnectionString": "SQL"
  },
  "MongoDatabase": {
    "ConnectionString": "Mongo"
  }
}
```

Registration:

```csharp
builder.Services.Configure<DatabaseOptions>(
    "Sql",
    builder.Configuration.GetSection("SqlDatabase"));

builder.Services.Configure<DatabaseOptions>(
    "Mongo",
    builder.Configuration.GetSection("MongoDatabase"));
```

Usage:

```csharp
public class Demo
{
    public Demo(IOptionsSnapshot<DatabaseOptions> options)
    {
        var sql = options.Get("Sql");
        var mongo = options.Get("Mongo");
    }
}
```

---

# Common Mistakes

## Reading IConfiguration Everywhere

Bad:

```csharp
_configuration["Database:ConnectionString"]
```

Use strongly typed options instead.

---

## Injecting IOptionsSnapshot into Singleton

```csharp
public class CacheService
{
    public CacheService(
        IOptionsSnapshot<DatabaseOptions> options)
    {
    }
}
```

Result:

```
InvalidOperationException

Cannot consume scoped service
from singleton.
```

Use `IOptionsMonitor` instead.

---

## Expecting IOptions to Reload

```csharp
_options.Value
```

Changing `appsettings.json` will not update this value.

Use `IOptionsMonitor`.

---

## Forgetting Validation

Without validation:

```json
{
    "Timeout": -500
}
```

The application may start with invalid configuration.

Always validate important settings.

---

# Real-World Use Cases

## IOptions

- Connection strings
- Static application settings
- API URLs
- JWT configuration

---

## IOptionsSnapshot

- Per-request tenant configuration
- Request-specific feature flags
- Reload configuration between requests

---

## IOptionsMonitor

- Live feature flags
- Logging levels
- Rate limits
- Background services
- Long-running hosted services

---

# Which One Should You Use?

```text
Does configuration change while application runs?

        │

      No

        │

        ▼

     IOptions

        │

      Yes

        │

        ▼

Need updates immediately?

        │

     Yes

        │

        ▼

 IOptionsMonitor

        │

      No

        │

        ▼

IOptionsSnapshot
```

---

# Interview Questions

## Q1. What is the Options Pattern?

The Options Pattern binds configuration sections to strongly typed classes and integrates them with the DI container.

---

## Q2. What is the difference between IOptions, IOptionsSnapshot, and IOptionsMonitor?

| Type | Description |
|------|-------------|
| IOptions | Singleton configuration object created once. |
| IOptionsSnapshot | Scoped configuration object recreated for each request. |
| IOptionsMonitor | Singleton object that supports live configuration updates and change notifications. |

---

## Q3. Why can't IOptionsSnapshot be injected into a Singleton?

Because `IOptionsSnapshot` has a **Scoped** lifetime, while a Singleton lives for the application's lifetime. Injecting a scoped service into a singleton causes a lifetime mismatch.

---

## Q4. Which options type should a BackgroundService use?

`IOptionsMonitor<T>` because background services are singletons and may need updated configuration while running.

---

## Q5. How do you validate configuration?

```csharp
builder.Services
    .AddOptions<DatabaseOptions>()
    .Bind(builder.Configuration.GetSection("Database"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

or by using custom validation with `.Validate(...)`.

---

# Key Takeaways

- The **Options Pattern** provides strongly typed access to configuration.
- `IOptions<T>` is a singleton and is best for static configuration.
- `IOptionsSnapshot<T>` creates a new options instance for every request and is useful when configuration can change between requests.
- `IOptionsMonitor<T>` supports live configuration updates and change notifications, making it ideal for long-running services.
- Always validate important configuration during startup to catch errors early.
- Prefer the Options Pattern over reading values directly from `IConfiguration` throughout your application.