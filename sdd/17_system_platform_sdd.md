# 17. Software Design Document: System Platform, Storage & Audit Trail

## 1. Ringkasan Modul & Tanggung Jawab
Modul **System Platform & Audit Trail** mengelola fondasi infrastruktur perangkat lunak: Jejak Audit Forensik (*Immutable Audit Trail Diff Logger*), Pengantrian Tugas Asinkron (*Valkey / Asynq Background Queue*), Penyimpanan Berkas Berbasis S3 (*Floci Cloud Emulator Adapter*), Pengaturan Global Sistem (*System Settings*), dan Pemeriksaan Kesehatan Server (*Telemetry & Health Checks*).

- **Lokasi Backend**: `services/api/internal/platform/` & `services/api/internal/modules/system/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/settings/audit/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    SysAdmin["System Administrator"]
    Auditor["Lead Auditor / Compliance Officer"]
    Worker["Asynq Queue Worker (Daemon)"]

    subgraph PlatformModule ["Modul System Platform & Audit Trail"]
        UC1["Pencatatan Jejak Audit Mutasi (Diff Before & Diff After)"]
        UC2["Inspeksi Forensik Log Audit (Filter per Modul / Actor / Tanggal)"]
        UC3["Pengelolaan Berkas Terenkripsi S3 (Bukti POD, Tanda Tangan, Lampiran)"]
        UC4["Pemrosesan Tugas Asinkron (Generate PDF Laporan, Kirim Notifikasi)"]
        UC5["Konfigurasi Pengaturan Global Sistem (System Settings)"]
        UC6["Pemantauan Metrik Kesehatan Sistem (DB Pool, Memory, Uptime)"]
    end

    SysAdmin --> UC2
    SysAdmin --> UC5
    SysAdmin --> UC6

    Auditor --> UC2

    Worker --> UC4
```

---

## 3. Activity Diagram (Perekaman Jejak Audit Otomatis pada Mutasi Data)

```mermaid
flowchart TD
    Start([Eksekusi Mutasi API / CUD]) --> BeginTx[Buka Transaksi Database ACID]
    BeginTx --> FetchOld[SELECT Snapshot Data Lama / diff_before]
    FetchOld --> ApplyMutation[Eksekusi INSERT / UPDATE / DELETE Entitas]
    
    ApplyMutation --> GenerateDiff[Bandingkan State Lama vs State Baru / JSON Diff]
    GenerateDiff --> CaptureContext[Ambil Konteks: User ID, Branch ID, Client IP, Resource Type]
    
    CaptureContext --> InsertAudit[INSERT INTO audit_logs (Immutable Record)]
    InsertAudit --> CommitTx[Commit Transaksi Database]
    
    CommitTx --> IsAsyncJobNeeded{Apakah Perlu Background Task?}
    IsAsyncJobNeeded -- Ya --> EnqueueAsynq[Kirim Tugas ke Queue Valkey / Asynq]
    IsAsyncJobNeeded -- Tidak --> ReturnAPI
    
    EnqueueAsynq --> ReturnAPI[Kembalikan Respon Sukses ke Klien]
    ReturnAPI --> EndDone([Mutasi & Audit Selesai])
```

---

## 4. Sequence Diagram (Penyimpanan Berkas ke Floci S3 & Pengambilan Presigned URL)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Frontend Web Client
    participant Hdl as Storage Handler
    participant S3Adapter as Floci S3 Storage Adapter
    participant Floci as Local S3 Cloud Emulator
    participant Repo as System Repository
    participant DB as PostgreSQL 16

    Client->>Hdl: POST /api/v1/storage/upload (Multipart File: POD Photo / Signature)
    
    rect rgb(240, 248, 255)
        Hdl->>Hdl: Validasi Format MIME Type & Batas Ukuran File (Max 10MB)
        Hdl->>S3Adapter: Upload(ctx, s3Key, rawBytes, contentType)
        S3Adapter->>Floci: PutObject(Bucket, Key, Body)
        Floci-->>S3Adapter: S3 ETag & Upload Success
        
        Hdl->>S3Adapter: GetSignedURL(ctx, s3Key, 24 Hours Expiry)
        S3Adapter-->>Hdl: Presigned Download URL
        
        Hdl->>Repo: RecordFileMetadata(fileName, s3Key, sizeBytes, uploadedBy)
        Repo->>DB: INSERT INTO uploaded_files ...
    end

    Hdl-->>Client: 201 Created (s3_key, direct_url, presigned_url)
    Client->>Client: Tautkan s3_key ke Entitas Dokumen Terkait
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ AUDIT_LOGS : "performed_by"
    BRANCHES ||--o{ AUDIT_LOGS : "scoped_to"
    USERS ||--o{ UPLOADED_FILES : "uploaded_by"

    AUDIT_LOGS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid actor_id FK
        string actor_name
        string actor_role
        string action "CREATE | UPDATE | DELETE | POST | APPROVE | REJECT"
        string resource_type "JOURNAL | SALES_ORDER | PURCHASE_ORDER | EMPLOYEE | ASSET"
        string resource_id
        string client_ip
        string user_agent
        jsonb diff_before
        jsonb diff_after
        timestamp timestamp
    }

    SYSTEM_SETTINGS {
        string key PK
        string value
        string data_type "STRING | NUMBER | BOOLEAN | JSON"
        string description
        boolean is_encrypted
        timestamp updated_at
    }

    UPLOADED_FILES {
        uuid id PK
        uuid company_id FK
        string s3_key UK
        string original_file_name
        string content_type
        integer size_bytes
        uuid uploaded_by FK
        timestamp created_at
    }

    BACKGROUND_JOBS {
        uuid id PK
        string queue_name "default | critical | export"
        string payload_json
        string status "PENDING | PROCESSING | COMPLETED | FAILED"
        integer retry_count
        string error_message
        timestamp scheduled_at
        timestamp executed_at
    }
```
