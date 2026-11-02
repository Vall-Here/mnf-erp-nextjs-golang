# 07. Software Design Document: Quality Assurance & ISO Control

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Quality Assurance & Control** mengelola standar mutu ISO 9001:2015 serta standar mutu industri pengolahan kelapa sawit PT SAWIT JAYA (SWT): Inspeksi Bahan Baku Masuk (*Incoming Quality Control / IQC* seperti Tandan Buah Segar / TBS), Inspeksi Proses Pabrikasi (*In-Process Quality Control / IPQC* kadar FFA, moisture, dan impurities), Penerbitan Sertifikat Analisis Laboratorium (*Certificate of Analysis / CoA* Crude Palm Oil - CPO & Palm Kernel), serta Pelaporan Ketidaksesuaian Mutu (*Non-Conformance Report / NCR*) dan Siklus Tindakan Korektif-Preventif (*CAPA 8D*).

- **Lokasi Backend**: `services/api/internal/modules/quality/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/quality/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    LabTech["Analis QC Lab"]
    QCLead["QA/QC Inspector Lead"]
    QAMgr["Manajer Kendali Mutu (QA Manager)"]
    Auditor["Auditor ISO 9001 & RSPO"]

    subgraph QualityModule ["Modul Quality Assurance & ISO Control"]
        UC1["Inspeksi Bahan Baku Masuk (IQC TBS & Material)"]
        UC2["Uji Laboratorium Sampel CPO (Kadar FFA, Air, Kotoran)"]
        UC3["Rilis Sertifikat Analisis (CoA) Pengapalan / Pengiriman CPO"]
        UC4["Pencatatan Insiden Mutu & Deviasi (NCR)"]
        UC5["Analisis Akar Masalah (Root Cause Analysis - 5 Why / Ishikawa)"]
        UC6["Penyusunan & Verifikasi Rencana Tindakan CAPA"]
        UC7["Penutupan Tiket NCR & CAPA Terotorisasi"]
    end

    LabTech --> UC1
    LabTech --> UC2

    QCLead --> UC3
    QCLead --> UC4
    QCLead --> UC5

    QAMgr --> UC6
    QAMgr --> UC7

    Auditor --> UC3
    Auditor --> UC4
    Auditor --> UC7
```

---

## 3. Activity Diagram (Siklus Penanganan Ketidaksesuaian Mutu NCR / CAPA)

```mermaid
flowchart TD
    Start([Temuan Deviasi / Produk Cacat]) --> LogNCR[Catat Tiket Non-Conformance Report]
    LogNCR --> QuarantineStock[Karantina Stok Terdampak di Gudang]
    QuarantineStock --> Investigate[Investigasi & Analisis Akar Masalah / RCA]
    
    Investigate --> FormulateCAPA[Susun Tindakan Korektif & Preventif / CAPA]
    FormulateCAPA --> AssignOwner[Tugaskan Penanggung Jawab & Tenggat Waktu]
    
    AssignOwner --> ExecuteAction[Pelaksanaan Tindakan di Lapangan]
    ExecuteAction --> InspectResult[Pemeriksaan Ulang Efektivitas Tindakan]
    
    InspectResult --> IsEffective{Apakah Masalah Teratasi?}
    IsEffective -- Belum Efektif --> ReEvaluate[Evaluasi Ulang Akar Masalah]
    ReEvaluate --> FormulateCAPA
    
    IsEffective -- Efektif & Memenuhi Standar --> ApproveClose[Otorisasi Penutupan Tiket oleh QA Manager]
    ApproveClose --> ReleaseStock{Keputusan Status Barang}
    
    ReleaseStock -- Rework Selesai --> ReleaseFG[Rilis Kembali ke Tangki / Stok Siap Jual]
    ReleaseStock -- Scrap/Musnahkan --> WriteOff[Pencatatan Kerugian Pemusnahan]
    
    ReleaseFG --> EndDone([Selesai])
    WriteOff --> EndDone
```

---

## 4. Sequence Diagram (Inspeksi Masuk IQC & Penolakan Mutu)

```mermaid
sequenceDiagram
    autonumber
    actor QC as Analis QC Lab
    participant UI as Quality Web (Next.js 15)
    participant Hdl as Quality Handler
    participant Svc as Quality Service
    participant Repo as Quality Repository
    participant DB as PostgreSQL 16

    QC->>UI: Input Hasil Uji Sampel Laboratorium
    UI->>Hdl: POST /api/quality/inspections
    Hdl->>Svc: CreateInspection(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Svc: Evaluasi Parameter Uji (FFA <= 3.5%, Moisture <= 0.15%)
        
        alt Nilai Melebihi Toleransi Standar
            Svc->>Repo: Set Status: REJECTED
            Repo->>DB: INSERT INTO quality_inspections (status: REJECTED, notes: DefectFound)
            Svc->>Repo: Otomatis Terbitkan Draft NCR
            Repo->>DB: INSERT INTO non_conformance_reports (severity: MAJOR)
        else Nilai Memenuhi Spesifikasi
            Svc->>Repo: Set Status: PASSED
            Repo->>DB: INSERT INTO quality_inspections (status: PASSED)
        end
        
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Inspection Result Entity
    Hdl-->>UI: 201 Created (Hasil Uji Mutu Terverifikasi)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    ITEMS ||--o{ QUALITY_INSPECTIONS : "inspected"
    QUALITY_INSPECTIONS ||--o{ NON_CONFORMANCE_REPORTS : "triggers"
    NON_CONFORMANCE_REPORTS ||--|{ CAPA_ACTIONS : "remedied_by"
    ITEMS ||--o{ CERTIFICATES_OF_ANALYSIS : "certified_by"

    QUALITY_INSPECTIONS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid item_id FK
        string inspection_type "IQC | IPQC | OQC"
        string reference_number
        numeric sample_size
        string status "PASSED | REJECTED"
        string test_parameters_json
        uuid inspector_id FK
        timestamp inspected_at
    }

    NON_CONFORMANCE_REPORTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string ncr_number UK
        string title
        string description
        string severity "CRITICAL | MAJOR | MINOR"
        string root_cause
        string status "OPEN | UNDER_REVIEW | RESOLVED | CLOSED"
        uuid reported_by FK
        uuid closed_by FK
    }

    CAPA_ACTIONS {
        uuid id PK
        uuid ncr_id FK
        string action_type "CORRECTIVE | PREVENTIVE"
        string action_plan
        uuid assigned_to FK
        date target_due_date
        date completed_date
        string status "PENDING | IN_PROGRESS | COMPLETED | VERIFIED"
    }

    CERTIFICATES_OF_ANALYSIS {
        uuid id PK
        uuid company_id FK
        string coa_number UK
        uuid item_id FK
        string lot_number
        string parameters_json
        uuid certified_by FK
        timestamp issued_at
    }
```

---

## 6. Cakupan Implementasi Terkini

- **Inspeksi IPQC Kelapa Sawit**: Pengujian sampel berkala pada tangki klarifikasi dan stasiun pengempaan untuk memastikan kadar asam lemak bebas (FFA) minyak sawit mentah (CPO) berada di bawah ambang batas legal ekspor (maksimal 3% s.d 5%).
- **Digital Certificate of Analysis (CoA)**: Format cetak resmi CoA berstandar industri ekspor dilengkapi kode verifikasi hash untuk pengapalan tangki CPO.
- **Integrasi dengan Modul Purchasing**: Kegagalan inspeksi bahan masuk (IQC) secara otomatis memicu opsi penerbitan Vendor Debit Note dan klaim penalti rafaksi harga tandan buah sawit.
