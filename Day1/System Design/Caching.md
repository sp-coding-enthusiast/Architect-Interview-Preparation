# Caching Patterns in .NET & Distributed Systems

# Introduction

Caching is one of the most effective techniques for improving application performance. Instead of repeatedly fetching data from a slow data source such as a database, external API, or file system, frequently accessed data is temporarily stored in a **fast storage layer (cache)**.

Typical cache technologies include:

- In-Memory Cache
- Redis
- Memcached
- NCache
- Azure Cache for Redis

---

# Why Caching is Needed

Without caching, every request hits the database.

```text
Client

↓

Application

↓

Database
```

Problems:

- High database load
- Slow response time
- Higher infrastructure cost
- Poor scalability
- Increased latency

---

## With Caching

```text
                Client

                   │

                   ▼

              Application

                   │

           Cache Available?

             │          │

           Yes          No

            │            │

            ▼            ▼

      Return Data     Database

                           │

                           ▼

                     Store In Cache

                           │

                           ▼

                      Return Data
```

Most requests are served directly from cache.

---

# Benefits of Caching

## Faster Response Time

Database query:

```
120 ms
```

Cache lookup:

```
2 ms
```

---

## Reduced Database Load

Without cache:

```
1000 Requests

↓

1000 Database Queries
```

With cache:

```
1000 Requests

↓

950 Cache Hits

↓

50 Database Queries
```

---

## Better Scalability

Instead of scaling the database:

```text
More Users

↓

Cache Handles Requests

↓

Database Load Remains Low
```

---

## Lower Infrastructure Cost

Fewer database reads mean:

- Lower CPU usage
- Lower memory usage
- Fewer database servers
- Reduced cloud costs

---

# Cache Hit vs Cache Miss

## Cache Hit

```text
Request

↓

Cache

↓

Data Found

↓

Return Data
```

Fastest scenario.

---

## Cache Miss

```text
Request

↓

Cache

↓

Not Found

↓

Database

↓

Store In Cache

↓

Return Data
```

The first request is slower; subsequent requests are faster.

---

# Cache Aside Pattern (Lazy Loading)

## Definition

The application is responsible for reading from the cache first.

If the data is not found:

1. Read from the database
2. Store in cache
3. Return data

The cache is populated **on demand**.

---

## Flow

```text
Application

↓

Check Cache

↓

Hit?

│          │

Yes        No

│           │

▼           ▼

Return     Database

             │

             ▼

      Store In Cache

             │

             ▼

        Return Data
```

---

## Example

```csharp
public async Task<Product> GetProduct(int id)
{
    var key = $"product:{id}";

    var product = await cache.GetAsync<Product>(key);

    if (product != null)
        return product;

    product = await repository.GetAsync(id);

    await cache.SetAsync(key, product);

    return product;
}
```

---

## Advantages

- Simple to implement
- Only caches frequently used data
- Efficient memory usage
- Most commonly used caching strategy

---

## Disadvantages

- First request is slow
- Cache miss causes database access
- Possible cache stampede under high load

---

## Best Use Cases

- Product catalog
- User profiles
- Search results
- Configuration
- Frequently read data

---

# Read Through Cache

## Definition

The application never talks directly to the database.

Instead:

- Reads always go to the cache.
- The cache itself loads missing data from the database.

---

## Flow

```text
Application

↓

Cache

↓

Hit?

│          │

Yes        No

│           │

▼           ▼

Return     Database

             │

             ▼

      Store In Cache

             │

             ▼

         Return Data
```

The difference is **who loads the data**.

---

## Cache Aside vs Read Through

Cache Aside:

```text
Application

↓

Cache

↓

Database
```

Application manages cache.

---

Read Through:

```text
Application

↓

Cache

↓

Database
```

Cache manages data loading.

---

## Advantages

- Cleaner application code
- Centralized caching logic
- Easier maintenance

---

## Disadvantages

- Requires cache provider support
- More complex infrastructure

---

## Common Technologies

- Redis Modules
- Hazelcast
- Apache Ignite
- NCache

---

# Write Through Cache

## Definition

Every write goes to the cache first.

The cache immediately writes the same data to the database.

---

## Flow

```text
Application

↓

Cache

↓

Database

↓

Success

↓

Return
```

---

## Example

```text
Update Product

↓

Cache Updated

↓

Database Updated

↓

Return Success
```

Cache and database stay synchronized.

---

## Advantages

- Cache always contains latest data
- Strong consistency
- Reads are always fast

---

## Disadvantages

- Write latency increases
- Every write touches both cache and database
- Higher write cost

---

## Best Use Cases

- Banking
- Inventory
- Financial systems
- User settings

---

# Write Behind (Write Back)

## Definition

Writes go to the cache immediately.

The database is updated **later**, asynchronously.

---

## Flow

```text
Application

↓

Cache

↓

Return Immediately

↓

Background Worker

↓

Database
```

---

## Example

```text
Update Product

↓

Update Cache

↓

Return Success

↓

Queue Write

↓

Database Updated Later
```

---

## Advantages

- Extremely fast writes
- Reduced database load
- Batch database updates
- Better throughput

---

## Disadvantages

- Risk of data loss before persistence
- Eventual consistency
- More complex implementation

---

## Best Use Cases

- Analytics
- Logging
- IoT
- Telemetry
- High-volume event processing

---

# Write Through vs Write Behind

## Write Through

```text
Application

↓

Cache

↓

Database

↓

Return
```

Database updated immediately.

---

## Write Behind

```text
Application

↓

Cache

↓

Return

↓

Background Worker

↓

Database
```

Database updated later.

---

# Refresh Ahead Pattern

## Definition

Refresh Ahead proactively updates cached data **before it expires**.

Instead of waiting for a cache miss, the cache refreshes popular entries in the background.

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

↓

Slow Response
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

User Gets Fresh Data
```

No cache miss occurs.

---

## Timeline

```text
Time

│

Cache Created

│

User Requests

│

Refresh Triggered

│

Cache Updated

│

Expiration Extended
```

---

## Advantages

- No cache misses for hot data
- Consistently low latency
- Better user experience

---

## Disadvantages

- Refreshes data that may never be requested again
- Additional background processing
- More complex cache management

---

## Best Use Cases

- Home page
- Trending products
- Weather data
- Exchange rates
- Dashboards

---

# Comparison

| Pattern | Reads | Writes | Consistency | Performance |
|----------|--------|---------|-------------|-------------|
| Cache Aside | Application loads cache | Database | Eventual | Excellent for reads |
| Read Through | Cache loads data | Database | Eventual | Excellent |
| Write Through | Cache | Cache → Database | Strong | Slower writes |
| Write Behind | Cache | Cache → Queue → Database | Eventual | Fastest writes |
| Refresh Ahead | Cache proactively refreshes | Database | Fresh cache | Excellent |

---

# Real-World Examples

| Scenario | Recommended Pattern |
|----------|---------------------|
| Product Catalog | Cache Aside |
| User Profile | Cache Aside |
| News Feed | Refresh Ahead |
| Banking | Write Through |
| Inventory | Write Through |
| Shopping Cart | Cache Aside |
| Analytics | Write Behind |
| IoT Devices | Write Behind |
| Dashboard | Refresh Ahead |
| Weather API | Refresh Ahead |

---

# Decision Flow

```text
Is data read frequently?

        │

       Yes

        │

        ▼

Use Cache

        │

Need lazy loading?

        │

      Yes

        ▼

Cache Aside

        │

Need cache to load data automatically?

        │

      Yes

        ▼

Read Through

────────────────────────────

Is write consistency critical?

        │

      Yes

        ▼

Write Through

        │

Need highest write performance?

        │

      Yes

        ▼

Write Behind

────────────────────────────

Need to avoid cache expiry delays?

        │

      Yes

        ▼

Refresh Ahead
```

---

# Interview Questions

## Q1. Why is caching needed?

Caching reduces database load, improves response time, increases scalability, and lowers infrastructure costs by serving frequently accessed data from fast storage instead of repeatedly querying the primary data source.

---

## Q2. What is Cache Aside?

The application checks the cache first. On a cache miss, it loads data from the database, stores it in the cache, and returns it to the caller.

---

## Q3. What is the difference between Cache Aside and Read Through?

| Cache Aside | Read Through |
|--------------|--------------|
| Application loads missing data | Cache loads missing data |
| Simpler to implement | Requires cache provider support |
| Most common pattern | Centralized cache logic |

---

## Q4. What is the difference between Write Through and Write Behind?

| Write Through | Write Behind |
|----------------|--------------|
| Database updated immediately | Database updated asynchronously |
| Strong consistency | Eventual consistency |
| Slower writes | Faster writes |

---

## Q5. What is Refresh Ahead?

Refresh Ahead updates frequently accessed cache entries **before** they expire, ensuring users continue receiving fresh data without experiencing cache misses.

---

# Key Takeaways

- Caching improves performance by reducing expensive database or API calls.
- **Cache Aside** is the most widely used caching pattern and is simple to implement.
- **Read Through** delegates cache population to the cache layer.
- **Write Through** prioritizes strong consistency by updating both cache and database synchronously.
- **Write Behind** prioritizes write performance by updating the database asynchronously.
- **Refresh Ahead** proactively refreshes popular cache entries to avoid cache misses and maintain low latency.
- Choosing the right caching pattern depends on the application's consistency, latency, and scalability requirements.