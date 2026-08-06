# Event-Driven Architecture (EDA)

# Introduction

**Event-Driven Architecture (EDA)** is an architectural style where systems communicate by producing and consuming **events** instead of making direct synchronous calls.

An **event** represents something that has already happened.

Examples:

- Order Created
- Payment Completed
- Invoice Generated
- Commodity Price Updated
- User Registered

Instead of one service calling another directly, services publish events to an event broker, allowing multiple consumers to react independently.

---

# Why Event-Driven Architecture?

Traditional synchronous communication:

```text
Order Service

↓

Payment Service

↓

Inventory Service

↓

Notification Service
```

Problems:

- Tight coupling
- Higher latency
- Cascading failures
- Difficult to scale

---

With Event-Driven Architecture:

```text
Order Service

↓

Event Broker

↓

Payment Service

Inventory Service

Notification Service

Analytics Service
```

Advantages:

- Loose coupling
- Independent scaling
- Better resilience
- Easier extensibility

---

# Event Flow

```text
Producer

↓

Event Broker

↓

Consumer 1

Consumer 2

Consumer 3
```

The producer doesn't know who consumes the event.

---

# Event Streaming vs Messaging vs Event Notification

Microsoft Azure provides multiple services because they solve different problems.

| Service | Purpose |
|----------|----------|
| Kafka | High-throughput event streaming |
| Azure Event Hubs | Telemetry and event ingestion |
| Azure Service Bus | Reliable enterprise messaging |
| Azure Event Grid | Lightweight event routing and notifications |

---

# 1. Apache Kafka

## What is Kafka?

Kafka is a **distributed event streaming platform** designed for:

- Massive throughput
- Event streaming
- Log aggregation
- Real-time analytics
- Event sourcing

---

## Architecture

```text
Producer

↓

Kafka Topic

↓

Partition

↓

Consumer Group
```

---

## Features

- Millions of events/sec
- Partitioning
- Replication
- Ordered events within a partition
- Event retention
- Replay capability

---

## Typical Use Cases

- Commodity market feeds
- Clickstream analytics
- IoT telemetry
- Financial transactions
- Audit logs
- Event sourcing
- AI training pipelines

---

## Advantages

- Very high throughput
- Horizontal scaling
- Long-term event retention
- Event replay
- Excellent for streaming

---

## Limitations

- Operational complexity (self-managed)
- Not ideal for request/reply messaging
- No built-in dead-letter queues (handled differently)

---

# 2. Azure Event Hubs

## What is Azure Event Hubs?

Azure Event Hubs is Microsoft's **fully managed event streaming platform**.

It provides Kafka-like capabilities without managing Kafka infrastructure.

---

## Architecture

```text
Devices

↓

Event Hubs

↓

Consumer Groups

↓

Analytics
```

---

## Features

- Millions of events/sec
- Partitioned event streams
- Multiple consumers
- Kafka protocol support
- Automatic scaling

---

## Typical Use Cases

- IoT devices
- Telemetry
- Application logs
- Sensor data
- Commodity market feeds
- Streaming analytics

---

## Advantages

- Fully managed
- High throughput
- Kafka-compatible endpoint
- Integrates with Azure services

---

## Limitations

- Primarily designed for streaming ingestion
- Not intended for enterprise workflows requiring transactions or complex messaging patterns

---

# 3. Azure Service Bus

## What is Azure Service Bus?

Azure Service Bus is a **reliable enterprise messaging service**.

It is designed for guaranteed message delivery between business applications.

---

## Architecture

```text
Application

↓

Queue / Topic

↓

Consumer
```

---

## Features

- Queues
- Topics
- Publish/Subscribe
- Dead Letter Queue (DLQ)
- Duplicate detection
- FIFO support (Sessions)
- Transactions
- Scheduled messages

---

## Typical Use Cases

- Order processing
- Payments
- Billing
- Inventory updates
- Financial systems
- Workflow orchestration

---

## Advantages

- Reliable delivery
- Enterprise-grade messaging
- Message durability
- Retry support
- Dead-letter queues
- Transactions

---

## Limitations

- Lower throughput than Kafka/Event Hubs
- Not optimized for large-scale telemetry streaming

---

# 4. Azure Event Grid

## What is Azure Event Grid?

Azure Event Grid is an **event routing service**.

It delivers lightweight notifications when something changes.

---

## Architecture

```text
Blob Created

↓

Event Grid

↓

Azure Function

↓

Logic App

↓

Webhook
```

---

## Features

- Push-based delivery
- Event filtering
- Serverless integration
- Near real-time notifications
- Cloud-native event routing

---

## Typical Use Cases

- Blob uploaded
- VM created
- Cosmos DB change
- Resource changes
- Storage notifications
- Serverless automation

---

## Advantages

- Extremely lightweight
- Automatic routing
- Native Azure integration
- Low operational overhead

---

## Limitations

- Not suitable for long-running workflows
- No event replay like Kafka
- Limited retention

---

# Service Comparison

| Feature | Kafka | Event Hubs | Service Bus | Event Grid |
|----------|--------|------------|-------------|------------|
| Primary Purpose | Event Streaming | Event Streaming | Enterprise Messaging | Event Notification |
| Managed Service | No (Apache) | Yes | Yes | Yes |
| High Throughput | ✅ Excellent | ✅ Excellent | Moderate | High |
| Event Replay | ✅ Yes | Limited (Retention-based) | ❌ No | ❌ No |
| Ordering | Per Partition | Per Partition | FIFO with Sessions | No |
| Transactions | ❌ | ❌ | ✅ Yes | ❌ |
| Dead Letter Queue | Limited | ❌ | ✅ Yes | Retry & Dead Letter (configurable destinations) |
| Pub/Sub | ✅ | Consumer Groups | Topics | Native |
| Typical Latency | Low | Low | Low | Very Low |
| Best For | Streaming | Telemetry | Business Workflows | Notifications |

---

# When Would You Use Each?

## Kafka

Use when:

- Streaming millions of events
- Event sourcing
- Real-time analytics
- Commodity market feeds
- Log aggregation
- AI/ML pipelines

Example:

```text
Commodity Prices

↓

Kafka

↓

Pricing Engine
```

---

## Azure Event Hubs

Use when:

- Collecting telemetry
- IoT devices
- Sensor data
- Streaming application logs
- Azure-native event ingestion

Example:

```text
100,000 IoT Devices

↓

Event Hubs

↓

Azure Stream Analytics
```

---

## Azure Service Bus

Use when:

- Orders
- Payments
- Billing
- Inventory updates
- Financial transactions
- Reliable asynchronous workflows

Example:

```text
Order Created

↓

Service Bus Queue

↓

Payment Service
```

---

## Azure Event Grid

Use when:

- Blob uploaded
- Storage changes
- Serverless automation
- Resource lifecycle events
- Event notifications

Example:

```text
Blob Uploaded

↓

Event Grid

↓

Azure Function
```

---

# Enterprise Solution Example

## Commodity Trading Platform

Requirements:

- Live commodity prices
- Order processing
- Notifications
- Analytics
- AI predictions
- Audit logging

---

## Architecture

```text
               Market Data Feed

                      │

                      ▼

                Azure Event Hubs

                      │

          ┌───────────┴───────────┐

          ▼                       ▼

    Pricing Engine          Analytics Engine

          │

          ▼

         Kafka

          │

     ┌────┼────┬─────────┐

     ▼    ▼    ▼         ▼

 Risk   AI   Reporting  Audit
 Engine Model  Service   Service

          │

          ▼

     Order Created

          │

          ▼

    Azure Service Bus

     ┌────┼─────┐

     ▼    ▼     ▼

 Payment Inventory Email

          │

          ▼

  Blob Storage Updated

          │

          ▼

     Azure Event Grid

     ┌────┼─────┐

     ▼    ▼     ▼

 Azure  Logic  Webhook
Function Apps
```

---

# Why This Combination?

### Azure Event Hubs

- Ingests massive real-time market data streams.
- Handles millions of incoming events efficiently.

---

### Kafka

- Distributes market events internally.
- Supports replay for analytics and AI model training.
- Feeds multiple downstream consumers independently.

---

### Azure Service Bus

- Coordinates critical business workflows.
- Guarantees reliable delivery for orders, payments, and settlements.
- Supports retries, transactions, and dead-letter queues.

---

### Azure Event Grid

- Triggers automation based on Azure resource changes.
- Sends notifications when blobs, files, or cloud resources change.
- Integrates easily with Azure Functions and Logic Apps.

---

# End-to-End Flow

```text
Live Market Feed

↓

Azure Event Hubs

↓

Pricing Engine

↓

Kafka

↓

Risk Engine

AI Models

Reporting

↓

Order Placed

↓

Azure Service Bus

↓

Payment

Inventory

Settlement

↓

Invoice PDF Stored

↓

Blob Storage

↓

Azure Event Grid

↓

Email Customer

↓

Archive Document
```

---

# Best Practices

- Use **Kafka** or **Event Hubs** for high-volume streaming data.
- Use **Service Bus** for business-critical commands and workflows.
- Use **Event Grid** for lightweight event notifications and Azure resource events.
- Keep events immutable and include timestamps and correlation IDs.
- Implement idempotent consumers to handle duplicate deliveries safely.
- Monitor consumer lag, dead-letter queues, throughput, and retry rates.
- Apply resilience patterns such as retries, circuit breakers, and bulkheads around consumers.

---

# Common Mistakes

## Using Service Bus for High-Volume Telemetry

```text
Millions of Sensor Events

↓

Service Bus
```

Poor fit. Event Hubs or Kafka is designed for this workload.

---

## Using Kafka for Business Transactions

```text
Payment Processing

↓

Kafka
```

Kafka excels at streaming but lacks the enterprise messaging features (transactions, sessions, DLQs) needed for many business workflows.

---

## Using Event Grid as a Queue

```text
Long Workflow

↓

Event Grid
```

Event Grid is for notifications, not workflow orchestration.

---

## Mixing Commands and Events

Commands:

```text
ProcessPayment
```

Sent to a single consumer.

Events:

```text
PaymentCompleted
```

Published for multiple interested consumers.

Keep these concepts separate.

---

# Interview Questions

## Q1. When would you choose Kafka over Azure Service Bus?

Kafka is ideal for high-throughput event streaming, analytics, and replayable event logs. Azure Service Bus is better for reliable business messaging, transactions, and workflow coordination.

---

## Q2. When should you use Azure Event Hubs?

Use Azure Event Hubs for ingesting massive volumes of telemetry, IoT data, application logs, or market data streams in Azure.

---

## Q3. What is Azure Event Grid best suited for?

Azure Event Grid is designed for lightweight event routing and notifications, especially for Azure resource changes and serverless automation.

---

## Q4. Can all four services be used together?

Yes. A common enterprise architecture uses:

- **Event Hubs** for ingestion
- **Kafka** for internal event streaming and analytics
- **Service Bus** for business workflows
- **Event Grid** for Azure event notifications and automation

Each service solves a different problem.

---

## Q5. Which service would you use for a commodity pricing platform?

- **Azure Event Hubs** to ingest live market feeds
- **Kafka** to distribute pricing events to analytics, AI, and risk systems
- **Azure Service Bus** for order processing, settlements, and payment workflows
- **Azure Event Grid** to trigger Azure Functions, notifications, and storage-based automation

---

# Key Takeaways

- **Kafka** is best for high-throughput, replayable event streaming.
- **Azure Event Hubs** is the Azure-native choice for ingesting telemetry and streaming data at massive scale.
- **Azure Service Bus** is designed for reliable enterprise messaging and business workflows.
- **Azure Event Grid** provides lightweight event routing for Azure resources and serverless automation.
- Large enterprise systems often combine all four services, using each where its strengths provide the greatest value.