# Distributed Cache in .NET

# Introduction

A **Distributed Cache** is a cache that is shared across multiple application instances (servers).

Unlike an in-memory cache, where data is stored inside a single application's memory, a distributed cache stores data in a separate cache server that every application instance can access.

Common distributed cache technologies include:

- Redis
- Azure Cache for Redis
- NCache
- Memcached
- SQL Server Distributed Cache

---

# Why Do We Need Distributed Cache?

Imagine an application running on multiple servers behind a load balancer.

```text
                Users

                   │

        ┌──────────┴──────────┐

        ▼                     ▼

   Application 1         Application 2

        │                     │

        ▼                     ▼

     Database             Database
```

If each application keeps its own cache, the cached data is not shared.

Example:

```text
Application 1

User Profile Cached

Application 2

Cache Empty
```

This causes:

- Duplicate database calls
- Inconsistent cache
- Poor scalability

---

# Distributed Cache Solution

Instead of each server having its own cache:

```text
                 Users

                    │

        ┌───────────┴───────────┐

        ▼                       ▼

  Application 1          Application 2

        │                       │

        └──────────┬────────────┘

                   ▼

          Distributed Cache

                   │

                   ▼

               Database
```

Every application shares the same cache.

---

# In-Memory Cache vs Distributed Cache

## In-Memory Cache

```text
Application

↓

Memory Cache

↓

Database
```

Cache exists only inside one application.

---

## Distributed Cache

```text
Application

↓

Redis

↓

Database
```

Cache is shared across all servers.

---

# Comparison

| Feature | In-Memory Cache | Distributed Cache |
|----------|-----------------|-------------------|
| Shared Across Servers | ❌ No | ✅ Yes |
| Survives Application Restart | ❌ No | ✅ Yes |
| Fastest Access | ✅ Yes | Slightly slower |
| Horizontal Scaling | ❌ Limited | ✅ Excellent |
| Suitable for Web Farms | ❌ No | ✅ Yes |

---

# Distributed Cache Architecture

```text
                    Client

                       │

                       ▼

                Load Balancer

           ┌───────────┴───────────┐

           ▼                       ▼

     ASP.NET Core            ASP.NET Core

      Instance 1              Instance 2

           │                       │

           └───────────┬───────────┘

                       ▼

             Distributed Cache

                  (Redis)

                       │

                       ▼

                  SQL Database
```

All instances use the same cache.

---

# How Distributed Cache Works

Suppose a client requests product information.

```text
Client

↓

Application

↓

Redis

↓

Hit?

│         │

Yes       No

│          │

▼          ▼

Return    Database

             │

             ▼

      Store In Redis

             │

             ▼

        Return Data
```

---

# IDistributedCache Interface

ASP.NET Core provides the `IDistributedCache` interface.

```csharp
public interface IDistributedCache
{
    Task<byte[]> GetAsync(string key);

    Task SetAsync(
        string key,
        byte[] value);

    Task RemoveAsync(string key);

    Task RefreshAsync(string key);
}
```

All distributed cache providers implement this interface.

---

# Registering Redis

Install package:

```text
Microsoft.Extensions.Caching.StackExchangeRedis
```

Registration:

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";

    options.InstanceName = "Shop";
});
```

Now `IDistributedCache` is available through DI.

---

# Using IDistributedCache

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;

    public ProductService(IDistributedCache cache)
    {
        _cache = cache;
    }
}
```

---

# Storing Data

`IDistributedCache` stores **byte arrays**, so objects are typically serialized.

Example using JSON:

```csharp
var product = new Product
{
    Id = 1,
    Name = "Laptop"
};

var json = JsonSerializer.Serialize(product);

await _cache.SetStringAsync(
    "product:1",
    json);
```

---

# Reading Data

```csharp
var json = await _cache.GetStringAsync("product:1");

if (json != null)
{
    var product =
        JsonSerializer.Deserialize<Product>(json);
}
```

---

# Cache Expiration

Distributed caches support expiration policies.

---

## Absolute Expiration

Expires at a fixed time.

```csharp
await _cache.SetStringAsync(
    "product",
    json,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow =
            TimeSpan.FromMinutes(30)
    });
```

Timeline:

```text
Created

↓

30 Minutes

↓

Removed
```

---

## Sliding Expiration

Expiration resets whenever the item is accessed.

```csharp
await _cache.SetStringAsync(
    "product",
    json,
    new DistributedCacheEntryOptions
    {
        SlidingExpiration =
            TimeSpan.FromMinutes(20)
    });
```

Timeline:

```text
Created

↓

Access

↓

20 Minutes Reset

↓

Access Again

↓

Reset Again
```

---

## Combined Expiration

```csharp
new DistributedCacheEntryOptions
{
    SlidingExpiration =
        TimeSpan.FromMinutes(10),

    AbsoluteExpirationRelativeToNow =
        TimeSpan.FromHours(2)
}
```

The cache expires after **2 hours**, even if it is accessed every few minutes.

---

# Internal Working

```text
Application

↓

Serialize Object

↓

Redis

↓

Store Byte[]

↓

Later

↓

Retrieve Byte[]

↓

Deserialize

↓

Return Object
```

---

# Popular Distributed Cache Providers

## Redis

Most popular.

Features:

- Extremely fast
- In-memory
- Replication
- Clustering
- Persistence
- Pub/Sub

---

## Azure Cache for Redis

Managed Redis service.

Benefits:

- High availability
- Monitoring
- Automatic patching
- Scaling
- Geo-replication

---

## SQL Server Cache

Stores cache inside SQL Server.

Useful when Redis is unavailable.

Slower than Redis.

---

## NCache

.NET-focused distributed cache.

Supports:

- Replication
- Clustering
- High availability

---

## Memcached

Simple distributed cache.

Very fast.

Does not support persistence.

---

# Distributed Cache vs In-Memory Cache

Suppose two application servers.

## Memory Cache

```text
Application 1

Product Cached

Application 2

Cache Empty
```

Two database reads.

---

## Redis

```text
Application 1

↓

Redis

↓

Database

Application 2

↓

Redis

↓

Return Cached Data
```

One database read.

---

# Common Use Cases

## Session Storage

```text
User Login

↓

Redis

↓

Any Server Can Read Session
```

---

## Product Catalog

```text
Product

↓

Redis

↓

All Servers Share
```

---

## Authentication Tokens

```text
JWT Metadata

↓

Redis
```

---

## Shopping Cart

```text
Cart

↓

Redis
```

---

## Frequently Read Data

- Country list
- Currency rates
- Product details
- Exchange rates
- Application configuration

---

# Cache Invalidation

Keeping cached data synchronized with the database is one of the hardest challenges.

Example:

```text
Update Product

↓

Database Updated

↓

Redis Still Has Old Data
```

Users receive stale data.

---

## Solution 1

Remove cache after update.

```csharp
await _cache.RemoveAsync("product:1");
```

Next request reloads fresh data.

---

## Solution 2

Update cache immediately.

```text
Update Database

↓

Update Redis

↓

Return
```

---

# Advantages

- Shared by all application servers
- Reduces database load
- Improves scalability
- Survives application restarts
- High availability
- Supports cloud-native applications
- Ideal for microservices

---

# Disadvantages

- Additional infrastructure
- Network latency
- Serialization/deserialization overhead
- Cache invalidation complexity
- Eventual consistency in some scenarios

---

# Best Practices

- Cache only frequently accessed data.
- Set appropriate expiration policies.
- Avoid caching extremely large objects.
- Use meaningful cache keys.
- Remove or update cache after data modifications.
- Monitor cache hit ratio.
- Compress large payloads if necessary.
- Prefer Redis for production distributed caching.

---

# Real-World Example

E-commerce application.

Without cache:

```text
User

↓

Application

↓

Database

↓

Return Product
```

Every request queries the database.

---

With Redis:

```text
User

↓

Application

↓

Redis

↓

Hit?

│        │

Yes      No

│         │

▼         ▼

Return   Database

            │

            ▼

      Save To Redis

            │

            ▼

         Return
```

Most requests never reach the database.

---

# Interview Questions

## Q1. What is a distributed cache?

A distributed cache is a shared cache that multiple application instances can access. Unlike in-memory caching, the cache is stored outside the application process, enabling data sharing across servers.

---

## Q2. Why use a distributed cache instead of an in-memory cache?

Because in-memory cache is local to a single application instance, while a distributed cache is shared across all instances, making it suitable for load-balanced and distributed environments.

---

## Q3. What is `IDistributedCache`?

`IDistributedCache` is the ASP.NET Core abstraction for distributed caching. It provides methods to store, retrieve, refresh, and remove cached data, regardless of the underlying cache provider.

---

## Q4. Why does `IDistributedCache` store byte arrays?

Distributed caches are network-based systems. Objects must be serialized (typically to JSON or binary) before storage and deserialized after retrieval.

---

## Q5. What is the difference between Absolute and Sliding Expiration?

| Absolute Expiration | Sliding Expiration |
|---------------------|--------------------|
| Expires after a fixed duration | Expiration timer resets on every access |
| Good for data with a fixed lifetime | Good for frequently accessed data |

---

# Distributed Cache vs Memory Cache Summary

| Feature | Memory Cache | Distributed Cache |
|----------|--------------|-------------------|
| Storage Location | Application memory | Separate cache server |
| Shared Across Servers | ❌ No | ✅ Yes |
| Performance | Fastest | Very fast |
| Network Call | ❌ No | ✅ Yes |
| Suitable for Microservices | ❌ No | ✅ Yes |
| Survives Restart | ❌ No | ✅ Yes (provider dependent) |
| Typical Provider | IMemoryCache | Redis, Azure Cache for Redis, NCache |

---

# Key Takeaways

- A **Distributed Cache** is shared across multiple application instances, making it essential for scalable web applications and microservices.
- `IDistributedCache` provides a provider-independent API for distributed caching in ASP.NET Core.
- **Redis** is the most widely used distributed cache because of its speed, scalability, clustering, and persistence features.
- Objects stored in a distributed cache must be serialized and deserialized.
- Use **Absolute Expiration** for fixed lifetimes and **Sliding Expiration** for frequently accessed data.
- Proper cache invalidation and expiration strategies are critical to avoid serving stale data.