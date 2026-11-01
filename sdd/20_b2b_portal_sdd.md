# 20. Software Design Document: B2B Vendor & Customer Portal

## 1. Ringkasan Modul & Tanggung Jawab
Modul **B2B Vendor & Customer Portal** menyediakan akses mandiri terbatas bagi mitra eksternal. Vendor dapat melihat purchase order miliknya, mengakui atau menolak PO beserta tanggal janji kirim, dan mengirim Advance Shipping Notice (ASN). Pelanggan dapat melihat sales order dan invoice miliknya dalam mode baca. Portal tidak membuka data internal lintas mitra dan tidak menggantikan modul operasional internal.

- **Lokasi Backend**: `services/api/internal/modules/portal/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/portal/`
- **Tabel Utama**: `portal_user_partners`, `vendor_po_acknowledgments`, `vendor_asns`, `vendor_asn_items`
- **Endpoint**:
  - `GET /api/portal/profile`
  - `GET /api/portal/vendor/pos`
  - `GET /api/portal/vendor/pos/{id}`
  - `POST /api/portal/vendor/pos/{id}/acknowledge`
  - `GET /api/portal/vendor/asns`
  - `POST /api/portal/vendor/asns`
  - `GET /api/portal/customer/orders`
  - `GET /api/portal/customer/invoices`
- **Akun Demo**: `vendor.krakatau@erp.local` dan `customer.astra@erp.local` dengan password `DemoPassword2026!`

Status acknowledgment PO: `ACCEPTED`, `REJECTED`, `REVISION_REQUESTED`.
Status ASN: `IN_TRANSIT`, `ARRIVED_AT_GATE`, `INSPECTED`, `RECEIVED`, `CANCELLED`.

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    VendorUser["Vendor Partner (External User)"]
    CustomerUser["Customer Partner (External User)"]
    Purchasing["Staf Purchasing Internal"]
    Sales["Staf Sales Internal"]

    subgraph PortalModule ["Modul B2B Vendor & Customer Portal"]
        UC1["Melihat Profil Mitra dan Ketentuan Pembayaran"]
        UC2["Melihat Daftar dan Detail PO Milik Vendor"]
        UC3["Mengakui atau Menolak PO dengan Tanggal Janji Kirim"]
        UC4["Mengirim Advance Shipping Notice (ASN) dan Item"]
        UC5["Melihat Daftar ASN yang Pernah Dikirim"]
        UC6["Melihat Daftar Sales Order Milik Pelanggan"]
        UC7["Melihat Daftar Invoice dan Status Pembayaran"]
    end

    VendorUser --> UC1
    VendorUser --> UC2
    VendorUser --> UC3
    VendorUser --> UC4
    VendorUser --> UC5

    CustomerUser --> UC1
    CustomerUser --> UC6
    CustomerUser --> UC7

    Purchasing --> UC2
    Sales --> UC6
```

---

## 3. Activity Diagram (Akseptasi PO dan Pengiriman ASN oleh Vendor)

```mermaid
flowchart TD
    Start([Vendor Login ke /portal]) --> ResolveProfile[Resolver portal_user_partners Berdasarkan User ID]
    ResolveProfile --> HasPartner{User Terpetakan ke Vendor atau Customer?}
    HasPartner -- Tidak --> DenyAccess[Tolak Akses 403]
    HasPartner -- Ya --> LoadPO[Ambil PO Milik Vendor dan company_id Terkait]

    LoadPO --> ViewPO[Vendor Melihat Detail PO dan Item]
    ViewPO --> AckDecision{Keputusan Vendor}
    AckDecision -- Tolak --> RejectPO[Simpan Status REJECTED dan Catatan]
    AckDecision -- Minta Revisi --> RevisePO[Simpan Status REVISION_REQUESTED]
    AckDecision -- Terima --> AcceptPO[Simpan Status ACCEPTED dan Tanggal Janji Kirim]

    RejectPO --> SaveAck[INSERT atau UPDATE vendor_po_acknowledgments]
    RevisePO --> SaveAck
    AcceptPO --> SaveAck

    SaveAck --> CreateASNOpt{Apakah Vendor Mengirim ASN?}
    CreateASNOpt -- Tidak --> EndDone([Selesai])
    CreateASNOpt -- Ya --> FillASN[Isi Nomor Surat Jalan, Kurir, Tracking, Tanggal Kirim, dan Item]
    FillASN --> ValidateQty{Validasi Kuantitas dan PO Item?}
    ValidateQty -- Tidak Valid --> ShowError[Tampilkan Kesalahan Validasi]
    ShowError --> FillASN
    ValidateQty -- Valid --> InsertASN[INSERT vendor_asns dan vendor_asn_items]
    InsertASN --> NotifyInternal[ASN Tersedia untuk Proses GRN Internal]
    NotifyInternal --> EndDone
```

---

## 4. Sequence Diagram (Permintaan Data Portal Vendor dan Pelanggan)

```mermaid
sequenceDiagram
    autonumber
    actor Partner as Vendor / Customer Partner
    participant UI as Portal Web (Next.js 15)
    participant MW as Auth Middleware
    participant Hdl as Portal Handler
    participant Svc as Portal Service
    participant Repo as Portal Repository
    participant DB as PostgreSQL 16

    Partner->>UI: Login dan Buka /portal
    UI->>MW: GET /api/portal/profile (Bearer Token)
    MW->>MW: Validasi JWT dan company_id
    MW->>Hdl: Teruskan Request Terautentikasi
    Hdl->>Svc: GetProfile(ctx, userID)
    Svc->>Repo: FindPartnerByUser(userID)
    Repo->>DB: SELECT ... FROM portal_user_partners JOIN vendors / customers
    DB-->>Repo: Partner Mapping
    Repo-->>Svc: PortalProfile
    Svc-->>Hdl: Profil Mitra
    Hdl-->>UI: 200 OK (partner_type, partner_code, payment_terms_days)

    alt Partner Bertipe VENDOR
        UI->>Hdl: GET /api/portal/vendor/pos
        Hdl->>Svc: ListVendorPOs(ctx, companyID, vendorID)
        Svc->>Repo: Query PO Milik Vendor
        Repo->>DB: SELECT ... FROM purchase_orders WHERE vendor_id = $1
        DB-->>Repo: Daftar PO
        Repo-->>Svc: VendorPO[]
        Svc-->>Hdl: Daftar PO
        Hdl-->>UI: 200 OK

        Partner->>UI: Klik Acknowledge PO
        UI->>Hdl: POST /api/portal/vendor/pos/{id}/acknowledge
        Hdl->>Svc: AcknowledgeVendorPO(ctx, companyID, vendorID, poID, input)
        Svc->>Repo: Upsert vendor_po_acknowledgments
        Repo->>DB: INSERT ... ON CONFLICT (purchase_order_id) DO UPDATE
        DB-->>Repo: Acknowledgment Tersimpan
        Repo-->>Svc: Status Terbaru
        Svc-->>Hdl: Hasil Akseptasi
        Hdl-->>UI: 200 OK
    else Partner Bertipe CUSTOMER
        UI->>Hdl: GET /api/portal/customer/orders
        Hdl->>Svc: ListCustomerOrders(ctx, companyID, customerID)
        Svc->>Repo: Query Sales Order Milik Pelanggan
        Repo->>DB: SELECT ... FROM sales_orders WHERE customer_id = $1
        DB-->>Repo: Daftar Sales Order
        Repo-->>Svc: CustomerOrder[]
        Svc-->>Hdl: Daftar Sales Order
        Hdl-->>UI: 200 OK

        UI->>Hdl: GET /api/portal/customer/invoices
        Hdl->>Svc: ListCustomerInvoices(ctx, companyID, customerID)
        Svc->>Repo: Query Invoice Milik Pelanggan
        Repo->>DB: SELECT ... FROM sales_invoices WHERE customer_id = $1
        DB-->>Repo: Daftar Invoice
        Repo-->>Svc: CustomerInvoice[]
        Svc-->>Hdl: Daftar Invoice
        Hdl-->>UI: 200 OK
    end

    UI-->>Partner: Tampilkan Data Sesuai Tipe Mitra
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o{ PORTAL_USER_PARTNERS : "owns"
    USERS ||--o{ PORTAL_USER_PARTNERS : "maps"
    VENDORS ||--o{ PORTAL_USER_PARTNERS : "vendor_scope"
    CUSTOMERS ||--o{ PORTAL_USER_PARTNERS : "customer_scope"

    VENDORS ||--o{ VENDOR_PO_ACKNOWLEDGMENTS : "acknowledges"
    PURCHASE_ORDERS ||--o| VENDOR_PO_ACKNOWLEDGMENTS : "has_one"

    VENDORS ||--o{ VENDOR_ASNS : "ships"
    PURCHASE_ORDERS ||--o{ VENDOR_ASNS : "references"
    VENDOR_ASNS ||--o{ VENDOR_ASN_ITEMS : "contains"
    PURCHASE_ORDER_ITEMS ||--o{ VENDOR_ASN_ITEMS : "fulfills"
    ITEMS ||--o{ VENDOR_ASN_ITEMS : "identifies"

    CUSTOMERS ||--o{ SALES_ORDERS : "places"
    CUSTOMERS ||--o{ SALES_INVOICES : "billed"

    PORTAL_USER_PARTNERS {
        uuid id PK
        uuid company_id FK
        uuid user_id FK
        string partner_type "VENDOR | CUSTOMER"
        uuid vendor_id FK
        uuid customer_id FK
        boolean is_primary_contact
        timestamp created_at
    }

    VENDOR_PO_ACKNOWLEDGMENTS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        uuid purchase_order_id FK
        uuid vendor_id FK
        string status "ACCEPTED | REJECTED | REVISION_REQUESTED"
        date promised_delivery_date
        text notes
        uuid acknowledged_by FK
        timestamp created_at
    }

    VENDOR_ASNS {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string asn_number UK
        uuid purchase_order_id FK
        uuid vendor_id FK
        string vendor_delivery_note_number
        string carrier_name
        string tracking_number
        date ship_date
        date estimated_arrival_date
        string status "IN_TRANSIT | ARRIVED_AT_GATE | INSPECTED | RECEIVED | CANCELLED"
        text notes
        uuid created_by FK
        timestamp created_at
        timestamp updated_at
    }

    VENDOR_ASN_ITEMS {
        uuid id PK
        uuid asn_id FK
        uuid po_item_id FK
        uuid item_id FK
        numeric shipped_quantity
        string lot_number
        date expiry_date
        int package_count
        text notes
    }
```

Catatan integritas dan keamanan:
- `portal_user_partners` memiliki constraint unik `(user_id, partner_type)` dan check constraint yang memastikan tepat satu dari `vendor_id` atau `customer_id` terisi.
- Setiap endpoint portal memerlukan autentikasi dan memvalidasi kepemilikan data berdasarkan pemetaan mitra, `company_id`, dan `branch_id`.
- Vendor tidak dapat mengakses PO vendor lain, dan pelanggan tidak dapat mengakses pesanan atau tagihan pelanggan lain.
- e-Faktur dan dokumen pajak pelanggan belum diekspos melalui portal.
