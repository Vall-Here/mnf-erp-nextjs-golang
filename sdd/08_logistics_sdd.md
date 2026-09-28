# 08. Software Design Document: Logistics & Fleet Transport

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Logistics & Fleet Transport** mengelola armada transportasi dan distribusi fisik: Registrasi Armada Truk (*Fleet Vehicles*), Penerbitan Surat Perintah Jalan (*SPJ / Trip Orders*), Alokasi & Penyelesaian Uang Jalan Supir (*Driver Cash Allowance Settlement*), serta Validasi Bukti Pengiriman (*Proof of Delivery / POD*) dengan tanda tangan digital dan foto penerimaan.

- **Lokasi Backend**: `services/api/internal/modules/logistics/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/logistics/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Dispatcher["Dispatcher Logistik"]
    Driver["Supir / Driver Ekspedisi"]
    Cashier["Kasir Treasury"]
    LogMgr["Manajer Logistik"]

    subgraph LogisticsModule ["Modul Logistics & Fleet Transport"]
        UC1["Kelola Master Armada & Masa Berlaku KIR/STNK"]
        UC2["Penerbitan Surat Perintah Jalan (SPJ)"]
        UC3["Alokasi Uang Jalan (BBM, Tol, Uang Saku)"]
        UC4["Unggah Foto Bukti & Tanda Tangan Penerima (POD)"]
        UC5["Penyelesaian Biaya Riil Perjalanan (Expense Settlement)"]
        UC6["Integrasi Ekspedisi Pihak Ketiga (3PL Tracking)"]
        UC7["Otorisasi Penyelesaian Perjalanan & SPJ Status Closed"]
    end

    Dispatcher --> UC1
    Dispatcher --> UC2
    Dispatcher --> UC3
    Dispatcher --> UC6

    Driver --> UC4
    Driver --> UC5

    Cashier --> UC3
    Cashier --> UC5

    LogMgr --> UC7
```

---

## 3. Activity Diagram (Alur Pengiriman SPJ & Validasi POD)

```mermaid
flowchart TD
    Start([Jadwal Pengiriman DO]) --> AssignFleet[Pilih Armada & Driver Sesuai Kapasitas Kg/CBM]
    AssignFleet --> CalcAllowance[Hitung Estimasi Uang Jalan BBM + Tol + Saku]
    CalcAllowance --> IssueSPJ[Terbitkan Surat Perintah Jalan / SPJ]
    
    IssueSPJ --> DisburseCash[Pencairan Uang Saku oleh Kasir]
    DisburseCash --> DispatchTruck[Armada Berangkat / Status BERANGKAT]
    
    DispatchTruck --> ArriveCust[Tiba di Lokasi Pelanggan]
    ArriveCust --> Handover[Bongkar Muatan & Serah Terima Fisik]
    
    Handover --> CapturePOD[Ambil Foto Bukti & Tanda Tangan Digital Penerima]
    CapturePOD --> UploadPOD[Kirim Berkas POD ke Sistem]
    
    UploadPOD --> VerifyCondition{Barang Lengkap & Baik?}
    VerifyCondition -- Rusak Sebagian --> MarkPartial[Catat Catatan Kerusakan / Retur]
    VerifyCondition -- Lengkap & Baik --> MarkVerified[Status POD Terverifikasi]
    
    MarkPartial --> ReturnBase[Armada Kembali ke Pabrik]
    MarkVerified --> ReturnBase
    
    ReturnBase --> SubmitSettlement[Driver Serahkan Struk BBM/Tol & Sisa Uang Saku]
    SubmitSettlement --> ReconcileCash[Rekonsiliasi Kasbon Uang Jalan]
    ReconcileCash --> CloseSPJ[Tutup SPJ / Status SETTLED]
    CloseSPJ --> EndDone([Pengiriman Selesai])
```

---

## 4. Sequence Diagram (Unggah Proof of Delivery / POD)

```mermaid
sequenceDiagram
    autonumber
    actor Driver as Driver / Dispatcher
    participant UI as Logistics Web (Next.js 15)
    participant Hdl as Logistics Handler
    participant Svc as Logistics Service
    participant S3 as Storage Adapter (Floci S3)
    participant Repo as Logistics Repository
    participant DB as PostgreSQL 16

    Driver->>UI: Input Nama Penerima, Tanda Tangan & Foto Bukti
    UI->>Hdl: POST /api/v1/logistics/delivery-proofs
    Hdl->>Svc: RecordProofOfDelivery(ctx, input)
    
    rect rgb(240, 248, 255)
        Svc->>S3: Upload Photo Proof & Signature Base64
        S3-->>Svc: S3 File Keys & Direct URLs
        
        Svc->>Repo: Begin Tx
        Svc->>Repo: Insert Delivery Proof Record
        Repo->>DB: INSERT INTO delivery_proofs (recipient_name, signature_url, photo_proof_url, condition, status)
        
        Svc->>Repo: Update SPJ & DO Status to DELIVERED
        Repo->>DB: UPDATE trip_orders SET status = 'DELIVERED' WHERE id = ...
        Repo->>DB: UPDATE delivery_orders SET status = 'DELIVERED' WHERE id = ...
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: POD Entity
    Hdl-->>UI: 201 Created
    UI-->>Driver: Konfirmasi Sukses & Tampilkan Berkas POD Tervalidasi
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    FLEET_VEHICLES ||--o{ TRIP_ORDERS : "assigned_to"
    TRIP_ORDERS ||--|{ TRIP_DELIVERY_ORDERS : "contains"
    DELIVERY_ORDERS ||--o{ TRIP_DELIVERY_ORDERS : "shipped_via"
    TRIP_ORDERS ||--o| TRIP_EXPENSE_SETTLEMENTS : "settled_in"
    TRIP_DELIVERY_ORDERS ||--o| DELIVERY_PROOFS : "validated_by"

    FLEET_VEHICLES {
        uuid id PK
        uuid company_id FK
        uuid branch_id FK
        string plate_number UK
        string vehicle_type "ENGKEL_BOX | CDD_BOX | FUSO | TRONTON | CONTAINER"
        string brand_model
        numeric capacity_weight_kg
        numeric capacity_volume_cbm
        date kir_expiry_date
        date stnk_expiry_date
        string status "ACTIVE | IN_TRANSIT | MAINTENANCE"
    }

    TRIP_ORDERS {
        uuid id PK
        uuid fleet_vehicle_id FK
        uuid driver_id FK
        string spj_number UK
        date departure_date
        date return_date
        string destination_city
        numeric cash_allowance
        numeric actual_total_cost
        string status "DRAFT | DISPATCHED | DELIVERED | SETTLED"
    }

    TRIP_DELIVERY_ORDERS {
        uuid id PK
        uuid trip_order_id FK
        uuid delivery_order_id FK
        string delivery_order_ref
    }

    TRIP_EXPENSE_SETTLEMENTS {
        uuid id PK
        uuid trip_order_id FK
        numeric fuel_liters
        numeric fuel_cost
        numeric toll_cost
        numeric parking_cost
        numeric other_cost
        string fuel_receipt_url
        string toll_receipt_url
        string settlement_notes
        timestamp settled_at
    }

    DELIVERY_PROOFS {
        uuid id PK
        uuid trip_delivery_order_id FK
        string recipient_name
        string confirmation_status "VERIFIED | DISCREPANCY"
        string signature_url
        string photo_proof_url
        timestamp received_date
        string notes
    }
```
