# High-Level Architecture: AI-Powered Commodity Pricing Assistant

# Problem Statement

Design an **AI-powered Commodity Pricing Assistant** that enables traders, analysts, and business users to:

- Ask natural language questions about commodity prices
- Retrieve live and historical market data
- Analyze market trends
- Explain pricing movements using AI
- Query internal pricing policies and documentation
- Generate intelligent insights using Retrieval-Augmented Generation (RAG)

The system should be:

- Highly scalable
- Fault tolerant
- Secure
- Low latency
- Cloud-native
- AI-enabled

---

# Functional Requirements

Users should be able to:

- Ask questions like:
  - "What is today's Brent crude price?"
  - "Why did LNG prices increase yesterday?"
  - "Show the 30-day price trend for Natural Gas."
  - "What is the pricing policy for LNG contracts?"
- View live market prices
- View historical prices
- Access AI-generated explanations
- Search internal documentation
- Receive near real-time updates

---

# Non-Functional Requirements

- Response time < 3 seconds for AI queries
- High availability
- Horizontal scalability
- Secure authentication
- Real-time market updates
- Observability
- Audit logging
- Cost optimization

---

# High-Level Architecture

```text
                          Users

                            │

                            ▼

                    React Frontend

                            │

                    Azure AD / Entra ID

                            │

                            ▼

                  ASP.NET Core Web API

        ┌──────────────┬──────────────┬──────────────┐
        ▼              ▼              ▼

 Authentication   Pricing Service   AI Orchestrator

        │              │              │
        │              ▼              │
        │          Redis Cache        │
        │              │              │
        │              ▼              ▼
        │       SQL / MongoDB   Azure AI Search
        │                             │
        │                             ▼
        │                     Azure OpenAI
        │
        ▼

 Event Consumers

        ▲

 Kafka / Azure Event Hubs

        ▲

 Market Data Providers


Monitoring

↓

Application Insights

↓

Azure Monitor

↓

Grafana


Logging

↓

Serilog

↓

Elastic / Azure Log Analytics
```

---

# Component Overview

## React Frontend

Responsibilities:

- User authentication
- Commodity search
- AI chat interface
- Price dashboards
- Trend charts
- Historical reports
- Streaming updates

Technologies

- React
- TypeScript
- Material UI
- SignalR (optional)
- MSAL Authentication

---

# Authentication

Use:

```text
Microsoft Entra ID
```

Flow

```text
User

↓

Login

↓

Entra ID

↓

JWT Token

↓

React

↓

ASP.NET API
```

Benefits

- OAuth2
- OpenID Connect
- MFA
- Role-Based Access Control (RBAC)
- Single Sign-On (SSO)

---

# ASP.NET Core Backend

Acts as the central orchestration layer.

Responsibilities

- Authentication
- Authorization
- AI orchestration
- Commodity APIs
- Pricing APIs
- Chat APIs
- Redis integration
- SQL access
- Vector search
- Event publishing

---

# AI Orchestrator

Coordinates AI workflows.

Responsibilities

```text
Question

↓

Intent Detection

↓

Retrieve Context

↓

Build Prompt

↓

Azure OpenAI

↓

Return Answer
```

---

# Azure OpenAI

Provides

- GPT model
- Chat completion
- Summarization
- Reasoning
- Explanation generation

Example

```text
User

↓

Why is Brent increasing?

↓

GPT

↓

AI Explanation
```

---

# Azure AI Search (Vector Store)

Stores

- Pricing policies
- Market reports
- Contract documents
- Research papers
- Commodity documentation

Workflow

```text
Documents

↓

Chunking

↓

Embeddings

↓

Azure AI Search

↓

Vector Search

↓

Relevant Chunks

↓

GPT
```

---

# RAG Flow

```text
User Question

↓

Embedding

↓

Azure AI Search

↓

Top K Chunks

↓

Prompt Builder

↓

Azure OpenAI

↓

Answer
```

---

# Market Data Pipeline

Live feeds arrive continuously.

```text
Commodity Exchange

↓

Market Feed

↓

Kafka / Event Hubs

↓

Pricing Service

↓

Redis

↓

SQL Database

↓

API
```

---

# Kafka / Azure Event Hubs

Responsibilities

- Live market prices
- Event streaming
- High throughput
- Decoupling producers and consumers

Example Events

```text
PriceUpdated

CurveUpdated

FXUpdated

SettlementPriceReceived
```

---

# Redis

Purpose

- Low-latency cache
- Session cache
- Frequently requested prices
- AI response cache

Example Keys

```text
price:BRENT

price:LNG

fx:USDINR

chat:session:123
```

TTL

- Live prices: 5–10 seconds
- AI responses: 5–15 minutes
- Metadata: Several hours

---

# SQL Database

Stores

- Historical prices
- Trades
- Pricing rules
- User data
- Audit records

Example Tables

```text
CommodityPrice

HistoricalPrice

Exchange

AuditLog
```

---

# MongoDB

Stores

- Chat history
- AI conversations
- Prompt logs
- Document metadata
- User preferences

Example Document

```json
{
  "userId": "1001",
  "question": "Why is LNG rising?",
  "response": "...",
  "timestamp": "2026-08-06T09:30:00Z"
}
```

---

# Monitoring

Tools

- Azure Monitor
- Application Insights
- Grafana
- Prometheus (optional)

Monitor

- API latency
- AI latency
- Redis hit ratio
- Kafka consumer lag
- Token usage
- Cache misses
- Error rate
- Database response time

---

# Logging

Use

```text
Serilog
```

Logs

```text
API Request

↓

Structured Logs

↓

Azure Log Analytics

↓

Dashboard
```

Capture

- Request IDs
- Correlation IDs
- AI prompt IDs (excluding sensitive content where appropriate)
- Exceptions
- User actions
- Performance metrics

---

# Security

Authentication

- Microsoft Entra ID
- OAuth2
- OpenID Connect

Authorization

- RBAC

Encryption

- HTTPS
- TLS
- Encryption at Rest

Secrets

- Azure Key Vault

API Protection

- Azure API Management (Optional)
- Rate Limiting

---

# Complete Data Flow

## AI Query

```text
User

↓

React UI

↓

Entra Authentication

↓

ASP.NET API

↓

AI Orchestrator

↓

Azure AI Search

↓

Relevant Documents

↓

Azure OpenAI

↓

AI Response

↓

React UI
```

---

## Live Pricing Flow

```text
Commodity Exchange

↓

Market Feed

↓

Kafka / Event Hubs

↓

Pricing Service

↓

Redis

↓

SQL Database

↓

Pricing API

↓

React Dashboard
```

---

## RAG Document Ingestion

```text
Pricing Policy PDF

↓

Document Processor

↓

Chunking

↓

Embedding Model

↓

Azure AI Search

↓

Vector Index
```

---

# Scalability

```text
                 Azure Front Door

                        │

             Azure API Management

                        │

      ┌──────────┬──────────┬──────────┐

      ▼          ▼          ▼

   API 1      API 2      API 3

      │          │          │

      └──────────┼──────────┘

                 ▼

          Redis Cluster

                 ▼

      SQL Always On Cluster

                 ▼

        Azure AI Search

                 ▼

         Azure OpenAI
```

Each API instance is stateless and can scale independently.

---

# Resilience Patterns

Use

- Retry
- Circuit Breaker
- Timeout
- Bulkhead
- Cache Aside
- Refresh Ahead
- Rate Limiting

Example

```text
OpenAI

↓

Timeout

↓

Retry

↓

Circuit Breaker

↓

Fallback Response
```

---

# Monitoring Dashboard

Track

- Requests/sec
- Active users
- Token consumption
- AI latency
- Cache hit ratio
- Kafka lag
- SQL latency
- Error rate
- Failed prompts
- OpenAI cost
- Search latency

---

# Future Enhancements

- AI Agents for automated pricing analysis
- Voice-enabled commodity assistant
- Multi-language support
- Fine-tuned commodity-specific LLM
- Predictive price forecasting
- Personalized recommendations
- Automated market alerts
- Multi-region deployment

---

# Technology Stack

| Layer | Technology |
|---------|------------|
| Frontend | React + TypeScript |
| Authentication | Microsoft Entra ID (Azure AD) |
| API | ASP.NET Core |
| AI | Azure OpenAI |
| Vector Search | Azure AI Search |
| Event Streaming | Kafka / Azure Event Hubs |
| Cache | Redis |
| Relational Database | SQL Server / Azure SQL |
| NoSQL | MongoDB |
| Monitoring | Azure Monitor + Application Insights + Grafana |
| Logging | Serilog + Azure Log Analytics |
| Secrets | Azure Key Vault |
| API Gateway | Azure API Management |
| Hosting | Azure App Service / AKS |

---

# Architecture Decisions

| Decision | Reason |
|----------|--------|
| React | Rich, responsive user experience |
| ASP.NET Core | High-performance backend with strong Azure integration |
| Azure OpenAI | Enterprise-grade generative AI capabilities |
| Azure AI Search | Semantic search with vector indexing for RAG |
| Kafka/Event Hubs | High-throughput streaming of live market data |
| Redis | Ultra-fast caching to reduce latency and database load |
| SQL | Strong consistency for transactional and historical pricing data |
| MongoDB | Flexible storage for chat history and AI metadata |
| Entra ID | Secure enterprise authentication and SSO |
| Application Insights | End-to-end observability and diagnostics |
| Serilog | Structured logging with correlation across distributed services |

---

# Interview Questions

## Q1. Why use both SQL Server and MongoDB?

SQL Server stores structured transactional and historical pricing data requiring ACID guarantees. MongoDB stores flexible, evolving data such as chat history, prompt metadata, and AI conversations.

---

## Q2. Why use Azure AI Search instead of querying SQL directly?

Azure AI Search provides semantic vector search over unstructured documents, enabling Retrieval-Augmented Generation (RAG). SQL is optimized for structured relational queries, not semantic retrieval.

---

## Q3. Why introduce Redis?

Redis reduces latency by caching frequently accessed prices, metadata, and AI responses while significantly lowering database load.

---

## Q4. Why use Kafka or Event Hubs?

They ingest and distribute high-volume market events in real time, decoupling producers from consumers and allowing multiple downstream systems to process pricing updates independently.

---

## Q5. How is the system secured?

Authentication is handled by Microsoft Entra ID using OAuth2/OpenID Connect. APIs validate JWT tokens, enforce role-based authorization, store secrets in Azure Key Vault, and expose endpoints over HTTPS.

---

# Key Takeaways

- The architecture combines **React**, **ASP.NET Core**, **Azure OpenAI**, and **Azure AI Search** to deliver an AI-powered RAG experience.
- **Kafka/Event Hubs** stream live commodity market data, while **Redis** provides low-latency caching for frequently accessed information.
- **SQL Server** stores transactional and historical pricing data, while **MongoDB** stores flexible AI-related data such as conversations and prompt metadata.
- **Microsoft Entra ID**, **Azure Monitor**, **Application Insights**, **Serilog**, and **Azure Key Vault** provide enterprise-grade security, observability, and operations.
- The design is scalable, resilient, and cloud-native, making it suitable for high-volume commodity pricing platforms used by enterprise trading organizations.