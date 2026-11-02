# PT SAWIT JAYA (SWT) - Industrial ERP Architecture & Technical Showcase

Dokumentasi arsitektur perangkat lunak dan spesifikasi teknis untuk sistem **Enterprise Resource Planning (ERP)** industri terpadu berbasis **Golang 1.24+** (Clean Architecture Modular Monolith) dan **Next.js 15 App Router** (React 19, TypeScript Strict).

---

## 1. Tata Kelola Korporat & Entitas Legal

Sistem mengimplementasikan model tata kelola industri perseroan tunggal terpadu (**Single Corporate Legal Entity**):
- **Entitas Korporasi**: **PT SAWIT JAYA (SWT)**.
- **Buku Besar Terpadu (Single General Ledger)**: Seluruh transaksi finansial bermuara pada satu Buku Besar korporat yang memenuhi standar akuntansi PSAK / IFRS.
- **Fasilitas Fisik Operasional (`branch_id`)**: Berfungsi sebagai operational site tag untuk fasilitas fisik (misalnya: Kantor Pusat & Pabrik Kelapa Sawit PKS-01, Pabrik PKS-02, dan Distribution Center DC-03) guna evaluasi performa biaya segmental (*Cost Centers*).
- **Multi-Warehouse Governance**: Pergerakan fisik persediaan tercatat dan terlacak secara akurat melalui `warehouse_id` dan mutasi persediaan internal (*Stock Transfer Order* / STO).

---

## 2. Pilar Invarian Bisnis Industri

1. **Accounting Double-Entry**: Setiap entri jurnal wajib memenuhi invarian mutlak `SUM(debit) == SUM(credit)`. Jurnal tidak seimbang ditolak di lapisan service dan database.
2. **Perpetual Inventory**: Setiap mutasi fisik barang (masuk, keluar, transfer, penyesuaian) mencatat histori permanen tidak dapat diubah (*immutable ledger record*) pada `stock_ledger` dengan agregat real-time pada `item_stock_summary`.
3. **Granular RBAC & Segregation of Duties (SoD)**: Setiap mutasi dan pembacaan data diverifikasi terhadap matriks izin `module:action`. Transaksi berisiko tinggi (posting jurnal, persetujuan penyesuaian stok, pelepasan tahanan kredit, penetapan tender) mewajibkan otorisasi persetujuan terpisah (*maker-checker segregation*).
4. **Horizontal Branch Isolation (Anti-IDOR)**: Pengguna non-Super Admin terkunci secara otomatis pada cabang wewenang operasionalnya (`GetAuthorizedBranchID`), mencegah eskalasi hak akses horizontal antar-fasilitas.
5. **Audit Trail Menyeluruh**: Setiap mutasi data merekam jejak audit (`actor_id`, `branch_id`, `action`, `resource`, `diff_before`, `diff_after`, `created_at`).
6. **Pure DDL Database Migrations**: Seluruh migrasi database di `services/api/internal/db/migrations/` murni DDL (`CREATE TABLE`, `ALTER TABLE`, `CREATE INDEX`). Inisialisasi data master dan demo dipusatkan pada `services/api/cmd/seed/main.go`.
7. **Interactive OpenAPI 3.0**: Seluruh endpoint RESTful tersinkronisasi dalam spesifikasi OpenAPI 3.0 dan dapat diakses interaktif melalui rute `/api/docs`.

---

## 3. Direktori Dokumentasi & Spesifikasi Teknis

- [Portal Dokumentasi Interaktif](./index.html)
- [Diagram Arsitektur Sistem Global](./architecture/erp-architecture.html)
- [Spesifikasi Arsitektur Backend (Golang)](./backend/backend-architecture.html)
- [Spesifikasi Arsitektur Frontend (Next.js 15)](./frontend/frontend-architecture.html)
- [Matriks Software Design Document (SDD 20 Modul)](./sdd/README.md)
