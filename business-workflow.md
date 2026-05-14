# Part 2: Business Workflow — ABC Trading ERP System

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber

    participant C   as Customer
    participant ERP as ERP System
    participant INV as Inventory Module
    participant ACC as Accounting Module

    Note over C,ACC: CHANNEL 1 - POS Walk-in Retail (Instant Flow)

    Note over ERP: Cashier logs in and opens cash shift
    ERP->>ERP: Create CASH_SHIFTS record with store_id, employee_id, and opening_cash

    C->>ERP: Scan barcode or select products at counter
    ERP->>INV: Check real-time stock availability
    INV-->>ERP: Stock confirmed with quantity and price
    ERP->>C: Display cart and total amount including VAT

    C->>ERP: Tender payment via Cash, Card, or QR
    ERP->>ACC: Record payment and create ORDER with store_id and cashier_id
    ACC-->>ERP: Payment confirmed and journal entry created

    ERP->>INV: Deduct inventory qty_on_hand for sold items
    INV-->>ERP: Stock updated successfully
    ERP->>C: Issue receipt (print or digital)
    Note right of ERP: Order status set to Completed

    Note over ERP: End of shift — cashier counts register
    ERP->>ERP: Update CASH_SHIFTS with closing_cash and closed_at

    Note over C,ACC: CHANNEL 2 - E-commerce Online Order Flow

    C->>ERP: Place order via website or mobile app
    ERP->>INV: Check stock and soft-reserve items
    INV-->>ERP: Reservation confirmed with reservation ID
    ERP->>ACC: Generate proforma invoice
    ERP->>C: Send order confirmation and payment link

    C->>ERP: Complete online payment
    ERP->>ACC: Verify and record payment
    ACC-->>ERP: Payment verified and posted

    ERP->>INV: Confirm reservation and trigger Pick and Pack
    INV-->>ERP: Goods ready for dispatch
    ERP->>C: Send shipment notification with tracking number
    ERP->>ACC: Issue tax invoice
    ACC-->>ERP: Invoice recorded in AR ledger
    Note right of ERP: Order status set to Shipped

    Note over C,ACC: CHANNEL 3 - Sales Rep B2B (Quotation to Sales Order Flow)

    C->>ERP: Submit Request for Quotation (RFQ)
    ERP->>INV: Check availability and lead time
    INV-->>ERP: Available qty and lead time confirmed

    ERP->>ERP: Create QUOTATION_ITEMS with negotiated unit price and discount per product
    ERP->>C: Send Quotation with line items and validity period

    alt Customer Approves Quotation
        C->>ERP: Approve quotation and convert to Sales Order
        ERP->>ERP: Copy QUOTATION_ITEMS to ORDER_ITEMS with agreed pricing
        ERP->>INV: Reserve inventory against Sales Order
        INV-->>ERP: Inventory reserved and qty_reserved updated
        ERP->>C: Send Sales Order acknowledgement

        ERP->>INV: Trigger warehouse fulfillment Pick and Pack and Ship
        INV-->>ERP: Shipment dispatched with tracking number and delivery date

        ERP->>ACC: Issue tax invoice with credit terms applied
        ACC-->>ERP: Invoice created and AR entry posted
        ERP->>C: Send invoice and delivery note
        Note right of ACC: Credit period starts e.g. Net 30 days

        C->>ACC: Pay via bank transfer or cheque within credit period
        ACC->>ACC: Match payment to invoice and clear AR balance
        ACC->>ERP: Invoice marked as settled
        ERP->>C: Send payment receipt and account statement

    else Customer Rejects or Quotation Expires
        C->>ERP: Decline or no response before expiry
        ERP->>ERP: Mark quotation as Cancelled or Expired
        ERP->>INV: Release any soft reservation
        Note right of ERP: Order status set to Cancelled
    end
```

---

## Flow Summary Table

| Step | POS | E-commerce | Sales Rep (B2B) |
|------|-----|------------|-----------------|
| **Order Creation** | Instant scan at counter (store_id + cashier_id recorded) | Online cart checkout | Request for Quotation (RFQ) |
| **Inventory Check** | Real-time at POS | Real-time + soft reserve | Check availability + lead time |
| **Pricing** | Fixed retail price | Fixed retail price | Negotiated per product via QUOTATION_ITEMS |
| **Approval Step** | None | None | Quotation requires customer approval |
| **Payment Timing** | Immediate at counter | Before shipment (online) | After delivery within credit period |
| **Fulfillment** | Customer takes goods instantly | Warehouse picks and ships | Warehouse picks and ships |
| **Cashier Tracking** | Shift opened/closed per session (CASH_SHIFTS) | N/A | N/A |
| **Invoice** | Receipt issued instantly | Tax invoice after payment | Tax invoice with credit terms |
| **Settlement** | Immediate | Immediate | Within credit period (Net 30 or Net 60) |

---

## Status Lifecycles

### Order Status
```
pending -> paid -> shipped -> completed
       \-> cancelled (before shipped)
```

### Quotation Status (B2B only)
```
draft -> sent -> approved -> [converted to Sales Order]
              \-> rejected
              \-> expired
```

### Payment Status
```
pending -> success
       \-> failed -> [retry]
```

### Inventory Reservation
```
available -> soft_reserved (on order or quotation)
          -> committed (deducted on shipment)
          \-> released (on cancellation)
```

---

## Failure Scenarios

### Scenario 1: Payment Failure and Retry

```mermaid
sequenceDiagram
    participant C   as Customer
    participant ERP as ERP System
    participant GW  as Payment Gateway
    participant ACC as Accounting Module

    C->>ERP: Submit payment for order
    ERP->>GW: Charge request with idempotency key (payment_id + attempt 1)
    GW-->>ERP: Failed - card declined
    ERP->>ACC: Log failed PAYMENT_TRANSACTIONS record attempt 1
    ERP->>C: Notify failure and prompt retry

    C->>ERP: Retry payment
    ERP->>GW: Charge request with same idempotency key (payment_id + attempt 2)
    GW-->>ERP: Payment success
    ERP->>ACC: Log successful PAYMENT_TRANSACTIONS record attempt 2
    ERP->>ERP: Update order status to paid
    Note right of ERP: Same idempotency key prevents double charge on retry
```

### Scenario 2: Inventory Conflict - Oversell Prevention

```mermaid
sequenceDiagram
    participant UA  as User A
    participant UB  as User B
    participant ERP as ERP System
    participant INV as Inventory Module

    Note over INV: qty_on_hand is 10 for SKU-001

    UA->>ERP: Order 10 units of SKU-001
    UB->>ERP: Order 8 units of SKU-001

    ERP->>INV: Reserve 10 units for User A using SELECT FOR UPDATE row lock
    INV-->>ERP: Reserved - qty_reserved set to 10 and qty_available now 0
    ERP->>UA: Order confirmed

    ERP->>INV: Reserve 8 units for User B using SELECT FOR UPDATE row lock
    INV-->>ERP: Rejected - qty_available is 0
    ERP->>UB: Order rejected - insufficient stock
    Note right of INV: Row-level lock prevents concurrent oversell
```

### Scenario 3: Shipment Failure - Saga Compensation

```mermaid
sequenceDiagram
    participant ERP as ERP System
    participant INV as Inventory Module
    participant SHP as Shipment Service
    participant ACC as Accounting Module

    ERP->>INV: Commit inventory - deduct qty_on_hand
    INV-->>ERP: Committed and INVENTORY_MOVEMENTS record written
    ERP->>SHP: Create shipment record
    SHP-->>ERP: Shipment creation failed

    Note over ERP: Saga compensation triggered
    ERP->>INV: Compensate - restore qty_on_hand via cancel_commit movement
    INV-->>ERP: Inventory restored and compensation movement logged
    ERP->>ACC: Set invoice status to on_hold
    ERP->>ERP: Schedule shipment retry with exponential backoff
    Note right of ERP: Order stays in processing until retry succeeds or escalation
```

---

## Event-Driven Flow

Domain events decouple modules and guarantee reliable async processing.

```
Event                   Consumers
────────────────────────────────────────────────────────────────
OrderCreated         -> Inventory: ReserveStock
PaymentSuccess       -> Order: SetStatus(paid)
                     -> Inventory: CommitStock
                     -> Shipment: CreateShipment
                     -> Accounting: IssueInvoice
ShipmentCreated      -> Notification: NotifyCustomer(shipped)
ShipmentFailed       -> Inventory: RollbackCommit (cancel_commit)
                     -> Accounting: PutInvoiceOnHold
                     -> Order: ScheduleRetry
OrderCancelled       -> Inventory: ReleaseReservation
                     -> Accounting: VoidInvoice
PaymentFailed        -> Notification: NotifyCustomer(retry)
                     -> Order: SchedulePaymentRetry
```

---

## Inventory State Transitions

Every state change writes an immutable record to INVENTORY_MOVEMENTS.

```
AVAILABLE
  |-- [order confirmed]       --> SOFT_RESERVED  (movement_type: order_reserve)
  |-- [order cancelled]       --> AVAILABLE       (movement_type: cancel)

SOFT_RESERVED
  |-- [payment confirmed]     --> COMMITTED       (movement_type: order_commit)
  |-- [order cancelled]       --> AVAILABLE       (movement_type: cancel)

COMMITTED
  |-- [shipment dispatched]   --> FULFILLED        qty_on_hand decremented
  |-- [shipment failed]       --> SOFT_RESERVED   (movement_type: cancel_commit)
```
