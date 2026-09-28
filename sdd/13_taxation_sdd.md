# 13. Software Design Document: Taxation Engine & e-Faktur

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Taxation Engine** mengelola kepatuhan perpajakan Indonesia: Pajak Pertambahan Nilai (*PPN 11% / 12% & PPN WAPU*), Pajak Penghasilan Pasal 23 & 4(2) Final (*Withholding Tax*), Alokasi Nomor Seri Faktur Pajak (*NSFP*), serta Pembuatan Berkas Ekspor Standar DJP (*e-Faktur CSV/XML & Bukti Potong Unifikasi*).

- **Lokasi Backend**: `services/api/internal/modules/taxation/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/taxation/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    TaxOfficer["Staf Pajak (Tax Specialist)"]
    AcctSpv["Supervisor Akuntansi"]
    Auditor["Auditor Pajak"]

    subgraph TaxModule ["Modul Taxation Engine & e-Faktur"]
        UC1["Pendaftaran Range Nomor Seri Faktur Pajak (NSFP)"]
        UC2["Penerbitan Faktur Pajak Keluaran (Sales VAT)"]
        UC3["Pencatatan & Validasi Faktur Pajak Masukan (Purchase VAT)"]
        UC4["Pemotongan Pajak PPh 23 / 4(2) Atas Jasa"]
        UC5["Export Format CSV / XML e-Faktur DJP"]
        UC6["Rekonsiliasi Pajak vs SPT Masa PPN"]
        UC7["Otorisasi SPT Masa & Bukti Pembayaran Pajak (NTPN)"]
    end

    TaxOfficer --> UC1
    TaxOfficer --> UC2
    TaxOfficer --> UC3
    TaxOfficer --> UC4
    TaxOfficer --> UC5

    AcctSpv --> UC6
    AcctSpv --> UC7

    Auditor --> UC2
    Auditor --> UC3
    Auditor --> UC6
```

---

## 3. Activity Diagram (Alur Penerbitan Faktur Pajak & Ekspor e-Faktur)

```mermaid
flowchart TD
    Start([Transaksi Penjualan Terbit]) --> CheckNSFP{Ketersediaan NSFP Aktif}
    
    CheckNSFP -- NSFP Habis --> AlertTax[Peringatan: Kuota NSFP Habis, Minta Kuota DJP]
    AlertTax --> RegisterNSFP[Input Alokasi Range NSFP Baru]
    RegisterNSFP --> CheckNSFP
    
    CheckNSFP -- NSFP Tersedia --> AllocateNSFP[Ambil Nomor Seri Faktur Pajak Terurut]
    AllocateNSFP --> GenTaxInvoice[Buat Faktur Pajak Keluaran]
    
    GenTaxInvoice --> CalcTax[Hitung DPP & Tarif PPN 11% / 12%]
    CalcTax --> LinkSales[Tautkan dengan ID Invoice Penjualan]
    
    LinkSales --> ValidateFormat[Validasi Format Data Standar DJP]
    ValidateFormat --> ExportEFaktur[Export Berkas CSV / XML e-Faktur]
    
    ExportEFaktur --> ImportDJP[Import ke Aplikasi e-Faktur DJP / Web-Based]
    ImportDJP --> GetApproval[Faktur Pajak Disetujui / QR Barcode Terbit]
    
    GetApproval --> UpdateStatus[Update Status: APPROVED & Terekonsiliasi]
    UpdateStatus --> EndDone([Faktur Pajak Sah])
```

---

## 4. Sequence Diagram (Alokasi NSFP & Pembuatan Faktur Pajak)

```mermaid
sequenceDiagram
    autonumber
    actor TaxStaff as Staf Pajak
    participant UI as Taxation Web (Next.js 15)
    participant Hdl as Taxation Handler
    participant Svc as Taxation Service
    participant Repo as Taxation Repository
    participant DB as PostgreSQL 16

    TaxStaff->>UI: Request Generate Faktur Pajak dari Sales Invoice
    UI->>Hdl: POST /api/v1/taxation/invoices
    Hdl->>Svc: GenerateTaxInvoice(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: GetNextAvailableNSFP(companyID, year)
        Repo->>DB: SELECT * FROM nsfp_ranges WHERE is_used = false ORDER BY nsfp_number ASC LIMIT 1 FOR UPDATE
        DB-->>Repo: Available NSFP Record
        
        Svc->>Repo: Mark NSFP as Used
        Repo->>DB: UPDATE nsfp_ranges SET is_used = true, used_at = NOW() WHERE id = ...
        
        Svc->>Repo: Insert Tax Invoice Record
        Repo->>DB: INSERT INTO tax_invoices (invoice_type, nsfp_number, customer_id, dpp_amount, vat_amount, status)
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Tax Invoice Entity
    Hdl-->>UI: 201 Created
    UI-->>TaxStaff: Tampilkan Faktur Pajak & Tombol Download CSV e-Faktur
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o{ NSFP_RANGES : "allocated_to"
    NSFP_RANGES ||--o| TAX_INVOICES : "assigns"
    CUSTOMERS ||--o{ TAX_INVOICES : "billed_to"
    TAX_INVOICES ||--o{ WITHHOLDING_TAXES : "has_withholding"

    NSFP_RANGES {
        uuid id PK
        uuid company_id FK
        string nsfp_start UK
        string nsfp_end
        string current_number
        integer total_allocated
        integer total_used
        integer year
        boolean is_active
    }

    TAX_INVOICES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string invoice_type "OUTWARD_PPN | INWARD_PPN"
        string nsfp_number UK
        date tax_invoice_date
        uuid counterparty_id FK
        string counterparty_npwp
        string counterparty_name
        numeric dpp_amount
        numeric vat_rate_percent
        numeric vat_amount
        string status "DRAFT | EXPORTED | APPROVED | REJECTED | CANCELLED"
        uuid sales_invoice_id FK
        uuid vendor_bill_id FK
    }

    WITHHOLDING_TAXES {
        uuid id PK
        uuid company_id FK
        string tax_type "PPH_23 | PPH_4_2 | PPH_22"
        string bupot_number UK
        date bupot_date
        uuid vendor_id FK
        string object_tax_code
        numeric gross_amount
        numeric tax_rate_percent
        numeric tax_amount
        string status "DRAFT | REPORTED"
    }
```
