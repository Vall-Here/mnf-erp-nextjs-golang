# Enterprise ERP Software Design Document (SDD) Matrix

Panduan spesifikasi teknis dan desain arsitektur perangkat lunak (*Software Design Document*) untuk 20 modul inti Industrial Enterprise ERP PT SAWIT JAYA (SWT).

Seluruh spesifikasi mencerminkan implementasi kode aktual:
- **Backend**: Golang 1.24+ Modular Monolith (Clean Architecture), `chi` router, parameterized SQL eksplisit dengan `pgx/v5`, PostgreSQL 16.
- **Frontend**: Next.js 15 (App Router), React 19, TypeScript Strict, Tailwind CSS, TanStack Table v8, React Hook Form, Zod, Zustand.
- **Tata Kelola Industri**: Single Legal Corporate Entity (PT SAWIT JAYA), Single General Ledger, Multi-Site Operational Tagging (`branch_id`), Multi-Warehouse Perpetual Inventory (`warehouse_id`), Granular RBAC dengan Segregation of Duties (SoD), Anti-IDOR Horizontal Isolation, Pure DDL Database Migrations, dan Interactive OpenAPI 3.0 Documentation (`/api/docs`).

---

## Indeks Dokumen Desain Modul (20 Modul)

| No | Modul | Berkas SDD | Diagram yang Tersedia |
| :---: | :--- | :--- | :--- |
| **00** | **System Architecture Overview** | [00_system_architecture_overview.md](./00_system_architecture_overview.md) | Component Diagram, Data Pipeline, Global ERD, Invariant Enforcement |
| **01** | **Accounting & General Ledger** | [01_accounting_sdd.md](./01_accounting_sdd.md) | Use Case, Activity, Sequence, ERD, SoD Post Journal, Budget Encumbrance, PSAK 65 |
| **02** | **Treasury & Bank Reconciliation** | [02_treasury_sdd.md](./02_treasury_sdd.md) | Use Case, Activity, Sequence, ERD, MT940 & CSV Auto-Matching |
| **03** | **Sales & Customer Distribution** | [03_sales_sdd.md](./03_sales_sdd.md) | Use Case, Activity, Sequence, ERD, Credit Limit Hold & Release Otorisasi, POD |
| **04** | **Purchasing & Procurement** | [04_purchasing_sdd.md](./04_purchasing_sdd.md) | Use Case, Activity, Sequence, ERD, PR & PO Approval, RFQ Award, 3-Way Match |
| **05** | **Inventory & Warehouse Management**| [05_inventory_sdd.md](./05_inventory_sdd.md) | Use Case, Activity, Sequence, ERD, Multi-Warehouse STO, SoD Stock Adjustment |
| **06** | **Manufacturing Execution & Costing**| [06_manufacturing_sdd.md](./06_manufacturing_sdd.md) | Use Case, Activity, Sequence, ERD, BOM, SPK Work Order Lifecycle, Scrap Absorption |
| **07** | **Quality Assurance & Control (ISO)**| [07_quality_sdd.md](./07_quality_sdd.md) | Use Case, Activity, Sequence, ERD, IQC, IPQC Palm Oil FFA Testing, Certificate of Analysis |
| **08** | **Logistics & Fleet Transport** | [08_logistics_sdd.md](./08_logistics_sdd.md) | Use Case, Activity, Sequence, ERD, Surat Perintah Jalan (SPJ), Fleet Dispatch |
| **09** | **Plant Maintenance & CMMS** | [09_maintenance_sdd.md](./09_maintenance_sdd.md) | Use Case, Activity, Sequence, ERD, Preventive Maintenance, Machine Reliability |
| **10** | **HRIS & Indonesian Payroll PPh 21** | [10_hrm_payroll_sdd.md](./10_hrm_payroll_sdd.md) | Use Case, Activity, Sequence, ERD, TER PPh 21, Geofenced Clock-In, ESS |
| **11** | **Fixed Assets & Depreciation** | [11_fixedassets_sdd.md](./11_fixedassets_sdd.md) | Use Case, Activity, Sequence, ERD, PSAK 16 Straight-Line Monthly Depreciation |
| **12** | **Returns & RMA (Sales & Purchase)**| [12_returns_sdd.md](./12_returns_sdd.md) | Use Case, Activity, Sequence, ERD, RMA Approval, Vendor Debit Notes |
| **13** | **Taxation Engine (PPN/PPh/e-Faktur)**| [13_taxation_sdd.md](./13_taxation_sdd.md) | Use Case, Activity, Sequence, ERD, NSFP Management, e-Faktur CSV Export |
| **14** | **Multi-Tier Approval Workflow** | [14_workflow_sdd.md](./14_workflow_sdd.md) | Use Case, Activity, Sequence, ERD, Dynamic Steps, Role-Based Matrix, Delegations |
| **15** | **Master Data & Multi-Branch** | [15_master_data_sdd.md](./15_master_data_sdd.md) | Use Case, Activity, Sequence, ERD, Single Entity Governance, Site Tagging |
| **16** | **IAM, RBAC & Security Matrix** | [16_iam_rbac_sdd.md](./16_iam_rbac_sdd.md) | Use Case, Activity, Sequence, ERD, 21 Modules x 7 Actions, SoD Rules, Anti-IDOR |
| **17** | **System Platform & Audit Trail** | [17_system_platform_sdd.md](./17_system_platform_sdd.md) | Use Case, Activity, Sequence, ERD, Diff Logger, Device Health Telemetry |
| **18** | **Executive Dashboard & Analytics** | [18_executive_dashboard_sdd.md](./18_executive_dashboard_sdd.md) | Use Case, Activity, Sequence, ERD, Segmental Margin, OEE, Real-Time Alert Stream |
| **19** | **Corporate Documents Hub** | [19_documents_hub_sdd.md](./19_documents_hub_sdd.md) | Use Case, Activity, Sequence, ERD, Nomor Surat Klasifikasi, S3 Storage Vault |
| **20** | **B2B Vendor & Customer Portal** | [20_b2b_portal_sdd.md](./20_b2b_portal_sdd.md) | Use Case, Activity, Sequence, ERD, Counterparty Scoped Access, Advance Shipping Notice |

---

## Diagram Interaktif Archify

| Diagram Archify | Tipe | Berkas HTML Interaktif | Cakupan Arsitektur |
| :--- | :---: | :--- | :--- |
| **Global ERP System Architecture** | `architecture` | [erp-architecture.html](../architecture/erp-architecture.html) | Next.js 15, Caddy, Golang Monolith, PostgreSQL 16, Valkey, Floci S3 |
| **Order-to-Cash End-to-End** | `workflow` | [order-to-cash-workflow.html](./archify/order-to-cash-workflow.html) | Quotation &rarr; SO &rarr; Multi-Tier Approval &rarr; FEFO &rarr; SPJ &rarr; POD &rarr; Invoicing &rarr; GL |
| **Procure-to-Pay & 3-Way Match** | `workflow` | [procure-to-pay-workflow.html](./archify/procure-to-pay-workflow.html) | PR &rarr; Vendor RFQ &rarr; PO Approval &rarr; GRN IQC &rarr; 3-Way Match &rarr; AP Journal |
| **Double-Entry Journal Posting** | `sequence` | [double-entry-posting.html](./archify/double-entry-posting.html) | Chi Router JWT Guard &rarr; Handler &rarr; Double-Entry Assert &rarr; pgx.Tx &rarr; Audit Diff |
| **Approval Matrix Lifecycle** | `lifecycle` | [approval-matrix-lifecycle.html](./archify/approval-matrix-lifecycle.html) | Draft &rarr; Submitted &rarr; Review &rarr; Revision/Reject &rarr; Approved &rarr; Posted |
| **Manufacturing & Job Costing** | `workflow` | [mfg-job-costing-flow.html](./archify/mfg-job-costing-flow.html) | BOM &rarr; Work Order &rarr; FEFO Material Issue &rarr; Shop Floor &rarr; CoA &rarr; Costing Variance |
