# 16. Software Design Document: IAM, Granular RBAC & Segregation of Duties (SoD)

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Identity & Access Management (IAM), Granular RBAC & Segregation of Duties (SoD)** bertanggung jawab atas integritas keamanan akses sistem PT SAWIT JAYA (SWT): Otentikasi Berbasis JWT (*JSON Web Tokens*), Hashing Kata Sandi (*bcrypt*), Penegakan Matriks Izin Granular 21 Modul × 7 Aksi (`module:action`), Isolasi Akses Fasilitas Cabang (*Horizontal Branch Isolation / Anti-IDOR*), Penegakan Pemisahan Tugas (*Segregation of Duties / Maker-Checker Rules*), dan Pembatasan Laju Kueri Terdistribusi (*Distributed Rate Limiting via Valkey*).

- **Lokasi Backend**: `services/api/internal/modules/iam/` & `services/api/internal/platform/middleware/`
- **Lokasi Frontend**: `apps/web/app/(auth)/login/` & `apps/web/app/(dashboard)/settings/iam/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    User["Pengguna Operasional"]
    SecAdmin["Security Administrator"]
    Auditor["Internal / Forensic Auditor"]

    subgraph IAMModule ["Modul IAM, RBAC & SoD Matrix"]
        UC1["Login Aman & Pembatasan Upaya Gagal (Rate Limiting)"]
        UC2["Manajemen Akun Pengguna & Composite Company-Role Integrity"]
        UC3["Konfigurasi Peran Jabatan (Roles) & Hak Akses Granular"]
        UC4["Penegakan Pemisahan Tugas Keras (Segregation of Duties - SoD)"]
        UC5["Penugasan Cabang Operasional Terkunci (user_branches)"]
        UC6["Pencegahan Eskalasi Hak Akses Horizontal (Anti-IDOR)"]
        UC7["Audit Jejak Akses Super Admin & Bypass Logging"]
    end

    User --> UC1

    SecAdmin --> UC2
    SecAdmin --> UC3
    SecAdmin --> UC4
    SecAdmin --> UC5

    Auditor --> UC4
    Auditor --> UC6
    Auditor --> UC7
```

---

## 3. Activity Diagram (Otentikasi JWT, Anti-IDOR Branch Guard, & Verifikasi SoD)

```mermaid
flowchart TD
    Start([Klien Mengirim Permintaan API]) --> CheckRateLimit{Laju Request Melebihi Batas IP?}
    
    CheckRateLimit -- Ya --> Return429[Tolak: 429 Too Many Requests]
    Return429 --> EndFail([Selesai Gagal])
    
    CheckRateLimit -- Tidak --> HasToken{Ada Header Bearer Token?}
    HasToken -- Tidak --> Return401[Tolak: 401 Unauthorized]
    Return401 --> EndFail
    
    HasToken -- Ya --> ValidateJWT[Verifikasi Kriptografi Token & Expiry]
    ValidateJWT --> IsTokenValid{Token Valid?}
    
    IsTokenValid -- Tidak --> Return401
    IsTokenValid -- Ya --> ExtractClaims[Ekstraksi User ID, Role, Permissions, & Allowed Branches]
    
    ExtractClaims --> CheckBranchScope{Apakah Request Meminta Cabang di Luar Hak Akses?}
    CheckBranchScope -- Ya / Ilegal --> Return403Branch[Tolak: 403 Forbidden - Branch Access Denied]
    Return403Branch --> EndFail
    
    CheckBranchScope -- Sesuai Wewenang --> CheckPermission{Memiliki Izin Spesifik module:action?}
    CheckPermission -- Tidak Ada Izin --> Return403Perm[Tolak: 403 Forbidden - Insufficient Permission]
    Return403Perm --> EndFail
    
    CheckPermission -- Izin Sah --> CheckSoD{Apakah Melanggar Aturan SoD Pembuat == Penyetuju?}
    CheckSoD -- Ya: Maker == Approver --> Return403SoD[Tolak: 403 Forbidden - Segregation of Duties Violation]
    Return403SoD --> EndFail
    
    CheckSoD -- Tidak: Lolos SoD --> InjectContext[Sematkan Authorized Branch & Claims ke Request Context]
    InjectContext --> ForwardHandler[Teruskan ke Handler Modul Bisnis]
    ForwardHandler --> EndSuccess([Eksekusi Berhasil])
```

---

## 4. Sequence Diagram (Alur Login & Pembuatan Token Sesi Terisolasi)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Pengguna Web
    participant UI as Login Page (Next.js 15)
    participant Rate as Valkey Distributed Rate Limiter
    participant Hdl as IAM Auth Handler
    participant Svc as IAM Auth Service
    participant Repo as IAM Repository
    participant DB as PostgreSQL 16

    Client->>UI: Input Email & Password
    UI->>Rate: POST /api/auth/login
    Rate->>Rate: Validasi Batas Upaya Login IP
    Rate->>Hdl: HandleLogin(w, r)
    Hdl->>Svc: Authenticate(ctx, email, password)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: FindUserByEmail(email)
        Repo->>DB: SELECT * FROM users u JOIN roles r ON r.id = u.role_id WHERE u.email = $1
        DB-->>Repo: User Record with Password Hash & Role
        
        Svc->>Svc: Verifikasi bcrypt.CompareHashAndPassword()
        
        Svc->>Repo: GetRolePermissions(roleID)
        Repo->>DB: SELECT p.code FROM permissions p JOIN role_permissions rp ON rp.permission_id = p.id WHERE rp.role_id = $1
        DB-->>Repo: Matriks Permissions Array (e.g. ["accounting:approve", "inventory:view"])
        
        Svc->>Repo: GetUserBranches(userID)
        Repo->>DB: SELECT branch_id FROM user_branches WHERE user_id = $1
        DB-->>Repo: Daftar ID Cabang yang Diizinkan (Allowed Branches)
        
        Svc->>Svc: Generate JWT Claims (UserID, CompanyID, Role, Permissions, AllowedBranches)
        Svc->>Repo: Simpan Sesi Login
        Repo->>DB: INSERT INTO auth_sessions (user_id, refresh_token_hash, client_ip, expires_at)
    end

    Svc-->>Hdl: Auth Result (Access Token, Profile, Permissions, AllowedBranches)
    Hdl-->>UI: 200 OK (JWT Token Envelope)
    UI-->>Client: Simpan Token di Secure Storage & Redirect ke Dashboard
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o{ ROLES : "defines"
    COMPANIES ||--o{ USERS : "employs"
    ROLES ||--o{ USERS : "assigned_to"
    ROLES ||--o{ ROLE_PERMISSIONS : "contains"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "mapped_in"
    USERS ||--o{ USER_BRANCHES : "authorized_in"
    BRANCHES ||--o{ USER_BRANCHES : "accessible_by"
    USERS ||--o{ AUTH_SESSIONS : "holds"

    COMPANIES {
        uuid id PK
        string code UK "SWT"
        string name "PT SAWIT JAYA"
        boolean is_active
    }

    ROLES {
        uuid id PK
        uuid company_id FK
        string code UK
        string name
        string description
        boolean is_system_role
    }

    USERS {
        uuid id PK
        uuid company_id FK
        uuid role_id FK "Composite FK: (role_id, company_id) REFERENCES roles"
        string email UK
        string full_name
        string password_hash
        string status "ACTIVE | SUSPENDED"
        timestamp last_login_at
        timestamp created_at
    }

    PERMISSIONS {
        uuid id PK
        string module "21 Modules (accounting, inventory, sales, purchasing, mfg, etc.)"
        string action "view | create | edit | delete | approve | export | admin"
        string code UK "e.g. accounting:approve, inventory:create"
        string description
    }

    ROLE_PERMISSIONS {
        uuid role_id PK,FK
        uuid permission_id PK,FK
    }

    USER_BRANCHES {
        uuid id PK
        uuid user_id FK
        uuid branch_id FK
        boolean is_default
    }

    AUTH_SESSIONS {
        uuid id PK
        uuid user_id FK
        string refresh_token_hash UK
        string client_ip
        string user_agent
        timestamp expires_at
        boolean is_revoked
    }
```

---

## 6. Matriks Pemisahan Tugas (Segregation of Duties - SoD)

Untuk mencegah kecurangan dan konflik kepentingan (*fraud & ghost transactions*), sistem menegakkan aturan SoD keras (*HARD SoD Rules*):

| Modul | Tindakan Maker (Pembuat) | Tindakan Approver (Penyetuju) | Tingkat Konflik | Risiko yang Dicegah |
|---|---|---|---|---|
| **Accounting** | `accounting:create` (Buat Jurnal Memorial) | `accounting:approve` (Posting ke Buku Besar) | **HARD** | Pembuatan dan posting voucher jurnal fiktif tanpa pengawasan manajer keuangan. |
| **Inventory** | `inventory:create` (Input Opname Fisik) | `inventory:approve` (Approve Selisih Stok) | **HARD** | Staf gudang menutupi kehilangan fisik barang dengan menyetujui stock adjustment sendiri. |
| **Purchasing** | `purchasing:create` (Buat PO / Submit Bid) | `purchasing:approve` (Approve PO / Award RFQ) | **HARD** | Pembelian fiktif atau penetapan pemenang tender tanpa otorisasi pimpinan pengadaan. |
| **Purchasing / AP** | `accounting:create` / `accounting:edit` (Catat Faktur) | `purchasing:approve` / `accounting:approve` (Approve Tagihan) | **HARD** | Staf pencatat invoice vendor meloloskan pembayaran fiktif tanpa verifikasi 3-Way Match. |
| **Sales** | `sales:create` / `sales:edit` (Buat Sales Order) | `sales:approve` / `accounting:approve` (Release Credit Hold) | **HARD** | Staf sales lapangan melepaskan pesanan pelanggan macet yang melebihi batas plafon kredit. |
