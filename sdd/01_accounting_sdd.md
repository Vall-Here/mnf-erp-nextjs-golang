# 01. Software Design Document: Accounting & General Ledger

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Accounting & General Ledger** bertanggung jawab atas pencatatan seluruh mutasi finansial perusahaan dengan prinsip *Double-Entry Bookkeeping*. Setiap transaksi keuangan wajib memenuhi invarian `SUM(debit) == SUM(credit)`.

- **Lokasi Backend**: `services/api/internal/modules/accounting/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/accounting/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    AcctStaff["Staf Akuntansi"]
    FinMgr["Manajer Keuangan"]
    Auditor["Auditor Eksternal"]

    subgraph AccountingModule ["Modul Accounting & General Ledger"]
        UC1["Kelola Bagan Akun (Chart of Accounts)"]
        UC2["Input Jurnal Umum Manual (JV)"]
        UC3["Pemeriksaan & Posting Jurnal (Debit == Credit)"]
        UC4["Generate Trial Balance & Buku Besar"]
        UC5["Generate Laporan Laba Rugi & Neraca"]
        UC6["Penutupan Periode Fiskal (Month-End Close)"]
        UC7["Audit Trail Pemeriksaan Jurnal"]
    end

    AcctStaff --> UC1
    AcctStaff --> UC2
    AcctStaff --> UC4

    FinMgr --> UC3
    FinMgr --> UC5
    FinMgr --> UC6

    Auditor --> UC4
    Auditor --> UC5
    Auditor --> UC7
```

---

## 3. Activity Diagram (Alur Posting Jurnal Umum)

```mermaid
flowchart TD
    Start([Mulai Input Jurnal]) --> InputLines[Input Baris Debit dan Kredit]
    InputLines --> CheckBalance{Apakah SUM Debit == SUM Kredit?}
    
    CheckBalance -- Tidak --> ShowError[Tolak Transaksi: Saldo Jurnal Tidak Seimbang]
    ShowError --> InputLines
    
    CheckBalance -- Ya --> CheckPeriod{Apakah Periode Fiskal Terbuka?}
    CheckPeriod -- Terkunci/Tutup --> ShowPeriodError[Tolak: Periode Telah Ditutup]
    ShowPeriodError --> EndFail([Selesai Gagal])
    
    CheckPeriod -- Terbuka --> SaveDraft[Simpan Status DRAFT]
    SaveDraft --> RequestApprove{Butuh Otorisasi Manajer?}
    
    RequestApprove -- Ya --> SubmitApproval[Kirim ke Workflow Otorisasi]
    SubmitApproval --> WaitApprove[Menunggu Persetujuan]
    WaitApprove --> PostAction[Posting Status POSTED]
    
    RequestApprove -- Tidak / Disetujui Langsung --> PostAction
    
    PostAction --> UpdateGL[Update Akumulasi Saldo Rekening Buku Besar]
    UpdateGL --> CreateAudit[Rekam Jejak Audit Trail]
    CreateAudit --> SuccessEnd([Jurnal Berhasil Diposting])
```

---

## 4. Sequence Diagram (API Request & Response Lifecycle)

```mermaid
sequenceDiagram
    autonumber
    actor User as Staf Akuntansi
    participant UI as Accounting Web (Next.js 15)
    participant Auth as Auth Middleware (RBAC: accounting:create)
    participant Hdl as Accounting Handler
    participant Svc as Accounting Service
    participant Repo as Accounting Repository
    participant DB as PostgreSQL 16

    User->>UI: Submit Journal Entry Form
    UI->>Auth: POST /api/v1/accounting/journals
    Auth->>Auth: Validasi Token & Izin Cabang (user_branches)
    Auth->>Hdl: HandleCreateJournal(w, r)
    Hdl->>Svc: CreateJournal(ctx, input)
    
    rect rgb(245, 245, 245)
        Note over Svc,DB: Validasi Invarian & Database Transaction
        Svc->>Svc: Validasi SUM(debit) == SUM(credit)
        Svc->>Repo: Begin Tx
        Repo->>DB: Check fiscal_periods.is_locked
        DB-->>Repo: Period Status: OPEN
        Repo->>DB: INSERT INTO journal_entries (entry_number, date, status, branch_id)
        Repo->>DB: INSERT INTO journal_lines (entry_id, account_id, debit, credit, line_desc)
        Repo->>DB: INSERT INTO audit_logs (actor, action, resource, diff)
        Repo->>DB: Commit Tx
    end

    Svc-->>Hdl: JournalEntry Created Entity
    Hdl-->>UI: 201 Created (JSON Response Envelope)
    UI-->>User: Tampilkan Notifikasi Sukses & Tombol Cetak Dokumen JV
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    CHART_OF_ACCOUNTS ||--o{ JOURNAL_LINES : "referenced_by"
    CHART_OF_ACCOUNTS ||--o{ CLOSING_BALANCES : "has_periodic"
    JOURNAL_ENTRIES ||--|{ JOURNAL_LINES : "contains"
    FISCAL_YEARS ||--|{ FISCAL_PERIODS : "divided_into"
    FISCAL_PERIODS ||--o{ JOURNAL_ENTRIES : "groups"

    CHART_OF_ACCOUNTS {
        uuid id PK
        uuid company_id FK
        string account_code UK
        string account_name
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
        string entry_type "MANUAL | AUTO_SALES | AUTO_PURCHASE | AUTO_PAYROLL | AUTO_INVENTORY"
        string reference_number
        string description
        string status "DRAFT | POSTED | VOID"
        uuid created_by FK
        uuid posted_by FK
        timestamp posted_at
        timestamp created_at
    }

    JOURNAL_LINES {
        uuid id PK
        uuid entry_id FK
        uuid account_id FK
        numeric debit "NOT NULL DEFAULT 0"
        numeric credit "NOT NULL DEFAULT 0"
        string line_description
        integer line_order
    }

    FISCAL_YEARS {
        uuid id PK
        uuid company_id FK
        integer year UK
        date start_date
        date end_date
        boolean is_closed
    }

    FISCAL_PERIODS {
        uuid id PK
        uuid fiscal_year_id FK
        integer period_month
        date start_date
        date end_date
        boolean is_locked
    }

    CLOSING_BALANCES {
        uuid id PK
        uuid account_id FK
        uuid fiscal_period_id FK
        numeric opening_balance
        numeric total_debit
        numeric total_credit
        numeric closing_balance
    }
```
