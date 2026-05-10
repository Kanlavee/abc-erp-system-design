# Part 2: Business Workflow — ABC Trading ERP System

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber

    participant C  as 🧑 Customer<br/>(POS / Online / Wholesale)
    participant ERP as ⚙️ ERP System
    participant INV as 📦 Inventory Module
    participant ACC as 💰 Accounting Module

    %% ════════════════════════════════════════════════════════
    Note over C,ACC: ── CHANNEL 1: POS (Walk-in / Retail — Instant Flow) ──
    %% ════════════════════════════════════════════════════════

    C->>ERP: Scan barcode / Select products at counter
    ERP->>INV: Check real-time stock availability
    INV-->>ERP: Stock confirmed (quantity & price)
    ERP->>C: Display cart & total amount (incl. VAT)

    C->>ERP: Tender payment (Cash / Card / QR)
    ERP->>ACC: Record payment transaction
    ACC-->>ERP: Payment confirmed & journal entry created

    ERP->>INV: Commit inventory deduction (QuantityOnHand -=)
    INV-->>ERP: Stock updated successfully
    ERP->>C: Print / send e-Receipt
    Note right of ERP: Order Status → Delivered ✅<br/>No shipment step needed

    %% ════════════════════════════════════════════════════════
    Note over C,ACC: ── CHANNEL 2: E-commerce (Online Order Flow) ──
    %% ════════════════════════════════════════════════════════

    C->>ERP: Place order via website / app
    ERP->>INV: Check & soft-reserve stock
    INV-->>ERP: Reservation confirmed (ReservationID)
    ERP->>ACC: Generate Proforma Invoice
    ERP->>C: Order confirmation + payment link / instructions

    C->>ERP: Complete online payment
    ERP->>ACC: Verify & record payment (AR settled)
    ACC-->>ERP: Payment verified

    ERP->>INV: Confirm reservation → trigger Pick & Pack
    INV-->>ERP: Goods ready for dispatch
    ERP->>C: Shipment notification (TrackingNo)
    Note right of ERP: Order Status → Shipped 🚚

    ERP->>ACC: Issue Tax Invoice
    ACC-->>ERP: Invoice recorded in AR ledger

    %% ════════════════════════════════════════════════════════
    Note over C,ACC: ── CHANNEL 3: Sales Rep / B2B (Quotation → Sales Order Flow) ──
    %% ════════════════════════════════════════════════════════

    C->>ERP: Request for Quotation (RFQ)
    ERP->>INV: Check availability & lead time
    INV-->>ERP: Available qty, warehouse location, lead time

    ERP->>ERP: Apply wholesale pricing / discount rules
    ERP->>C: Send Quotation (QT) with validity period

    alt Customer Approves Quotation
        C->>ERP: Approve QT → Convert to Sales Order (SO)
        ERP->>INV: Reserve inventory against SO
        INV-->>ERP: Inventory reserved (QuantityReserved +=)
        ERP->>C: SO Acknowledgement (order confirmed)

        ERP->>INV: Trigger warehouse fulfillment (Pick → Pack → Ship)
        INV-->>ERP: Shipment dispatched (TrackingNo, DeliveryDate)

        ERP->>ACC: Issue Tax Invoice (credit terms applied)
        ACC-->>ERP: Invoice created; AR entry posted

        ERP->>C: Send Invoice + Delivery Note
        Note right of ACC: Credit period starts (e.g., Net 30)

        C->>ACC: Bank transfer / cheque within credit period
        ACC->>ACC: Match payment to invoice (AR cleared)
        ACC->>ERP: Invoice marked as Settled ✅
        ERP->>C: Payment receipt / statement

    else Customer Rejects / Quotation Expires
        C->>ERP: Decline or no response
        ERP->>ERP: Mark Quotation as Cancelled/Expired
        ERP->>INV: Release any soft reservation
        Note right of ERP: Order Status → Cancelled ❌
    end
```

---

## Flow Summary Table

| Step | POS | E-commerce | Sales Rep (B2B) |
|------|-----|------------|-----------------|
| **Order Creation** | Instant (scan at counter) | Online cart checkout | Request for Quotation (RFQ) |
| **Inventory Check** | Real-time at POS | Real-time + soft reserve | Check availability + lead time |
| **Pricing** | Fixed retail price | Fixed retail price | Wholesale / negotiated price |
| **Approval Step** | None | None | Quotation → Customer Approval |
| **Payment Timing** | Immediate (at counter) | Before shipment (online) | After delivery (credit terms) |
| **Fulfillment** | Customer takes goods | Warehouse → delivery | Warehouse → delivery |
| **Invoice** | Receipt issued instantly | Tax invoice after payment | Tax invoice with credit terms |
| **Settlement** | Immediate | Immediate | Within credit period (Net 30/60) |

---

## Status Lifecycles

### Order Status
```
Draft → Confirmed → Processing → Shipped → Delivered → [Closed]
                  ↘ Cancelled (at any point before Shipped)
```

### Quotation Status (B2B only)
```
Draft → Sent → Approved → [Converted to SO]
             ↘ Rejected / Expired
```

### Payment Status
```
Pending → Verified → Settled
        ↘ Failed → Retry
```

### Inventory Reservation
```
Available → SoftReserved (on order/quotation) → Committed (deducted on shipment)
          ↘ Released (on cancellation)
```
