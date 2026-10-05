<div align="center">

# 08. Scalability, Resilience, and Fault Tolerance

[![Scale](https://img.shields.io/badge/Traffic_Scale-10k_to_500k_RPS-red?style=for-the-badge)]()
[![High Availability](https://img.shields.io/badge/Availability-99.99%25-brightgreen?style=for-the-badge)]()
[![Fault Tolerance](https://img.shields.io/badge/Fault_Tolerance-Self--Healing-blue?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 07 Design Patterns](../07_Design_Patterns/README.md) |
[Next: 09 Security & Observability ->](../09_Security_Observability/README.md)

---

</div>

## Scalability and Reliability Architecture Overview

```mermaid
flowchart TD
    subgraph TrafficScale["Traffic Scaling Dimensions"]
        T1["Edge Caching (Cloudflare CDN)"]
        T2["Stateless App Auto-scaling (Kubernetes HPA)"]
        T3["In-Memory Hot Counters (Redis Cluster)"]
        T4["PostgreSQL Read Replicas (PgBouncer Pooling)"]
    end

    subgraph FaultTolerance["Resilience & Recovery"]
        F1["Circuit Breakers (Resilience4j)"]
        F2["Dead Letter Queues (Kafka DLQ)"]
        F3["Idempotency Deduplication Store"]
        F4["Background Reconciliation Poller"]
    end
```

---

## 1. Handling Traffic Surges (10,000 to 500,000 RPS)

| Layer | Scaling Mechanism | Sizing & Throughput Strategy |
|---|---|---|
| **DNS & Edge** | Cloudflare Anycast CDN & WAF | Absorbs 90% of read traffic, filters layer-7 bot spikes |
| **API Gateway** | Kong / Spring Cloud Gateway | Token-bucket rate limiting (500 rps/client IP) |
| **Microservices Mesh** | Kubernetes Pod Autoscaler (HPA) | Auto-scales instances based on CPU (>70%) and request queue depth |
| **Caching Layer** | Redis 7.0 Cluster | Sub-millisecond read/write latency; 100,000+ ops/sec per node |
| **Database Master** | PostgreSQL 16 (Write-Optimized) | Row-level versioning, connection pooling via PgBouncer |
| **Database Replicas**| PostgreSQL Read Replicas | Multi-AZ async replication for catalog and order history queries |

---

## 2. Horizontal Scaling Topology

```mermaid
graph TD
    ALB["AWS Application Load Balancer"] --> Node1["App Pod 1"]
    ALB --> Node2["App Pod 2"]
    ALB --> Node3["App Pod 3"]
    ALB --> NodeN["App Pod N (Auto-Scaled)"]

    Node1 & Node2 & Node3 & NodeN --> Redis[("Redis Cluster (In-Memory Hot Path)")]
    Node1 & Node2 & Node3 & NodeN --> Master[("PostgreSQL Master (Writes)")]
    Node1 & Node2 & Node3 & NodeN -.-> Replica1[("PostgreSQL Replica 1 (Reads)")]
    Node1 & Node2 & Node3 & NodeN -.-> Replica2[("PostgreSQL Replica 2 (Reads)")]
```

---

## 3. Reliability and Self-Healing Mechanisms

```mermaid
stateDiagram-v2
    [*] --> InFlight: Customer Checkout Initiated
    InFlight --> Captured: Bank Payment Succeeded
    InFlight --> TimedOut: Bank Gateway Unresponsive (Timeout > 3000ms)
    
    TimedOut --> Reconciling: Background Worker Polls Bank API
    Reconciling --> ConfirmedOrder: Bank Confirmed Transaction
    Reconciling --> ReleaseStock: Bank Confirmed Charge Failed
    
    Captured --> ConfirmedOrder: Order Record Created
    ReleaseStock --> AvailablePool: Stock Restored (+1)
    
    ConfirmedOrder --> [*]
    AvailablePool --> [*]
```

<details open>
<summary><b>Dead Letter Queue (DLQ) & Auto-Reconciliation Engine (Click to expand/collapse)</b></summary>

### 1. Dead Letter Queue (DLQ) Workflow
- When an asynchronous event (e.g., `OrderConfirmedEvent` for shipment dispatch) fails processing after **3 exponential backoff retries** (1s, 2s, 4s), it is moved to `order.confirmed.dlq`.
- Prevents poison-pill messages from blocking the active partition consumer while triggering an automated SRE alert.

### 2. Automated Reconciliation Poller
- If a payment request times out after 3 seconds, the system does NOT immediately assume failure.
- A scheduled cron worker queries the payment gateway with the `Idempotency-Key` / `transactionReference` every 30 seconds.
- If verified successful, the order transitions to `CONFIRMED`. If verified failed or expired after 5 minutes, the stock reservation is released back to the general pool.

</details>

---

[Return to System Overview](../README.md) | [Proceed to 09_Security_Observability ->](../09_Security_Observability/README.md)