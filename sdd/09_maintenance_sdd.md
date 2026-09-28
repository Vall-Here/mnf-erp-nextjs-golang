# 09. Software Design Document: Plant Maintenance & CMMS

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Plant Maintenance & CMMS (Computerized Maintenance Management System)** mengelola keandalan mesin pabrik dan utilitas: Master Tag Aset Mesin (*Machine Assets*), Servis Terjadwal (*Preventive Maintenance / PM*), Penanganan Kerusakan Darurat (*Breakdown Work Orders / MWO*), Konsumsi Suku Cadang (*Spare Parts*), serta Analisis Keandalan (*MTBF / MTTR / Availability Rate*).

- **Lokasi Backend**: `services/api/internal/modules/maintenance/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/maintenance/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Tech["Teknisi / Mekanik"]
    SpvMaint["Supervisor Pemeliharaan"]
    PlantMgr["Kepala Pabrik (Plant Manager)"]
    ProdOperator["Operator Mesin Produksi"]

    subgraph CMMSModule ["Modul Plant Maintenance & CMMS"]
        UC1["Pendaftaran Tag Mesin & Lokasi Lini"]
        UC2["Penjadwalan Servis Preventif (PM)"]
        UC3["Lapor Kerusakan Mesin Mendadak (Breakdown)"]
        UC4["Penerbitan Surat Perintah Kerja Perbaikan (MWO)"]
        UC5["Pencatatan Alokasi & Pemakaian Suku Cadang"]
        UC6["Pencatatan Jam Henti Mesin (Downtime Hours)"]
        UC7["Otorisasi Penutupan Work Order & Pemulihan Mesin"]
        UC8["Pemantauan Metrik Keandalan (MTBF / MTTR)"]
    end

    ProdOperator --> UC3

    Tech --> UC1
    Tech --> UC2
    Tech --> UC5
    Tech --> UC6

    SpvMaint --> UC4
    SpvMaint --> UC7
    SpvMaint --> UC8

    PlantMgr --> UC7
    PlantMgr --> UC8
```

---

## 3. Activity Diagram (Siklus Penanganan Breakdown & Pemulihan Mesin)

```mermaid
flowchart TD
    Start([Insiden Kerusakan Mesin]) --> ReportFault[Operator Lapor Kerusakan di Lini]
    ReportFault --> CreateMWO[Sistem Terbitkan MWO Status DRAFT]
    CreateMWO --> AssignTech[Tugaskan Teknisi & Tentukan Prioritas TINGGI/DARURAT]
    
    AssignTech --> LockoutTagout[Pemasangan Safety LOTO / Karantina Mesin]
    LockoutTagout --> ChangeStatus[Ubah Status Mesin: DALAM_PERBAIKAN]
    
    ChangeStatus --> Diagnose[Teknisi Diagnosa Akar Masalah]
    Diagnose --> NeedParts{Butuh Ganti Suku Cadang?}
    
    NeedParts -- Ya --> RequestParts[Ambil Suku Cadang dari Gudang Spareparts]
    RequestParts --> RecordPartsUsage[Catat Konsumsi Part pada MWO]
    RecordPartsUsage --> RepairAction
    
    NeedParts -- Tidak --> RepairAction[Lakukan Perbaikan & Penyetelan Mekanikal/Elektrikal]
    
    RepairAction --> RunTest[Uji Coba Pengoperasian (Run-Test 30 Menit)]
    RunTest --> IsNormal{Mesin Beroperasi Normal?}
    
    IsNormal -- Belum Normal --> Diagnose
    IsNormal -- Normal --> RecordDowntime[Catat Total Jam Henti Lini / Downtime Hours]
    
    RecordDowntime --> CompleteMWO[Submit Penyelesaian MWO & Tindakan Korektif]
    CompleteMWO --> SpvVerify[Verifikasi oleh Supervisor Maintenance]
    SpvVerify --> CloseWO[Otorisasi Selesai / Status MESIN_BEROPERASI]
    CloseWO --> EndDone([Pemeliharaan Selesai])
```

---

## 4. Sequence Diagram (Penyelesaian Work Order & Pengembalian Status Operasional)

```mermaid
sequenceDiagram
    autonumber
    actor Tech as Teknisi Mekanik
    participant UI as Maintenance Web (Next.js 15)
    participant Hdl as Maintenance Handler
    participant Svc as Maintenance Service
    participant Repo as Maintenance Repository
    participant DB as PostgreSQL 16

    Tech->>UI: Input Tindakan Perbaikan, Jam Downtime & Suku Cadang
    UI->>Hdl: POST /api/v1/maintenance/work-orders/{id}/complete
    Hdl->>Svc: CompleteWorkOrder(ctx, id, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: Update MWO (corrective_action, root_cause, downtime_hours, status = 'COMPLETED')
        Repo->>DB: UPDATE maintenance_work_orders SET status = 'COMPLETED', ... WHERE id = ...
        
        Svc->>Repo: Update Machine Asset Status
        Repo->>DB: UPDATE maintenance_assets SET status = 'OPERATIONAL', last_serviced_at = NOW() WHERE id = ...
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Completed MWO Entity
    Hdl-->>UI: 200 OK
    UI-->>Tech: Notifikasi Sukses & Tampilkan Dokumen MWO Berstatus OPERATIONAL
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    MAINTENANCE_ASSETS ||--o{ PM_SCHEDULES : "scheduled_for"
    MAINTENANCE_ASSETS ||--o{ MAINTENANCE_WORK_ORDERS : "serviced_in"
    PM_SCHEDULES ||--|{ PM_CHECKLIST_ITEMS : "has"
    MAINTENANCE_WORK_ORDERS ||--o{ MWO_SPARE_PARTS : "replaces"

    MAINTENANCE_ASSETS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string asset_tag UK
        string machine_name
        string model_number
        string serial_number
        string location
        date install_date
        string status "OPERATIONAL | UNDER_MAINTENANCE | STANDBY | SCRAPPED"
        timestamp last_serviced_at
    }

    PM_SCHEDULES {
        uuid id PK
        uuid asset_id FK
        string task_title
        string frequency_type "DAILY | WEEKLY | MONTHLY | SEMESTER | YEARLY"
        integer frequency_value
        date next_due_date
        string status "SCHEDULED | OVERDUE | COMPLETED"
    }

    PM_CHECKLIST_ITEMS {
        uuid id PK
        uuid pm_schedule_id FK
        string item_description
        integer sequence_order
        boolean is_mandatory
    }

    MAINTENANCE_WORK_ORDERS {
        uuid id PK
        uuid asset_id FK
        string wo_number UK
        string wo_type "PREVENTIVE | BREAKDOWN_EMERGENCY | CALIBRATION"
        string priority "RENDAH | SEDANG | TINGGI | DARURAT"
        string issue_reported
        string root_cause
        string corrective_action
        numeric downtime_hours
        string technician_name
        string status "DRAFT | IN_PROGRESS | COMPLETED | CANCELLED"
        timestamp created_at
    }

    MWO_SPARE_PARTS {
        uuid id PK
        uuid work_order_id FK
        uuid item_id FK
        string part_name
        string part_number
        numeric quantity
        numeric unit_cost
    }
```
