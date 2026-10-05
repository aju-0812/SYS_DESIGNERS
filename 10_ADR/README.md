<div align="center">

# 10. Architecture Decision Records (ADRs)

[![Status](https://img.shields.io/badge/ADR_Status-Approved-success?style=for-the-badge)]()
[![Standard](https://img.shields.io/badge/Format-MADR_Standard-blue?style=for-the-badge)]()
[![Decisions](https://img.shields.io/badge/Total_Records-7_ADRs-orange?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 09 Security & Observability](../09_Security_Observability/README.md)

---

</div>

## Architecture Decision Index

| ADR ID | Decision Title | Status | Primary Trade-Off Considered |
|:---:|---|:---:|---|
| **[ADR-001](#adr-001--primary-transactional-database-selection)** | Primary Transactional Database | `ACCEPTED` | ACID Consistency & Row-level Locking vs NoSQL Schema Flexibility |
| **[ADR-002](#adr-002--hybrid-synchronous-rest-and-asynchronous-event-messaging)** | Hybrid Synchronous REST + Async Messaging | `ACCEPTED` | Synchronous immediate UX feedback vs Asynchronous throughput |
| **[ADR-003](#adr-003--two-tier-concurrency-control-model)** | Two-Tier Concurrency Control Model | `ACCEPTED` | Pessimistic DB deadlocks vs Optimistic versioning with in-memory gate |
| **[ADR-004](#adr-004--in-memory-caching-tier)** | In-Memory Caching Tier | `ACCEPTED` | Cache invalidation complexity vs Sub-millisecond read latency |
| **[ADR-005](#adr-005--event-broker-and-dead-letter-queues)** | Event Broker & Dead Letter Queues | `ACCEPTED` | Operational overhead vs Message durability & fault isolation |
| **[ADR-006](#adr-006--end-to-end-idempotency-keys)** | End-to-End Idempotency Keys | `ACCEPTED` | Redis storage for UUIDs vs Preventing duplicate bank deductions |
| **[ADR-007](#adr-007--resilience-circuit-breakers-and-auto-reconciliation)** | Circuit Breakers & Reconciliation | `ACCEPTED` | Implementation complexity vs Preventing cascading system outages |

---

## ADR-001 — Primary Transactional Database Selection

- **Context**: Flash sales require absolute data consistency across inventory counters, customer accounts, and order state transitions.
- **Decision**: Adopt **PostgreSQL 16** with master-replica clustering.
- **Options Considered**: PostgreSQL (ACID relational), MongoDB (Document store), DynamoDB (Key-Value).
- **Rationale**: PostgreSQL provides strict serializability, ACID transactions, and deterministic row-level locking needed to guarantee zero overselling.

---

## ADR-002 — Hybrid Synchronous REST and Asynchronous Event Messaging

- **Context**: Customers need immediate checkout feedback, but backend fulfillment (3PL logistics, notifications) should not block response times.
- **Decision**: Use **Synchronous REST** for Checkout/Payment and **Asynchronous Message Queues** for Shipment/Notifications.
- **Rationale**: Isolates 3rd-party logistics latencies from user-facing checkout critical paths.

---

## ADR-003 — Two-Tier Concurrency Control Model

- **Context**: 10,000 requests competing for 100 stock units will cause database connection exhaustion and lock contention if hitting PostgreSQL directly.
- **Decision**: Implement a **Two-Tier Concurrency Gate**:
  1. **L1 (In-Memory)**: Redis Lua atomic decrement (`DECR`) filters out 9,900 requests in sub-millisecond time.
  2. **L2 (Database)**: PostgreSQL atomic conditional update with row versioning ensures authoritative persistence.
- **Rationale**: Protects the database while guaranteeing 100% stock consistency and zero overselling.

---

## ADR-004 — In-Memory Caching Tier

- **Context**: Product details and flash-sale countdown timers receive millions of read requests.
- **Decision**: Deploy a **Redis 7.0 Cluster** for product catalog caching and reservation hold tracking.
- **Rationale**: Eliminates 95% of database read queries during peak sales.

---

## ADR-005 — Event Broker and Dead Letter Queues

- **Context**: Downstream worker failures must not lose confirmed customer orders.
- **Decision**: Use **Apache Kafka / RabbitMQ** with Dead Letter Queues (DLQ) and exponential backoff retries.
- **Rationale**: Provides message durability, consumer grouping, and quarantined error handling.

---

## ADR-006 — End-to-End Idempotency Keys

- **Context**: Network retries or frantic double-clicks can submit identical payment requests.
- **Decision**: Enforce mandatory client-generated `Idempotency-Key` headers on all mutating APIs (`POST /api/inventory/reserve`, `POST /api/payments`).
- **Rationale**: Guarantees that duplicate requests return the original transaction result without double-charging.

---

## ADR-007 — Resilience: Circuit Breakers and Auto-Reconciliation

- **Context**: 3rd-party banking APIs can experience latency spikes, timeouts, or partial outages during flash sales.
- **Decision**: Implement **Resilience4j Circuit Breakers** combined with an **Auto-Reconciliation Polling Worker**.
- **Rationale**: Fails fast when payment providers degrade and self-heals ambiguous transactions without manual intervention.

---

[Return to System Overview](../README.md)