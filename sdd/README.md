# Enterprise ERP Software Design Document (SDD) Matrix

Dokumen ini merupakan panduan spesifikasi teknis dan desain arsitektur perangkat lunak (*Software Design Document*) untuk 18 modul inti Industrial Enterprise ERP.

Seluruh spesifikasi grounded secara langsung pada implementasi kode aktual:
- **Backend**: Golang 1.22+ Modular Monolith (Clean Architecture), `chi` router, `pgx/v5`, PostgreSQL 16.
- **Frontend**: Next.js 15 (App Router), TypeScript Strict, Tailwind CSS, TanStack Table v8, TanStack Query.
- **Invariants**: Accounting Double-Entry Balance (`SUM(debit) == SUM(credit)`), Perpetual Inventory, Multi-Tier RBAC, Audit Trail, Multi-Branch Governance (PSAK 65 / IFRS 10), and Pure DDL Migrations.

---

## Indeks Dokumen Desain Modul (18 Modul)

| No | Modul | Berkas SDD | Diagram yang Tersedia |
| :---: | :--- | :--- | :--- |
| **00** | **System Architecture Overview** | [00_system_architecture_overview.md](file:///e:/project/Next/ERP/docs/sdd/00_system_architecture_overview.md) | Component Diagram, Data Pipeline, Global ERD |
| **01** | **Accounting & General Ledger** | [01_accounting_sdd.md](file:///e:/project/Next/ERP/docs/sdd/01_accounting_sdd.md) | Use Case, Activity, Sequence, ERD |
| **02** | **Treasury & Bank Reconciliation** | [02_treasury_sdd.md](file:///e:/project/Next/ERP/docs/sdd/02_treasury_sdd.md) | Use Case, Activity, Sequence, ERD |
| **03** | **Sales & Customer Distribution** | [03_sales_sdd.md](file:///e:/project/Next/ERP/docs/sdd/03_sales_sdd.md) | Use Case, Activity, Sequence, ERD |
| **04** | **Purchasing & Procurement** | [04_purchasing_sdd.md](file:///e:/project/Next/ERP/docs/sdd/04_purchasing_sdd.md) | Use Case, Activity, Sequence, ERD |
| **05** | **Inventory & Warehouse Management**| [05_inventory_sdd.md](file:///e:/project/Next/ERP/docs/sdd/05_inventory_sdd.md) | Use Case, Activity, Sequence, ERD |
| **06** | **Manufacturing Execution & Costing**| [06_manufacturing_sdd.md](file:///e:/project/Next/ERP/docs/sdd/06_manufacturing_sdd.md) | Use Case, Activity, Sequence, ERD |
| **07** | **Quality Assurance & Control (ISO)**| [07_quality_sdd.md](file:///e:/project/Next/ERP/docs/sdd/07_quality_sdd.md) | Use Case, Activity, Sequence, ERD |
| **08** | **Logistics & Fleet Transport** | [08_logistics_sdd.md](file:///e:/project/Next/ERP/docs/sdd/08_logistics_sdd.md) | Use Case, Activity, Sequence, ERD |
| **09** | **Plant Maintenance & CMMS** | [09_maintenance_sdd.md](file:///e:/project/Next/ERP/docs/sdd/09_maintenance_sdd.md) | Use Case, Activity, Sequence, ERD |
| **10** | **HRIS & Indonesian Payroll PPh 21** | [10_hrm_payroll_sdd.md](file:///e:/project/Next/ERP/docs/sdd/10_hrm_payroll_sdd.md) | Use Case, Activity, Sequence, ERD |
| **11** | **Fixed Assets & Depreciation** | [11_fixedassets_sdd.md](file:///e:/project/Next/ERP/docs/sdd/11_fixedassets_sdd.md) | Use Case, Activity, Sequence, ERD |
| **12** | **Returns & RMA (Sales & Purchase)**| [12_returns_sdd.md](file:///e:/project/Next/ERP/docs/sdd/12_returns_sdd.md) | Use Case, Activity, Sequence, ERD |
| **13** | **Taxation Engine (PPN/PPh/e-Faktur)**| [13_taxation_sdd.md](file:///e:/project/Next/ERP/docs/sdd/13_taxation_sdd.md) | Use Case, Activity, Sequence, ERD |
| **14** | **Multi-Tier Approval Workflow** | [14_workflow_sdd.md](file:///e:/project/Next/ERP/docs/sdd/14_workflow_sdd.md) | Use Case, Activity, Sequence, ERD |
| **15** | **Master Data & Multi-Branch** | [15_master_data_sdd.md](file:///e:/project/Next/ERP/docs/sdd/15_master_data_sdd.md) | Use Case, Activity, Sequence, ERD |
| **16** | **IAM, RBAC & Security Matrix** | [16_iam_rbac_sdd.md](file:///e:/project/Next/ERP/docs/sdd/16_iam_rbac_sdd.md) | Use Case, Activity, Sequence, ERD |
| **17** | **System Platform & Audit Trail** | [17_system_platform_sdd.md](file:///e:/project/Next/ERP/docs/sdd/17_system_platform_sdd.md) | Use Case, Activity, Sequence, ERD |
| **18** | **Executive Dashboard & Analytics** | [18_executive_dashboard_sdd.md](file:///e:/project/Next/ERP/docs/sdd/18_executive_dashboard_sdd.md) | Use Case, Activity, Sequence, ERD |

---

## Peta Diagram Interaktif Archify (Standalone HTML Viewer)

Selain diagram Mermaid di dalam dokumen spesifikasi, sistem menyediakan visualisasi interaktif **Archify** *(zoom, pan, trace motion, multi-view focus, dark/light mode)* yang dapat dibuka langsung di browser:

| Diagram Archify | Tipe | Berkas HTML Interaktif | Cakupan Arsitektur |
| :--- | :---: | :--- | :--- |
| **Global ERP System Architecture** | `architecture` | [erp-architecture.html](file:///e:/project/Next/ERP/docs/architecture/erp-architecture.html) | Next.js 15, Caddy, Golang Monolith, PostgreSQL 16, Valkey, Floci S3 |
| **Order-to-Cash End-to-End** | `workflow` | [order-to-cash-workflow.html](file:///e:/project/Next/ERP/docs/sdd/archify/order-to-cash-workflow.html) | Quotation &rarr; SO &rarr; Multi-Tier Approval &rarr; FEFO &rarr; SPJ &rarr; POD &rarr; Invoicing &rarr; GL |
| **Procure-to-Pay & 3-Way Match** | `workflow` | [procure-to-pay-workflow.html](file:///e:/project/Next/ERP/docs/sdd/archify/procure-to-pay-workflow.html) | PR &rarr; Vendor RFQ &rarr; PO Approval &rarr; GRN IQC &rarr; 3-Way Match &rarr; AP Journal |
| **Double-Entry Journal Posting** | `sequence` | [double-entry-posting.html](file:///e:/project/Next/ERP/docs/sdd/archify/double-entry-posting.html) | Chi Router JWT Guard &rarr; Handler &rarr; Double-Entry Assert &rarr; pgx.Tx &rarr; Audit Diff |
| **Approval Matrix Lifecycle** | `lifecycle` | [approval-matrix-lifecycle.html](file:///e:/project/Next/ERP/docs/sdd/archify/approval-matrix-lifecycle.html) | Draft &rarr; Submitted &rarr; Review &rarr; Revision/Reject &rarr; Approved &rarr; Posted |
| **Manufacturing & Job Costing** | `workflow` | [mfg-job-costing-flow.html](file:///e:/project/Next/ERP/docs/sdd/archify/mfg-job-costing-flow.html) | BOM &rarr; Work Order &rarr; FEFO Material Issue &rarr; Shop Floor &rarr; CoA &rarr; Costing Variance |

---

## Standar Notasi Diagram Modul (18 Berkas SDD)

Setiap modul dianalisis dan dimodelkan dengan 4 pilar diagram standar:
1. **Use Case Diagram**: Menggambarkan interaksi aktor manusia dan sub-sistem terhadap kapabilitas sistem.
2. **Activity / Process Flow Diagram**: Menggambarkan alur kerja langkah-demi-langkah, percabangan keputusan, dan penanganan kondisi kegagalan.
3. **Sequence Diagram**: Memetakan urutan eksekusi sinkron/asinkron antar-layer (*Frontend Client &rarr; API Handler &rarr; Service Domain &rarr; Database & Transaksi Jurnal*).
4. **Entity Relationship Diagram (ERD)**: Memetakan skema tabel, tipe data, kunci primer/asing, dan relasi integritas referensial.
