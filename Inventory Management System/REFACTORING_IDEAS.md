# Refactoring Ideas — Inventory Management System


---

## Daftar Isi

1. [P0 — Keamanan (Security)](#1-p0--keamanan-security)
2. [P0 — Bug Kritis & Deadlock](#2-p0--bug-kritis--deadlock)
3. [P1 — Arsitektur & SOLID](#3-p1--arsitektur--solid)
4. [P1 — Persistence Layer](#4-p1--persistence-layer)
5. [P1 — Service Layer](#5-p1--service-layer)
6. [P2 — Models & DTOs](#6-p2--models--dtos)
7. [P2 — Validators](#7-p2--validators)
8. [P2 — AutoMapper Profiles](#8-p2--automapper-profiles)
9. [P2 — Controller Layer](#9-p2--controller-layer)
10. [P2 — Utilities](#10-p2--utilities)
11. [P3 — Testing](#11-p3--testing)
12. [P3 — Code Style & Naming](#12-p3--code-style--naming)
13. [P3 — Program.cs & DI Configuration](#13-p3--programcs--di-configuration)
14. [Ringkasan Checklist](#14-ringkasan-checklist)

---

## 1. P0 — Keamanan (Security)

### 1.1 Secret / Credential Bocor di Source Control

| File | Data yang bocor |
|------|----------------|
| `appsettings.json` | Google OAuth `ClientId` & `ClientSecret` |
| `appsettings.json` | SMTP password (`wgjg pasv xsbi fqhp`) |
| `appsettings.json` | Admin default password (`Admin123@`) |
| `client_secret_841964895929-...json` (root) | Google OAuth `client_secret` penuh |

**Solusi:**
```bash
# 1. Pindahkan semua secret ke User Secrets (development)
dotnet user-secrets init
dotnet user-secrets set "Authentication:Google:ClientSecret" "GOCSPX-..."
dotnet user-secrets set "SenderEmailDetails:AdminEmailPassword" "wgjg..."

# 2. Untuk production: gunakan Azure Key Vault / Environment Variables
# 3. Tambahkan ke .gitignore:
#    client_secret_*.json
#    appsettings.*.json (kecuali template tanpa value)
# 4. ROTASI semua secret yang sudah terekspos
```

### 1.2 `NotificationsController` Tanpa `[Authorize]`

**File:** `Controllers/NotificationsController.cs`

Controller ini bisa diakses tanpa login. Ditambah `user` bisa `null` → `NullReferenceException`.

```csharp
// SEBELUM (BERBAHAYA)
public class NotificationsController : Controller { ... }

// SESUDAH
[Authorize]
public class NotificationsController : Controller { ... }
```

### 1.3 Email Dikirim via GET Request (CSRF)

**File:** `Areas/Admin/Controllers/ActiveRequestController.cs` — `NotifyUser`

GET request tidak dilindungi anti-forgery token. Penyerang bisa mengirim email atas nama admin via link injection.

```csharp
// SEBELUM (BERBAHAYA)
[HttpGet]
public async Task<IActionResult> NotifyUser(Guid? ActiveRequestId, string? email, ...)

// SESUDAH
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> NotifyUser(Guid? ActiveRequestId, string? email, ...)
```

### 1.4 File Upload Path Traversal

**File:** `Areas/Admin/Services/FileImportService.cs`

Filename asli dari user digunakan langsung tanpa sanitasi.

```csharp
// SEBELUM (BERBAHAYA)
var filePath = Path.Combine(uploadFolder, file.FileName);

// SESUDAH
var safeFileName = Path.GetRandomFileName() + Path.GetExtension(file.FileName);
var filePath = Path.Combine(uploadFolder, safeFileName);
```

### 1.5 Tidak Ada Validasi Tipe File pada Upload Gambar Profil

**File:** `Services/UserProfileService.cs`

Ekstensi apapun diterima (.exe, .html, dll).

```csharp
// SESUDAH — Tambahkan whitelist
var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".webp" };
var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
if (!allowedExtensions.Contains(extension))
    return IdentityResult.Failed(new IdentityError { Description = "File type not allowed." });
```

### 1.6 Tidak Ada Batas Ukuran File Upload

Seluruh validator hanya cek `file.Length > 0`, tidak ada maximum. Penyerang bisa upload file multi-GB.

```csharp
// Tambahkan rule di semua IFormFile validator:
RuleFor(x => x.Picture)
    .Must(file => file == null || file.Length <= 5 * 1024 * 1024)
    .WithMessage("Ukuran file maksimal 5 MB.");
```

### 1.7 Missing `[ValidateAntiForgeryToken]` pada POST Actions

Banyak POST action tanpa proteksi CSRF:
- `Areas/Admin/Controllers/ActiveRequestController.cs` — semua POST
- `Areas/Admin/Controllers/RequestController.cs` — `AdminNoteRejection`, `Approve`
- `Areas/Admin/Controllers/CategoryViewController.cs` — semua POST

**Solusi:** Tambahkan `[ValidateAntiForgeryToken]` pada setiap `[HttpPost]`, atau gunakan global filter:
```csharp
builder.Services.AddControllersWithViews(options =>
{
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

### 1.8 Employee Bisa Lihat Active Request Semua User

**File:** `Areas/Employee/Services/EmployeeActiveRequestService.cs`

`GetActiveRequestsAsync` mengambil SEMUA active request tanpa filter `UserId`.

```csharp
// SESUDAH — Filter by current user
var allActiveRequests = await _activeRequestRepository
    .GetAllAsync(includeProperties)
    .Where(ar => ar.UserId == currentUserId);
```

### 1.9 `SeedDataController` Aktif di Production

**File:** `Areas/Admin/Controllers/SeedDataController.cs`

Endpoint untuk generate/clear dummy data harus dibatasi hanya di development:
```csharp
#if DEBUG
[Area("Admin")]
[Authorize(Roles = AllRoles.Role_Admin)]
public class SeedDataController : Controller { ... }
#endif
```

---

## 2. P0 — Bug Kritis & Deadlock

### 2.1 Blocking `.Result` pada Async Call → Deadlock Risk

**File:** `Areas/Admin/Controllers/HomeController.cs`

```csharp
// SEBELUM (DEADLOCK RISK)
public IActionResult Index()
{
    var result = _homeAdminService.GetViewModelsAsync(User).Result;
    return View(result);
}

// SESUDAH
public async Task<IActionResult> Index()
{
    var result = await _homeAdminService.GetViewModelsAsync(User);
    return View(result);
}
```

### 2.2 Generic Type Bug — Repository Mewarisi Dirinya Sendiri

**File:** `Persistence/Repository/ActiveRequestEmployeeRepository.cs`

```csharp
// SEBELUM (BUG — DbSet<ActiveRequestEmployeeRepository> tidak ada)
public class ActiveRequestEmployeeRepository : Repository<ActiveRequestEmployeeRepository>, ...

// SESUDAH
public class ActiveRequestEmployeeRepository : Repository<ActiveRequest>, ...
```

### 2.3 N+1 Query — Load Seluruh Tabel ke Memory

**File:** `Persistence/Repository/RequestRepository.cs` — `GetRequestByItemIdWithDetailsAsync`

```csharp
// SEBELUM (LOAD SEMUA REQUEST KE MEMORY)
var requests = await _db.Set<Request>()
    .Include(r => r.Item).ThenInclude(i => i.Type)
    .Include(r => r.Item).ThenInclude(i => i.Category)
    .Where(r => !r.IsDeleted)
    .ToListAsync();  // ← semua data masuk memory
return requests.FirstOrDefault(r => r.ItemId == itemId); // filter di memory

// SESUDAH
return await _db.Set<Request>()
    .Include(r => r.Item).ThenInclude(i => i.Type)
    .Include(r => r.Item).ThenInclude(i => i.Category)
    .Where(r => !r.IsDeleted && r.ItemId == itemId)
    .FirstOrDefaultAsync();
```

### 2.4 `SaveChangesAsync()` Tidak Dipanggil (Data Hilang)

Beberapa method memodifikasi entity tapi tidak pernah menyimpan:

| File | Method |
|------|--------|
| `ItemRepository.cs` | `SoftDeleteItemsBySubCategoryIdAsync` |
| `ItemViewRepository.cs` | `SoftDeleteItemViewsBySubCategoryIdAsync` |
| `CategoryRepository.cs` | `ActivateCategoryAsync` |

**Solusi:** Tambahkan `await _db.SaveChangesAsync();` di akhir setiap method, atau adopsi Unit of Work pattern (lihat §4.1).

### 2.5 `SmtpClient` Tidak Di-Dispose (Resource Leak)

**File:** `Utilities/EmailSender.cs`

```csharp
// SEBELUM
var client = new SmtpClient(_host, _port) { ... };

// SESUDAH
using var client = new SmtpClient(_host, _port) { ... };
```

---

## 3. P1 — Arsitektur & SOLID

### 3.1 God Classes (Terlalu Banyak Dependency)

| Service | Dependencies | Rekomendasi Pecah Menjadi |
|---------|:-----------:|---------------------------|
| `ItemViewAdminService` | **19** | `ItemViewCrudService`, `ItemViewImportService`, `ItemViewPaginationService` |
| `ItemProposeAdminService` | **16** | `ItemProposeCrudService`, `ItemProposeApprovalService` |
| `ItemViewEmployeeService` | **12** | Split per concern |
| `UserViewService` | **11** | `UserCrudService`, `UserEmailValidationService` |

**Best Practice:** Sebuah class sebaiknya memiliki **≤ 5-7 dependency**. Lebih dari itu mengindikasikan SRP violation.

### 3.2 Service Menerima Parameter `Controller` (Layer Violation)

Service layer **tidak boleh** mengetahui Controller/MVC. Berikut service yang melanggar:

| Service | Method |
|---------|--------|
| `ItemHistoryService` | `GetItemHistoriesAsync(..., Controller controller)` |
| `AuditLogAdminService` | `SetupViewBag(Controller controller, ...)` |
| `ItemViewAdminService` | `SetupViewBagAsync(Controller controller, ...)` |
| `UserViewService` | `GetPaginatedUsersAsync(..., Controller controller)` |

**Solusi:** Kembalikan data pagination sebagai DTO, biarkan controller yang set ViewBag:

```csharp
// SEBELUM (di service)
public async Task SetupViewBag(Controller controller, PaginationResult<T> result)
{
    controller.ViewBag.CurrentPage = result.CurrentPage;
    ...
}

// SESUDAH (service mengembalikan DTO)
public PaginationMetadata GetPaginationMetadata(PaginationResult<T> result)
{
    return new PaginationMetadata { CurrentPage = result.CurrentPage, ... };
}

// Controller:
var meta = _service.GetPaginationMetadata(result);
ViewBag.CurrentPage = meta.CurrentPage;
```

### 3.3 6+ Pola Result Type Yang Berbeda

| Pattern | Digunakan Oleh |
|---------|---------------|
| `ServiceResult<T>` | ChartViewService, ProposeLimitService, ReservationAdminService |
| `CategoryServiceResultDto<T>` | CategoryAdminService |
| `SubCategoryServiceResultDto<T>` | SubCategoryAdminService |
| `OperationResult` | RequestAdminService |
| `ServiceResultDto` | UserViewService |
| `(bool, string, ...)` tuple | ItemViewAdminService, CsvService |
| `IdentityResult` | UserProfileService |

**Solusi:** Konsolidasi ke satu tipe generik:

```csharp
// Gunakan satu tipe saja di seluruh codebase:
public class Result<T>
{
    public bool IsSuccess { get; init; }
    public string Message { get; init; } = string.Empty;
    public T? Data { get; init; }
    public List<string> Errors { get; init; } = new();

    public static Result<T> Success(T data, string message = "") => ...;
    public static Result<T> Failure(string message, List<string>? errors = null) => ...;
}
```

### 3.4 Duplikasi Masif Antara Admin & Employee Services

Service-service berikut **copy-paste** dengan perbedaan minimal:

| Admin Service | Employee Service | Kode Duplikat |
|--------------|-----------------|:-------------:|
| `ActiveRequestService` | `EmployeeActiveRequestService` | ~80% |
| `AuditLogAdminService` | `EmployeeAuditLogService` | ~90% |
| `NotificationCenterService` | `EmployeeNotificationService` | ~90% |
| `ItemHistoryService` | `EmployeeItemHistoryService` | ~70% |

**Solusi:** Buat **base service** dengan logic shared, lalu Admin/Employee service hanya override filter:

```csharp
public abstract class BaseActiveRequestService
{
    protected abstract IQueryable<ActiveRequest> ApplyRoleFilter(IQueryable<ActiveRequest> query, string userId);
    
    public async Task<PaginationResult<ActiveRequestDto>> GetActiveRequestsAsync(...)
    {
        var query = _repo.GetAll();
        query = ApplyRoleFilter(query, userId);
        // shared pagination, mapping, etc.
    }
}

public class AdminActiveRequestService : BaseActiveRequestService
{
    protected override IQueryable<ActiveRequest> ApplyRoleFilter(IQueryable<ActiveRequest> q, string _) => q;
}

public class EmployeeActiveRequestService : BaseActiveRequestService
{
    protected override IQueryable<ActiveRequest> ApplyRoleFilter(IQueryable<ActiveRequest> q, string userId) 
        => q.Where(ar => ar.UserId == userId);
}
```

### 3.5 `UserProfileController` 100% Duplikat di Kedua Area

**File:** `Areas/Admin/Controllers/UserProfileController.cs` dan `Areas/Employee/Controllers/UserProfileController.cs` adalah **identik kecuali attribute Area dan Role**.

**Solusi:** Pindahkan ke shared controller dengan multi-role authorization:

```csharp
[Authorize(Roles = $"{AllRoles.Role_Admin},{AllRoles.Role_Employee}")]
public class UserProfileController : Controller { ... }
```

### 3.6 DIP Violation — Concrete Type Injection

**File:** `Controllers/NotificationsController.cs`

```csharp
// SEBELUM
private readonly NotificationModule _notificationModule; // concrete

// SESUDAH
private readonly INotificationModule _notificationModule; // interface
```

### 3.7 DIP Violation — Cast ke Concrete `EmailSender`

**File:** `Areas/Admin/Services/RequestAdminService.cs`

```csharp
// SEBELUM (mengalahkan tujuan DI)
string emailContent = await ((EmailSender)_emailSender).GetEmailTemplateAsync(...);

// SESUDAH — Buat interface IEmailTemplateService terpisah
public interface IEmailTemplateService
{
    Task<string> GetEmailTemplateAsync(string templateName, Dictionary<string, string> placeholders);
}
```

### 3.8 `HomeAdminService` Bypass Repository Pattern

**File:** `Areas/Admin/Services/HomeAdminService.cs`

Service ini inject `ApplicationDbContext` langsung alih-alih menggunakan repository yang sudah ada.

```csharp
// SEBELUM
private readonly ApplicationDbContext _context;

// SESUDAH — gunakan repository-repository yang sudah ada
private readonly IRequestRepository _requestRepo;
private readonly IActiveRequestRepository _activeRequestRepo;
// dst.
```

### 3.9 Cross-Area Dependencies

Admin service (`ActiveRequestService`) mengimport Employee service (`IEmployeeAuditLogService`). Ini melanggar isolasi Area.

**Solusi:** Pindahkan shared service (audit logging) ke layer bersama di luar Area.

---

## 4. P1 — Persistence Layer

### 4.1 Tidak Ada Unit of Work Pattern

Setiap `AddAsync`, `UpdateAsync`, `RemoveAsync` di base `Repository<T>` langsung call `SaveChangesAsync()`. Artinya **setiap operasi adalah transaksi terpisah** — tidak mungkin melakukan multiple update secara atomic.

**Solusi:**

```csharp
public interface IUnitOfWork : IDisposable
{
    IItemRepository Items { get; }
    IRequestRepository Requests { get; }
    IActiveRequestRepository ActiveRequests { get; }
    // ... repository lainnya
    Task<int> CommitAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

// Hapus SaveChangesAsync() dari setiap method di Repository<T>
// Biarkan service yang mengontrol kapan commit
```

### 4.2 Tidak Ada Explicit Transaction Management

**Nol** penggunaan `BeginTransactionAsync()` / `IDbContextTransaction` di seluruh codebase. Multi-step business operations (approve request → create active request → kurangi stock) masing-masing save sendiri.

### 4.3 String-Based Include (Tidak Type-Safe)

**File:** `Persistence/Repository/Repository.cs`

```csharp
// SEBELUM — typo akan gagal saat runtime
GetFirstAsync(x => x.Id == id, "Item,User,Category")

// SESUDAH — compile-time safety
GetFirstAsync(x => x.Id == id, 
    q => q.Include(x => x.Item).Include(x => x.User).Include(x => x.Category))
```

### 4.4 Repository Mengembalikan `SelectListItem` (MVC Type)

4 repository (`CategoryRepository`, `SubCategoryRepository`, `CityRepository`, `ItemTypeRepository`) return `SelectListItem`. Tipe ini milik MVC presentation layer.

**Solusi:** Repository return entity/DTO biasa, service/controller yang mapping ke `SelectListItem`.

### 4.5 Repository Return `object` (Anonymous Type)

**File:** `Persistence/Repository/SubCategoryRepository.cs` — `GetSubcategoriesByCategoryAsync`

```csharp
// SEBELUM
public async Task<object> GetSubcategoriesByCategoryAsync(...)
{
    return subcategories.Select(s => new { value = s.SubCategoryId, text = s.SubCategoryName });
}

// SESUDAH — gunakan typed DTO
public record SubCategorySelectDto(Guid Value, string Text);

public async Task<List<SubCategorySelectDto>> GetSubcategoriesByCategoryAsync(...)
```

### 4.6 `NotificationRepository` — Load All Then Update (Performa)

**File:** `Persistence/Repository/NotificationRepository.cs` — `MarkAllAsReadByUserIdAsync`

```csharp
// SEBELUM (load semua ke memory)
var notifications = await _context.Notifications
    .Where(n => n.UserId == userId && !n.IsRead).ToListAsync();
foreach (var n in notifications) n.IsRead = true;
await _context.SaveChangesAsync();

// SESUDAH (batch update tanpa load)
await _context.Notifications
    .Where(n => n.UserId == userId && !n.IsRead)
    .ExecuteUpdateAsync(s => s.SetProperty(n => n.IsRead, true));
```

### 4.7 Fake Async Methods (`Task.FromResult`)

File-file berikut menandai method sebagai `async` tapi hanya wrap `IQueryable` dengan `Task.FromResult`:

| File | Method |
|------|--------|
| `RequestRepository.cs` | `GetRequestsByStatusAsync` |
| `ItemViewRepository.cs` | `GetFilteredItemViewAsync` |

**Solusi:** Hilangkan `async`/`Task.FromResult`, kembalikan `IQueryable` langsung.

### 4.8 Business Logic di Repository

| File | Method | Logic yang Salah Tempat |
|------|--------|------------------------|
| `ItemRepository.cs` | `GetLastItemIdBySubCategoryAsync` | Parsing item code string → seharusnya di service |
| `RequestRepository.cs` | `RemoveExpiredRejectedRequestsDirectAsync` | Policy "10 hari" → seharusnya di service |
| `RequestRepository.cs` | `ApplyFilters` | Menerima `FilterModel` (DTO) → seharusnya di service |

### 4.9 Missing Database Indexes

Query yang sering dipakai tapi belum ada index:

| Column(s) | Dipakai Di |
|-----------|-----------|
| `Request.Status` + `Request.UserId` | `GetRequestsByStatusAsync`, `CheckExistingWaitingRequestAsync` |
| `Notification.UserId` + `Notification.IsRead` | `GetUnreadCountByUserIdAsync`, `MarkAllAsReadByUserIdAsync` |
| `ActiveRequest.UserId` + `ActiveRequest.Status` | `CountUserBorrowedItemsAsync` |
| `ItemPropose.UserId` + `ItemPropose.CreateAt` | `CountUserProposesOnDateAsync` |
| `Item.SubCategoryId` | `SoftDeleteItemsBySubCategoryIdAsync` |
| `Item.IsDeleted` | Hampir semua query |
| `Request.IsDeleted` | Hampir semua query |
| `Reservation.Status` | `GetWaitingApprovalAsync` |

**Solusi:** Tambahkan di `OnModelCreating` atau migration:
```csharp
entity.HasIndex(e => new { e.Status, e.UserId });
entity.HasIndex(e => e.IsDeleted).HasFilter("IsDeleted = 0"); // filtered index
```

### 4.10 Monolithic `OnModelCreating` (262 baris)

**File:** `Persistence/Data/ApplicationDbContext.cs`

**Solusi:** Pecah ke `IEntityTypeConfiguration<T>` per entity:
```csharp
public class ItemConfiguration : IEntityTypeConfiguration<Item>
{
    public void Configure(EntityTypeBuilder<Item> builder) { ... }
}

// Di OnModelCreating:
modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
```

### 4.11 Magic Numbers di Repository

**File:** `Persistence/Repository/ItemHistoryRepository.cs`

```csharp
// SEBELUM
.Where(ih => ih.ItemStatusId == 3)  // 3 = "Lost" ???
.Where(ih => ih.ItemStatusId == 1 && ih.Item.TypeId == 1)

// SESUDAH — gunakan enum
.Where(ih => ih.ItemStatusId == (int)ItemStatusEnum.Lost)
.Where(ih => ih.ItemStatusId == (int)ItemStatusEnum.WellReturned 
          && ih.Item.TypeId == (int)ItemTypeEnum.Borrowable)
```

### 4.12 `DbInitializer` — Hapus & Re-seed Setiap Startup

**File:** `Persistence/Data/DbInitializer.cs` — `RestockStatusInitialized`

Method ini menghapus _semua_ `RestockStatuses` lalu insert ulang setiap kali app start. Jika ada foreign key constraint, ini akan crash.

**Solusi:** Gunakan idempotent seeding (cek existence dulu).

---

## 5. P1 — Service Layer

### 5.1 `ex.Message` Bocor ke User

Banyak service memasukkan `ex.Message` ke response error:
```csharp
// BERBAHAYA — info internal bocor
Errors = new List<string> { ex.Message }

// SESUDAH
Errors = new List<string> { "Terjadi kesalahan internal. Silakan coba lagi." }
// Log detail error secara internal:
_logger.LogError(ex, "Error saat membuat kategori {Name}", name);
```

### 5.2 Inconsistent Error Handling Strategy

4 pattern berbeda ditemukan:
1. **Return failure result** — sebagian besar service
2. **Swallow exception silently** (empty catch) — `AuditLogAdminService`
3. **Throw generic `Exception`** — `GetEntityIdService`
4. **Redundant try-catch di controller** setelah service sudah handle

**Solusi:** Standarisasikan: service SELALU return `Result<T>`, tidak pernah throw. Controller tidak perlu try-catch.

### 5.3 Inconsistent Log Levels

```csharp
// Validation failure di-log sebagai Error (seharusnya Warning)
_logger.LogError("Can't add category, CategoryName already exists");

// Actual error di-log sebagai Warning (seharusnya Error)
_logger.LogWarning("An unexpected issue occurred while retrieving item propose data.");
```

**Panduan:**
- `LogDebug` — detail teknis untuk debugging
- `LogInformation` — operasi bisnis sukses
- `LogWarning` — validasi gagal, data tidak ditemukan
- `LogError` — exception / kegagalan sistem
- `LogCritical` — aplikasi tidak bisa lanjut

### 5.4 String Interpolation dalam Log (Structured Logging Violation)

**File:** `Areas/Admin/Services/ExcelService.cs`

```csharp
// SEBELUM
_logger.LogWarning($"Skipping row {i + 1} due to error: {rowEx.Message}");

// SESUDAH (structured logging — mendukung agregasi)
_logger.LogWarning("Skipping row {Row} due to error: {Error}", i + 1, rowEx.Message);
```

### 5.5 Async Methods Tanpa Await

| File | Method |
|------|--------|
| `ItemViewAdminService.cs` | `CheckLowStockAsync` — loop sinkron, bukan async |
| `ItemProposeAdminService.cs` | `SetupViewBag` — panggilan sinkron, bukan async |
| `ActiveRequestService.cs` | `SearchActiveRequestsAsync` — pakai `Task.FromResult` |

**Solusi:** Hapus `async`/`Task` jika method sebenarnya sinkron. Jangan gunakan `Task.FromResult` untuk "pura-pura async".

### 5.6 `CsvService` vs `ExcelService` — Duplicate Import Logic

`ImportItemsToDatabase` identik di kedua service. `CsvService` juga menduplikasi `SaveFileAsync` dari base `FileImportService`.

**Solusi:** `CsvService` harus extend `FileImportService` dan override hanya bagian parsing.

### 5.7 User Lookup Duplikat 6x di `UserViewService`

6 method (`GetUserForEditAsync`, `GetUserForDeleteAsync`, `GetActiveAsync`, dll.) melakukan pattern identik:
```csharp
var users = await _userViewRepository.GetAllAsync();
var user = users.FirstOrDefault(u => u.Id == id);
var roles = await _userManager.GetRolesAsync(user);
```

**Solusi:** Extract ke private helper method.

---

## 6. P2 — Models & DTOs

### 6.1 Namespace Chaos (5+ Namespace Berbeda untuk DTOs)

| Namespace | Contoh File |
|-----------|-------------|
| `InventoryManagementSystem.Models.DTOs` | ActiveRequest DTOs, AuditLog DTOs |
| `InventoryManagementSystem.Models.ModelsDTO` | Category DTOs, ItemView DTOs |
| `InventoryManagementSystem.DTOs` | SubCategory DTOs, Notification DTOs |
| `InventoryManagementSystem.ViewModels` | UserViewFilter |
| `InventoryManagementSystem.Services` | PaginationResult |

**Solusi:** Unifikasi ke **satu namespace**: `InventoryManagementSystem.Models.DTOs`.

### 6.2 `IFormFile` / `SelectListItem` di Domain Models

Tipe ASP.NET MVC ada di dalam entity/model:

| File | Tipe ASP.NET |
|------|-------------|
| `ActiveRequest.cs` → `ActivePictureModel` | `IFormFile` |
| `Item.cs` → `ItemPictureModel` | `IFormFile` |
| `ItemViewCreate.cs` | `IFormFile`, `IEnumerable<SelectListItem>` |
| `SubCategoryCreate.cs` | `IEnumerable<SelectListItem>` |
| `ItemViewCreateDTO.cs` | `List<SelectListItem>` × 4 |

**Solusi:** Pindahkan ke folder ViewModels terpisah. Models project tidak boleh referensi `Microsoft.AspNetCore.Mvc`.

### 6.3 ASP.NET Core 2.2 Package di Project .NET 8

**File:** `InventoryManagementSystem.Models.csproj`

```xml
<!-- SEBELUM (outdated, unsupported) -->
<PackageReference Include="Microsoft.AspNetCore.Http" Version="2.2.2" />
<PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.2.0" />

<!-- SESUDAH -->
<FrameworkReference Include="Microsoft.AspNetCore.App" />
```

### 6.4 `DateTime.Now` Hardcoded di Constructors / Properties

11 model menggunakan `DateTime.Now` di property initializer. Ini membuat unit test non-deterministik dan tidak bisa direproduksi. Satu model (`ProposeLimit`) menggunakan `DateTime.UtcNow` — inkonsisten.

**Solusi:**
```csharp
// Buat abstraksi waktu
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

// Set timestamp di service layer, bukan di model constructor
item.CreateAt = _dateTime.UtcNow;
```

### 6.5 Hardcoded GUID di Annotation

**File:** `Models/Item.cs`

```csharp
[DefaultValue("0DE4C95C-EAB5-435E-A737-DD6DA19D9AB6")]
public Guid CityId { get; set; }
```

Hapus — default value ini coupled ke seed data spesifik.

### 6.6 Duplikat DTO Classes

| Duplikat | Rekomendasi |
|----------|-------------|
| `PaginationResult<T>` vs `ItemViewPaginationResult<T>` | Gunakan satu generic class |
| `CategoryPaginationDto` vs `SubCategoryPaginationDto` | Gunakan satu generic class |
| `CategoryServiceResultDto<T>` vs `SubCategoryServiceResultDto<T>` | Gunakan shared `Result<T>` |
| `ItemHistoryDetailsDto : ItemHistoryDto` (empty body) | Hapus, gunakan `ItemHistoryDto` langsung |

### 6.7 Multiple Classes dalam Satu File

| File | Classes |
|------|:-------:|
| `AdminRequestDTO.cs` | 7 classes |
| `ActiveRequest.cs` | 2 classes (entity + view model) |
| `Item.cs` | 2 classes (entity + view model) |

**Best Practice:** Satu class per file.

### 6.8 Missing Namespace

**File:** `ModelsDTO/UserProfileDTO.cs` — tidak ada deklarasi namespace (masuk global namespace).

### 6.9 Non-Nullable Strings Tanpa Inisialisasi

Dengan `<Nullable>enable</Nullable>`, puluhan property `string` tanpa inisialisasi menghasilkan warning CS8618. Contoh: `AdminDetail`, `AuditLog`, `Category`, `User`, dll.

**Solusi:** Tambahkan `= string.Empty;` atau `= null!;` (dengan alasan jelas), atau jadikan `string?`.

### 6.10 Enum Design Issues

- `ItemTypeEnum.All = 0` — `All` adalah filter/UI concept, bukan domain value
- `ActiveRequestStatusEnum` mulai dari 0, `ItemStatusEnum` dari 1 — inkonsisten
- Status enum (`RequestStatusEnum`, `ReservationStatusEnum`) share 3 value identik → pertimbangkan shared base

---

## 7. P2 — Validators

### 7.1 Inkonsisten Aturan Password

| Validator | Min Length | Complexity |
|-----------|:---------:|:----------:|
| `RegisterInputModelValidator` | 6 | Tidak ada |
| `UserCreateValidator` | 8 | Uppercase, lowercase, angka, special char |
| `UserEditValidator` | 8 | Uppercase, lowercase, angka, special char |

**Solusi:** Buat satu shared password rule dan gunakan di semua validator.

### 7.2 Inkonsisten Validasi Nomor Telepon

- Register: `^(?:62|0)8[1-9][0-9]{6,14}$` (format Indonesia)
- User Create/Edit: `^[+]?[0-9\s\-\(\)]+$` (generic international)

Standarisasikan ke satu format.

### 7.3 `BeValidImageFile` Duplikat 3×

Method identik di-copy di 3 validator. Extract ke extension method:
```csharp
public static class FileValidationExtensions
{
    public static IRuleBuilderOptions<T, IFormFile?> MustBeValidImage<T>(
        this IRuleBuilder<T, IFormFile?> rule)
    {
        return rule.Must(BeValidImageFile).WithMessage("...");
    }
}
```

### 7.4 `RegisterInputModelValidator` Tanpa Namespace

File ini berada di global namespace, tidak konsisten dengan validator lain.

### 7.5 Business Logic di Validator

**File:** `ReservationViewCreateValidator.cs` — `.LessThanOrEqualTo(x => x.Stock)` melakukan pengecekan stock yang bisa stale. Seharusnya di service layer dengan fresh DB read.

---

## 8. P2 — AutoMapper Profiles

### 8.1 6+ Namespace Berbeda untuk File dalam Folder yang Sama

| File | Namespace |
|------|-----------|
| `Mapper.cs` | `InventoryManagementSystem.Mappers` |
| `ActiveRequestMappingProfile.cs` | `InventoryManagementSystem.Mapping` |
| `AdminRequestMappingProfile.cs` | `InventoryManagementSystem.Services.Mappings` |
| `ItemHistoryMappingProfile.cs` | `InventoryManagementSystem.Profiles` |
| `NotificationMappingProfile.cs` | `InventoryManagementSystem.Mappings` |
| `SubCategoryMappingProfile.cs` | `InventoryManagementSystem.MappingProfiles` |

**Solusi:** Unifikasi ke `InventoryManagementSystem.Mappers`.

### 8.2 `Mapper.cs` — God Profile + Mapping Duplikat

`Mapper.cs` berisi mapping untuk hampir seluruh domain (Items, Requests, Reservations, Categories, dll.), menduplikasi mapping yang sudah ada di profile terpisah. Ditemukan juga:
- `CreateMap<Item, ItemView>().ReverseMap()` muncul **2 kali**
- `CreateMap<FilterModel, FilterModel>()` muncul **2 kali**

**Solusi:** Hapus `Mapper.cs`, pindahkan mapping uniknya ke profile yang sesuai.

### 8.3 Presentation Logic di Mapping Profile

**File:** `NotificationMappingProfile.cs` — berisi method `GetTimeAgo()` dengan kalkulasi waktu relatif. Ini presentation logic yang seharusnya di helper/extension method.

### 8.4 Over-verbose `ForMember` Mappings

**File:** `ItemViewMappingProfile.cs` — 200+ baris `ForMember` untuk property yang namanya sama. AutoMapper otomatis map property dengan nama yang sama.

### 8.5 Registrasi AutoMapper Redundant di `Program.cs`

```csharp
// 10 panggilan AddAutoMapper untuk profile yang berbeda
builder.Services.AddAutoMapper(typeof(Mapper));
builder.Services.AddAutoMapper(typeof(UserViewMappingProfile));
builder.Services.AddAutoMapper(typeof(NotificationMappingProfile));
// ... dst

// SESUDAH — satu panggilan scan assembly
builder.Services.AddAutoMapper(typeof(Program).Assembly);
```

---

## 9. P2 — Controller Layer

### 9.1 Business Logic di Controller

| Controller | Logic |
|------------|-------|
| `LandingController` | Role-based redirect logic |
| `NotificationsController` | Role check untuk admin vs employee notifications |
| `UserViewController` | Manual parsing `Request.Query` alih-alih model binding |
| `RequestController` | String matching `result.Message.StartsWith("You have rejected")` |

**Solusi:** Pindahkan logic ke service layer.

### 9.2 Redundant Exception Handling

Beberapa controller wrap service call dengan try-catch meskipun service sudah handle exception secara internal. Ini bisa menyebabkan redirect loop karena `catch → RedirectToAction(Index)` → exception lagi → redirect.

### 9.3 Injected But Unused Dependencies

| Controller | Unused |
|------------|--------|
| `Admin ItemProposeController` | `IMapper _mapper` |
| `RequestEmployeeService` | `UserManager<User> _userManager` (declared tapi tidak di-assign) |

---

## 10. P2 — Utilities

### 10.1 `TimeToWib` Tidak Cross-Platform

```csharp
// SEBELUM — crash di Linux
TimeZoneInfo.FindSystemTimeZoneById("SE Asia Standard Time");

// SESUDAH — pakai TimeZoneConverter NuGet package
using TimeZoneConverter;
var wibZone = TZConvert.GetTimeZoneInfo("Asia/Jakarta");
```

### 10.2 `OperationResult` vs `ServiceResult` Redundan

Dua tipe result yang overlap:

| Property | `OperationResult` | `ServiceResult<T>` |
|----------|:-:|:-:|
| Success flag | `Success` | `IsSuccess` |
| Message | ✅ | ✅ |
| Typed data | ❌ (`Dictionary<string, object>?`) | ✅ (`T?`) |
| Error list | ❌ | ✅ |

Konsolidasikan ke `ServiceResult<T>` saja.

### 10.3 `EmailSender.GetEmailTemplateAsync` Violasi SRP

Email sending dan template rendering ada di satu class. Extract ke `IEmailTemplateService`.

### 10.4 `int.TryParse` pada Port Gagal Secara Diam-diam

```csharp
// SEBELUM — port jadi 0 jika invalid, tanpa error
int.TryParse(port, out _port);

// SESUDAH
if (!int.TryParse(port, out _port) || _port <= 0 || _port > 65535)
    throw new ArgumentException($"Invalid SMTP port: {port}");
```

---

## 11. P3 — Testing

### 11.1 Tidak Ada Test untuk Validator

10 FluentValidation validators tanpa test. Padahal ini business rules kritis.

```csharp
// Contoh test yang seharusnya ada:
[Test]
public void UserCreate_PasswordTooShort_ShouldHaveError()
{
    var validator = new UserCreateValidator();
    var dto = new UserCreateDto { Password = "Ab1!" };
    var result = validator.TestValidate(dto);
    result.ShouldHaveValidationErrorFor(x => x.Password);
}
```

### 11.2 Tidak Ada Test untuk Mapping Profile

Tambahkan `AssertConfigurationIsValid` test untuk mendeteksi mapping error lebih awal:

```csharp
[Test]
public void AutoMapperConfig_ShouldBeValid()
{
    var config = new MapperConfiguration(cfg =>
    {
        cfg.AddProfile<ItemViewMappingProfile>();
        cfg.AddProfile<RequestMappingProfile>();
        // dll.
    });
    config.AssertConfigurationIsValid();
}
```

### 11.3 Tidak Ada Test untuk ViewComponent atau Notification Hub

5 ViewComponent dan 1 SignalR Hub tanpa test.

### 11.4 Inkonsisten Test Naming

3+ naming convention dipakai:
- `Method_Condition_Expected`
- `Method_WithCondition_ShouldExpected`
- `Method_WithCondition_Expected`

**Standarisasi:** `MethodName_Condition_ExpectedBehavior` (tanpa `Should`).

### 11.5 Over-Mocking

`ItemViewAdminServiceTests` setup **18 mocks** dan verify `Times.Once` — testing implementation detail bukan behavior. Jika refactoring internal berubah, test akan pecah meskipun behavior benar.

### 11.6 Unnecessary TearDown

Banyak test class memiliki `[TearDown]` yang set fields ke null. Ini tidak diperlukan di NUnit karena `[SetUp]` sudah reinitialize.

---

## 12. P3 — Code Style & Naming

### 12.1 Inkonsisten DTO Suffix

Campuran `DTO` (uppercase) dan `Dto` (PascalCase):
- `CategoryCreateDTO`, `ItemViewDTO`, `AdminReservationDTO` — uppercase
- `AuditLogDto`, `ActiveRequestDto`, `SubCategoryDto` — PascalCase

**Best Practice C#:** Gunakan `Dto` (PascalCase) untuk akronim 3+ huruf.

### 12.2 Property camelCase di C#

Filter classes menggunakan camelCase: `filterModel`, `category`, `subCategories`. Semua public property harus PascalCase.

### 12.3 `String` vs `string`

**File:** `ItemProposeDto.cs` — menggunakan `String` (System.String) alih-alih keyword `string`.

### 12.4 Trailing Space di Nama File

**File:** `EmployeeNotificationService .cs` — ada spasi di akhir nama file.

### 12.5 Inkonsisten Action Naming

- Admin: `Rejected`, `Approved` (past tense)
- Employee: `Reject`, `Approve` (present tense)

### 12.6 Typo di Nama Method

**File:** `ItemProposeAdminService.cs`:
```
GetProposeInformatationAsync  →  GetProposeInformationAsync
```

### 12.7 Inkonsisten Formatting

- Beberapa file gunakan tab, lainnya spaces
- `IsDeleted{get; set;}` tanpa spasi vs `IsDeleted { get; set; }`
- Mix antara old-style `namespace { }` dan file-scoped `namespace;`

---

## 13. P3 — Program.cs & DI Configuration

### 13.1 Seed Data dan Interface di `Program.cs`

File `Program.cs` berisi:
- `interface IAddData` dan `class AddData` (+ seed logic)
- `void SeedDatabase()` method
- 369 baris total

**Solusi:** Pindahkan `IAddData`/`AddData` ke Persistence project. Buat extension methods untuk DI registration:

```csharp
// Di file terpisah: ServiceRegistration.cs
public static class ServiceRegistration
{
    public static IServiceCollection AddRepositories(this IServiceCollection services) { ... }
    public static IServiceCollection AddApplicationServices(this IServiceCollection services) { ... }
    public static IServiceCollection AddValidators(this IServiceCollection services) { ... }
}

// Program.cs menjadi bersih:
builder.Services.AddRepositories();
builder.Services.AddApplicationServices();
builder.Services.AddValidators();
```

### 13.2 Duplikat Registrasi DI

```csharp
// Terdaftar DUA KALI:
builder.Services.AddScoped<IRestockRepository, RestockRepository>();  // baris ~120
builder.Services.AddScoped<IRestockRepository, RestockRepository>();  // baris ~134

builder.Services.AddScoped<IRequestRepository, RequestRepository>();  // baris ~107
builder.Services.AddScoped<IRequestRepository, RequestRepository>();  // baris ~139

builder.Services.AddScoped<IRequestEmployeeRepository, RequestEmployeeRepository>(); // 2x

// Juga: AddControllers() dan AddControllersWithViews() keduanya dipanggil
builder.Services.AddControllers();           // → redundant
builder.Services.AddControllersWithViews();  // → sudah include controllers
```

### 13.3 `AddAutoMapper` Dipanggil 10x

Lihat §8.5 — cukup satu panggilan assembly scan.

### 13.4 `AddData` Constructor Menjalankan Async sebagai Fire-and-Forget

```csharp
public AddData(ApplicationDbContext db)
{
    _db = db;
    RestockData(); // async void — exception tidak ter-catch
}

public async void RestockData() { ... } // BERBAHAYA: async void
```

**Solusi:** Pindahkan seed logic ke `IDbInitializer` dan gunakan `async Task`.

### 13.5 Logging Providers Dikonfigurasi dalam Urutan Membingungkan

```csharp
// Tambah filter SEBELUM clear providers:
builder.Logging.AddFilter("System", LogLevel.Debug)
    .AddFilter("Microsoft.EntityFrameworkCore.*", LogLevel.Warning);

// Baru kemudian clear:
builder.Logging.ClearProviders();
builder.Logging.AddLog4Net();
```

`AddFilter` yang ditambahkan **sebelum** `ClearProviders()` mungkin tidak efektif.

---

## 14. Ringkasan Checklist

### P0 — Harus Segera Diperbaiki

- [ ] Pindahkan semua secret dari `appsettings.json` ke User Secrets / env var
- [ ] Tambahkan `[Authorize]` pada `NotificationsController` + null check
- [ ] Ubah email `NotifyUser` dari GET ke POST + tambah anti-forgery
- [ ] Sanitasi filename pada file upload
- [ ] Tambahkan whitelist ekstensi file + max size pada upload
- [ ] Tambahkan global `[AutoValidateAntiforgeryToken]`
- [ ] Filter `EmployeeActiveRequestService` berdasarkan UserId
- [ ] Nonaktifkan `SeedDataController` di production
- [ ] Fix `.Result` deadlock di Admin `HomeController`
- [ ] Fix generic type bug di `ActiveRequestEmployeeRepository`
- [ ] Fix N+1 query di `RequestRepository.GetRequestByItemIdWithDetailsAsync`
- [ ] Tambahkan `SaveChangesAsync()` yang hilang di 3 repository method
- [ ] Dispose `SmtpClient` di `EmailSender`
- [ ] Hapus file `client_secret_*.json` dari repository + rotasi secret

### P1 — Arsitektur (Sprint Berikutnya)

- [ ] Implementasi Unit of Work pattern
- [ ] Pecah god classes (ItemViewAdminService, ItemProposeAdminService)
- [ ] Buat base service untuk Admin/Employee duplicate logic
- [ ] Konsolidasi 6+ result type ke satu `Result<T>`
- [ ] Hapus parameter `Controller` dari service methods
- [ ] Perbaiki DIP violation (concrete NotificationModule, EmailSender cast)
- [ ] Pindahkan `SelectListItem` dari repository ke service/controller
- [ ] Tambahkan database indexes yang hilang
- [ ] Standarisasi error handling strategy

### P2 — Code Quality (Gradual)

- [ ] Unifikasi namespace DTO dan Mapper
- [ ] Pisahkan ViewModels dari Domain Models
- [ ] Update package references (ASP.NET 2.2 → FrameworkReference)
- [ ] Implementasi `IDateTimeProvider` 
- [ ] Hapus DTO duplikat
- [ ] Satu class per file
- [ ] Pecah `OnModelCreating` ke `IEntityTypeConfiguration<T>`
- [ ] Standarisasi DTO suffix ke `Dto` (PascalCase)
- [ ] Standarisasi password/phone validation rules
- [ ] Extract `BeValidImageFile` ke shared extension
- [ ] Bersihkan `Mapper.cs` (god profile + duplikat mapping)
- [ ] Ganti `AddAutoMapper()` × 10 dengan single assembly scan
- [ ] Fix structured logging violations
- [ ] Standarisasi log levels

### P3 — Nice to Have

- [ ] Tambahkan validator unit tests
- [ ] Tambahkan mapping profile `AssertConfigurationIsValid` tests
- [ ] Standarisasi test naming convention
- [ ] Pindahkan `IAddData`/`AddData` dan DI registration ke extension methods
- [ ] Fix camelCase properties, trailing space di filename, typo
- [ ] Buat `TimeToWib` cross-platform compatible
- [ ] Tambahkan `[Authorize]` di `NotificationHub`
- [ ] Hapus unused imports dan injected dependencies

---

