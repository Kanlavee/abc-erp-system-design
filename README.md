# ABC Trading — ERP System Design

> **Full-Stack Interview — System Architecture Document**  
> Designed by a Senior System Architect perspective

---

## 📋 Table of Contents

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

### 🏪 Channel 1: POS (Point of Sale)
- Walk-in customers purchase products at the physical store
- **Instant fulfillment** — customer leaves with goods immediately
- Payment is collected **at the time of purchase**
- Receipt issued immediately (print or digital)

### 🤝 Channel 2: Sales Representative (B2B Wholesale)
- Sales reps visit wholesale customers (restaurants, retailers, distributors)
- Process follows **Quotation → Approval → Sales Order → Fulfillment**
- **Credit terms** apply (e.g., Net 30, Net 60 days)
- Tax invoice issued after delivery; payment collected within credit period

### 🛒 Channel 3: E-commerce (Online)
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
│  │   POS    │   │ Sales Rep    │   │     E-commerce        │  │
│  │ Terminal │   │ Mobile/Web   │   │   Web / Mobile App    │  │
│  └────┬─────┘   └──────┬───────┘   └───────────┬───────────┘  │
│       │                │                        │               │
│  ─────┴────────────────┴────────────────────────┴──────────    │
│                    Order Management Layer                        │
│  ─────────────────────────────────────────────────────────────  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Inventory  │  │   Pricing &  │  │   Accounting (AR/AP) │  │
│  │   Module    │  │   Catalog    │  │     & Invoicing      │  │
│  └─────────────┘  └──────────────┘  └──────────────────────┘  │
│  ─────────────────────────────────────────────────────────────  │
│                    Shared Database Layer                         │
│         (Products, Customers, Inventory, Orders, Payments)      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 1: ER Diagram

📄 See full diagram: [`er-diagram.md`](./er-diagram.md)

### Core Entities at a Glance

```
CATEGORIES ──< PRODUCTS >──< INVENTORIES >── WAREHOUSES
                    │
                    └──< ORDER_ITEMS >──< ORDERS >── CUSTOMERS
                                              │           │
                                         QUOTATIONS  SALES_CHANNELS
                                              │
                               ┌─────────────┼─────────────┐
                           PAYMENTS      INVOICES       SHIPMENTS
```

### Entity Summary

| Entity | Purpose | Key Attributes |
|--------|---------|----------------|
| `PRODUCTS` | Product master catalog | `SKU`, `BasePrice`, `CategoryID` |
| `CATEGORIES` | Product grouping | `CategoryName` |
| `WAREHOUSES` | Storage locations | `Location`, `IsActive` |
| `INVENTORIES` | Stock levels per warehouse (many-to-many junction) | `QuantityOnHand`, `QuantityReserved` |
| `CUSTOMERS` | All customer records | `CustomerType` (Retail/Wholesale), `CreditLimit` |
| `SALES_CHANNELS` | Channel master (POS/SalesRep/Ecommerce) | `ChannelName` |
| `ORDERS` | Master order across all channels | `ChannelID`, `Status`, `TotalAmount` |
| `ORDER_ITEMS` | Line items in an order | `Quantity`, `UnitPrice`, `Discount` |
| `QUOTATIONS` | B2B quotation before SO conversion | `ValidUntil`, `Status` |
| `PAYMENTS` | Payment transactions | `PaymentMethod`, `Status` |
| `INVOICES` | Tax invoices and AR tracking | `DueDate`, `PaidAmount` |
| `SHIPMENTS` | Fulfillment and delivery records | `TrackingNo`, `Carrier` |

### Key Relationship Rules

- **Products ↔ Warehouses** → Many-to-Many via `INVENTORIES` (junction table)
- **Orders → OrderItems** → One-to-Many (an order has multiple line items)
- **Orders → Payments** → One-to-Many (support partial payments / installments)
- **CUSTOMERS.CustomerType** drives pricing tier and credit term logic
- **ORDERS.ChannelID** is the unified channel identifier for cross-channel reporting

---

## Part 2: Business Workflow

📄 See full diagram: [`business-workflow.md`](./business-workflow.md)

### POS Flow (Instant)

```
Customer → Scan Items → ERP checks stock → Payment → 
Receipt issued → Inventory deducted → Done ✅
```

### E-commerce Flow

```
Customer → Place Order → Stock Reserved → Payment Link →
Payment Confirmed → Warehouse Pick & Pack → Shipped 🚚 →
Tax Invoice issued
```

### Sales Rep / B2B Flow (Quotation-based)

```
Customer RFQ → ERP checks stock → Quotation sent →
Customer Approves → Sales Order created → Stock Reserved →
Warehouse Fulfillment → Delivery → Tax Invoice (credit terms) →
Payment within credit period → AR Settled ✅
```

---

## Key Business Rules

### Inventory Management
- `QuantityAvailable = QuantityOnHand − QuantityReserved`
- Stock is **soft-reserved** when an order/quotation is confirmed
- Stock is **hard-committed** (deducted) only when goods are shipped/handed over
- Reservation is **released** if an order is cancelled

### Pricing Rules
- `BasePrice` in PRODUCTS is the standard retail price
- Wholesale customers may receive negotiated prices (percentage discount stored in `ORDER_ITEMS.DiscountPct`)
- POS applies retail price; Sales Rep can apply approved discounts

### Credit & Payment Terms
- **Retail customers**: Payment required immediately (POS/E-commerce)
- **Wholesale customers**: Credit terms defined by `CUSTOMERS.CreditDays` (e.g., 30, 60 days)
- Outstanding invoices tracked in `INVOICES.PaidAmount` vs `TotalAmount`

### Order Status Lifecycle
```
Draft → Confirmed → Processing → Shipped → Delivered → Closed
                  ↘ Cancelled
```

---

## Technology Stack Recommendation

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend** | React + TypeScript | Component-based, type-safe, scalable |
| **Backend API** | Node.js (NestJS) or Go | High throughput, microservice-ready |
| **Database** | PostgreSQL | Relational integrity, ACID compliance |
| **Cache** | Redis | Real-time inventory reservation, session management |
| **Message Queue** | RabbitMQ / Kafka | Async order events, inventory sync |
| **Search** | Elasticsearch | Product catalog search |
| **Storage** | AWS S3 / GCS | Invoice PDFs, delivery documents |
| **Deployment** | Docker + Kubernetes | Scalable, cloud-agnostic |

---

## Scalability Considerations

### Short-term
- **Database indexing** on `Orders(ChannelID, Status, OrderDate)` and `Inventories(ProductID, WarehouseID)`
- **Connection pooling** (PgBouncer) for PostgreSQL under high load

### Medium-term
- **Read replicas** for reporting queries (sales dashboards, inventory reports)
- **Redis caching** for product catalog and frequently-read inventory levels
- **Async processing** for invoice generation and notification delivery

### Long-term
- **CQRS pattern** — separate read and write models for orders and inventory
- **Event Sourcing** — full audit trail of every inventory and order state change
- **Multi-warehouse routing** — intelligent fulfillment from nearest/optimal warehouse
- **Multi-currency / multi-branch** support for regional expansion

---

## Project File Structure

```
abc-erp-system-design/
├── README.md                  ← This document (system overview)
├── er-diagram.md              ← Part 1: Mermaid ER Diagram + entity descriptions
└── business-workflow.md       ← Part 2: Mermaid Sequence Diagram + flow summary
```

---

*Designed for ABC Trading ERP System — Full-Stack Interview Assessment*
# abc-erp-system-design-
