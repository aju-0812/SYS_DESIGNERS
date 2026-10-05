<div align="center">

# 06. SOLID Object-Oriented Principles in SALESTORM

[![Standard](https://img.shields.io/badge/Architecture-Clean%20%26%20Decoupled-blue?style=for-the-badge)]()
[![Code Quality](https://img.shields.io/badge/Maintainability-High-brightgreen?style=for-the-badge)]()
[![Testability](https://img.shields.io/badge/Testability-Isolated%20Mocks-orange?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 05 API Design](../05_API/README.md) |
[Next: 07 Design Patterns ->](../07_Design_Patterns/README.md)

---

</div>

## SOLID Architecture Diagram Preview

<div align="center">

![SALESTORM SOLID Architecture Diagram](DIAGRAM.jpeg)

*Figure 6.1: SALESTORM SOLID Principles Implementation — Domain Separation, Interfaces, and Dependency Inversion*

</div>

---

## Summary of SOLID Principles Applied

```mermaid
mindmap
  root((SOLID Principles))
    S: Single Responsibility
      Dedicated services per domain
      Isolated fault boundaries
    O: Open Closed
      Plug-and-play payment providers
      Configurable checkout strategies
    L: Liskov Substitution
      Interchangeable payment gateways
      Standardized order handlers
    I: Interface Segregation
      Narrow payment vs refund interfaces
      Specialized query vs command traits
    D: Dependency Inversion
      Business logic depends on abstractions
      Mockable database repositories
```

---

## 1. S — Single Responsibility Principle (SRP)

> **Rule:** *A class or module should have one, and only one, reason to change.*

```mermaid
graph LR
    subgraph Services["Single-Domain Services"]
        INV["InventoryService<br/>(Stock checks & reservations)"]
        PAY["PaymentService<br/>(Gateway calls & idempotency)"]
        ORD["OrderService<br/>(Order state persistence)"]
        NOTIF["NotificationService<br/>(Email/SMS alerts)"]
    end
```

- **In Practice:** Modifying bank payment handling never forces changes or redeployment of `InventoryService` or `OrderService`.

---

## 2. O — Open/Closed Principle (OCP)

> **Rule:** *Software entities should be open for extension, but closed for modification.*

```mermaid
classDiagram
    class PaymentGateway {
        <<interface>>
        +processPayment(PaymentRequest): PaymentResponse
        +verifyStatus(String txnRef): PaymentStatus
    }
    class UPIPaymentGateway {
        +processPayment(PaymentRequest): PaymentResponse
    }
    class StripePaymentGateway {
        +processPayment(PaymentRequest): PaymentResponse
    }
    class RazorpayPaymentGateway {
        +processPayment(PaymentRequest): PaymentResponse
    }

    PaymentGateway <|.. UPIPaymentGateway
    PaymentGateway <|.. StripePaymentGateway
    PaymentGateway <|.. RazorpayPaymentGateway
```

- **In Practice:** Adding Apple Pay or digital wallet checkout requires authoring a new implementation of `PaymentGateway` without modifying existing core payment controllers.

---

## 3. L — Liskov Substitution Principle (LSP)

> **Rule:** *Subtypes must be substitutable for their base types without altering system correctness.*

```java
// Any implementation fulfills the exact same transactional contract
public interface PaymentProcessor {
    PaymentResult executeTransaction(PaymentContext context);
}

// PaymentService operates transparently regardless of concrete gateway
PaymentProcessor processor = paymentFactory.getProcessor(userSelectedMethod);
PaymentResult result = processor.executeTransaction(context);
```

---

## 4. I — Interface Segregation Principle (ISP)

> **Rule:** *Clients should not be forced to depend upon interfaces that they do not use.*

```mermaid
classDiagram
    class PaymentProcessable {
        <<interface>>
        +charge(BigDecimal amount)
    }
    class Refundable {
        <<interface>>
        +refund(String txnId, BigDecimal amount)
    }
    class PaymentReconcilable {
        <<interface>>
        +pollStatus(String txnId)
    }

    class StandardCreditCardProcessor
    PaymentProcessable <|.. StandardCreditCardProcessor
    Refundable <|.. StandardCreditCardProcessor
    PaymentReconcilable <|.. StandardCreditCardProcessor
```

---

## 5. D — Dependency Inversion Principle (DIP)

> **Rule:** *High-level modules should not depend on low-level modules. Both should depend on abstractions.*

```mermaid
graph TD
    HighLevel["OrderService (High-Level Domain Logic)"] -->|Depends On| Abstraction["«interface»<br/>OrderRepository"]
    PostgresImpl["PostgresOrderRepository (Low-Level DB Driver)"] -.->|Implements| Abstraction
```

- **In Practice:** We can test `OrderService` with an in-memory test repository without needing an active PostgreSQL instance.

---

[Return to System Overview](../README.md) | [Proceed to 07_Design_Patterns ->](../07_Design_Patterns/README.md)