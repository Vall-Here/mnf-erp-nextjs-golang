# 07. Software Design Document: Quality Assurance & ISO Control

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Quality Assurance & Control** mengelola standar mutu ISO 9001:2015: Inspeksi Bahan Masuk (*Incoming Quality Control / IQC*), Penerbitan Sertifikat Analisis Laboratorium (*Certificate of Analysis / CoA*), serta Pelaporan Ketidaksesuaian Mutu (*Non-Conformance Report / NCR*) dan Siklus Tindakan Korektif-Preventif (*CAPA 8D*).

- **Lokasi Backend**: `services/api/internal/modules/quality/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/quality/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    LabTech["Analis QC Lab"]
    QCLead["QA/QC Inspector Lead"]
    QAMgr["Manajer Kendali Mutu (QA Manager)"]
    Auditor["Auditor ISO 9001"]

    subgraph QualityModule ["Modul Quality Assurance & ISO Control"]
        UC1["Inspeksi Bahan Baku Masuk (IQC)"]
        UC2["Uji Laboratorium & Input Hasil Sampel"]
        UC3["Rilis Sertifikat Analisis (CoA) Produk Jadi"]
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
    
    ReleaseStock -- Rework Selesai --> ReleaseFG[Rilis Kembali ke Stok Siap Jual]
    ReleaseStock -- Scrap/Musnahkan --> WriteOff[Pencatatan Kerugian Pemusnahan]
    
    ReleaseFG --> GenCert[Cetak Laporan NCR & CAPA Action Sheet]
    WriteOff --> GenCert
    GenCert --> EndDone([Kasus Ditutup])
```

---

## 4. Sequence Diagram (Penerbitan Certificate of Analysis / CoA)

```mermaid
sequenceDiagram
    autonumber
    actor Lab as Analis QC Lab
    participant UI as Quality Web (Next.js 15)
    participant Hdl as Quality Handler
    participant Svc as Quality Service
    participant Repo as Quality Repository
    participant S3 as Storage Adapter (Floci)
    participant DB as PostgreSQL 16

    Lab->>UI: Input Parameter Uji & Hasil Uji Laboratorium
    UI->>Hdl: POST /api/v1/quality/coa
    Hdl->>Svc: IssueCertificateOfAnalysis(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Validate Batch/Lot & Product Specs
        Repo->>DB: SELECT * FROM item_lots WHERE id = ...
        
        Svc->>Svc: Evaluasi Hasil vs Batas Toleransi Standar ISO
        Svc->>Repo: Insert CoA Record & Test Parameter Lines
        Repo->>DB: INSERT INTO certificates_of_analysis ...
        Repo->>DB: INSERT INTO coa_parameters ...
        
        Svc->>S3: Generate Digital Certificate Hash & QR Seal
        Repo->>DB: UPDATE certificates_of_analysis SET status = 'RELEASED'
    end

    Svc-->>Hdl: CoA Entity
    Hdl-->>UI: 201 Created
    UI-->>Lab: Tampilkan Dokumen Certificate of Analysis (CoA A4 ISO)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    ITEMS ||--o{ QUALITY_INSPECTIONS : "inspected"
    ITEMS ||--o{ CERTIFICATES_OF_ANALYSIS : "certified_for"
    ITEM_LOTS ||--o{ CERTIFICATES_OF_ANALYSIS : "batch_of"
    CERTIFICATES_OF_ANALYSIS ||--|{ COA_PARAMETERS : "specifies"
    QUALITY_INSPECTIONS ||--o{ NON_CONFORMANCE_REPORTS : "triggers"
    NON_CONFORMANCE_REPORTS ||--o{ CAPA_ACTIONS : "remediated_by"

    QUALITY_INSPECTIONS {
        uuid id PK
        uuid item_id FK
        uuid lot_id FK
        string inspection_type "IQC | IPQC | OQC"
        numeric sample_size
        numeric passed_qty
        numeric failed_qty
        string result "PASS | FAIL | CONDITIONAL_PASS"
        uuid inspector_id FK
        timestamp inspected_at
    }

    CERTIFICATES_OF_ANALYSIS {
        uuid id PK
        uuid item_id FK
        uuid lot_id FK
        string coa_number UK
        date release_date
        string status "DRAFT | RELEASED | REVOKED"
        string verified_by
        timestamp created_at
    }

    COA_PARAMETERS {
        uuid id PK
        uuid coa_id FK
        string parameter_name
        string standard_specification
        string test_method
        string actual_result
        boolean is_conforming
    }

    NON_CONFORMANCE_REPORTS {
        uuid id PK
        uuid branch_id FK
        string ncr_number UK
        date incident_date
        string severity "MINOR | MAJOR | CRITICAL"
        string description
        string root_cause_analysis
        string disposition "REWORK | SCRAP | RETURN_TO_VENDOR"
        string status "OPEN | UNDER_INVESTIGATION | CAPA_ASSIGNED | CLOSED"
    }

    CAPA_ACTIONS {
        uuid id PK
        uuid ncr_id FK
        string action_type "CORRECTIVE | PREVENTIVE"
        string action_plan
        uuid assigned_to FK
        date target_due_date
        date completed_date
        string verification_notes
        string status "PENDING | IN_PROGRESS | COMPLETED | VERIFIED"
    }
```
