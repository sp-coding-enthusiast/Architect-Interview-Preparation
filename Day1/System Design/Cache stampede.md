# Cache Stampede in .NET & Distributed Systems

# Introduction

A **Cache Stampede** (also called the **Thundering Herd Problem**) occurs when a cached item expires and **many requests simultaneously attempt to rebuild the same cache entry**.

Instead of one request loading the data from the database, hundreds or thousands of requests hit the database at the same time, causing:

- Database overload
- Increased latency
- High CPU usage
- Request failures
- Cascading failures across the system

Cache stampedes are one of the most common challenges in high-traffic applications.

---

# Why Cache Stampede Happens

Suppose a product is cached for 30 minutes.

```text
Cache

Product:1

↓

Expires
```

Immediately after expiration, thousands of users request the same product.

Since the cache is empty, every request goes to the database.

---

# Cache Stampede Scenario

```text
                1000 Users

                     │

                     ▼

               Application

                     │

               Cache Miss

                     │

        ┌────────────┼────────────┐
        ▼            ▼            ▼

     Request 1   Request 2   Request 3
        │            │            │
        ▼            ▼            ▼
       Database    Database    Database
        │            │            │
        └────────────┼────────────┘
                     ▼

              Database Overloaded
```

Instead of **1 database query**, the system performs **1000 database queries**.

---

# Without Cache Stampede

Normal operation:

```text
Client

↓

Application

↓

Cache

↓

Hit

↓

Return Data
```

Database is rarely accessed.

---

# During Cache Stampede

```text
Client

↓

Application

↓

Cache

↓

Miss

↓

Thousands of Database Queries

↓

Slow Response

↓

Database Crash
```

---

# Real-World Example

Imagine an e-commerce application.

```text
Product

↓

Redis

↓

Expires
```

At 9:00 AM a flash sale begins.

```text
50,000 Users

↓

Same Product

↓

Cache Expired

↓

50,000 Database Queries
```

The database becomes overwhelmed.

---

# Why It Is Dangerous

Problems caused by a cache stampede:

- Database overload
- Slow API responses
- Connection pool exhaustion
- High CPU utilization
- Thread starvation
- Cascading failures in downstream services
- Poor user experience

---

# Solution 1: Distributed Lock (Most Common)

Allow **only one request** to rebuild the cache.

Other requests wait.

---

## Flow

```text
Request

↓

Cache Miss

↓

Acquire Lock

│

├── Lock Acquired

│       │

│       ▼

│   Query Database

│       │

│       ▼

│   Update Cache

│       │

│       ▼

│ Release Lock

│

└── Lock Not Acquired

        │

        ▼

Wait

↓

Read Cache Again
```

Only one request reaches the database.

---

## Example

```text
1000 Requests

↓

Cache Miss

↓

One Request Gets Lock

↓

Database

↓

Redis Updated

↓

999 Requests Read Redis
```

---

# Example Using Redis Lock

Pseudo-code:

```csharp
var data = await cache.GetAsync(key);

if (data == null)
{
    if (await AcquireLockAsync(key))
    {
        data = await repository.GetAsync(id);

        await cache.SetAsync(key, data);

        await ReleaseLockAsync(key);
    }
    else
    {
        await Task.Delay(100);

        data = await cache.GetAsync(key);
    }
}

return data;
```

A distributed lock ensures only one application instance rebuilds the cache.

---

# Solution 2: Request Coalescing (Single Flight)

Instead of every request rebuilding the cache, combine concurrent requests into a single operation.

---

## Flow

```text
100 Requests

↓

Same Cache Key

↓

Single Database Query

↓

Shared Result

↓

100 Responses
```

Only one database query is executed.

---

## Benefits

- Eliminates duplicate work
- Reduces database load
- Improves response times

---

# Solution 3: Refresh Ahead

Refresh popular cache entries **before** they expire.

---

## Without Refresh Ahead

```text
Cache

↓

Expires

↓

Next Request

↓

Database
```

---

## With Refresh Ahead

```text
Cache

↓

Almost Expired

↓

Background Refresh

↓

Cache Updated

↓

No Cache Miss
```

Users never experience a cache miss.

---

# Solution 4: Random Expiration (Jitter)

Avoid having many cache keys expire at the same time.

Instead of:

```text
All Keys

↓

Expire In 30 Minutes
```

Use:

```text
Key 1 → 27 Minutes

Key 2 → 31 Minutes

Key 3 → 34 Minutes

Key 4 → 29 Minutes
```

This spreads database traffic over time.

---

## Example

```csharp
var random = Random.Shared.Next(25, 35);

await cache.SetStringAsync(
    key,
    value,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow =
            TimeSpan.FromMinutes(random)
    });
```

---

# Solution 5: Never Expire Hot Data

Instead of expiring highly requested data, update it asynchronously.

```text
Hot Product

↓

Never Expires

↓

Background Job

↓

Refresh Every 5 Minutes
```

Useful for:

- Product catalog
- Exchange rates
- Home page
- Trending content

---

# Solution 6: Cache Null Values

Sometimes requests repeatedly ask for data that doesn't exist.

Example:

```text
Product 999999

↓

Cache Miss

↓

Database

↓

Not Found
```

Without caching the negative result:

```text
1000 Requests

↓

1000 Database Queries
```

Instead:

```text
Not Found

↓

Cache NULL

↓

Future Requests

↓

Return NULL
```

This prevents unnecessary database access.

---

# Solution 7: Background Cache Warming

Populate the cache before users request data.

Example:

```text
Application Starts

↓

Load Popular Products

↓

Store In Redis

↓

Users Hit Cache
```

No initial cache misses.

---

# Solution 8: Use a Distributed Cache

In-memory caches are local to each application instance.

```text
App 1

↓

Memory Cache

App 2

↓

Memory Cache
```

Both applications may rebuild the same cache independently.

With Redis:

```text
App 1

↓

Redis

App 2

↓

Redis
```

All instances share the same cache.

---

# Timeline of a Cache Stampede

Without protection:

```text
Time

│

Cache Expires

│

1000 Requests

│

1000 Database Queries

│

Cache Updated
```

With distributed locking:

```text
Time

│

Cache Expires

│

1000 Requests

│

One Lock

│

One Database Query

│

Cache Updated

│

999 Requests Read Cache
```

---

# Cache Stampede vs Cache Avalanche vs Cache Penetration

| Problem | Description | Solution |
|----------|-------------|----------|
| Cache Stampede | Many requests rebuild the same expired cache entry | Distributed lock, request coalescing, refresh ahead |
| Cache Avalanche | Many different cache keys expire simultaneously | Randomized expiration (jitter), staggered TTLs |
| Cache Penetration | Repeated requests for non-existent data | Cache null values, Bloom filters |

---

# Real-World Example

E-commerce flash sale.

Without protection:

```text
100,000 Users

↓

Cache Expires

↓

100,000 Database Queries

↓

Database Crash
```

With Redis locking:

```text
100,000 Users

↓

One Lock

↓

One Database Query

↓

Redis Updated

↓

100,000 Cache Reads
```

---

# Best Practices

- Use distributed locks for rebuilding expensive cache entries.
- Add random expiration times (TTL jitter).
- Refresh popular cache entries before they expire.
- Cache negative results when appropriate.
- Warm the cache during application startup or before peak traffic.
- Use a distributed cache (such as Redis) in multi-instance deployments.
- Monitor cache hit ratio, misses, and rebuild frequency.

---

# Common Mistakes

## Letting Every Request Hit the Database

```text
Cache Miss

↓

Every Request

↓

Database
```

This defeats the purpose of caching.

---

## Expiring All Keys Together

```text
10,000 Keys

↓

Expire At 12:00

↓

Database Overload
```

Use randomized expiration.

---

## Ignoring Hot Keys

Frequently requested data should be refreshed proactively rather than waiting for expiration.

---

## No Locking Mechanism

Without locking:

```text
500 Requests

↓

500 Database Queries
```

With locking:

```text
500 Requests

↓

1 Database Query
```

---

# Interview Questions

## Q1. What is a Cache Stampede?

A cache stampede occurs when a cache entry expires and many concurrent requests simultaneously attempt to rebuild it, causing excessive load on the underlying data source.

---

## Q2. Why is a Cache Stampede dangerous?

It can overwhelm the database, increase latency, exhaust connection pools, and even cause cascading failures across the system.

---

## Q3. How do you prevent a Cache Stampede?

Common techniques include:

- Distributed locking
- Request coalescing (single-flight)
- Refresh Ahead
- Background cache warming
- Randomized expiration (TTL jitter)

---

## Q4. Why is Redis commonly used to prevent Cache Stampedes?

Redis supports distributed locking (for example, using `SET key value NX EX` or libraries implementing the Redlock algorithm), allowing only one application instance to rebuild a cache entry while others wait or retry.

---

## Q5. What is the difference between a Cache Stampede and a Cache Avalanche?

| Cache Stampede | Cache Avalanche |
|----------------|-----------------|
| One hot key expires | Many keys expire simultaneously |
| Many requests rebuild the same data | Many cache misses occur across different keys |
| Solved with locking and request coalescing | Solved with randomized expiration and staggered TTLs |

---

# Key Takeaways

- A **Cache Stampede** occurs when many requests try to rebuild the same expired cache entry simultaneously.
- The main risks are database overload, high latency, and cascading failures.
- **Distributed locking** is the most common and effective solution.
- **Request coalescing** ensures concurrent requests share a single data retrieval operation.
- **Refresh Ahead**, **background cache warming**, and **TTL jitter** further reduce the likelihood of stampedes.
- Combining Redis, distributed locks, and intelligent expiration strategies provides a robust caching solution for high-traffic applications.
