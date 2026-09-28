# 10. Software Design Document: HRIS & Indonesian Statutory Payroll

## 1. Ringkasan Modul & Tanggung Jawab
Modul **HRIS & Indonesian Statutory Payroll** mengelola siklus ketenagakerjaan dan penggajian sesuai regulasi Indonesia: Data Induk Karyawan (*Employee Profiles*), Presensi & Lembur (*Attendance & Overtime*), Permohonan Cuti (*Leaves*), Perhitungan Pajak PPh 21 Skema TER (*PP 58/2023 & PMK 168/2023*), Iuran BPJS Ketenagakerjaan & BPJS Kesehatan, *Auto-Posting* Jurnal Beban Gaji ke General Ledger, serta Penerbitan Slip Gaji Rahasia (*Confidential Payslips*).

- **Lokasi Backend**: `services/api/internal/modules/hrm/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/hrm/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Emp["Karyawan (Employee)"]
    HRStaff["Staf HR & GA"]
    PayrollOfficer["Payroll Specialist"]
    FinMgr["Manajer Keuangan / CFO"]

    subgraph HRMModule ["Modul HRIS & Statutory Payroll"]
        UC1["Kelola Database Profil Karyawan & PTKP"]
        UC2["Pencatatan Presensi & Jam Lembur"]
        UC3["Pengajuan & Otorisasi Cuti / Izin"]
        UC4["Pemrosesan Batch Payroll Bulanan"]
        UC5["Perhitungan Pajak PPh 21 TER & Iuran BPJS"]
        UC6["Auto-Posting Jurnal Beban Gaji ke GL"]
        UC7["Cetak & Distribusi Slip Gaji Rahasia (Payslip)"]
        UC8["Otorisasi Pembayaran Gaji Karyawan"]
    end

    Emp --> UC2
    Emp --> UC3
    Emp --> UC7

    HRStaff --> UC1
    HRStaff --> UC2
    HRStaff --> UC3

    PayrollOfficer --> UC4
    PayrollOfficer --> UC5
    PayrollOfficer --> UC6
    PayrollOfficer --> UC7

    FinMgr --> UC8
```

---

## 3. Activity Diagram (Pemrosesan Batch Payroll & Pembukuan GL)

```mermaid
flowchart TD
    Start([Inisiasi Periode Penggajian]) --> FetchActive[Ambil Seluruh Karyawan Aktif Cabang]
    FetchActive --> FetchAtt[Agregasi Data Presensi & Total Jam Lembur]
    FetchAtt --> CalcGross[Hitung Penghasilan Bruto: Gaji Pokok + Tunjangan + Lembur]
    
    CalcGross --> CheckPTKP[Tentukan Kategori TER Berdasarkan Status PTKP]
    CheckPTKP --> CalcTER[Kalkulasi Potongan Pajak PPh 21 TER Bulan Berjalan]
    
    CalcTER --> CalcBPJS[Hitung Iuran BPJS TK & BPJS Kes Porsi Karyawan dan Perusahaan]
    CalcBPJS --> CalcNet[Hitung Gaji Bersih / Take-Home Pay]
    
    CalcNet --> GenerateSlips[Buat Rincian Payroll Slip per Karyawan]
    GenerateSlips --> CheckBatchTotal[Rekapitulasi Total Beban & Utang Gaji]
    
    CheckBatchTotal --> RequestFinApprove[Minta Otorisasi Penggajian Manajer Keuangan]
    RequestFinApprove --> IsApproved{Disetujui?}
    
    IsApproved -- Ditolak --> ReviewCalc[Evaluasi Rincian & Koreksi Komponen]
    ReviewCalc --> CalcGross
    
    IsApproved -- Disetujui --> PostBatch[Kunci Batch Payroll Status DISBURSED]
    PostBatch --> AutoJournal[Auto-Posting Jurnal Gaji ke General Ledger]
    
    AutoJournal --> DistributePayslip[Terbitkan Slip Gaji Digital Karyawan]
    DistributePayslip --> EndDone([Penggajian Selesai])
```

---

## 4. Sequence Diagram (Eksekusi Batch Payroll & Pembuatan Jurnal Gaji)

```mermaid
sequenceDiagram
    autonumber
    actor Officer as Payroll Specialist
    participant UI as HRM Web (Next.js 15)
    participant Hdl as HRM Handler
    participant Svc as HRM Service
    participant Acct as Accounting Module (GL)
    participant Repo as HRM Repository
    participant DB as PostgreSQL 16

    Officer->>UI: Klik "Jalankan Proses Payroll Periode"
    UI->>Hdl: POST /api/v1/hrm/payroll-runs
    Hdl->>Svc: ProcessPayrollRun(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: GetActiveEmployees(branchID)
        Repo->>DB: SELECT * FROM employees WHERE status = 'ACTIVE' ...
        DB-->>Repo: List Karyawan Aktif
        
        loop Tiap Karyawan
            Svc->>Svc: Hitung PPh 21 TER (Kategori A/B/C) & BPJS TK/Kes
            Svc->>Repo: Insert Payroll Slip Record
            Repo->>DB: INSERT INTO payroll_slips (employee_id, base_salary, allowances, pph21_amount, bpjs_tk, bpjs_kes, net_salary)
        end
        
        Svc->>Acct: CreateAutoJournal(Debit Beban Gaji, Kredit Utang Gaji, Utang PPh 21, Utang BPJS)
        Acct->>DB: INSERT INTO journal_entries & journal_lines (Double-Entry Balanced)
        
        Svc->>Repo: Update Payroll Run Status to DISBURSED
        Repo->>DB: UPDATE payroll_runs SET status = 'DISBURSED' WHERE id = ...
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Payroll Run Summary
    Hdl-->>UI: 201 Created
    UI-->>Officer: Tampilkan Hasil Penggajian & Opsi Cetak Slip Gaji (Payslip A4)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    EMPLOYEES ||--o{ ATTENDANCE_RECORDS : "clocks"
    EMPLOYEES ||--o{ LEAVE_REQUESTS : "applies"
    EMPLOYEES ||--o{ PAYROLL_SLIPS : "receives"
    PAYROLL_RUNS ||--|{ PAYROLL_SLIPS : "contains"
    PAYROLL_RUNS ||--o| JOURNAL_ENTRIES : "posts_to"

    EMPLOYEES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string employee_code UK
        string full_name
        string email
        string phone
        string department
        string position
        string employment_status "PERMANENT | CONTRACT | PROBATION"
        string ptkp_status "TK/0 | TK/1 | TK/2 | TK/3 | K/0 | K/1 | K/2 | K/3"
        string npwp
        string bank_account_number
        numeric base_salary
        numeric transport_allowance
        numeric meal_allowance
        boolean is_active
    }

    ATTENDANCE_RECORDS {
        uuid id PK
        uuid employee_id FK
        date attendance_date
        time check_in_time
        time check_out_time
        numeric overtime_hours
        string status "PRESENT | LATE | ABSENT | LEAVE"
    }

    LEAVE_REQUESTS {
        uuid id PK
        uuid employee_id FK
        string leave_type "ANNUAL | SICK | MATERNITY | SPECIAL"
        date start_date
        date end_date
        integer total_days
        string reason
        string status "PENDING | APPROVED | REJECTED"
        uuid approved_by FK
    }

    PAYROLL_RUNS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        integer period_year
        integer period_month
        numeric total_gross_amount
        numeric total_deductions_amount
        numeric total_net_amount
        string status "DRAFT | PROCESSED | APPROVED | DISBURSED"
        uuid journal_id FK
        timestamp executed_at
    }

    PAYROLL_SLIPS {
        uuid id PK
        uuid payroll_run_id FK
        uuid employee_id FK
        numeric base_salary
        numeric allowances
        numeric overtime_pay
        numeric bpjs_tk_deduction
        numeric bpjs_kes_deduction
        numeric pph21_amount
        numeric net_salary
        string payment_status "PENDING | TRANSFERRED"
    }
```
