<div align="center">

# 07. GoF and Distributed Design Patterns

[![Pattern Catalog](https://img.shields.io/badge/Catalog-8%20Key%20Patterns-blueviolet?style=for-the-badge)]()
[![Resilience](https://img.shields.io/badge/Resilience-Circuit%20Breaker-red?style=for-the-badge)]()
[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-success?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 06 SOLID Principles](../06_SOLID/README.md) |
[Next: 08 Scalability & Reliability ->](../08_Scalability_Reliability/README.md)

---

</div>

## Design Patterns Overview Matrix

| # | Pattern | Domain / Service | Primary Problem Solved |
|---|---|---|---|
| **1** | **Strategy** | `PaymentService` | Dynamically switch between UPI, Card, and NetBanking algorithms |
| **2** | **Factory** | `PaymentGatewayFactory` | Instantiate appropriate gateway clients at runtime |
| **3** | **State** | `OrderService` | Enforce rigid state transitions (`CREATED` -> `DELIVERED`) |
| **4** | **Observer / Pub-Sub** | `EventBroker` | Decouple order confirmation from Email/SMS & 3PL logistics |
| **5** | **Adapter** | `PaymentGatewayAdapter` | Standardize disparate 3rd-party banking payloads into standard DTOs |
| **6** | **Repository** | Storage Layer | Abstract relational PostgreSQL queries away from domain services |
| **7** | **Facade** | `CheckoutFacade` | Orchestrate cart validation, reservation, and payment behind 1 API |
| **8** | **Circuit Breaker** | `Resilience4j` | Prevent cascading crashes when downstream banking APIs degrade |

---

## 1. Deep Dive: Strategy and Factory Patterns

```mermaid
classDiagram
    class PaymentStrategy {
        <<interface>>
        +pay(BigDecimal amount): PaymentResult
    }
    class UPIStrategy {
        +pay(BigDecimal amount): PaymentResult
    }
    class CardStrategy {
        +pay(BigDecimal amount): PaymentResult
    }
    class PaymentStrategyFactory {
        +getStrategy(PaymentType type): PaymentStrategy
    }

    PaymentStrategy <|.. UPIStrategy
    PaymentStrategy <|.. CardStrategy
    PaymentStrategyFactory ..> PaymentStrategy : creates
```

---

## 2. State Pattern: Order State Transition Engine

```mermaid
stateDiagram-v2
    [*] --> CREATED: Checkout Triggered
    CREATED --> PAYMENT_PENDING: Stock Reserved (5-Min TTL)
    
    PAYMENT_PENDING --> CONFIRMED: Payment Authorized
    PAYMENT_PENDING --> CANCELLED: Payment Failed / TTL Expired
    
    CONFIRMED --> PROCESSING: Warehouse Pick Ticket Generated
    PROCESSING --> SHIPPED: Carrier Manifest Assigned
    SHIPPED --> OUT_FOR_DELIVERY: Hub Transit Completed
    OUT_FOR_DELIVERY --> DELIVERED: OTP Verified at Doorstep
    
    CANCELLED --> [*]
    DELIVERED --> [*]
```

---

## 3. Circuit Breaker Pattern (Resilience and Fail-Fast)

```mermaid
stateDiagram-v2
    [*] --> CLOSED: Normal Operations (Requests pass through)
    CLOSED --> OPEN: Failure rate exceeds 50% threshold
    
    OPEN --> HALF_OPEN: Sleep window expires (5 seconds)
    note right of OPEN: Fails fast immediately.<br/>Does NOT call bank gateway.
    
    HALF_OPEN --> CLOSED: Trial requests succeed (Healthy)
    HALF_OPEN --> OPEN: Trial requests fail (Still Down)
```

<details open>
<summary><b>Circuit Breaker State Transitions (Click to collapse)</b></summary>

1. **`CLOSED`**: All payment requests reach the gateway. Latency and errors are recorded in a rolling sliding window.
2. **`OPEN`**: If error rate surpasses 50%, the circuit trips to `OPEN`. All incoming calls immediately fail with `503 Service Unavailable`, protecting both the payment gateway and internal thread pools.
3. **`HALF-OPEN`**: After 5,000ms, a canary batch of 5 requests is permitted. If they succeed, the system returns to `CLOSED`.

</details>

---

## 4. Facade Pattern (Checkout Orchestration)

```mermaid
graph TD
    Client["Mobile / Web Client"] -->|Single POST /api/checkout| Facade["CheckoutFacade"]
    
    Facade -->|1. Validate| Cart["CartService"]
    Facade -->|2. Hold Stock (5m)| Inv["InventoryService"]
    Facade -->|3. Authorize| Pay["PaymentService"]
    Facade -->|4. Create| Ord["OrderService"]
```

---

[Return to System Overview](../README.md) | [Proceed to 08_Scalability_Reliability ->](../08_Scalability_Reliability/README.md)