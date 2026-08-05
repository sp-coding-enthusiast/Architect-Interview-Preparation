# System Design: Caching Strategy for a Commodity Pricing API

# Problem Statement

Design a **highly scalable caching strategy** for a **Commodity Pricing API** that:

- Serves **millions of requests per day**
- Handles **live market prices**
- Supports **multiple commodity exchanges**
- Maintains **low latency (<50 ms)**
- Provides **high availability**
- Prevents stale pricing
- Supports horizontal scaling
- Handles traffic spikes during market hours

Examples of commodities:

- Crude Oil
- Natural Gas
- LNG
- Electricity
- Gold
- Silver
- Copper

---

# Functional Requirements

- Get latest commodity price
- Get historical prices
- Get settlement price
- Get market curves
- Get volatility data
- Get correlation data
- Get exchange rates
- Support multiple exchanges

---

# Non-Functional Requirements

- Millions of requests/day
- <50 ms response time
- High availability
- Horizontal scalability
- Eventual consistency (acceptable for most read scenarios)
- Fault tolerance
- Low database load
- Cache consistency

---

# High-Level Architecture

```text
                           Clients

                               │

                               ▼

                       API Gateway

                               │

                               ▼

                    ASP.NET Core Pricing API

                               │

         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼

   Local Memory Cache      Redis Cluster      Pricing Service

         │                     │                     │
         └──────────────┬──────┘                     │
                        ▼                            ▼

                  Kafka Event Bus             Pricing Database

                        ▲
                        │

                Market Data Feed
```

---

# Multi-Level Caching Strategy

Use **three cache levels**.

```text
L1 Cache

↓

Application Memory

↓

L2 Cache

↓

Redis Cluster

↓

L3

↓

SQL Database
```

---

# Level 1 Cache (In-Memory)

Technology

```
IMemoryCache
```

Stores:

- Hot prices
- Frequently requested symbols
- Exchange metadata

Example

```text
Brent

↓

₹72.10
```

TTL

```
5–15 Seconds
```

Advantages

- No network call
- Extremely fast
- Reduces Redis traffic

---

# Level 2 Cache (Redis)

Stores

- Commodity prices
- Curves
- Historical snapshots
- Market metadata
- Exchange rates

Example Key

```text
price:ICE:BRENT

price:NYMEX:NG

curve:BRENT

volatility:BRENT
```

TTL

```
30 Seconds
```

---

# Level 3

Persistent Database

```text
SQL Server

↓

Historical Prices

↓

Reference Data
```

Only used when cache misses occur.

---

# Request Flow

```text
Client

↓

Pricing API

↓

Memory Cache

↓

Hit?

│

├── Yes

│      │

│      ▼

│ Return

│

└── No

       │

       ▼

Redis

↓

Hit?

│

├── Yes

│      │

│      ▼

│ Update Memory Cache

│

└── No

       │

       ▼

Database

↓

Redis

↓

Memory

↓

Return
```

---

# Cache Pattern

Use **Cache Aside Pattern**

```text
Request

↓

Redis

↓

Miss

↓

Database

↓

Update Cache

↓

Return
```

Reason

- Simple
- High performance
- Most common
- Easy invalidation

---

# Cache Keys

Good keys are critical.

Example

```text
price:ICE:BRENT

price:NYMEX:NG

curve:BRENT:2027

historical:BRENT:2026-08-01

fx:USDINR

volatility:BRENT

correlation:BRENT:GAS
```

Never use

```text
Price1

Test

Commodity
```

---

# Serialization

Store compressed JSON.

```json
{
  "Commodity":"BRENT",
  "Exchange":"ICE",
  "Price":72.15,
  "Currency":"USD",
  "Timestamp":"2026-08-05T10:30:01Z"
}
```

---

# TTL Strategy

Different data has different freshness requirements.

| Data | TTL |
|------|------|
| Live Price | 5–10 sec |
| FX Rate | 30 sec |
| Settlement Price | 1 hour |
| Historical Price | 24 hours |
| Commodity Metadata | 24 hours |
| Volatility | 5 min |
| Correlation | 10 min |

---

# Cache Invalidation

Never wait only for TTL.

Whenever market data changes

```text
Market Feed

↓

Kafka

↓

Pricing Service

↓

Update Database

↓

Delete Redis Key

↓

Publish Event

↓

API Refreshes Memory Cache
```

---

# Event-Driven Cache Refresh

```text
Market Feed

↓

Kafka

↓

Pricing Engine

↓

Redis Updated

↓

API Nodes Receive Event

↓

Invalidate Local Cache
```

Advantages

- Fresh prices
- No stale data
- Fast synchronization

---

# Refresh Ahead

Frequently accessed commodities

```text
Brent

Natural Gas

Gold

Silver
```

are refreshed before expiration.

```text
Cache

↓

Almost Expired

↓

Background Refresh

↓

Fresh Cache
```

No cache miss occurs.

---

# Prevent Cache Stampede

Problem

```text
Redis Key Expired

↓

10000 Requests

↓

10000 Database Calls
```

Solution

Distributed Lock

```text
Cache Miss

↓

Redis Lock

↓

One Database Query

↓

Redis Updated

↓

Remaining Requests Read Cache
```

---

# Prevent Cache Avalanche

Randomize expiration.

Instead of

```
30 sec
```

Use

```
25–35 sec
```

---

# Prevent Cache Penetration

Cache null values.

```text
Invalid Commodity

↓

Database

↓

Not Found

↓

Cache NULL

↓

Future Requests

↓

No Database Call
```

---

# Distributed Redis Cluster

```text
                 Redis Cluster

      ┌──────────┬──────────┬──────────┐

      ▼          ▼          ▼

   Node 1     Node 2     Node 3

        Replicas for High Availability
```

Benefits

- Horizontal scaling
- High availability
- Failover
- Sharding

---

# Read/Write Flow

Read

```text
API

↓

Memory

↓

Redis

↓

Database
```

Write

```text
Market Feed

↓

Kafka

↓

Pricing Engine

↓

Database

↓

Redis

↓

Invalidate Memory Cache
```

---

# Background Cache Warming

Before market opens

```text
Scheduler

↓

Top 500 Commodities

↓

Load Redis
```

First user never experiences cache miss.

---

# Rate Limiting

Protect API.

```text
Client

↓

API Gateway

↓

100 Requests/Second

↓

Reject Excess
```

---

# Bulkhead

Separate thread pools

```text
Live Prices

↓

Dedicated Threads

Historical Prices

↓

Dedicated Threads
```

One service cannot block another.

---

# Circuit Breaker

If Redis fails

```text
Redis Down

↓

Circuit Open

↓

Read Memory Cache

↓

Database Fallback
```

---

# Hedging

For Redis cluster

```text
Request

↓

Primary Redis

↓

100 ms Delay

↓

Replica Redis

↓

Fastest Response Wins
```

Useful when latency spikes occur.

---

# Monitoring

Monitor

- Cache Hit Ratio
- Redis Latency
- Database Queries
- Cache Miss Rate
- Redis Memory Usage
- Key Evictions
- Stampede Count
- TTL Expirations
- Kafka Consumer Lag
- API Response Time

---

# Expected Cache Hit Ratio

| Layer | Expected Hit Rate |
|--------|-------------------|
| Memory Cache | 60–80% |
| Redis | 90–98% |
| Database | Less than 2% |

---

# Scalability

```text
          Load Balancer

      ┌─────────┬─────────┬─────────┐

      ▼         ▼         ▼

 API 1      API 2      API 3

      ▼         ▼         ▼

        Redis Cluster

              ▼

        SQL Cluster
```

Every API instance shares Redis.

---

# Failure Scenarios

## Redis Failure

```text
Redis Down

↓

Memory Cache

↓

Database

↓

Recover Redis

↓

Warm Cache
```

---

## Database Failure

```text
Database Down

↓

Redis Still Serves Cached Data

↓

Grace Period
```

This improves availability during short outages.

---

## Market Feed Delay

```text
No New Feed

↓

Serve Last Known Price

↓

Include Timestamp

↓

Mark Response As Potentially Stale
```

Clients can decide whether to use the data.

---

# Technology Stack

| Component | Technology |
|------------|------------|
| API | ASP.NET Core |
| L1 Cache | IMemoryCache |
| L2 Cache | Redis Cluster |
| Database | SQL Server / PostgreSQL |
| Messaging | Kafka |
| Background Jobs | Hosted Service / Quartz.NET |
| Monitoring | Prometheus + Grafana |
| Logging | Serilog |
| Resilience | Polly |

---

# Interview Questions

## Q1. Why use two cache levels?

A two-level cache combines the speed of in-memory caching with the consistency and scalability of Redis. Memory cache reduces Redis traffic, while Redis provides a shared cache across all application instances.

---

## Q2. Why use Kafka for cache invalidation?

Kafka distributes market update events to all API instances. Each instance invalidates or refreshes its local cache immediately, reducing stale data and avoiding cache inconsistency.

---

## Q3. How do you prevent stale commodity prices?

- Event-driven cache invalidation
- Short TTLs for live data
- Refresh Ahead for hot commodities
- Timestamp every response
- Background cache warming

---

## Q4. How do you prevent cache stampedes during market open?

Use Redis distributed locks so that only one request reloads missing data while other requests wait or read the refreshed cache once available.

---

## Q5. How would you scale this system to handle 100 million requests per day?

- Add more API instances behind a load balancer
- Scale Redis using clustering and replicas
- Partition data where appropriate
- Use Kafka for asynchronous market updates
- Maintain multi-level caching (L1 + L2)
- Apply rate limiting, bulkheads, circuit breakers, and monitoring to maintain resilience

---

# Key Takeaways

- Use a **multi-level caching architecture**: **IMemoryCache (L1)** → **Redis Cluster (L2)** → **Database (L3)**.
- Implement the **Cache Aside** pattern for reads and **event-driven cache invalidation** using Kafka for writes.
- Apply **short, data-specific TTLs**, **Refresh Ahead**, and **background cache warming** for frequently requested commodities.
- Protect the system from **cache stampedes**, **avalanches**, and **penetration** using distributed locks, TTL jitter, and negative caching.
- Combine caching with resilience patterns such as **Bulkhead**, **Circuit Breaker**, **Rate Limiting**, and **Hedging**.
- Continuously monitor cache hit ratios, latency, and invalidation events to ensure low latency and high availability under heavy load.