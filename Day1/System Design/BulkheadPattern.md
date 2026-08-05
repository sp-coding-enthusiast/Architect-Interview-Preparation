# Bulkhead Pattern in Distributed Systems

# Introduction

The **Bulkhead Pattern** is a resilience pattern that **isolates different parts of an application so that a failure in one component does not affect the others**.

The name comes from ships.

A ship is divided into multiple **watertight compartments (bulkheads)**.

If one compartment is flooded, the other compartments remain unaffected, preventing the ship from sinking.

Software systems use the same idea by **isolating resources such as threads, connections, or service calls**.

---

# Why Bulkhead is Needed

Suppose an application calls three external services.

- Payment Service
- Product Service
- Notification Service

Without isolation:

```text
Application

↓

Shared Thread Pool

↓

Payment Service (Slow)

↓

Threads Blocked

↓

Product Service

↓

Notification Service

↓

Also Blocked
```

A slow Payment Service can make the entire application slow.

---

# Ship Analogy

Without bulkheads:

```text
Ship

──────────────

Flood

↓

Entire Ship Sinks
```

With bulkheads:

```text
Ship

┌────┬────┬────┐

│ A  │ B  │ C  │

└────┴────┴────┘

Flood in B

↓

A Safe

B Flooded

C Safe
```

Only one section is affected.

---

# Bulkhead in Software

Without Bulkhead

```text
                 Users

                    │

                    ▼

             ASP.NET Core API

                    │

           Shared Thread Pool

     ┌──────────┼───────────┐

     ▼          ▼           ▼

 Payment     Product    Notification
```

If Payment becomes slow:

```text
Shared Thread Pool

↓

All Threads Busy

↓

Everything Slows Down
```

---

# With Bulkhead

```text
                 Users

                    │

                    ▼

             ASP.NET Core API

      ┌─────────┬─────────┬─────────┐

      ▼         ▼         ▼

 Payment     Product   Notification

Thread Pool Thread Pool Thread Pool

    20          20          20
```

Each service has its own resources.

If Payment fails:

```text
Payment

↓

Threads Full

↓

Product Works

↓

Notification Works
```

---

# Real-World Example

An e-commerce application:

```text
Checkout

↓

Payment API

↓

Email API

↓

Inventory API
```

Payment becomes slow.

Without bulkhead:

```text
Payment

↓

Threads Exhausted

↓

Inventory Stops

↓

Email Stops
```

With bulkhead:

```text
Payment

↓

Payment Threads Full

↓

Inventory Still Running

↓

Email Still Running
```

Only payment requests are affected.

---

# How Bulkhead Works

```text
Incoming Requests

↓

Bulkhead

↓

Dedicated Resource Pool

↓

Service
```

Each service gets:

- Dedicated threads
- Dedicated connections
- Dedicated queues
- Dedicated resources

---

# Types of Bulkhead Isolation

## 1. Thread Pool Isolation

Each service has its own thread pool.

```text
Application

↓

Payment Pool

↓

20 Threads

Application

↓

Inventory Pool

↓

20 Threads
```

If one pool is exhausted, the others continue processing.

---

## 2. Connection Pool Isolation

Each downstream service uses its own connection pool.

```text
Payment Database

↓

Connection Pool

Inventory Database

↓

Connection Pool
```

One exhausted pool does not affect another.

---

## 3. Queue Isolation

Separate queues for different workloads.

```text
Orders Queue

Payments Queue

Emails Queue
```

A backlog in one queue doesn't block others.

---

## 4. Resource Isolation

Dedicated CPU, memory, or containers.

```text
Kubernetes

↓

Payment Pod

Inventory Pod

Notification Pod
```

Each workload has its own resource limits.

---

# Without Bulkhead Example

```text
100 Threads

↓

Payment Requests

↓

All Threads Busy

↓

Product Requests Wait

↓

Timeouts
```

---

# With Bulkhead Example

```text
Payment

↓

20 Threads

↓

Full

↓

Reject New Payment Requests

↓

Inventory Uses Its Own Threads

↓

Still Works
```

---

# Bulkhead with Polly (.NET)

Polly v8 provides a **Concurrency Limiter**, which is the recommended approach for bulkhead-style isolation.

```csharp
builder.Services.AddHttpClient("PaymentApi")
    .AddResilienceHandler("payment", builder =>
    {
        builder.AddConcurrencyLimiter(permitLimit: 20, queueLimit: 50);
    });
```

Meaning:

- Maximum **20 concurrent requests**
- Up to **50 additional requests** can wait in the queue
- Beyond that, requests are rejected immediately

---

# Request Flow

```text
Incoming Requests

↓

Concurrency Limit

↓

Available Permit?

│

├── Yes

│      │

│      ▼

│  Call Service

│

└── No

       │

       ▼

Queue

↓

Still Full?

↓

Reject Request
```

---

# Bulkhead vs Circuit Breaker

## Bulkhead

```text
Service Slow

↓

Limit Concurrent Requests

↓

Other Services Continue
```

---

## Circuit Breaker

```text
Service Failing

↓

Stop Calling Service

↓

Fail Fast
```

---

# Comparison

| Bulkhead | Circuit Breaker |
|----------|-----------------|
| Isolates resources | Stops calling failing service |
| Prevents resource exhaustion | Prevents repeated failures |
| Protects healthy services | Protects unhealthy services |
| Works during high load | Works during repeated failures |

---

# Bulkhead vs Rate Limiter

| Bulkhead | Rate Limiter |
|----------|--------------|
| Limits concurrent work | Limits request rate over time |
| Protects internal resources | Protects against excessive traffic |
| Focuses on isolation | Focuses on traffic control |

---

# Bulkhead vs Timeout

| Bulkhead | Timeout |
|----------|----------|
| Limits concurrent operations | Limits execution time |
| Prevents thread exhaustion | Prevents long-running requests |
| Controls capacity | Controls duration |

---

# Combining Resilience Patterns

A robust API often combines multiple patterns.

```text
Client

↓

Rate Limiter

↓

Timeout

↓

Bulkhead

↓

Circuit Breaker

↓

Retry

↓

External Service
```

Each pattern solves a different problem.

---

# Advantages

- Prevents cascading failures
- Isolates slow or failing services
- Protects critical resources
- Improves system stability
- Increases availability
- Reduces thread starvation
- Improves fault isolation

---

# Disadvantages

- More configuration
- Additional resource management
- Possible request rejection under heavy load
- Requires careful tuning of limits

---

# Real-World Use Cases

- Payment gateways
- Inventory systems
- Banking applications
- Airline reservation systems
- Cloud microservices
- AI inference services
- External REST APIs
- Database access layers

---

# Best Practices

- Isolate each critical downstream dependency.
- Configure realistic concurrency limits based on testing.
- Combine bulkheads with timeouts and circuit breakers.
- Monitor queue lengths and rejected requests.
- Reject requests quickly rather than allowing thread pools to become exhausted.
- Apply separate bulkheads for services with different performance characteristics.

---

# Common Mistakes

## Shared Thread Pool

```text
Payment Slow

↓

All Threads Busy

↓

Entire API Slow
```

---

## Unlimited Concurrency

```text
10,000 Requests

↓

10,000 Database Calls

↓

Database Crash
```

Limit concurrent work.

---

## Large Queues

```text
Queue Size = 10,000

↓

Long Wait Times

↓

Poor User Experience
```

Keep queues small and reject excess requests when appropriate.

---

## No Monitoring

Without observing:

- Rejected requests
- Queue length
- Active concurrency

it's difficult to tune the system effectively.

---

# Interview Questions

## Q1. What is the Bulkhead Pattern?

The Bulkhead Pattern isolates resources such as threads, connections, or queues so that a failure or overload in one component does not affect the rest of the application.

---

## Q2. Why is it called the Bulkhead Pattern?

It is inspired by ship bulkheads, which divide a ship into watertight compartments. If one compartment floods, the others remain intact, preventing the entire ship from sinking.

---

## Q3. How does the Bulkhead Pattern improve resilience?

By limiting the resources available to each dependency, it prevents one slow or failing service from exhausting shared resources and causing cascading failures.

---

## Q4. What is the difference between Bulkhead and Circuit Breaker?

| Bulkhead | Circuit Breaker |
|----------|-----------------|
| Isolates resources | Stops requests to an unhealthy service |
| Protects healthy services | Protects the failing dependency |
| Prevents resource exhaustion | Prevents repeated failures |

---

## Q5. How is Bulkhead implemented in modern .NET?

In modern .NET (using Polly v8), bulkhead-style isolation is typically implemented with the **Concurrency Limiter**, which restricts the number of concurrent operations and optionally queues a limited number of additional requests.

---

# Key Takeaways

- The **Bulkhead Pattern** isolates resources so failures in one area do not spread to others.
- It is especially useful in **microservices**, **cloud applications**, and systems that depend on multiple external services.
- Common forms of isolation include **thread pools**, **connection pools**, **queues**, and **resource limits**.
- In .NET, Polly's **Concurrency Limiter** provides an effective way to implement bulkhead isolation.
- Bulkheads work best when combined with **Timeouts**, **Circuit Breakers**, **Retries**, and **Rate Limiting** to create a comprehensive resilience strategy.