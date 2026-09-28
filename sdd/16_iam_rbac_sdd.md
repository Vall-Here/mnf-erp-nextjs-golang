# 16. Software Design Document: IAM & Granular RBAC Matrix

## 1. Ringkasan Modul & Tanggung Jawab
Modul **Identity & Access Management (IAM) & Granular RBAC Matrix** bertanggung jawab atas keamanan akses sistem: Otentikasi Berbasis JWT (*JSON Web Tokens*), Hashing Kata Sandi (*bcrypt*), Matriks Izin Granular Modul-Aksi (`module:action`), Penugasan Cabang Pengguna (*User Branch Governance*), dan Pembatasan Laju Kueri (*Rate Limiting*).

- **Lokasi Backend**: `services/api/internal/modules/iam/`
- **Lokasi Frontend**: `apps/web/app/(auth)/login/` & `apps/web/app/(dashboard)/settings/iam/`

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    User["Pengguna Sistem"]
    SecAdmin["Security Administrator"]
    Auditor["Auditor Keamanan"]

    subgraph IAMModule ["Modul IAM & Granular RBAC"]
        UC1["Login & Otentikasi Dua Faktor (JWT Access + Refresh Token)"]
        UC2["Kelola Akun Pengguna & Reset Kata Sandi"]
        UC3["Konfigurasi Peran (Roles) & Hierarki Jabatan"]
        UC4["Pemetaan Matriks Izin Granular (module:action)"]
        UC5["Penugasan Pengguna ke Cabang Tertentu (user_branches)"]
        UC6["Pemblokiran Akun Otomatis (Brute-Force Lockout)"]
        UC7["Audit Jejak Sesi & Aktivitas Login"]
    end

    User --> UC1

    SecAdmin --> UC2
    SecAdmin --> UC3
    SecAdmin --> UC4
    SecAdmin --> UC5
    SecAdmin --> UC6

    Auditor --> UC4
    Auditor --> UC7
```

---

## 3. Activity Diagram (Otentikasi JWT & Verifikasi Otorisasi Granular)

```mermaid
flowchart TD
    Start([Klien Mengirim Permintaan API]) --> HasToken{Ada Header Bearer Token?}
    
    HasToken -- Tidak --> Return401[Tolak: 401 Unauthorized]
    Return401 --> EndFail([Selesai Gagal])
    
    HasToken -- Ya --> ValidateJWT[Verifikasi Tanda Tangan Kriptografi JWT & Expiry]
    ValidateJWT --> IsTokenValid{Token Valid & Belum Kadaluarsa?}
    
    IsTokenValid -- Invalid/Expired --> Return401
    IsTokenValid -- Valid --> ExtractClaims[Ekstraksi User ID, Role, & Allowed Branches]
    
    ExtractClaims --> CheckBranch{Apakah Request Memiliki Izin di activeBranchId?}
    CheckBranch -- Tidak Diizinkan --> Return403Branch[Tolak: 403 Forbidden - Branch Scoped Access Denied]
    Return403Branch --> EndFail
    
    CheckBranch -- Diizinkan --> CheckPermission{Apakah User/Role Memiliki Izin module:action?}
    CheckPermission -- Tidak Memiliki Izin --> Return403Perm[Tolak: 403 Forbidden - Insufficient Module Permission]
    Return403Perm --> EndFail
    
    CheckPermission -- Izin Lengkap --> InjectContext[Sematkan Claims ke Request Context]
    InjectContext --> ForwardHandler[Teruskan ke Handler Modul Bisnis]
    ForwardHandler --> EndSuccess([Eksekusi Berhasil])
```

---

## 4. Sequence Diagram (Alur Login & Pembuatan Token Sesi)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Pengguna Web
    participant UI as Login Page (Next.js 15)
    participant Rate as Rate Limiter Middleware
    participant Hdl as IAM Auth Handler
    participant Svc as IAM Auth Service
    participant Repo as IAM Repository
    participant DB as PostgreSQL 16

    Client->>UI: Input Email / Username & Password
    UI->>Rate: POST /api/v1/auth/login
    Rate->>Rate: Validasi Batas Upaya Login IP / Menit
    Rate->>Hdl: HandleLogin(w, r)
    Hdl->>Svc: Authenticate(ctx, email, password)
    
    rect rgb(240, 248, 255)
        Svc->>Repo: FindUserByEmail(email)
        Repo->>DB: SELECT * FROM users WHERE email = ...
        DB-->>Repo: User Record with Password Hash
        
        Svc->>Svc: Verifikasi bcrypt.CompareHashAndPassword()
        
        Svc->>Repo: GetUserRolesAndPermissions(userID)
        Repo->>DB: SELECT p.code FROM permissions p JOIN ...
        DB-->>Repo: List Permissions Array ["sales:*", "accounting:view"]
        
        Svc->>Repo: GetUserBranches(userID)
        Repo->>DB: SELECT branch_id FROM user_branches WHERE user_id = ...
        DB-->>Repo: List Allowed Branch IDs
        
        Svc->>Svc: Generate Access Token (JWT 15m) & Refresh Token (7d)
        Svc->>Repo: Record Auth Session & Last Login Timestamp
        Repo->>DB: INSERT INTO auth_sessions (user_id, token_hash, expires_at)
    end

    Svc-->>Hdl: Auth Result (Access Token, User Profile, Permissions, Branches)
    Hdl-->>UI: 200 OK (JWT Token Envelope)
    UI-->>Client: Simpan Token di Secure Storage & Arahkan ke Dashboard
```

---

## 5. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o{ USERS : "employs"
    USERS ||--o{ USER_ROLES : "assigned"
    ROLES ||--o{ USER_ROLES : "granted_to"
    ROLES ||--o{ ROLE_PERMISSIONS : "contains"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "mapped_in"
    USERS ||--o{ USER_BRANCHES : "authorized_in"
    BRANCHES ||--o{ USER_BRANCHES : "accessible_by"
    USERS ||--o{ AUTH_SESSIONS : "holds"

    USERS {
        uuid id PK
        uuid company_id FK
        string email UK
        string username UK
        string full_name
        string password_hash
        string phone
        string status "ACTIVE | SUSPENDED | LOCKED"
        integer failed_login_attempts
        timestamp last_login_at
        timestamp created_at
    }

    ROLES {
        uuid id PK
        uuid company_id FK
        string role_code UK
        string role_name
        string description
        boolean is_system_role
    }

    PERMISSIONS {
        uuid id PK
        string module "accounting | sales | purchasing | inventory | mfg | quality | hrm | system"
        string action "view | create | edit | delete | approve | export | *"
        string code UK "e.g. sales:create, accounting:approve"
        string description
    }

    ROLE_PERMISSIONS {
        uuid id PK
        uuid role_id FK
        uuid permission_id FK
    }

    USER_ROLES {
        uuid id PK
        uuid user_id FK
        uuid role_id FK
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
        string user_agent
        string client_ip
        timestamp expires_at
        boolean is_revoked
    }
```
