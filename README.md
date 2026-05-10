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
                                                |
CUSTOMERS ──< ADDRESSES                    SHIPMENTS
    |               |                          |
    └──────< ORDERS >──────────────────────────┘
               |    \
          SALES_CHANNELS    SALES_REPS
               |
    ┌──────────┼───────────┐
ORDER_ITEMS  QUOTATIONS  PAYMENTS  INVOICES
```

### Entity Summary

| Entity | Purpose | Key Attributes |
|--------|---------|----------------|
| `CATEGORIES` | Product grouping | `name` |
| `PRODUCTS` | Master catalog; no stock stored here | `sku` (UK), `base_price`, `category_id` |
| `WAREHOUSES` | Physical storage locations | `location`, `is_active` |
| `INVENTORY` | Stock per warehouse — resolves Products many-to-many Warehouses | `qty_on_hand`, `qty_reserved`, `qty_available` |
| `SALES_CHANNELS` | Channel master — POS / SalesRep / Ecommerce | `name`, `is_active` |
| `SALES_REPS` | Field sales reps for B2B orders | `employee_code` (UK), `region` |
| `CUSTOMERS` | All customers across channels | `customer_type` Retail/Wholesale, `credit_limit`, `credit_days` |
| `ADDRESSES` | Multiple addresses per customer | `address_type` billing/shipping, `is_default` |
| `ORDERS` | Unified order record; status = pending / paid / shipped / completed / cancelled | `channel_id`, `sales_rep_id` (nullable), `shipping_addr_id` |
| `ORDER_ITEMS` | Line items; resolves Orders many-to-many Products | `quantity`, `unit_price`, `discount_pct` |
| `QUOTATIONS` | B2B only; converts to Sales Order on approval | `valid_until`, `status` |
| `PAYMENTS` | Transactions per order; supports partial payment | `method` cash/card/transfer/qr, `status` pending/success/failed, `paid_at` |
| `INVOICES` | Tax invoices; tracks AR balance | `invoice_no` (UK), `due_date`, `paid_amount` |
| `SHIPMENTS` | Fulfillment records per dispatch | `tracking_no` (UK), `carrier`, `shipped_at`, `delivered_at` |

### Key Relationship Rules

- **Products and Warehouses** — Many-to-many via `INVENTORY` (junction table); stock is never stored in PRODUCTS
- **Customers and Addresses** — One-to-many; ADDRESSES is reused by both ORDERS and SHIPMENTS
- **Orders and Sales Reps** — Optional FK; only B2B orders have a `sales_rep_id`
- **Orders and Payments** — One-to-many; supports partial payments and retries
- **`customer_type`** drives pricing tier and credit term logic

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
