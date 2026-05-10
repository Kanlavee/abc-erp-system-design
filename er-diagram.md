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

    INVENTORY {
        int      inventory_id   PK
        int      product_id     FK
        int      warehouse_id   FK
        int      qty_on_hand
        int      qty_reserved
        int      qty_available
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
        datetime order_date
        varchar  status
        decimal  subtotal
        decimal  tax_amount
        decimal  discount_amount
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

    QUOTATIONS {
        int      quotation_id  PK
        int      order_id      FK
        int      sales_rep_id  FK
        datetime valid_until
        varchar  status
        varchar  remarks
        datetime created_at
        datetime updated_at
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

    %% Product catalog and inventory
    CATEGORIES     ||--o{ PRODUCTS      : "categorizes"
    PRODUCTS       ||--o{ INVENTORY     : "tracked in"
    WAREHOUSES     ||--o{ INVENTORY     : "stores"

    %% Customer and address
    CUSTOMERS      ||--o{ ADDRESSES     : "has"
    CUSTOMERS      ||--o{ ORDERS        : "places"

    %% Order core relations
    SALES_CHANNELS ||--o{ ORDERS        : "channel for"
    SALES_REPS     ||--o{ ORDERS        : "manages"
    ADDRESSES      ||--o{ ORDERS        : "ships to"
    ORDERS         ||--o{ ORDER_ITEMS   : "contains"
    PRODUCTS       ||--o{ ORDER_ITEMS   : "included in"

    %% B2B quotation flow
    ORDERS         ||--o| QUOTATIONS    : "quoted as"
    SALES_REPS     ||--o{ QUOTATIONS    : "creates"

    %% Financial records
    ORDERS         ||--o{ PAYMENTS      : "paid via"
    ORDERS         ||--o| INVOICES      : "invoiced as"

    %% Fulfillment
    ORDERS         ||--o{ SHIPMENTS     : "fulfilled via"
    WAREHOUSES     ||--o{ SHIPMENTS     : "dispatched from"
    ADDRESSES      ||--o{ SHIPMENTS     : "delivered to"
```

---

## Entity Descriptions

| Entity | Key Attributes | Description |
|--------|---------------|-------------|
| **CATEGORIES** | `category_id`, `name` | Product groupings (e.g., Beverages, Snacks) |
| **PRODUCTS** | `sku` (UK), `base_price`, `category_id` | Master product catalog; no stock stored here |
| **WAREHOUSES** | `warehouse_id`, `location` | Physical storage locations |
| **INVENTORY** | `qty_on_hand`, `qty_reserved`, `qty_available` | Junction table (Products ↔ Warehouses); tracks live stock per location |
| **SALES_CHANNELS** | `name` — POS / SalesRep / Ecommerce | Channel master; extensible for future channels |
| **SALES_REPS** | `employee_code` (UK), `region` | Field sales representatives for B2B orders |
| **CUSTOMERS** | `customer_type` — Retail / Wholesale, `credit_limit`, `credit_days` | All customer accounts across channels |
| **ADDRESSES** | `address_type` — billing / shipping, `is_default` | Multiple addresses per customer; used for both ORDER and SHIPMENT |
| **ORDERS** | `channel_id`, `sales_rep_id` (nullable), `status`, `shipping_addr_id` | Unified order record; `status` = pending / paid / shipped / completed / cancelled |
| **ORDER_ITEMS** | `quantity`, `unit_price`, `discount_pct` | Line items resolving many-to-many between ORDERS and PRODUCTS |
| **QUOTATIONS** | `valid_until`, `status` — draft / sent / approved / expired | B2B only; converts to confirmed ORDER on approval |
| **PAYMENTS** | `method` — cash / credit_card / transfer / qr, `status` — pending / success / failed, `paid_at` | Multiple payments per order (supports partial / split payment) |
| **INVOICES** | `invoice_no` (UK), `due_date`, `paid_amount` | Tax invoices; tracks Accounts Receivable balance |
| **SHIPMENTS** | `tracking_no` (UK), `carrier`, `shipped_at`, `delivered_at` | Fulfillment records per warehouse dispatch |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **INVENTORY as junction table** | Many-to-many between PRODUCTS and WAREHOUSES; stock never lives in PRODUCTS |
| **ADDRESSES as separate entity** | Customers can have multiple billing/shipping addresses; reused by both ORDERS and SHIPMENTS |
| **SALES_REPS linked to ORDERS** | `sales_rep_id` is nullable — only B2B orders have a rep; POS and e-commerce orders do not |
| **PAYMENTS supports multiple records per order** | Allows partial payments, installments, and retries on failure |
| **QUOTATIONS linked to ORDERS** | Quotation is a sub-state of an order (only in B2B); avoids duplicate order data |
| **`qty_available` computed field** | `qty_available = qty_on_hand - qty_reserved`; maintained on every reservation/release event |
| **`status` as varchar not enum** | Allows adding new statuses (e.g., `on_hold`, `backordered`) without schema migration |
| **Timestamps on every entity** | `created_at` and `updated_at` enable full audit trail and change tracking |
