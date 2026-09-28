# 14. Software Design Document: Multi-Tier Approval Workflow Engine

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Multi-Tier Approval Workflow Engine** adalah mesin orkestrasi persetujuan terpusat untuk seluruh dokumen bisnis (PO, PR, SO, MWO, Jurnal, Voucher, Cuti): Matriks Otorisasi Dinamis Berdasarkan Ambang Batas Nilai (*Threshold-Based Routing*), Pendelegasian Wewenang (*Delegation*), Pencatatan Histori Pengesahan (*Audit History Trail*), dan Integrasi Tanda Tangan Digital pada Dokumen Cetak Standar Industri.

- **Lokasi Backend**: `services/api/internal/modules/workflow/`
- **Lokasi Frontend**: `apps/web/app/(dashboard)/approvals/` & `components/shared/print-document/workflow-adapter.ts`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    Requester["Staff Pembuat Dokumen"]
    SpvApprover["Supervisor (Level 1)"]
    MgrApprover["Manajer (Level 2)"]
    DirApprover["Direktur / BOD (Level 3-4)"]
    SysAdmin["Administrator Sistem"]

    subgraph WorkflowModule ["Modul Approval Workflow Engine"]
        UC1["Konfigurasi Matriks Aturan Approval per Modul"]
        UC2["Pengajuan Dokumen ke Jalur Persetujuan"]
        UC3["Review & Approval / Penolakan / Permintaan Revisi"]
        UC4["Eskalasi Otomatis Berdasarkan Ambang Batas Nilai"]
        UC5["Delegasi Wewenang Sementara (Cuti/Dinas)"]
        UC6["Pencatatan Audit Trail Histori Pengesahan"]
        UC7["Penyematan Digital Signature Hash pada Dokumen"]
    end

    SysAdmin --> UC1

    Requester --> UC2

    SpvApprover --> UC3
    SpvApprover --> UC5

    MgrApprover --> UC3
    MgrApprover --> UC4

    DirApprover --> UC3
    DirApprover --> UC4

    Requester --> UC6
    DirApprover --> UC7
```

---

## 3. Activity Diagram (Siklus Penilaian & Persetujuan Multi-Tier)

```mermaid
flowchart TD
    Start([Dokumen Bisnis Disubmit]) --> MatchRule[Cari Aturan Workflow Sesuai Tipe Dokumen & Nominal]
    MatchRule --> CalcTiers[Tentukan Jumlah Tingkat Otorisasi N Level]
    CalcTiers --> InitRequest[Inisialisasi Approval Request: Step 1 / N]
    
    InitRequest --> NotifyApprover[Kirim Notifikasi ke Approver Tingkat Saat Ini]
    NotifyApprover --> CheckDelegation{Ada Delegasi Aktif?}
    
    CheckDelegation -- Ya --> RouteDelegate[Alihkan ke Pejabat Pengganti yang Didelegasikan]
    CheckDelegation -- Tidak --> WaitAction[Tunggu Aksi Otorisator]
    
    RouteDelegate --> WaitAction
    WaitAction --> ApproverAction{Keputusan Approver}
    
    ApproverAction -- Minta Revisi --> MarkRevision[Status REVISION_REQUESTED]
    MarkRevision --> NotifyRequester[Kembalikan ke Pemohon untuk Diperbaiki]
    NotifyRequester --> EndFail([Revisi])
    
    ApproverAction -- Tolak (Reject) --> MarkRejected[Status REJECTED & Catat Alasan]
    MarkRejected --> LockDoc[Kunci Dokumen Bisnis Status Ditolak]
    LockDoc --> EndFail
    
    ApproverAction -- Setujui (Approve) --> RecordStep[Catat Histori Approval, Tanggal & Digital Hash]
    RecordStep --> IsFinalStep{Apakah Step Saat Ini == N (Final)?}
    
    IsFinalStep -- Belum --> NextStep[Naikkan Step Order: Current Step + 1]
    NextStep --> NotifyApprover
    
    IsFinalStep -- Ya (Final) --> MarkApproved[Status APPROVAL: APPROVED]
    MarkApproved --> UnlockDoc[Ubah Status Dokumen Bisnis: APPROVED / POSTED]
    UnlockDoc --> StampSignatures[Sematkan Bukti Pengesahan ke Dokumen Cetak]
    StampSignatures --> EndSuccess([Otorisasi Selesai Penuh])
```

---

## 4. Sequence Diagram (Eksekusi Approval Step oleh Pejabat Berwenang)

```mermaid
sequenceDiagram
    autonumber
    actor Approver as Pejabat Otorisasi
    participant UI as Approval Portal (Next.js 15)
    participant Hdl as Workflow Handler
    participant Svc as Workflow Service
    participant Mod as Calling Module (e.g. Purchasing PO)
    participant Repo as Workflow Repository
    participant DB as PostgreSQL 16

    Approver->>UI: Klik "Setujui (Approve)" dengan Catatan Otorisasi
    UI->>Hdl: POST /api/v1/workflow/requests/{id}/approve
    Hdl->>Svc: ApproveStep(ctx, requestID, approverUserID, comments)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: Begin Tx
        Svc->>Repo: GetApprovalRequest(requestID) FOR UPDATE
        Repo->>DB: SELECT * FROM approval_requests WHERE id = ...
        
        Svc->>Svc: Validasi Hak Akses User / Role Approver
        
        Svc->>Repo: Insert Approval History Record
        Repo->>DB: INSERT INTO approval_history (request_id, step_order, actor_id, action, comments, hash)
        
        alt Step Belum Final (Masih ada step berikutnya)
            Svc->>Repo: Increment Current Step Order
            Repo->>DB: UPDATE approval_requests SET current_step_order = current_step_order + 1
        else Step Final Selesai
            Svc->>Repo: Mark Request Status APPROVED
            Repo->>DB: UPDATE approval_requests SET status = 'APPROVED'
            
            Svc->>Mod: NotifyDocumentApproved(documentType, documentID)
            Mod->>DB: UPDATE purchase_orders SET status = 'CONFIRMED' WHERE id = ...
        end
        Svc->>Repo: Commit Tx
    end

    Svc-->>Hdl: Approval Step Result
    Hdl-->>UI: 200 OK (New Workflow State)
    UI-->>Approver: Tampilkan Status Otorisasi Sah & Dokumen Terperbarui
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    WORKFLOW_RULES ||--|{ WORKFLOW_STEPS : "defines"
    WORKFLOW_RULES ||--o{ APPROVAL_REQUESTS : "governs"
    APPROVAL_REQUESTS ||--|{ APPROVAL_HISTORY : "tracks"
    APPROVAL_REQUESTS ||--o{ APPROVAL_DELEGATIONS : "checks"

    WORKFLOW_RULES {
        uuid id PK
        uuid company_id FK
        string document_type UK "PURCHASE_ORDER | PURCHASE_REQ | SALES_ORDER | WORK_ORDER | PAYMENT_VOUCHER"
        string rule_name
        numeric min_amount
        numeric max_amount
        integer total_steps
        boolean is_active
    }

    WORKFLOW_STEPS {
        uuid id PK
        uuid workflow_rule_id FK
        integer step_order
        string step_name
        string approver_role
        uuid specific_user_id FK
        boolean can_delegate
    }

    APPROVAL_REQUESTS {
        uuid id PK
        uuid workflow_rule_id FK
        uuid branch_id FK
        string document_type
        string document_number
        uuid document_id
        numeric amount
        integer current_step_order
        integer total_steps
        string status "PENDING | APPROVED | REJECTED | REVISION_REQUESTED | CANCELLED"
        uuid requester_id FK
        timestamp submitted_at
    }

    APPROVAL_HISTORY {
        uuid id PK
        uuid request_id FK
        integer step_order
        string step_name
        uuid actor_id FK
        string actor_name
        string actor_role
        string action "SUBMITTED | APPROVED | REJECTED | REVISION_REQUESTED | DELEGATED"
        string comments
        string digital_signature_hash
        timestamp acted_at
    }

    APPROVAL_DELEGATIONS {
        uuid id PK
        uuid delegator_id FK
        uuid delegatee_id FK
        date start_date
        date end_date
        string reason
        boolean is_active
    }
```
