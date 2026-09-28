# 03. Software Design Document: Sales & Customer Distribution

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Sales & Customer Distribution** mengelola siklus pendapatan (*Order-to-Cash*): Penawaran Harga Pelanggan (*Quotations*), Konfirmasi Pesanan (*Sales Orders*), Surat Jalan Pengiriman (*Delivery Orders*), Faktur Penjualan (*Sales Invoices*), dan Pelacakan Piutang Usaha (*AR Aging*).

- **Lokasi Backend**: `services/api/internal/modules/sales/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/sales/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    SalesRep["Sales Executive"]
    SalesMgr["Manajer Penjualan"]
    WhLead["Warehouse / Dispatcher"]
    ARStaff["Staf Piutang (AR)"]

    subgraph SalesModule ["Modul Sales & Customer Distribution"]
        UC1["Kelola Data Pelanggan & Plafon Kredit"]
        UC2["Buat & Kirim Surat Penawaran Harga (Quotation)"]
        UC3["Konversi Quotation ke Sales Order (SO)"]
        UC4["Otorisasi Sales Order & Diskon Khusus"]
        UC5["Terbitkan Surat Jalan Pengiriman (DO)"]
        UC6["Terbitkan Faktur Penjualan (Sales Invoice)"]
        UC7["Pantau Umur Piutang (AR Aging Schedule)"]
    end

    SalesRep --> UC1
    SalesRep --> UC2
    SalesRep --> UC3

    SalesMgr --> UC4

    WhLead --> UC5

    ARStaff --> UC6
    ARStaff --> UC7
```

---

## 3. Activity Diagram (Siklus Order-to-Cash)

```mermaid
flowchart TD
    Start([Inisiasi Pesanan Pelanggan]) --> CreateQuo[Buat Quotation Penawaran]
    CreateQuo --> SendCustomer[Kirim ke Pelanggan]
    SendCustomer --> CustAccept{Pelanggan Menyetujui?}
    
    CustAccept -- Revisi/Batal --> CreateQuo
    CustAccept -- Setuju --> ConvertSO[Konversi ke Sales Order / SO]
    
    ConvertSO --> CheckCredit{Cek Plafon Kredit & Diskon}
    CheckCredit -- Melebihi Batas --> RequestApproval[Memerlukan Otorisasi Manajer / Direktur]
    RequestApproval --> ApproveSO[Otorisasi Disetujui]
    
    CheckCredit -- Dalam Batas Normal --> ApproveSO
    
    ApproveSO --> CheckStock{Ketersediaan Stok Barang Jadi}
    CheckStock -- Stok Kurang --> TriggerMfg[Terbitkan Work Order Produksi ke Pabrik]
    TriggerMfg --> WaitStock[Tunggu Barang Siap di Gudang]
    WaitStock --> CreateDO
    
    CheckStock -- Stok Cukup --> CreateDO[Terbitkan Surat Jalan / Delivery Order]
    CreateDO --> Dispatch[Dispatch Logistik & Pengiriman]
    Dispatch --> ReceivePOD[Konfirmasi Serah Terima / POD]
    
    ReceivePOD --> GenInvoice[Generate Sales Invoice / AR]
    GenInvoice --> AutoJournal[Auto-Posting Jurnal: Debit Piutang, Kredit Penjualan + PPN]
    AutoJournal --> EndDone([Selesai Siklus Penjualan])
```

---

## 4. Sequence Diagram (Penerbitan Delivery Order & Pengurangan Stok)

```mermaid
sequenceDiagram
    autonumber
    actor User as Warehouse Dispatcher
    participant UI as Sales Web (Next.js 15)
    participant Hdl as Sales Handler
    participant Svc as Sales Service
    participant Inv as Inventory Module
    participant Repo as Sales Repository
    participant DB as PostgreSQL 16

    User->>UI: Buat Delivery Order dari SO Terkonfirmasi
    UI->>Hdl: POST /api/v1/sales/delivery-orders
    Hdl->>Svc: CreateDeliveryOrder(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: Validate SO Status & Unfulfilled Qty
        Repo->>DB: SELECT * FROM sales_orders WHERE id = ... FOR UPDATE
        
        Svc->>Inv: DeductStock(branchID, warehouseID, itemID, qty, "OUT_SALE", doNumber)
        Inv->>DB: INSERT INTO stock_movements (OUT_SALE)
        Inv->>DB: UPDATE items SET balance_qty = balance_qty - qty
        
        Svc->>Repo: Insert Delivery Order & Lines
        Repo->>DB: INSERT INTO delivery_orders ...
        Repo->>DB: INSERT INTO delivery_order_items ...
        
        Svc->>Repo: Update SO Delivered Quantity
        Repo->>DB: UPDATE sales_orders SET status = 'PARTIALLY_DELIVERED' / 'DELIVERED'
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: DO Entity
    Hdl-->>UI: 201 Created
    UI-->>User: Tampilkan Dokumen Cetak Surat Jalan Resmi (A4)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    CUSTOMERS ||--o{ QUOTATIONS : "requests"
    CUSTOMERS ||--o{ SALES_ORDERS : "places"
    CUSTOMERS ||--o{ SALES_INVOICES : "billed_to"
    QUOTATIONS ||--o{ QUOTATION_ITEMS : "has"
    QUOTATIONS ||--o| SALES_ORDERS : "converted_to"
    SALES_ORDERS ||--|{ SALES_ORDER_ITEMS : "contains"
    SALES_ORDERS ||--o{ DELIVERY_ORDERS : "fulfilled_by"
    DELIVERY_ORDERS ||--|{ DELIVERY_ORDER_ITEMS : "contains"
    DELIVERY_ORDERS ||--o| SALES_INVOICES : "invoiced_as"
    SALES_INVOICES ||--|{ SALES_INVOICE_ITEMS : "contains"

    CUSTOMERS {
        uuid id PK
        uuid company_id FK
        string customer_code UK
        string name
        string email
        string phone
        string tax_id
        numeric credit_limit
        integer payment_term_days
        boolean is_active
    }

    SALES_ORDERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid customer_id FK
        string so_number UK
        date order_date
        date expected_delivery_date
        string status "DRAFT | CONFIRMED | PARTIALLY_DELIVERED | DELIVERED | CANCELLED"
        numeric subtotal
        numeric discount_amount
        numeric tax_amount
        numeric grand_total
    }

    SALES_ORDER_ITEMS {
        uuid id PK
        uuid sales_order_id FK
        uuid item_id FK
        numeric quantity
        numeric delivered_quantity
        numeric unit_price
        numeric discount_percent
        numeric total_amount
    }

    DELIVERY_ORDERS {
        uuid id PK
        uuid sales_order_id FK
        uuid branch_id FK
        string do_number UK
        date delivery_date
        string tracking_number
        string status "DRAFT | DISPATCHED | DELIVERED"
        uuid driver_id FK
    }

    SALES_INVOICES {
        uuid id PK
        uuid sales_order_id FK
        uuid delivery_order_id FK
        uuid customer_id FK
        string invoice_number UK
        date invoice_date
        date due_date
        numeric grand_total
        numeric paid_amount
        string payment_status "UNPAID | PARTIALLY_PAID | PAID | OVERDUE"
        uuid journal_id FK
    }
```
