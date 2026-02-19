# Refactoring Guide — RecruitmentTracking


---

## Daftar Isi

1. [Executive Summary](#1-executive-summary)
2. [Critical Bugs yang Harus Segera Diperbaiki](#2-critical-bugs-yang-harus-segera-diperbaiki)
3. [Security Issues](#3-security-issues)
4. [Architecture & SOLID Violations](#4-architecture--solid-violations)
5. [Controllers Layer](#5-controllers-layer)
6. [Services Layer](#6-services-layer)
7. [Repository Layer](#7-repository-layer)
8. [Models & DTOs](#8-models--dtos)
9. [Data Access & EF Core](#9-data-access--ef-core)
10. [Program.cs & Configuration](#10-programcs--configuration)
11. [Validation Strategy](#11-validation-strategy)
12. [SignalR Hub](#12-signalr-hub)
13. [AutoMapper Profiles](#13-automapper-profiles)
14. [Background Services](#14-background-services)
15. [Utility Layer](#15-utility-layer)
16. [Test Project](#16-test-project)
17. [Naming Convention & Code Style](#17-naming-convention--code-style)
18. [Performance Issues](#18-performance-issues)
19. [Refactoring Priority Roadmap](#19-refactoring-priority-roadmap)

---

## 1. Executive Summary

### Statistik Temuan

| Severity | Count | Deskripsi |
|----------|-------|-----------|
| **CRITICAL** | 12 | Bug aktif, security vulnerability, data corruption risk |
| **HIGH** | 25 | Pelanggaran arsitektur besar, code duplication masif |
| **MEDIUM** | 40+ | Inconsistency, missing validation, code smell |
| **LOW** | 20+ | Naming convention, unused imports, style issues |

### Nilai Keseluruhan

| Area | Grade | Catatan |
|------|-------|---------|
| Separation of Concerns | C | Controller 18+ dependency; Program.cs monolithic |
| Consistency | D | Bahasa campuran (EN/ID), dual validation, dual mocking framework |
| Security | C | Hub vulnerability, no input length limits, hardcoded secrets |
| Testability | B- | Coverage breadth bagus, tapi pola Mock+InMemory fragile |
| Performance | C | MIMECheck rebuild setiap call, no DB indexes, sync in async |
| Maintainability | C | 8+ file tanpa namespace, 42 global usings, mapping bloat |

---

## 2. Critical Bugs yang Harus Segera Diperbaiki

### 2.1 — Program.cs: Leaked IServiceScope

**File:** `RecruitmentTracking/Program.cs`

```csharp
// ❌ SEKARANG — Scope TIDAK pernah di-dispose
IServiceProvider service = app.Services.CreateScope().ServiceProvider;

// ✅ SEHARUSNYA
using var scope = app.Services.CreateScope();
var service = scope.ServiceProvider;
```

**Impact:** `DbContext` dan scoped service lain tidak pernah dilepas dari memory → memory leak.

---

### 2.2 — Program.cs: Fire-and-Forget Seeding

**File:** `RecruitmentTracking/Program.cs`

```csharp
// ❌ SEKARANG — .Initialize() return Task tapi TIDAK di-await
dbInitializer.Initialize();

// ✅ SEHARUSNYA
await dbInitializer.Initialize();
```

**Impact:** Aplikasi mulai menerima request sebelum database selesai di-seed → `NullReferenceException` saat startup.

---

### 2.3 — DbInitializer: Synchronous Query di Async Method

**File:** `RecruitmentTracking.DataAccess/Data/DbInitializer.cs`

```csharp
// ❌ SEKARANG — .Any() synchronous di async method → thread blocking
if (!_db.Departments.Any())

// ✅ SEHARUSNYA
if (!await _db.Departments.AnyAsync())
```

---

### 2.4 — AdminController: ScheduleEndUser Memanggil Method yang Salah

**File:** `RecruitmentTracking/Controllers/AdminController.cs`

```csharp
// ❌ SEKARANG — ScheduleEndUser memanggil GetHRSchedulesAsync (copy-paste bug)
public async Task<IActionResult> ScheduleEndUser(int page = 1, int pageSize = 6)
{
    var result = await _scheduleService.GetHRSchedulesAsync(username, page, pageSize);
}

// ✅ SEHARUSNYA
var result = await _scheduleService.GetEndUserSchedulesAsync(username, page, pageSize);
```

---

### 2.5 — TalentPoolService: Notifikasi Salah

**File:** `RecruitmentTracking/Services/ATS/TalentPoolService.cs` (Line ~82)

```csharp
// ❌ SEKARANG — Message mengatakan "rejected" saat save ke talent pool
"Your application for {jobTitle} has been rejected, Rejected"

// ✅ SEHARUSNYA
"Your profile has been saved to our Talent Pool for {jobTitle}"
```

---

### 2.6 — SubscriptionBackgroundService: String Interpolation Bug

**File:** `RecruitmentTracking/Services/SubscriptionBackgroundService.cs` (Line ~214)

```csharp
// ❌ SEKARANG — Double curly braces escape interpolation, URL tidak terisi
$"{{_applicationUrl}}"  // Output literal: {_applicationUrl}

// ✅ SEHARUSNYA
$"{_applicationUrl}"
```

---

### 2.7 — HRController: ViewBag Overwrite Bug

**File:** `RecruitmentTracking/Controllers/HRController.cs` (Line ~799-800)

```csharp
// ❌ SEKARANG
ViewBag.returnURL = Request.Headers.Referer.ToString();
ViewBag.returnURL = "/Admin";  // Langsung overwrite + salah role (harusnya /HR)

// ✅ SEHARUSNYA
ViewBag.returnURL = "/HR";
```

---

### 2.8 — AiAnalysisWorker: Infinite Retry Loop

**File:** `RecruitmentTracking/Services/Background/AiAnalysisWorker.cs`

```csharp
// ❌ SEKARANG — Loop tanpa batas retry. Jika AI service mati permanen, queue terblock selamanya
while (!success)
{
    // retry with 5s delay, no max count
}

// ✅ SEHARUSNYA — Tambahkan max retry + circuit breaker
const int maxRetries = 5;
int attempt = 0;
while (!success && attempt < maxRetries)
{
    attempt++;
    // exponential backoff: delay = 5s * 2^attempt
}
```

---

### 2.9 — JobService: Copy-Paste Bug di Log Message

**File:** `RecruitmentTracking/Services/Jobs/JobService.cs` (Line ~381)

```csharp
// ❌ SEKARANG — BulkHardDeleteJobsAsync log mengatakan "soft delete"
_logger.LogWarning("No job soft deleted for IDs: {Ids}");
auditAction = "Bulk soft delete";

// ✅ SEHARUSNYA
_logger.LogWarning("No job hard deleted for IDs: {Ids}");
auditAction = "Bulk hard delete";
```

---

### 2.10 — EmailService: Property Overwrite Bug

**File:** `RecruitmentTracking/Services/Notifications/EmailService.cs` (Line ~46-47)

```csharp
// ❌ SEKARANG — HrInterviewDate di-set 2x, yang pertama langsung tertimpa
jobApplication.HrInterview.HrInterviewDate = date;
jobApplication.HrInterview.HrInterviewDate = time;  // Overwrite!
```

---

### 2.11 — Job.cs: Typo di Foreign Key Property

**File:** `RecruitmentTracking.Models/Models/Job.cs` (Line ~39)

```csharp
// ❌ SEKARANG — Typo: "EndUse" → "EndUser"
public int? EndUseStaffId { get; set; }

// ✅ SEHARUSNYA
public int? EndUserStaffId { get; set; }
```

---

### 2.12 — GenericRepository: SaveChangesAsync Tidak Dipanggil

**File:** `RecruitmentTracking/Repositories/GenericRepository.cs`

```csharp
// ❌ SEKARANG — AddAsync, DeleteAsync, UpdateAsync TIDAK memanggil SaveChangesAsync
public async Task AddAsync(T entity)
{
    await _dbSet.AddAsync(entity);
    // SaveChangesAsync() MISSING!
}

// ✅ SEHARUSNYA — Tambahkan SaveChanges, atau gunakan Unit of Work pattern
public async Task AddAsync(T entity)
{
    await _dbSet.AddAsync(entity);
    await _context.SaveChangesAsync();
}
```

---

## 3. Security Issues

### 3.1 — NotificationHub: Client Bisa Spoof Notification ke User Lain

**File:** `RecruitmentTracking/Hubs/NotificationHub.cs`

```csharp
// ❌ SEKARANG — Public hub method, client bisa kirim notifikasi ke user manapun
public async Task SendNotification(string userId, string message, ...)

// ✅ SEHARUSNYA — Hapus sebagai client-callable method
// Gunakan IHubContext<NotificationHub> di server-side service saja
// ATAU tambahkan validasi:
if (userId != Context.UserIdentifier)
    throw new HubException("Unauthorized");
```

**Impact:** User jahat bisa memanggil `SendNotification("admin_user_id", "fake message")`.

---

### 3.2 — Missing `[ValidateAntiForgeryToken]` Secara Sistematis

**Files yang terdampak:**
- `AdminController.cs` — 15+ POST actions tanpa CSRF protection
- `HRController.cs` — 15+ POST actions tanpa CSRF protection
- `EndUserController.cs` — Semua POST actions
- `NotificationApplicantController.cs` — Semua POST/DELETE actions
- `ConfigureController.cs` — Setengah dari POST actions

**Solusi:** Gunakan `[AutoValidateAntiforgeryToken]` di level global/controller:

```csharp
// Di Program.cs — apply globally
builder.Services.AddControllersWithViews(options =>
{
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

---

### 3.3 — Exception Message Exposed ke Client

**File:** `RecruitmentTracking/Controllers/HRController.cs` (Line ~242-252)

```csharp
// ❌ SEKARANG — ex.Message dikembalikan ke client
return Json(new { success = false, errorDetails = ex.Message });

// ✅ SEHARUSNYA — Generic message ke client, detail ke log
_logger.LogError(ex, "Error closing jobs");
return Json(new { success = false, message = "An error occurred." });
```

---

### 3.4 — MIMECheck: Hanya Cek Content-Type Header

**File:** `RecruitmentTracking.Utility/MIMECheck.cs`

Content-Type dari `IFormFile` bisa di-spoof oleh client. Harus **juga** verifikasi file magic bytes:

```csharp
// ✅ Tambahkan validasi magic bytes selain Content-Type
var fileBytes = await file.OpenReadStream();
var magicBytes = new byte[4];
await fileBytes.ReadAsync(magicBytes, 0, 4);
// Validate against known signatures (e.g., %PDF, PNG header, JPEG SOI)
```

---

### 3.5 — Hardcoded Secret Paths

**File:** `RecruitmentTracking/Program.cs`

```csharp
// ❌ SEKARANG
"./Secrets/google_calendar_service_account.json"  // Hardcoded path

// ✅ SEHARUSNYA
builder.Configuration["GoogleCalendar:ServiceAccountPath"]
```

---

## 4. Architecture & SOLID Violations

### 4.1 — Single Responsibility Principle (SRP)

| Class | Dependencies | Methods | Pelanggaran |
|-------|-------------|---------|-------------|
| `AdminController` | **18** DI params | **58** actions | God Controller — harus dipecah minimal 5 controller |
| `HRController` | **16** DI params | **51** actions | God Controller — 90% duplikat AdminController |
| `ConfigureController` | 7 DI params | **34** actions | Mengelola 6 entitas berbeda |
| `ApplicantService` | **21** DI params | **20** methods | God Service — gabung 6 concern area |
| `RecruitmentProcessService` | **16** DI params | **14** methods | Gabung scheduling, assignment, status transitions |

**Rekomendasi:**

```
AdminController (1164 LOC) → Pecah menjadi:
├── AdminDashboardController (Index, statistics)
├── AdminJobController (CRUD, bulk operations)
├── AdminRecruitmentController (recruitment process stages)
├── AdminScheduleController (HR/EndUser schedules)
├── AdminEmailController (templates, sending emails)
└── AdminTalentPoolController (talent pool management)

ApplicantService (698 LOC, 21 deps) → Pecah menjadi:
├── ApplicantProfileService (edit profile, get profile)
├── SavedJobService (save/delete/get saved jobs)
├── JobApplicationService (apply, track, withdraw)
├── SubscriptionService (subscribe/unsubscribe)
└── ApplicantExportService (CSV export)
```

---

### 4.2 — Open/Closed Principle (OCP)

**Masalah:** `AdminController` dan `HRController` memiliki **~90% kode identik** yang hanya berbeda di route prefix (`/Admin/` vs `/HR/`) dan role string.

**Solusi:** Extract shared logic ke base controller:

```csharp
// ✅ Base controller baru
[Authorize]
public abstract class RecruitmentBaseController : Controller
{
    protected abstract string RoleName { get; }
    protected abstract string RoutePrefix { get; }

    // Shared methods: CreateJob, EditJob, PreviewJob, RecruitmentProcess,
    // SendEmail, TalentPool, DownloadCV, ViewCV, etc.
}

[Authorize(Roles = "Admin")]
public class AdminController : RecruitmentBaseController
{
    protected override string RoleName => "Admin";
    protected override string RoutePrefix => "/Admin";
}

[Authorize(Roles = "HR")]
public class HRController : RecruitmentBaseController
{
    protected override string RoleName => "HR";
    protected override string RoutePrefix => "/HR";
}
```

**Estimasi pengurangan LOC:** ~800 baris duplikasi dihilangkan.

---

### 4.3 — Interface Segregation Principle (ISP)

| Interface | Methods | Concern Areas | Rekomendasi Pemecahan |
|-----------|---------|---------------|----------------------|
| `IApplicantService` | 16 | 6 areas | Split ke 5 interface |
| `IJobService` | 21 | 4 areas | Split ke `IJobManagementService`, `IJobQueryService`, `IJobExportService` |
| `IRecruitmentProcessService` | 14 | 4 areas | Split ke `IStatusTransitionService`, `IInterviewSchedulingService`, `IInterviewerAssignmentService` |

---

### 4.4 — Dependency Inversion Principle (DIP)

**Masalah:**
1. `ApplicantController` menggunakan Service Locator anti-pattern:
   ```csharp
   // ❌ Service Locator — langsung resolve dari RequestServices
   HttpContext.RequestServices.GetRequiredService<IWebHostEnvironment>()
   ```
2. `SubscriptionBackgroundService` langsung akses `ApplicationDbContext` tanpa repository
3. `FileService` return `IActionResult` — coupling service ke ASP.NET MVC
4. `ViewRenderService` tidak punya interface → tidak bisa di-mock

---

### 4.5 — Missing Unit of Work Pattern

Saat ini setiap repository memanggil `SaveChangesAsync()` secara individual. Ketika satu operasi bisnis melibatkan beberapa entity, tidak ada transaksi yang melindungi konsistensi data.

**Files terdampak:**
- `ApplicantService.ApplyJobAsync` — Update 4 entity tanpa transaksi
- `RecruitmentProcessService.AcceptApplicantAsync` — Update status + auto-close lain tanpa transaksi
- `TalentPoolService.SaveToTalentPoolAsync` — Update JobApplication + TalentPool + Notification tanpa transaksi

**Solusi:**

```csharp
// ✅ Tambahkan IUnitOfWork
public interface IUnitOfWork : IDisposable
{
    IJobRepository Jobs { get; }
    IApplicantRepository Applicants { get; }
    IJobApplicationRepository JobApplications { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

---

## 5. Controllers Layer

### 5.1 — Wrong Logger Type (4 Controllers)

| Controller | Sekarang | Seharusnya |
|-----------|----------|------------|
| `AdminController` | `ILogger<Home.HomeController>` | `ILogger<AdminController>` |
| `HRController` | `ILogger<Home.HomeController>` | `ILogger<HRController>` |
| `ApplicantController` | `ILogger<Home.HomeController>` | `ILogger<ApplicantController>` |
| `EndUserController` | `ILogger<Home.HomeController>` | `ILogger<EndUserController>` |

**Impact:** Semua log entry muncul di bawah kategori `HomeController` — tidak berguna untuk debugging.

---

### 5.2 — ViewBag Overuse → Ganti dengan Strongly-Typed ViewModel

```csharp
// ❌ SEKARANG — ViewBag fragile, no compile-time checking
ViewBag.Page = page;
ViewBag.PageSize = pageSize;
ViewBag.TotalPages = totalPages;
ViewBag.Departments = selectListItems.ElementAt(0);  // Index-based = fragile!
ViewBag.JobLocations = selectListItems.ElementAt(1);
ViewBag.MinEducations = selectListItems.ElementAt(2);
ViewBag.EmploymentTypes = selectListItems.ElementAt(3);

// ✅ SEHARUSNYA — Strongly-typed ViewModel
public class JobListViewModel
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public SelectList Departments { get; set; } = default!;
    public SelectList JobLocations { get; set; } = default!;
    public SelectList MinEducations { get; set; } = default!;
    public SelectList EmploymentTypes { get; set; } = default!;
    public IEnumerable<JobDTO> Jobs { get; set; } = [];
}
```

---

### 5.3 — Hardcoded Route Strings → Gunakan `nameof()` atau Constants

```csharp
// ❌ SEKARANG
return Redirect("/Admin/JobDeleted");
return Redirect("/Configure/AppUsers");
var currentUserRole = "Admin";

// ✅ SEHARUSNYA
return RedirectToAction(nameof(JobDeleted));
return RedirectToAction(nameof(AppUsers), "Configure");
var currentUserRole = RoleConstants.Admin;
```

Buat file constants:

```csharp
public static class RoleConstants
{
    public const string Admin = "Admin";
    public const string HR = "HR";
    public const string EndUser = "EndUser";
    public const string Applicant = "Applicant";
}
```

---

### 5.4 — Business Logic di Controller → Pindah ke Service

| Controller | Logic yang harus dipindah |
|-----------|-------------------------|
| `ApplicantController` | Timezone conversion (`TimeZoneInfo.ConvertTimeFromUtc`) |
| `HRController` | Dashboard statistics (Sum, Count, OrderByDescending, Take) |
| `HRController` | Inline DTO mapping (LINQ Select to JobDTO) |
| `ConfigureController` | Inline DTO mapping (Location → LocationDTO) |
| `HomeController` | 11 parameter method → bungkus dalam Query DTO |

```csharp
// ❌ SEKARANG — 11 parameter di action method
public async Task<IActionResult> Index(
    string searchString, string? chosenLocation, string? chosenCountry,
    string? chosenDepartment, List<int>? chosenLocations, List<int>? chosenCountries,
    List<int>? chosenDepartments, int page = 1, int pageSize = 8, SortBy sortBy = SortBy.Default)

// ✅ SEHARUSNYA — Single query DTO
public async Task<IActionResult> Index([FromQuery] JobSearchQueryDTO query)
```

---

### 5.5 — Null-Forgiving Operator Tanpa Null Check

**File:** `ApplicantController.cs` — 5+ lokasi

```csharp
// ❌ SEKARANG — NullReferenceException jika user tidak ditemukan
AppUser user = (await GetUser())!;

// ✅ SEHARUSNYA
var user = await GetUser();
if (user is null) return RedirectToAction("Login", "Account");
```

---

### 5.6 — Duplicated Methods Across Controllers

| Method | Duplikasi di |
|--------|-------------|
| `DownloadCV` | Admin, HR, EndUser, Applicant |
| `ViewCV` | Admin, HR, EndUser |
| `GetApplicantProfilePartial` | Admin, HR, EndUser |
| `AddInterviewNotes` | HR, EndUser |
| `ToJobViewModel` / `ToJobViewModels` | Admin, HR |

**Solusi:** Pindahkan ke shared base controller atau helper class.

---

### 5.7 — Dead Code: EndUserController.Calendar.cs

**File:** `RecruitmentTracking/Controllers/EndUser/EndUserController.Calendar.cs`

- Seluruh file adalah kode yang di-comment-out
- Class name salah: `AdminController` di namespace `EndUser`
- Berisi direct DB access (`_context.JobApplications`)

**Aksi:** Hapus file ini sepenuhnya. Version control sudah menyimpan history-nya.

---

### 5.8 — ConfigureController Harus Dipecah

Controller ini mengelola 6 entitas berbeda (Department, AppUser, Country, Education, EmploymentType, Location).

```
ConfigureController (683 LOC) → Pecah menjadi:
├── DepartmentController
├── UserManagementController
├── CountryController
├── EducationController
├── EmploymentTypeController
└── LocationController
```

---

## 6. Services Layer

### 6.1 — IApplicantService: Leaking Infrastructure Types

```csharp
// ❌ SEKARANG — Service interface tergantung pada ASP.NET Core types
Task<(bool, string?)> EditProfileAsync(
    AppUser user, UserProfileDTO profileView, IFormFile? file,
    string webRootPath,      // ← infrastructure detail
    ILogger logger,           // ← service harus punya logger sendiri
    IConfiguration config     // ← infrastructure detail
);

// ✅ SEHARUSNYA
Task<Result<UserProfileDTO>> EditProfileAsync(AppUser user, UserProfileDTO profileView, IFormFile? file);
// Inject IWebHostEnvironment dan IConfiguration via constructor
```

---

### 6.2 — Inconsistent Return Types di JobService

```csharp
// ❌ SEKARANG — Campur-campur return type
Task<JobDTO> CreateJobAsync(...)     // return null! on failure
Task<Result<JobDTO>> UpdateJobAsync(...) // proper Result pattern
Task<IEnumerable<Job>> GetAvailableJobsAsync(...)  // leak entity

// ✅ SEHARUSNYA — Konsisten pakai Result<T>
Task<Result<JobDTO>> CreateJobAsync(...)
Task<Result<JobDTO>> UpdateJobAsync(...)
Task<Result<IEnumerable<JobDTO>>> GetAvailableJobsAsync(...)
```

---

### 6.3 — Code Duplication Masif di Service Layer

| Pattern | Duplikasi | Solusi |
|---------|-----------|--------|
| HR vs EndUser scheduling | `SaveHRInterviewScheduleAsync` ≈ `SaveEndUserInterviewScheduleAsync` | Generic `SaveInterviewScheduleAsync<TInterview>` |
| HR vs EndUser assignment | `AssignHRInterviewerAsync` ≈ `AssignEndUserInterviewerAsync` | Strategy pattern atau generic method |
| HR vs EndUser email | `SendHRInterviewEmailAsync` ≈ `SendEndUserInterviewEmailAsync` | Parameterized shared method |
| HR vs EndUser schedule query | `GetHRSchedulesAsync` ≈ `GetEndUserSchedulesAsync` | Generic dengan filter parameter |

---

### 6.4 — Magic Strings untuk Status

```csharp
// ❌ SEKARANG — String scattered di banyak file
if (status == "Rejected") ...
if (status == "Accepted") ...
"Administration", "HRInterview", "UserInterview", "Offering"

// ✅ SEHARUSNYA — Gunakan enum yang sudah ada (StatusInJob)
// Dan ubah JobApplication.StatusInJob dari string? ke enum StatusInJob
public StatusInJob Status { get; set; }
```

---

### 6.5 — No Transactions untuk Multi-Entity Operations

**Critical concern:** Tidak ada `IDbContextTransaction` di manapun.

```csharp
// ❌ SEKARANG — 4 entity diupdate tanpa transaksi
await _jobApplicationRepository.AddAsync(newJobApplication);
await _jobRepository.UpdateAsync(job);
await _applicantRepository.UpdateAsync(applicant);
await _appUserRepository.UpdateAsync(user);
// Jika step ke-3 gagal, step 1-2 sudah tersimpan = data inconsistent

// ✅ SEHARUSNYA
await using var transaction = await _context.Database.BeginTransactionAsync();
try
{
    // ... all operations ...
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

### 6.6 — FileService Return IActionResult

```csharp
// ❌ SEKARANG — Service layer coupled ke MVC
public Task<IActionResult> DownloadCVAsync(...)
public Task<IActionResult> ViewCVAsync(...)

// ✅ SEHARUSNYA — Return data, controller buat IActionResult
public Task<Result<FileDownloadDTO>> DownloadCVAsync(...)
// Buat DTO:
public class FileDownloadDTO
{
    public byte[] Content { get; set; }
    public string ContentType { get; set; }
    public string FileName { get; set; }
}
```

---

### 6.7 — SubscriptionBackgroundService: Bypass Repository Layer

```csharp
// ❌ SEKARANG — Direct DbContext access di background service
var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
var activeSubscriptions = await context.JobSubscriptions.Where(...).ToListAsync();

// ✅ SEHARUSNYA — Gunakan repository
var subscriptionRepository = scope.ServiceProvider.GetRequiredService<IJobSubscriptionRepository>();
var activeSubscriptions = await subscriptionRepository.GetActiveSubscriptionsAsync();
```

Juga mengandung:
- Duplicate SMTP logic (harus pakai `IEmailService`)
- Hardcoded URL `"http://localhost:5186"`
- Hardcoded company name `"Formulatrix Recruitment"`

---

### 6.8 — TalentMatchingService: Wrong Logger Type

```csharp
// ❌ SEKARANG
ILogger<TalentPoolService> _logger  // Salah class

// ✅ SEHARUSNYA
ILogger<TalentMatchingService> _logger
```

---

### 6.9 — TalentMatchingService: Load All Data ke Memory

```csharp
// ❌ SEKARANG — Load SEMUA talent pool entries ke memory, lalu filter
var allTalents = await _talentPoolRepository.GetAllNonDeletedAsync();

// ✅ SEHARUSNYA — Filter di database level
var matchingTalents = await _talentPoolRepository
    .GetBySkillsAsync(requiredSkills, minMatchPercentage);
```

---

## 7. Repository Layer

### 7.1 — Dual Repository Layer (Redundant)

Terdapat **3 abstraksi paralel** yang membingungkan:

| Layer | Location | Status |
|-------|----------|--------|
| `DataAccess.Repository<T>` + `IRepository<T>` | `RecruitmentTracking.DataAccess/Repository/` | **TIDAK DIGUNAKAN** oleh web layer |
| `GenericRepository<T>` | `RecruitmentTracking/Repositories/` | Ada tapi **jarang dipakai** — specific repos duplicate patterns |
| Specific repositories (22) | `RecruitmentTracking/Repositories/` | Langsung akses `ApplicationDbContext` |

**Rekomendasi:**
1. Hapus `DataAccess/Repository/` sepenuhnya (dead code)
2. Semua specific repository harus extend `GenericRepository<T>`
3. Fix `GenericRepository<T>` agar panggil `SaveChangesAsync()`

---

### 7.2 — Heavy Eager Loading di JobApplicationRepository

```csharp
// ❌ SEKARANG — 12 level navigation property di-include SETIAP query
private IQueryable<JobApplication> IncludeRelatedEntities()
{
    return query
        .Include(ja => ja.Job).ThenInclude(j => j.Department)
        .Include(ja => ja.Job).ThenInclude(j => j.Location).ThenInclude(l => l.Country)
        .Include(ja => ja.Job).ThenInclude(j => j.Education)
        .Include(ja => ja.Job).ThenInclude(j => j.EmploymentType)
        .Include(ja => ja.HrInterview).ThenInclude(hi => hi.HrStaff).ThenInclude(...)
        .Include(ja => ja.EndUserInterview).ThenInclude(ei => ei.EndUserStaff).ThenInclude(...)
        .Include(ja => ja.Applicant).ThenInclude(a => a.Education)
        .Include(ja => ja.AppUser)
        .Include(ja => ja.HrInterview);  // Duplicate include!
}

// ✅ SEHARUSNYA — Projection queries / split per use-case
IQueryable<JobApplication> GetForListView()  // Minimal includes
IQueryable<JobApplication> GetForDetailView()  // Full includes
IQueryable<JobApplication> GetForStatusUpdate()  // No includes needed
```

---

### 7.3 — N+1 Queries di Bulk Operations

**File:** `JobService.cs`

```csharp
// ❌ SEKARANG — Hit DB satu per satu di loop
foreach (var id in jobIds)
{
    var job = await _jobRepository.GetByIdAsync(id);  // N queries!
    await _jobRepository.UpdateAsync(job);              // N more queries!
}

// ✅ SEHARUSNYA — Batch operation
var jobs = await _jobRepository.GetByIdsAsync(jobIds);
foreach (var job in jobs) job.IsOpen = true;
await _jobRepository.UpdateRangeAsync(jobs);
await _context.SaveChangesAsync();  // 1 save for all
```

---

### 7.4 — Inconsistent Null Return Behavior

| Repository Method | Return on Not Found |
|---|---|
| `ScheduleService.GetHRSchedulesAsync` | `new PagedResultDTO()` (safe default) |
| `ScheduleService.GetEndUserSchedulesAsync` | `null!` (crash) |
| `JobService.CreateJobAsync` | `null!` (crash) |

**Standarisasi:** Gunakan `Result<T>` pattern untuk semua method.

---

### 7.5 — DataAccess IRepository.cs: Typos di Comments

```csharp
// ❌ SEKARANG
/// AdddRange   → AddRange
/// REmove      → Remove
```

---

## 8. Models & DTOs

### 8.1 — Models Project: Wrong Package Reference

**File:** `RecruitmentTracking.Models.csproj`

```xml
<!-- ❌ SEKARANG — Test package di Models project -->
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" ... />

<!-- ✅ SEHARUSNYA — Pindahkan ke RecruitmentTracking.Test.csproj -->
```

Juga: `Google.Apis.Calendar.v3` di Models project seharusnya di service/infrastructure layer.

---

### 8.2 — Monetary Values Menggunakan `double`

**Files terdampak:** `JobApplication.cs`, ExportModels

```csharp
// ❌ SEKARANG — Floating-point imprecision untuk monetary values
public double? Salary { get; set; }
public double? CurrentSalary { get; set; }
public double? ExpectedSalary { get; set; }

// ✅ SEHARUSNYA
public decimal? Salary { get; set; }
public decimal? CurrentSalary { get; set; }
public decimal? ExpectedSalary { get; set; }
```

---

### 8.3 — StatusInJob Adalah String, Bukan Enum

```csharp
// ❌ SEKARANG — String type, enum StatusInJob tidak dipakai
public string? StatusInJob { get; set; }

// ✅ SEHARUSNYA — Gunakan enum yang sudah ada
public StatusInJob? Status { get; set; }
```

**Impact:** Enum `StatusInJob` yang sudah didefinisikan di `Enum/StatusInJob.cs` menjadi dead code.

---

### 8.4 — Contradictory Annotations

```csharp
// ❌ SEKARANG — [Required] + nullable type = kontradiksi
[Required]
public string? HrInterviewLocation { get; set; }

[Required]
public DateTime? JobPostedDate { get; set; }

[Required]
public bool IsOpen { get; set; }  // Value type selalu required

// ✅ SEHARUSNYA
[Required, StringLength(200)]
public string HrInterviewLocation { get; set; } = string.Empty;

[Required]
public DateTime JobPostedDate { get; set; }

public bool IsOpen { get; set; }  // Remove [Required]
```

---

### 8.5 — Missing `[StringLength]` Secara Sistematis

Hampir **semua** string property di semua model tidak memiliki `[StringLength]` attribute, artinya:
- Database kolom akan menjadi `nvarchar(MAX)` (SQLite: TEXT tanpa limit)
- Tidak ada validasi panjang di server-side

**Contoh yang harus ditambahkan:**

```csharp
[Required, StringLength(200)]
public string JobTitle { get; set; } = string.Empty;

[StringLength(5000)]
public string? Description { get; set; }

[StringLength(20)]
public string? Phone { get; set; }
```

---

### 8.6 — Missing Namespace pada 8+ Files

| File | Namespace yang Hilang |
|------|----------------------|
| `JobRecommendation.cs` | Global namespace (harus `RecruitmentTracking.Models`) |
| `SortBy.cs` | Global namespace |
| `StatusInJob.cs` | Global namespace |
| `AiFlexibleIntConverter.cs` | Global namespace |
| `AtsRequestValidator.cs` | Global namespace |
| `ExtractCvTextValidator.cs` | Global namespace |
| `JobRecommendationValidator.cs` | Global namespace |
| Beberapa DTOs | Global namespace |
| Mapping profiles | Global namespace |

**Aksi:** Tambahkan `namespace` declaration yang sesuai ke semua file.

---

### 8.7 — CS8618: Non-Nullable Without Initializer

```csharp
// ❌ SEKARANG — CS8618 warnings
public string Message { get; set; }           // NotificationApplicant
public string Keywords { get; set; }          // JobSubscription
public string UnsubscribeToken { get; set; }  // JobSubscription

// ✅ SEHARUSNYA
public required string Message { get; set; }
// ATAU
public string Message { get; set; } = string.Empty;
```

---

### 8.8 — Code Duplication: HrInterview ≈ EndUserInterview, HrStaff ≈ EndUserStaff

```csharp
// ✅ SOLUSI — Base class dengan inheritance
public abstract class InterviewBase
{
    public DateTime? InterviewDate { get; set; }
    public DateTime? InterviewTime { get; set; }
    [Required, StringLength(200)]
    public string InterviewLocation { get; set; } = string.Empty;
    public bool IsOnline { get; set; }
    public string? MeetingUrl { get; set; }
    public string? Notes { get; set; }
}

public class HrInterview : InterviewBase { /* HR-specific fields */ }
public class EndUserInterview : InterviewBase { /* EndUser-specific fields */ }
```

---

### 8.9 — Behavior di Entity Model

**File:** `Location.cs`

```csharp
// ❌ SEKARANG — Method di entity model (seharusnya anemic domain)
public string GetLocationName() => $"{CityName}, {Country!.CountryName!}";
public string GetLocationCountry() => $"{Country!.CountryName!}";
// Null-forgiving operator AKAN crash jika Country belum di-include

// ✅ SEHARUSNYA — Pindahkan ke DTO atau mapping
public class LocationDTO
{
    public string FullName => $"{CityName}, {CountryName}";
}
```

---

### 8.10 — DTOs dengan Domain Entity References

```csharp
// ❌ SEKARANG — Entity model di dalam DTO
public class JobDTO
{
    public ICollection<Job> SavedJobs { get; set; }  // ← ENTITY!
}

// ✅ SEHARUSNYA
public class JobDTO
{
    public ICollection<SavedJobDTO> SavedJobs { get; set; }
}
```

---

### 8.11 — `[DefaultValue]` Attribute Tidak Berfungsi di EF Core

```csharp
// ❌ SEKARANG — Tidak ada efek runtime
[DefaultValue(false)]
public bool IsOpen { get; set; }

// ✅ SEHARUSNYA — Gunakan property initializer atau Fluent API
public bool IsOpen { get; set; } = false;
// ATAU di OnModelCreating:
entity.Property(j => j.IsOpen).HasDefaultValue(false);
```

---

### 8.12 — Inconsistent DateTime Strategy

| File | Pattern |
|------|---------|
| `NotificationApplicant.cs` | `DateTime.UtcNow.ToLocalTime()` |
| `JobRecommendation.cs` | `DateTime.Now` |
| `JobService.cs` | `DateTime.UtcNow` (untuk satu field), `DateTime.Now` (untuk perbandingan) |
| `NotificationApplicantService.cs` | `DateTime.UtcNow.ToLocalTime()` |

**Standarisasi:** Gunakan `DateTime.UtcNow` di semua tempat. Konversi timezone dilakukan di presentation layer.

---

### 8.13 — Enum Issues

**ProcessType.cs:** Gap di nilai enum (5 → 7, missing 6) tanpa dokumentasi

**SortBy.cs, StatusInJob.cs:** Tidak punya explicit integer values — fragile jika di-persist ke DB lalu enum di-reorder.

```csharp
// ✅ SEHARUSNYA — Explicit values untuk semua persisted enums
public enum StatusInJob
{
    Administration = 1,
    HRInterview = 2,
    EndUserInterview = 3,
    Offering = 4,
    Accepted = 5,
    // 6 = Reserved
    Rejected = 7,
    Withdrawn = 8
}
```

---

## 9. Data Access & EF Core

### 9.1 — ApplicationDbContext: Nullable DbSet

```csharp
// ❌ SEKARANG — Satu-satunya nullable DbSet
public DbSet<JobApplication>? JobApplications { get; set; }

// ✅ SEHARUSNYA — Konsisten non-nullable
public DbSet<JobApplication> JobApplications { get; set; }
```

---

### 9.2 — No Delete Behavior Configuration

EF Core default ke `Cascade` delete untuk required relationships. Ini berbahaya:
`Job` dihapus → semua `JobApplication` ikut terhapus → semua `HrInterview` dan `EndUserInterview` juga hilang.

```csharp
// ✅ Tambahkan di OnModelCreating
modelBuilder.Entity<JobApplication>()
    .HasOne(ja => ja.Job)
    .WithMany(j => j.JobApplications)
    .OnDelete(DeleteBehavior.Restrict);
```

---

### 9.3 — No Database Indexes

Tidak ada index yang dikonfigurasi. Kolom yang sering di-query/join harus diindex:

```csharp
// ✅ Tambahkan indexes
modelBuilder.Entity<JobApplication>()
    .HasIndex(ja => ja.ApplicantId);

modelBuilder.Entity<JobApplication>()
    .HasIndex(ja => ja.JobId);

modelBuilder.Entity<Job>()
    .HasIndex(j => j.IsOpen);

modelBuilder.Entity<Job>()
    .HasIndex(j => j.IsDeleted);
```

---

### 9.4 — DbInitializer: Seeder Order Dependency

`LocationSeeder` berjalan sebelum `CountrySeeder`. Jika Location memiliki FK ke Country, ini akan gagal di fresh database.

**Aksi:** Verifikasi dan urutkan seeder sesuai dependency graph.

---

## 10. Program.cs & Configuration

### 10.1 — Monolithic DI Registration (~160 Lines)

```csharp
// ❌ SEKARANG — Semua registrasi di satu file
builder.Services.AddScoped<IJobRepository, JobRepository>();
builder.Services.AddScoped<IApplicantRepository, ApplicantRepository>();
// ... 40+ registrasi lainnya ...

// ✅ SEHARUSNYA — Extension methods terpisah
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRepositories(this IServiceCollection services)
    {
        services.AddScoped<IJobRepository, JobRepository>();
        services.AddScoped<IApplicantRepository, ApplicantRepository>();
        // ...
        return services;
    }

    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<IJobService, JobService>();
        // ...
        return services;
    }
}

// Di Program.cs:
builder.Services.AddRepositories();
builder.Services.AddApplicationServices();
builder.Services.AddValidators();
```

---

### 10.2 — Google/Facebook Auth: No Null Check

```csharp
// ❌ SEKARANG — Jika config section tidak ada, ClientId = null
options.ClientId = config["Authentication:Google"]["ClientId"];

// ✅ SEHARUSNYA
var googleConfig = config.GetRequiredSection("Authentication:Google");
options.ClientId = googleConfig["ClientId"]
    ?? throw new InvalidOperationException("Google ClientId is not configured");
```

---

### 10.3 — CalendarService: Hardcoded Path

```csharp
// ❌ SEKARANG
GoogleCredential.FromFile("./Secrets/google_calendar_service_account.json")

// ✅ SEHARUSNYA
var path = builder.Configuration["GoogleCalendar:ServiceAccountPath"]
    ?? throw new InvalidOperationException("Google Calendar service account path not configured");
GoogleCredential.FromFile(path)
```

---

### 10.4 — Mixed Language Comments

```csharp
// ❌ SEKARANG
// Registrasi layanan email
// Konfigurasi MailSettings

// ✅ SEHARUSNYA — Standarisasi ke English
// Register email services
// Configure MailSettings
```

---

### 10.5 — Middleware Ordering

```csharp
// ✅ Recommended ordering:
app.UseHttpsRedirection();
app.UseStatusCodePagesWithRedirects("/Error/{0}");  // Before UseStaticFiles
app.UseStaticFiles();
app.UseCookiePolicy();  // After UseRouting
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
```

---

## 11. Validation Strategy

### Masalah Utama: Dual Validation System

Project menggunakan **BOTH** Data Annotations dan FluentValidation secara bersamaan:
- `CreateJobDTO` → `[Required]` Data Annotations + `CreateJobValidator` FluentValidation
- Sebagian besar model → Data Annotations saja (tanpa FluentValidation)

**Rekomendasi:** Pilih **satu** strategy dan konsisten:

| Opsi | Pro | Con |
|------|-----|-----|
| FluentValidation saja | Clean separation, testable, complex rules | Perlu buat validator untuk semua DTOs |
| Data Annotations saja | Built-in, simple | Sulit untuk complex cross-field validation |
| **Recommended:** FluentValidation | Best practice untuk MVC/API | Migration effort sedang |

### Validators: Missing MaxLength

Semua validator tidak memiliki `MaximumLength` rules — user bisa submit megabytes of text.

### Validators: Missing Namespace

3 dari 4 validator di global namespace.

### Validators: Mixed Language Error Messages

```csharp
// ❌ SEKARANG — Bahasa campuran
"Summary Message is empty!"                                    // English
"List Recommended Jobs tidak boleh null."                      // Indonesian
"Item Job di dalam list tidak boleh null."                     // Indonesian

// ✅ SEHARUSNYA — Konsisten satu bahasa, idealnya English
"Summary message is required."
"Recommended jobs list cannot be null."
"Job items in the list cannot be null."
```

---

## 12. SignalR Hub

### NotificationHub Issues

1. **Security:** `SendNotification()` bisa dipanggil client → spoof risk (lihat Section 3.1)
2. **Server timezone:** `DateTime.UtcNow.ToLocalTime()` → kirim UTC, convert di client
3. **No authentication check:** Hub class tidak validate `Context.User`

---

## 13. AutoMapper Profiles

### 13.1 — Presentation Logic di Mapping Profile

```csharp
// ❌ SEKARANG — HTML conversion di AutoMapper
.ForMember(dest => dest.Requirement,
    opt => opt.MapFrom(src => src.JobRequirement.Replace("\r\n", "<br/>")));

// ✅ SEHARUSNYA — Ini adalah view concern
// Gunakan HTML helper atau custom tag helper di Razor view
@Html.Raw(Model.Requirement.Replace("\r\n", "<br/>"))
```

---

### 13.2 — Redundant Explicit Mappings

```csharp
// ❌ SEKARANG — AutoMapper sudah handle same-name properties by convention
.ForMember(dest => dest.JobTitle, opt => opt.MapFrom(src => src.JobTitle))  // Redundant!

// ✅ SEHARUSNYA — Hapus explicit mapping untuk same-name properties
// Cukup configure yang BEDA saja
```

---

### 13.3 — 25+ Property Ignores di JobApplicationProfile

```csharp
// ❌ SEKARANG — Manual ignore list yang harus di-maintain
.ForMember(dest => dest.Property1, opt => opt.Ignore())
.ForMember(dest => dest.Property2, opt => opt.Ignore())
// ... 25x ...

// ✅ ALTERNATIVE — Gunakan ReverseMap() atau dedicated request DTO yang lebih kecil
```

---

### 13.4 — Missing Namespace di Mapping Profiles

Semua mapping profile files tidak memiliki namespace declaration.

---

## 14. Background Services

### 14.1 — AiAnalysisWorker: Infinite Retry

Lihat Section 2.8. Tambahkan:
- Max retry count (e.g., 5)
- Exponential backoff (5s, 10s, 20s, 40s, 80s)
- Dead-letter queue / logging untuk items yang gagal setelah max retries
- Circuit breaker pattern

---

### 14.2 — BackgroundTaskQueue: Unbounded Channel

```csharp
// ❌ SEKARANG — Memori bisa tumbuh tanpa batas
Channel.CreateUnbounded<Guid>()

// ✅ SEHARUSNYA — Bounded channel dengan backpressure
Channel.CreateBounded<Guid>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});
```

---

### 14.3 — SubscriptionBackgroundService: Duplicate SMTP Logic

Service ini mengimplementasikan SMTP sending sendiri alih-alih menggunakan `IEmailService` yang sudah ada. Ini berarti:
- Perubahan di email config harus diupdate di 2 tempat
- Email formatting tidak konsisten
- Tidak ada retry/error handling yang sama

---

## 15. Utility Layer

### 15.1 — MIMECheck: ContentInspector Rebuild Every Call

```csharp
// ❌ SEKARANG — Mahal, dipanggil setiap file upload
public static bool IsFileValid(IFormFile file, ...)
{
    var inspector = new ContentInspectorBuilder() { ... }.Build();  // Rebuild setiap kali!
}

// ✅ SEHARUSNYA
private static readonly IContentInspector _inspector = new ContentInspectorBuilder()
{
    // ...
}.Build();
```

---

### 15.2 — MIMECheck: No Bounds Check

```csharp
// ❌ SEKARANG — IndexOutOfRangeException jika file unrecognizable
result.ByMimeType()[0].MimeType

// ✅ SEHARUSNYA
var mimeResults = result.ByMimeType();
if (mimeResults.Count == 0) return false;
var detectedMime = mimeResults[0].MimeType;
```

---

### 15.3 — MIMECheck: Static Tanpa Interface

```csharp
// ❌ SEKARANG — Tidak bisa di-mock untuk testing
public static class MIMECheck

// ✅ SEHARUSNYA — Injectable service
public interface IMimeTypeValidator
{
    bool IsFileValid(IFormFile file, string[] allowedMimeTypes, long maxSize);
}

public class MimeTypeValidator : IMimeTypeValidator { ... }
```

---

### 15.4 — MailSettings: Wrong Namespace

```csharp
// ❌ SEKARANG — Di project Utility tapi namespace Models
namespace RecruitmentTracking.Models;

// ✅ SEHARUSNYA
namespace RecruitmentTracking.Utility;
```

---

### 15.5 — Utility csproj: Obsolete Package

```xml
<!-- ❌ SEKARANG — Package v2.2.2 untuk .NET 8 app -->
<PackageReference Include="Microsoft.AspNetCore.Http" Version="2.2.2" />

<!-- ✅ SEHARUSNYA -->
<FrameworkReference Include="Microsoft.AspNetCore.App" />
```

---

## 16. Test Project

### 16.1 — Dual Mocking Framework

```xml
<!-- ❌ SEKARANG — Kedua framework direferensikan -->
<PackageReference Include="Moq" Version="4.20.72" />
<PackageReference Include="FakeItEasy" Version="8.1.0" />

<!-- ✅ SEHARUSNYA — Pilih satu, hapus yang lain -->
```

---

### 16.2 — Unnecessary SqlServer Package

```xml
<!-- ❌ SEKARANG — App pakai SQLite, test project punya SqlServer -->
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />

<!-- ✅ SEHARUSNYA — Hapus, cukup InMemory untuk testing -->
```

---

### 16.3 — EF Package Version Mismatch

```xml
<!-- ❌ SEKARANG — Mixed versions -->
InMemory: 8.0.2
Sqlite: 8.0.0
Design: 8.0.0

<!-- ✅ SEHARUSNYA — Samakan semua ke versi terbaru -->
```

---

### 16.4 — 42 Global Using Statements

**File:** `Usings.cs`

Terlalu banyak global usings. Hanya masukkan yang benar-benar dipakai di >50% test files.

```csharp
// ❌ SEKARANG — 42 global usings termasuk:
global using NUnit.Framework.Internal;   // Internal namespace
global using NUnit.Framework.Legacy;      // Legacy assertions
global using MimeKit;                     // Hanya dipakai di email tests

// ✅ SEHARUSNYA — ~10-15 essential global usings
global using NUnit.Framework;
global using Moq;  // (setelah standarisasi)
global using RecruitmentTracking.Models;
global using RecruitmentTracking.Models.DTOs;
// Import spesifik di file yang membutuhkan
```

---

### 16.5 — Mock+InMemory Hybrid Pattern (Fragile)

**File:** `JobRepositoryTest.cs`

```csharp
// ❌ SEKARANG — Mock<DbContext> wrapping InMemory database + CallBase = true
var mockContext = new Mock<ApplicationDbContext>(options) { CallBase = true };
// SaveChangesAsync verification vs actual InMemory execution = unpredictable

// ✅ SEHARUSNYA — Pilih satu:
// Option A: Pure InMemory (integration test)
var context = new ApplicationDbContext(options);
// Option B: Pure Mock (unit test)
var mockContext = new Mock<ApplicationDbContext>();
mockContext.Setup(x => x.Jobs).Returns(mockDbSet.Object);
```

---

### 16.6 — Variable Typos di Tests

```csharp
// ❌ SEKARANG
_departements  // → _departments
_countrys      // → _countries
```

---

### 16.7 — Missing Negative/Edge Case Tests

Berdasarkan test files yang di-review:
- Tests fokus pada happy path
- Kurang test untuk: null inputs, concurrent access, boundary values, error conditions
- Tidak ada integration tests

---

## 17. Naming Convention & Code Style

### 17.1 — C# Naming Convention Violations

| Sekarang | Seharusnya | File |
|----------|------------|------|
| `EndUseStaffId` | `EndUserStaffId` | `Job.cs` |
| `EmailTemplateID` | `EmailTemplateId` | `EmailTemplate.cs` |
| `HRInterviewStaffID` | `HrInterviewStaffId` | `JobApplicationCSV.cs` |
| `AppliedJobsIDSSV` | `AppliedJobsIdSsv` | `ApplicantCSV.cs` |
| `MIMECheck` | `MimeCheck` | `MIMECheck.cs` |
| `Guid Id` (PascalCase param) | `Guid id` | `EndUserController.cs` |

---

### 17.2 — File/Class Name Mismatch

| File | Class di Dalamnya |
|------|-------------------|
| `NotificationApplicantController.cs` | `NotificationController` |
| `ApplicantCSV.cs` | `ApplicantCSVExport` + `ApplicantCSVExporter` (2 classes) |

---

### 17.3 — Unused Using Directives

| File | Unused Using |
|------|-------------|
| `Applicant.cs` | `System.Collections` |
| `JobApplication.cs` | `System.Collections` |
| `AppUser.cs` | `System.ComponentModel.DataAnnotations` |
| `ConfigureController.cs` | `Microsoft.CodeAnalysis.CSharp.Syntax` (Roslyn!) |
| `ScheduleService.cs` | `System.Security.Claims` |
| `HomeController.cs` | `RecruitmentTracking.Data`, `RecruitmentTracking.Interfaces` |

---

### 17.4 — Mixed Language (English/Indonesian)

| Lokasi | Contoh |
|--------|--------|
| Program.cs comments | "Registrasi layanan email" |
| Validator error messages | "tidak boleh null" |
| Some DTO property names | Mostly English (acceptable) |

**Standarisasi:** Semua code comments, error messages, dan documentation dalam **English**.

---

## 18. Performance Issues

| Issue | File | Impact | Fix |
|-------|------|--------|-----|
| MIMECheck rebuild inspector per-call | `MIMECheck.cs` | CPU spike on file upload | Static readonly field |
| No DB indexes | `ApplicationDbContext.cs` | Slow queries on large data | Add indexes on FK columns |
| Sync `.Any()` in async method | `DbInitializer.cs` | Thread blocking | Use `AnyAsync()` |
| Load all entities to memory | `TalentMatchingService.cs` | Memory spike, slow | Database-level filtering |
| N+1 queries in bulk ops | `JobService.cs` | N+1 DB calls per bulk | Batch `GetByIdsAsync` |
| 12-level eager loading | `JobApplicationRepository.cs` | Heavy SQL queries | Projection / split |
| New SmtpClient per email | `EmailService.cs` | SMTP connection overhead | Connection pooling |
| Unbounded Channel | `BackgroundTaskQueue.cs` | Unbounded memory growth | `CreateBounded<>` |

---

## 19. Refactoring Priority Roadmap

### Phase 1: Critical Bug Fixes (1-2 hari)

| # | Task | Impact |
|---|------|--------|
| 1 | Fix leaked `IServiceScope` di Program.cs | Memory leak |
| 2 | Await `dbInitializer.Initialize()` | Startup crash risk |
| 3 | Fix `.Any()` → `.AnyAsync()` di DbInitializer | Thread blocking |
| 4 | Fix `ScheduleEndUser` memanggil method yang salah | Wrong data displayed |
| 5 | Fix TalentPool notification message | Wrong user notification |
| 6 | Fix string interpolation bug di SubscriptionBackgroundService | Broken email URLs |
| 7 | Fix ViewBag overwrite bug di HRController | Wrong redirect |
| 8 | Fix log message di BulkHardDelete | Misleading logs |
| 9 | Fix EmailService property overwrite | Wrong interview dates |
| 10 | Fix `EndUseStaffId` typo | Potential FK issue |
| 11 | Fix GenericRepository missing SaveChanges | Silent data loss |
| 12 | Add max retry ke AiAnalysisWorker | Queue blocking |

---

### Phase 2: Security Hardening (2-3 hari)

| # | Task |
|---|------|
| 1 | Secure NotificationHub (remove client-callable `SendNotification`) |
| 2 | Add `[AutoValidateAntiforgeryToken]` globally |
| 3 | Remove `ex.Message` from client responses |
| 4 | Add MIMECheck magic bytes validation |
| 5 | Move secrets to User Secrets / environment variables |
| 6 | Add null checks pada authentication config |
| 7 | Fix MIMECheck `IndexOutOfRangeException` |

---

### Phase 3: Architecture Refactoring (1-2 minggu)

| # | Task | LOC Saved |
|---|------|-----------|
| 1 | Extract `RecruitmentBaseController` (merge Admin + HR duplication) | ~800 LOC |
| 2 | Split `ConfigureController` → 6 controllers | Better SRP |
| 3 | Split `ApplicantService` (21 deps → 5 focused services) | Better testability |
| 4 | Split `RecruitmentProcessService` (16 deps → 3 services) | Better SRP |
| 5 | Implement Unit of Work pattern | Data consistency |
| 6 | Standardize return type ke `Result<T>` | Consistent error handling |
| 7 | Remove DataAccess `Repository<T>` (dead code) | Less confusion |
| 8 | Make all specific repos extend `GenericRepository<T>` | DRY |
| 9 | Split fat interfaces (ISP compliance) | Better DI |
| 10 | Refactor `FileService` — return data, not `IActionResult` | Clean architecture |

---

### Phase 4: Code Quality (3-5 hari)

| # | Task |
|---|------|
| 1 | Fix all wrong logger types (4 controllers + 1 service) |
| 2 | Replace `ViewBag` with strongly-typed ViewModels |
| 3 | Replace hardcoded strings with constants (`RoleConstants`, `StatusConstants`) |
| 4 | Add `[StringLength]` ke semua model string properties |
| 5 | Fix contradictory annotations (`[Required]` on nullable) |
| 6 | Change `double` → `decimal` untuk monetary values |
| 7 | Use `StatusInJob` enum instead of string |
| 8 | Add namespaces ke semua files yang missing |
| 9 | Fix nullable reference type warnings (CS8618) |
| 10 | Standardize DateTime strategy (UTC everywhere) |
| 11 | Delete dead code (EndUserController.Calendar.cs, commented code) |
| 12 | Remove unused `using` directives |
| 13 | Extract DI registration ke extension methods |
| 14 | Standardize error messages ke English |

---

### Phase 5: Performance & Testing (3-5 hari)

| # | Task |
|---|------|
| 1 | Cache `ContentInspector` di MIMECheck (static readonly) |
| 2 | Add database indexes |
| 3 | Fix JobApplicationRepository eager loading → projection |
| 4 | Fix N+1 queries di bulk operations |
| 5 | Fix TalentMatchingService → DB-level filtering |
| 6 | Bounded channel di BackgroundTaskQueue |
| 7 | Standarisasi mocking framework (Moq ATAU FakeItEasy) |
| 8 | Remove SqlServer package dari test project |
| 9 | Align EF package versions |
| 10 | Trim global usings ke essential ~15 |
| 11 | Fix Mock+InMemory hybrid pattern |
| 12 | Add negative/edge case tests |

---

### Phase 6: Validation & Mapping Cleanup (2-3 hari)

| # | Task |
|---|------|
| 1 | Pilih satu: FluentValidation atau DataAnnotations (recommended: FluentValidation) |
| 2 | Add MaximumLength rules ke semua validators |
| 3 | Add namespaces ke validator files |
| 4 | Remove HTML conversion dari AutoMapper profiles |
| 5 | Remove redundant same-name ForMember mappings |
| 6 | Simplify 25-property ignore blocks |
| 7 | Buat `MimeTypeValidator` sebagai injectable service |
| 8 | Fix `MailSettings` namespace |
| 9 | Remove obsolete `Microsoft.AspNetCore.Http` v2.2.2 |
| 10 | Remove `Microsoft.AspNetCore.Mvc.Testing` dari Models csproj |

---

## Total Estimasi: ~4-5 minggu kerja

| Phase | Estimasi | Priority |
|-------|----------|----------|
| Phase 1: Critical Bug Fixes | 1-2 hari | **P0 — Segera** |
| Phase 2: Security Hardening | 2-3 hari | **P0 — Segera** |
| Phase 3: Architecture Refactoring | 1-2 minggu | **P1 — Sprint berikutnya** |
| Phase 4: Code Quality | 3-5 hari | **P1 — Sprint berikutnya** |
| Phase 5: Performance & Testing | 3-5 hari | **P2 — Planned** |
| Phase 6: Validation & Mapping | 2-3 hari | **P2 — Planned** |

---


