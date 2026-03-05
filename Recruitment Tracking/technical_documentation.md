# Recruitment Tracking — Technical Documentation (Per-Module)

> **Purpose**: This document provides a comprehensive, per-module technical breakdown of the Recruitment Tracking system. It is designed for new programmers to understand the architecture, data flow, and implementation details of every module.

---

## Table of Contents

1. [Solution Architecture](#1-solution-architecture)
2. [Module 1 — RecruitmentTracking.Models (Domain Layer)](#2-module-1--recruitmenttrackingmodels-domain-layer)
3. [Module 2 — RecruitmentTracking.DataAccess (Data Access Layer)](#3-module-2--recruitmenttrackingdataaccess-data-access-layer)
4. [Module 3 — RecruitmentTracking.Utility (Shared Utilities)](#4-module-3--recruitmenttrackingutility-shared-utilities)
5. [Module 4 — RecruitmentTracking (Main Web Application)](#5-module-4--recruitmenttracking-main-web-application)
   - [4a. Program.cs & DI Configuration](#5a-programcs--dependency-injection-configuration)
   - [4b. Controllers Module](#5b-controllers-module)
   - [4c. Services Module](#5c-services-module)
   - [4d. Repositories Module](#5d-repositories-module)
   - [4e. AutoMapper Profiles](#5e-automapper-profiles)
   - [4f. FluentValidation Validators](#5f-fluentvalidation-validators)
   - [4g. SignalR Hubs](#5g-signalr-hubs-module)
   - [4h. Identity & Authentication](#5h-identity--authentication-module)
   - [4i. Views & Frontend](#5i-views--frontend-module)
6. [Module 5 — RecruitmentTracking.Test (Test Project)](#6-module-5--recruitmenttrackingtest-test-project)
7. [Cross-Cutting Concerns](#7-cross-cutting-concerns)
8. [Request Lifecycle (End-to-End Flow)](#8-request-lifecycle-end-to-end-flow)

---

## 1. Solution Architecture

```
RecruitmentTracking.sln
│
├── RecruitmentTracking/                    ← Main MVC Web Application (Presentation + Business Logic)
├── RecruitmentTracking.DataAccess/         ← EF Core DbContext, Migrations, Seeders, Generic Repository
├── RecruitmentTracking.Models/             ← Entity Models, DTOs, Enums, Export Models, JSON Converters
├── RecruitmentTracking.Utility/            ← Mail settings model, MIME type validation
└── RecruitmentTracking.Test/               ← Unit & integration tests (NUnit, Moq, FakeItEasy)
```

### Dependency Flow

```
RecruitmentTracking (Web App)
    ├── → RecruitmentTracking.DataAccess
    ├── → RecruitmentTracking.Models
    └── → RecruitmentTracking.Utility

RecruitmentTracking.DataAccess
    └── → RecruitmentTracking.Models

RecruitmentTracking.Test
    └── → References all projects above
```

### Layered Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                         │
│  Controllers (8) → Razor Views → wwwroot (JS/CSS/Images)     │
│  Areas/Identity (Login, Register, Manage)                     │
├──────────────────────────────────────────────────────────────┤
│                    BUSINESS LOGIC LAYER                        │
│  Services/ (8 sub-folders, 40+ service classes)               │
│  Validator/ (4 FluentValidation validators)                   │
│  Mapping/ (7 AutoMapper profile classes)                      │
│  Hubs/ (SignalR NotificationHub)                              │
├──────────────────────────────────────────────────────────────┤
│                    DATA ACCESS LAYER                           │
│  Repositories/ (22 domain-specific + generic repositories)    │
│  RecruitmentTracking.DataAccess (DbContext, Migrations)       │
├──────────────────────────────────────────────────────────────┤
│                    DOMAIN LAYER                                │
│  RecruitmentTracking.Models (20 entities, 39 DTOs, 3 enums)  │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Module 1 — RecruitmentTracking.Models (Domain Layer)

**Project**: `RecruitmentTracking.Models.csproj`
**Purpose**: Contains all domain entities, DTOs, enums, export models, and JSON converters. This is a **dependency-free** project that all other projects reference.

### 2.1 Entity Models (`Models/` — 20 files)

Each entity maps to a database table via EF Core.

#### Core Entities

| Entity | Primary Key | Properties | Relationships |
|--------|-------------|------------|---------------|
| `AppUser` | `string Id` (Identity) | `Name`, `CreateDate` | Extends `IdentityUser`. Referenced by `Applicant`, `HrStaff`, `EndUserStaff`, `JobApplication`. |
| `Applicant` | `Guid ApplicantId` | `Phone`, `Major`, `GPA`, `ProfileImage`, `EducationId` | FK → `AppUser`, FK → `Education`. Has many `JobApplications`, `SavedJobs`. |
| `Job` | `Guid JobId` | `JobTitle`, `Description`, `Requirement`, `IsOpen`, `IsDraft`, `IsDeleted`, `ApplicantCount`, `Major`, `JobPostedDate`, `JobExpiredDate` | FK → `Department`, `Location`, `Education`, `EmploymentType`, `HrStaff`, `EndUserStaff`. Has many `JobApplications`. |
| `JobApplication` | `Guid JobApplicationId` | `CV`, `Salary`, `StatusInJob`, `IsWithdraw`, `EmailSent`, `AtsScore`, `AtsSummary`, `AtsPros`, `AtsCons`, `AtsMatchedSkills`, `AtsMissingSkills`, `ExtractedCvText`, `CreatedAt`, `UpdatedAt`, `CurrentSalary`, `ExpectedSalary`, `HrInterviewNotes`, `EndUserInterviewNotes` | FK → `Job`, `Applicant`, `AppUser`, `HrInterview`, `EndUserInterview`. |

#### Interview & Staff Entities

| Entity | Primary Key | Properties | Relationships |
|--------|-------------|------------|---------------|
| `HrStaff` | `Guid HrStaffId` | — | FK → `AppUser`. Has many `Jobs`, `HrInterviews`. |
| `EndUserStaff` | `Guid EndUserStaffId` | — | FK → `AppUser`. Has many `Jobs`, `EndUserInterviews`. |
| `HrInterview` | `Guid HrInterviewId` | `HrInterviewDate`, `HrInterviewLocation`, `MeetingLink`, `IsOnline` | FK → `HrStaff`. |
| `EndUserInterview` | `Guid EndUserInterviewId` | `EndUserInterviewDate`, `EndUserInterviewLocation`, `MeetingLink`, `IsOnline` | FK → `EndUserStaff`. |

#### Reference Data Entities

| Entity | Primary Key | Purpose |
|--------|-------------|---------|
| `Department` | `int DepartmentId` | Department name. Has many Jobs. |
| `Location` | `int LocationId` | City name. FK → `Country`. Has many Jobs. |
| `Country` | `int CountryId` | Country name. Has many Locations. |
| `Education` | `int EducationId` | Education level (name, description, level number). |
| `EmploymentType` | `int EmploymentTypeId` | Employment type name (Full-time, Part-time, etc.). |

#### Feature Entities

| Entity | Primary Key | Purpose |
|--------|-------------|---------|
| `SavedJob` | `Guid SavedJobId` | Bookmarked job by applicant. Links `Job` ↔ `Applicant`. |
| `JobSubscription` | `Guid SubscriptionId` | Applicant's job alert preferences: keywords, departments, locations, unsubscribe token. |
| `JobRecommendation` | `Guid JobRecommendationId` | AI-generated job recommendations. Stores recommended job IDs as JSON. |
| `NotificationApplicant` | `Guid NotificationId` | In-app notification with message, type, read status. FK → `AppUser`, `JobApplication`. |
| `TalentPool` | `Guid TalentPoolId` | Saved applicant for future opportunities. FK → `JobApplication`. Includes admin notes and cached skills. |
| `AuditTrail` | `Guid ActivityId` | Activity log: user name, role, action, job, location, timestamp. |
| `EmailTemplate` | `Guid EmailTemplateID` | Customizable email templates for each recruitment stage. |

### 2.2 Enumerations (`Enum/` — 3 files)

#### `ProcessType` — Recruitment Pipeline Stages
```csharp
public enum ProcessType
{
    Administration = 1,
    HRInterview = 2,
    UserInterview = 3,
    Offering = 4,
    Accepted = 5,
    Rejected = 7,
    Withdraw = 8,
    TalentPool = 9,
    Sourced = 10,
    Closed = 11,
}
```
**Usage**: Used in controllers and services to determine which pipeline stage view to render and what transitions are allowed.

#### `StatusInJob` — Current Application Status
```csharp
public enum StatusInJob
{
    Administration,
    HRInterview,
    UserInterview,
    Offering,
    Rejected,
    TalentPool,
    Sourced,
    Closed
}
```
**Usage**: Stored as a string in `JobApplication.StatusInJob`. Compared via `.ToString()` in service layer queries.

#### `SortBy` — Job Listing Sort Options
```csharp
public enum SortBy
{
    Default,
    JobTitleDesc, JobTitleAsc,
    JobPostedDateAsc, JobPostedDateDesc,
    JobExpiredDateAsc, JobExpiredDateDesc
}
```
**Usage**: Passed from the Home page to `JobSearchService` for sorting job listings.

### 2.3 DTOs (`DTOs/` — 39 files)

DTOs are used to transfer data between layers. Key categories:

| Category | DTOs | Purpose |
|----------|------|---------|
| **Job Management** | `JobDTO`, `CreateJobDTO`, `JobSearchDTO`, `JobNotificationEmailDTO` | Job display, creation, search results |
| **Applicant** | `ApplicantProfileDTO`, `ApplicantApplicationDTO`, `ApplicantApplicationStatusDTO`, `ApplicantActionDTO`, `UserProfileDTO` | Profile display, application tracking |
| **ATS / AI** | `AtsRequestDTO`, `AtsResponseDTO`, `ExtractCvTextDTO`, `AiJobRecommendationDTO`, `JobRecommendationDTO` | AI analysis request/response |
| **Interview** | `InterviewerScheduleDTO`, `AddNotesDTO` | Schedule display, interview notes |
| **Configuration** | `DepartmentDTO`, `CountryDTO`, `LocationDTO`, `EducationDTO`, `EmploymentTypeDTO`, `AppUsersDTO`, `UserRoleDTO` + delete DTOs | CRUD for reference data |
| **Talent Pool** | `TalentPoolDTO`, `AdministrationTalentPoolDTO`, `SaveToTalentPoolDTO` | Talent pool management |
| **Notification** | `EmailTemplateDTO` | Email template management |
| **Infrastructure** | `PagedResultDTO<T>`, `ErrorDTO`, `SavedJobsDTO`, `SubscriptionPreferencesDTO`, `AuditTrailDTO` | Pagination, errors, bookmarks |
| **Export** | `JobCSV`, `JobApplicationCSV`, `ApplicantCSV` (in `ExportModels/`) | CSV export models |

### 2.4 JSON Converters (`Converters/` — 1 file)

Custom `JsonConverter` implementations for handling special serialization scenarios (e.g., enum-to-string conversion).

---

## 3. Module 2 — RecruitmentTracking.DataAccess (Data Access Layer)

**Project**: `RecruitmentTracking.DataAccess.csproj`
**Purpose**: Contains the EF Core `DbContext`, database migrations, seed data, and the generic base repository.

### 3.1 ApplicationDbContext (`Data/ApplicationDbContext.cs`)

Inherits from `IdentityDbContext<AppUser>` to integrate ASP.NET Core Identity.

**20 DbSet Properties**:
```
Jobs, JobRecommendations, JobApplications, Applicants,
Departments, EmploymentTypes, Locations, Countries, Educations,
SavedJobs, EndUserStaffs, HrStaffs, EmailTemplates, AuditTrails,
EndUserInterviews, HrInterviews, JobSubscriptions,
NotificationApplicants, TalentPools
```

**`OnModelCreating` Relationships Configured**:
- `Applicant` → many `SavedJobs`, many `JobApplications`
- `Department` → many `Jobs`
- `EmploymentType` → many `Jobs`
- `HrStaff` → many `Jobs`, many `HrInterviews`
- `EndUserStaff` → many `Jobs`, many `EndUserInterviews`
- `Job` → many `JobApplications`
- `Location` → many `Jobs`
- `Country` → many `Locations`
- `JobApplication` → one `HrInterview`, one `EndUserInterview`
- `NotificationApplicant` → one `AppUser`, one `JobApplication`
- `JobRecommendation` → one `Applicant`

### 3.2 DbInitializer (`Data/DbInitializer.cs`)

Implements `IDbInitializer`. Called at startup from `Program.cs` via `SeedDataBase()`.

**Seeding Order** (conditional — only seeds if table is empty):
1. `DepartmentSeeder` → Departments
2. `EducationSeeder` → Education levels
3. `EmploymentTypeSeeder` → Employment types
4. `LocationSeeder` → Locations
5. `CountrySeeder` → Countries
6. `HumanResourceSeeder` → HR staff records
7. `EndUserSeeder` → EndUser staff records
8. `JobSeeder` → Sample job postings
9. `EmailTemplateSeeder` → Default email templates
10. `JobRecommendationSeeder` → Sample AI recommendations
11. `JobApplicationSeeder` → Sample job applications

### 3.3 Generic Repository (`Repository/`)

| Interface/Class | Purpose |
|-----------------|---------|
| `IRepository<T>` | Base CRUD interface: `GetAll`, `GetById`, `Add`, `Update`, `Delete` |
| `Repository<T>` | Base implementation using `ApplicationDbContext` |

These are extended by domain-specific repositories in the main web application project.

---

## 4. Module 3 — RecruitmentTracking.Utility (Shared Utilities)

**Project**: `RecruitmentTracking.Utility.csproj`
**Purpose**: Small utility library shared across projects.

| File | Purpose |
|------|---------|
| `MailSettings.cs` | POCO model for SMTP configuration (bound from `appsettings.json` section `MailSettings`). Properties: `Server`, `Port`, `SenderName`, `SenderEmail`, `UserName`, `Password`. |
| `MIMECheck.cs` | Static helper for MIME type validation during file uploads. Used by `FileService` to validate CV and profile image uploads. |

---

## 5. Module 4 — RecruitmentTracking (Main Web Application)

**Project**: `RecruitmentTracking.csproj`
**Purpose**: The main ASP.NET Core 8.0 MVC application containing all presentation logic, business logic, and domain-specific repositories.

### 5a. Program.cs & Dependency Injection Configuration

`Program.cs` (228 lines) is the application entry point. Here is the complete startup flow:

#### 1. Service Registration Order

```
1. AddControllersWithViews()
2. Database: SQLite via EF Core
3. Identity: AddDefaultIdentity<AppUser>() with role management
4. Cookie configuration (SameSite=Lax, HttpOnly, Secure)
5. OAuth: Google + Facebook authentication
6. MailSettings configuration from appsettings.json
7. AutoMapper (assembly scan)
8. FluentValidation (auto-validation)
9. Hosted Services: SubscriptionBackgroundService, AiAnalysisWorker
10. ViewRenderService
11. SignalR
12. Generic Repository: IGenericRepository<> → GenericRepository<>
13. All 22 domain repositories (Scoped)
14. All 30+ services (Scoped)
15. Background task queue (Singleton): IBackgroundTaskQueue → BackgroundTaskQueue
```

#### 2. Middleware Pipeline

```
1. ExceptionHandler / HSTS (production)
2. UseStaticFiles
3. UseRouting
4. UseCookiePolicy
5. UseAuthentication
6. UseAuthorization
7. MapControllerRoute (default: {controller=Home}/{action=Index}/{id?})
8. MapHub<NotificationHub>("/notificationHub")
9. MapRazorPages
10. SeedDataBase() → DbInitializer.Initialize()
```

#### 3. Database Seeding at Startup

```csharp
void SeedDataBase()
{
    using (var scope = app.Services.CreateScope())
    {
        var dbInitializer = scope.ServiceProvider.GetService<IDbInitializer>();
        dbInitializer.Initialize();
    }
}
```

Identity roles and users are seeded separately via:
- `SeedUserRole.CreateRoles(service, config)` — Creates 4 roles: Admin, Applicant, EndUser, HR
- `SeedAppUser.Seed(service, config)` — Creates default users for each role

---

### 5b. Controllers Module

Located in `RecruitmentTracking/Controllers/`. 8 controllers with 200+ combined actions.

#### HomeController (3 actions — Public)

**Namespace**: `RecruitmentTracking.Controllers.Home`
**Authorization**: None (public-facing)
**Dependencies**: `IJobSearchService`, `IJobService`, `IReferenceDataService`, `IApplicantService`, `UserManager<AppUser>`

| Action | HTTP | Purpose |
|--------|------|---------|
| `Index` | GET | Public job listing page with multi-filter search (department, location, country), sorting (`SortBy` enum), and pagination. Loads saved job IDs for logged-in applicants. |
| `DetailJob(Guid id)` | GET `/DetailJob/{id}` | Job detail view. Returns 404 if job not found. |
| `Error` | GET | Error page with request ID. |

**Data Flow**: `HomeController.Index` → `JobSearchService.GetJobSearchViewModelWithFilterWithPaginationAsync()` → Returns `JobSearchDTO` to view.

---

#### AdminController (62 actions — Admin only)

**Namespace**: `RecruitmentTracking.Controllers.Admin`
**Authorization**: `[Authorize(Roles = "Admin")]`
**Dependencies**: 15+ injected services including `IJobService`, `IJobSearchService`, `IRecruitmentProcessService`, `IEmailService`, `IEmailTemplateService`, `ITalentPoolService`, `ITalentMatchingService`, `IAuditService`, `ICsvService`, `IFileService`, `IScheduleService`, `IReferenceDataService`, `IApplicantProfileService`, `INotificationApplicantService`

**Action Groups**:

| Group | Actions | Purpose |
|-------|---------|---------|
| **Dashboard** | `Index` | Job statistics: active, closed, drafted, deleted counts. |
| **Job CRUD** | `CreateJob` (GET/POST), `EditJob` (GET/POST), `PreviewJob`, `JobHistory` | Full job lifecycle management. |
| **Job Status** | `ActivateJob`, `BulkActivateJobs`, `CloseSelectedJobs`, `SoftDeleteJob`, `BulkSoftDeleteJobs`, `HardDeleteJob`, `RestoreJob`, `BulkDeleteJobs` | Bulk and individual job status management. |
| **Job Lists** | `JobClosed` (2 overloads), `JobDeleted`, `JobDrafted` | Filtered job listing views with search and pagination. |
| **CSV Export** | `GetJobsCSV`, `GetJobApplicationsCSV`, `GetApplicantsCSV`, `GetAuditTrailsCSV` | Data export to CSV files. |
| **Recruitment Pipeline** | `Administration`, `HRInterview`, `EndUserInterview`, `Offering` | View applicants at each pipeline stage. |
| **Interview Management** | `ScheduleHR`, `ScheduleEndUser`, `SaveHRInterview`, `SaveEndUserInterview`, `AssignHRInterviewer`, `AssignEndUserInterviewer` | Interview scheduling and interviewer assignment. |
| **Applicant Actions** | `Accept`, `Rejected`, `AddToTalentPool`, `ViewCV`, `DownloadCV`, `DownloadCVOnSchedule`, `GetApplicantProfilePartial` | Manage applicant lifecycle. |
| **Email Templates** | `TemplateEmail`, `SaveRejectionEmail`, `SaveHREmail`, `SaveInterviewHR`, `SaveEndUserEmail`, `SaveInterviewEndUser`, `SaveOfferEmail` | Create and edit email templates. |
| **Email Sending** | `SendHRInterview`, `SendEndUserInterview`, `SendOffering`, `SendEmailRejection` | Send stage-specific emails. |
| **Talent Pool** | `SaveToTalentPool`, `TalentPool`, `DeleteTalent`, `TalentMatchingJobs`, `ImportFromTalentPool` | Talent pool management with AI matching. |
| **Audit Trail** | `Activity`, `GetAllAuditTrails` | View activity logs. |

---

#### HRController (56 actions — HR only)

**Namespace**: `RecruitmentTracking.Controllers.HR`
**Authorization**: `[Authorize(Roles = "HR")]`

Nearly identical to `AdminController` with these differences:
- **No** `HardDeleteJob` action (HR cannot permanently delete jobs)
- **No** user management capabilities
- **Additional**: `UpdateTalentNotes`, `ImportCandidate`, `AddInterviewNotes`
- **Additional**: `ToJobViewModels` helper for multiple job view model mapping
- Uses the same services as AdminController

---

#### ApplicantController (22 actions — Applicant only)

**Namespace**: `RecruitmentTracking.Controllers.Applicants`
**Authorization**: `[Authorize(Roles = "Applicant")]`
**Dependencies**: `IApplicantService`, `IJobService`, `IJobRecommendationService`, `IReferenceDataService`, `UserManager<AppUser>`

| Action | HTTP | Purpose |
|--------|------|---------|
| `Profile` | GET | View applicant profile with education levels and application history. |
| `EditProfile` | GET/POST | Edit profile (phone, major, GPA, education, image). |
| `ApplyJob(Guid id)` | GET | Show job application form. |
| `ApplyJobs` | POST | Submit job application with CV upload. Triggers ATS analysis. |
| `WithdrawApplyJob` | POST | Withdraw a submitted application. |
| `DownloadCV` | POST | Download own CV. |
| `TrackJob` | GET | Track application status with search and pagination. |
| `SavedJobs` | GET | View bookmarked jobs. |
| `SaveJob` | POST | Bookmark a job. |
| `DeleteSavedJob` | POST | Remove a bookmark. |
| `RecommendJobs` | GET | Get AI-powered job recommendations. |
| `ManagePreferences` | GET | View subscription preferences. |
| `UpdatePreferences` | POST | Update subscription preferences (keywords, departments, locations). |
| `Subscribe` | POST | Subscribe to job alerts. |
| `Unsubscribe` | POST | Unsubscribe from job alerts. |
| `UnsubscribeByToken` | GET | Unsubscribe via email token link. |

**Key Flow — Job Application**:
```
ApplicantController.ApplyJobs (POST)
  → ApplicantService.ApplyJobAsync()
    → Validates eligibility (profile complete, not already applied)
    → FileService.SaveCVFileAsync() — saves PDF to wwwroot/uploads/cv/
    → PdfProcessingService.ExtractTextAsync() — extracts text from CV
    → Creates JobApplication with status "Administration"
    → BackgroundTaskQueue.QueueAsync() — queues ATS analysis
    → NotificationApplicantService.CreateNotificationAsync() — push notification
```

---

#### EndUserController (13 actions — EndUser only)

**Namespace**: `RecruitmentTracking.Controllers.EndUser`
**Authorization**: `[Authorize(Roles = "EndUser")]`
**Dependencies**: `IJobSearchService`, `IJobService`, `IScheduleService`, `IFileService`, `IReferenceDataService`, `IRecruitmentProcessService`, `IApplicantProfileService`

| Action | HTTP | Purpose |
|--------|------|---------|
| `Index` | GET | View active jobs assigned to this end user with search and pagination. |
| `JobClosed` | GET | View closed jobs. |
| `ScheduleEndUser` | GET | View assigned interview schedules (filters by logged-in user). |
| `PreviewJob` | GET | Job detail preview. |
| `JobHistory` | GET | View applicants for a specific job. |
| `DownloadCV` | POST | Download applicant CV. |
| `GetApplicantProfilePartial` | GET | Load applicant profile as partial view (AJAX). |
| `ViewCV` | GET | View CV inline as PDF. |
| `AddInterviewNotes` | POST (JSON) | Save interview notes for an applicant. |
| `GetAll` | GET | API endpoint: returns all job applications for a job as JSON. |

---

#### ConfigureController (38 actions — Admin only)

**Namespace**: `RecruitmentTracking.Controllers.Configure`
**Authorization**: `[Authorize(Roles = "Admin")]`
**Purpose**: CRUD operations for all reference data entities.

| Entity | Actions |
|--------|---------|
| **Departments** | `Index` (list all), `CreateDepartment`, `EditDepartment` (GET/POST), `DeleteDepartment` (GET/POST) |
| **App Users** | `AppUsers`, `CreateAppUser`, `EditAppUser` (GET/POST), `DeleteAppUser` (GET/POST) |
| **Countries** | `Countries`, `CreateCountry`, `EditCountry` (GET/POST), `DeleteCountry` (GET/POST) |
| **Education** | `MinEducations`, `CreateMinEducation`, `EditMinEducation`, `DeleteMinEducation` |
| **Employment Types** | `EmploymentTypes`, `CreateEmploymentType`, `EditEmploymentType` (GET/POST), `DeleteEmploymentType` (GET/POST) |
| **Locations** | `Locations`, `CreateLocation`, `EditLocation` (GET/POST), `DeleteLocation` (GET/POST) |

**Dependency Checks**: Delete actions validate that no jobs reference the entity before allowing deletion.

---

#### NotificationController (7 actions — Authenticated users)

**Namespace**: `RecruitmentTracking.Controllers`
**Authorization**: `[Authorize]`
**Dependencies**: `INotificationApplicantService`

| Action | HTTP | Purpose |
|--------|------|---------|
| `Index` | GET | Paginated notification list. |
| `MarkAsRead` | POST | Mark single notification as read. |
| `MarkAllAsRead` | POST | Mark all notifications as read. |
| `GetUnreadCount` | GET | Return unread count as JSON. |
| `GetPreview` | GET | Return recent notifications as JSON. |
| `DeleteReadNotification` | DELETE | Delete a single read notification. |
| `DeleteAllReadNotification` | DELETE | Delete all read notifications. |

---

#### ErrorController (2 actions — Public)

**Namespace**: `RecruitmentTracking.Controllers.Error`
**Route**: `/StatusCodeError/{statusCode}`

Handles HTTP error codes (404, 440, 3xx, 5xx) with appropriate error messages.

---

### 5c. Services Module

Located in `RecruitmentTracking/Services/`. Organized into 8 sub-folders plus 2 standalone services. All services follow the **Interface + Implementation** pattern and are registered as **Scoped** in DI.

#### Result Pattern

All services that can fail use the `Result<T>` struct:
```csharp
public readonly struct Result<T>
{
    public bool Success { get; }
    public string Error { get; }
    public T Value { get; }
    
    public static Result<T> Ok(T value);
    public static Result<T> Failed(string error);
}
```

---

#### ATS Sub-Module (`Services/ATS/`)

| Service | Interface | Key Methods | Description |
|---------|-----------|-------------|-------------|
| `AiAtsService` | `IAiAtsService` | `AnalyzeResumeAsync(AtsRequestDTO)` → `Result<AtsResponseDTO>` | Sends CV text + job requirements to DeepSeek API. Returns ATS score (0-10), summary, pros/cons, matched/missing skills. Uses structured JSON prompt. Validates input with `AtsRequestValidator`. |
| `AiProcessAnalysisService` | `IAiProcessAnalysisService` | `AnalyzeProcessAsync()` | AI analysis of recruitment process data. |
| `TalentPoolService` | `ITalentPoolService` | `SaveToTalentPoolAsync`, `IsInPoolAsync`, `ImportFromTalentPoolAsync`, `GetTalentPoolForAdminAsync`, `SoftDeleteTalentAsync`, `HardDeleteTalentAsync`, `UpdateTalentNotesAsync` | Full talent pool lifecycle. When saving: caches skills from ATS data, creates notification, logs audit trail. When importing: creates new `JobApplication` with status "Sourced". |
| `TalentMatchingService` | `ITalentMatchingService` | `GetTalentsMatchingJobAsync`, `GetJobsMatchingTalentAsync` | **Keyword-based matching** (not AI). Extracts keywords from job requirements and talent skills, normalizes words, calculates weighted scores. Returns ranked matches with compatibility percentages. |

**ATS Flow**:
```
Applicant submits CV (PDF)
  → PdfProcessingService.ExtractTextAsync()
    → iText extracts text layer
    → If < 50 chars: fallback to Tesseract OCR
    → Parses sections (Skills, Experience, Education)
  → BackgroundTaskQueue.QueueAsync()
  → AiAnalysisWorker picks up task
    → AiAtsService.AnalyzeResumeAsync()
      → DeepSeek API call with structured JSON schema
      → Returns AtsResponseDTO (score, summary, pros, cons, matched/missing skills)
    → Updates JobApplication with ATS results
```

---

#### Applicant Sub-Module (`Services/Applicant/`)

| Service | Interface | Key Methods |
|---------|-----------|-------------|
| `ApplicantService` | `IApplicantService` | `GetApplicantAsync`, `GetUserProfileViewAsync`, `EditProfileAsync`, `ApplyJobAsync`, `GetApplicantApplicationStatusViewsAsync`, `WithdrawApplyJobAsync`, `GetSavedJobsForApplicantAsync`, `SaveJobAsync`, `DeleteSavedJobAsync`, `GetRecommendedJobsAsync`, `GetSubscriptionPreferencesAsync`, `UpdateSubscriptionPreferencesAsync`, `SubscribeAsync`, `UnsubscribeAsync`, `UnsubscribeByTokenAsync`, `PatchApplicantProfileAsync`, `ExportApplicantsToCSVAsync` |

**`ApplyJobAsync` Detailed Flow** (the most complex method, ~120 lines):
1. Validates applicant profile is complete
2. Checks if already applied to this job
3. Saves CV file via `FileService`
4. Extracts CV text via `PdfProcessingService`
5. Creates `JobApplication` record with status `Administration`
6. Increments `Job.ApplicantCount`
7. Queues ATS analysis via `BackgroundTaskQueue`
8. Sends in-app notification via `NotificationApplicantService`
9. Returns success/failure result

---

#### Background Sub-Module (`Services/Background/`)

| Service | Type | Purpose |
|---------|------|---------|
| `BackgroundTaskQueue` | Singleton (`IBackgroundTaskQueue`) | Channel-based async task queue using `System.Threading.Channels`. Methods: `QueueAsync(Func<IServiceProvider, Task>)`, `DequeueAsync(CancellationToken)`. |
| `AiAnalysisWorker` | Hosted Service (`BackgroundService`) | Continuously dequeues tasks from `BackgroundTaskQueue` and executes them. Creates a new DI scope for each task. Handles exceptions gracefully. |

**Flow**: `ApplicantService` → `BackgroundTaskQueue.QueueAsync()` → `AiAnalysisWorker.ExecuteAsync()` → `AiAtsService.AnalyzeResumeAsync()` → Update `JobApplication`

---

#### Configuration Sub-Module (`Services/Configuration/`)

| Service | Interface | Purpose |
|---------|-----------|---------|
| `AppUserService` | `IAppUserService` | User CRUD, role assignment, password management via `UserManager<AppUser>`. |
| `DepartmentService` | `IDepartmentService` | Department CRUD with job dependency validation. |
| `CountryService` | `ICountryService` | Country CRUD with location dependency checks. |
| `LocationService` | `ILocationService` | Location CRUD with country linkage. |
| `EducationService` | `IEducationService` | Education level CRUD. |
| `EmploymentTypeService` | `IEmploymentTypeService` | Employment type CRUD with job dependency checks. |
| `AiClient` | `IAiClient` | Wrapper around OpenAI SDK configured for DeepSeek API. Single method: `CompleteChatAsync(systemPrompt, userMessage)`. |

---

#### Core Sub-Module (`Services/Core/`)

| Service | Interface | Key Methods | Purpose |
|---------|-----------|-------------|---------|
| `FileService` | `IFileService` | `DownloadCVAsync`, `ViewCVAsync`, `SaveProfileImageAsync`, `SaveCVFileAsync`, `ValidateFileUploadAsync`, `DeleteFileAsync` | File upload/download management for CVs (`wwwroot/uploads/cv/`) and profile pictures (`wwwroot/uploads/profile/`). Includes MIME type validation and cleanup. |
| `ReferenceDataService` | `IReferenceDataService` | `GetDepartmentsAsync`, `GetLocationsAsync`, `GetCountriesAsync`, `GetEducationsAsync`, `GetEmploymentTypesAsync`, `GetHrStaffsAsync`, `GetEndUserStaffsAsync` | Aggregates reference data for dropdown lists in views. |
| `ScheduleService` | `IScheduleService` | `GetHRSchedulesAsync`, `GetEndUserSchedulesAsync`, `GetAllHRSchedulesAsync`, `GetAllEndUserSchedulesAsync` | Interview schedule management with pagination. Uses `IPaginationRepository<JobApplication>` with complex expression filters and deep includes. |
| `ApplicantProfileService` | `IApplicantProfileService` | `GetApplicantProfileAsync(userId, jobId?)` | Aggregates applicant profile data for partial views (used in Admin/HR/EndUser applicant profile modals). |

---

#### Jobs Sub-Module (`Services/Jobs/`)

| Service | Interface | Key Methods | Purpose |
|---------|-----------|-------------|---------|
| `JobService` | `IJobService` | `CreateJobAsync`, `UpdateJobAsync`, `ActivateJobAsync`, `CloseJobAsync`, `SoftDeleteJobAsync`, `HardDeleteJobAsync`, `RestoreJobAsync`, `BulkActivateJobsAsync`, `BulkCloseJobsAsync`, `BulkSoftDeleteJobsAsync`, `BulkHardDeleteJobsAsync`, `GetJobPreviewAsync`, `GetAvailableJobsAsync`, `GetClosedJobsAsync`, `GetJobApplicationsAsync`, `ExportJobsToCSVAsync`, `ExportJobApplicationsToCSVAsync` | Complete job lifecycle management. Uses `CreateJobValidator` for validation. Audit trail logged on create/update/delete. |
| `JobSearchService` | `IJobSearchService` | `GetJobSearchViewModelWithFilterWithPaginationAsync(page, pageSize, search, departments[], locations[], countries[], isDraft, isDeleted, isOpen, sortBy)` | Advanced multi-filter search with pagination and sorting. Used by Home, Admin, HR, and EndUser controllers. |
| `JobRecommendationService` | `IJobRecommendationService` | `GetRecommendationsAsync(applicantId)` | AI-powered job recommendations. Sends applicant profile (major, education, past applications) + available jobs to DeepSeek. Caches results in `JobRecommendation` entity. Checks for cached results before making API call. |
| `RecruitmentProcessService` | `IRecruitmentProcessService` | `GetAdministrationApplicantsAsync`, `GetHRInterviewApplicantsAsync`, `GetEndUserInterviewApplicantsAsync`, `GetOfferingApplicantsAsync`, `AcceptApplicantAsync`, `RejectApplicantAsync`, `SaveHRInterviewScheduleAsync`, `SaveEndUserInterviewScheduleAsync`, `AssignHRInterviewerAsync`, `AssignEndUserInterviewerAsync`, `AddInterviewNotesAsync`, `AutoCloseOtherApplicationsAsync` | **Core recruitment pipeline engine** (810 lines). Manages stage transitions, interview scheduling with Google Calendar/Jitsi integration, email sending, accept/reject logic, audit trail logging. When accepting: auto-closes other applications for the same job. |

**Recruitment Pipeline State Machine** (managed by `RecruitmentProcessService`):
```
Administration → HRInterview → UserInterview → Offering → Accepted
     ↓               ↓              ↓             ↓
  Rejected        Rejected      Rejected      Rejected
     ↓               ↓              ↓
  TalentPool     TalentPool    TalentPool

GetNextStatus() determines: Administration→HRInterview→UserInterview→Offering
```

---

#### Notifications Sub-Module (`Services/Notifications/`)

| Service | Interface | Key Methods | Purpose |
|---------|-----------|-------------|---------|
| `NotificationApplicantService` | `INotificationApplicantService` | `CreateNotificationAsync`, `GetNotificationsAsync`, `MarkAsReadAsync`, `MarkAllAsReadAsync`, `GetUnreadCountAsync`, `GetPreviewNotificationsAsync`, `CreateBulkNotificationAsync`, `DeleteReadNotificationAsync`, `DeleteAllReadNotificationAsync` | In-app notification management. Pushes notifications in real-time via SignalR `NotificationHub`. Broadcasts unread count updates. |
| `EmailService` | `IEmailService` | `SendHRInterviewEmailAsync`, `SendEndUserInterviewEmailAsync`, `SendOfferingEmailAsync`, `SendRejectionEmailAsync` | Sends SMTP emails via MailKit. Uses `EmailTemplate` entity for customizable content. Validates templates before sending. Generates `MimeMessage` with HTML body. |
| `EmailTemplateService` | `IEmailTemplateService` | CRUD for `EmailTemplate` entities | Manages customizable email templates per recruitment stage. |

---

#### Shared Sub-Module (`Services/Shared/`)

**Audit** (`Services/Shared/Audit/`):
| Service | Interface | Purpose |
|---------|-----------|---------|
| `AuditService` | `IAuditService` | Creates `AuditTrail` entries: user name, role, action description, job reference, location, timestamp. Called from controllers and services after significant operations. |

**Utilities** (`Services/Shared/Utilities/`):
| Service | Interface | Purpose |
|---------|-----------|---------|
| `CsvService` | `ICsvService` | Exports data to CSV using CsvHelper. Methods: `ExportJobsToCSV`, `ExportJobApplicationsToCSV`, `ExportApplicantsToCSV`. Returns `FileContentResult`. |
| `PdfProcessingService` | `IPdfProcessingService` | Extracts text from PDF CVs. **Two-stage process**: (1) iText text layer extraction, (2) Tesseract OCR fallback if text is < 50 chars. Parses CV sections (skills, experience, education, organization). |
| `JitsiService` | `IJitsiService` | Generates Jitsi Meet room URLs for online interviews. |
| `GoogleCalendarService` | — | Creates Google Calendar events for interview schedules (uses service account key). |
| `Result<T>` | — | Generic result struct: `Success` (bool), `Value` (T), `Error` (string). Factory methods: `Ok(value)`, `Failed(error)`. |

---

#### Standalone Services

| Service | Type | Purpose |
|---------|------|---------|
| `SubscriptionBackgroundService` | `BackgroundService` (Hosted) | Runs periodically. Scans `JobSubscription` entries → finds matching new jobs by keywords/departments/locations → sends email notifications via SMTP. Creates structured HTML emails with job listings. |
| `ViewRenderService` | Scoped | Renders Razor views to HTML strings. Used by `EmailService` to generate rich email content from Razor templates. |

---

### 5d. Repositories Module

Located in `RecruitmentTracking/Repositories/`. All follow the **Repository Pattern** with interfaces in `Repositories/Interfaces/`.

#### Generic Repositories

| Repository | Purpose |
|------------|---------|
| `GenericRepository<T>` / `IGenericRepository<T>` | Base CRUD: `GetAll`, `GetById`, `Add`, `Update`, `Delete`, `SaveAsync`. |
| `PaginationRepository<T>` / `IPaginationRepository<T>` | Generic pagination: `GetPagedResultAsync(page, pageSize, filter, includes)` → `PagedResultDTO<T>`. |

#### Domain-Specific Repositories (22 total)

| Repository | Entity | Special Operations |
|------------|--------|-------------------|
| `JobRepository` | `Job` | `GetAllWithIncludesAsync` (deep includes: Department, Location, Education, EmploymentType, HrStaff, EndUserStaff), `GetByIdWithIncludesAsync`, status-filtered queries, search by title/description. |
| `JobApplicationRepository` | `JobApplication` | `GetByJobIdAsync`, `GetByApplicantIdAsync`, `GetByIdWithIncludesAsync` (deep includes: Job, Applicant, AppUser, HrInterview, EndUserInterview). |
| `ApplicantRepository` | `Applicant` | `GetByUserIdAsync` with Education and SavedJobs includes. |
| `AppUserRepository` | `AppUser` | `GetByUserNameAsync`, `GetByEmailAsync`, role-based user queries. |
| `HrStaffRepository` | `HrStaff` | `GetByUserIdAsync` with AppUser include. |
| `EndUserStaffRepository` | `EndUserStaff` | `GetByUserIdAsync` with AppUser include. |
| `HrInterviewRepository` | `HrInterview` | `GetAllAsync` with HrStaff includes. |
| `EndUserInterviewRepository` | `EndUserInterview` | `GetAllAsync` with EndUserStaff includes. |
| `DepartmentRepository` | `Department` | Job count aggregation. Dependency checks before delete. |
| `LocationRepository` | `Location` | Country include. Dependency checks before delete. |
| `CountryRepository` | `Country` | Location include. Dependency checks before delete. |
| `EducationRepository` | `Education` | Standard CRUD. |
| `EmploymentTypeRepository` | `EmploymentType` | Job count aggregation. Dependency checks before delete. |
| `EmailTemplateRepository` | `EmailTemplate` | Standard CRUD. |
| `SavedJobRepository` | `SavedJob` | `GetByApplicantAndJobAsync`, `GetByApplicantIdAsync` with Job includes. |
| `JobSubscriptionRepository` | `JobSubscription` | `GetByApplicantIdAsync`, `GetByTokenAsync` (for unsubscribe links). |
| `JobRecommendationRepository` | `JobRecommendation` | `GetByApplicantIdAsync`, caching support. |
| `NotificationApplicantRepository` | `NotificationApplicant` | `GetByUserIdAsync` (ordered by CreatedAt desc), `GetUnreadCountAsync`, `AddRangeAsync`. |
| `AuditTrailRepository` | `AuditTrail` | Append-only storage. |
| `TalentPoolRepository` | `TalentPool` | Deep includes (JobApplication → Applicant → Education, Job → Department). |

---

### 5e. AutoMapper Profiles

Located in `RecruitmentTracking/Mapping/`. 7 profile classes that define Entity ↔ DTO mappings.

| Profile | Mappings | Notes |
|---------|----------|-------|
| `JobProfile` | `Job` ↔ `JobDTO`, `CreateJobDTO` | Includes flattening of navigation properties (Department.DepartmentName → DepartmentName). |
| `JobApplicationProfile` | `JobApplication` ↔ `ApplicantApplicationDTO`, `ApplicantApplicationStatusDTO` | Complex mappings with nested navigation properties. |
| `ApplicantProfile` | `Applicant` ↔ `ApplicantProfileDTO` | Includes Education flattening. |
| `InterviewerProfile` | `HrInterview`/`EndUserInterview` ↔ `InterviewerScheduleDTO` | Maps interview schedules. |
| `SavedJobProfile` | `SavedJob` ↔ `SavedJobsDTO` | Includes Job navigation property flattening. |
| `JobRecommendationProfile` | `JobRecommendation` ↔ `JobRecommendationDTO` | Maps AI recommendation data. |
| `TalentPoolProfile` | `TalentPool` ↔ `TalentPoolDTO`, `AdministrationTalentPoolDTO` | Complex deep mappings: TalentPool → JobApplication → Applicant → Education. |

---

### 5f. FluentValidation Validators

Located in `RecruitmentTracking/Validator/`. Registered via `AddFluentValidationAutoValidation()`.

| Validator | DTO | Key Rules |
|-----------|-----|-----------|
| `AtsRequestValidator` | `AtsRequestDTO` | CV text must not be empty. Job description must not be empty. |
| `ExtractCvTextValidator` | `ExtractCvTextDTO` | Validates extracted CV text sections. |
| `JobRecommendationValidator` | `JobRecommendationDTO` | Validates recommendation request. |
| `CreateJobValidator` | `CreateJobDTO` | Title required, description required, requirement required, valid dates (posted < expired), department/location/education/employment type IDs required. |

---

### 5g. SignalR Hubs Module

Located in `RecruitmentTracking/Hubs/`.

#### NotificationHub (`/notificationHub`)

**Authorization**: `[Authorize]`
**Dependencies**: `INotificationApplicantService`

**Connection Lifecycle**:
```csharp
OnConnectedAsync()    → Adds user to group "User_{userId}"
OnDisconnectedAsync() → Removes user from group "User_{userId}"
```

**Server Methods**:
| Method | Parameters | Description |
|--------|------------|-------------|
| `SendNotification` | `userId`, `message`, `notificationType`, `notificationId` | Push notification to specific user's group. |
| `MarkNotificationAsRead` | `notificationId` | Marks notification as read in DB and broadcasts to user group. |

**Client Events** (JavaScript listens for):
- `ReceiveNotification` — New notification received
- `NotificationMarkedAsRead` — A notification was marked as read
- `AllNotificationsMarkedAsRead` — All notifications marked as read
- `UnreadCountUpdated` — Unread count changed
- `NotificationDeletedSuccessful` — A notification was deleted
- `AllReadNotificationsHasBeenDeleted` — All read notifications deleted

---

### 5h. Identity & Authentication Module

Located in `RecruitmentTracking/Areas/Identity/`.

**Framework**: ASP.NET Core Identity with Razor Pages (scaffolded).
**User Model**: `AppUser` extends `IdentityUser` with `Name` and `CreateDate`.

**Authentication Methods**:
1. **Local login** — Email + password via ASP.NET Identity
2. **Google OAuth** — Configured via `Authentication:Google` settings
3. **Facebook OAuth** — Configured via `Authentication:Facebook` settings

**Roles** (seeded at startup):
| Role | Default Email | Default Password |
|------|--------------|-----------------|
| Admin | `admin@fmlx.com` | `Admin0!!` |
| HR | `hr@fmlx.com` | `HumanResource0!!` |
| EndUser | `enduser@fmlx.com` | `Enduser0!!` |
| Applicant | `applicant@fmlx.com` | `Applicant0!!` |

**Cookie Configuration**: `SameSite=Lax`, `HttpOnly=true`, `SecurePolicy=Always`.

---

### 5i. Views & Frontend Module

Located in `RecruitmentTracking/Views/` and `RecruitmentTracking/wwwroot/`.

#### Views Structure
```
Views/
├── Admin/          # 19 views — Dashboard, job CRUD, pipeline stages, email templates
├── HR/             # 18 views — Same as Admin with slightly reduced scope
├── Applicant/      # 8 views  — Profile, applications, saved jobs, recommendations
├── EndUser/        # 5 views  — Schedules, job history, interview notes
├── Configure/      # 24 views — CRUD for all reference data entities
├── Home/           # 2 views  — Public job listing, job detail
├── Notification/   # 1 view   — Notification center
├── Error/          # 1 view   — Error page with status code display
└── Shared/         # 18 views — _Layout, partials, components
    ├── _Layout.cshtml            — Main layout with navbar, sidebar, notification bell
    ├── _ApplicantProfilePartial  — Applicant profile modal (AJAX loaded)
    ├── _ViewImports.cshtml       — Tag helpers and namespace imports
    └── _ValidationScriptsPartial — Client-side validation scripts
```

#### Static Assets (`wwwroot/`)
```
wwwroot/
├── css/          # 7 stylesheets (site.css, admin.css, applicant.css, etc.)
├── js/           # 33 scripts (AJAX handlers, form validation, DataTables, notifications)
├── lib/          # jQuery, Bootstrap, DataTables, SignalR client, Chart.js
├── uploads/      # Runtime file storage
│   ├── cv/       # Uploaded CVs (PDFs)
│   └── profile/  # Profile images
└── images/       # Static images, logos
```

**Key JavaScript Files**:
- SignalR notification client (connects to `/notificationHub`)
- AJAX form handlers for interview scheduling, email sending
- DataTables configuration for applicant lists
- Chart.js for dashboard statistics

---

## 6. Module 5 — RecruitmentTracking.Test (Test Project)

**Project**: `RecruitmentTracking.Test.csproj`
**Framework**: NUnit 4.2.2

### Test Structure
```
RecruitmentTracking.Test/
├── Controllers/   # 8 controller test classes
├── Services/      # 26 service test classes
├── Repositories/  # 22 repository test classes
├── Converters/    # 1 converter test class
├── Helpers/       # 1 test helper class
└── TestData/      # 2 test data files (sample PDFs, etc.)
```

### Testing Tools
| Tool | Purpose |
|------|---------|
| **NUnit** | Test framework (attributes: `[Test]`, `[SetUp]`, `[TestFixture]`) |
| **Moq** | Mocking interfaces for service/repository dependencies |
| **FakeItEasy** | Alternative mocking framework |
| **FluentAssertions** | Readable assertions: `result.Should().BeTrue()` |
| **Bogus** | Fake data generation for entity creation |
| **EF Core InMemory** | In-memory database provider for repository integration tests |

### Testing Pattern
```csharp
[TestFixture]
public class JobServiceTests
{
    private Mock<IJobRepository> _jobRepository;
    private JobService _sut; // System Under Test

    [SetUp]
    public void Setup()
    {
        _jobRepository = new Mock<IJobRepository>();
        _sut = new JobService(_jobRepository.Object, ...);
    }

    [Test]
    public async Task CreateJobAsync_ValidJob_ReturnsSuccess()
    {
        // Arrange
        _jobRepository.Setup(r => r.AddAsync(It.IsAny<Job>())).ReturnsAsync(true);
        
        // Act
        var result = await _sut.CreateJobAsync(new CreateJobDTO { ... });
        
        // Assert
        result.Success.Should().BeTrue();
    }
}
```

---

## 7. Cross-Cutting Concerns

### 7.1 Result Pattern

All service methods that can fail return `Result<T>`:
```csharp
public readonly struct Result<T>
{
    public bool Success { get; }
    public string Error { get; }
    public T Value { get; }
    
    public static Result<T> Ok(T value);
    public static Result<T> Failed(string error);
}
```
Controllers check `result.Success` and redirect or display errors accordingly.

### 7.2 Audit Trail

Every significant action is logged via `AuditService`:
- **Who**: User name + role
- **What**: Action description
- **Where**: Job title/reference
- **When**: Timestamp

Actions logged: job creation/edit/delete/close, applicant accept/reject, talent pool operations, email sending.

### 7.3 Pagination

Generic `PaginationRepository<T>` provides:
```csharp
Task<PagedResultDTO<T>> GetPagedResultAsync(
    int page, int pageSize,
    Expression<Func<T, bool>> filter,
    Func<IQueryable<T>, IQueryable<T>> includes
);
```

`PagedResultDTO<T>` contains: `Items`, `Page`, `PageSize`, `TotalCount`.

### 7.4 File Upload Security

`FileService` validates:
1. File size against configured maximum
2. MIME type against whitelist (`MIMECheck.cs`)
3. File extension matching
4. Generates unique filenames to prevent overwriting

---

## 8. Request Lifecycle (End-to-End Flow)

### Example: Applicant Applies to a Job

```
1. [Browser] User clicks "Apply" → POST /Applicant/ApplyJobs (with CV file)
   
2. [Controller] ApplicantController.ApplyJobs()
   → Gets current AppUser via UserManager
   → Calls ApplicantService.ApplyJobAsync()
   
3. [Service] ApplicantService.ApplyJobAsync()
   → Validates: profile complete? already applied?
   → FileService.SaveCVFileAsync() → saves PDF to wwwroot/uploads/cv/
   → PdfProcessingService.ExtractTextAsync()
     → iText extracts text from PDF pages
     → If text < 50 chars → Tesseract OCR fallback
     → Parses CV into sections (name, skills, experience, education)
   → Creates JobApplication (status = "Administration")
   → JobApplicationRepository.AddAsync()
   → Increments Job.ApplicantCount
   → BackgroundTaskQueue.QueueAsync(ATS analysis task)
   → NotificationApplicantService.CreateNotificationAsync()
     → Writes notification to DB
     → SignalR push: Clients.Group("User_{hrUserId}").SendAsync("ReceiveNotification", ...)
   
4. [Background] AiAnalysisWorker picks up queued task
   → AiAtsService.AnalyzeResumeAsync()
     → AiClient.CompleteChatAsync() → DeepSeek API
     → JSON response: { score, summary, pros, cons, matchedSkills, missingSkills }
   → Updates JobApplication with ATS results
   
5. [Controller] Returns RedirectToAction("TrackJob") with success TempData message
   
6. [Browser] User sees their application in the tracking view with status "Administration"
```

### Example: HR Advances Applicant to Interview

```
1. [Browser] HR clicks "Schedule Interview" → fills form → POST /HR/SaveHRInterview
   
2. [Controller] HRController.SaveHRInterview()
   → Calls RecruitmentProcessService.SaveHRInterviewScheduleAsync()
   
3. [Service] RecruitmentProcessService.SaveHRInterviewScheduleAsync()
   → Creates HrInterview entity (date, location, meeting link)
   → If online: JitsiService generates meeting room URL
   → Links HrInterview to JobApplication
   → Updates JobApplication.StatusInJob = "HRInterview"
   → GoogleCalendarService creates calendar event (optional)
   → AuditService logs the action
   → NotificationApplicantService notifies the applicant
   
4. [Controller] Returns redirect with success message
   
5. [Applicant's Browser] SignalR receives "ReceiveNotification" → notification bell updates
```

---

> **Last Updated**: February 25, 2026
> **Generated From**: Full codebase analysis of the Recruitment Tracking solution

