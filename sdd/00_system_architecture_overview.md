# 00. System Architecture Overview & Global Design

## 1. Ikhtisar Arsitektur Sistem

Enterprise ERP dibangun menggunakan arsitektur **Modular Monolith** dengan prinsip *Clean Architecture* di sisi Backend (Golang) dan *Next.js 15 App Router* di sisi Frontend.

```mermaid
graph TB
    subgraph ClientLayer ["Frontend Client Layer (Next.js 15 App Router)"]
        UI_DASH["Dashboard & Executive Views"]
        UI_OPS["Operational Modules (Sales, Purchasing, Inventory, Mfg, Logistics, CMMS)"]
        UI_FIN["Financial Modules (Accounting, Treasury, Tax, Fixed Assets)"]
        UI_HR["HRM & Payroll"]
        UI_SHARED["Shared Components (TanStack Table, Print Engine, Multi-Tier Approval Modal)"]
    end

    subgraph Gateway ["Reverse Proxy & Web Server"]
        CADDY["Caddy Reverse Proxy (Auto HTTPS / SSL & Static Routing)"]
    end

    subgraph BackendMonolith ["Backend Modular Monolith (Golang 1.22+ / Chi Router)"]
        AUTH_MW["Middleware (JWT Auth, RBAC Matrix, Tenant/Branch Guard, Audit Logger)"]
        
        subgraph CoreBusinessModules ["18 Core Business Modules"]
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
