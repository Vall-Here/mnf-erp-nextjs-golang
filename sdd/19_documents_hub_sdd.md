# 19. Software Design Document: Corporate Documents Hub

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Corporate Documents Hub** mengelola tata naskah dinas dan surat keputusan administratif perusahaan: penerbitan dokumen resmi, penomoran surat, register agenda keluar, penandatanganan berjenjang, tanda air (*watermark*), serta pengarsipan berkas digital ke penyimpanan S3-compatible. Dokumen komersial dan transaksional seperti PO, SO, delivery order, SPK, dan invoice tetap berada di modul bisnis masing-masing.

- **Lokasi Backend**: `services/api/internal/modules/documents/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/documents/`
- **Tabel Utama**: `corporate_documents`
- **Endpoint**: `GET /api/documents`, `GET /api/documents/{id}`, `POST /api/documents`, `PATCH /api/documents/{id}/status`
- **Permission**: `documents:view`, `documents:create`, `documents:edit`

Jenis dokumen yang didukung: `SK`, `ST`, `PAKLARING`, `SP`, `BAST`, `SKU`, `MEMO`, `SKET`.
Kategori: `LEGAL`, `HR`, `OPERASIONAL`, `EKSEKUTIF`, `UMUM`.
Status: `DRAFT`, `ISSUED`, `ARCHIVED`, `VOID`.

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Staff["Staf Administrasi / HR"]
    Manager["Kepala Departemen"]
    Archivist["Pengelola Arsip"]
    Auditor["Auditor Internal"]

    subgraph DocumentsModule ["Modul Corporate Documents Hub"]
        UC1["Menerbitkan Dokumen Resmi Baru (SK, ST, BAST, Paklaring)"]
        UC2["Menelusuri Register Surat Berdasarkan Nomor, Jenis, Status"]
        UC3["Melihat Detail Dokumen dan Riwayat Tanda Tangan"]
        UC4["Mengubah Status Dokumen (DRAFT, ISSUED, ARCHIVED, VOID)"]
        UC5["Melampirkan Berkas Resmi ke Penyimpanan S3"]
        UC6["Mencetak Ulang Dokumen A4 dengan Tanda Air"]
    end

    Staff --> UC1
    Staff --> UC2
    Staff --> UC3
    Staff --> UC5

    Manager --> UC4
    Manager --> UC6

    Archivist --> UC2
    Archivist --> UC4
    Archivist --> UC6

    Auditor --> UC2
    Auditor --> UC3
```

---

## 3. Activity Diagram (Penerbitan dan Pengarsipan Dokumen Resmi)

```mermaid
flowchart TD
    Start([Staf Membuka Menu Documents Hub]) --> ChooseType[Pilih Jenis Dokumen: SK, ST, PAKLARING, SP, BAST, SKU, MEMO, SKET]
    ChooseType --> FillMeta[Isi Nomor Surat, Judul, Kategori, Tingkat Kerahasiaan, Penerima]
    FillMeta --> FillBody[Isi Pembuka, Isi Penutup, Catatan, dan Daftar Item Bila Ada]

    FillBody --> UploadOpt{Apakah Ada Lampiran Berkas?}
    UploadOpt -- Ya --> UploadS3[Unggah Berkas ke S3 dan Simpan attachment_s3_key]
    UploadOpt -- Tidak --> Signature

    UploadS3 --> Signature[Lengkapi Blok Tanda Tangan: Prepared, Reviewed, Approved]
    Signature --> Validate{Validasi Nomor Unik dan Tanggal?}
    Validate -- Tidak Valid --> ShowError[Tampilkan Kesalahan Validasi]
    ShowError --> FillMeta

    Validate -- Valid --> InsertDoc[INSERT corporate_documents dengan Status Awal]
    InsertDoc --> Audit[Catat Audit Trail Mutasi]
    Audit --> Register[Tampilkan Dokumen pada Register Surat]
    Register --> PrintOpt{Perlu Cetak Ulang?}
    PrintOpt -- Ya --> RenderA4[Render Dokumen A4 dengan Tanda Air]
    PrintOpt -- Tidak --> EndDone([Dokumen Tersimpan])
    RenderA4 --> EndDone
```

---

## 4. Sequence Diagram (Penerbitan Dokumen dan Pengambilan Daftar Register)

```mermaid
sequenceDiagram
    autonumber
    actor Staff as Staf Administrasi
    participant UI as Documents Hub (Next.js 15)
    participant Hdl as Documents Handler
    participant Svc as Documents Service
    participant Repo as Documents Repository
    participant DB as PostgreSQL 16
    participant S3 as Floci S3 Storage

    Staff->>UI: Isi Form Dokumen Resmi
    UI->>S3: POST /api/storage/upload (Lampiran PDF/Gambar)
    S3-->>UI: attachment_s3_key dan attachment_url

    UI->>Hdl: POST /api/documents
    Hdl->>Hdl: Validasi Permission documents:create
    Hdl->>Svc: CreateDocument(ctx, companyID, fullName, input)
    Svc->>Svc: Validasi Jenis, Kategori, Status, dan Kelengkapan Nomor
    Svc->>Repo: InsertCorporateDocument(...)
    Repo->>DB: INSERT INTO corporate_documents
    DB-->>Repo: Dokumen Tersimpan
    Repo-->>Svc: CorporateDocument
    Svc-->>Hdl: Dokumen Berhasil Diterbitkan
    Hdl-->>UI: 201 Created

    Staff->>UI: Buka Register Surat
    UI->>Hdl: GET /api/documents?category=HR&status=ISSUED
    Hdl->>Svc: ListDocuments(ctx, filter)
    Svc->>Repo: Query dengan Filter Company, Branch, Category, Type, Status
    Repo->>DB: SELECT ... FROM corporate_documents WHERE ...
    DB-->>Repo: Baris Dokumen dan Total
    Repo-->>Svc: Items dan Total
    Svc-->>Hdl: Hasil Register
    Hdl-->>UI: 200 OK
    UI-->>Staff: Tampilkan Tabel Register dan Aksi Cetak
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o{ CORPORATE_DOCUMENTS : "owns"
    BRANCHES ||--o{ CORPORATE_DOCUMENTS : "scopes"

    CORPORATE_DOCUMENTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string document_number UK
        string document_type "SK | ST | PAKLARING | SP | BAST | SKU | MEMO | SKET"
        string title
        string category "LEGAL | HR | OPERASIONAL | EKSEKUTIF | UMUM"
        string confidentiality "BIASA | PENTING | RAHASIA"
        string recipient_type "CUSTOMER | VENDOR | EMPLOYEE | INTERNAL | INSTANSI"
        string recipient_name
        string recipient_title
        text recipient_address
        date document_date
        string reference_number
        text opening_text
        text closing_text
        text notes
        string watermark
        jsonb items
        jsonb signatures
        string attachment_s3_key
        string attachment_url
        string file_s3_key
        string file_url
        string status "DRAFT | ISSUED | ARCHIVED | VOID"
        string created_by
        timestamp created_at
        timestamp updated_at
    }
```

Catatan integritas:
- Kombinasi `company_id` dan `document_number` bersifat unik.
- `branch_id` bersifat opsional untuk dokumen tingkat perusahaan.
- `items` dan `signatures` disimpan sebagai JSONB agar struktur dokumen fleksibel tanpa mengubah skema.
- Berkas lampiran disimpan di object storage dan hanya kunci S3 yang dicatat pada database.
