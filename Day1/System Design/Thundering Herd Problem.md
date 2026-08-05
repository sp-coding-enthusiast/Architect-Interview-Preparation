# Thundering Herd Problem in Distributed Systems

## Introduction

The **Thundering Herd Problem** occurs when **a large number of threads, processes, or application instances wake up simultaneously and compete for the same resource after an event occurs**.

The resource could be:

- Cache
- Database
- API
- Message Queue
- Distributed Lock
- File
- Network Connection

Instead of requests being spread over time, they arrive **all at once**, creating a massive spike in load that can overwhelm the system.

---

# Why is it Called "Thundering Herd"?

Imagine thousands of buffalo standing still.

Suddenly, a gate opens.

```text
Buffalo

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

One Gate

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

Massive Rush
```

The same happens in software.

Thousands of waiting clients suddenly rush toward the same resource.

---

# Real-World Example

Suppose your e-commerce website caches product information for 30 minutes.

```text
Redis

↓

Product Cache

↓

Expires
```

Immediately after expiration:

```text
50,000 Users

↓

Request Same Product

↓

Redis Miss

↓

50,000 Database Queries
```

Instead of one query, the database receives **50,000 simultaneous queries**.

---

# Architecture

```text
              Users

                 │

      50,000 Concurrent Requests

                 │

                 ▼

           ASP.NET Core API

                 │

                 ▼

               Redis

                 │

            Cache Miss

                 │

                 ▼

            SQL Database
```

Since the cache is empty, every request goes to the database.

---

# Common Causes

## 1. Cache Expiration

Most common cause.

```text
Cache Entry

↓

Expires

↓

Thousands of Requests

↓

Database
```

---

## 2. Service Recovery

Suppose an external API is down.

```text
API Down

↓

Clients Wait
```

When it recovers:

```text
API Up

↓

All Clients Retry

↓

API Overloaded Again
```

---

## 3. Distributed Lock Release

Many services wait for a lock.

```text
Redis Lock

↓

Released

↓

1000 Threads Compete
```

---

## 4. Message Queue

A queue suddenly receives many messages.

```text
Kafka

↓

Workers Wake Up

↓

Database Overloaded
```

---

## 5. Scheduled Jobs

Many jobs start at the same time.

```text
12:00 AM

↓

100 Jobs Start

↓

Database Spike
```

---

## 6. Retry Storm

Requests fail.

Every client retries immediately.

```text
Failure

↓

10000 Clients Retry

↓

More Failures
```

---

# Timeline

```text
Time

│

Clients Waiting

│

Resource Available

│

All Clients Wake Up

│

Massive Traffic Spike

│

System Slow
```

---

# Problems Caused

- Database overload
- High CPU usage
- Connection pool exhaustion
- Increased latency
- API timeouts
- Thread starvation
- Cascading failures
- Poor user experience

---

# Example Without Protection

```text
1000 Requests

↓

Cache Miss

↓

1000 Database Queries

↓

Database CPU = 100%

↓

Timeouts
```

---

# Solution 1: Distributed Lock (Recommended)

Allow only **one request** to rebuild the resource.

All others wait.

```text
Cache Miss

↓

Acquire Lock

│

├── Success

│      │

│      ▼

│  Database

│      │

│      ▼

│ Update Cache

│      │

│      ▼

│ Release Lock

│

└── Wait

       │

       ▼

Read Cache
```

Only **one database query** is executed.

---

# Pseudo-Code

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

---

# Solution 2: Request Coalescing (Single Flight)

Merge identical requests into one operation.

```text
1000 Requests

↓

One Database Query

↓

Shared Result

↓

1000 Responses
```

Benefits:

- Eliminates duplicate work
- Reduces database load
- Improves response time

---

# Solution 3: Exponential Backoff

Instead of retrying immediately:

```text
Retry

↓

1 Second

↓

2 Seconds

↓

4 Seconds

↓

8 Seconds
```

Requests become naturally spread out.

---

# Solution 4: Random Jitter

Without jitter:

```text
1000 Clients

↓

Retry After 5 Seconds
```

Traffic spike again.

With jitter:

```text
Client 1 → 4.8 sec

Client 2 → 5.4 sec

Client 3 → 6.1 sec

Client 4 → 5.7 sec
```

Retries are distributed over time.

---

# Solution 5: Circuit Breaker

If a downstream service keeps failing:

```text
Application

↓

Circuit Breaker

↓

Open

↓

Reject Requests

↓

Service Recovers
```

This prevents repeated overload.

---

# Solution 6: Rate Limiting

Limit incoming traffic.

```text
10,000 Requests

↓

Rate Limiter

↓

500 Allowed

↓

9,500 Delayed/Rejected
```

Useful for protecting APIs and databases.

---

# Solution 7: Queue-Based Processing

Instead of processing requests immediately:

```text
Application

↓

Message Queue

↓

Workers

↓

Database
```

Traffic is processed gradually.

Technologies:

- Kafka
- RabbitMQ
- Azure Service Bus
- Amazon SQS

---

# Solution 8: Refresh Ahead

Refresh frequently used cache entries before they expire.

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

Users never experience a herd.

---

# Solution 9: Cache Warming

Populate cache before traffic arrives.

```text
Application Starts

↓

Load Popular Products

↓

Redis

↓

Users Hit Cache
```

---

# Real-World Example

Flash sale starts at 10:00 AM.

Without protection:

```text
100,000 Users

↓

Cache Miss

↓

100,000 Database Queries

↓

Database Crash
```

With Redis Lock:

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

# Thundering Herd vs Cache Stampede

| Thundering Herd | Cache Stampede |
|-----------------|----------------|
| General concurrency problem | Specific caching problem |
| Can occur with APIs, databases, queues, locks, operating systems | Occurs when cached data expires |
| Many clients compete for the same resource | Many requests rebuild the same cache entry |
| Broader concept | Special case of Thundering Herd |

Relationship:

```text
Thundering Herd

│

├── Cache Stampede

├── Retry Storm

├── Lock Contention

├── Queue Surge

└── Service Recovery Surge
```

---

# Best Practices

- Use distributed locks for expensive operations.
- Implement request coalescing for identical requests.
- Use exponential backoff with random jitter for retries.
- Protect downstream services with circuit breakers.
- Apply rate limiting to prevent overload.
- Refresh popular cache entries before expiration.
- Warm caches before expected traffic spikes.
- Monitor retry rates, cache hit ratio, and database load.

---

# Common Mistakes

## Immediate Retries

```text
Failure

↓

Retry Immediately

↓

Failure

↓

Retry Again
```

Creates retry storms.

---

## No Locking

```text
500 Requests

↓

500 Database Queries
```

A distributed lock reduces this to one query.

---

## Fixed Retry Delay

```text
1000 Clients

↓

Retry After 5 Seconds
```

Always add random jitter.

---

## Expiring All Cache Entries Together

```text
10,000 Keys

↓

Expire At Same Time

↓

Database Spike
```

Randomize expiration times to spread load.

---

# Interview Questions

## Q1. What is the Thundering Herd Problem?

The Thundering Herd Problem occurs when many clients, threads, or services simultaneously compete for the same resource after it becomes available, creating a sudden surge in traffic that can overload the system.

---

## Q2. What causes the Thundering Herd Problem?

Common causes include:

- Cache expiration
- Service recovery
- Retry storms
- Distributed lock release
- Queue surges
- Scheduled jobs

---

## Q3. How do you prevent the Thundering Herd Problem?

Common techniques include:

- Distributed locks
- Request coalescing
- Exponential backoff
- Random jitter
- Circuit breakers
- Rate limiting
- Queue-based processing
- Refresh Ahead
- Cache warming

---

## Q4. Is Cache Stampede the same as the Thundering Herd Problem?

No.

A **Cache Stampede** is a **specific type** of Thundering Herd Problem caused by an expired cache entry. The Thundering Herd Problem is a broader concurrency issue that can occur with many shared resources.

---

## Q5. Why is random jitter important?

Without jitter, clients retry simultaneously after the same delay, causing another traffic spike. Jitter randomizes retry intervals, spreading requests over time and reducing contention.

---

# Key Takeaways

- The **Thundering Herd Problem** is a general concurrency issue where many clients simultaneously access the same resource.
- It commonly occurs during cache expiration, service recovery, retry storms, or lock release.
- The consequences include database overload, high latency, connection pool exhaustion, and cascading failures.
- **Cache Stampede** is one specific example of the Thundering Herd Problem.
- Effective mitigation techniques include **distributed locks**, **request coalescing**, **exponential backoff with jitter**, **circuit breakers**, **rate limiting**, **queue-based processing**, **Refresh Ahead**, and **cache warming**.
- Designing systems to distribute load over time is essential for building resilient, high-scale applications.