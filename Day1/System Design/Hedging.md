# Hedging Pattern in Distributed Systems

# Introduction

The **Hedging Pattern** is a resilience technique where an application **sends multiple requests for the same operation to different replicas (or after a small delay)** and uses **the first successful response**, cancelling the remaining requests.

The goal is to reduce **tail latency** (slowest requests) and improve the overall response time.

Unlike a normal retry, **hedging does not wait for the first request to fail**. It proactively sends another request if the original request is taking too long.

---

# Why Hedging is Needed

Imagine an API where:

- 95% of requests complete in **100 ms**
- 5% of requests take **5 seconds**

Even though most requests are fast, users occasionally experience long delays.

```text
100 Requests

↓

95 Complete in 100 ms

↓

5 Complete in 5 Seconds
```

Those slow requests are called **tail latency**.

Hedging helps reduce these slow responses.

---

# Without Hedging

```text
Client

↓

API

↓

Server

↓

Slow Response (5 sec)

↓

Client Waits
```

The user must wait for the slow server.

---

# With Hedging

```text
                    Client

                       │

             First Request Sent

                       │

          ┌────────────┴────────────┐

          ▼                         ▼

      Server A                  Server B
      (Slow)                    (Fast)

          │                         │

     5 Seconds                 120 ms

          │                         │

          └────────────┬────────────┘

                       ▼

            Return First Response

                       │

                Cancel Slow Request
```

The application returns the **fastest response**.

---

# How Hedging Works

```text
Request Starts

↓

Send Request 1

↓

Wait 100 ms

↓

No Response?

│

├── No

│      │

│      ▼

│ Send Request 2

│

└── Yes

       │

       ▼

Return Response

↓

First Response Wins

↓

Cancel Remaining Requests
```

---

# Timeline Example

Without hedging:

```text
0 ms

↓

Request Sent

↓

5 Seconds

↓

Response
```

---

With hedging:

```text
0 ms

↓

Request 1

↓

100 ms

↓

Request 2

↓

220 ms

↓

Request 2 Returns

↓

Cancel Request 1
```

The user waits **220 ms** instead of **5 seconds**.

---

# Real-World Example

Suppose an application calls a product service.

```text
Application

↓

Product Service

↓

Sometimes Slow
```

Instead of waiting:

```text
Application

↓

Server A

↓

Slow
```

The application sends another request.

```text
Application

↓

Server A

↓

Slow

Application

↓

Server B

↓

Fast
```

Whichever server responds first wins.

---

# Hedging vs Retry

## Retry

```text
Request

↓

Failure

↓

Retry

↓

Success
```

Retry happens **after failure**.

---

## Hedging

```text
Request

↓

Still Running

↓

Second Request

↓

First Response Wins
```

Hedging happens **before failure**.

---

# Retry vs Hedging Comparison

| Retry | Hedging |
|--------|----------|
| Waits for failure | Doesn't wait for failure |
| Sequential | Parallel (or delayed parallel) |
| Higher latency | Lower latency |
| Fewer requests | More requests |
| Better for transient failures | Better for tail latency |

---

# Hedging Architecture

```text
                    Client

                       │

                       ▼

                  API Gateway

                       │

          ┌────────────┴────────────┐

          ▼                         ▼

    Service Instance 1       Service Instance 2

          │                         │

          └────────────┬────────────┘

                       ▼

            Return First Response
```

---

# Delayed Hedging

Instead of sending two requests immediately:

```text
0 ms

↓

Request 1

↓

Wait 100 ms

↓

Still Running?

↓

Send Request 2
```

This avoids unnecessary duplicate requests.

---

# Immediate Hedging

```text
Request

↓

Server A

Server B

↓

First Response Wins
```

Fastest but increases resource usage.

---

# Benefits

- Reduces tail latency
- Improves user experience
- Protects against slow servers
- Handles temporary network delays
- Increases availability
- Improves reliability

---

# Drawbacks

- More network traffic
- Additional CPU usage
- Higher infrastructure cost
- Duplicate requests
- Not suitable for non-idempotent operations

---

# Idempotency Requirement

Hedging should only be used with **idempotent operations**, where executing the same request multiple times produces the same result.

Safe examples:

- GET /products/1
- GET /orders/10
- Search APIs
- Read operations

Unsafe examples:

- Create Payment
- Place Order
- Transfer Money
- Send Email

Without idempotency:

```text
Request 1

↓

Create Order

Request 2

↓

Create Order Again
```

Two orders could be created.

---

# ASP.NET Core Example with Polly

Polly v8 provides built-in support for the Hedging strategy.

```csharp
builder.Services.AddHttpClient("ProductApi")
    .AddResilienceHandler("hedging", builder =>
    {
        builder.AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
        {
            MaxHedgedAttempts = 2,
            Delay = TimeSpan.FromMilliseconds(100)
        });
    });
```

Behavior:

```text
Request 1

↓

100 ms Delay

↓

Request 2

↓

First Successful Response Returned
```

---

# Hedging with Multiple Regions

```text
              Client

                 │

                 ▼

             API Gateway

        ┌────────┴────────┐

        ▼                 ▼

  India Region      Singapore Region

        │                 │

        └────────┬────────┘

                 ▼

       Fastest Response Wins
```

Useful in globally distributed systems.

---

# Hedging vs Load Balancing

| Hedging | Load Balancer |
|----------|---------------|
| Sends multiple requests | Sends one request |
| Uses first response | Chooses one server |
| Reduces latency | Distributes traffic |
| Higher resource usage | Lower resource usage |

---

# Hedging vs Failover

| Hedging | Failover |
|----------|----------|
| Parallel requests | Sequential requests |
| No need to wait for failure | Waits until failure |
| Lower latency | Higher latency |
| Higher resource cost | Lower resource cost |

---

# When to Use Hedging

Use when:

- Read-heavy workloads
- High latency variability
- Multiple service replicas
- Multi-region deployments
- Search services
- Recommendation engines
- Product catalog APIs
- AI inference endpoints
- CDN lookups

---

# When NOT to Use Hedging

Avoid when:

- Creating payments
- Bank transfers
- Order creation
- Inventory updates
- Sending emails
- SMS notifications
- Non-idempotent operations

---

# Best Practices

- Hedge only **idempotent** requests.
- Use a small delay (e.g., 50–200 ms) before sending the hedge request.
- Limit the number of hedged attempts.
- Cancel losing requests as soon as a winner is selected.
- Combine hedging with **timeouts**, **circuit breakers**, and **rate limiting**.
- Monitor the number of hedged requests to avoid unnecessary resource usage.

---

# Common Mistakes

## Hedging Every Request

```text
100 Requests

↓

200 Actual Requests
```

Unnecessary overhead.

Only hedge operations that are prone to tail latency.

---

## Hedging Non-Idempotent Operations

```text
Create Payment

↓

Two Requests

↓

Double Payment
```

Always ensure the operation is safe to repeat.

---

## Very Short Delay

```text
Request 1

↓

5 ms

↓

Request 2
```

The second request is almost always sent, increasing load.

---

## Never Cancelling Losing Requests

```text
Winner Returned

↓

Slow Requests Continue Running
```

This wastes server resources.

---

# Interview Questions

## Q1. What is the Hedging Pattern?

The Hedging Pattern sends multiple requests for the same operation (either immediately or after a short delay) and returns the first successful response while cancelling the remaining requests.

---

## Q2. What problem does Hedging solve?

It reduces **tail latency**, where a small percentage of requests take significantly longer than the average.

---

## Q3. How is Hedging different from Retry?

| Retry | Hedging |
|--------|----------|
| Waits for failure | Sends another request before failure |
| Sequential | Parallel or delayed parallel |
| Handles transient failures | Handles slow responses (tail latency) |

---

## Q4. Why should Hedging only be used for idempotent operations?

Because multiple requests may be processed simultaneously. If the operation changes data (such as creating an order), duplicate side effects may occur.

---

## Q5. Can Hedging be combined with other resilience patterns?

Yes. It is commonly combined with:

- Timeouts
- Circuit Breakers
- Rate Limiting
- Retries (for different failure scenarios)
- Load Balancing

---

# Key Takeaways

- **Hedging** reduces **tail latency** by sending an additional request before the first one fails.
- The **first successful response wins**, and remaining requests are cancelled.
- It is ideal for **read-only, idempotent operations** where latency is more important than minimizing duplicate requests.
- Hedging improves user experience but increases resource consumption, so it should be applied selectively.
- Modern .NET applications can implement hedging using **Polly v8 Resilience Pipelines**, often alongside **timeouts** and **circuit breakers** for comprehensive fault tolerance.