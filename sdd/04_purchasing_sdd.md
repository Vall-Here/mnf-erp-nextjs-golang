# 04. Software Design Document: Purchasing & Procurement

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Purchasing & Procurement** mengendalikan seluruh proses pengadaan barang dan jasa (*Procure-to-Pay* / P2P): Permintaan Pembelian (*Purchase Requisition / PR*), Tender Penawaran Pemasok (*RFQ & Vendor Bidding*), Penetapan Pemenang (*Awarding*), Pesanan Pembelian (*Purchase Order / PO*), Penerimaan Barang Gudang (*Goods Receipt Note / GRN*), Alokasi Biaya Tambahan (*Landed Cost*), Verifikasi 3-Way Matching, dan Pengesahan Tagihan Pemasok (*Vendor Bills / AP*).

- **Lokasi Backend**: `services/api/internal/modules/purchasing/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/purchasing/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Requester["Staff Pemohon (User Dept)"]
    Buyer["Procurement Officer"]
    PurchMgr["Purchasing Manager"]
    FinMgr["Finance & Accounting Manager"]
    Receiver["Staff Gudang (GRN Receiver)"]
    APStaff["Staff Utang Usaha (AP Staff)"]

    subgraph PurchasingModule ["Modul Purchasing & Procurement"]
        UC1["Pengajuan Purchase Requisition (PR)"]
        UC2["Penerbitan Tender & RFQ ke Pemasok"]
        UC3["Pencatatan Penawaran Vendor (Vendor Bids)"]
        UC4["Penetapan Pemenang Tender (Award RFQ: purchasing:approve)"]
        UC5["Penerbitan Purchase Order (PO)"]
        UC6["Otorisasi PO Multi-Tier (purchasing:approve)"]
        UC7["Penerimaan Barang Fisik Gudang (GRN)"]
        UC8["Verifikasi 3-Way Matching (PO vs GRN vs Bill)"]
        UC9["Pengesahan Tagihan Pemasok (purchasing:approve / accounting:approve)"]
    end

    Requester --> UC1
    Buyer --> UC2
    Buyer --> UC3
    Buyer --> UC5

    PurchMgr --> UC4
    PurchMgr --> UC6
    PurchMgr --> UC9
    FinMgr --> UC9

    Receiver --> UC7
    APStaff --> UC8
```

---

## 3. Activity Diagram (Siklus Procure-to-Pay, RFQ Award, & SoD 3-Way Match)

```mermaid
flowchart TD
    Start([Kebutuhan Pengadaan]) --> CreatePR[User Buat Purchase Requisition]
    CreatePR --> ApprovePR[Otorisasi PR oleh Atasan / Manajer]
    ApprovePR --> CreateRFQ[Buyer Terbitkan RFQ ke Minimal 3 Pemasok]
    
    CreateRFQ --> ReceiveBids[Catat Penawaran Harga & Spesifikasi Pemasok]
    ReceiveBids --> AwardRFQ{Penetapan Pemenang: Memiliki purchasing:approve?}
    
    AwardRFQ -- Tidak --> DenyAward[403 Forbidden: Staf Buyer Dilarang Menetapkan Pemenang Sendiri]
    DenyAward --> EndFail([Tender Tertunda])
    
    AwardRFQ -- Ya --> SetWinner[Pemenang Ditetapkan & PO Diterbitkan]
    SetWinner --> ApprovePO[Persetujuan PO Multi-Tier Berdasarkan Nominal]
    
    ApprovePO --> ArriveGoods[Barang Tiba di Fasilitas Gudang Pabrik]
    ArriveGoods --> ReceiveGRN[Staff Gudang Terbitkan GRN & Inspeksi IQC]
    ReceiveGRN --> ArriveBill[Terima Faktur Tagihan dari Vendor]
    
    ArriveBill --> ThreeWayMatch{3-Way Match: Kuantitas & Harga Sesuai PO & GRN?}
    ThreeWayMatch -- Selisih Material --> HoldBill[Tahan Tagihan & Terbitkan Debit Note]
    HoldBill --> EndFail
    
    ThreeWayMatch -- Cocok --> ApproveBill{Otorisasi Tagihan: purchasing:approve / accounting:approve?}
    ApproveBill -- Ditolak / SoD Violation --> RejectBill[Tolak: Pembuat Tagihan Dilarang Approve Sendiri]
    ApproveBill -- Disetujui --> PostBill[Posting Vendor Bill ke Utang Dagang AP]
    PostBill --> EndDone([Siklus Pengadaan Selesai])
```

---

## 4. Sequence Diagram (Goods Receipt Note & Stock Ledger)

```mermaid
sequenceDiagram
    autonumber
    actor Receiver as Staff Gudang Penerima
    participant UI as Purchasing Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Purchasing Handler
    participant Svc as Purchasing Service
    participant Repo as Purchasing Repository
    participant DB as PostgreSQL 16

    Receiver->>UI: Submit Penerimaan Barang dari PO
    UI->>MW: POST /api/purchasing/orders/{id}/receive (Bearer Token)
    MW->>MW: Verifikasi izin inventory:create / purchasing:edit
    MW->>Hdl: HandleReceiveOrder
    Hdl->>Svc: ReceiveOrder
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Mutasi Stok Masuk Perpetual (pgx.Tx)
        Svc->>Repo: Validasi Sisa PO Belum Diterima
        Repo->>DB: SELECT * FROM purchase_orders WHERE id = $1 FOR UPDATE
        
        Svc->>Repo: Catat Mutasi Penerimaan
        Repo->>DB: INSERT INTO stock_ledger (movement_type: IN_PURCHASE, warehouse_id, qty)
        Repo->>DB: UPDATE item_stock_summary SET balance_quantity = balance_quantity + qty
        
        Svc->>Repo: Catat Dokumen Penerimaan Gudang (GRN)
        Repo->>DB: INSERT INTO purchase_receipts (grn_number, status: RECEIVED)
        Repo->>DB: UPDATE purchase_orders SET status = 'RECEIVED'
    end

    Svc-->>Hdl: Receipt Entity
    Hdl-->>UI: 200 OK (GRN Berhasil Diterbitkan)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    VENDORS ||--o{ PURCHASE_REQUISITIONS : "targets"
    VENDORS ||--o{ PURCHASE_ORDERS : "receives"
    VENDORS ||--o{ VENDOR_BILLS : "bills"
    PURCHASE_REQUISITIONS ||--o| PURCHASE_ORDERS : "converted_to"
    PURCHASE_ORDERS ||--|{ PURCHASE_ORDER_ITEMS : "contains"
    PURCHASE_ORDERS ||--o{ PURCHASE_RECEIPTS : "fulfilled_by"
    PURCHASE_RECEIPTS ||--|{ PURCHASE_RECEIPT_ITEMS : "contains"
    PURCHASE_RECEIPTS ||--o| VENDOR_BILLS : "matched_to"

    VENDORS {
        uuid id PK
        uuid company_id FK
        string vendor_code UK
        string name
        string email
        string phone
        string tax_id
        integer payment_term_days
        boolean is_active
    }

    PURCHASE_ORDERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid vendor_id FK
        string po_number UK
        date order_date
        string status "DRAFT | PENDING_APPROVAL | APPROVED | RECEIVED | CANCELLED"
        numeric subtotal
        numeric tax_amount
        numeric grand_total
        uuid approved_by FK
        timestamp approved_at
    }

    PURCHASE_RECEIPTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid purchase_order_id FK
        uuid warehouse_id FK
        string receipt_number UK
        date receipt_date
        string status "RECEIVED | QC_PENDING | COMPLETED"
    }

    VENDOR_BILLS {
        uuid id PK
        uuid company_id FK
        uuid vendor_id FK
        uuid purchase_order_id FK
        uuid purchase_receipt_id FK
        string bill_number UK
        date bill_date
        date due_date
        numeric grand_total
        string status "DRAFT | APPROVED | PAID"
        boolean three_way_matched
        uuid approved_by FK
    }
```
