# Part 1: ER Diagram — ABC Trading ERP System

## Entity Relationship Diagram

```mermaid
erDiagram
    CATEGORIES {
        int     CategoryID   PK
        string  CategoryName
        string  Description
    }

    PRODUCTS {
        int     ProductID    PK
        string  SKU
        string  ProductName
        string  Description
        decimal BasePrice
        int     CategoryID   FK
        string  Unit
        boolean IsActive
    }

    WAREHOUSES {
        int     WarehouseID   PK
        string  WarehouseName
        string  Location
        string  ContactPerson
        boolean IsActive
    }

    INVENTORIES {
        int      InventoryID       PK
        int      ProductID         FK
        int      WarehouseID       FK
        int      QuantityOnHand
        int      QuantityReserved
        int      QuantityAvailable
        int      ReorderLevel
        datetime LastUpdated
    }

    CUSTOMERS {
        int     CustomerID    PK
        string  CustomerName
        enum    CustomerType
        string  Email
        string  Phone
        string  Address
        string  TaxID
        decimal CreditLimit
        int     CreditDays
        boolean IsActive
    }

    SALES_CHANNELS {
        int    ChannelID    PK
        string ChannelName
        string Description
    }

    ORDERS {
        int      OrderID         PK
        int      CustomerID      FK
        int      ChannelID       FK
        datetime OrderDate
        enum     Status
        decimal  SubTotal
        decimal  TaxAmount
        decimal  DiscountAmount
        decimal  TotalAmount
        string   ShippingAddress
        string   Remarks
        string   CreatedBy
    }

    ORDER_ITEMS {
        int     OrderItemID PK
        int     OrderID     FK
        int     ProductID   FK
        int     Quantity
        decimal UnitPrice
        decimal DiscountPct
        decimal SubTotal
    }

    QUOTATIONS {
        int      QuotationID  PK
        int      OrderID      FK
        datetime ValidUntil
        enum     Status
        string   Remarks
    }

    PAYMENTS {
        int      PaymentID     PK
        int      OrderID       FK
        datetime PaymentDate
        decimal  Amount
        enum     PaymentMethod
        enum     Status
        string   ReferenceNo
        string   Remarks
    }

    INVOICES {
        int      InvoiceID    PK
        int      OrderID      FK
        datetime InvoiceDate
        datetime DueDate
        decimal  TotalAmount
        decimal  PaidAmount
        enum     Status
    }

    SHIPMENTS {
        int      ShipmentID   PK
        int      OrderID      FK
        int      WarehouseID  FK
        datetime ShipmentDate
        datetime DeliveryDate
        string   TrackingNo
        string   Carrier
        enum     Status
    }

    %% ── Relationships ──────────────────────────────────────────────────────────

    CATEGORIES    ||--o{ PRODUCTS       : "categorizes"
    PRODUCTS      ||--o{ INVENTORIES    : "stocked in"
    WAREHOUSES    ||--o{ INVENTORIES    : "stores"
    CUSTOMERS     ||--o{ ORDERS         : "places"
    SALES_CHANNELS||--o{ ORDERS         : "processed via"
    ORDERS        ||--o{ ORDER_ITEMS    : "contains"
    PRODUCTS      ||--o{ ORDER_ITEMS    : "included in"
    ORDERS        ||--o| QUOTATIONS     : "may have"
    ORDERS        ||--o{ PAYMENTS       : "paid via"
    ORDERS        ||--o| INVOICES       : "generates"
    ORDERS        ||--o{ SHIPMENTS      : "fulfilled via"
    WAREHOUSES    ||--o{ SHIPMENTS      : "dispatched from"
```

---

## Entity Descriptions

| Entity | Description |
|---|---|
| **CATEGORIES** | Product category/group (e.g., Beverages, Snacks) |
| **PRODUCTS** | Master product catalog with SKU and base price |
| **WAREHOUSES** | Physical storage locations |
| **INVENTORIES** | Junction table bridging Products ↔ Warehouses (many-to-many); tracks real-time stock levels |
| **CUSTOMERS** | Both Retail (POS/E-commerce) and Wholesale (B2B) customers |
| **SALES_CHANNELS** | `POS`, `SalesRep`, `Ecommerce` |
| **ORDERS** | Master order record across all channels; `Status` = Draft / Confirmed / Processing / Shipped / Delivered / Cancelled |
| **ORDER_ITEMS** | Line items within an order |
| **QUOTATIONS** | Applies to Sales Rep (B2B) channel; converts to confirmed Order on approval |
| **PAYMENTS** | Payment records; `PaymentMethod` = Cash / CreditCard / BankTransfer / QR |
| **INVOICES** | Tax invoices linked to orders; tracks outstanding balances (AR) |
| **SHIPMENTS** | Fulfillment records; tracks dispatch from specific warehouse with carrier & tracking |

---

## Key Design Decisions

- **INVENTORIES** acts as a **junction table** (many-to-many) between PRODUCTS and WAREHOUSES, also storing live stock quantities.
- **CUSTOMERS.CustomerType** = `Retail` | `Wholesale` — drives credit terms and pricing tiers.
- **ORDERS.ChannelID** (FK → SALES_CHANNELS) is the single source of truth for which channel originated the order, enabling unified sales reporting.
- **QUOTATIONS** is a sub-entity of ORDERS, used exclusively in the Sales Rep workflow before order confirmation.
- **QuantityAvailable** in INVENTORIES is a computed/maintained field: `QuantityOnHand − QuantityReserved`.
