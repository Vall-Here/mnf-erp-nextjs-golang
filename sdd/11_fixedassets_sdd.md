# 11. Software Design Document: Fixed Assets & Depreciation

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Fixed Assets & Depreciation** mengelola aset berwujud perusahaan sesuai PSAK 16: Registrasi Aset Tetap (*Fixed Asset Registry*), Klasifikasi Kategori Fiskal, Eksekusi Penyusutan Bulanan Otomatis (*Depreciation Engine* - Garis Lurus / Saldo Menurun), Mutasi Lokasi/Departemen Aset, serta Pelepasan Aset (*Asset Disposal & Write-Off*) beserta perhitungan Laba/Rugi Pelepasan Aset.

- **Lokasi Backend**: `services/api/internal/modules/fixedassets/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/fixed-assets/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    AssetOfficer["Staf Aset & Inventaris"]
    AcctSpv["Supervisor Akuntansi"]
    FinDir["Direktur Keuangan"]
    Auditor["Auditor Aset"]

    subgraph FixedAssetsModule ["Modul Fixed Assets & Depreciation"]
        UC1["Pendaftaran Aset Tetap & Tag Barcode"]
        UC2["Konfigurasi Masa Manfaat & Metode Penyusutan"]
        UC3["Eksekusi Batch Depresiasi Bulanan"]
        UC4["Auto-Posting Jurnal Beban & Akumulasi Penyusutan"]
        UC5["Mutasi Lokasi & Serah Terima Aset Antar-Divisi"]
        UC6["Pelepasan / Penjualan Aset (Disposal)"]
        UC7["Laporan Nilai Buku Bersih (Net Book Value Register)"]
    end

    AssetOfficer --> UC1
    AssetOfficer --> UC2
    AssetOfficer --> UC5

    AcctSpv --> UC3
    AcctSpv --> UC4
    AcctSpv --> UC6
    AcctSpv --> UC7

    FinDir --> UC6

    Auditor --> UC1
    Auditor --> UC7
```

---

## 3. Activity Diagram (Eksekusi Batch Penyusutan Bulanan)

```mermaid
flowchart TD
    Start([Akhir Periode Bulan]) --> SelectPeriod[Pilih Periode Bulan & Tahun Penyusutan]
    SelectPeriod --> FetchAssets[Ambil Seluruh Aset Aktif yang Belum Habis Masa Manfaat]
    
    FetchAssets --> LoopAsset[Proses Kalkulasi Penyusutan per Aset]
    LoopAsset --> CheckMethod{Metode Penyusutan}
    
    CheckMethod -- Garis Lurus (Straight-Line) --> CalcSL[Depresiasi = (Harga Perolehan - Residu) / Bulan Masa Manfaat]
    CheckMethod -- Saldo Menurun (Declining) --> CalcDB[Depresiasi = Nilai Buku Awal * Tarif Saldo Menurun]
    
    CalcSL --> UpdateNBV[Hitung Nilai Buku Baru / Net Book Value]
    CalcDB --> UpdateNBV
    
    UpdateNBV --> SaveLine[Simpan Rincian Depreciation Line]
    SaveLine --> CheckRemaining{Masih Ada Aset Lain?}
    
    CheckRemaining -- Ya --> LoopAsset
    CheckRemaining -- Selesai --> SummarizeBatch[Akumulasi Total Beban Depresiasi Periode]
    
    SummarizeBatch --> CreateJournal[Auto-Posting Jurnal ke General Ledger]
    CreateJournal --> JournalPost[Debit: Beban Penyusutan, Kredit: Akumulasi Penyusutan]
    
    JournalPost --> UpdateAssetRecords[Perbarui Total Akumulasi Penyusutan Aset]
    UpdateAssetRecords --> EndDone([Penyusutan Selesai])
```

---

## 4. Sequence Diagram (Pelepasan Aset & Perhitungan Laba/Rugi)

```mermaid
sequenceDiagram
    autonumber
    actor Officer as Supervisor Akuntansi
    participant UI as Fixed Assets Web (Next.js 15)
    participant Hdl as FixedAssets Handler
    participant Svc as FixedAssets Service
    participant Acct as Accounting Module (GL)
    participant Repo as FixedAssets Repository
    participant DB as PostgreSQL 16

    Officer->>UI: Submit Pelepasan / Penjualan Aset
    UI->>Hdl: POST /api/v1/fixed-assets/{id}/dispose
    Hdl->>Svc: DisposeAsset(ctx, id, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: GetAssetDetails(id)
        Repo->>DB: SELECT * FROM fixed_assets WHERE id = ...
        
        Svc->>Svc: Hitung Gain/Loss = Nilai Penjualan - (Harga Perolehan - Akumulasi Penyusutan)
        
        Svc->>Repo: Insert Asset Disposal Record
        Repo->>DB: INSERT INTO asset_disposals (asset_id, disposal_date, sale_price, gain_loss_amount, ...)
        
        Svc->>Acct: CreateDisposalJournal(Kas, Akumulasi Penyusutan, Aset Tetap, Laba/Rugi Disposal)
        Acct->>DB: INSERT INTO journal_entries & journal_lines
        
        Svc->>Repo: Update Asset Status to DISPOSED
        Repo->>DB: UPDATE fixed_assets SET status = 'DISPOSED' WHERE id = ...
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Disposal Result Entity
    Hdl-->>UI: 200 OK
    UI-->>Officer: Tampilkan Berita Acara Pelepasan Aset & Jurnal Terposting
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    ASSET_CATEGORIES ||--o{ FIXED_ASSETS : "classifies"
    FIXED_ASSETS ||--o{ DEPRECIATION_LINES : "depreciated_in"
    FIXED_ASSETS ||--o| ASSET_DISPOSALS : "disposed_in"
    DEPRECIATION_RUNS ||--|{ DEPRECIATION_LINES : "contains"
    DEPRECIATION_RUNS ||--o| JOURNAL_ENTRIES : "posted_as"

    ASSET_CATEGORIES {
        uuid id PK
        uuid company_id FK
        string category_code UK
        string category_name
        integer useful_life_months
        string depreciation_method "STRAIGHT_LINE | DECLINING_BALANCE"
        uuid asset_account_id FK
        uuid accumulated_depr_account_id FK
        uuid depr_expense_account_id FK
    }

    FIXED_ASSETS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid category_id FK
        string asset_code UK
        string name
        date acquisition_date
        numeric acquisition_cost
        numeric salvage_value
        integer useful_life_months
        numeric accumulated_depreciation
        numeric net_book_value
        string status "ACTIVE | FULLY_DEPRECIATED | DISPOSED | UNDER_REPAIR"
    }

    DEPRECIATION_RUNS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        integer period_year
        integer period_month
        numeric total_depreciation_amount
        string status "DRAFT | POSTED"
        uuid journal_id FK
        timestamp executed_at
    }

    DEPRECIATION_LINES {
        uuid id PK
        uuid depreciation_run_id FK
        uuid asset_id FK
        numeric beginning_book_value
        numeric depreciation_amount
        numeric ending_book_value
    }

    ASSET_DISPOSALS {
        uuid id PK
        uuid asset_id FK
        date disposal_date
        string disposal_type "SALE | SCRAP | LOSS"
        numeric sale_price
        numeric gain_loss_amount
        string reason
        uuid journal_id FK
    }
```
