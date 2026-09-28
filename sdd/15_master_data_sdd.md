# 15. Software Design Document: Master Data & Multi-Branch Governance

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Master Data & Multi-Branch Governance** mengelola fondasi organisasi perusahaan: Struktur Entitas Induk (*Companies*), Cabang Operasional (*Branches / Multi-Branch Governance* - PSAK 65 / IFRS 10), Mata Uang & Kurs (*Currencies & Exchange Rates*), Satuan Pengukuran (*UOM & Konversi*), Pusat Biaya (*Cost Centers*), serta Mitra Bisnis Terpadu (*Counterparties*).

- **Lokasi Backend**: `services/api/internal/modules/master/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/master-data/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    SysAdmin["Super Administrator"]
    BranchMgr["Kepala Cabang (Branch Manager)"]
    FinController["Financial Controller"]

    subgraph MasterModule ["Modul Master Data & Multi-Branch"]
        UC1["Konfigurasi Entitas Perusahaan & NPWP"]
        UC2["Kelola Cabang & Alokasi Akun Rekening Koran Timbal-Balik"]
        UC3["Kelola Mata Uang & Pembaharuan Kurs Harian"]
        UC4["Manajemen Satuan Barang (UOM) & Faktor Konversi"]
        UC5["Struktur Pusat Biaya (Cost Centers & Profit Centers)"]
        UC6["Kelola Database Mitra Bisnis Terpusat (Counterparty)"]
        UC7["Audit Kepatuhan Isolasi Data Multi-Cabang"]
    end

    SysAdmin --> UC1
    SysAdmin --> UC2
    SysAdmin --> UC3
    SysAdmin --> UC4
    SysAdmin --> UC7

    BranchMgr --> UC5
    BranchMgr --> UC6

    FinController --> UC2
    FinController --> UC3
    FinController --> UC5
```

---

## 3. Activity Diagram (Tata Kelola Multi-Cabang & Validasi Isolasi)

```mermaid
flowchart TD
    Start([Pengguna Mengakses Sistem]) --> AuthToken[Validasi Sesi Pengguna & Token JWT]
    AuthToken --> FetchBranches[Ambil Daftar Cabang yang Diizinkan dari user_branches]
    
    FetchBranches --> SelectBranch{Pilih Cabang Aktif (activeBranchId)}
    SelectBranch --> SetContext[Sematkan Tenant Context: Company ID & Branch ID]
    
    SetContext --> ExecQuery[Jalankan Transaksi / Kueri Data]
    ExecQuery --> FilterBranch[Tambahkan Filter Otomatis: WHERE branch_id = activeBranchId]
    
    FilterBranch --> IsInterBranch{Apakah Transaksi Antar-Cabang?}
    IsInterBranch -- Tidak --> CommitLocal[Simpan Data Mutasi Cabang Lokal]
    
    IsInterBranch -- Ya --> CreateReciprocal[Bentuk Pencatatan Rekening Koran Timbal-Balik]
    CreateReciprocal --> PostDueToDueFrom[Debit: Due From Branch B, Kredit: Due To Branch A]
    PostDueToDueFrom --> CommitLocal
    
    CommitLocal --> EndDone([Kueri Terisolasi Aman])
```

---

## 4. Sequence Diagram (Pengalihan Konteks Cabang & Isolasi Kueri)

```mermaid
sequenceDiagram
    autonumber
    actor User as Pengguna Operasional
    participant UI as Master Data Web (Next.js 15)
    participant MW as Tenant / Branch Guard Middleware
    participant Hdl as Master Data Handler
    participant Svc as Master Data Service
    participant Repo as Master Data Repository
    participant DB as PostgreSQL 16

    User->>UI: Pilih Cabang Operasional dari Header Cabang
    UI->>MW: Request dengan Header 'X-Branch-ID: {branch_id}'
    MW->>DB: SELECT * FROM user_branches WHERE user_id = ... AND branch_id = ...
    DB-->>MW: Hak Akses Cabang Valid
    
    MW->>Hdl: Teruskan Request dengan Context (CompanyID, BranchID)
    Hdl->>Svc: GetBranchScopedData(ctx)
    Svc->>Repo: ListEntitiesByBranch(companyID, branchID)
    Repo->>DB: SELECT * FROM warehouses WHERE branch_id = ...
    DB-->>Repo: Dataset Terisolasi Cabang
    
    Repo-->>Svc: Clean Scoped Entities
    Svc-->>Hdl: Response Data
    Hdl-->>UI: 200 OK
    UI-->>User: Tampilkan Data Khusus Cabang yang Aktif
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--|{ BRANCHES : "operates"
    COMPANIES ||--o{ CURRENCIES : "configures"
    COMPANIES ||--o{ UOMS : "defines"
    COMPANIES ||--o{ COST_CENTERS : "establishes"
    COMPANIES ||--o{ COUNTERPARTIES : "interacts_with"
    BRANCHES ||--o{ COST_CENTERS : "contains"
    CURRENCIES ||--o{ EXCHANGE_RATES : "has_rates"

    COMPANIES {
        uuid id PK
        string code UK
        string name
        string legal_name
        string tax_id
        string address
        string phone
        string base_currency
        boolean is_active
    }

    BRANCHES {
        uuid id PK
        uuid company_id FK
        string code UK
        string name
        string address
        string phone
        string timezone
        uuid due_from_account_id FK
        uuid due_to_account_id FK
        boolean is_active
    }

    CURRENCIES {
        string code PK "e.g. IDR, USD, EUR, SGD"
        string name
        string symbol
        integer decimal_places
        boolean is_base
    }

    EXCHANGE_RATES {
        uuid id PK
        string currency_code FK
        date effective_date
        numeric rate_to_base
        string rate_source "BI_MIDDLE | JISDOR | MANUAL"
    }

    UOMS {
        uuid id PK
        uuid company_id FK
        string code UK "e.g. PCS, BOX, KG, LTR, MTR"
        string name
        string category "COUNT | WEIGHT | VOLUME | LENGTH"
    }

    COST_CENTERS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string code UK
        string name
        string department
        boolean is_active
    }

    COUNTERPARTIES {
        uuid id PK
        uuid company_id FK
        string code UK
        string name
        string counterparty_type "CUSTOMER | VENDOR | EMPLOYEE | BOTH"
        string tax_id
        string address
        boolean is_active
    }
```
