# Refactoring Guide — MeetingRoomBooking


---

## Daftar Isi

1. [Ringkasan Eksekutif](#1-ringkasan-eksekutif)
2. [Temuan Keamanan (KRITIS)](#2-temuan-keamanan-kritis)
3. [Bug yang Harus Diperbaiki](#3-bug-yang-harus-diperbaiki)
4. [Arsitektur & Desain](#4-arsitektur--desain)
5. [Program.cs & Dependency Injection](#5-programcs--dependency-injection)
6. [Controller Layer](#6-controller-layer)
7. [Service Layer](#7-service-layer)
8. [Database & Repository Layer](#8-database--repository-layer)
9. [Model & Validasi](#9-model--validasi)
10. [Middleware & Error Handling](#10-middleware--error-handling)
11. [Konfigurasi & DevOps](#11-konfigurasi--devops)
12. [Testing](#12-testing)
13. [Code Style & Konsistensi](#13-code-style--konsistensi)
14. [Prioritized Action Plan](#14-prioritized-action-plan)

---

## 1. Ringkasan Eksekutif

| Kategori | Jumlah Temuan |
|----------|:------------:|
| **KRITIS (Keamanan/Bug)** | 16 |
| **TINGGI (Arsitektur/Performa)** | 22 |
| **SEDANG (Kualitas Kode)** | 19 |
| **RENDAH (Style/Minor)** | 12 |
| **Total** | **69** |

**Health Score: 4/10** — Aplikasi berfungsi, namun memiliki kerentanan keamanan serius, bug kritis, masalah performa, dan inkonsistensi arsitektur yang signifikan. Diperlukan refactoring menyeluruh sebelum production deployment.

---

## 2. Temuan Keamanan (KRITIS)

### 2.1 Kredensial Hardcoded dalam Source Code

| # | File | Deskripsi | Dampak |
|---|------|-----------|--------|
| S-01 | `MeetingRoomBooking.Api/Services/EmailService.cs` | Email password hardcoded: `"uxuw alht cyoe fbll"` | Siapapun yang akses repo bisa membaca SMTP credential |
| S-02 | `MeetingRoomBooking.Api/appsettings.json` | JWT signing key dalam plain text: `"hvxDKJQqVYfyHBkA0FYl9..."` | Penyerang bisa membuat JWT token valid sendiri |
| S-03 | `MeetingRoomBooking.Api/Seed/DbSeed.cs` | Admin password hardcoded: `"Adm!nP4$$"` | Default credential yang diketahui semua developer |
| S-04 | `MeetingRoomBooking.Tests/Database/Secret/google_calendar_service_account.json` | Private key GCP service account **nyata** committed ke Git | Full akses ke Google Cloud project |

**Rekomendasi:**
```
1. SEGERA rotasi semua credential yang ter-expose
2. Pindahkan ke User Secrets (development) atau Azure Key Vault / environment variables (production)
3. Tambahkan ke .gitignore: *.json secrets/, firebase-adminsdk.json
4. Gunakan `git filter-branch` atau BFG Repo-Cleaner untuk menghapus dari Git history
5. Gunakan IOptions<T> pattern untuk semua konfigurasi sensitif
```

### 2.2 Kerentanan Token/Autentikasi

| # | File | Deskripsi |
|---|------|-----------|
| S-05 | `TokenService.cs` | Access token expire 7 hari — standar: 15-30 menit |
| S-06 | `TokenService.cs` | Refresh token hanya GUID biasa — tidak cryptographically random, tidak disimpan di DB, tidak divalidasi |
| S-07 | `TokenService.cs` | `DecodeToken()` menggunakan `ReadJwtToken()` yang **tidak memvalidasi signature** — siapapun bisa memalsukan claims |
| S-08 | `AuthService.cs` | `Console.WriteLine(JsonSerializer.Serialize(request))` di `RegisterAsync` **mungkin mencetak password** ke stdout |
| S-09 | `AuthService.cs` | Full `SigninRequest` object (termasuk password) di-log di `AttemptLoginAsync` |

**Rekomendasi:**
```csharp
// TokenService.cs — Perbaikan
public string GenerateAccessToken(User user)
{
    // Kurangi expiry menjadi 15-30 menit
    Expires = DateTime.UtcNow.AddMinutes(30), // bukan AddDays(7)
}

public string GenerateRefreshToken()
{
    // Gunakan cryptographic random bytes
    var randomBytes = RandomNumberGenerator.GetBytes(64);
    return Convert.ToBase64String(randomBytes);
    // Simpan hash-nya di database dengan expiry date
}

public ClaimsPrincipal? ValidateToken(string token)
{
    // Gunakan ValidateToken, BUKAN ReadJwtToken
    var principal = tokenHandler.ValidateToken(token, _validationParameters, out _);
    return principal;
}
```

### 2.3 User Impersonation Vulnerability

| # | File | Deskripsi |
|---|------|-----------|
| S-10 | `Models/Requests/ReservationRequest.cs` | `UserId` diterima dari request body — user bisa mengirim UserId orang lain |
| S-11 | `Models/Requests/ReportRequest.cs` | Sama — `UserId` dari request body |

**Rekomendasi:**
```csharp
// Hapus UserId dari request body
// Ambil dari authenticated claims di controller/service:
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
```

### 2.4 Information Disclosure

| # | File | Deskripsi |
|---|------|-----------|
| S-12 | `ExceptionMiddleware.cs` | Exception type name dan raw message dikirim ke client |
| S-13 | `ReservationController.cs` (UpdateReservation) | `ex.StackTrace` dan `ex.InnerException?.Message` dikirim dalam HTTP response |

**Rekomendasi:**
```csharp
// ExceptionMiddleware — jangan expose detail internal
var response = new
{
    statusCode = statusCode,
    message = environment.IsDevelopment() ? exception.Message : "An error occurred",
    // HAPUS: exceptionType, stackTrace, inner
};
```

---

## 3. Bug yang Harus Diperbaiki

### 3.1 KRITIS: UserService — Status Paksa Rejected

**File:** `MeetingRoomBooking.Api/Services/UserService.cs`

```csharp
// BUG: Ketika user mengupdate profil sendiri dan status tidak berubah,
// kode memaksa status menjadi Rejected — user langsung ter-lock!
if (isSameStatus)
{
    updatedUser.Status = ApprovalStatus.Rejected;
}
```

**Fix:** Hapus blok ini sepenuhnya. Jika status sama, pertahankan status yang ada.

### 3.2 KRITIS: RoomService — Unawaited Task

**File:** `MeetingRoomBooking.Api/Services/RoomService.cs`

```csharp
// BUG: ACL insert tasks dibuat tapi TIDAK di-await — fire-and-forget
var aclRequest1 = _calendarService.InsertAclAsync(newCalendar.Id, aclRule1);
var aclRequest2 = _calendarService.InsertAclAsync(newCalendar.Id, aclRule2);
// Error silently hilang, permission calendar tidak pernah diset
```

**Fix:**
```csharp
await Task.WhenAll(
    _calendarService.InsertAclAsync(newCalendar.Id, aclRule1),
    _calendarService.InsertAclAsync(newCalendar.Id, aclRule2)
);
```

### 3.3 TINGGI: RoomService — DateModified Sort Bug

**File:** `MeetingRoomBooking.Api/Services/RoomService.cs`

```csharp
// BUG: Sorting by DateModified menggunakan DateCreated
RoomOrderOptions.DateModified => r => r.DateCreated, // SALAH
// Fix:
RoomOrderOptions.DateModified => r => r.DateModified, // BENAR
```

### 3.4 TINGGI: ActivityRepository — Update Bug

**File:** `MeetingRoomBooking.Database/Repositories/ActivityRepository.cs`

```csharp
// BUG: Fetch existing entity (tracked), lalu Update parameter entity (detached)
// → akan throw InvalidOperationException karena double-tracking
var existing = await _db.Activities.FirstAsync(a => a.ActivityId == activity.ActivityId);
_db.Activities.Update(activity); // Ini meng-attach entity kedua dengan key yang sama!
```

**Fix:**
```csharp
var existing = await _db.Activities.FirstAsync(a => a.ActivityId == activity.ActivityId);
// Manual map properties ke tracked entity
existing.PropertyA = activity.PropertyA;
existing.PropertyB = activity.PropertyB;
await _db.SaveChangesAsync();
return existing;
```

### 3.5 TINGGI: ReservationService — Double Null Check

**File:** `MeetingRoomBooking.Api/Services/ReservationService.cs`

```csharp
if (reservation is null) { throw new KeyNotFoundException(...); }
if (reservation == null) { return null!; }  // Dead code — tidak pernah tercapai
```

### 3.6 TINGGI: AuthService — Login Priority Bug

**File:** `MeetingRoomBooking.Api/Services/AuthService.cs`

```csharp
// BUG: Username lookup MENIMPA hasil email lookup
if (!string.IsNullOrEmpty(request.Email))
    user = await FindUserByEmailAsync(request.Email);     // Dicari...
if (!string.IsNullOrEmpty(request.Username))
    user = await FindUserByUserNameAsync(request.Username); // ...lalu ditimpa!
```

**Fix:** Gunakan `else if`:
```csharp
if (!string.IsNullOrEmpty(request.Email))
    user = await FindUserByEmailAsync(request.Email);
else if (!string.IsNullOrEmpty(request.Username))
    user = await FindUserByUserNameAsync(request.Username);
```

### 3.7 SEDANG: UserController — GetMe Tanpa [Authorize]

**File:** `MeetingRoomBooking.Api/Controllers/UserController.cs`

Endpoint `GetMe()` tidak memiliki atribut `[Authorize]` — anonymous user bisa memanggilnya.

### 3.8 SEDANG: ReservationRepository — CanceledReservation Misleading

**File:** `MeetingRoomBooking.Database/Repositories/ResevartionRepository.cs`

```csharp
// Method bernama CanceledReservation tetapi mengambil reservasi ACTIVE
public async Task<List<Reservation>> CanceledReservation(Guid roomId)
{
    return await _context.Reservations
        .Where(r => r.RoomId == roomId && r.Status == ReservationStatus.Active) // Active!
        .ToListAsync();
}
```

**Fix:** Rename ke `GetActiveReservationsByRoomAsync(Guid roomId)`.

---

## 4. Arsitektur & Desain

### 4.1 God Classes (Terlalu Banyak Responsibility)

| Service | Lines | Dependencies | Rekomendasi |
|---------|------:|:----------:|------------|
| `UserService.cs` | 808 | 10+ | Pecah menjadi: `UserCrudService`, `ProfilePictureService`, `UserAuthorizationService` |
| `ReservationService.cs` | 827 | 12 | Pecah menjadi: `ReservationCrudService`, `ReservationNotificationOrchestrator`, `ReservationCalendarSyncService` |
| `EmailService.cs` | 772 | - | Pisahkan 700 baris HTML template ke file `.html` atau gunakan template engine (Scriban/RazorLight) |
| `RoomService.cs` | 604 | 11 | Pisahkan calendar sync dan notification logic |
| `AuthService.cs` | 454 | 8 | Pisahkan Google OAuth logic ke `GoogleAuthService` |

### 4.2 Entity Leaking Through Interfaces

Service interface membocorkan entity database:

```csharp
// BURUK — IReservationService
Task<List<Reservation>> ReservationOrder(...);       // Returns DB entity
Task<ReservationRequest> UpdateReservationAsync(...); // Returns request DTO (?)

// BAIK — Seharusnya
Task<List<ReservationResponse>> GetReservationsAsync(...); // Returns response DTO
Task<ReservationResponse> UpdateReservationAsync(...);     // Returns response DTO
```

**Affected:** `IReservationService`, `IAuthService` (returns `User` entity), `ActivityController` (returns `Activity` entity directly).

### 4.3 Inconsistent Error Handling Strategy

Proyek saat ini mencampur 4 pola error handling:

| Pola | Digunakan di | Masalah |
|------|------------|---------|
| Throw exceptions | `ReservationService`, `AuthService` | Controller harus catch manual |
| Return `ServiceResult<T>` | `RoomService`, `UserService`, `ChangePasswordService` | Baik — pola yang konsisten |
| Return `bool` | `NotificationService.CreateNotificationAsync` | Tidak informatif saat gagal |
| Return `string?` / `null` | `AuthService.ConfirmEmailAsync` | Ambiguous — null artinya apa? |

**Rekomendasi:** Standarisasi ke `ServiceResult<T>` di **semua** service methods. Gunakan `ExceptionMiddleware` hanya sebagai safety net, bukan flow control utama.

### 4.4 Missing Unit of Work Pattern

`RoomService` inject `MeetingRoomBookingDbContext` langsung (untuk transaction), sementara service lain tidak. Ini melanggar konsistensi arsitektur.

```csharp
// RoomService.cs — inject DbContext langsung di samping repository
private readonly MeetingRoomBookingDbContext _dbContext;
private readonly IRoomRepository _roomRepository;
// Menggunakan keduanya secara bersamaan
```

**Rekomendasi:** Implementasi `IUnitOfWork` pattern:
```csharp
public interface IUnitOfWork : IDisposable
{
    IRoomRepository Rooms { get; }
    IReservationRepository Reservations { get; }
    // ... other repos
    Task<int> CommitAsync(CancellationToken ct = default);
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

### 4.5 Concrete + Interface Double Injection

```csharp
// ReservationService.cs — inject KEDUA-DUANYA
private readonly IGoogleCalendarService _calendarService;     // Interface
private readonly GoogleCalendarService _googleCalendarService; // Concrete!
```

**Fix:** Tambahkan method yang kurang ke interface, gunakan interface saja.

---

## 5. Program.cs & Dependency Injection

### 5.1 Duplikasi Registrasi DI

```csharp
// Program.cs — service yang sama didaftarkan 2x:
builder.Services.AddScoped<IActivityRepository, ActivityRepository>();  // Line ~112
builder.Services.AddScoped<IActivityRepository, ActivityRepository>();  // Line ~143 (duplikat!)

builder.Services.AddScoped<IPasswordResetTokenRepository, PasswordResetTokenRepository>(); // Line ~126
builder.Services.AddScoped<IPasswordResetTokenRepository, PasswordResetTokenRepository>(); // Line ~145 (duplikat!)

builder.Services.AddScoped<IActivityService, ActivityService>();  // Line ~113
builder.Services.AddScoped<IActivityService, ActivityService>();  // Line ~144 (duplikat!)

builder.Services.AddScoped<IExport, ExportService>();  // Line ~113
builder.Services.AddScoped<IExport, ExportService>();  // Line ~133 (duplikat!)

// AddControllers dipanggil 2x dengan JsonOptions:
builder.Services.AddControllers().AddJsonOptions(x => ...); // Line ~48
builder.Services.AddControllers().AddJsonOptions(x => ...); // Line ~82
```

### 5.2 Commented-Out Code

~15 baris kode yang di-comment (bekas registrasi lama) masih ada:
```csharp
// builder.Services.AddScoped<IRepository<Report>, ReportRepository>();
// builder.Services.AddScoped<IReportService, ReportService>();
// builder.Services.AddScoped<IRepository<Room>, RoomRepository>();
// ... dll
```

**Fix:** Hapus semua commented-out code — gunakan Git history.

### 5.3 Scope Issue pada Data Seeding

```csharp
// Program.cs — scope tidak di-dispose
var services = app.Services.CreateScope().ServiceProvider; // ← Memory leak!
```

**Fix:**
```csharp
using var scope = app.Services.CreateScope();
var services = scope.ServiceProvider;
```

### 5.4 Tidak Ada Extension Methods untuk DI

Semua 50+ registrasi DI ada di dalam satu `Program.cs` (317 baris). 

**Rekomendasi:** Pecah ke extension methods:
```csharp
// Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRepositories(this IServiceCollection services) { ... }
    public static IServiceCollection AddBusinessServices(this IServiceCollection services) { ... }
    public static IServiceCollection AddGoogleServices(this IServiceCollection services, IConfiguration config) { ... }
    public static IServiceCollection AddAuthenticationConfig(this IServiceCollection services, IConfiguration config) { ... }
}
```

---

## 6. Controller Layer

### 6.1 Missing [ApiController] Attribute

4 controller tidak memiliki `[ApiController]`:

| Controller | Dampak |
|-----------|--------|
| `UserController` | Tidak ada automatic model validation, binding source inference |
| `NotificationController` | Sama |
| `ReportController` | Sama |
| `ActivityController` | Sama |

**P.S.** `ReservationController`, `AuthController`, `RoomController` juga tidak — seharusnya SEMUA API controller memilikinya.

**Fix:** Tambahkan `[ApiController]` di semua controller.

### 6.2 Massive Code Duplication: ResultType Switch

Kode di bawah ini di-copy-paste **30+ kali** di seluruh controller:

```csharp
return result.Type switch
{
    ResultType.BadRequest => BadRequest(new { message = result.Message }),
    ResultType.NotFound => NotFound(new { message = result.Message }),
    ResultType.Unauthorized => Unauthorized(new { message = result.Message }),
    ResultType.Forbid => Forbid(),
    ResultType.Conflict => Conflict(new { message = result.Message }),
    ResultType.InternalServerError => StatusCode(500, new { message = result.Message }),
    ResultType.StatusCode => StatusCode(result.StatusCode, new { message = result.Message }),
    _ => StatusCode(500, new { message = result.Message })
};
```

**Fix:** Buat base controller atau extension method:
```csharp
public abstract class ApiControllerBase : ControllerBase
{
    protected ActionResult HandleFailure<T>(ServiceResult<T> result) =>
        result.Type switch
        {
            ResultType.BadRequest => BadRequest(new { message = result.Message }),
            ResultType.NotFound => NotFound(new { message = result.Message }),
            ResultType.Unauthorized => Unauthorized(new { message = result.Message }),
            ResultType.Forbid => Forbid(),
            ResultType.Conflict => Conflict(new { message = result.Message }),
            _ => StatusCode(result.StatusCode > 0 ? result.StatusCode : 500, 
                    new { message = result.Message })
        };
}

// Penggunaan:
[HttpGet("{roomId}")]
public async Task<ActionResult<RoomResponse>> GetRoom(Guid roomId)
{
    var room = await _roomService.GetRoomAsync(roomId);
    return room.Success ? Ok(room.Response) : HandleFailure(room);
}
```

### 6.3 Mixed Error Handling Strategy

Beberapa controller menggunakan try-catch manual meskipun ada `ExceptionMiddleware`:

| Controller | Pola |
|-----------|-------|
| `ReservationController` | Try-catch di setiap action |
| `ActivityController` | Try-catch di setiap action |
| `RoomController` | ServiceResult pattern (tanpa try-catch) |
| `FacilityController` | ServiceResult pattern |

**Rekomendasi:** Pilih SATU:
- **Opsi A (Preferred):** Semua service return `ServiceResult<T>`, controller memeriksa `.Success`, TIDAK ADA try-catch di controller. `ExceptionMiddleware` hanya sebagai safety net.
- **Opsi B:** Semua service throw exception, `ExceptionMiddleware` menangani semua error. Controller tidak perlu error handling.

### 6.4 String Interpolation dalam Logger

```csharp
// BURUK — performansi rendah, tidak structured
_logger.LogInformation($"GET: api/reports/{reportId}");

// BAIK — structured logging
_logger.LogInformation("GET: api/reports/{ReportId}", reportId);
```

**Affected files:** `ReportController.cs`, `AuthController.cs`.

### 6.5 StackTrace Exposed dalam Response

```csharp
// ReservationController.cs — UpdateReservation
return StatusCode(500, new
{
    error = "An unexpected error occurred...",
    details = ex.Message,        // ← Detail internal exposed!
    stackTrace = ex.StackTrace,  // ← SANGAT BERBAHAYA!
    inner = ex.InnerException?.Message
});
```

**Fix:** Hapus semua detail exception dari response. Hanya log ke server.

---

## 7. Service Layer

### 7.1 Performa: Full-Table In-Memory Filtering

**File:** `MeetingRoomBooking.Api/Services/ReservationService.cs`

```csharp
// BURUK — load SELURUH tabel ke memori, lalu filter di C#
var reservations = await _reservationRepository.GetAllAsync(); // No pagination!
var query = reservations.AsQueryable(); // LINQ-to-Objects, bukan LINQ-to-SQL!
if (userId != Guid.Empty) query = query.Where(...);
if (startDate != null) query = query.Where(...);
```

**Dampak:** Jika ada 10,000 reservasi, SEMUA dimuat ke RAM setiap kali API dipanggil.

**Fix:** Push filters ke database via `IQueryable<T>`:
```csharp
// Repository
public IQueryable<Reservation> Query() => _context.Reservations.AsQueryable();

// Service
var query = _reservationRepository.Query()
    .Include(r => r.Room)
    .Include(r => r.User);

if (userId != Guid.Empty)
    query = query.Where(r => r.UserId == userId.ToString());
    
// Database menangani filtering, sorting, paging
var result = await query
    .OrderByDescending(r => r.DateCreated)
    .Skip(startIndex)
    .Take(count)
    .ToListAsync();
```

**Juga terjadi di:**
- `ActivityRepository.SearchAsync()` — load seluruh `Activities` lalu filter in-memory
- `ReservationRepository.GetAllAsync()` — tanpa pagination sama sekali

### 7.2 Missing Transaction Management

**File:** `ReservationService.cs` — `CreateReservationAsync`

```
1. Create Google Calendar event  ← Jika ini berhasil...
2. Save to database              ← ...tapi ini gagal...
3. Add attendees                 ← Calendar event yatim piatu!
```

**Fix:** Wrap dalam transaction + compensating logic:
```csharp
using var transaction = await _unitOfWork.BeginTransactionAsync();
try
{
    var calendarEvent = await _calendarService.CreateEventAsync(...);
    await _reservationRepository.AddAsync(reservation);
    await _unitOfWork.CommitAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    if (calendarEvent != null)
        await _calendarService.DeleteEventAsync(calendarEvent.Id); // Compensate
    throw;
}
```

### 7.3 Manual Token Parsing (Bypassing ASP.NET Identity)

**File:** `UserService.cs` — Dilakukan di SETIAP method yang butuh user info:

```csharp
// BERULANG 5x di UserService — seharusnya tidak ada
var bearerToken = _httpContextAccessor.HttpContext?
    .Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
var decodedToken = _tokenService.DecodeToken(bearerToken);
var userClaims = decodedToken.Claims.ToList();
var userClaimId = userClaims.FirstOrDefault(c => c.Type == "nameid")?.Value;
var userRole = userClaims.FirstOrDefault(c => c.Type == "role")?.Value;
```

**Fix:** ASP.NET Core sudah mem-parse token melalui authentication middleware:
```csharp
// Controller level sudah auto-populated:
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
var role = User.FindFirstValue(ClaimTypes.Role);

// Atau inject IHttpContextAccessor di service:
var userId = _httpContextAccessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier);
```

### 7.4 N+1 Notification Problem

**File:** `RoomService.cs` — `DeleteRoomAsync`

```csharp
// Kirim notifikasi ke SEMUA user satu-per-satu
foreach (var user in allUsers)
{
    await _notificationService.CreateNotificationAsync(...); // N database calls!
}
```

**Fix:** Buat bulk insert method:
```csharp
await _notificationService.CreateBulkNotificationsAsync(
    allUsers.Select(u => u.Id), title, message, type);
```

### 7.5 UserService — Return Type Anti-Pattern

```csharp
// BURUK — mengembalikan list of unawaited tasks!
public async Task<List<Task<UserResponse>>> GetUsersAsync(...)

// BAIK
public async Task<List<UserResponse>> GetUsersAsync(...)
{
    var tasks = users.Select(u => MapToUserResponseAsync(u));
    return (await Task.WhenAll(tasks)).ToList();
}
```

### 7.6 File System Coupling

**File:** `UserService.cs`

```csharp
// TIDAK TESTABLE — langsung mengakses file system
Directory.CreateDirectory(directoryPath);
using var stream = new FileStream(filePath, FileMode.Create);
File.Delete(filePath);
```

**Fix:** Buat abstraksi `IFileStorageService`:
```csharp
public interface IFileStorageService
{
    Task<string> SaveAsync(Stream stream, string fileName, string directory);
    Task DeleteAsync(string filePath);
    bool Exists(string filePath);
}
```

### 7.7 JsonReader — Tidak di DI, File I/O Setiap Panggilan

```csharp
// RoomService.cs — instantiate langsung, bukan dari DI
private readonly JsonReader _jsonReader = new JsonReader();

// JsonReader.cs — baca file JSON dari disk SETIAP kali dipanggil
public string? GetOwnerEmail()
{
    string json = File.ReadAllText("./Secrets/secret-key.json"); // Every. Single. Call.
    var jsonObj = JObject.Parse(json);
    return jsonObj["OwnerEmail"]?.ToString();
}
```

**Fix:** Register sebagai singleton, baca sekali saat startup:
```csharp
builder.Services.AddSingleton<ISecretConfiguration>(sp =>
{
    var json = File.ReadAllText(Path.Combine(env.ContentRootPath, "Secrets", "secret-key.json"));
    return JsonSerializer.Deserialize<SecretConfiguration>(json);
});
```

### 7.8 No CancellationToken Support

Tidak ada satu pun async method yang menerima `CancellationToken`. Operasi long-running (Google Calendar API calls, email sending, database queries) tidak bisa dibatalkan.

**Fix:** Tambahkan `CancellationToken cancellationToken = default` ke semua async method di interface dan implementation.

---

## 8. Database & Repository Layer

### 8.1 Filename Typo

File `ResevartionRepository.cs` memiliki typo: **Resevartion** → seharusnya **Reservation**.

### 8.2 IRepository<T> Tidak Pernah Digunakan

`IRepository<T>` generic interface didefinisikan tetapi **tidak ada repository yang mengimplementasinya**. Ini dead code.

**Rekomendasi:** 
- **Opsi A:** Implementasikan `BaseRepository<T>` yang mengimplementasi `IRepository<T>`, lalu semua repository inherit darinya.
- **Opsi B:** Hapus `IRepository<T>` jika tidak diperlukan.

### 8.3 Missing Database Indexes

| Entity | Kolom yang Perlu Index | Alasan |
|--------|----------------------|--------|
| `Reservation` | `UserId` | Frequently filtered by user |
| `Reservation` | `RoomId` | Frequently filtered by room |
| `Reservation` | `DateStart, DateEnd` | Range queries |
| `Reservation` | `Status` | Filtered in almost every query |
| `Activity` | `AuthorId` | Filtered in `SearchAsync` |
| `Notification` | `UserId + Status` (composite) | `GetUserNotificationsAsync`, `GetUnreadCountAsync` |
| `Report` | `IsDeleted` | Filtered in every query |
| `Room` | `IsDeleted` | Filtered in every query |

**Saat ini:** Hanya `BrowserNotificationTrigger` dan `NotificationBrowser` yang punya index.

**Fix:** Tambahkan di `OnModelCreating`:
```csharp
modelBuilder.Entity<Reservation>(entity =>
{
    entity.HasIndex(r => r.UserId);
    entity.HasIndex(r => r.RoomId);
    entity.HasIndex(r => new { r.DateStart, r.DateEnd });
    entity.HasIndex(r => r.Status);
});
```

### 8.4 Incomplete Foreign Key Configuration

```csharp
// MeetingRoomBookingDbContext.cs — FK tidak lengkap
entity.HasOne(r => r.User);  // Tanpa .WithMany() dan .HasForeignKey()
entity.HasOne(r => r.Room);  // Sama

// Activity entity — ZERO relationship config!
modelBuilder.Entity<Activity>(entity =>
{
    entity.ToTable("Activity");
    entity.HasKey(a => a.ActivityId);
    // Tidak ada FK definition untuk AuthorId, UserId, RoomId, ReservationId
});
```

**Impact:** EF Core akan membuat shadow FK dengan nama yang mungkin salah dan tanpa explicit cascade rules.

### 8.5 DateTime.Now vs DateTime.UtcNow

| Entity | Menggunakan | Seharusnya |
|--------|------------|-----------|
| `Reservation.cs` | `DateTime.Now` | `DateTime.UtcNow` |
| `Room.cs` | `DateTime.Now` | `DateTime.UtcNow` |
| `Report.cs` | `DateTime.Now` | `DateTime.UtcNow` |
| `Notification.cs` | `DateTime.Now` | `DateTime.UtcNow` |
| `Activity.cs` | `DateTime.Now` | `DateTime.UtcNow` |
| `PasswordResetToken.cs` | ✅ `DateTime.UtcNow` | Sudah benar |

**Rekomendasi:** Standarisasi ke `DateTime.UtcNow` atau `DateTimeOffset.UtcNow` di **semua** entity dan mapper.

### 8.6 Inconsistent Repository Patterns

| Aspek | Variasi yang Ditemukan |
|-------|----------------------|
| **Create method** | `AddAsync()` vs `CreateAsync()` |
| **Update signature** | `Update(entity, id)` vs `UpdateAsync(entity)` |
| **Delete signature** | `Remove(id, userId)` vs `DeleteAsync(id)` vs `DeleteAsync(entity)` vs `Delete(id)` |
| **Not-found behavior** | `throw KeyNotFoundException` vs `return null!` vs `return null` vs `return false` |
| **Context field name** | `_context` vs `_db` |
| **Exception handling** | `UserRepository` wrap semua di try-catch; yang lain tidak |

**Rekomendasi:** Definisikan konvensi repository yang konsisten:
```csharp
// Konvensi yang direkomendasikan:
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    IQueryable<T> Query();
    Task<T> CreateAsync(T entity, CancellationToken ct = default);
    Task<T> UpdateAsync(T entity, CancellationToken ct = default);
    Task<bool> DeleteAsync(Guid id, CancellationToken ct = default);
}
```

### 8.7 SaveChangesAsync Tanpa CancellationToken

```csharp
// IMeetingRoomBookingDbContext.cs
public Task<int> SaveChangesAsync(); // Tanpa CancellationToken!

// Fix:
public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
```

### 8.8 Guid vs String ID Mismatch

`User` inherit dari `IdentityUser` (string Id), tetapi repository menerima `Guid`:
```csharp
public async Task<User?> GetByIdAsync(Guid id)
{
    string idString = id.ToString(); // Konversi di setiap panggilan
    var user = await _db.Users.Where(u => u.Id == idString).FirstOrDefaultAsync();
}
```

**Rekomendasi:** Gunakan `string` secara konsisten untuk User ID di seluruh stack, atau migrasi ke `IdentityUser<Guid>`.

### 8.9 Google.Apis.Calendar.v3 di Database Project

```xml
<!-- MeetingRoomBooking.Database.csproj — SALAH -->
<PackageReference Include="Google.Apis.Calendar.v3" Version="1.66.0.3297" />
```

Google Calendar API dependency **tidak seharusnya ada** di data layer. Pindahkan ke API project.

### 8.10 ReservationAttendeeRepository — Set Navigation Property

```csharp
// BUG POTENSIAL — set navigation property dari untracked entity
attendee.Reservation = requestReservationAttendee.Reservation; // BAD
attendee.User = requestReservationAttendee.User;               // BAD
// Harus set FK property saja:
attendee.ReservationId = requestReservationAttendee.ReservationId;
attendee.UserId = requestReservationAttendee.UserId;
```

---

## 9. Model & Validasi

### 9.1 Hanya 3 dari 20 Request Model Tervalidasi

| Request Model | Punya Validator? |
|--------------|:---:|
| `RegisRequest` | ✅ |
| `ReservationRequest` | ✅ |
| `ChangePasswordRequest` | ✅ |
| `RoomRequest` | ❌ |
| `SigninRequest` | ❌ |
| `UserRequest` | ❌ |
| `ReportRequest` | ❌ |
| `FacilityRequest` | ❌ |
| `RoomFacilityRequest` | ❌ |
| `AnnouncementRequest` | ❌ |
| `MaintenanceAlertRequest` | ❌ |
| `NotificationPushRequest` | ❌ |
| `NotificationTokenRequest` | ❌ |
| `NewUserRequest` | ❌ |
| `RoleRequest` | ❌ |
| `GoogleSigninRequest` | ❌ |
| `LogoutRequest` | ❌ |
| `GetRoomsRequest` | ❌ |
| `AvailableRoomRequest` | ❌ |
| `AiChatRequest` | ❌ |

### 9.2 Mixed Validation Approach

`ReservationRequest` menggunakan `[Required]` DataAnnotations **DAN** FluentValidation secara bersamaan:
```csharp
// ReservationRequest.cs — DataAnnotations
[Required]
public string RoomName { get; set; }
[Required]
public Guid UserId { get; set; }

// ReservationRequestValidation.cs — FluentValidation
RuleFor(x => x.Title).NotEmpty().MaximumLength(100);
```

**Rekomendasi:** Pilih satu — lebih baik FluentValidation saja karena lebih fleksibel dan testable.

### 9.3 Synchronous DB Call di Validator

```csharp
// ReservationRequestValidation.cs — BLOCKING DB call di dalam Must()
.Must(emails => _userManager.Users
    .Where(u => emails.Select(email => email.ToLower()).Contains((u.Email ?? "").ToLower()))
    .Select(u => u.Email)
    .ToList()  // ← Synchronous materialization = thread pool starvation risk
    .Count == emails.Count())
```

**Fix:** Gunakan `MustAsync()`:
```csharp
.MustAsync(async (emails, ct) =>
{
    var count = await _userManager.Users
        .CountAsync(u => emails.Contains(u.Email.ToLower()), ct);
    return count == emails.Count();
})
```

### 9.4 Enum Namespace Issues

| File | Masalah |
|------|---------|
| `ResultType.cs` | Namespace `MeetingRoomBooking.Api.Services` — padahal ada di folder `Models/Enums` |
| `AiResponseType.cs` | Tidak ada namespace declaration sama sekali |

---

## 10. Middleware & Error Handling

### 10.1 ExceptionMiddleware Issues

```csharp
// 1. Double logger — injected + resolved
private readonly ILogger<ExceptionMiddleware> _logger;  // Dari constructor
var logger = context.RequestServices.GetRequiredService<ILogger<ExceptionMiddleware>>(); // Duplikat!

// 2. InvalidOperationException → 409 Conflict (SALAH)
InvalidOperationException => StatusCodes.Status409Conflict, // Seharusnya 400 atau 500

// 3. Internal details exposed
var response = new
{
    exceptionType = exception.GetType().Name, // HAPUS — information disclosure
    message = exception.Message,              // Di production, gunakan generic message
};
```

### 10.2 Redundant Manual Error Handling

Controller seperti `ReservationController` dan `ActivityController` punya try-catch di setiap action meskipun `ExceptionMiddleware` sudah ada. Ini berarti error dihandle di DUA tempat.

**Rekomendasi:** Jika menggunakan `ServiceResult<T>` pattern, controller cukup cek `.Success`. `ExceptionMiddleware` hanya menangani unexpected exception yang lolos.

---

## 11. Konfigurasi & DevOps

### 11.1 Hardcoded URLs

```csharp
// ConstantUrl.cs — COMPILE-TIME constants
public const string ApiBaseUrl = "https://localhost:7098";
public const string FrontendBaseUrl = "https://localhost:44459";
```

**Fix:** Pindahkan ke `appsettings.json`:
```json
{
  "AppUrls": {
    "Api": "https://localhost:7098",
    "Frontend": "https://localhost:44459"
  }
}
```
```csharp
builder.Services.Configure<AppUrlsOptions>(builder.Configuration.GetSection("AppUrls"));
```

### 11.2 NuGet Package Issues

| Package | Masalah | Rekomendasi |
|---------|---------|-------------|
| `FluentValidation.AspNetCore` v11.3.1 | **Deprecated** | Ganti dengan `FluentValidation.DependencyInjectionExtensions` |
| `AutoMapper` + `Mapster.DependencyInjection` | Dua mapping library sekaligus | Pilih satu (prefer Mapster — lebih performan) |
| `BCrypt.Net-Next` | Tidak perlu — Identity sudah handle hashing | Hapus jika tidak digunakan |
| `Microsoft.SemanticKernel.Connectors.Ollama` | Alpha package | Pertimbangkan stabilitas untuk production |
| `Hangfire.Storage.SQLite` v0.4.2 | Versi sangat lama, not production-ready | Upgrade atau gunakan Hangfire.InMemory |
| `itext7` | AGPL license — wajib open-source kode jika digunakan | Gunakan QuestPDF (MIT) atau PdfSharp |
| `Google.Apis.Calendar.v3` di Database project | Salah layer | Pindahkan ke API project |

### 11.3 Multiple Solution Files

```
MeetingRoomBooking.sln                     (root)
MeetingRoomBooking.Api/MeetingRoomBooking.Api.sln
MeetingRoomBooking.Tests/MeetingRoomBooking.Tests.sln
MeetingRoomBooking.Web/MeetingRoomBooking.Web.sln
```

Seharusnya hanya ada **satu** `.sln` file di root. Per-project `.sln` file menyebabkan kebingungan.

### 11.4 SQLite Database Files di Project Directory

```
MeetingRoomBooking.Api/App.db
MeetingRoomBooking.Api/App-Old-2.db
MeetingRoomBooking.Api/App-old.db
MeetingRoomBooking.Database/App.db
```

SQLite files dan backup files seharusnya ada di `.gitignore`.

### 11.5 Missing .editorconfig

Tidak ada `.editorconfig` — terbukti dari inkonsistensi:
- Tabs vs spaces (2-space di `NotificationService`, 4-space di `RoomService`)
- Inconsistent brace placement
- Inconsistent line endings

---

## 12. Testing

### 12.1 Coverage Overview

| Layer | Files Tested | Estimasi Jumlah Test |
|-------|:----------:|:-------------------:|
| Controllers | 14 | ~80 |
| Services | 19 | ~120 |
| Repositories | 12 | ~80 |
| **Total** | **45** | **~280** |

### 12.2 Zero Test Coverage

| Komponen | Risiko |
|----------|--------|
| FluentValidation validators (3 file) | Validation bugs bisa lolos |
| ExceptionMiddleware | Error handling bisa salah |
| AutoMapper/Mapster profiles | Mapping bugs bisa lolos |
| Program.cs / DI configuration | Container misconfiguration |
| DbSeed | Seeding bisa gagal tanpa diketahui |
| NotificationMessageBuilder | Message format bisa salah |
| Integration tests | Tidak ada sama sekali |
| End-to-end tests | Tidak ada sama sekali |

### 12.3 Typo di Test File

`PasswordRisetTokenServiceTestData.cs` — seharusnya "Reset", bukan "Riset".

### 12.4 Test Project References Web Unnecessarily

```xml
<!-- MeetingRoomBooking.Tests.csproj -->
<ProjectReference Include="..\MeetingRoomBooking.Web\MeetingRoomBooking.Web.csproj" />
```

Jika test hanya untuk Api dan Database, reference ke Web tidak perlu dan memperlambat build.

### 12.5 Missing Tests Recommendation

Prioritas test yang harus ditambahkan:
1. **Validator tests** — pastikan semua validation rule bekerja
2. **AutoMapper profile tests** — `mapper.ConfigurationProvider.AssertConfigurationIsValid()`
3. **Integration tests** — gunakan `WebApplicationFactory<Program>` untuk test full HTTP pipeline
4. **ExceptionMiddleware tests** — pastikan mapping exception → HTTP status correct

---

## 13. Code Style & Konsistensi

### 13.1 Naming Convention Violations

| Temuan | Lokasi |
|--------|--------|
| `isUser` (camelCase property) | `ChatMessage.cs` — seharusnya `IsUser` |
| `ResevartionRepository.cs` (typo) | Database/Repositories — seharusnya `ReservationRepository.cs` |
| `PasswordRisetTokenServiceTestData.cs` (typo) | Tests — seharusnya `PasswordResetTokenServiceTestData.cs` |
| `AutoMapperService` (bukan service) | Sebenarnya AutoMapper `Profile` — rename ke `MappingProfile` |
| `Console.WriteLine` (4 occurrences) | `AuthService.cs`, `RegisRequestValidations.cs` — hapus |

### 13.2 Mixed Language

Codebase mencampur bahasa Inggris dan Indonesia:
- Comments dalam Bahasa Indonesia di test files
- Code, variable names, dan API responses dalam Bahasa Inggris

**Rekomendasi:** Konsistenkan ke satu bahasa (Bahasa Inggris untuk code & comments).

### 13.3 Inconsistent Response Format

| Controller | Error Format |
|-----------|-------------|
| `RoomController` | `new { message = "..." }` |
| `ReservationController` | `new { error = "..." }` |
| `AuthController` | Plain string `"message"` |
| `FacilityController` | `new { message = "..." }` |

**Fix:** Standarisasi format error response:
```csharp
public record ApiError(string Message, string? Code = null);
```

---

## 14. Prioritized Action Plan

### 🔴 Phase 1: Security & Critical Bugs (1-2 minggu)

| # | Action | Effort |
|---|--------|--------|
| 1 | Rotasi semua credential yang ter-expose (email, JWT key, GCP service account) | 1 hari |
| 2 | Pindahkan secrets ke User Secrets / environment variables | 1 hari |
| 3 | Tambahkan `.gitignore` entries untuk database files, secrets, credentials | 30 menit |
| 4 | Fix TokenService — kurangi expiry, implement proper refresh token, validate signatures | 2 hari |
| 5 | Fix UserService `isSameStatus` bug (status paksa Rejected) | 30 menit |
| 6 | Fix RoomService unawaited ACL tasks | 15 menit |
| 7 | Fix RoomService DateModified sort bug | 5 menit |
| 8 | Hapus `ex.StackTrace` dari ReservationController response | 15 menit |
| 9 | Hapus `Console.WriteLine` yang mungkin print password | 15 menit |
| 10 | Fix UserId dari request body → ambil dari claims | 1 hari |

### 🟡 Phase 2: Arsitektur & Performa (2-4 minggu)

| # | Action | Effort |
|---|--------|--------|
| 11 | Standarisasi error handling ke `ServiceResult<T>` pattern | 3 hari |
| 12 | Buat `ApiControllerBase` dengan `HandleFailure()` method | 1 hari |
| 13 | Tambahkan `[ApiController]` ke semua controller | 30 menit |
| 14 | Fix ReservationService full-table loading → push filters ke DB via IQueryable | 2 hari |
| 15 | Fix ActivityRepository.SearchAsync in-memory filtering | 1 hari |
| 16 | Implementasi `IUnitOfWork` pattern | 2 hari |
| 17 | Tambahkan database indexes | 1 hari |
| 18 | Fix FK configuration di DbContext | 1 hari |
| 19 | Standarisasi `DateTime.UtcNow` di semua entity dan service | 1 hari |
| 20 | Pindahkan hardcoded URLs ke configuration | 1 hari |

### 🟢 Phase 3: Kualitas Kode (2-4 minggu)

| # | Action | Effort |
|---|--------|--------|
| 21 | Refactor `UserService` (808 lines) menjadi services yang lebih kecil | 3 hari |
| 22 | Refactor `EmailService` — pisahkan HTML template ke file terpisah | 2 hari |
| 23 | Refactor `ReservationService` (827 lines) — pisahkan notification & calendar logic | 2 hari |
| 24 | Tambahkan FluentValidation untuk 17 request model yang belum tervalidasi | 2 hari |
| 25 | Bersihkan Program.cs — hapus duplikasi DI, buat extension methods | 1 hari |
| 26 | Register `JsonReader` di DI, cache results | 1 hari |
| 27 | Standarisasi repository patterns (naming, return types, error handling) | 2 hari |
| 28 | Rename file typo: `ResevartionRepository.cs`, `PasswordRisetTokenServiceTestData.cs` | 15 menit |
| 29 | Ganti deprecated `FluentValidation.AspNetCore` | 30 menit |
| 30 | Pilih satu mapping library (AutoMapper atau Mapster) | 1 hari |
| 31 | Tambahkan `CancellationToken` ke semua async methods | 2 hari |
| 32 | Tambahkan `.editorconfig` untuk enforcing code style | 30 menit |

### 🔵 Phase 4: Testing & Polish (2-3 minggu)

| # | Action | Effort |
|---|--------|--------|
| 33 | Tambahkan unit tests untuk validators | 1 hari |
| 34 | Tambahkan unit tests untuk ExceptionMiddleware | 1 hari |
| 35 | Tambahkan AutoMapper configuration validation tests | 30 menit |
| 36 | Tambahkan integration tests dengan `WebApplicationFactory<Program>` | 3 hari |
| 37 | Pindahkan `Google.Apis.Calendar.v3` dari Database project ke API project | 30 menit |
| 38 | Hapus unused `IRepository<T>` atau implementasikan `BaseRepository<T>` | 1 hari |
| 39 | Hapus semua commented-out code | 1 hari |

