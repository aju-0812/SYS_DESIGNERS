<div align="center">

# 03. Low-Level Design (LLD) and Sequence Workflows

[![Scope](https://img.shields.io/badge/Design-Low--Level%20UML%20%26%20Classes-blueviolet?style=for-the-badge)]()
[![State Machine](https://img.shields.io/badge/State%20Machine-Deterministic-success?style=for-the-badge)]()
[![Idempotency](https://img.shields.io/badge/Idempotency-Guaranteed-informational?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 02 High-Level Design](../02_HLD/README.md) |
[Next: 04 Database Design ->](../04_Database/README.md)

---

</div>

## LLD Diagram Preview

<div align="center">

![SALESTORM Low Level Design UML and Class Diagram](DIAGRAM.jpeg)

*Figure 3.1: SALESTORM Low-Level Design — Class Architecture, Service Controllers, Repositories, and State Machines*

</div>

---

## 1. Core Class Hierarchy and Component Model

```mermaid
classDiagram
    class InventoryController {
        +reserveStock(ReserveRequest): ResponseEntity
        +confirmReservation(String resId): ResponseEntity
        +releaseReservation(String resId): ResponseEntity
    }

    class InventoryService {
        -InventoryRepository inventoryRepo
        -ReservationRepository reservationRepo
        -RedisAtomicCounter redisCounter
        +reserve(Long productId, int qty, String idempotencyKey) ReservationResult
        +confirm(String reservationId) boolean
        +release(String reservationId) boolean
    }

    class PaymentController {
        +processPayment(PaymentRequest): ResponseEntity
        +reconcilePayment(String paymentId): ResponseEntity
    }

    class PaymentService {
        -PaymentGateway paymentGateway
        -PaymentRepository paymentRepo
        -IdempotencyManager idempotencyManager
        -CircuitBreaker circuitBreaker
        +pay(PaymentCommand cmd) PaymentResult
        +reconcile(String transactionRef) PaymentStatus
    }

    class OrderService {
        -OrderRepository orderRepo
        -OrderStateMachine stateMachine
        -EventPublisher eventPublisher
        +createOrder(OrderRequest req) Order
        +transitionState(Long orderId, OrderEvent event) Order
    }

    class OrderStateMachine {
        +validateTransition(OrderStatus current, OrderStatus target) boolean
        +applyEvent(Order order, OrderEvent event) void
    }

    InventoryController --> InventoryService
    PaymentController --> PaymentService
    OrderService --> OrderStateMachine
```

---

## 2. Concurrency-Safe Flash Sale Sequence

```mermaid
sequenceDiagram
    autonumber
    actor Shopper as Shopper
    participant API as API Gateway
    participant InvCtrl as InventoryController
    participant InvSvc as InventoryService
    participant Redis as Redis (Lua Script)
    participant DB as PostgreSQL
    participant PaySvc as PaymentService
    participant Bank as Bank Gateway

    Shopper->>API: POST /api/inventory/reserve (productId=101, qty=1, idempotencyKey="RES-8899")
    API->>InvCtrl: Route Request
    InvCtrl->>InvSvc: reserve(101, 1, "RES-8899")
    
    InvSvc->>Redis: Execute Atomic Lua Script (EVALSHA check_and_decr)
    alt Available Stock > 0 (1 of 100 units)
        Redis-->>InvSvc: DECR successful (Remaining: 99)
        InvSvc->>DB: INSERT INTO inventory_reservation (id, customer_id, qty, status='RESERVED', expires_at=NOW()+5m)
        InvSvc-->>Shopper: 201 Created (reservationId: "RES-8899", expiresAt: "15:35:00")
    else Available Stock == 0 (Units 101+)
        Redis-->>InvSvc: 0 (Inventory Depleted)
        InvSvc-->>Shopper: 409 Conflict ("Flash Sale Stock Out")
    end

    Shopper->>API: POST /api/payments (reservationId="RES-8899", amount=999.00, key="PAY-7766")
    API->>PaySvc: Process Payment
    PaySvc->>Bank: Charge Customer
    alt Payment Succeeded
        Bank-->>PaySvc: 200 OK (txn="TXN-3344")
        PaySvc->>InvSvc: confirm("RES-8899")
        InvSvc->>DB: UPDATE inventory SET sold_quantity = sold_quantity + 1, reserved_quantity = reserved_quantity - 1
        PaySvc-->>Shopper: 200 OK (Payment Confirmed & Order Placed)
    else Payment Failed / Expired
        Bank-->>PaySvc: 400 Failed
        PaySvc->>InvSvc: release("RES-8899")
        InvSvc->>Redis: INCR product:101:stock (Restored +1)
        InvSvc->>DB: UPDATE inventory SET available_quantity = available_quantity + 1, reserved_quantity = reserved_quantity - 1
        PaySvc-->>Shopper: 400 Bad Request ("Payment failed. Stock returned to pool.")
    end
```

---

## 3. Deterministic Order State Machine

```mermaid
stateDiagram-v2
    [*] --> CREATED: User clicks Checkout
    CREATED --> PAYMENT_PENDING: Stock Reserved (5 min TTL)
    
    PAYMENT_PENDING --> CONFIRMED: Payment Captured Successfully
    PAYMENT_PENDING --> CANCELLED: Payment Failed / TTL Expired
    
    CONFIRMED --> PROCESSING: Warehouse Pack Ticket Created
    PROCESSING --> SHIPPED: Handed to 3PL Logistics
    SHIPPED --> OUT_FOR_DELIVERY: Final Mile Hub Dispatch
    OUT_FOR_DELIVERY --> DELIVERED: Customer Delivery Handshake
    
    CANCELLED --> [*]
    DELIVERED --> [*]
```

---

## 4. Concurrency and Optimistic Lock Implementation

<details open>
<summary><b>Atomic SQL Update Query (Click to Expand)</b></summary>

```sql
-- Prevents race conditions at the database level with version validation
UPDATE inventory
SET 
    available_quantity = available_quantity - 1,
    reserved_quantity = reserved_quantity + 1,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE 
    product_id = :productId 
    AND available_quantity >= 1 
    AND version = :currentVersion;
```

</details>

---

[Return to System Overview](../README.md) | [Proceed to 04_Database ->](../04_Database/README.md)