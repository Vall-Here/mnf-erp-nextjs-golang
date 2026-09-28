# 06. Software Design Document: Manufacturing Execution & Job Costing

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Manufacturing Execution & Job Costing** mengelola alur pabrikasi: Struktur Bahan Baku (*Bill of Materials / BOM*), Surat Perintah Kerja (*Work Orders / SPK*), Pengeluaran Bahan Baku ke Lantai Produksi (*Material Issues / Pick List*) dengan metode FEFO (*First-Expired, First-Out*), serta Kalkulasi Biaya Pesanan Aktual (*Job Order Costing*).

- **Lokasi Backend**: `services/api/internal/modules/manufacturing/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/manufacturing/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    PPIC["Staf PPIC / Planner"]
    ProdSpv["Supervisor Produksi"]
    CostAcct["Akuntan Biaya (Costing)"]
    PlantMgr["Kepala Pabrik (Plant Manager)"]

    subgraph MfgModule ["Modul Manufacturing & Job Costing"]
        UC1["Kelola Formula & Struktur BOM Multi-Level"]
        UC2["Rilis Perintah Kerja Produksi (Work Order / SPK)"]
        UC3["Pengeluaran Bahan Baku (Pick List Lot FEFO)"]
        UC4["Pencatatan Hasil Produksi & Scrap / Reject"]
        UC5["Alokasi Biaya Tenaga Kerja & Overhead (BOP)"]
        UC6["Analisis Selisih Biaya (Job Cost Variance)"]
        UC7["Otorisasi Penyelesaian Batch Produksi"]
    end

    PPIC --> UC1
    PPIC --> UC2

    ProdSpv --> UC3
    ProdSpv --> UC4

    CostAcct --> UC5
    CostAcct --> UC6

    PlantMgr --> UC7
```

---

## 3. Activity Diagram (Siklus Eksekusi Pabrikasi & Job Costing)

```mermaid
flowchart TD
    Start([Perencanaan Produksi]) --> CreateBOM[Buat / Pilih Struktur BOM]
    CreateBOM --> IssueWO[Rilis Work Order SPK & Target Jadwal]
    IssueWO --> CheckMat{Ketersediaan Bahan Baku}
    
    CheckMat -- Bahan Kurang --> TriggerPR[Terbitkan PR Pengadaan Bahan]
    TriggerPR --> WaitMat[Tunggu Penerimaan Bahan di Gudang]
    WaitMat --> CheckMat
    
    CheckMat -- Bahan Lengkap --> PickFEFO[Pengeluaran Bahan Baku Sistem FEFO]
    PickFEFO --> PrintPickList[Cetak Slip Pengeluaran Bahan Baku]
    
    PrintPickList --> ProdRun[Proses Produksi di Lantai Pabrik]
    ProdRun --> TrackOutput[Catat Output Barang Jadi & Jam Mesin/Buruh]
    
    TrackOutput --> QCPass{Inspeksi Mutu QC Lolos?}
    QCPass -- Gagal / Cacat --> Scrap[Catat Kuantitas Scrap / Reject]
    Scrap --> CalcCost
    
    QCPass -- Lolos --> ReceiveFG[Penerimaan Barang Jadi ke Gudang FG]
    ReceiveFG --> CalcCost[Kalkulasi Biaya Aktual HPP]
    
    CalcCost --> BreakCost[Hitung: Direct Materials + Direct Labor + Overhead BOP]
    BreakCost --> CompareStd[Bandingkan dengan Standar BOM & Varians]
    
    CompareStd --> GenCostSheet[Cetak Lembar Biaya Pesanan Pabrikasi]
    GenCostSheet --> AutoJournal[Auto-Posting Jurnal: Debit Persediaan FG, Kredit WIP]
    AutoJournal --> EndDone([Selesai Siklus Produksi])
```

---

## 4. Sequence Diagram (Pengeluaran Bahan Baku & Rekomendasi FEFO)

```mermaid
sequenceDiagram
    autonumber
    actor Spv as Supervisor Produksi
    participant UI as Manufacturing Web (Next.js 15)
    participant Hdl as Mfg Handler
    participant Svc as Mfg Service
    participant Inv as Inventory Module
    participant Repo as Mfg Repository
    participant DB as PostgreSQL 16

    Spv->>UI: Request Pengeluaran Bahan Baku untuk WO
    UI->>Hdl: POST /api/v1/manufacturing/issues
    Hdl->>Svc: IssueMaterials(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: GetWorkOrder(woID)
        Repo->>DB: SELECT * FROM work_orders WHERE id = ...
        
        loop Tiap Komponen Bahan Baku
            Svc->>Inv: GetFEFOLots(itemID, requiredQty)
            Inv->>DB: SELECT * FROM item_lots WHERE item_id = ... ORDER BY expiry_date ASC
            DB-->>Inv: Sorted Active Lots
            
            Svc->>Inv: DeductLotStock(lotID, allocatedQty, "OUT_MFG", woNumber)
            Inv->>DB: UPDATE item_lots SET current_qty = current_qty - allocatedQty
            Inv->>DB: INSERT INTO stock_movements (OUT_MFG)
            
            Svc->>Repo: RecordMaterialIssueLine(woID, itemID, lotID, qty, cost)
            Repo->>DB: INSERT INTO work_order_material_issues ...
        end
        
        Svc->>Repo: Update WO Actual Material Cost
        Repo->>DB: UPDATE work_orders SET actual_material_cost = ...
    end

    Svc-->>Hdl: Issue Result & Lot Allocations
    Hdl-->>UI: 201 Created
    UI-->>Spv: Tampilkan Slip Pengeluaran Bahan Baku (Pick List A4)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    ITEMS ||--o{ BILLS_OF_MATERIAL : "as_finished_good"
    BILLS_OF_MATERIAL ||--|{ BOM_ITEMS : "composed_of"
    ITEMS ||--o{ BOM_ITEMS : "as_raw_material"
    BILLS_OF_MATERIAL ||--o{ WORK_ORDERS : "manufactured_by"
    WORK_ORDERS ||--o{ WORK_ORDER_MATERIAL_ISSUES : "consumes"
    WORK_ORDERS ||--o| WORK_ORDER_COSTINGS : "costed_in"

    BILLS_OF_MATERIAL {
        uuid id PK
        uuid finished_good_id FK
        string bom_code UK
        string name
        numeric base_quantity
        numeric standard_cost
        boolean is_active
    }

    BOM_ITEMS {
        uuid id PK
        uuid bom_id FK
        uuid component_item_id FK
        numeric quantity
        numeric scrap_tolerance_percent
    }

    WORK_ORDERS {
        uuid id PK
        uuid bom_id FK
        uuid branch_id FK
        string wo_number UK
        date start_date
        date due_date
        numeric planned_qty
        numeric completed_qty
        numeric rejected_qty
        string status "DRAFT | RELEASED | IN_PROGRESS | COMPLETED | CANCELLED"
        numeric actual_material_cost
        numeric actual_labor_cost
        numeric actual_overhead_cost
    }

    WORK_ORDER_MATERIAL_ISSUES {
        uuid id PK
        uuid work_order_id FK
        uuid item_id FK
        uuid lot_id FK
        uuid bin_id FK
        numeric quantity_issued
        numeric unit_cost
        timestamp issued_at
    }

    WORK_ORDER_COSTINGS {
        uuid id PK
        uuid work_order_id FK
        numeric total_material_cost
        numeric total_labor_cost
        numeric total_overhead_cost
        numeric total_actual_cost
        numeric standard_cost_variance
        string variance_type "FAVORABLE | UNFAVORABLE"
        uuid journal_id FK
    }
```
