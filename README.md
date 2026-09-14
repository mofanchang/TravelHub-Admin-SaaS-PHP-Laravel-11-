# TravelHub-Admin-SaaS-PHP-Laravel-11

# TravelHub Admin SaaS (PHP / Laravel 11 版本)

[![PHP](https://img.shields.io/badge/PHP-v8.3%2B-777BB4.svg)](https://www.php.net/)
[![Laravel](https://img.shields.io/badge/Laravel-v11-FF2D20.svg)](https://laravel.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-v15%2B-336791.svg)](https://www.postgresql.org/)
[![JWT](https://img.shields.io/badge/JWT-Firebase%20PHP--JWT-000000.svg)](https://github.com/firebase/php-jwt)
[![PHPUnit](https://img.shields.io/badge/PHPUnit-v12-3776AB.svg)](https://phpunit.de/)

**TravelHub SaaS (Laravel 11 版本)** 是一個專為旅遊產業設計的**企業級多租戶（Multi-Tenant）B2B 後台管理系統**。本專案採用 **PHP 8.3 / Laravel 11** 開發，實作高標準的 **OWASP 資安架構**、**Service Layer 業務邏輯分離** 與 **跨框架 (Django & Laravel) 資料庫共用機制**，展示健全的企業級 PHP 系統架構與後端開發能力。

---


### 1. 多租戶 (Multi-Tenant) 行級資料隔離與 BOLA / IDOR 防禦
- **Row-Level Tenant Isolation**：所有組織成員、角色權限、行程（Trips）、訂單（Bookings）與操作日誌均於 PostgreSQL 中與 `organization_id` 強制關聯。
- **後端 Middleware 與 Eloquent Scope 隔離**：由 `RequireAuth` 中間件從 Bearer Token 中萃取 `organization_id` 等租戶內容並附加到 request 上，所有 Controller 查詢都會加上這個條件，杜絕 OWASP Top 10 BOLA (Broken Object Level Authorization) / IDOR 越權存取漏洞。
- 這項隔離是實際踩過坑才補上的：早期版本的 `Trip` model 沒有把 `organization_id` 放進 `$fillable`，導致新建立的行程建立時會被 Eloquent 的 Mass Assignment 保護悄悄丟棄該欄位，隔離形同虛設，之後才修正。

### 2. Firebase JWT 雙 Token 認證與即時存取撤銷 (XSS & CSRF 防護)
- **記憶體端 Access Token (Anti-XSS)**：短效（15 分鐘）Access Token 僅於前端記憶體中保存，防止腳本竊取。
- **HttpOnly Cookie Refresh Token (Anti-CSRF)**：長效 Refresh Token 採用 `HttpOnly` + `SameSite=Lax` Cookie 傳遞，資料庫只存放 `SHA-256` 雜湊後的值，原始 Token 只在核發當下出現一次。
- **登出時明確撤銷**：呼叫 `/api/auth/logout` 會找到對應的 Refresh Token 並標記撤銷（`revoked_at`），之後就算 Cookie 還在，也無法再用它換發新的 Access Token。
- **成員被移除時的即時存取撤銷**：這不是靠額外呼叫「撤銷」函式做到的，而是 `RequireAuth` Middleware 在**每一次請求**都會重新查詢 `Membership::find()` 並檢查 `isActive()`——一旦後台管理員把某個成員刪除或停用，即使他手上的 Access Token 還沒過期，下一次打 API 就會直接被擋下（403），達到近乎即時的存取撤銷效果。
- **端點層級限流**：登入與 Refresh 端點都加上 `throttle:5,1`（每分鐘最多 5 次），避免密碼被暴力枚舉。

### 3. 動態 RBAC (Role-Based Access Control) 權限矩陣
- **細粒度權限模型**：實作 `User`, `Organization`, `Membership`, `Role`, `Permission` 五表關聯架構。支援 `Owner`、`Manager`、`Viewer` 角色與細粒度權限聚合（如 `users:read`, `users:create`, `users:delete`, `audit:read`, `trips:write`, `bookings:write`）。
- **中間件級別權限攔截**：透過 `RequirePermission` 中間件傳遞權限參數（如 `permission:users:delete`），在 Controller 執行前精確校驗。

### 4. Service Layer 服務層架構與高內聚解耦
- **分層架構設計 (Clean Architecture)**：將認證簽發、Token 刷新、雜湊檢驗與 Token 撤銷邏輯抽離至獨立的 `AuthService`，保持 Controller 的精簡與職責單一 (SRP 原則)。
- **跨框架資料表唯讀對應 (Django Interoperability)**：設計 `DjangoUser` 模型，唯讀對應 Django C 端平台（`accounts_customuser`）的客戶帳號資料表，用於在訂單列表中正確顯示「這筆訂單是哪位客戶下的」。這個 model 刻意設成 `guarded = ['*']`（完全不可寫入）——Laravel 只負責讀取顯示，不處理、也不驗證 Django 那邊的密碼雜湊，兩邊的登入系統完全獨立，互不相通。

### 5. 系統異動審計日誌 (Audit Logging)
- **合規性軌跡紀錄**：整合 `AuditLogController` 與 `AuditLog` 模型，針對關鍵業務（成員增刪、行程異動、訂單狀態更新）紀錄 `organization_id`、`actor_membership_id`、`action`、`target_type` 以及異動內容的 metadata。

---

## 系統架構圖 (Architecture)

```text
┌─────────────────────────────────────────────────────────────┐
│               Laravel 11 Admin Web Server (Port 8000)       │
│  ─────────────────────────────────────────────────────────  │
│  · Controllers: Auth, Membership, Trip, Booking, AuditLog   │
│  · Services: AuthService (JWT & SHA-256 Refresh Token)      │
│  · Middlewares: RequireAuth, RequirePermission (RBAC)       │
│  · Models: User, DjangoUser, Organization, Role, Permission │
└──────────────────────────────┬──────────────────────────────┘
                               │ Eloquent ORM / Query Builder
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL (traveltrip_db)                │
│  ─────────────────────────────────────────────────────────  │
│  · Multi-Tenant Row Isolation (organization_id)              │
│  · SHA-256 Refresh Tokens & Revocation Storage               │
│  · Shared Trips & Bookings Schema with Django C-End          │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心資料庫模型 (Domain Models)

| 模型名稱 | 說明 | 關鍵職責 |
|---------|------|---------|
| `User` | 後台系統使用者 | 處理登入、密碼雜湊與組織綁定 |
| `DjangoUser` | Django 舊系統使用者（唯讀） | 顯示訂單所屬的 C 端客戶資訊，不處理登入 |
| `Organization` | 多租戶商戶組織 | 定義租戶隔離邊界 (Tenant Domain) |
| `Membership` | 租戶成員關聯表 | 連結 User, Organization 與 Role |
| `Role` | 系統角色 | 定義 Owner, Manager, Viewer 等角色 |
| `Permission` | 系統權限 | 定義如 `users:delete`, `trips:write` 細粒度權限 |
| `RefreshToken` | Refresh Token 管理表 | 存放 SHA-256 雜湊後的 Token 與過期時間 |
| `Trip` | 旅遊行程 | 產品資料維護與狀態控管 |
| `Booking` | 訂單與預約 | 顧客預約紀錄與審核流程 |
| `AuditLog` | 審計日誌 | 紀錄系統操作軌跡 |

---

## API 端點列表 (API Reference)

### 認證模組 (Authentication)
- `POST /api/auth/login` — 使用者登入，回傳 Access Token 並寫入 HttpOnly Cookie
- `POST /api/auth/refresh` — 檢驗 Cookie Refresh Token，發放新 Access Token
- `POST /api/auth/logout` — 撤銷 Refresh Token 並清除 Cookie

### 個人與成員管理 (Me & Memberships)
- `GET    /api/v1/me` — 取得目前登入者資訊、角色與細粒度權限矩陣
- `GET    /api/v1/memberships` — 取得當前租戶下的成員名單（受租戶隔離，需 `users:read`）
- `POST   /api/v1/memberships` — 新增成員（需 `users:write`）
- `DELETE /api/v1/memberships/{id}` — 移除成員（需 `users:delete`）

### 業務管理 (Trips & Bookings)
- `GET    /api/v1/trips` — 行程列表查詢（需 `trips:read`）
- `POST   /api/v1/trips` — 新增旅遊行程（需 `trips:write`）
- `DELETE /api/v1/trips/{id}` — 刪除行程（需 `trips:delete`）
- `GET    /api/v1/bookings` — 訂單列表查詢（需 `bookings:read`）
- `PATCH  /api/v1/bookings/{id}` — 變更訂單狀態，轉為 Cancelled/Expired 時會自動歸還座位（需 `bookings:write`）

### 審計日誌 (Audit Logs)
- `GET /api/v1/audit-logs` — 檢視租戶異動紀錄（需 `audit:read`）

---

## 技術棧 (Tech Stack)

- **核心框架**：Laravel 11.x
- **執行環境**：PHP 8.3+
- **認證安全**：Firebase PHP-JWT (`firebase/php-jwt 7.1`), SHA-256 Hashing, HttpOnly Cookie
- **資料庫**：PostgreSQL 15+（與 Django 平台共用同一個 `traveltrip_db`）
- **程式碼規範與測試**：Laravel Pint, PHPUnit 12.x, Mockery
- **前端工具**：Vite + Tailwind CSS / Blade Templates

---

## 快速開始（獨立啟動）

### 1. 安裝 Composer 與 NPM 依賴套件

```bash
git clone <your-repo-url>
cd travel-agency-admin-saas-php
composer install
npm install
```

### 2. 設定 `.env` 環境變數

```bash
cp .env.example .env
php artisan key:generate
```

在 `.env` 中設定資料庫連線資訊：

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=traveltrip_db
DB_USERNAME=traveltrip_user
DB_PASSWORD=your_password

JWT_SECRET=請自行產生一組獨立、隨機的密鑰，不要沿用 Django 或 Node.js 版本用過的密鑰
```


### 3. 執行資料庫遷移與資料填充

```bash
php artisan migrate
php artisan db:seed
```

### 4. 啟動本機開發伺服器與前端建構

```bash
# 啟動 Laravel 後端伺服器 (Port 8000)
php artisan serve --port=8000

# 建構靜態資源
npm run build
```

訪問後台網址：`http://127.0.0.1:8000`

---

## 執行單元測試 (Testing)

\

### 1. 建立一個獨立的測試用資料庫（不要跟正式用的 `traveltrip_db` 混在一起）

```bash
psql -U <your_postgres_user> -h 127.0.0.1 -d postgres -c "CREATE DATABASE traveltrip_db_test;"
```

### 2. 複製 `.env.testing.example` 並填入你本機的 Postgres 帳密

```bash
cp .env.testing.example .env.testing
```

### 3. 跑測試

```bash
php artisan test
```

---

## 與 C端 Django 平台的關聯

此 Laravel 後台與 Django C端平台（`TRAVELTRIP_PROJECT`）連接同一個 `traveltrip_db` 資料庫，後台新增修改的行程與訂單狀態會即時反映於 C端網站與 RAG 推薦系統中。兩邊的使用者系統完全獨立：這個後台的 `User`/`Membership` 是後台員工帳號，Django 那邊的 `accounts_customuser` 是消費者帳號，兩者不共用密碼或登入機制，只有 `Trip`、`Booking` 這兩張業務資料表是真正共用的。

