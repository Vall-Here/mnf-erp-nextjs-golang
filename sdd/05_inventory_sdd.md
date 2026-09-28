# 05. Software Design Document: Inventory & Warehouse Management

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Inventory & Warehouse Management** mengelola pergerakan fisik dan valuasi persediaan (*Perpetual Inventory System*). Setiap mutasi stok (masuk, keluar, transfer, penyesuaian) wajib mencatat histori permanen yang tidak dapat diubah (*immutable ledger record*).

- **Lokasi Backend**: `services/api/internal/modules/inventory/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/inventory/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    WhKeeper["Staf Gudang (Warehouse Keeper)"]
    WhLead["Kepala Gudang"]
    Auditor["Internal Auditor"]

    subgraph InventoryModule ["Modul Inventory & Warehouse"]
        UC1["Kelola Master Barang & Satuan (UOM)"]
        UC2["Manajemen Lokasi Rak / Bin Gudang"]
        UC3["Pelacakan Batch / Lot & Rekomendasi FEFO"]
        UC4["Mutasi Stok Barang Masuk / Keluar"]
        UC5["Transfer Stok Antar Gudang / Cabang"]
        UC6["Pelaksanaan Stock Opname Fisik"]
        UC7["Penyesuaian Persediaan (Stock Adjustment)"]
        UC8["Pemantauan Reorder Point & Safety Stock"]
    end

    WhKeeper --> UC1
    WhKeeper --> UC2
    WhKeeper --> UC3
    WhKeeper --> UC4

    WhLead --> UC5
    WhLead --> UC7
    WhLead --> UC8

    Auditor --> UC6
    Auditor --> UC7
```

---

## 3. Activity Diagram (Transfer Stok Antar Gudang & Konfirmasi)

```mermaid
flowchart TD
    Start([Inisiasi Mutasi Stok]) --> CreateTransfer[Buat Dokumen Stock Transfer]
    CreateTransfer --> CheckAvail{Cek Saldo di Gudang Asal}
    
    CheckAvail -- Saldo Kurang --> ShowError[Tolak: Saldo Barang Tidak Mencukupi]
    ShowError --> EndFail([Selesai Batal])
    
    CheckAvail -- Saldo Cukup --> Dispatch[Keluarkan Barang dari Gudang Asal]
    Dispatch --> RecordOut[Catat Mutasi: TRANSFER_OUT & Status IN_TRANSIT]
    RecordOut --> DeductOrigin[Kurangi Saldo Gudang Asal]
    
    DeductOrigin --> Transit[Barang Dalam Perjalanan Ekspedisi]
    Transit --> ArriveDest[Barang Tiba di Gudang Tujuan]
    
    ArriveDest --> InspectGoods[Pemeriksaan Fisik Kuantitas & Kondisi]
    InspectGoods --> IsDamaged{Ada Selisih / Kerusakan?}
    
    IsDamaged -- Ya --> RecordDiscrepancy[Catat Berita Acara Kerusakan / Selisih]
    RecordDiscrepancy --> ReceiveGoods
    
    IsDamaged -- Tidak --> ReceiveGoods[Konfirmasi Penerimaan di Gudang Tujuan]
    ReceiveGoods --> RecordIn[Catat Mutasi: TRANSFER_IN & Status COMPLETED]
    RecordIn --> AddDest[Tambah Saldo Gudang Tujuan]
    AddDest --> EndDone([Transfer Selesai])
```

---

## 4. Sequence Diagram (Pencatatan Mutasi Stok Perpetual)

```mermaid
sequenceDiagram
    autonumber
    actor Caller as Modul Transaksi (Sales / Purchasing / Mfg)
    participant Svc as Inventory Service
    participant Repo as Inventory Repository
    participant DB as PostgreSQL 16

    Caller->>Svc: RecordStockMovement(input)
    
    rect rgb(245, 245, 245)
        Note over Svc,DB: Atomic Ledger Execution
        Svc->>Repo: Lock Item Record
        Repo->>DB: SELECT * FROM items WHERE id = ... FOR UPDATE
        
        Svc->>Svc: Hitung Saldo Akhir Baru (Balance Quantity)
        Svc->>Svc: Hitung Biaya Rata-Rata Baru (Moving Average Cost)
        
        Svc->>Repo: Insert Immutable Movement
        Repo->>DB: INSERT INTO stock_movements (item_id, warehouse_id, movement_type, qty, unit_cost, total_cost, balance_quantity)
        
        Svc->>Repo: Update Item Balance
        Repo->>DB: UPDATE items SET balance_qty = ..., avg_cost = ..., total_valuation = ...
    end

    Svc-->>Caller: Movement Confirmation & New Balance
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    WAREHOUSES ||--o{ BIN_LOCATIONS : "contains"
    WAREHOUSES ||--o{ STOCK_MOVEMENTS : "stores"
    ITEMS ||--o{ ITEM_LOTS : "tracked_by"
    ITEMS ||--o{ STOCK_MOVEMENTS : "records"
    ITEMS ||--o{ STOCK_TRANSFERS : "transferred"
    ITEMS ||--o{ STOCK_ADJUSTMENTS : "adjusted"
    BIN_LOCATIONS ||--o{ ITEM_LOTS : "houses"

    ITEMS {
        uuid id PK
        uuid company_id FK
        string sku UK
        string barcode
        string name
        string item_type "RAW_MATERIAL | WIP | FINISHED_GOOD | SPAREPART"
        string cost_method "AVERAGE | FIFO"
        numeric purchase_price
        numeric sale_price
        numeric min_stock_level
        numeric balance_qty
        numeric avg_cost
        numeric total_valuation
    }

    WAREHOUSES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string code UK
        string name
        string address
        boolean is_active
    }

    BIN_LOCATIONS {
        uuid id PK
        uuid warehouse_id FK
        string bin_code UK
        string rack
        string shelf
        string bin
        numeric max_weight_kg
        numeric max_volume_cbm
    }

    ITEM_LOTS {
        uuid id PK
        uuid item_id FK
        string lot_number UK
        date manufacture_date
        date expiry_date
        numeric initial_qty
        numeric current_qty
        string status "ACTIVE | QUARANTINE | EXPIRED"
    }

    STOCK_MOVEMENTS {
        uuid id PK
        uuid branch_id FK
        uuid warehouse_id FK
        uuid item_id FK
        string movement_type "IN_PURCHASE | OUT_SALE | TRANSFER_IN | TRANSFER_OUT | ADJUSTMENT"
        string reference_id
        numeric quantity
        numeric unit_cost
        numeric total_cost
        numeric balance_quantity
        timestamp created_at
    }
```
