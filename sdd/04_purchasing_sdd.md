# 04. Software Design Document: Purchasing & Procurement

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Purchasing & Procurement** mengendalikan siklus pengadaan barang & jasa (*Procure-to-Pay*): Permintaan Pembelian (*Purchase Requisition / PR*), Permintaan Penawaran Pemasok (*RFQ*), Pesanan Pembelian (*Purchase Order / PO*), Penerimaan Barang Gudang (*Goods Receipt Note / GRN*), Alokasi Biaya Tambahan (*Landed Cost*), dan Tagihan Pemasok (*Vendor Bills / AP*).

- **Lokasi Backend**: `services/api/internal/modules/purchasing/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/purchasing/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Requester["Staff Pemohon (User Dept)"]
    Buyer["Procurement Specialist"]
    Mgr["Manajer Pengadaan / Direktur"]
    Receiver["Staff Gudang Penerima"]
    APStaff["Staff Utang Usaha (AP)"]

    subgraph PurchasingModule ["Modul Purchasing & Procurement"]
        UC1["Ajukan Permintaan Pembelian (PR)"]
        UC2["Buat Tender & RFQ ke Pemasok"]
        UC3["Komparasi Harga Penawaran Vendor"]
        UC4["Terbitkan Purchase Order (PO)"]
        UC5["Otorisasi PO Multi-Tier (1x s.d 4x Level)"]
        UC6["Penerimaan Fisik Barang Gudang (GRN)"]
        UC7["Pencatatan Biaya Tambahan (Landed Cost)"]
        UC8["Verifikasi 3-Way Match & Tagihan Vendor"]
    end

    Requester --> UC1
    Buyer --> UC2
    Buyer --> UC3
    Buyer --> UC4

    Mgr --> UC5

    Receiver --> UC6

    Buyer --> UC7
    APStaff --> UC8
```

---

## 3. Activity Diagram (Siklus Procure-to-Pay & 3-Way Matching)

```mermaid
flowchart TD
    Start([Kebutuhan Pengadaan]) --> CreatePR[User Buat Purchase Requisition]
    CreatePR --> ApprovePR[Otorisasi PR oleh Atasan]
    ApprovePR --> CreateRFQ[Buyer Terbitkan RFQ ke Minimal 3 Vendor]
    
    CreateRFQ --> ReceiveBids[Terima Penawaran & Evaluasi Komparasi]
    ReceiveBids --> SelectVendor[Pilih Vendor Pemenang]
    SelectVendor --> CreatePO[Terbitkan Purchase Order / PO]
    
    CreatePO --> CheckValue{Ambang Batas Nominal PO}
    CheckValue -- < Rp 100 Juta --> ApprL2[Otorisasi SPV & Manajer]
    CheckValue -- >= Rp 100 Juta --> ApprL3[Otorisasi Direktur Keuangan / CFO]
    CheckValue -- >= Rp 500 Juta --> ApprL4[Otorisasi Direktur Utama / BOD]
    
    ApprL2 --> SendPO[Kirim PO ke Pemasok]
    ApprL3 --> SendPO
    ApprL4 --> SendPO
    
    SendPO --> ArriveGoods[Barang Tiba di Gudang]
    ArriveGoods --> ReceiveGRN[Inspeksi & Terbitkan Goods Receipt Note]
    ReceiveGRN --> AutoStock[Auto-Movement Stok Masuk IN_PURCHASE]
    
    AutoStock --> ArriveBill[Terima Faktur / Invoice dari Vendor]
    ArriveBill --> ThreeWayMatch{3-Way Match PO vs GRN vs Bill}
    
    ThreeWayMatch -- Selisih Kuantitas/Harga --> HoldBill[Tahan Tagihan & Minta Klarifikasi]
    HoldBill --> ArriveBill
    
    ThreeWayMatch -- Cocok & Valid --> PostBill[Posting Vendor Bill]
    PostBill --> AutoJournal[Auto-Posting Jurnal: Debit Persediaan/Beban, Kredit Utang AP]
    AutoJournal --> EndDone([Selesai Pengadaan])
```

---

## 4. Sequence Diagram (Goods Receipt Note & Stock Valuation)

```mermaid
sequenceDiagram
    autonumber
    actor Receiver as Staff Gudang Penerima
    participant UI as Purchasing Web (Next.js 15)
    participant Hdl as Purchasing Handler
    participant Svc as Purchasing Service
    participant Inv as Inventory Module
    participant Repo as Purchasing Repository
    participant DB as PostgreSQL 16

    Receiver->>UI: Input Penerimaan Barang dari PO
    UI->>Hdl: POST /api/v1/purchasing/receipts
    Hdl->>Svc: CreateGoodsReceipt(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: Lock PO & Validate Open Quantity
        Repo->>DB: SELECT * FROM purchase_orders WHERE id = ... FOR UPDATE
        
        Svc->>Inv: AddStock(branchID, warehouseID, itemID, qty, unitCost, "IN_PURCHASE", grnNumber)
        Inv->>DB: INSERT INTO stock_movements (IN_PURCHASE)
        Inv->>DB: UPDATE items SET balance_qty = balance_qty + qty, avg_cost = ...
        
        Svc->>Repo: Insert GRN & Lines
        Repo->>DB: INSERT INTO goods_receipt_notes ...
        Repo->>DB: INSERT INTO goods_receipt_note_items ...
        
        Svc->>Repo: Update PO Received Qty & Status
        Repo->>DB: UPDATE purchase_orders SET status = 'PARTIALLY_RECEIVED' / 'RECEIVED'
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: GRN Entity
    Hdl-->>UI: 201 Created
    UI-->>Receiver: Tampilkan Surat Penerimaan Barang (GRN) A4
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    VENDORS ||--o{ PURCHASE_ORDERS : "receives"
    VENDORS ||--o{ VENDOR_BILLS : "bills"
    PURCHASE_REQUISITIONS ||--|{ PURCHASE_REQUISITION_ITEMS : "contains"
    PURCHASE_REQUISITIONS ||--o{ PURCHASE_ORDERS : "converted_to"
    PURCHASE_ORDERS ||--|{ PURCHASE_ORDER_ITEMS : "contains"
    PURCHASE_ORDERS ||--o{ GOODS_RECEIPT_NOTES : "fulfilled_by"
    PURCHASE_ORDERS ||--o{ VENDOR_BILLS : "matched_to"
    GOODS_RECEIPT_NOTES ||--|{ GOODS_RECEIPT_NOTE_ITEMS : "contains"
    VENDOR_BILLS ||--|{ VENDOR_BILL_ITEMS : "contains"

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

    PURCHASE_REQUISITIONS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string pr_number UK
        date request_date
        date required_date
        string department
        string status "DRAFT | SUBMITTED | APPROVED | REJECTED"
        uuid requester_id FK
    }

    PURCHASE_ORDERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid vendor_id FK
        uuid pr_id FK
        string po_number UK
        date order_date
        date expected_delivery_date
        string status "DRAFT | CONFIRMED | RECEIVED | CANCELLED"
        numeric subtotal
        numeric tax_amount
        numeric grand_total
        uuid approval_id FK
    }

    GOODS_RECEIPT_NOTES {
        uuid id PK
        uuid purchase_order_id FK
        uuid branch_id FK
        uuid warehouse_id FK
        string grn_number UK
        date receipt_date
        string delivery_note_ref
        string status "RECEIVED | INSPECTED"
        uuid received_by FK
    }

    VENDOR_BILLS {
        uuid id PK
        uuid purchase_order_id FK
        uuid vendor_id FK
        string bill_number UK
        date bill_date
        date due_date
        numeric grand_total
        string payment_status "UNPAID | PARTIALLY_PAID | PAID"
        uuid journal_id FK
    }
```
