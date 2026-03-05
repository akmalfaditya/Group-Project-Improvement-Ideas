# Future Development — Inventory Management System


---

## Daftar Isi

1. [Manajemen Aset & Inventaris](#1-manajemen-aset--inventaris)
2. [Workflow & Approval](#2-workflow--approval)
3. [Reporting & Analytics](#3-reporting--analytics)
4. [Notifikasi & Komunikasi](#4-notifikasi--komunikasi)
5. [User & Access Management](#5-user--access-management)
6. [Import, Export & Integrasi](#6-import-export--integrasi)
7. [Mobile & UX](#7-mobile--ux)
8. [Keamanan & Compliance](#8-keamanan--compliance)
9. [Infrastruktur & DevOps](#9-infrastruktur--devops)
10. [Gamification & Employee Engagement](#10-gamification--employee-engagement)
11. [AI & Smart Features](#11-ai--smart-features)
12. [Multi-Tenancy & Scalability](#12-multi-tenancy--scalability)
13. [Ringkasan Prioritas](#13-ringkasan-prioritas)

---

## 1. Manajemen Aset & Inventaris

### 1.1 Permanent Asset Tagging (Barcode/QR per Item)

**Status saat ini:** QR code di-generate per _request_ (per transaksi), bukan per item.

**Fitur baru:**
- Generate QR code / barcode permanen untuk setiap item fisik
- Print label dengan item code, nama, dan QR
- Scan barcode untuk langsung buka detail item
- Support barcode format: QR Code, Code 128, EAN-13

**Implementasi:**
```
Item → memiliki permanent QRCodeData (UUID/ItemCode)
Controller: GET /Admin/ItemView/PrintLabel/{id} → generate printable label PDF
Library: QRCoder (sudah ada) + iTextSharp atau SkiaSharp untuk PDF
```

---

### 1.2 Multi-Location / Multi-Warehouse Inventory

**Status saat ini:** Ada entity `City`, tapi hanya sebagai label — tidak ada tracking quantity per lokasi.

**Fitur baru:**
- Track stock quantity per lokasi/warehouse
- Transfer item antar lokasi dengan approval workflow
- Dashboard stock per lokasi
- Alert stock rendah per lokasi

**Entities baru:**
```
Warehouse (Id, Name, CityId, Address, IsActive)
WarehouseStock (WarehouseId, ItemId, Quantity, MinimumStock)
StockTransfer (Id, FromWarehouseId, ToWarehouseId, ItemId, Quantity, Status, RequestDate, ApproveDate)
```

---

### 1.3 Item Condition Tracking & Maintenance Log

**Status saat ini:** Saat return, admin pilih kondisi (WellReturned/Damaged/Lost/Consumed) tapi tidak ada follow-up.

**Fitur baru:**
- Maintenance request workflow: Damaged → Submitted for Repair → In Repair → Repaired → Available
- Track riwayat kondisi item sepanjang waktu
- Vendor management untuk repair external
- Biaya maintenance per item
- Schedule maintenance berkala (preventive maintenance)

**Entities baru:**
```
MaintenanceRequest (Id, ItemId, ReportedBy, Description, Priority, Status, VendorId, Cost, CreatedAt, CompletedAt)
MaintenanceStatus enum: Reported, InReview, InRepair, Repaired, WrittenOff
Vendor (Id, Name, Contact, Email, Specialization)
MaintenanceSchedule (Id, ItemId, IntervalDays, LastMaintenanceDate, NextDueDate)
```

---

### 1.4 Asset Depreciation & Valuation

**Status saat ini:** Tidak ada data finansial terkait item.

**Fitur baru:**
- Input purchase price, purchase date, expected lifespan
- Kalkulasi depresiasi otomatis (straight-line, declining balance)
- Current book value per item
- Total asset valuation report
- Write-off workflow untuk item yang sudah habis umurnya

**Penambahan di model `Item`:**
```csharp
public decimal PurchasePrice { get; set; }
public DateTime PurchaseDate { get; set; }
public int UsefulLifeMonths { get; set; }
public decimal SalvageValue { get; set; }
public DepreciationMethod DepreciationMethod { get; set; }
// Computed: CurrentBookValue, MonthlyDepreciation, AccumulatedDepreciation
```

---

### 1.5 Physical Inventory Count / Stocktake

**Status saat ini:** Tidak ada fitur untuk rekonsiliasi stok fisik vs stok sistem.

**Fitur baru:**
- Jadwalkan stocktake event (full / cycle count)
- Assign petugas per area/kategori
- Input count via mobile (scan item → input qty ditemukan)
- Hitung variance (selisih stok fisik vs sistem)
- Adjustment approval workflow
- Audit trail untuk setiap adjustment

**Entities baru:**
```
StocktakeEvent (Id, Name, ScheduledDate, Status, CreatedBy)
StocktakeAssignment (EventId, UserId, CategoryId, WarehouseId)
StocktakeCount (EventId, ItemId, SystemQty, PhysicalQty, Variance, CountedBy, CountedAt)
StocktakeAdjustment (EventId, ItemId, AdjustmentQty, Reason, ApprovedBy)
```

---

### 1.6 Item Variants & Serial Number Tracking

**Status saat ini:** Item hanya memiliki satu SKU. Untuk item yang perlu dilacak per unit (laptop, monitor), tidak ada serial number.

**Fitur baru:**
- Serial number tracking per individual unit
- Track lokasi dan assignee masing-masing unit
- Lifecycle per serial: Purchased → In Stock → Assigned → Returned → Retired
- Bulk register serial numbers saat item masuk

**Entity baru:**
```
ItemUnit (Id, ItemId, SerialNumber, PurchaseDate, WarrantyExpiry, Status, AssignedToUserId, LocationId)
ItemUnitStatus enum: InStock, Assigned, InRepair, Lost, Retired
```

---

### 1.7 Item Image Gallery

**Status saat ini:** Satu gambar per item (single `PicturePath`).

**Fitur baru:**
- Multiple images per item (gallery)
- Upload batch gambar
- Primary image selection
- Image compression & thumbnail generation
- Support drag-and-drop upload

**Entity baru:**
```
ItemImage (Id, ItemId, FileName, FilePath, IsPrimary, SortOrder, UploadedAt)
```

---

### 1.8 Item Bundling / Kit Management

**Fitur baru:**
- Buat bundle/kit dari beberapa item (contoh: "New Employee Kit" = laptop + mouse + headset + ID card)
- Request bundle sebagai satu transaksi
- Auto-check availability semua item dalam bundle
- Track bundle component secara terpisah

**Entity baru:**
```
ItemBundle (Id, BundleName, Description, IsActive)
BundleItem (BundleId, ItemId, Quantity)
```

---

### 1.9 Low Stock Automation & Auto-Restock

**Status saat ini:** Ada `QuantityLow` di ItemView untuk visual indicator, tapi tidak ada action otomatis. Restock module hanya read-only.

**Fitur baru:**
- Auto-create restock request ketika stock ≤ threshold
- Approval workflow lengkap untuk restock: Request → Review → Approve → Received
- Track supplier/vendor per item
- Purchase order generation
- Restock history & analytics

**Enhancement di entity `Restock`:**
```csharp
public Guid? SupplierId { get; set; }
public decimal EstimatedCost { get; set; }
public string PurchaseOrderNumber { get; set; }
public DateTime? ReceivedDate { get; set; }
public int ReceivedQuantity { get; set; }
```

---

### 1.10 Item Tagging & Custom Attributes

**Fitur baru:**
- Custom tags per item (contoh: "Urgent", "VIP Only", "Fragile")
- Custom attribute fields yang bisa didefinisikan admin (contoh: "Warna", "Ukuran", "Spesifikasi RAM")
- Filter dan search berdasarkan tags/attributes

**Entities baru:**
```
Tag (Id, Name, Color)
ItemTag (ItemId, TagId)
CustomAttribute (Id, Name, DataType, CategoryId)  // attribute berlaku per kategori
ItemAttributeValue (ItemId, CustomAttributeId, Value)
```

---

### 1.11 Warranty Management

**Fitur baru:**
- Track informasi garansi per item (start date, end date, provider, terms)
- Alert sebelum garansi expire
- Claim warranty workflow
- Lampirkan dokumen garansi (scan, upload)

**Entity baru:**
```
Warranty (Id, ItemId, Provider, StartDate, EndDate, Terms, DocumentPath, ClaimStatus)
```

---

## 2. Workflow & Approval

### 2.1 Batch Approve / Reject Requests

**Status saat ini:** Admin memproses request satu per satu.

**Fitur baru:**
- Checkbox multi-select pada list request
- "Approve All Selected" / "Reject All Selected" button
- Batch admin note untuk rejection
- Confirmation modal sebelum batch action
- Audit log per item yang di-batch process

---

### 2.2 Multi-Level Approval

**Status saat ini:** Single-level approval (Admin only).

**Fitur baru:**
- Configurable approval chain: Employee → Manager → Admin
- Approval rules berdasarkan: kategori, quantity, value
- Escalation otomatis jika pending terlalu lama
- Delegation: admin bisa delegate approval ke orang lain saat cuti

**Entity baru:**
```
ApprovalChain (Id, Name, Description)
ApprovalStep (ChainId, StepOrder, RoleRequired, AutoEscalateDays)
ApprovalLog (RequestId, StepId, ApprovedBy, Action, Timestamp, Notes)
```

---

### 2.3 Item Transfer Between Users

**Status saat ini:** Tidak ada fitur transfer — item harus dikembalikan dulu, lalu dipinjam lagi oleh user baru.

**Fitur baru:**
- Transfer request: User A → User B
- Approval diperlukan (admin atau manager)
- History tercatat di audit log dan item history
- Notifikasi ke kedua user

---

### 2.4 Return Extension Request

**Fitur baru:**
- Employee bisa request perpanjangan waktu pinjam sebelum jatuh tempo
- Admin approve/reject extension
- Maximum extension count configurable
- Auto-notification sebelum jatuh tempo

---

### 2.5 Scheduled/Recurring Requests

**Fitur baru:**
- Employee bisa buat request berulang (contoh: setiap Senin butuh projector)
- Admin approve recurring schedule sekali
- Auto-create request pada jadwal yang ditentukan
- Calendar view untuk lihat jadwal recurring

---

### 2.6 Request Priority System

**Fitur baru:**
- Priority levels: Low, Normal, High, Urgent
- Urgent request → notifikasi real-time ke semua admin
- SLA berdasarkan priority (contoh: Urgent harus di-approve dalam 1 jam)
- Visual indicator pada request list

---

### 2.7 Waitlist / Queue System

**Fitur baru:**
- Jika item tidak tersedia, employee bisa masuk waitlist
- Auto-notification ketika item tersedia kembali
- Queue position visibility
- Option untuk cancel dari waitlist

**Entity baru:**
```
Waitlist (Id, ItemId, UserId, Position, RequestedDate, NotifiedDate, Status)
```

---

### 2.8 Configurable Business Rules Engine

**Fitur baru:**
- Admin bisa konfigurasi rules tanpa coding:
  - Max items per employee (per kategori/total)
  - Max borrow duration per kategori
  - Auto-reject jika employee punya item terlambat
  - Blacklist period setelah item hilang/rusak
- Rules disimpan di database, evaluasi saat request

---

## 3. Reporting & Analytics

### 3.1 Export ke CSV / Excel / PDF

**Status saat ini:** Hanya import (CSV/Excel). **Tidak ada export sama sekali.**

**Fitur baru:**
- Export semua data list ke CSV, Excel, atau PDF:
  - Inventory list (per kategori, lokasi, status)
  - Request history
  - Active requests
  - Audit log
  - Item history
  - User list
- Customizable columns (pilih kolom yang mau di-export)
- Scheduled export (kirim report otomatis via email setiap bulan)

**Library:** ClosedXML (Excel), CsvHelper (sudah ada), QuestPDF atau iTextSharp (PDF)

---

### 3.2 Advanced Dashboard & Analytics

**Status saat ini:** Dashboard hanya menampilkan KPI cards + monthly trend chart (admin saja).

**Fitur baru:**
- **Admin Dashboard Enhancement:**
  - Item utilization rate (berapa % waktu item dipinjam vs idle)
  - Average borrow duration per category
  - Top 10 most requested items
  - Top 10 most active users (borrowers)
  - Cost analysis (maintenance cost, depreciation)
  - Overdue rate trend
  - Request approval time (SLA tracking)
  - Stock turnover ratio

- **Employee Dashboard Enhancement:**
  - Personal borrow history chart
  - Items currently borrowed with due dates
  - Upcoming reservations calendar
  - Personal propose status overview

---

### 3.3 Reservation Calendar View

**Status saat ini:** FullCalendar.js sudah di-include di semua layout, tapi **tidak digunakan di view manapun**.

**Fitur baru:**
- Calendar view untuk reservasi (admin: semua reservasi, employee: reservasi sendiri)
- Drag-and-drop untuk create/extend reservation
- Color coding per status (waiting, approved, rejected)
- Filter by item/category/user
- Conflict detection visual (highlight jika ada overlap)

**Implementasi:** Manfaatkan `fullcalendar.js` yang sudah ada, buat API endpoint `/api/reservations/calendar` yang return event data.

---

### 3.4 Custom Report Builder

**Fitur baru:**
- Admin bisa buat custom report dengan pilih:
  - Data source (items, requests, active requests, history, dll.)
  - Filters (date range, category, status, user)
  - Grouping (per category, per month, per user)
  - Aggregation (count, sum, average)
- Save report template untuk dijalankan ulang
- Schedule report generation

---

### 3.5 Real-time Operational Dashboard

**Fitur baru:**
- Live dashboard dengan SignalR:
  - Jumlah item sedang dipinjam (real-time)
  - Queue length di pickup desk dan return desk
  - Pending approval count (real-time)
  - Alert untuk overdue items (real-time)
- Map view: distribusi item per lokasi (manfaatkan `jvectormap.js` yang sudah ada)

---

### 3.6 Heatmap & Usage Pattern Analysis

**Fitur baru:**
- Heatmap penggunaan item per hari/jam dalam seminggu
- Identifikasi peak hours untuk item tertentu
- Seasonal pattern analysis
- Recommendation: "Item X paling banyak dipinjam Selasa pagi, pertimbangkan tambah stock"

---

### 3.7 KPI Scorecards per Department

**Fitur baru:**
- Group employees by department
- KPI per department: borrow rate, overdue rate, damage rate, propose rate
- Comparative analysis antar department
- Monthly scorecard email

---

## 4. Notifikasi & Komunikasi

### 4.1 Automated Overdue Reminders

**Status saat ini:** Admin harus manual klik "NotifyUser" untuk kirim email ke user terlambat, satu per satu.

**Fitur baru:**
- Background job (Hangfire/Quartz.NET) untuk auto-check overdue items
- Kirim reminder otomatis:
  - H-3 sebelum jatuh tempo → reminder
  - Hari jatuh tempo → warning
  - H+1 terlambat → first reminder
  - H+3 → escalation ke manager
  - H+7 → escalation ke admin
- Configurable reminder schedule per kategori

**Implementasi:**
```csharp
// Hosted Service atau Hangfire Recurring Job
services.AddHostedService<OverdueReminderService>();
// atau
RecurringJob.AddOrUpdate<IOverdueReminderService>(
    "check-overdue", x => x.CheckAndNotifyAsync(), Cron.Daily);
```

---

### 4.2 In-App Chat / Messaging

**Fitur baru:**
- Chat antara admin dan employee terkait request tertentu
- Chat thread ter-link ke request/active request
- Notifikasi real-time (extend `NotificationHub`)
- Chat history tersimpan

**Entity baru:**
```
ChatMessage (Id, SenderId, ReceiverId, RequestId?, ActiveRequestId?, Message, SentAt, IsRead)
```

---

### 4.3 Push Notification (Browser/Mobile)

**Fitur baru:**
- Web Push Notification via Service Worker
- Employee terima notifikasi bahkan saat browser di-minimize
- Admin terima alert untuk request baru, item overdue
- Configurable: user bisa pilih notifikasi mana yang mau diterima

**Implementasi:** Web Push API + `Notification` JavaScript API + VAPID keys

---

### 4.4 Email Digest (Daily/Weekly Summary)

**Fitur baru:**
- Admin terima email digest harian:
  - Pending requests count
  - Overdue items count
  - New proposals
  - Low stock alerts
- Employee terima weekly summary:
  - Items currently borrowed + due dates
  - Proposal status updates
  - Upcoming reservations

---

### 4.5 Customizable Email Templates (Admin UI)

**Status saat ini:** Email template berupa static HTML files (`EmailConfirmation.html`, `RequestApproved.html`, `RequestRejected.html`).

**Fitur baru:**
- Admin bisa edit email template via WYSIWYG editor di web
- Template variables (placeholders) yang bisa di-drag-drop
- Preview before save
- Version history untuk templates
- Tambah template baru: Overdue Reminder, Return Confirmation, Reservation Approved, dll.

---

### 4.6 Announcement / Bulletin Board

**Fitur baru:**
- Admin bisa post announcement (contoh: "Gudang tutup tanggal 25-26 Desember")
- Muncul sebagai banner/card di dashboard employee
- Prioritas announcement: Info, Warning, Urgent
- Schedule publish/unpublish date

**Entity baru:**
```
Announcement (Id, Title, Content, Priority, PublishDate, ExpiryDate, CreatedBy, IsActive)
```

---

### 4.7 SMS / WhatsApp Notification

**Fitur baru:**
- Integrasi Twilio / WhatsApp Business API untuk notifikasi kritis
- Kirim SMS saat item overdue > 7 hari
- OTP via SMS untuk verifikasi akun

---

## 5. User & Access Management

### 5.1 Additional Roles (Manager, Warehouse Clerk, Auditor)

**Status saat ini:** Hanya 2 role: Admin dan Employee.

**Fitur baru:**

| Role | Capabilities |
|------|-------------|
| **Super Admin** | Konfigurasi sistem, manage admins, lihat semua audit log |
| **Admin** | Existing admin capabilities |
| **Manager** | Approve request team sendiri, lihat laporan team |
| **Warehouse Clerk** | Pickup desk, return desk, stocktake, tidak bisa approve request |
| **Auditor** | Read-only access ke semua data + semua audit log + reports |
| **Employee** | Existing employee capabilities |

---

### 5.2 Department / Team Structure

**Status saat ini:** Tidak ada konsep department. User berdiri sendiri.

**Fitur baru:**
- Entity `Department` (Id, Name, ManagerId, ParentDepartmentId)
- Employee belongs to Department
- Manager bisa lihat request dari team-nya
- Budget allocation per department
- Report per department

---

### 5.3 Activity-Based Access Control (Fine-Grained Permissions)

**Fitur baru:**
- Buat granular permissions alih-alih hanya role-based:
  - `item.create`, `item.edit`, `item.delete`, `item.view`
  - `request.approve`, `request.reject`, `request.view`
  - `report.export`, `report.create`
- Permission groups (bundles)
- Admin bisa assign/revoke permissions per user

**Entity baru:**
```
Permission (Id, Name, Description, Module)
RolePermission (RoleId, PermissionId)
UserPermission (UserId, PermissionId, IsGranted)
```

---

### 5.4 User Self-Service Portal

**Fitur baru:**
- Employee bisa lihat semua item yang sedang dipinjam di satu page
- "My Items" dashboard: borrowed items + due date + renew button
- Self-service return initiation (request return → admin confirms at desk)
- Personal inventory: items yang di-assign secara permanen (laptop, meja, dll.)

---

### 5.5 Employee Onboarding / Offboarding Workflow

**Fitur baru:**
- **Onboarding:** Auto-assign default items saat employee baru (laptop, ID card, headset)
  - Configurable checklist per role/department
  - Track completion status
- **Offboarding:** Checklist semua item yang harus dikembalikan
  - Block account deactivation sampai semua item returned
  - Automated reminder ke employee yang akan resign

---

### 5.6 SSO Integration (SAML / Azure AD)

**Fitur baru:**
- Selain Google OAuth, support:
  - Azure Active Directory (Microsoft Entra ID)
  - SAML 2.0 (for enterprise)
  - LDAP sync untuk sinkronisasi user dari corporate directory

---

### 5.7 User Activity Analytics

**Fitur baru:**
- Login history per user
- Borrow frequency, return punctuality score
- Risk score (berdasarkan history: damaged items, late returns)
- Admin bisa lihat user profile lengkap dengan statistics

---

## 6. Import, Export & Integrasi

### 6.1 REST API Layer

**Status saat ini:** Hanya beberapa JSON endpoint ( `/Notifications/GetUnreadCount`, chart data). **Tidak ada proper REST API.**

**Fitur baru:**
- Full REST API untuk semua entities:
  - `GET/POST/PUT/DELETE /api/v1/items`
  - `GET/POST /api/v1/requests`
  - `GET /api/v1/active-requests`
  - `GET /api/v1/reports/...`
  - dll.
- API versioning (v1, v2, ...)
- JWT token authentication untuk API
- Swagger/OpenAPI documentation
- Rate limiting

**Implementasi:**
```csharp
// Tambah API controllers terpisah dari MVC controllers
[ApiController]
[Route("api/v1/[controller]")]
[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
public class ItemsApiController : ControllerBase { ... }

// Swagger
builder.Services.AddSwaggerGen();
```

---

### 6.2 Webhook System

**Fitur baru:**
- Admin bisa register webhook URL untuk events tertentu:
  - `request.created`, `request.approved`, `request.rejected`
  - `item.low_stock`, `item.created`
  - `active_request.overdue`
- POST JSON payload ke registered URL saat event terjadi
- Retry mechanism + delivery log

**Entity baru:**
```
WebhookSubscription (Id, Url, Secret, Events[], IsActive, CreatedBy)
WebhookDelivery (Id, SubscriptionId, Event, Payload, ResponseCode, Attempts, LastAttemptAt)
```

---

### 6.3 Integration dengan ERP / Accounting System

**Fitur baru:**
- Sync data item dengan SAP, Oracle ERP, atau sistem finance
- Auto-update purchase price, asset class
- Export depreciation data ke accounting software
- Integration via REST API atau file-based (scheduled CSV export)

---

### 6.4 Integration dengan HR System

**Fitur baru:**
- Sync employee data (nama, department, manager, status aktif)
- Auto-create/deactivate user saat ada new hire/resign dari HR system
- Sync department hierarchy

---

### 6.5 Slack / Microsoft Teams Integration

**Fitur baru:**
- Notification ke Slack/Teams channel saat:
  - Request baru masuk
  - Item overdue
  - Low stock alert
- Slash command: `/inventory search laptop` → return availble items
- Approve/reject request langsung dari Slack/Teams

---

### 6.6 Google Workspace Integration

**Status saat ini:** Google OAuth untuk login saja.

**Fitur baru:**
- Sync dengan Google Calendar: otomatis buat calendar event saat reservation approved
- Google Sheets export
- Google Drive integration untuk file attachment

---

### 6.7 Bulk Operations API

**Fitur baru:**
- Bulk create items via API (JSON array)
- Bulk update item categories
- Bulk transfer items antar lokasi
- Bulk deactivate items
- Import dari barcode scanner batch file

---

## 7. Mobile & UX

### 7.1 Progressive Web App (PWA)

**Fitur baru:**
- Service Worker untuk offline capability
- Add to Home Screen on mobile
- Push notifications (lihat §4.3)
- Cache frequently accessed pages (item list, dashboard)
- Background sync saat kembali online

**Implementasi:**
```javascript
// manifest.json
{ "name": "IMS", "short_name": "IMS", "start_url": "/", "display": "standalone" }

// service-worker.js
self.addEventListener('install', event => { ... });
self.addEventListener('fetch', event => { ... });
```

---

### 7.2 Responsive Mobile Scanner Mode

**Fitur baru:**
- Dedicated mobile-optimized scanner page
- Gunakan kamera ponsel sebagai barcode/QR scanner
- Quick actions: scan → lihat detail → approve/confirm pickup/return
- Minimal UI, fokus pada scan & action
- Support scan tanpa internet (queue then sync)

**Library:** `html5-qrcode` atau `QuaggaJS` untuk browser-based scanning.

---

### 7.3 Dark Mode

**Fitur baru:**
- Toggle dark/light mode
- Simpan preferensi user
- CSS custom properties untuk theme switching
- Auto-detect system preference (`prefers-color-scheme`)

---

### 7.4 Keyboard Shortcuts

**Fitur baru:**
- `Ctrl+K` → global search
- `Ctrl+N` → new request
- `Ctrl+/` → shortcut help
- Arrow keys untuk navigasi list

---

### 7.5 Drag-and-Drop File Upload

**Status saat ini:** Standard file input untuk upload.

**Fitur baru:**
- Drag-and-drop area pada halaman create/edit item
- Image preview sebelum upload
- Progress bar saat upload
- Multi-file upload

---

### 7.6 Global Search (Unified Search)

**Status saat ini:** Setiap module punya search sendiri. Tidak ada global search.

**Fitur baru:**
- Single search bar di header
- Search across: Items, Users, Requests, Categories, Active Requests
- Result grouping per entity type
- Search suggestion / autocomplete
- Recent searches

**Implementasi:**
```csharp
[HttpGet("/api/search")]
public async Task<IActionResult> GlobalSearch([FromQuery] string q)
{
    var items = await _itemRepo.SearchAsync(q);
    var users = await _userRepo.SearchAsync(q);
    var requests = await _requestRepo.SearchAsync(q);
    return Ok(new { items, users, requests });
}
```

---

### 7.7 Improved Item Catalog View

**Fitur baru:**
- Grid view (kartu) vs List view toggle
- Image-centric catalog browsing
- Quick view popup (tanpa navigasi ke halaman baru)
- "Similar items" recommendation
- Recently viewed items

---

### 7.8 Multi-Language Support (i18n)

**Fitur baru:**
- Support Bahasa Indonesia dan English (minimal)
- Resource-based localization (`*.resx` files)
- User bisa pilih bahasa preferensi
- Admin bisa tambah/edit terjemahan

---

### 7.9 Accessibility (a11y) Improvements

**Fitur baru:**
- WCAG 2.1 AA compliance
- Screen reader support (aria labels)
- Keyboard navigation untuk semua interactive elements
- High contrast mode
- Focus indicators yang jelas

---

## 8. Keamanan & Compliance

### 8.1 Comprehensive Audit Trail

**Status saat ini:** `AuditLog` hanya record Action, Role, Name, Timestamp. Tidak ada detail _apa yang berubah_.

**Fitur baru:**
- Track perubahan field-level: `OldValue`, `NewValue`, `FieldName`
- IP address dan User Agent di setiap audit entry
- Non-repudiation: audit log tidak bisa dihapus
- Exportable audit trail untuk compliance
- Filter audit log by entity type, date range, user, action type

**Enhancement entity `AuditLog`:**
```csharp
public string EntityType { get; set; }     // "Item", "Request", dll.
public string EntityId { get; set; }
public string ActionType { get; set; }     // "Create", "Update", "Delete"
public string OldValues { get; set; }      // JSON serialized
public string NewValues { get; set; }      // JSON serialized
public string IpAddress { get; set; }
public string UserAgent { get; set; }
```

---

### 8.2 Two-Factor Authentication Enforcement

**Status saat ini:** 2FA page exists, tapi tidak enforced.

**Fitur baru:**
- Admin bisa enforce 2FA untuk role tertentu
- Support TOTP (Google Authenticator, Authy)
- Recovery codes management
- Backup authentication method (email OTP)

---

### 8.3 Session Management

**Fitur baru:**
- User bisa lihat semua active sessions
- Remote logout: "Sign out from all devices"
- Session timeout warning popup
- Single session mode (opsional): login baru akan logout session lama

---

### 8.4 Data Encryption at Rest

**Fitur baru:**
- Encrypt sensitive fields di database:
  - Employee personal info
  - Email addresses
  - Phone numbers
- Column-level encryption via EF Core value converters

---

### 8.5 GDPR / Data Privacy Compliance

**Fitur baru:**
- Right to access: user bisa download semua data personal-nya
- Right to erasure: permanently delete user data (dengan audit trail)
- Data retention policy: auto-purge data lebih tua dari X tahun
- Cookie consent banner
- Privacy policy page

---

### 8.6 IP Whitelisting / Geoblocking

**Fitur baru:**
- Admin bisa restrict access dari IP address tertentu
- Block login dari lokasi yang tidak diexpect
- VPN detection

---

### 8.7 Password Policy Configuration

**Status saat ini:** Password rules berbeda antar validator (lihat REFACTORING_GUIDE).

**Fitur baru:**
- Configurable password policy dari admin panel:
  - Minimum length, complexity requirements
  - Password expiry (force change every N days)
  - Password history (tidak bisa reuse N password terakhir)
  - Account lockout threshold & duration

---

## 9. Infrastruktur & DevOps

### 9.1 Database Migration ke SQL Server / PostgreSQL

**Status saat ini:** SQLite — cocok untuk development, tidak ideal untuk production dengan banyak concurrent users.

**Fitur baru:**
- Support SQL Server dan PostgreSQL sebagai production database
- Database provider configurable via `appsettings.json`
- Migration tooling yang support multiple providers

---

### 9.2 Caching Layer (Redis / In-Memory)

**Fitur baru:**
- Cache frequently accessed data:
  - Category/SubCategory dropdown lists
  - Item type lists
  - City lists
  - Dashboard KPI counts
- Distributed cache (Redis) untuk multi-instance deployment
- Cache invalidation saat data berubah

**Implementasi:**
```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});
// atau untuk single-instance:
builder.Services.AddMemoryCache();
```

---

### 9.3 Background Job Processing

**Status saat ini:** Tidak ada background job. Seed data dijalankan di startup.

**Fitur baru:**
- Hangfire atau Quartz.NET untuk scheduled/background tasks:
  - Overdue check & notification (daily)
  - Rejected request cleanup (daily)
  - Database backup (weekly)
  - Report generation & email (scheduled)
  - Low stock check (daily)
  - Warranty expiry check (weekly)

---

### 9.4 Health Checks & Monitoring

**Fitur baru:**
- ASP.NET Core Health Checks:
  - Database connectivity
  - SMTP connectivity
  - Redis connectivity (if used)
  - Disk space
- Endpoint: `/health` untuk monitoring tools
- Integration dengan Prometheus + Grafana, atau Application Insights

---

### 9.5 Docker Containerization

**Fitur baru:**
- Dockerfile untuk build dan run aplikasi
- docker-compose untuk full stack (app + db + redis + signalr)
- CI/CD pipeline dengan Docker images

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80 443
# ...
```

---

### 9.6 CI/CD Pipeline

**Fitur baru:**
- GitHub Actions / Azure DevOps pipeline:
  - Build → Test → Analyze → Deploy
  - SonarQube code quality gate
  - Automated database migrations
  - Blue/green deployment

---

### 9.7 Centralized Logging (ELK / Seq)

**Status saat ini:** log4net menulis ke file lokal.

**Fitur baru:**
- Structured logging ke centralized platform:
  - Seq (lightweight, .NET-friendly)
  - ELK Stack (Elasticsearch + Logstash + Kibana)
  - Application Insights
- Correlation ID per request
- Log search dan visualization

---

### 9.8 Rate Limiting & Throttling

**Fitur baru:**
- Protect API & form endpoints dari abuse
- Rate limit per user, per IP, per endpoint
- ASP.NET Core built-in rate limiter (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.PermitLimit = 100;
    });
});
```

---

### 9.9 Feature Flags

**Fitur baru:**
- Hidupkan/matikan fitur tanpa deploy ulang
- A/B testing capability
- Gradual rollout (aktifkan untuk % user tertentu)
- Library: `Microsoft.FeatureManagement`

---

## 10. Gamification & Employee Engagement

### 10.1 Borrower Reputation Score

**Fitur baru:**
- Setiap employee punya "reliability score" berdasarkan:
  - Return tepat waktu: +10 poin
  - Terlambat: -5 poin per hari
  - Item rusak: -20 poin
  - Item hilang: -50 poin
  - Item dikembalikan dalam kondisi baik: +5 poin
- Score mempengaruhi approval priority (employee dengan score tinggi → auto-approve)
- Leaderboard (opt-in)

---

### 10.2 Achievement Badges

**Fitur baru:**
- Badge otomatis saat milestone tercapai:
  - "First Borrow" — pinjam item pertama kali
  - "Always On Time" — 10 return berturut-turut tepat waktu
  - "Proposer" — propose 5 item yang diterima
  - "Clean Record" — 1 tahun tanpa damage/lost
- Tampilkan di profil user

---

### 10.3 Feedback & Rating System

**Fitur baru:**
- Employee bisa rate item setelah mengembalikan (1-5 stars + komentar)
- Rate kondisi item: "Item ini sudah usang, pertimbangkan penggantian"
- Admin bisa lihat agregat rating per item
- Influence purchasing decisions

**Entity baru:**
```
ItemReview (Id, ItemId, UserId, ActiveRequestId, Rating, Comment, CreatedAt)
```

---

## 11. AI & Smart Features

### 11.1 Predictive Stock Forecasting

**Fitur baru:**
- Machine learning model untuk prediksi demand item berdasarkan historical data
- Auto-suggest restock quantity dan timing
- Seasonal pattern detection
- Alert: "Berdasarkan trend, item X akan habis dalam 2 minggu"

---

### 11.2 Smart Item Recommendation

**Fitur baru:**
- "Users who borrowed this also borrowed..." recommendation
- Recommendation berdasarkan department, role, past history
- "You might also need..." saat create request

---

### 11.3 Anomaly Detection

**Fitur baru:**
- Detect unusual patterns:
  - Employee yang tiba-tiba borrow banyak item tidak biasa
  - Spike di request volume untuk item tertentu
  - Unusual login patterns (potential account compromise)
- Alert admin untuk review

---

### 11.4 Natural Language Search

**Fitur baru:**
- Search dengan bahasa natural: "laptop yang tersedia di Jakarta"
- AI-powered intent recognition
- Fuzzy matching untuk typo tolerance
- Implementation: Azure Cognitive Search / Elasticsearch

---

### 11.5 Chatbot Assistant

**Fitur baru:**
- Chat widget di halaman (employee-facing):
  - "Bagaimana cara meminjam projector?"
  - "Berapa item yang sedang saya pinjam?"
  - "Item apa saja yang tersedia kategori elektronik?"
- Powered by rule-based bot atau LLM integration

---

### 11.6 Image Recognition for Item Condition

**Fitur baru:**
- Saat return, employee upload foto kondisi item
- AI model menilai: Good, Acceptable, Damaged
- Auto-flag items yang terdeteksi rusak untuk review admin
- Library: Azure Computer Vision / custom ML.NET model

---

### 11.7 OCR untuk Import Data

**Fitur baru:**
- Upload foto/scan dokumen invoice/packing list
- OCR extract data item (nama, quantity, harga)
- Auto-populate form create item
- Library: Tesseract OCR / Azure Form Recognizer

---

## 12. Multi-Tenancy & Scalability

### 12.1 Multi-Tenant Support

**Status saat ini:** Single organization.

**Fitur baru:**
- Support multiple organizations dalam satu instance
- Tenant isolation: setiap org hanya lihat data sendiri
- Configurable branding per tenant (logo, warna, nama)
- Shared vs dedicated database per tenant

---

### 12.2 Microservices Architecture (Long-term)

**Fitur baru (jangka panjang):**
- Decompose ke microservices:
  - Item Service
  - Request Service
  - Notification Service
  - User Service
  - Report Service
- Event-driven communication (RabbitMQ / Azure Service Bus)
- API Gateway (Ocelot / YARP)

---

### 12.3 Horizontal Scaling

**Fitur baru:**
- Stateless application design (session di Redis)
- SignalR backplane (Redis/Azure SignalR Service)
- File storage ke cloud (Azure Blob / AWS S3) alih-alih local disk
- Load balancer ready

---

### 12.4 Multi-Region Deployment

**Fitur baru:**
- Deploy ke beberapa region untuk latency rendah
- Database replication
- CDN untuk static assets
- Geo-routing

---

## 13. Ringkasan Prioritas

### 🔴 High Impact, Low Effort (Quick Wins)

| # | Fitur | Effort | Impact |
|---|-------|:------:|:------:|
| 1 | Export CSV/Excel/PDF (§3.1) | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 2 | Batch Approve/Reject (§2.1) | ⭐⭐ | ⭐⭐⭐⭐ |
| 3 | Automated Overdue Reminders (§4.1) | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 4 | Reservation Calendar View (§3.3) | ⭐⭐ | ⭐⭐⭐⭐ |
| 5 | Global Search (§7.6) | ⭐⭐ | ⭐⭐⭐⭐ |
| 6 | Permanent Asset QR/Barcode (§1.1) | ⭐⭐ | ⭐⭐⭐⭐ |
| 7 | Dark Mode (§7.3) | ⭐ | ⭐⭐⭐ |
| 8 | Email Digest (§4.4) | ⭐⭐ | ⭐⭐⭐ |
| 9 | PWA Basic (§7.1) | ⭐⭐ | ⭐⭐⭐ |
| 10 | Health Checks (§9.4) | ⭐ | ⭐⭐⭐ |

### 🟡 High Impact, Medium Effort (Next Sprint)

| # | Fitur | Effort | Impact |
|---|-------|:------:|:------:|
| 11 | REST API Layer (§6.1) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 12 | Advanced Dashboard (§3.2) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 13 | Additional Roles (§5.1) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 14 | Maintenance Log (§1.3) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 15 | Background Job Processing (§9.3) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 16 | Comprehensive Audit Trail (§8.1) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 17 | Department Structure (§5.2) | ⭐⭐⭐ | ⭐⭐⭐ |
| 18 | Restock Full Workflow (§1.9) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 19 | Configurable Business Rules (§2.8) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 20 | Docker Containerization (§9.5) | ⭐⭐⭐ | ⭐⭐⭐⭐ |

### 🟢 High Impact, High Effort (Roadmap)

| # | Fitur | Effort | Impact |
|---|-------|:------:|:------:|
| 21 | Multi-Location Inventory (§1.2) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 22 | Multi-Level Approval (§2.2) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 23 | Asset Depreciation (§1.4) | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 24 | Physical Stocktake (§1.5) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 25 | Serial Number Tracking (§1.6) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 26 | Mobile Scanner Mode (§7.2) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 27 | Webhook System (§6.2) | ⭐⭐⭐ | ⭐⭐⭐ |
| 28 | Custom Report Builder (§3.4) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 29 | Database Migration (SQL Server/PostgreSQL) (§9.1) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 30 | Fine-Grained Permissions (§5.3) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 🔵 Long-term Vision

| # | Fitur | Effort | Impact |
|---|-------|:------:|:------:|
| 31 | Multi-Tenant Support (§12.1) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 32 | Predictive Stock Forecasting (§11.1) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 33 | Chatbot Assistant (§11.5) | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 34 | Microservices Architecture (§12.2) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 35 | ERP Integration (§6.3) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 36 | AI Anomaly Detection (§11.3) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

---
