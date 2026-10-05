<div align="center">

# 04. Database Architecture and Relational Schema

[![Engine](https://img.shields.io/badge/Primary_DB-PostgreSQL_16_ACID-336791?style=for-the-badge&logo=postgresql)]()
[![Cache](https://img.shields.io/badge/Cache-Redis_7.0_Cluster-DC382D?style=for-the-badge&logo=redis)]()
[![Consistency](https://img.shields.io/badge/Model-Strict_Serializability-success?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 03 Low-Level Design](../03_LLD/README.md) |
[Next: 05 API Design ->](../05_API/README.md)

---

</div>

## Database Diagram Preview

<div align="center">

![SALESTORM Database Entity Relationship ER Diagram](DIAGRAM.jpeg)

*Figure 4.1: SALESTORM Relational Database Schema — Entity Relationships, Foreign Keys, and Concurrency Controls*

</div>

---

## 1. Interactive Entity Relationship (ER) Diagram

```mermaid
erDiagram
    CUSTOMER ||--o{ CART : owns
    CUSTOMER ||--o{ ORDER : places
    CUSTOMER ||--o{ INVENTORY_RESERVATION : holds
    
    CATEGORY ||--o{ PRODUCT : categorizes
    PRODUCT ||--|| INVENTORY : tracks
    PRODUCT ||--o{ CART_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : ordered_in
    
    CART ||--|{ CART_ITEM : has
    
    ORDER ||--|{ ORDER_ITEM : includes
    ORDER ||--|| PAYMENT : settled_by
    ORDER ||--o| SHIPMENT : fulfilled_by
    ORDER ||--o{ NOTIFICATION : triggers
    
    INVENTORY ||--o{ INVENTORY_RESERVATION : reserves

    CUSTOMER {
        bigint customer_id PK
        varchar email UK
        varchar password_hash
        varchar full_name
        timestamp created_at
    }

    PRODUCT {
        bigint product_id PK
        bigint category_id FK
        varchar title
        numeric price
        varchar status
    }

    INVENTORY {
        bigint inventory_id PK
        bigint product_id FK,UK
        int available_quantity
        int reserved_quantity
        int sold_quantity
        bigint version
        timestamp updated_at
    }

    INVENTORY_RESERVATION {
        uuid reservation_id PK
        bigint inventory_id FK
        bigint customer_id FK
        int quantity
        varchar status
        varchar idempotency_key UK
        timestamp expires_at
    }

    ORDER {
        bigint order_id PK
        bigint customer_id FK
        uuid reservation_id FK
        numeric total_amount
        varchar status
        timestamp created_at
    }

    PAYMENT {
        bigint payment_id PK
        bigint order_id FK
        varchar transaction_reference UK
        varchar idempotency_key UK
        numeric amount
        varchar status
        timestamp created_at
    }
```

---

## 2. Comprehensive Table Definitions

<details open>
<summary><b>Core DDL Specifications (Click to expand/collapse)</b></summary>

### 1. `INVENTORY` and `INVENTORY_RESERVATION`
```sql
CREATE TABLE inventory (
    inventory_id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL UNIQUE REFERENCES product(product_id),
    available_quantity INT NOT NULL CHECK (available_quantity >= 0),
    reserved_quantity INT NOT NULL DEFAULT 0 CHECK (reserved_quantity >= 0),
    sold_quantity INT NOT NULL DEFAULT 0 CHECK (sold_quantity >= 0),
    version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE inventory_reservation (
    reservation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id BIGINT NOT NULL REFERENCES inventory(inventory_id),
    customer_id BIGINT NOT NULL REFERENCES customer(customer_id),
    quantity INT NOT NULL DEFAULT 1,
    status VARCHAR(30) NOT NULL DEFAULT 'RESERVED', -- RESERVED, CONFIRMED, RELEASED, EXPIRED
    idempotency_key VARCHAR(128) NOT NULL UNIQUE,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inv_res_expires ON inventory_reservation(expires_at) WHERE status = 'RESERVED';
```

### 2. `PAYMENT` and `ORDER`
```sql
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customer(customer_id),
    reservation_id UUID NOT NULL REFERENCES inventory_reservation(reservation_id),
    total_amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'CREATED', -- CREATED, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE payment (
    payment_id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(order_id),
    transaction_reference VARCHAR(128) UNIQUE,
    idempotency_key VARCHAR(128) NOT NULL UNIQUE,
    payment_method VARCHAR(50) NOT NULL, -- UPI, CREDIT_CARD, WALLET
    amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING', -- PENDING, SUCCESS, FAILED, TIMEOUT
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

</details>

---

## 3. Indexing and Optimization Strategy

| Index Name | Target Table | Indexed Columns | Purpose |
|---|---|---|---|
| `idx_inventory_product` | `inventory` | `product_id` (Unique) | Sub-millisecond lookup during stock checks |
| `idx_res_idempotency` | `inventory_reservation`| `idempotency_key` (Unique) | Deduplicate rapid consecutive click storms |
| `idx_res_expiry_status`| `inventory_reservation`| `(expires_at, status)` | Fast batch polling for expired hold cleanup |
| `idx_payment_idempotency`| `payment` | `idempotency_key` (Unique) | Prevent duplicate credit/bank charges |
| `idx_orders_customer` | `orders` | `(customer_id, created_at DESC)`| Rapid order history retrieval |

---

[Return to System Overview](../README.md) | [Proceed to 05_API ->](../05_API/README.md)