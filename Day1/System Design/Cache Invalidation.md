
# Cache Invalidation in .NET & Distributed Systems

# Introduction

Cache invalidation is the process of **removing or updating cached data when the original data changes**.

It ensures that users receive **fresh and consistent data** instead of outdated (stale) information.

One of the famous sayings in computer science is:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

A cache significantly improves performance, but if it is not invalidated correctly, users may receive incorrect data.

---

# Why Cache Invalidation is Needed

Suppose a product is cached.

```text
Database

Product Price = ₹1000

↓

Cache

Product Price = ₹1000
```

A user updates the price.

```text
Database

Product Price = ₹1200
```

But the cache still contains:

```text
Cache

Product Price = ₹1000
```

The next user receives incorrect data.

---

# Problem Without Cache Invalidation

```text
Client

↓

Application

↓

Cache

↓

Price = ₹1000 ❌

↓

Database

↓

Price = ₹1200
```

The application serves stale data.

---

# Cache Invalidation Workflow

```text
Update Request

↓

Database Updated

↓

Invalidate Cache

↓

Next Read

↓

Database

↓

Store Updated Data

↓

Return Fresh Data
```

---

# Types of Cache Invalidation

1. Time-Based Invalidation
2. Manual Invalidation
3. Write-Through
4. Write-Behind
5. Event-Based Invalidation
6. Version-Based Invalidation
7. Tag-Based Invalidation
8. Refresh Ahead

---

# 1. Time-Based Invalidation

The cache automatically expires after a fixed duration.

```text
Cache Created

↓

30 Minutes

↓

Expired

↓

Removed
```

Example:

```csharp
await cache.SetStringAsync(
    "product:1",
    json,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow =
            TimeSpan.FromMinutes(30)
    });
```

---

## Advantages

- Very simple
- No manual cleanup
- Prevents indefinitely stale data

---

## Disadvantages

- Data may remain stale until expiration
- Choosing the correct expiration time can be difficult

---

## Best Use Cases

- Product catalog
- Exchange rates
- News
- Weather

---

# 2. Manual Invalidation

Whenever data changes, remove the cache entry.

Example:

```text
Update Product

↓

Update Database

↓

Remove Cache

↓

Next Read

↓

Reload Database

↓

Cache Updated
```

Example code:

```csharp
await repository.UpdateAsync(product);

await cache.RemoveAsync($"product:{product.Id}");
```

Next request:

```text
Cache Miss

↓

Database

↓

New Cache Entry
```

---

## Advantages

- Always returns fresh data
- Easy to understand

---

## Disadvantages

- Developer must remember to invalidate the cache
- Missing an invalidation can lead to stale data

---

# 3. Write-Through Invalidation

Update both the cache and the database in the same operation.

```text
Application

↓

Update Cache

↓

Update Database

↓

Return
```

Example:

```text
Update Product

↓

Cache Updated

↓

Database Updated

↓

Success
```

---

## Advantages

- Cache always contains fresh data
- Strong consistency

---

## Disadvantages

- Slower writes
- Every write updates two systems

---

# 4. Write-Behind Invalidation

Update the cache immediately.

Persist changes to the database asynchronously.

```text
Application

↓

Update Cache

↓

Return

↓

Background Worker

↓

Database
```

---

## Advantages

- Extremely fast writes
- Lower database load

---

## Disadvantages

- Eventual consistency
- Risk of losing queued writes if not handled properly

---

# 5. Event-Based Invalidation

When data changes, publish an event.

All applications receiving the event invalidate their cache.

Architecture:

```text
Application A

↓

Update Database

↓

Publish Event

↓

Message Broker

↓

Application B

↓

Remove Cache
```

Example using Kafka or Azure Service Bus:

```text
Product Updated

↓

Kafka Topic

↓

Consumers

↓

Invalidate Cache
```

---

## Advantages

- Excellent for microservices
- Keeps caches synchronized
- Highly scalable

---

## Technologies

- Kafka
- RabbitMQ
- Azure Service Bus
- AWS SNS/SQS

---

# 6. Version-Based Invalidation

Instead of deleting cache entries, change the version number.

Old cache keys automatically become obsolete.

Old key:

```text
product:1:v1
```

New key:

```text
product:1:v2
```

Application requests:

```text
product:1:v2
```

The old key is never used again.

---

## Advantages

- No race conditions
- No cache deletion required
- Safe in distributed systems

---

## Disadvantages

- Old entries remain until they expire
- Increased cache memory usage

---

# 7. Tag-Based Invalidation

Group related cache entries using tags.

Example:

```text
Products

↓

Product 1

Product 2

Product 3
```

Tag:

```
Products
```

Invalidate tag:

```text
Products

↓

Remove All Product Cache Entries
```

Useful when updating multiple related items.

---

## Best Use Cases

- Product catalog
- Categories
- Search results
- CMS pages

---

# 8. Refresh Ahead

Instead of waiting for expiration, refresh cached data before it expires.

```text
Cache

↓

Almost Expired

↓

Background Refresh

↓

Cache Updated

↓

Users Continue Reading Fresh Data
```

No cache miss occurs.

---

# Cache Expiration Strategies

## Absolute Expiration

```text
Created

↓

30 Minutes

↓

Removed
```

Example:

```csharp
new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow =
        TimeSpan.FromMinutes(30)
}
```

---

## Sliding Expiration

```text
Created

↓

Access

↓

Expiration Reset

↓

Access

↓

Expiration Reset
```

Example:

```csharp
new DistributedCacheEntryOptions
{
    SlidingExpiration =
        TimeSpan.FromMinutes(20)
}
```

---

## Combined Strategy

```csharp
new DistributedCacheEntryOptions
{
    SlidingExpiration =
        TimeSpan.FromMinutes(10),

    AbsoluteExpirationRelativeToNow =
        TimeSpan.FromHours(2)
}
```

The cache expires after **2 hours**, even if it is accessed frequently.

---

# Cache Stampede

A cache stampede occurs when many requests simultaneously try to rebuild an expired cache entry.

Example:

```text
Cache Expires

↓

1000 Requests

↓

1000 Database Queries

↓

Database Overloaded
```

---

## Solution

Only one request reloads the cache.

Others wait.

```text
Cache Miss

↓

Acquire Lock

↓

Load Database

↓

Populate Cache

↓

Release Lock

↓

Remaining Requests Use Cache
```

---

# Cache Penetration

Repeated requests for data that doesn't exist.

Example:

```text
Product 999999

↓

Cache Miss

↓

Database

↓

Not Found

↓

Repeat Forever
```

---

## Solution

Cache negative results.

```text
Product Not Found

↓

Store NULL

↓

Future Requests

↓

Return NULL

↓

No Database Call
```

---

# Cache Avalanche

Many cache entries expire at exactly the same time.

```text
10,000 Keys

↓

Expire At 12:00

↓

Massive Database Traffic
```

---

## Solution

Add random expiration times.

Instead of:

```
30 Minutes
```

Use:

```
25–35 Minutes
```

This spreads cache refreshes over time.

---

# Cache Consistency Strategies

| Strategy | Consistency | Performance |
|----------|-------------|-------------|
| Cache Aside | Eventual | Excellent |
| Write Through | Strong | Good |
| Write Behind | Eventual | Excellent |
| Refresh Ahead | Very Good | Excellent |
| Version-Based | Strong | Very Good |

---

# Real-World Example

E-commerce application.

User updates a product.

```text
Admin

↓

Update Product

↓

Database Updated

↓

Redis Cache Removed

↓

Customer Request

↓

Cache Miss

↓

Database

↓

Fresh Cache

↓

Return Updated Product
```

---

# Best Practices

- Always invalidate cache after updates.
- Use expiration times to prevent stale data.
- Keep cache keys consistent and predictable.
- Avoid caching highly volatile data for long durations.
- Use distributed caches (e.g., Redis) for multi-instance applications.
- Protect against cache stampede using locks or request coalescing.
- Monitor cache hit ratio and eviction rates.
- Prefer event-driven invalidation in microservices.

---

# Common Mistakes

## Forgetting to Invalidate

```text
Update Database

↓

Cache Not Removed

↓

Users See Old Data
```

---

## Long Expiration

```text
Price Changed

↓

Cache Lives 24 Hours

↓

Incorrect Prices Displayed
```

---

## Caching Highly Dynamic Data

Examples:

- Stock prices
- Live sports scores
- Real-time trading data

Use short expirations or avoid caching entirely unless freshness requirements allow.

---

## Deleting Too Much

```text
Update Product

↓

Clear Entire Cache
```

This causes unnecessary cache misses and database load.

Instead, invalidate only the affected keys.

---

# Interview Questions

## Q1. What is cache invalidation?

Cache invalidation is the process of removing or updating cached data after the underlying data changes so users receive fresh and consistent information.

---

## Q2. Why is cache invalidation difficult?

Because there must be a balance between:

- Data freshness
- Performance
- Consistency
- Scalability

Incorrect invalidation can lead to stale data or unnecessary database load.

---

## Q3. What is the most common invalidation strategy?

**Cache Aside** with manual invalidation after updates and an expiration policy is the most commonly used strategy in web applications.

---

## Q4. What is event-based invalidation?

When data changes, an event is published through a message broker (such as Kafka or Azure Service Bus). All interested applications receive the event and invalidate their local or distributed cache.

---

## Q5. What are Cache Stampede, Cache Penetration, and Cache Avalanche?

| Problem | Description | Solution |
|----------|-------------|----------|
| Cache Stampede | Many requests rebuild the same expired cache | Locking, request coalescing, single-flight |
| Cache Penetration | Repeated requests for non-existent data | Cache null values, Bloom filters |
| Cache Avalanche | Many keys expire simultaneously | Randomize expiration times |

---

# Key Takeaways

- Cache invalidation keeps cached data synchronized with the source of truth.
- **Time-Based** invalidation is simple but may temporarily serve stale data.
- **Manual** invalidation provides fresh data but requires disciplined implementation.
- **Write-Through** offers strong consistency by updating cache and database together.
- **Write-Behind** improves write performance at the cost of eventual consistency.
- **Event-Based** invalidation is ideal for distributed systems and microservices.
- **Version-Based** invalidation avoids race conditions by changing cache keys instead of deleting entries.
- Protect your cache against **stampedes**, **penetration**, and **avalanches** using appropriate strategies.
- Cache invalidation is as important as caching itself—poor invalidation can negate the benefits of caching.
