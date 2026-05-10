# Part 1: ER Diagram — ABC Trading ERP System

## Entity Relationship Diagram

```mermaid
erDiagram
    CATEGORIES {
        int      category_id   PK
        varchar  name
        varchar  description
        datetime created_at
    }

    PRODUCTS {
        int      product_id    PK
        int      category_id   FK
        varchar  sku           UK
        varchar  name
        text     description
        decimal  base_price
        varchar  unit
        boolean  is_active
        datetime created_at
        datetime updated_at
    }

    WAREHOUSES {
        int      warehouse_id    PK
        varchar  name
        varchar  location
        varchar  contact_person
        boolean  is_active
        datetime created_at
    }

    %% qty_available is DERIVED: qty_on_hand - qty_reserved (computed at read time, never persisted)
    INVENTORY {
        int      inventory_id   PK
        int      product_id     FK
        int      warehouse_id   FK
        int      qty_on_hand
        int      qty_reserved
        int      reorder_level
        datetime last_updated
    }

    SALES_CHANNELS {
        int      channel_id   PK
        varchar  name
        varchar  description
        boolean  is_active
        datetime created_at
    }

    SALES_REPS {
        int      sales_rep_id   PK
        varchar  employee_code  UK
        varchar  full_name
        varchar  email
        varchar  phone
        varchar  region
        boolean  is_active
        datetime created_at
    }

    CUSTOMERS {
        int      customer_id    PK
        varchar  name
        varchar  email          UK
        varchar  phone
        varchar  tax_id
        varchar  customer_type
        decimal  credit_limit
        int      credit_days
        boolean  is_active
        datetime created_at
        datetime updated_at
    }

    ADDRESSES {
        int      address_id    PK
        int      customer_id   FK
        varchar  address_type
        varchar  line1
        varchar  line2
        varchar  city
        varchar  province
        varchar  postal_code
        varchar  country
        boolean  is_default
        datetime created_at
    }

    ORDERS {
        int      order_id          PK
        int      customer_id       FK
        int      channel_id        FK
        int      sales_rep_id      FK
        int      shipping_addr_id  FK
        int      quotation_id      FK
        datetime order_date
        varchar  status
        decimal  subtotal
        decimal  tax_amount
        decimal  discount_amount
        decimal  shipping_amount
        decimal  total_amount
        varchar  remarks
        varchar  created_by
        datetime created_at
        datetime updated_at
    }

    ORDER_ITEMS {
        int      order_item_id  PK
        int      order_id       FK
        int      product_id     FK
        int      quantity
        decimal  unit_price
        decimal  discount_pct
        decimal  subtotal
        datetime created_at
    }

    %% QUOTATION is a pre-order document; on approval a new ORDER is created referencing it via quotation_id
    QUOTATIONS {
        int      quotation_id  PK
        int      customer_id   FK
        int      sales_rep_id  FK
        datetime valid_until
        varchar  status
        varchar  remarks
        datetime created_at
        datetime updated_at
    }

    QUOTATION_ITEMS {
        int      quotation_item_id  PK
        int      quotation_id       FK
        int      product_id         FK
        int      quantity
        decimal  unit_price
        decimal  discount_pct
        decimal  subtotal
        datetime created_at
    }

    PAYMENTS {
        int      payment_id    PK
        int      order_id      FK
        decimal  amount
        varchar  method
        varchar  status
        varchar  reference_no
        datetime paid_at
        varchar  remarks
        datetime created_at
    }

    INVOICES {
        int      invoice_id    PK
        int      order_id      FK
        varchar  invoice_no    UK
        datetime invoice_date
        datetime due_date
        decimal  total_amount
        decimal  paid_amount
        varchar  status
        datetime created_at
        datetime updated_at
    }

    SHIPMENTS {
        int      shipment_id   PK
        int      order_id      FK
        int      warehouse_id  FK
        int      address_id    FK
        varchar  tracking_no   UK
        varchar  carrier
        varchar  status
        datetime shipped_at
        datetime delivered_at
        datetime created_at
    }

    %% Append-only audit / event-log tables (INSERT only, never updated)
    INVENTORY_MOVEMENTS {
        int      movement_id    PK
        int      product_id     FK
        int      warehouse_id   FK
        int      change_qty
        varchar  movement_type
        int      reference_id
        varchar  reference_type
        varchar  created_by
        datetime created_at
    }

    PAYMENT_TRANSACTIONS {
        int      transaction_id    PK
        int      payment_id        FK
        int      attempt_no
        varchar  gateway_ref
        varchar  gateway_response
        varchar  status
        datetime created_at
    }

    %% Product catalog and inventory
    CATEGORIES     ||--o{ PRODUCTS      : "categorizes"
    PRODUCTS       ||--o{ INVENTORY     : "tracked in"
    WAREHOUSES     ||--o{ INVENTORY     : "stores"

    %% Customer and address
    CUSTOMERS      ||--o{ ADDRESSES     : "has"
    CUSTOMERS      ||--o{ ORDERS        : "places"
    CUSTOMERS      ||--o{ QUOTATIONS    : "requests"

    %% Order core relations
    SALES_CHANNELS ||--o{ ORDERS        : "channel for"
    SALES_REPS     ||--o{ ORDERS        : "manages"
    ADDRESSES      ||--o{ ORDERS        : "ships to"
    ORDERS         ||--o{ ORDER_ITEMS   : "contains"
    PRODUCTS       ||--o{ ORDER_ITEMS   : "included in"

    %% B2B quotation flow - QUOTATION precedes ORDER; ORDERS.quotation_id (nullable) references source
    SALES_REPS      ||--o{ QUOTATIONS       : "creates"
    QUOTATIONS      ||--o{ QUOTATION_ITEMS  : "contains"
    PRODUCTS        ||--o{ QUOTATION_ITEMS  : "included in"
    QUOTATIONS      ||--o| ORDERS           : "converts to"

    %% Financial records
    ORDERS         ||--o{ PAYMENTS      : "paid via"
    ORDERS         ||--o| INVOICES      : "invoiced as"

    %% Fulfillment
    ORDERS         ||--o{ SHIPMENTS     : "fulfilled via"
    WAREHOUSES     ||--o{ SHIPMENTS     : "dispatched from"
    ADDRESSES      ||--o{ SHIPMENTS     : "delivered to"

    %% Audit logs - append-only; never updated; full traceability
    PRODUCTS       ||--o{ INVENTORY_MOVEMENTS  : "logged in"
    WAREHOUSES     ||--o{ INVENTORY_MOVEMENTS  : "tracked in"
    PAYMENTS       ||--o{ PAYMENT_TRANSACTIONS : "attempted via"
```

---

## Entity Descriptions

| Entity | Key Attributes | Description |
|--------|---------------|-------------|
| **CATEGORIES** | `category_id`, `name` | Product groupings (e.g., Beverages, Snacks) |
| **PRODUCTS** | `sku` (UK), `base_price`, `category_id` | Master product catalog; no stock stored here |
| **WAREHOUSES** | `warehouse_id`, `location` | Physical storage locations |
| **INVENTORY** | `qty_on_hand`, `qty_reserved` | Junction table (Products ↔ Warehouses); tracks live stock. `qty_available` is **derived** (`qty_on_hand - qty_reserved`), never persisted |
| **SALES_CHANNELS** | `name` — POS / SalesRep / Ecommerce | Channel master; extensible for future channels |
| **SALES_REPS** | `employee_code` (UK), `region` | Field sales representatives for B2B orders |
| **CUSTOMERS** | `customer_type` — Retail / Wholesale, `credit_limit`, `credit_days` | All customer accounts across channels |
| **ADDRESSES** | `address_type` — billing / shipping, `is_default` | Multiple addresses per customer; used for both ORDER and SHIPMENT |
| **ORDERS** | `channel_id`, `sales_rep_id` (nullable), `quotation_id` (nullable), `shipping_amount` | Unified order record; `total_amount = subtotal - discount_amount + tax_amount + shipping_amount` |
| **ORDER_ITEMS** | `quantity`, `unit_price`, `discount_pct` | Line items resolving many-to-many between ORDERS and PRODUCTS |
| **QUOTATIONS** | `customer_id`, `sales_rep_id`, `valid_until`, `status` | Pre-order document with its own line items (QUOTATION_ITEMS). On approval, a new ORDER is created referencing `quotation_id`; ORDER_ITEMS are populated from QUOTATION_ITEMS |
| **QUOTATION_ITEMS** | `quotation_id`, `product_id`, `quantity`, `unit_price`, `discount_pct` | Line items within a quotation; mirrors ORDER_ITEMS structure; enables pre-sales pricing negotiation per product |
| **PAYMENTS** | `method` — cash / credit_card / transfer / qr, `status` — pending / success / failed, `paid_at` | Multiple payments per order (supports partial / split payment) |
| **INVOICES** | `invoice_no` (UK), `due_date`, `paid_amount` | Tax invoices; tracks Accounts Receivable balance |
| **SHIPMENTS** | `tracking_no` (UK), `carrier`, `shipped_at`, `delivered_at` | Fulfillment records per warehouse dispatch |
| **INVENTORY_MOVEMENTS** | `change_qty`, `movement_type`, `reference_id`, `reference_type` | Append-only stock audit log. `movement_type` values: `order_reserve`, `order_commit`, `cancel`, `cancel_commit`, `adjustment`, `return` |
| **PAYMENT_TRANSACTIONS** | `attempt_no`, `gateway_ref`, `gateway_response`, `status` | Append-only log of every gateway call per PAYMENT; enables retry tracking, idempotency verification, and dispute resolution |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **INVENTORY as junction table** | Many-to-many between PRODUCTS and WAREHOUSES; stock never stored in PRODUCTS |
| **`qty_available` not persisted** | Computed as `qty_on_hand - qty_reserved` at query time; persisting it risks stale reads under concurrent writes |
| **INVENTORY_MOVEMENTS as append-only log** | Separates current state (INVENTORY) from history (MOVEMENTS); enables fast reads, full audit trail, and stock reconciliation without touching INVENTORY |
| **PAYMENT_TRANSACTIONS separate from PAYMENTS** | PAYMENTS = logical payment intent; PAYMENT_TRANSACTIONS = physical gateway attempt. Retries add a new row rather than mutating the payment record — supports idempotency and dispute resolution |
| **QUOTATIONS independent of ORDERS** | QUOTATION is a pre-order document owned by CUSTOMERS; ORDERS.quotation_id (nullable FK) references the source. Eliminates circular dependency and makes the lifecycle unambiguous |
| **QUOTATION_ITEMS mirrors ORDER_ITEMS** | Quotations need their own line items for pre-sales price negotiation; on approval, ORDER_ITEMS are created by copying QUOTATION_ITEMS with agreed pricing — no data duplication |
| **`shipping_amount` separate from subtotal** | Shipping cost is a distinct charge (varies by address, carrier, weight); kept separate for financial transparency and reporting |
| **ADDRESSES as separate entity** | Customers have multiple billing/shipping addresses; reused by ORDERS (shipping_addr_id) and SHIPMENTS (address_id) without data duplication |
| **`sales_rep_id` nullable on ORDERS** | Only B2B orders have an assigned rep; POS and e-commerce orders set this to NULL |
| **`status` as varchar not enum** | Allows adding statuses (e.g., `on_hold`, `backordered`) without a schema migration |
| **`created_at` on every entity** | Immutable event timestamp for ordering and audit; `updated_at` on mutable entities enables cache invalidation |
| **FK constraints everywhere** | Enforced at DB layer to guarantee referential integrity; no orphaned records possible |
