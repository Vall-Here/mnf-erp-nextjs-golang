# 00. System Architecture Overview & Global Design

## 1. Ikhtisar Arsitektur Sistem

Enterprise ERP dibangun menggunakan arsitektur **Modular Monolith** dengan prinsip *Clean Architecture* di sisi Backend (Golang) dan *Next.js 15 App Router* di sisi Frontend.

```mermaid
graph TB
    subgraph ClientLayer ["Frontend Client Layer (Next.js 15 App Router)"]
        UI_DASH["Dashboard & Executive Views"]
        UI_OPS["Operational Modules (Sales, Purchasing, Inventory, Mfg, Logistics, CMMS)"]
        UI_FIN["Financial Modules (Accounting, Treasury, Tax, Fixed Assets)"]
        UI_HR["HRM, Payroll & Employee Self-Service"]
        UI_DOC["Corporate Documents Hub"]
        UI_PORTAL["B2B Vendor & Customer Portal"]
        UI_SHARED["Shared Components (TanStack Table, Print Engine, Multi-Tier Approval Modal)"]
    end

    subgraph Gateway ["Reverse Proxy & Web Server"]
        CADDY["Caddy Reverse Proxy (Path Routing, TLS-ready)"]
    end

    subgraph BackendMonolith ["Backend Modular Monolith (Golang 1.24+ / Chi Router)"]
        AUTH_MW["Middleware (JWT Auth, RBAC Matrix, Tenant/Branch Guard, Audit Logger)"]
        
        subgraph CoreBusinessModules ["20 Core Business Modules"]
            MOD_MASTER["Master Data & Multi-Branch"]
            MOD_IAM["IAM & Security"]
            MOD_ACCT["Accounting & GL Engine"]
            MOD_TREAS["Treasury & Reconciliation"]
            MOD_SALES["Sales & Distribution"]
            MOD_PURCH["Purchasing & Procurement"]
            MOD_INV["Inventory & Bin Warehouse"]
            MOD_MFG["Manufacturing & Job Costing"]
            MOD_QC["Quality Control & ISO 9001"]
            MOD_LOG["Logistics & Fleet Dispatch"]
            MOD_CMMS["Plant Maintenance (CMMS)"]
            MOD_HRM["HRIS & Statutory Payroll"]
            MOD_FA["Fixed Assets & Depreciation"]
            MOD_RET["Sales & Purchase Returns"]
            MOD_TAX["Taxation & e-Faktur"]
            MOD_WF["Approval Workflow Engine"]
            MOD_SYS["System Platform & Telemetry"]
            MOD_DASH["Executive KPI Engine"]
            MOD_DOC["Corporate Documents Hub & S3 Archive"]
            MOD_PORTAL["B2B Vendor & Customer Portal"]
        end
    end

    subgraph StorageLayer ["Persistence & Caching Infrastructure"]
        PG[("PostgreSQL 16 (Relational DB & Pure DDL Migrations)")]
        VALKEY[("Valkey / Redis (Asynq Queue & Session Cache)")]
        FLOCI[("Floci S3 Adapter (Document Signatures, POD Photos, Attachments)")]
    end

    UI_DASH --> CADDY
    UI_OPS --> CADDY
    UI_FIN --> CADDY
    UI_HR --> CADDY
    UI_DOC --> CADDY
    UI_PORTAL --> CADDY
    UI_SHARED --> CADDY
    CADDY --> AUTH_MW
    AUTH_MW --> CoreBusinessModules

    CoreBusinessModules --> PG
    CoreBusinessModules --> VALKEY
    CoreBusinessModules --> FLOCI
```

---

## 2. Global Data Pipeline & Invariant Enforcement

```mermaid
sequenceDiagram
    autonumber
    actor User as Pengguna Operasional
    participant Web as Web Client (Next.js 15)
    participant MW as Chi Middleware (Branch & RBAC)
    participant Svc as Business Service
    participant Jnl as Accounting Journal Engine
    participant DB as PostgreSQL 16 (pgx/v5 Pool)
    participant S3 as Floci Storage / Queue

    User->>Web: Submit Transaksi Bisnis (e.g. Sales Delivery, Issue Bahan, Payroll Run)
    Web->>MW: HTTP POST /api/v1/... (Bearer Token, Active Branch ID)
    MW->>MW: Verifikasi JWT, Granular Permission (module:action), dan Branch ID
    MW->>Svc: Teruskan Request ke Handler & Service
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: Transaksi Database Terisolasi (ACID)
        Svc->>DB: Begin Tx (pgx.Tx)
        Svc->>DB: Validasi Stok / Status / Constraints
        Svc->>DB: Update Entity Mutation & Mutation Ledger
        
        opt Transaksi Berdampak Keuangan (Double-Entry Invariant)
            Svc->>Jnl: Generate Journal Entry Lines
            Jnl->>Jnl: Validasi SUM(Debit) == SUM(Credit)
            Jnl->>DB: Insert journal_entries & journal_lines
        end
        
        Svc->>DB: Record Audit Trail (actor, branch, diff_before, diff_after)
        Svc->>DB: Commit Tx
    end

    opt Ada Berkas Lampiran / Tanda Tangan Digital
        Svc->>S3: Simpan Bukti ke Floci S3
    end

    Svc-->>Web: JSON Response (Standard Success Envelope)
    Web-->>User: Notifikasi Berhasil & Refresh Table / Tampilkan Dokumen Cetak
```

---

## 3. Peta Modul Backend (20 Modul)

| Modul | Fokus Utama |
| :--- | :--- |
| `iam` | Autentikasi, user, role, permission matrix, user-branch mapping |
| `master` | Cabang, mata uang, UoM, tarif pajak, item, vendor, pelanggan |
| `accounting` | COA PSAK, jurnal double-entry, laporan keuangan, konsolidasi, anggaran |
| `inventory` | Stock ledger perpetual, gudang, bin, transfer, penyesuaian |
| `purchasing` | PR, RFQ, PO, GRN, vendor bill, 3-way matching |
| `sales` | Quotation, sales order, pengiriman, invoice, pembayaran, credit limit |
| `manufacturing` | BOM, routing, work order, job costing, MRP |
| `hrm` | Karyawan, presensi biometrik, cuti, lembur, payroll, ESS |
| `logistics` | Armada, trip SPJ, delivery order, proof of delivery |
| `quality` | Inspeksi QC, sertifikat CoA, NCR |
| `maintenance` | Aset mesin, preventive maintenance, work order breakdown |
| `fixedassets` | Klasifikasi aset, penyusutan, revaluasi, disposal |
| `treasury` | Rekening bank, kas kecil, transfer dana, rekonsiliasi MT940/CSV |
| `taxation` | PPN, PPh, e-Faktur, ekspor SPT |
| `returns` | RMA pelanggan, retur pembelian, debit note vendor |
| `workflow` | Approval multi-tier, threshold, riwayat persetujuan |
| `dashboard` | Ringkasan holding dan dashboard departemen |
| `system` | Audit trail, pengaturan sistem, integrasi perangkat, telemetry |
| `documents` | Tata naskah dinas, register surat, arsip S3 |
| `portal` | Portal mandiri vendor dan pelanggan B2B |

---

## 4. Batas Layanan Portal B2B

Portal bukan modul operasional internal, melainkan permukaan terbatas yang membaca data modul `purchasing`, `sales`, dan `logistics`:

- Pemetaan identitas mitra disimpan pada `portal_user_partners` dan membatasi satu user ke satu vendor atau satu pelanggan per tipe mitra.
- Vendor dapat melihat PO miliknya, mengakui atau menolak PO beserta tanggal janji kirim, dan mengirim Advance Shipping Notice (ASN).
- Pelanggan hanya dapat membaca sales order dan invoice miliknya.
- Setiap endpoint `/api/portal/*` memerlukan autentikasi dan memvalidasi `company_id` serta `branch_id` sebelum data dikembalikan.
- e-Faktur dan dokumen pajak pelanggan belum tersedia melalui portal.

