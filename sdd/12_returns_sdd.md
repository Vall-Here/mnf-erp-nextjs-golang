# 12. Software Design Document: Sales & Purchase Returns (RMA)

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Returns & RMA** mengelola penanganan pengembalian barang dari pelanggan (*Sales Return / RMA*) dan pengembalian barang ke pemasok (*Purchase Return / RTV*): Otorisasi Retur, Penerimaan/Pengeluaran Fisik di Gudang Karantina, Penerbitan Nota Kredit (*Credit Memo - Piutang*) & Nota Debet (*Debit Memo - Utang*), serta Jurnal Pembalik Akuntansi Otomatis.

- **Lokasi Backend**: `services/api/internal/modules/returns/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/returns/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    CustService["Customer Service / Sales"]
    WhReceiver["Staff Gudang Karantina"]
    Buyer["Procurement Specialist"]
    FinAcct["Akuntan Keuangan"]

    subgraph ReturnsModule ["Modul Returns & RMA"]
        UC1["Pengajuan Otorisasi Retur Penjualan (RMA)"]
        UC2["Penerimaan & Inspeksi Fisik Barang Retur"]
        UC3["Penerbitan Nota Kredit (Credit Memo AR)"]
        UC4["Pengajuan Retur Pembelian ke Pemasok (RTV)"]
        UC5["Pengiriman Barang Retur ke Vendor"]
        UC6["Penerbitan Nota Debet (Debit Memo AP)"]
        UC7["Auto-Posting Jurnal Pembalik Retur & PPN"]
    end

    CustService --> UC1
    WhReceiver --> UC2
    FinAcct --> UC3

    Buyer --> UC4
    WhReceiver --> UC5
    FinAcct --> UC6
    FinAcct --> UC7
```

---

## 3. Activity Diagram (Alur Penanganan Retur Penjualan / RMA)

```mermaid
flowchart TD
    Start([Klaim Retur Pelanggan]) --> CreateRMA[Buat Dokumen Sales Return / RMA]
    CreateRMA --> ApproveRMA[Persetujuan Otorisasi Retur oleh Sales Lead]
    
    ApproveRMA --> ReceiveGoods[Barang Tiba di Gudang Karantina]
    ReceiveGoods --> InspectCondition{Hasil Inspeksi Fisik QC}
    
    InspectCondition -- Layak Jual Kembali --> Restock[Restock ke Gudang Barang Jadi]
    InspectCondition -- Rusak / Cacat --> ScrapQuarantine[Masuk Karantina / Scrap]
    
    Restock --> GenCreditMemo[Terbitkan Nota Kredit / Credit Memo]
    ScrapQuarantine --> GenCreditMemo
    
    GenCreditMemo --> DeductAR[Kurangi Saldo Piutang Pelanggan]
    DeductAR --> AutoJournal[Auto-Posting Jurnal: Debit Retur Penjualan + PPN, Kredit Piutang]
    
    AutoJournal --> PrintDoc[Cetak Dokumen Nota Retur Pajak & Credit Memo]
    PrintDoc --> EndDone([Retur Selesai])
```

---

## 4. Sequence Diagram (Penerbitan Credit Memo & Penyesuaian Saldo Piutang)

```mermaid
sequenceDiagram
    autonumber
    actor Officer as Staf Keuangan / AR
    participant UI as Returns Web (Next.js 15)
    participant Hdl as Returns Handler
    participant Svc as Returns Service
    participant Acct as Accounting Module (GL)
    participant Repo as Returns Repository
    participant DB as PostgreSQL 16

    Officer->>UI: Submit Verifikasi Retur & Terbitkan Credit Memo
    UI->>Hdl: POST /api/v1/returns/sales/{id}/credit-memo
    Hdl->>Svc: IssueCreditMemo(ctx, returnID, input)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: GetSalesReturn(returnID)
        Repo->>DB: SELECT * FROM sales_returns WHERE id = ...
        
        Svc->>Repo: Insert Credit Memo
        Repo->>DB: INSERT INTO credit_memos (return_id, customer_id, memo_number, amount, tax_amount)
        
        Svc->>Acct: CreateReturnJournal(Debit Retur Penjualan, Debit PPN Keluaran, Kredit Piutang Usaha)
        Acct->>DB: INSERT INTO journal_entries & journal_lines
        
        Svc->>Repo: Update Sales Return Status to PROCESSED
        Repo->>DB: UPDATE sales_returns SET status = 'PROCESSED' WHERE id = ...
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Credit Memo Entity
    Hdl-->>UI: 201 Created
    UI-->>Officer: Tampilkan Dokumen Resmi Nota Kredit (Credit Memo A4)
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    SALES_INVOICES ||--o{ SALES_RETURNS : "returned_from"
    SALES_RETURNS ||--|{ SALES_RETURN_ITEMS : "contains"
    SALES_RETURNS ||--o| CREDIT_MEMOS : "credited_as"
    PURCHASE_ORDERS ||--o{ PURCHASE_RETURNS : "returned_from"
    PURCHASE_RETURNS ||--|{ PURCHASE_RETURN_ITEMS : "contains"
    PURCHASE_RETURNS ||--o| DEBIT_MEMOS : "debited_as"

    SALES_RETURNS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid invoice_id FK
        uuid customer_id FK
        string return_number UK
        date return_date
        string reason
        string status "DRAFT | APPROVED | RECEIVED | PROCESSED | REJECTED"
    }

    SALES_RETURN_ITEMS {
        uuid id PK
        uuid sales_return_id FK
        uuid item_id FK
        numeric quantity_returned
        numeric unit_price
        string condition "GOOD | DAMAGED | REWORKABLE"
        string disposition "RESTOCK | SCRAP"
    }

    CREDIT_MEMOS {
        uuid id PK
        uuid sales_return_id FK
        uuid customer_id FK
        string memo_number UK
        date memo_date
        numeric subtotal
        numeric tax_amount
        numeric total_credit_amount
        uuid journal_id FK
    }

    PURCHASE_RETURNS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid vendor_id FK
        string return_number UK
        date return_date
        string reason
        string status "DRAFT | APPROVED | SHIPPED | PROCESSED"
    }

    DEBIT_MEMOS {
        uuid id PK
        uuid purchase_return_id FK
        uuid vendor_id FK
        string memo_number UK
        date memo_date
        numeric total_debit_amount
        uuid journal_id FK
    }
```
