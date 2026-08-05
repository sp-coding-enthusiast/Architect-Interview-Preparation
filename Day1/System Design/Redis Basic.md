# Redis Basics

# Introduction

**Redis (Remote Dictionary Server)** is an **open-source, in-memory data store** used as a:

- Distributed Cache
- Database
- Message Broker
- Session Store
- Pub/Sub System
- Queue
- Distributed Lock Manager

Redis is extremely fast because it stores data primarily in **RAM (Memory)** instead of reading from disk for every operation.

---

# Why Redis?

Traditional databases read data from disk.

```text
Application

↓

SQL Server

↓

Disk
```

Disk access is relatively slow.

Redis stores data in memory.

```text
Application

↓

Redis

↓

RAM
```

Memory access is much faster than disk access.

---

# Speed Comparison

| Storage | Typical Latency |
|----------|-----------------|
| RAM (Redis) | ~0.1 - 1 ms |
| SSD Database | ~1 - 10 ms |
| HDD Database | ~10 - 100 ms |

Redis can handle **hundreds of thousands to millions of operations per second**, depending on hardware and workload.

---

# Redis Architecture

```text
            Client

               │

               ▼

      ASP.NET Core Application

               │

               ▼

              Redis

               │

               ▼

            SQL Server
```

Most reads are served by Redis.

Database is used only when required.

---

# Redis as a Cache

Without Redis

```text
Request

↓

Application

↓

Database

↓

Return Data
```

Every request queries the database.

---

With Redis

```text
Request

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

      Store In Redis

            │

            ▼

         Return
```

---

# Why Redis is Fast

Redis is fast because:

- Stores data in memory (RAM)
- Uses efficient data structures
- Single-threaded command execution (avoids locks)
- Minimal network protocol overhead
- Optimized C implementation

---

# Redis Data Structures

Redis is much more than a simple key-value store.

It supports multiple data structures.

| Data Structure | Description |
|---------------|-------------|
| String | Text, JSON, Numbers |
| Hash | Objects |
| List | Ordered collection |
| Set | Unique values |
| Sorted Set | Ordered by score |
| Bitmap | Bit operations |
| HyperLogLog | Approximate counting |
| Stream | Event streaming |
| Geospatial | Location data |

---

# 1. String

Most commonly used.

```text
Key

↓

product:1

↓

Laptop
```

Commands:

```redis
SET product:1 Laptop

GET product:1
```

Example:

```text
product:1

↓

Laptop
```

---

# 2. Hash

Stores objects.

Example:

```text
Product

↓

Id

↓

1

↓

Name

↓

Laptop

↓

Price

↓

70000
```

Commands:

```redis
HSET product:1 Name Laptop

HSET product:1 Price 70000

HGETALL product:1
```

Useful for:

- User profiles
- Product details
- Configuration

---

# 3. List

Maintains insertion order.

```text
Orders

↓

Order1

↓

Order2

↓

Order3
```

Commands:

```redis
LPUSH orders Order1

LPUSH orders Order2

LRANGE orders 0 -1
```

Common uses:

- Queues
- Task processing
- Logs

---

# 4. Set

Stores unique values.

```text
Tags

↓

Redis

Caching

Azure
```

Duplicate values are ignored.

Commands:

```redis
SADD tags Redis

SADD tags Azure

SMEMBERS tags
```

Use cases:

- User roles
- Tags
- Permissions

---

# 5. Sorted Set

Stores values with scores.

```text
Leaderboard

Player A

100

Player B

200

Player C

150
```

Commands:

```redis
ZADD leaderboard 100 Alice

ZADD leaderboard 200 Bob

ZRANGE leaderboard 0 -1 WITHSCORES
```

Use cases:

- Rankings
- Leaderboards
- Priority queues

---

# 6. Streams

Used for event processing.

```text
Order Created

↓

Redis Stream

↓

Consumer

↓

Order Processing
```

Similar to:

- Kafka
- Azure Event Hubs
- RabbitMQ (basic scenarios)

---

# Redis Persistence

Although Redis is in-memory, it can persist data to disk.

Two primary mechanisms are available.

---

## RDB (Redis Database Snapshot)

Creates snapshots at intervals.

```text
Memory

↓

Snapshot

↓

Disk
```

Example:

Every 5 minutes.

Advantages:

- Fast startup
- Compact files

Disadvantages:

- Recent changes since the last snapshot may be lost if Redis crashes.

---

## AOF (Append Only File)

Logs every write operation.

```text
SET A 10

SET B 20

DEL C

↓

Append File
```

Advantages:

- Better durability
- Less data loss

Disadvantages:

- Larger files
- Slightly slower writes

---

# Expiration

Redis supports automatic key expiration.

Command:

```redis
SET product Laptop EX 60
```

Meaning:

```
Expire after 60 seconds
```

Timeline:

```text
Create Key

↓

60 Seconds

↓

Automatically Deleted
```

---

# Common Redis Commands

## Store

```redis
SET user:1 John
```

---

## Read

```redis
GET user:1
```

---

## Delete

```redis
DEL user:1
```

---

## Check Existence

```redis
EXISTS user:1
```

---

## Set Expiration

```redis
EXPIRE user:1 60
```

---

## Increment

```redis
INCR counter
```

Useful for:

- Page views
- API rate limiting
- Counters

---

# Redis in ASP.NET Core

Install package:

```text
Microsoft.Extensions.Caching.StackExchangeRedis
```

Register:

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";

    options.InstanceName = "Shop";
});
```

Inject:

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

# Store Object

```csharp
var json =
    JsonSerializer.Serialize(product);

await _cache.SetStringAsync(
    "product:1",
    json);
```

---

# Read Object

```csharp
var json =
    await _cache.GetStringAsync("product:1");

if (json != null)
{
    var product =
        JsonSerializer.Deserialize<Product>(json);
}
```

---

# Redis Pub/Sub

Redis supports Publish/Subscribe messaging.

Publisher:

```text
Publish

↓

Redis

↓

Subscribers
```

Commands:

```redis
PUBLISH orders Created

SUBSCRIBE orders
```

Use cases:

- Notifications
- Chat
- Live updates

---

# Redis Transactions

Redis supports transactions.

Commands:

```redis
MULTI

SET A 1

SET B 2

EXEC
```

Either all commands execute together or none are executed if the transaction is discarded before `EXEC`.

> **Note:** Redis transactions do not provide the same isolation guarantees as relational database transactions.

---

# Redis Replication

Replication creates copies of data.

```text
Master

↓

Replica 1

↓

Replica 2
```

Benefits:

- High availability
- Read scaling
- Backup

---

# Redis Clustering

Large Redis deployments use clustering.

```text
Application

↓

Cluster

↓

Node 1

Node 2

Node 3
```

Benefits:

- Horizontal scaling
- Fault tolerance
- Automatic data partitioning (sharding)

---

# Common Use Cases

## Distributed Cache

```text
Product

↓

Redis

↓

Application
```

---

## Session Storage

```text
User Login

↓

Redis

↓

Shared Session
```

---

## Shopping Cart

```text
Cart

↓

Redis
```

---

## Leaderboards

```text
Game Scores

↓

Sorted Set
```

---

## Rate Limiting

```text
User

↓

Counter

↓

Requests Per Minute
```

---

## Distributed Lock

Prevent multiple servers from processing the same work simultaneously.

```text
Application 1

↓

Acquire Lock

↓

Redis

↓

Application 2 Waits
```

---

# Advantages

- Extremely fast
- In-memory performance
- Rich data structures
- Supports expiration
- High availability
- Clustering
- Replication
- Pub/Sub messaging
- Widely supported by cloud providers

---

# Disadvantages

- Memory is more expensive than disk
- Objects must often be serialized
- Limited by available RAM (though persistence and eviction policies help)
- Improper cache invalidation can lead to stale data
- Network latency exists because Redis is a separate service

---

# Best Practices

- Use meaningful cache keys.

Example:

```text
product:1

user:45

cart:102
```

---

- Set expiration for cache entries.

```csharp
AbsoluteExpirationRelativeToNow =
TimeSpan.FromMinutes(30);
```

---

- Avoid storing extremely large objects.

---

- Cache frequently accessed data.

---

- Monitor cache hit ratio.

---

- Remove or update cache entries when underlying data changes.

---

# Redis vs Memory Cache

| Feature | Memory Cache | Redis |
|----------|--------------|--------|
| Shared Across Servers | ❌ No | ✅ Yes |
| Distributed | ❌ No | ✅ Yes |
| Persistence | ❌ No | ✅ Yes (RDB/AOF) |
| Replication | ❌ No | ✅ Yes |
| Clustering | ❌ No | ✅ Yes |
| Network Access | ❌ No | ✅ Yes |
| Speed | Fastest | Very Fast |
| Suitable for Microservices | ❌ No | ✅ Yes |

---

# Redis vs SQL Database

| Redis | SQL Database |
|--------|--------------|
| In-memory | Disk-based |
| Key-value | Relational |
| Extremely fast | Slower than memory |
| Best for caching | Best for permanent storage |
| Temporary or frequently accessed data | Source of truth |

---

# Interview Questions

## Q1. What is Redis?

Redis is an open-source, in-memory data store commonly used as a distributed cache, database, message broker, and session store.

---

## Q2. Why is Redis faster than SQL Server?

Redis primarily stores data in RAM, which is significantly faster to access than disk-based storage used by traditional databases.

---

## Q3. What are the most commonly used Redis data structures?

- String
- Hash
- List
- Set
- Sorted Set
- Stream

---

## Q4. What is the difference between RDB and AOF?

| RDB | AOF |
|-----|-----|
| Periodic snapshots | Logs every write operation |
| Smaller files | Larger files |
| Faster recovery | Better durability |
| Possible data loss since last snapshot | Minimal data loss |

---

## Q5. Why is Redis commonly used with ASP.NET Core?

Redis provides a high-performance distributed cache that can be shared across multiple application instances, making it ideal for scalable web applications, APIs, and microservices.

---

# Key Takeaways

- Redis is an **in-memory** data store designed for speed.
- It is widely used as a **distributed cache**, session store, message broker, and lightweight database.
- Redis supports rich data structures such as **Strings, Hashes, Lists, Sets, Sorted Sets, and Streams**.
- **RDB** and **AOF** provide persistence options with different trade-offs between performance and durability.
- Redis supports **replication**, **clustering**, **Pub/Sub**, and **automatic key expiration**, making it suitable for highly available and scalable systems.
- In ASP.NET Core, Redis integrates seamlessly through the `IDistributedCache` abstraction and is the most common choice for production distributed caching.
