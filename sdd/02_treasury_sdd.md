# 02. Software Design Document: Treasury & Bank Reconciliation

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Treasury** mengelola rekening kas & bank perusahaan, penerimaan kas (*Cash Receipts*), pengeluaran kas (*Cash Disbursements*), impor mutasi rekening koran elektronik (BCA, Mandiri, BRI, BNI), serta mesin rekonsiliasi bank otomatis dengan jurnal umum.

- **Lokasi Backend**: `services/api/internal/modules/treasury/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/treasury/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Cashier["Kasir Treasury"]
    SpvAcct["Supervisor Akuntansi"]
    FinDir["Direktur Keuangan"]

    subgraph TreasuryModule ["Modul Treasury & Bank Reconciliation"]
        UC1["Kelola Rekening Kas & Bank Perusahaan"]
        UC2["Penerbitan Bukti Kas Masuk / Keluar (Voucher)"]
        UC3["Upload & Parsing Rekening Koran Elektronik (CSV)"]
        UC4["Eksekusi Mesin Auto-Matching Rekonsiliasi"]
        UC5["Pencocokan Manual Sisa Mutasi Bank"]
        UC6["Cetak Berita Acara Rekonsiliasi Bank"]
        UC7["Otorisasi Pengeluaran Kas Nominal Besar"]
    end

    Cashier --> UC1
    Cashier --> UC2
    Cashier --> UC3

    SpvAcct --> UC4
    SpvAcct --> UC5
    SpvAcct --> UC6

    FinDir --> UC7
```

---

## 3. Activity Diagram (Alur Rekonsiliasi Bank Otomatis)

```mermaid
flowchart TD
    Start([Mulai Rekonsiliasi]) --> UploadCSV[Upload Rekening Koran CSV Bank]
    UploadCSV --> ParseCSV[Ekstraksi Baris Mutasi, Saldo Awal, & Saldo Akhir]
    ParseCSV --> PreviewData[Pratinjau Mutasi Bank]
    PreviewData --> SaveStatement[Simpan Bank Statement & Statement Lines]
    
    SaveStatement --> RunEngine[Jalankan Engine Auto-Matching]
    RunEngine --> MatchLoop[Loop Tiap Baris Mutasi vs Jurnal Kas Terposting]
    
    MatchLoop --> IsExactMatch{Cocok Nominal, Tanggal & Ref?}
    IsExactMatch -- Ya --> LinkJournal[Tautkan Reconciled Journal ID & Status RECONCILED]
    IsExactMatch -- Tidak --> MarkUnmatched[Status UNRECONCILED]
    
    LinkJournal --> CheckProgress{Semua Baris Telah Dievaluasi?}
    MarkUnmatched --> CheckProgress
    
    CheckProgress -- Belum --> MatchLoop
    CheckProgress -- Selesai --> CalcDiscrepancy[Hitung Selisih / Net Discrepancy]
    
    CalcDiscrepancy --> IsBalanced{Selisih == 0?}
    IsBalanced -- Ya --> StatusFull[Status: FULLY_RECONCILED]
    IsBalanced -- Tidak --> StatusPartial[Status: PARTIALLY_RECONCILED]
    
    StatusFull --> GenReport[Generate Berita Acara Rekonsiliasi]
    StatusPartial --> ManualMatchUI[Sediakan Antarmuka Pencocokan Manual]
    ManualMatchUI --> GenReport
    GenReport --> EndDone([Selesai Rekonsiliasi])
```

---

## 4. Sequence Diagram (Auto-Matching Engine Execution)

```mermaid
sequenceDiagram
    autonumber
    actor User as Supervisor Akuntansi
    participant UI as Treasury Web (Next.js 15)
    participant Hdl as Treasury Handler
    participant Svc as Treasury Service
    participant Repo as Treasury Repository
    participant DB as PostgreSQL 16

    User->>UI: Klik "Jalankan Auto-Matching"
    UI->>Hdl: POST /api/v1/treasury/reconciliations/{id}/auto-match
    Hdl->>Svc: AutoMatchStatement(ctx, reconID)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: GetStatementLinesByReconID(reconID)
        Repo->>DB: SELECT * FROM bank_statement_lines WHERE statement_id = ...
        DB-->>Repo: List Statement Lines
        
        Svc->>Repo: GetPostedCashJournals(branchID, dateRange)
        Repo->>DB: SELECT * FROM journal_lines JOIN journal_entries ...
        DB-->>Repo: List Posted Journal Lines
        
        Svc->>Svc: Algoritma Pencocokan Nominal & Tanggal Toleransi
        
        loop Tiap Baris yang Cocok
            Svc->>Repo: LinkStatementToJournal(lineID, journalID, journalRef)
            Repo->>DB: UPDATE bank_statement_lines SET is_reconciled = true, ...
        end
        
        Svc->>Repo: UpdateReconciliationProgress(reconID)
        Repo->>DB: UPDATE bank_reconciliations SET matched_count = ..., status = ...
    end

    Svc-->>Hdl: Match Result Summary (Total Matched, Remaining Unmatched)
    Hdl-->>UI: 200 OK (Match Summary)
    UI-->>User: Update Progress Bar & Tampilkan Status Terekonsiliasi
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    BANK_ACCOUNTS ||--o{ CASH_TRANSACTIONS : "holds"
    BANK_ACCOUNTS ||--o{ BANK_STATEMENTS : "receives"
    BANK_STATEMENTS ||--|{ BANK_STATEMENT_LINES : "contains"
    BANK_STATEMENTS ||--o{ BANK_RECONCILIATIONS : "subject_to"
    BANK_STATEMENT_LINES }o--o| JOURNAL_ENTRIES : "reconciled_with"

    BANK_ACCOUNTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string account_number UK
        string account_name
        string bank_name
        string currency
        numeric current_balance
        uuid gl_account_id FK
        boolean is_active
    }

    CASH_TRANSACTIONS {
        uuid id PK
        uuid bank_account_id FK
        string voucher_number UK
        string transaction_type "RECEIPT | DISBURSEMENT | TRANSFER"
        date transaction_date
        numeric amount
        string recipient_or_payer
        string description
        string status "DRAFT | APPROVED | POSTED | VOID"
        uuid journal_id FK
    }

    BANK_STATEMENTS {
        uuid id PK
        uuid bank_account_id FK
        date statement_period_start
        date statement_period_end
        numeric opening_balance
        numeric closing_balance
        numeric total_debit
        numeric total_credit
        string file_name
        timestamp uploaded_at
    }

    BANK_STATEMENT_LINES {
        uuid id PK
        uuid statement_id FK
        date transaction_date
        string description
        string reference_number
        numeric debit_amount
        numeric credit_amount
        boolean is_reconciled
        uuid reconciled_journal_id FK
        string reconciled_journal_ref
    }

    BANK_RECONCILIATIONS {
        uuid id PK
        uuid statement_id FK
        uuid branch_id FK
        date reconciliation_date
        numeric book_balance
        numeric bank_balance
        numeric net_discrepancy
        string status "UNRECONCILED | PARTIALLY_RECONCILED | FULLY_RECONCILED"
        uuid reconciled_by FK
        timestamp reconciled_at
    }
```
