# 01. Software Design Document: Accounting & General Ledger

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Accounting & General Ledger** bertanggung jawab atas pencatatan seluruh transaksi finansial PT SAWIT JAYA (SWT) dengan prinsip *Double-Entry Bookkeeping*. Setiap transaksi keuangan wajib memenuhi invarian mutlak `SUM(debit) == SUM(credit)`. Transaksi berakar pada satu Buku Besar korporat terpadu (*Single General Ledger*), dengan `branch_id` bertindak sebagai penanda segmen fasilitas fisik (*Cost Centers*).

- **Lokasi Backend**: `services/api/internal/modules/accounting/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/accounting/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    AcctStaff["Staf Akuntansi (Maker)"]
    FinMgr["Finance & Accounting Manager (Checker/Approver)"]
    Auditor["Internal / External Auditor"]

    subgraph AccountingModule ["Modul Accounting & General Ledger"]
        UC1["Kelola Bagan Akun (Chart of Accounts PSAK)"]
        UC2["Input Draft Jurnal Umum Memorial (JV)"]
        UC3["Pemeriksaan & Otorisasi Posting Jurnal ke GL (accounting:approve)"]
        UC4["Generate Trial Balance & Buku Besar Korporat"]
        UC5["Generate Laporan Laba Rugi & Neraca"]
        UC6["Penutupan Periode Fiskal (Month-End Close)"]
        UC7["Pagu Anggaran & Komitmen Belanja (Budget Encumbrance)"]
        UC8["Kertas Kerja Konsolidasi Multi-Entitas (PSAK 65)"]
        UC9["Audit Trail & Jejak Pembukuan"]
    end

    AcctStaff --> UC1
    AcctStaff --> UC2
    AcctStaff --> UC4

    FinMgr --> UC3
    FinMgr --> UC5
    FinMgr --> UC6
    FinMgr --> UC7
    FinMgr --> UC8

    Auditor --> UC4
    Auditor --> UC5
    Auditor --> UC9
```

---

## 3. Activity Diagram (Alur Pembuatan & Otorisasi Posting Jurnal - SoD)

```mermaid
flowchart TD
    Start([Mulai Input Voucher Jurnal]) --> InputLines[Staf Akuntansi Input Baris Debit dan Kredit]
    InputLines --> CheckBalance{Apakah SUM Debit == SUM Kredit?}
    
    CheckBalance -- Tidak --> ShowBalanceError[Tolak Transaksi: Saldo Jurnal Tidak Seimbang]
    ShowBalanceError --> InputLines
    
    CheckBalance -- Ya --> CheckPeriod{Apakah Periode Fiskal Terbuka?}
    CheckPeriod -- Terkunci/Tutup --> ShowPeriodError[Tolak: Periode Fiskal Telah Ditutup]
    ShowPeriodError --> EndFail([Selesai Gagal])
    
    CheckPeriod -- Terbuka --> SaveDraft[Simpan Jurnal Status: DRAFT]
    SaveDraft --> NotifyChecker[Notifikasi ke Finance Manager untuk Verifikasi]
    
    NotifyChecker --> ReviewAction[Finance Manager Memeriksa Dokumen Pendukung]
    ReviewAction --> CanPost{Memiliki Izin accounting:approve?}
    
    CanPost -- Tidak --> Reject403[HTTP 403 Forbidden: Pelanggaran Segregation of Duties]
    Reject403 --> EndFail
    
    CanPost -- Ya --> PostAction[Eksekusi Endpoint /journals/{id}/post]
    PostAction --> UpdateGL[Ubah Status: POSTED & Set posted_by, posted_at]
    UpdateGL --> CreateAudit[Rekam Jejak Audit Trail]
    CreateAudit --> SuccessEnd([Jurnal Berhasil Diposting ke Buku Besar])
```

---

## 4. Sequence Diagram (Siklus Pembuatan Draft & Posting Jurnal Terpisah)

```mermaid
sequenceDiagram
    autonumber
    actor Maker as Staf Akuntansi (Maker)
    actor Checker as Finance Manager (Approver)
    participant UI as Accounting Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Accounting Handler
    participant Svc as Accounting Service
    participant Repo as Accounting Repository
    participant DB as PostgreSQL 16

    Maker->>UI: Submit Draft Voucher Jurnal
    UI->>MW: POST /api/accounting/journals (Bearer Maker Token)
    MW->>MW: Verifikasi izin accounting:create
    MW->>Hdl: HandleCreateJournal
    Hdl->>Svc: CreateJournal
    Svc->>Svc: Validasi SUM(debit) == SUM(credit)
    Svc->>Repo: Insert journal_entries (Status: DRAFT, created_by: MakerID)
    Repo->>DB: INSERT INTO journal_entries & journal_lines
    DB-->>Repo: Saved Draft Journal ID
    Repo-->>UI: 201 Created (Draft Voucher Siap Diverifikasi)

    Note over Maker,Checker: Pemisahan Tugas (Segregation of Duties - Maker != Checker)

    Checker->>UI: Buka Daftar Jurnal & Klik "Posting ke Buku Besar"
    UI->>MW: POST /api/accounting/journals/{id}/post (Bearer Checker Token)
    MW->>MW: Verifikasi izin accounting:approve
    MW->>Hdl: HandlePostJournal
    Hdl->>Svc: PostJournal
    Svc->>Repo: Update journal_entries SET status='POSTED', posted_by=CheckerID
    Repo->>DB: UPDATE journal_entries WHERE id=$1 AND status='DRAFT'
    DB-->>Repo: Rows Affected: 1
    Repo->>DB: INSERT INTO audit_logs (Action: POST_JOURNAL_VOUCHER)
    Repo-->>UI: 200 OK (Voucher Berhasil Diposting ke General Ledger)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    CHART_OF_ACCOUNTS ||--o{ JOURNAL_LINES : "referenced_by"
    CHART_OF_ACCOUNTS ||--o{ CLOSING_BALANCES : "has_periodic"
    CHART_OF_ACCOUNTS ||--o{ BUDGET_LINES : "budgeted_in"
    JOURNAL_ENTRIES ||--|{ JOURNAL_LINES : "contains"
    FISCAL_YEARS ||--|{ FISCAL_PERIODS : "divided_into"
    FISCAL_PERIODS ||--o{ JOURNAL_ENTRIES : "groups"
    BUDGETS ||--|{ BUDGET_LINES : "composed_of"
    BUDGETS ||--o{ BUDGET_ENCUMBRANCES : "commits"

    CHART_OF_ACCOUNTS {
        uuid id PK
        uuid company_id FK
        string account_number UK
        string name
        string account_type "ASSET | LIABILITY | EQUITY | REVENUE | EXPENSE"
        string normal_balance "DEBIT | CREDIT"
        uuid parent_id FK
        boolean is_active
        timestamp created_at
    }

    JOURNAL_ENTRIES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string entry_number UK
        date entry_date
        string source_type "MANUAL | SALES | PURCHASING | INVENTORY | PAYROLL"
        string source_document
        string memo
        string status "DRAFT | POSTED | VOID"
        uuid created_by FK
        uuid posted_by FK
        timestamp posted_at
        timestamp created_at
    }

    JOURNAL_LINES {
        uuid id PK
        uuid journal_entry_id FK
        uuid account_id FK
        string description
        numeric debit_amount "NOT NULL DEFAULT 0"
        numeric credit_amount "NOT NULL DEFAULT 0"
        string currency "DEFAULT IDR"
        numeric exchange_rate "DEFAULT 1"
        integer line_number
    }

    BUDGETS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string budget_number UK
        string name
        integer fiscal_year
        string department
        string control_mode "HARD_STOP | ADVISORY_WARNING | TRACK_ONLY"
        numeric total_allocated
        numeric total_encumbered
        numeric total_actual
        string status "DRAFT | APPROVED | CLOSED"
    }

    BUDGET_ENCUMBRANCES {
        uuid id PK
        uuid budget_id FK
        string source_type "PR | PO | CONTRACT"
        uuid source_id
        numeric encumbered_amount
        string status "ACTIVE | LIQUIDATED | RELEASED"
        timestamp created_at
    }
```

---

## 6. Cakupan Implementasi Terkini

- **Budget Encumbrance Engine**: Pemeriksaan ketersediaan pagu sebelum penerbitan PR dan PO. Transaksi dihentikan otomatis jika berstatus `HARD_STOP` dan pagu tidak mencukupi.
- **Konsolidasi Multi-Entitas PSAK 65 / IFRS 10**: Penyusunan laporan keuangan gabungan holding dan anak perusahaan dengan eliminasi otomatis akun transaksi timbal-balik (*intercompany elimination*).
- **Segregation of Duties Enforcement**: Staf pembukuan tidak memiliki hak posting langsung ke Buku Besar; fungsi posting diwajibkan melalui peran Manajer Keuangan (`accounting:approve`).
