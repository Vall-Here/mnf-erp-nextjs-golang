# 18. Software Design Document: Executive Dashboard & Analytics

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Executive Dashboard & Analytics** menyediakan ringkasan analitik tingkat dewan direksi (*C-Level Decision Support System*): Proyeksi Arus Kas & *Cash Runway* (30-60-90 Hari), Saldo Likuiditas Bank Terkonsolidasi, Saldo Piutang (*AR Aging*) vs Utang (*AP Aging*), Kinerja Saluran Penjualan (*Sales Pipeline*), Tingkat Utilisasi Pabrik (*OEE & Downtime*), dan Kepatuhan Tata Kelola Mutu ISO.

- **Lokasi Backend**: `services/api/internal/modules/dashboard/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/page.tsx`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    CEO["Chief Executive Officer (CEO)"]
    CFO["Chief Financial Officer (CFO)"]
    COO["Chief Operating Officer (COO)"]
    BranchLead["Kepala Cabang (Branch Manager)"]

    subgraph DashboardModule ["Modul Executive Dashboard & Analytics"]
        UC1["Pantau Saldo Likuiditas Kas & Bank Seluruh Cabang"]
        UC2["Proyeksi Arus Kas Masuk & Keluar (30-60-90 Hari)"]
        UC3["Analisis Perputaran Piutang (AR Aging) & Risiko Macet"]
        UC4["Pantau Realisasi Pendapatan vs Target Penjualan"]
        UC5["Pantau Efisiensi Produksi Pabrik & Jam Henti Mesin (Downtime)"]
        UC6["Pemantauan Status Kepatuhan Mutu & Insiden NCR Aktif"]
        UC7["Konsolidasi Laporan Keuangan Lintas Entitas (PSAK 65)"]
    end

    CEO --> UC1
    CEO --> UC4
    CEO --> UC7

    CFO --> UC1
    CFO --> UC2
    CFO --> UC3
    CFO --> UC7

    COO --> UC4
    COO --> UC5
    COO --> UC6

    BranchLead --> UC1
    BranchLead --> UC4
    BranchLead --> UC5
```

---

## 3. Activity Diagram (Agregasi Data Finansial & Proyeksi Arus Kas)

```mermaid
flowchart TD
    Start([Eksekutif Membuka Dashboard]) --> FetchContext[Ambil Konteks Cabang: Semua Cabang atau Cabang Tunggal]
    FetchContext --> QueryBank[Ambil Saldo Real-Time Seluruh Rekening Bank Aktif]
    
    QueryBank --> QueryAR[Ambil Jadwal Jatuh Tempo Piutang Penjualan / AR Invoices]
    QueryAR --> QueryAP[Ambil Jadwal Jatuh Tempo Utang Pemasok / AP Bills]
    QueryAP --> QueryRecurring[Ambil Estimasi Beban Rutin: Payroll, Sewa, Utilitas]
    
    QueryRecurring --> AggregateCashFlow[Kalkulasi Proyeksi Likuiditas Harian]
    AggregateCashFlow --> BucketPeriod[Kelompokkan ke Horizon Waktu: 30 Hari, 60 Hari, 90 Hari]
    
    BucketPeriod --> CalcRunway[Hitung Cash Runway = Saldo Kas / Rata-Rata Burn Rate Bulanan]
    CalcRunway --> RenderCharts[Render Grafik Tren Likuiditas & Matriks KPI]
    
    RenderCharts --> CheckAlerts{Apakah Ada Anomali Likuiditas / Defisit?}
    CheckAlerts -- Defisit Terdeteksi --> ShowWarning[Tampilkan Peringatan Dini Likuiditas Kas]
    CheckAlerts -- Aman --> ShowNormal[Status Likuiditas Aman & Prima]
    
    ShowWarning --> EndDone([Dashboard Siap])
    ShowNormal --> EndDone
```

---

## 4. Sequence Diagram (Kueri Agregasi Real-Time Executive KPI)

```mermaid
sequenceDiagram
    autonumber
    actor Exec as Direktur Keuangan / CFO
    participant UI as Dashboard Web (Next.js 15)
    participant Hdl as Dashboard Handler
    participant Svc as Dashboard Service
    participant Repo as Dashboard Repository
    participant DB as PostgreSQL 16

    Exec->>UI: Akses Halaman Utama Dashboard Eksekutif
    UI->>Hdl: GET /api/v1/dashboard/summary?branch_id=ALL
    Hdl->>Svc: GetExecutiveSummary(ctx, companyID, branchID)
    
    rect rgb(240, 248, 255)
        Note over Svc,DB: High-Performance Parallel Queries
        Svc->>Repo: GetTotalBankBalance()
        Repo->>DB: SELECT COALESCE(SUM(current_balance), 0) FROM bank_accounts WHERE is_active = true
        DB-->>Repo: Total Cash Balance
        
        Svc->>Repo: GetARAgingSummary()
        Repo->>DB: SELECT status, SUM(grand_total - paid_amount) FROM sales_invoices GROUP BY ...
        DB-->>Repo: AR Aging Buckets
        
        Svc->>Repo: GetAPAgingSummary()
        Repo->>DB: SELECT status, SUM(grand_total) FROM vendor_bills GROUP BY ...
        DB-->>Repo: AP Aging Buckets
        
        Svc->>Repo: GetPlantDowntimeKPI()
        Repo->>DB: SELECT COALESCE(SUM(downtime_hours), 0) FROM maintenance_work_orders WHERE ...
        DB-->>Repo: Downtime Metrics
        
        Svc->>Svc: Kalkulasi Cash Runway & Rasio Likuiditas
    end

    Svc-->>Hdl: Consolidated Dashboard Summary Payload
    Hdl-->>UI: 200 OK (Executive Metrics JSON)
    UI-->>Exec: Tampilkan Kartu KPI, Grafik Tren Arus Kas & Matriks Operasional
```

---

## 5. Entity Relationship Diagram (ERD & Keterikatan Data Agregasi)

```mermaid
erDiagram
    COMPANIES ||--o{ BRANCHES : "monitors"
    BRANCHES ||--o{ BANK_ACCOUNTS : "holds"
    BRANCHES ||--o{ SALES_INVOICES : "collects"
    BRANCHES ||--o{ VENDOR_BILLS : "owes"
    BRANCHES ||--o{ WORK_ORDERS : "manufactures"
    BRANCHES ||--o{ MAINTENANCE_WORK_ORDERS : "maintains"

    DASHBOARD_METRICS_SNAPSHOT {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        date snapshot_date
        numeric total_cash_balance
        numeric total_ar_receivable
        numeric total_ap_payable
        numeric net_cash_flow_forecast_30d
        numeric net_cash_flow_forecast_60d
        numeric net_cash_flow_forecast_90d
        numeric total_plant_downtime_hours
        numeric active_ncr_incidents_count
        timestamp calculated_at
    }

    BANK_ACCOUNTS {
        uuid id PK
        string account_name
        numeric current_balance
    }

    SALES_INVOICES {
        uuid id PK
        numeric grand_total
        numeric paid_amount
        date due_date
    }

    VENDOR_BILLS {
        uuid id PK
        numeric grand_total
        date due_date
    }

    WORK_ORDERS {
        uuid id PK
        numeric planned_qty
        numeric completed_qty
    }

    MAINTENANCE_WORK_ORDERS {
        uuid id PK
        numeric downtime_hours
    }
```
