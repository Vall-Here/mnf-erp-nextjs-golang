# 03. Software Design Document: Sales & Customer Distribution

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Sales & Customer Distribution** mengelola siklus pendapatan (*Order-to-Cash* / O2C): Penawaran Harga Pelanggan (*Quotations*), Konfirmasi Pesanan (*Sales Orders*), Pengendalian Batas Kredit Pelanggan (*Credit Limit & Credit Hold*), Surat Jalan Pengiriman (*Delivery Orders*), Konfirmasi Serah Terima (*Proof of Delivery* / POD), Faktur Penjualan (*Sales Invoices*), dan Pelacakan Piutang Usaha (*AR Aging*).

- **Lokasi Backend**: `services/api/internal/modules/sales/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/sales/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    SalesRep["Sales Officer"]
    SalesMgr["Sales & Marketing Manager"]
    FinMgr["Finance & Accounting Manager"]
    WhLead["Warehouse Dispatcher"]
    ARStaff["Staf Piutang (AR)"]

    subgraph SalesModule ["Modul Sales & Customer Distribution"]
        UC1["Kelola Data Pelanggan, NPWP, & Plafon Kredit"]
        UC2["Penerbitan Surat Penawaran Harga (Quotation)"]
        UC3["Konversi Quotation ke Sales Order (SO)"]
        UC4["Evaluasi Paparan Kredit & Otorisasi Pelepasan Credit Hold"]
        UC5["Penerbitan Surat Jalan Pengiriman (Delivery Order)"]
        UC6["Konfirmasi Tanda Terima Pengiriman (Proof of Delivery / POD)"]
        UC7["Penerbitan Faktur Penjualan (Sales Invoice)"]
        UC8["Penerimaan Pembayaran Pelanggan & Kartu Piutang"]
    end

    SalesRep --> UC1
    SalesRep --> UC2
    SalesRep --> UC3

    SalesMgr --> UC4
    FinMgr --> UC4

    WhLead --> UC5
    WhLead --> UC6

    ARStaff --> UC7
    ARStaff --> UC8
```

---

## 3. Activity Diagram (Siklus Order-to-Cash & Otorisasi Credit Hold)

```mermaid
flowchart TD
    Start([Pesanan Masuk dari Pelanggan]) --> CreateSO[Sales Buat Sales Order / SO]
    CreateSO --> CheckCredit{Cek Batas Kredit: Saldo Piutang + Nilai SO > Credit Limit?}
    
    CheckCredit -- Ya: Melebihi Plafon --> HoldCredit[Set credit_status: HOLD]
    HoldCredit --> NotifyApprover[Notifikasi ke Sales Manager / Finance Manager]
    NotifyApprover --> ReviewExposure[Evaluasi Riwayat Pembayaran & Jaminan]
    
    ReviewExposure --> CanRelease{Memiliki sales:approve / accounting:approve?}
    CanRelease -- Tidak --> DenyRelease[Tolak: 403 Forbidden - Staf Sales Dilarang Override Batas Kredit]
    DenyRelease --> EndFail([Pesanan Tertahan])
    
    CanRelease -- Ya --> ExecuteRelease[Eksekusi Endpoint /orders/{id}/release-credit]
    ExecuteRelease --> SetApproved[Set credit_status: APPROVED & Catat credit_released_by]
    SetApproved --> CreateDO
    
    CheckCredit -- Tidak: Dalam Plafon --> SetApprovedNormal[credit_status: APPROVED]
    SetApprovedNormal --> CreateDO[Terbitkan Surat Jalan / Delivery Order]
    
    CreateDO --> Dispatch[Pengiriman Barang oleh Tim Logistik]
    Dispatch --> ReceivePOD[Konfirmasi Serah Terima Barang / POD]
    ReceivePOD --> GenInvoice[Generate Faktur Penjualan / Sales Invoice]
    GenInvoice --> EndDone([Siklus Penjualan Lengkap])
```

---

## 4. Sequence Diagram (Penerbitan Surat Jalan & Mutasi Stock Ledger)

```mermaid
sequenceDiagram
    autonumber
    actor User as Warehouse Dispatcher
    participant UI as Sales Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Sales Handler
    participant Svc as Sales Service
    participant Repo as Sales Repository
    participant DB as PostgreSQL 16

    User->>UI: Buat Surat Jalan dari SO Terkonfirmasi
    UI->>MW: POST /api/sales/orders/{id}/deliver (Bearer Token)
    MW->>MW: Verifikasi izin sales:edit & Cabang Terotorisasi
    MW->>Hdl: HandleDeliverOrder
    Hdl->>Svc: DeliverOrder
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Transaksi Database Terisolasi (pgx.Tx)
        Svc->>Repo: Validasi SO & Verifikasi credit_status == APPROVED
        Repo->>DB: SELECT * FROM sales_orders WHERE id = $1 FOR UPDATE
        
        Svc->>Repo: Kurangi Stok & Rekam Mutasi Perpetual
        Repo->>DB: INSERT INTO stock_ledger (movement_type: OUT_SALE, warehouse_id, qty)
        Repo->>DB: UPDATE item_stock_summary SET balance_quantity = balance_quantity - qty
        
        Svc->>Repo: Buat Dokumen Delivery Order
        Repo->>DB: INSERT INTO delivery_orders (do_number, status: DISPATCHED)
        Repo->>DB: INSERT INTO delivery_order_items
        
        Svc->>Repo: Update Status Pesanan
        Repo->>DB: UPDATE sales_orders SET status = 'DELIVERED'
    end

    Svc-->>Hdl: Delivery Order Entity
    Hdl-->>UI: 200 OK (Surat Jalan Resmi Siap Cetak)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    CUSTOMERS ||--o{ QUOTATIONS : "requests"
    CUSTOMERS ||--o{ SALES_ORDERS : "places"
    CUSTOMERS ||--o{ SALES_INVOICES : "billed_to"
    SALES_ORDERS ||--|{ SALES_ORDER_ITEMS : "contains"
    SALES_ORDERS ||--o{ DELIVERY_ORDERS : "fulfilled_by"
    DELIVERY_ORDERS ||--|{ DELIVERY_ORDER_ITEMS : "contains"
    DELIVERY_ORDERS ||--o| SALES_INVOICES : "invoiced_as"

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
        string status "DRAFT | CONFIRMED | DELIVERED | CANCELLED"
        string credit_status "APPROVED | HOLD"
        string credit_release_notes
        uuid credit_released_by FK
        timestamp credit_released_at
        numeric subtotal
        numeric tax_amount
        numeric grand_total
    }

    DELIVERY_ORDERS {
        uuid id PK
        uuid company_id FK
        uuid sales_order_id FK
        uuid branch_id FK
        string do_number UK
        date delivery_date
        string status "DRAFT | DISPATCHED | DELIVERED"
        timestamp pod_confirmed_at
        string pod_recipient_name
    }

    SALES_INVOICES {
        uuid id PK
        uuid company_id FK
        uuid sales_order_id FK
        uuid delivery_order_id FK
        string invoice_number UK
        date invoice_date
        date due_date
        numeric grand_total
        numeric paid_amount
        string payment_status "UNPAID | PARTIALLY_PAID | PAID"
    }
```
