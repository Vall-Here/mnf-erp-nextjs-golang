# 15. Software Design Document: Master Data & Operational Site Governance

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Master Data & Operational Site Governance** mengelola fondasi organisasi korporasi industri PT SAWIT JAYA (SWT): Struktur Entitas Perseroan Tunggal (*Single Corporate Legal Entity*), Fasilitas Fisik Operasional Pabrik & Distribusi (*Operational Sites / Branches*), Mata Uang & Kurs (*Currencies & Exchange Rates*), Satuan Pengukuran (*UOM & Konversi*), Pusat Biaya (*Cost Centers*), serta Database Rekanan Mitra Bisnis Terpadu (*Counterparties*).

- **Lokasi Backend**: `services/api/internal/modules/master/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/master-data/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    SysAdmin["Super Administrator"]
    PlantMgr["Kepala Pabrik (Plant Manager)"]
    FinController["Financial Controller"]

    subgraph MasterModule ["Modul Master Data & Operational Sites"]
        UC1["Konfigurasi Entitas PT SAWIT JAYA & Legalitas NPWP"]
        UC2["Kelola Fasilitas Pabrik Kelapa Sawit & Gudang Distribusi (branches)"]
        UC3["Kelola Mata Uang & Pembaharuan Kurs Transaksi"]
        UC4["Manajemen Satuan Barang Industri (UOM) & Konversi"]
        UC5["Struktur Pusat Biaya Pabrik (Cost Centers)"]
        UC6["Kelola Master Rekanan Terpusat (Vendors & Customers)"]
        UC7["Audit Integritas Data & Isolasi Operasional Antar-Site"]
    end

    SysAdmin --> UC1
    SysAdmin --> UC2
    SysAdmin --> UC3
    SysAdmin --> UC4
    SysAdmin --> UC7

    PlantMgr --> UC5
    PlantMgr --> UC6

    FinController --> UC2
    FinController --> UC3
    FinController --> UC5
```

---

## 3. Activity Diagram (Tata Kelola Fasilitas Fisik & Isolasi Operasional)

```mermaid
flowchart TD
    Start([Pengguna Mengakses Sistem]) --> AuthToken[Validasi Sesi Pengguna & Token JWT]
    AuthToken --> FetchBranches[Ambil Daftar Cabang yang Diizinkan dari user_branches]
    
    FetchBranches --> SelectBranch{Pilih Site Operasional (activeBranchId)}
    SelectBranch --> IsSuperAdmin{Super Admin / Direksi?}
    
    IsSuperAdmin -- Ya --> AllowConsolidated[Dapat Memilih Semua Cabang atau Konsolidasi Nasional]
    IsSuperAdmin -- Tidak --> EnforceScoped[Wajib Terkunci pada Site Penugasan Resmi Pengguna]
    
    AllowConsolidated --> SetContext[Sematkan Konteks: Company ID & Authorized Branch ID]
    EnforceScoped --> SetContext
    
    SetContext --> ExecQuery[Jalankan Transaksi / Kueri Data]
    ExecQuery --> FilterBranch[Tambahkan Filter Otomatis: WHERE branch_id = activeBranchId]
    
    FilterBranch --> IsInterSiteTransfer{Pergerakan Barang Antar-Site?}
    IsInterSiteTransfer -- Tidak --> CommitLocal[Simpan Data Mutasi Site Lokal]
    
    IsInterSiteTransfer -- Ya --> STOFlow[Bentuk Stock Transfer Order / STO Antar-Gudang Fisik]
    STOFlow --> CommitLocal
    
    CommitLocal --> EndDone([Kueri Terisolasi & Sesuai Governance])
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

    User->>UI: Pilih Fasilitas Operasional dari Header Cabang
    UI->>MW: Request dengan Header 'X-Branch-ID: {branch_id}'
    MW->>MW: Verifikasi AllowedBranches di Token Claims
    
    MW->>Hdl: Teruskan Request dengan Context (CompanyID, AuthorizedBranchID)
    Hdl->>Svc: GetBranchScopedData(ctx)
    Svc->>Repo: ListEntitiesByBranch(companyID, branchID)
    Repo->>DB: SELECT * FROM warehouses WHERE branch_id = $branchID
    DB-->>Repo: Dataset Terisolasi Fasilitas Pabrik Tersebut
    
    Repo-->>Svc: Scoped Warehouses & Master Data
    Svc-->>Hdl: Response Data
    Hdl-->>UI: 200 OK
    UI-->>User: Tampilkan Data Khusus Fasilitas Pabrik yang Dipilih
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
    BRANCHES ||--o{ WAREHOUSES : "houses"

    COMPANIES {
        uuid id PK
        string code UK "SWT"
        string name "PT SAWIT JAYA"
        string legal_name "PT SAWIT JAYA (PERSEROAN TERBATAS)"
        string tax_id "NPWP Korporat"
        string address
        string base_currency "IDR"
        boolean is_active
    }

    BRANCHES {
        uuid id PK
        uuid company_id FK
        string code UK "PKS-01 | PKS-02 | DC-03"
        string name "Kantor Pusat / Pabrik Kelapa Sawit"
        string address
        string timezone "WIB | WITA"
        boolean is_active
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
