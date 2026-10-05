<div align="center">

# 02. High-Level Design (HLD)

[![Tier](https://img.shields.io/badge/Architecture-Distributed%20Microservices-blue?style=for-the-badge)]()
[![Pattern](https://img.shields.io/badge/Design-Event--Driven%20CQRS-orange?style=for-the-badge)]()
[![Flow Diagram](https://img.shields.io/badge/Draw.io-Available-green?style=for-the-badge)](../SALESTORM_SYSTEM_FLOW.drawio)

[Back to Overview](../README.md) |
[Prev: 01 Requirements](../01_Requirements/README.md) |
[Next: 03 Low-Level Design ->](../03_LLD/README.md)

---

</div>

## Architecture Diagram Preview

<div align="center">

![SALESTORM High Level Design Architecture Diagram](DIAGRAM.jpeg)

*Figure 2.1: SALESTORM High-Level Architecture Diagram — Ingestion, Microservices Mesh, Storage and Async Fulfillment Tier*

</div>

---

## 1. Interactive C4 Container Architecture

```mermaid
graph TB
    subgraph Users["Clients and Edge Gateway"]
        U["10,000 Flash Sale Shoppers"] -->|HTTPS / TLS 1.3| CDN["Cloudflare CDN & WAF<br/>(DDoS Protection & Caching)"]
        CDN -->|Filtered Traffic| ALB["Application Load Balancer<br/>(AWS ALB / NGINX)"]
        ALB -->|Forward Requests| GW["API Gateway<br/>(Kong / Spring Cloud)"]
    end

    subgraph ServiceMesh["Core Microservices Tier"]
        GW -->|Route Requests| PROD["Product & Catalog Service"]
        GW -->|Route Requests| CART["Cart Service"]
        GW -->|Route Requests| CHK["Checkout Facade Service"]
        GW -->|Route Requests| INV["Inventory & Reservation Service [Critical Gate]"]
        GW -->|Route Requests| PAY["Payment Service"]
        GW -->|Route Requests| ORD["Order Service"]
    end

    subgraph DataStorage["Data and In-Memory Tier"]
        INV & CHK -->|Sub-millisecond Atomic Holds| REDIS[("Redis 7.0 Cluster<br/>(Hot Stock Counter & Locks)")]
        INV & ORD & PAY -->|ACID Transactions| PG_MASTER[("PostgreSQL Master<br/>(Optimistic Locking)")]
        PROD & CART -.->|Read Scalability| PG_REPLICA[("PostgreSQL Read Replicas")]
    end

    subgraph AsyncTier["Asynchronous Event Mesh"]
        ORD & PAY -->|Publish State Events| KAFKA{{"Apache Kafka / RabbitMQ<br/>(Event Topics + DLQ)"}}
        KAFKA -->|order.confirmed| SHIP["Shipment Service"]
        KAFKA -->|order.confirmed / payment.failed| NOTIF["Notification Service"]
        KAFKA -->|reservation.expired| EXP_WORKER["Expiry Sweeper Worker"]
        EXP_WORKER -.->|Restore Released Stock| INV
    end

    subgraph External["External Service Integrations"]
        PAY -->|Circuit Breaker Monitored| BANK["External Payment Gateway<br/>(Razorpay / Stripe)"]
        SHIP -->|Logistics API| CARRIER["3PL Courier APIs<br/>(DHL / FedEx / Shiprocket)"]
        NOTIF -->|SMS & Email APIs| PROVIDER["Twilio / SendGrid"]
    end

    classDef edge fill:#e0f2fe,stroke:#0284c7,stroke-width:2px;
    classDef svc fill:#fef3c7,stroke:#d97706,stroke-width:2px;
    classDef data fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef async fill:#f3e8ff,stroke:#9333ea,stroke-width:2px;
    classDef ext fill:#f1f5f9,stroke:#475569,stroke-width:2px;

    class U,CDN,ALB,GW edge;
    class PROD,CART,CHK,INV,PAY,ORD svc;
    class REDIS,PG_MASTER,PG_REPLICA data;
    class KAFKA,SHIP,NOTIF,EXP_WORKER async;
    class BANK,CARRIER,PROVIDER ext;
```

---

## 2. Microservice Boundaries and Responsibilities

| Service | Primary Domain | Scaling Strategy | Failure Mode Mitigation |
|---|---|---|---|
| **API Gateway** | Rate limiting, Auth validation, Header tracing | Auto-scaled edge pods | Return `429 Too Many Requests` on surge beyond capacity |
| **Product Service** | Catalog browsing, Flash-sale countdowns | 99% Read Cached in Redis & CDN | Stale-while-revalidate caching fallback |
| **Cart Service** | Temporary user baskets and validation | In-Memory Redis session backing | Ephemeral storage with graceful cleanup |
| **Inventory Service [Gate]** | Concurrency-safe atomic reservations | Redis Lua Scripts + PostgreSQL locks | Strict rejection when stock hits 0; zero overselling |
| **Checkout Service** | Orchestration Facade across services | Stateless horizontal scaling | Idempotent transaction orchestration |
| **Payment Service** | Gateway integrations and reconciliation | Stateless workers with Circuit Breakers | Resilience4j Circuit Breaker + Auto-reconciliation worker |
| **Order Service** | State machine and order confirmation | ACID Transactions + Outbox pattern | Asynchronous retries via Dead Letter Queues |
| **Shipment Service** | 3PL logistics dispatch and tracking | Event-driven background consumers | Message queue consumer retry with exponential backoff |
| **Notification Service**| Email, SMS, and Push alert dispatch | Asynchronous queue consumers | Batching and circuit breaker on 3rd party providers |

---

## 3. The Flash Sale 10,000 : 100 Traffic Funnel

```mermaid
flowchart TD
    A["10,000 Concurrent Hits at 00:00:00"] --> B["Cloudflare WAF / CDN<br/>(Filters 1,000 bot & duplicate bursts)"]
    B --> C["API Gateway Rate Limiter<br/>(Smooths traffic spikes)"]
    C --> D["Inventory Service (Redis Atomic Lua DECR)"]
    
    D -->|First 100 Requests| E["[SUCCESS] 100 Reservations Granted<br/>(5-Minute Payment Window)"]
    D -->|Remaining 8,900+ Requests| F["[REJECTED] Out of Stock Response<br/>(Fast Fail in <15ms)"]
    
    E --> G["Payment Gateway Processing"]
    G -->|95 Successful Payments| H["[CONFIRMED] 95 Confirmed Orders"]
    G -->|5 Failed / Timed Out| I["[RESTORED] Stock Returned to Pool (+5 Available)"]
    
    I -.->|Next waiting shoppers reserve| D
```

---

## Related Modules and Diagrams

- **Interactive Draw.io Diagram**: [`SALESTORM_SYSTEM_FLOW.drawio`](../SALESTORM_SYSTEM_FLOW.drawio)
- **Low-Level Code and Sequences**: [`03_LLD/README.md`](../03_LLD/README.md)
- **PostgreSQL and Redis Design**: [`04_Database/README.md`](../04_Database/README.md)
- **Architecture Decisions (ADRs)**: [`10_ADR/README.md`](../10_ADR/README.md)

---

[Return to System Overview](../README.md) | [Proceed to 03_LLD ->](../03_LLD/README.md)