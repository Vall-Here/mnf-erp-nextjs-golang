# 05. Software Design Document: Inventory & Warehouse Management

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Inventory & Warehouse Management** mengelola pergerakan fisik barang dan valuasi persediaan terpadu (*Perpetual Inventory System*) pada seluruh fasilitas gudang dan pabrik PT SAWIT JAYA (SWT). Setiap mutasi stok (masuk, keluar, transfer antar-gudang, dan penyesuaian opname) mencatat transaksi permanen yang tidak dapat diubah (*immutable ledger record*) pada `stock_ledger` serta memperbarui ringkasan saldo saldo real-time pada `item_stock_summary`.

- **Lokasi Backend**: `services/api/internal/modules/inventory/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/inventory/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    WhStaff["Staf Gudang (Material Handler)"]
    WhSpv["Warehouse Supervisor"]
    PlantMgr["Plant Manager"]
    Auditor["Internal Auditor"]

    subgraph InventoryModule ["Modul Inventory & Multi-Warehouse"]
        UC1["Katalog Master Barang & Satuan (UOM)"]
        UC2["Manajemen Penataan Bin & Rak Gudang"]
        UC3["Pelacakan Lot / Batch & Rekomendasi FEFO"]
        UC4["Surat Jalan Transfer Antar-Gudang (STO)"]
        UC5["Konfirmasi Penerimaan Transfer di Gudang Tujuan"]
        UC6["Pencatatan Fisik Stock Opname (Opname Sheet)"]
        UC7["Otorisasi Penyesuaian Stok (inventory:approve)"]
        UC8["Pemantauan Saldo Minimum & Reorder Point"]
    end

    WhStaff --> UC1
    WhStaff --> UC2
    WhStaff --> UC3
    WhStaff --> UC4
    WhStaff --> UC6

    WhSpv --> UC5
    WhSpv --> UC7
    WhSpv --> UC8

    PlantMgr --> UC7
    Auditor --> UC6
    Auditor --> UC7
```

---

## 3. Activity Diagram (Alur Stock Opname Fisik & SoD Approval)

```mermaid
flowchart TD
    Start([Jadwal Stock Opname Berkala]) --> CountStock[Staf Gudang Menghitung Fisik Barang di Rak]
    CountStock --> CreateAdjustment[Staf Gudang Input Draft Stock Adjustment]
    CreateAdjustment --> FetchSystemQty[Sistem Mengunci Baris Item & Hitung Selisih: Fisik - Sistem]
    
    FetchSystemQty --> SaveDraft[Simpan Berita Acara Status: DRAFT]
    SaveDraft --> IsVarianceZero{Apakah Selisih Kuantitas == 0?}
    
    IsVarianceZero -- Ya: Sesuai --> AutoClose[Tutup Opname: Status COMPLETED Tanpa Jurnal]
    AutoClose --> EndDone([Selesai])
    
    IsVarianceZero -- Tidak: Ada Selisih --> RequestSpv[Eskalasi ke Warehouse Supervisor / Plant Manager]
    RequestSpv --> ReviewVariance[Supervisor Meninjau Berita Acara & Penyebab Selisih]
    
    ReviewVariance --> CanApprove{Memiliki Izin inventory:approve & Bukan Pembuat Draft?}
    CanApprove -- Tidak / Staf Gudang --> Reject403[403 Forbidden: Pelanggaran Segregation of Duties]
    Reject403 --> EndFail([Otorisasi Ditolak])
    
    CanApprove -- Ya: Supervisor Sah --> ApproveAdjustment[Eksekusi Endpoint /adjustments/{id}/approve]
    ApproveAdjustment --> WriteLedger[Catat Mutasi Penyesuaian ke stock_ledger]
    WriteLedger --> UpdateSummary[Update Saldo Akhir item_stock_summary]
    UpdateSummary --> AutoJournal[Terbitkan Jurnal Penyesuaian Persediaan ke GL]
    AutoJournal --> EndDone
```

---

## 4. Sequence Diagram (Surat Jalan Transfer Antar-Gudang / STO)

```mermaid
sequenceDiagram
    autonumber
    actor Dispatcher as Staf Gudang Asal
    actor Receiver as Staf Gudang Tujuan
    participant UI as Inventory Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Inventory Handler
    participant Svc as Inventory Service
    participant Repo as Inventory Repository
    participant DB as PostgreSQL 16

    Dispatcher->>UI: Buat Surat Jalan Transfer Antar-Gudang (STO)
    UI->>MW: POST /api/inventory/transfers (Bearer Token)
    MW->>MW: Verifikasi Authorized Branch & Izin inventory:create
    MW->>Hdl: HandleCreateTransfer
    Hdl->>Svc: CreateStockTransfer
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Mutasi Pengeluaran dari Gudang Asal (pgx.Tx)
        Svc->>Repo: Kunci Baris Item & Cek Saldo Gudang Asal
        Repo->>DB: SELECT balance_quantity FROM item_stock_summary WHERE warehouse_id = $source FOR UPDATE
        
        Svc->>Repo: Rekam Mutasi Pengeluaran
        Repo->>DB: INSERT INTO stock_ledger (movement_type: TRANSFER_OUT, warehouse_id: $source)
        Repo->>DB: UPDATE item_stock_summary SET balance_quantity = balance_quantity - $qty WHERE warehouse_id = $source
        
        Svc->>Repo: Terbitkan Dokumen STO Status: IN_TRANSIT
        Repo->>DB: INSERT INTO stock_transfers (transfer_number, status: IN_TRANSIT)
    end

    Svc-->>Hdl: Transfer Entity
    Hdl-->>UI: 201 Created (Surat Jalan Transfer Berhasil Diterbitkan)

    Note over Dispatcher,Receiver: Barang Dikirim Melalui Armada Ekspedisi Logistik

    Receiver->>UI: Terima & Verifikasi Fisik Barang di Gudang Tujuan
    UI->>MW: POST /api/inventory/transfers/{id}/receive (Bearer Token)
    MW->>Hdl: HandleReceiveTransfer
    Hdl->>Svc: ReceiveStockTransfer
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Mutasi Penerimaan di Gudang Tujuan (pgx.Tx)
        Svc->>Repo: Rekam Mutasi Masuk
        Repo->>DB: INSERT INTO stock_ledger (movement_type: TRANSFER_IN, warehouse_id: $dest)
        Repo->>DB: UPDATE item_stock_summary SET balance_quantity = balance_quantity + $qty WHERE warehouse_id = $dest
        Repo->>DB: UPDATE stock_transfers SET status = 'COMPLETED', received_by = $receiverID
    end

    Svc-->>Hdl: Completed Transfer Entity
    Hdl-->>UI: 200 OK (Stok Fisik Gudang Tujuan Bertambah)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    BRANCHES ||--o{ WAREHOUSES : "houses"
    WAREHOUSES ||--o{ BIN_LOCATIONS : "contains"
    WAREHOUSES ||--o{ STOCK_LEDGER : "stores"
    WAREHOUSES ||--o{ ITEM_STOCK_SUMMARY : "aggregates"
    ITEMS ||--o{ ITEM_LOTS : "tracked_by"
    ITEMS ||--o{ STOCK_LEDGER : "records"
    ITEMS ||--o{ ITEM_STOCK_SUMMARY : "summarizes"
    ITEMS ||--o{ STOCK_TRANSFERS : "transferred"
    ITEMS ||--o{ STOCK_ADJUSTMENTS : "adjusted"

    WAREHOUSES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string code UK
        string name
        string address
        boolean is_active
    }

    ITEMS {
        uuid id PK
        uuid company_id FK
        string sku UK
        string barcode
        string name
        string item_type "RAW_MATERIAL | WIP | FINISHED_GOOD | SPAREPART"
        numeric purchase_price
        numeric sale_price
        numeric min_stock_level
        boolean is_active
    }

    STOCK_LEDGER {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid warehouse_id FK
        uuid item_id FK
        string movement_type "IN_PURCHASE | OUT_SALE | TRANSFER_IN | TRANSFER_OUT | ADJUSTMENT"
        string reference_number
        numeric quantity
        numeric unit_cost
        numeric total_cost
        numeric balance_quantity
        timestamp created_at
    }

    ITEM_STOCK_SUMMARY {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid warehouse_id FK
        uuid item_id FK
        numeric balance_quantity
        timestamp updated_at
    }

    STOCK_TRANSFERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string transfer_number UK
        uuid source_warehouse_id FK
        uuid target_warehouse_id FK
        uuid item_id FK
        numeric quantity
        string status "DRAFT | IN_TRANSIT | COMPLETED | CANCELLED"
        uuid created_by FK
        uuid received_by FK
    }

    STOCK_ADJUSTMENTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string adjustment_number UK
        uuid warehouse_id FK
        uuid item_id FK
        numeric system_quantity
        numeric physical_quantity
        numeric variance_quantity
        numeric unit_cost
        numeric total_variance_cost
        string reason
        string status "DRAFT | APPROVED | REJECTED"
        uuid created_by FK
        uuid approved_by FK
    }
```
