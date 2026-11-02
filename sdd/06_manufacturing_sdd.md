# 06. Software Design Document: Manufacturing Execution & Job Costing

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Manufacturing Execution & Job Costing** mengelola alur pabrikasi kelapa sawit dan manufaktur industri PT SAWIT JAYA (SWT): Struktur Formula Bahan Baku (*Bill of Materials / BOM*), Surat Perintah Kerja (*Work Orders / SPK*), Pusat Kerja (*Work Centers*), Rute Operasi (*Routings*), Pengeluaran Bahan Baku Lantai Pabrik (*Material Issues / Pick List* FEFO), Perhitungan Perencanaan Kebutuhan Bahan (*MRP-I / MRP-II Engine*), serta Kalkulasi Biaya Pesanan Aktual (*Job Order Costing*) dengan penyerapan biaya scrap.

- **Lokasi Backend**: `services/api/internal/modules/manufacturing/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/manufacturing/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    PPIC["Staf PPIC / Planner"]
    ProdSpv["Production Supervisor"]
    PlantMgr["Plant Manager"]
    CostAcct["Cost Accountant"]

    subgraph MfgModule ["Modul Manufacturing & Job Costing"]
        UC1["Kelola Formula & BOM Multi-Tier"]
        UC2["Rilis Surat Perintah Kerja (SPK: manufacturing:create)"]
        UC3["Pengeluaran Bahan Baku ke Lantai Pabrik (Pick List FEFO)"]
        UC4["Tracking Progres Operasi Stasiun Mesin (Routings)"]
        UC5["Pencatatan Penyelesaian Produksi & Scrap Absorption"]
        UC6["Eksekusi Mesin Perencanaan Bahan (MRP Runs)"]
        UC7["Kalkulasi HPP Aktual & Analisis Selisih Biaya (Variance)"]
    end

    PPIC --> UC1
    PPIC --> UC2
    PPIC --> UC6

    ProdSpv --> UC3
    ProdSpv --> UC4
    ProdSpv --> UC5

    PlantMgr --> UC2
    PlantMgr --> UC5

    CostAcct --> UC7
```

---

## 3. Activity Diagram (Siklus Eksekusi SPK, Otorisasi Branch, & Penyelesaian Produksi)

```mermaid
flowchart TD
    Start([Perencanaan Batch Pabrikasi]) --> CreateSPK[Staf Buat SPK Work Order]
    CreateSPK --> BindBranch[Sistem Enforce Authorized Branch & Warehouse Pabrik]
    BindBranch --> GenerateDocNo[Generate Nomor SPK Unik SPK/YYYY/MM/...]
    
    GenerateDocNo --> AutoRoutings[Salin Rute Stasiun Kerja dari BOM ke SPK Operations]
    AutoRoutings --> IssueMaterials[Terbitkan Pick List Bahan Baku Lot FEFO]
    
    IssueMaterials --> ProcessFloor[Eksekusi Produksi di Lantai Pabrik & Rekam Jam Kerja]
    ProcessFloor --> InspectQC[Inspeksi Mutu IPQC Kadar Asam Lemak Bebas / FFA]
    
    InspectQC --> CompleteOrder[Supervisor Panggil /work-orders/{id}/complete]
    CompleteOrder --> DeductComponents[Konsumsi Otomatis Bahan Baku dari stock_ledger]
    DeductComponents --> AbsorbScrap[Serap Biaya Scrap ke Nilai HPP Barang Jadi]
    DeductComponents --> IncreaseFG[Tambah Stok Barang Jadi pada Gudang Tujuan]
    
    IncreaseFG --> CalculateJobCost[Kalkulasi HPP Aktual: Material + Tenaga Kerja + Overhead]
    CalculateJobCost --> AutoGL[Auto-Posting Jurnal Pemindahan WIP ke Persediaan Barang Jadi]
    AutoGL --> EndDone([Produksi Batch Selesai])
```

---

## 4. Sequence Diagram (Penerbitan SPK & Pengikatan Fasilitas Pabrik)

```mermaid
sequenceDiagram
    autonumber
    actor Planner as PPIC Planner
    participant UI as Manufacturing Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Mfg Handler
    participant Svc as Mfg Service
    participant Repo as Mfg Repository
    participant DB as PostgreSQL 16

    Planner->>UI: Input SPK Baru (BOM, Warehouse Pabrik, Target Qty)
    UI->>MW: POST /api/manufacturing/work-orders (Bearer Token)
    MW->>MW: Verifikasi Authorized Branch (Anti-IDOR) & manufacturing:create
    MW->>Hdl: HandleCreateWorkOrder
    Hdl->>Svc: CreateWorkOrder
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Transaksi Database Terisolasi (pgx.Tx)
        Svc->>Repo: Validasi BOM & Ambil Branch Fisik dari Warehouse
        Repo->>DB: SELECT branch_id FROM warehouses WHERE id = $warehouseID
        
        Svc->>Repo: Generate Nomor SPK Atomik
        Repo->>DB: INSERT INTO work_orders (wo_number, status: PLANNED, branch_id)
        
        Svc->>Repo: Gandakan Operasi dari Master BOM Routings
        Repo->>DB: INSERT INTO work_order_operations (work_center_id, operation_name, step_number)
    end

    Svc-->>Hdl: Work Order Entity
    Hdl-->>UI: 201 Created (SPK Berhasil Diterbitkan)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    BOM_HEADERS ||--|{ BOM_LINES : "contains"
    BOM_HEADERS ||--o{ BOM_ROUTINGS : "defines"
    BOM_HEADERS ||--o{ WORK_ORDERS : "manufactured_by"
    WORK_CENTERS ||--o{ BOM_ROUTINGS : "performed_at"
    WORK_ORDERS ||--o{ WORK_ORDER_OPERATIONS : "tracks"
    WORK_CENTERS ||--o{ WORK_ORDER_OPERATIONS : "executes"
    WORK_ORDERS ||--o{ WORK_ORDER_MATERIAL_ISSUES : "consumes"

    BOM_HEADERS {
        uuid id PK
        uuid company_id FK
        uuid finished_good_id FK
        string bom_code UK
        string name
        numeric base_quantity
        boolean is_active
    }

    BOM_LINES {
        uuid id PK
        uuid bom_id FK
        uuid item_id FK
        numeric quantity
        numeric scrap_factor
    }

    WORK_CENTERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string code UK
        string name
        numeric hourly_cost_rate
        numeric capacity_hours_per_day
        boolean is_active
    }

    WORK_ORDERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid bom_id FK
        uuid warehouse_id FK
        string wo_number UK
        numeric target_quantity
        numeric completed_quantity
        numeric scrap_quantity
        string status "PLANNED | IN_PROGRESS | COMPLETED | CANCELLED"
        date start_date
        date end_date
        uuid created_by FK
    }

    WORK_ORDER_OPERATIONS {
        uuid id PK
        uuid work_order_id FK
        uuid work_center_id FK
        string operation_name
        integer step_number
        numeric progress_percent
        string status "PENDING | IN_PROGRESS | COMPLETED"
    }
```
