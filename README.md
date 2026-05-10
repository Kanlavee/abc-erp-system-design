# ABC Trading — ERP System Design

> **Full-Stack Interview — System Architecture Document**
> Designed from a Senior System Architect perspective

---

## Table of Contents

1. [Business Overview](#business-overview)
2. [System Objectives](#system-objectives)
3. [Sales Channels](#sales-channels)
4. [Architecture Overview](#architecture-overview)
5. [Part 1: ER Diagram](#part-1-er-diagram)
6. [Part 2: Business Workflow](#part-2-business-workflow)
7. [Key Business Rules](#key-business-rules)
8. [Technology Stack Recommendation](#technology-stack-recommendation)
9. [Scalability Considerations](#scalability-considerations)

---

## Business Overview

**ABC Trading Co., Ltd.** operates wholesale and retail trading of consumer goods (FMCG) across three sales channels:

| Channel | Description | Customer Type |
|---------|-------------|---------------|
| **POS** (Point of Sale) | Walk-in retail store | Retail |
| **Sales Representative** | B2B field sales team | Wholesale |
| **E-commerce** | Online platform (web/app) | Retail / Wholesale |

The company requires a **unified ERP system** to centralize and standardize all business operations across these channels.

---

## System Objectives

| # | Objective | Benefit |
|---|-----------|---------|
| 1 | **Centralize sales data** | Single source of truth across all channels |
| 2 | **Standardize business processes** | Reduce manual errors and inconsistency |
| 3 | **Increase transparency** | Real-time visibility into inventory, orders, and financials |
| 4 | **Support future growth** | Scalable architecture ready for new channels and markets |

---

## Sales Channels

### Channel 1: POS (Point of Sale)
- Walk-in customers purchase products at the physical store
- **Instant fulfillment** — customer leaves with goods immediately
- Payment is collected **at the time of purchase**
- Receipt issued immediately (print or digital)

### Channel 2: Sales Representative (B2B Wholesale)
- Sales reps visit wholesale customers (restaurants, retailers, distributors)
- Process follows **Quotation → Approval → Sales Order → Fulfillment**
- **Credit terms** apply (e.g., Net 30, Net 60 days)
- Tax invoice issued after delivery; payment collected within credit period

### Channel 3: E-commerce (Online)
- Customers browse and order via website or mobile app
- **Payment required before shipment**
- Warehouse picks, packs, and ships after payment confirmation
- Shipment tracked with carrier tracking number

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        ABC Trading ERP                          │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌───────────────────────┐  │
│  │   POS    │   │  Sales Rep   │   │     E-commerce        │  │
│  │ Terminal │   │  Mobile/Web  │   │   Web / Mobile App    │  │
│  └────┬─────┘   └──────┬───────┘   └───────────┬───────────┘  │
│       │                │                        │              │
│  ─────┴────────────────┴────────────────────────┴──────────    │
│               Order Management Layer (unified)                  │
│  ─────────────────────────────────────────────────────────────  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Inventory  │  │  Pricing &   │  │  Accounting (AR/AP)  │  │
│  │   Module    │  │   Catalog    │  │    & Invoicing       │  │
│  └─────────────┘  └──────────────┘  └──────────────────────┘  │
│  ─────────────────────────────────────────────────────────────  │
│                    Shared Database Layer                         │
│     (Products, Customers, Inventory, Orders, Payments, ...)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 1: ER Diagram

See full diagram: [er-diagram.md](./er-diagram.md)

### Entity Map

```
CATEGORIES ──< PRODUCTS >──< INVENTORY >── WAREHOUSES
                |                               |
                |                    INVENTORY_MOVEMENTS
ORDER_ITEMS ────┘
                                 QUOTATIONS (pre-order)
CUSTOMERS ──< ADDRESSES               |
    |               |                 v (on approval)
    |               └──────< ORDERS >─────── SALES_CHANNELS
    |                          |    \─────── SALES_REPS
    └──────────────────────────┘
              |          |           |              |
          PAYMENTS    INVOICES   SHIPMENTS     ORDER_ITEMS
              |
    PAYMENT_TRANSACTIONS
```

### Entity Summary

| Entity | Purpose | Key Attributes |
|--------|---------|----------------|
| `CATEGORIES` | Product grouping | `name` |
| `PRODUCTS` | Master catalog; no stock stored here | `sku` (UK), `base_price`, `category_id` |
| `WAREHOUSES` | Physical storage locations | `location`, `is_active` |
| `INVENTORY` | Stock per warehouse — resolves Products many-to-many Warehouses | `qty_on_hand`, `qty_reserved` (`qty_available` is derived, not stored) |
| `SALES_CHANNELS` | Channel master — POS / SalesRep / Ecommerce | `name`, `is_active` |
| `SALES_REPS` | Field sales reps for B2B orders | `employee_code` (UK), `region` |
| `CUSTOMERS` | All customers across channels | `customer_type` Retail/Wholesale, `credit_limit`, `credit_days` |
| `ADDRESSES` | Multiple addresses per customer | `address_type` billing/shipping, `is_default` |
| `ORDERS` | Unified order record; status = pending / paid / shipped / completed / cancelled | `channel_id`, `sales_rep_id` (nullable), `shipping_addr_id` |
| `ORDER_ITEMS` | Line items; resolves Orders many-to-many Products | `quantity`, `unit_price`, `discount_pct` |
| `QUOTATIONS` | Pre-order document (B2B); independent of ORDERS. On approval, ORDER is created with `quotation_id` FK | `valid_until`, `status` |
| `PAYMENTS` | Transactions per order; supports partial payment | `method` cash/card/transfer/qr, `status` pending/success/failed, `paid_at` |
| `INVOICES` | Tax invoices; tracks AR balance | `invoice_no` (UK), `due_date`, `paid_amount` |
| `SHIPMENTS` | Fulfillment records per dispatch | `tracking_no` (UK), `carrier`, `shipped_at`, `delivered_at` |
| `INVENTORY_MOVEMENTS` | Append-only stock audit log; every reservation, commit, cancel, or adjustment writes here | `change_qty`, `movement_type`, `reference_id` |
| `PAYMENT_TRANSACTIONS` | Append-only gateway attempt log per PAYMENT; every call recorded regardless of outcome | `attempt_no`, `gateway_ref`, `gateway_response`, `status` |

### Key Relationship Rules

- **Products and Warehouses** — Many-to-many via `INVENTORY` (junction table); stock is never stored in PRODUCTS
- **`qty_available`** — Never persisted; always computed as `qty_on_hand - qty_reserved` at query time
- **Customers and Addresses** — One-to-many; ADDRESSES reused by ORDERS (`shipping_addr_id`) and SHIPMENTS
- **Quotations and Orders** — QUOTATION is independent; `ORDERS.quotation_id` (nullable FK) references the source quotation on conversion
- **Orders and Payments** — One-to-many; supports partial payments and retries
- **`sales_rep_id` on ORDERS** — Nullable; set only for B2B orders
- **INVENTORY_MOVEMENTS and PAYMENT_TRANSACTIONS** — Append-only audit tables; never updated, only inserted

---

## Part 2: Business Workflow

See full diagram: [business-workflow.md](./business-workflow.md)

### POS Flow (Instant)

```
Customer scans items
  -> ERP checks stock
  -> Customer pays
  -> Receipt issued
  -> Inventory deducted
  -> Order status: Completed
```

### E-commerce Flow

```
Customer places order online
  -> Stock soft-reserved
  -> Customer pays online
  -> Warehouse: Pick, Pack, Ship
  -> Tax invoice issued
  -> Order status: Shipped
```

### Sales Rep / B2B Flow (Quotation-based)

```
Customer submits RFQ
  -> ERP checks stock and lead time
  -> Quotation sent to customer
  -> Customer approves -> Sales Order created
  -> Inventory reserved
  -> Warehouse fulfillment
  -> Tax invoice issued (credit terms)
  -> Customer pays within credit period
  -> AR settled, Order status: Completed
```

---

## Key Business Rules

### Inventory Management
- `qty_available = qty_on_hand - qty_reserved`
- Stock is **soft-reserved** when an order or quotation is confirmed
- Stock is **committed** (deducted from `qty_on_hand`) only when goods are shipped
- Reservation is **released** automatically if an order is cancelled

### Pricing Rules
- `base_price` in PRODUCTS is the standard retail price
- Wholesale customers receive negotiated prices via `order_items.discount_pct`
- Sales Reps can apply pre-approved discount tiers per customer segment

### Credit and Payment Terms
- **Retail customers**: Payment required immediately (POS / E-commerce)
- **Wholesale customers**: Credit terms defined by `customers.credit_days` (e.g., 30, 60 days)
- AR balance tracked via `invoices.total_amount` vs `invoices.paid_amount`

### Order Status Lifecycle
```
pending -> paid -> shipped -> completed
       \-> cancelled (before shipped)
```

### Payment Method Values
| Method | Channel |
|--------|---------|
| `cash` | POS |
| `credit_card` | POS / E-commerce |
| `qr` | POS / E-commerce |
| `bank_transfer` | B2B / E-commerce |

---

## Data Consistency and Concurrency Strategy

### Inventory Concurrency Control
- **Row-level locking** (`SELECT ... FOR UPDATE`) on `INVENTORY` rows during reservation prevents concurrent oversell
- `qty_available` is **never persisted** — computed as `qty_on_hand - qty_reserved` at read time to avoid stale state
- All reservation and commit operations run inside an **atomic database transaction**
- **DB-level constraints** prevent negative stock:

```sql
ALTER TABLE inventory ADD CONSTRAINT chk_qty_non_negative  CHECK (qty_on_hand >= 0);
ALTER TABLE inventory ADD CONSTRAINT chk_reserved_lte_hand CHECK (qty_reserved <= qty_on_hand);
```

### Payment Idempotency
- Every gateway request carries an **idempotency key** (derived from `payment_id + attempt_no`) to prevent double charges on retry
- `PAYMENT_TRANSACTIONS` records every call independently — `PAYMENTS` is never mutated between retries
- Payment status transitions are one-directional: `pending → success` or `pending → failed`

### Locking Strategy by Scenario
| Scenario | Strategy | Reason |
|----------|----------|--------|
| Inventory reservation | Pessimistic (row lock) | High contention — must prevent oversell |
| Order status update | Optimistic (version field) | Low contention — avoids unnecessary lock overhead |
| Payment attempt | No lock — new row per attempt | Retries are new PAYMENT_TRANSACTIONS records |

---

## Failure Handling and Resilience

### Payment Failure Strategy
| Scenario | Behavior |
|----------|----------|
| Gateway timeout | Retry with same idempotency key using exponential backoff |
| Card declined | Notify customer, keep order `pending`, prompt retry |
| Max retries exceeded | Order stays `pending`, alert customer and support team |
| Double-charge risk | Idempotency key on gateway prevents re-charge on duplicate call |

### Shipment Failure — Saga Compensation
If shipment creation fails **after** inventory is committed:
1. **Compensate** — restore `qty_on_hand` via a `cancel_commit` record in `INVENTORY_MOVEMENTS`
2. **Hold** — set `INVOICES.status` to `on_hold`
3. **Retry** — schedule re-attempt with exponential backoff (max 3 attempts)
4. **Escalate** — if all retries fail, alert operations team for manual intervention

### Saga Step Map
| Step | Action | Compensation on Failure |
|------|--------|------------------------|
| 1 | Reserve inventory | Release reservation (cancel movement) |
| 2 | Record payment intent | Mark payment as failed |
| 3 | Commit inventory | Restore qty_on_hand (cancel_commit movement) |
| 4 | Create shipment | Cancel shipment record |
| 5 | Issue invoice | Void invoice |

---

## Audit and Traceability

### INVENTORY_MOVEMENTS — Stock Audit Log
Every stock change writes an **immutable** record:

| movement_type | Trigger | Effect |
|---------------|---------|--------|
| `order_reserve` | Order confirmed | `qty_reserved += n` |
| `order_commit` | Shipment dispatched | `qty_on_hand -= n`, `qty_reserved -= n` |
| `cancel` | Order/quotation cancelled | `qty_reserved -= n` |
| `cancel_commit` | Shipment failed (compensation) | `qty_on_hand += n` |
| `adjustment` | Manual stock correction | `qty_on_hand +/- n` |
| `return` | Customer return processed | `qty_on_hand += n` |

Use cases: stock reconciliation, shrinkage detection, regulatory compliance, warehouse audits.

### PAYMENT_TRANSACTIONS — Financial Trace
Every gateway call is logged regardless of outcome:
- `gateway_response` stores the raw payload for dispute resolution
- `attempt_no` + `gateway_ref` verify idempotency across retries
- Traceability chain: `PAYMENT_TRANSACTIONS → PAYMENTS → INVOICES → ORDERS`

### Full Audit Chain
```
ORDERS
  ├── ORDER_ITEMS              -> PRODUCTS (what was sold)
  ├── PAYMENTS
  │     └── PAYMENT_TRANSACTIONS  (every gateway attempt with full response)
  ├── INVOICES                 (AR balance: total_amount vs paid_amount)
  ├── SHIPMENTS                (dispatch trail per warehouse)
  └── INVENTORY_MOVEMENTS      (stock change log linked via reference_id = order_id)
```

---

## Technology Stack Recommendation

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend** | React + TypeScript | Component-based, type-safe, scalable UI |
| **Backend API** | Node.js (NestJS) or Go (Gin) | High throughput, microservice-ready |
| **Database** | PostgreSQL | Relational integrity, ACID compliance, JSON support |
| **Cache** | Redis | Real-time inventory reservation, session management |
| **Message Queue** | RabbitMQ / Kafka | Async order events, inventory sync across services |
| **Search** | Elasticsearch | Product catalog search and reporting |
| **File Storage** | AWS S3 / GCS | Invoice PDFs, delivery documents |
| **Deployment** | Docker + Kubernetes | Scalable, cloud-agnostic, zero-downtime deploys |

---

## Scalability Considerations

### Short-term
- Index on `orders(channel_id, status, order_date)` and `inventory(product_id, warehouse_id)`
- Connection pooling via PgBouncer for PostgreSQL under high concurrency

### Medium-term
- Read replicas for reporting queries (sales dashboards, inventory reports)
- Redis caching for product catalog and frequently-read inventory levels
- Async processing for invoice generation, email/SMS notifications

### Long-term
- **CQRS pattern** — separate read and write models for orders and inventory
- **Event Sourcing** — full audit trail of every inventory and order state change
- **Multi-warehouse routing** — intelligent fulfillment from nearest or optimal warehouse
- **Multi-currency / multi-branch** support for regional expansion

---

## Project File Structure

```
abc-erp-system-design/
├── README.md             <- System overview (this file)
├── er-diagram.md         <- Part 1: Mermaid ER Diagram + entity descriptions
└── business-workflow.md  <- Part 2: Mermaid Sequence Diagram + flow summary
```

---

*Designed for ABC Trading ERP System — Full-Stack Interview Assessment*
