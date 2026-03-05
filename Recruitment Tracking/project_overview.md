# Recruitment Tracking — Project Overview

## 1. Introduction

**Recruitment Tracking** is a comprehensive ASP.NET Core 8.0 MVC web application that automates the entire hiring workflow. It provides a centralized platform where candidates can discover and apply for jobs, while Admins and HR manage postings, track recruitment progress, and oversee the hiring lifecycle.

### Key Highlights
- **AI-Powered ATS** — Resume scoring and analysis via DeepSeek API (OpenAI SDK)
- **AI Job Recommendations** — Personalized job suggestions based on applicant profile
- **Real-Time Notifications** — Push notifications via SignalR
- **Role-Based Access Control** — Four roles: Admin, HR, EndUser, Applicant
- **OAuth Authentication** — Google & Facebook login alongside ASP.NET Core Identity
- **Talent Pool** — Save promising candidates with AI-powered talent matching
- **Email Automation** — Customizable email templates for each recruitment stage
- **Data Export** — CSV exports for jobs, applicants, and audit trails
- **Google Calendar Integration** — Interview scheduling with calendar events
- **Video Interviews** — Jitsi Meet integration for online interviews

---

## 2. Tech Stack

| Category | Technology |
|---|---|
| **Framework** | ASP.NET Core 8.0 (MVC + Razor Pages) |
| **Language** | C# / .NET 8.0 |
| **Database** | SQLite |
| **ORM** | Entity Framework Core 8.0 |
| **Identity** | ASP.NET Core Identity |
| **OAuth** | Google, Facebook |
| **AI** | OpenAI SDK (DeepSeek API) |
| **OCR** | Tesseract 5.2.0 |
| **PDF Processing** | iText 9.4.0, PDFtoImage 5.2.0 |
| **Real-Time** | SignalR |
| **Email** | MailKit 4.3.0 |
| **Mapping** | AutoMapper 12.0.1 |
| **Validation** | FluentValidation 11.3.1 |
| **CSV** | CsvHelper 31.0.2 |
| **Testing** | NUnit 4.2.2, Moq, FakeItEasy, FluentAssertions, Bogus |
| **Calendar** | Google Calendar API |
| **Video Meeting** | Jitsi Meet |

---

## 3. Architecture

The application follows a **Layered Architecture** pattern with a clean separation of concerns:

```
┌─────────────────────────────────────────────────┐
│            Presentation Layer                    │
│   Controllers → Views (Razor) → wwwroot (JS/CSS)│
│   Areas/Identity (Auth Pages)                    │
├─────────────────────────────────────────────────┤
│            Business Logic Layer                  │
│   Services/ (organized by domain)                │
│   Validator/ (FluentValidation)                  │
│   Mapping/ (AutoMapper Profiles)                 │
│   Hubs/ (SignalR)                                │
├─────────────────────────────────────────────────┤
│            Data Access Layer                     │
│   Repositories/ (specific + generic)             │
│   RecruitmentTracking.DataAccess (DbContext)     │
├─────────────────────────────────────────────────┤
│            Domain Layer                          │
│   RecruitmentTracking.Models                     │
│   (Entities, DTOs, Enums, ExportModels)          │
└─────────────────────────────────────────────────┘
```

### Dependency Flow
```
RecruitmentTracking (Web App)
    → RecruitmentTracking.DataAccess (EF Core, DbContext, Migrations)
    → RecruitmentTracking.Models (Entities, DTOs, Enums)
    → RecruitmentTracking.Utility (Mail, MIME helpers)

RecruitmentTracking.Test
    → References all above projects
```

---

## 4. Solution Structure

```
RecruitmentTracking.sln
├── RecruitmentTracking/                    # Main MVC Web Application
│   ├── Controllers/                        # 8 MVC controllers
│   ├── Views/                              # Razor views (Admin, HR, Applicant, EndUser, Configure, etc.)
│   ├── Services/                           # Business logic (8 sub-folders)
│   ├── Repositories/                       # 22 data access repositories + interfaces
│   ├── Mapping/                            # 7 AutoMapper profile classes
│   ├── Validator/                          # 4 FluentValidation validators
│   ├── Hubs/                               # SignalR NotificationHub
│   ├── Areas/Identity/                     # ASP.NET Identity pages & services
│   ├── Secrets/                            # Google Calendar service account key
│   ├── tessdata/                           # Tesseract OCR language data
│   ├── wwwroot/                            # Static files (CSS, JS, images, libs)
│   ├── Program.cs                          # App entry point & DI configuration
│   └── appsettings.json                    # Configuration
│
├── RecruitmentTracking.DataAccess/         # Data Access Layer
│   ├── Data/
│   │   ├── ApplicationDbContext.cs         # EF Core DbContext (20 DbSets)
│   │   ├── DbInitializer.cs               # Database seeding orchestrator
│   │   └── Seeder/                         # 13 seed data classes
│   ├── Migrations/                         # EF Core migrations
│   └── Repository/                         # Generic IRepository<T> base
│
├── RecruitmentTracking.Models/             # Domain Layer
│   ├── Models/                             # 20 entity classes
│   ├── DTOs/                               # 39 Data Transfer Objects
│   ├── Enum/                               # 3 enumerations
│   ├── ExportModels/                       # 3 CSV export models
│   └── Converters/                         # Custom JSON converters
│
├── RecruitmentTracking.Utility/            # Shared Utilities
│   ├── MailSettings.cs                     # Mail configuration model
│   └── MIMECheck.cs                        # MIME type validation
│
├── RecruitmentTracking.Test/               # Test Project
│   ├── Controllers/                        # Controller tests
│   ├── Services/                           # Service tests
│   ├── Repositories/                       # Repository tests
│   ├── Converters/                         # Converter tests
│   └── Helpers/                            # Test helpers
│
└── Documentation/                          # Architecture docs & diagrams
```

---

## 5. Entity Models (Database Schema)

### Core Entities

| Entity | Key | Description |
|---|---|---|
| `AppUser` | `string Id` (Identity) | Extends `IdentityUser`. Adds `Name`, `CreateDate`. |
| `Applicant` | `Guid ApplicantId` | Candidate profile: phone, major, GPA, education, profile image. Links to `AppUser`. |
| `Job` | `Guid JobId` | Job posting: title, description, requirements, major, dates, status flags (`IsOpen`, `IsDraft`, `IsDeleted`). Links to Department, Location, Education, EmploymentType, HrStaff, EndUserStaff. |
| `JobApplication` | `Guid JobApplicationId` | An applicant's application to a job. Stores CV, salary, ATS score/summary, interview notes, status. Links to Job, Applicant, AppUser, HrInterview, EndUserInterview. |

### Interview & Staff Entities

| Entity | Key | Description |
|---|---|---|
| `HrStaff` | `Guid HrStaffId` | HR staff member linked to `AppUser`. Has many Jobs and HrInterviews. |
| `EndUserStaff` | `Guid EndUserStaffId` | End-user (hiring manager) linked to `AppUser`. Has many Jobs and EndUserInterviews. |
| `HrInterview` | `Guid HrInterviewId` | HR interview schedule: date, location, meeting link, online flag. Linked to `HrStaff`. |
| `EndUserInterview` | `Guid EndUserInterviewId` | End-user interview schedule: date, location, meeting link, online flag. Linked to `EndUserStaff`. |

### Reference Data Entities

| Entity | Key | Description |
|---|---|---|
| `Department` | `int DepartmentId` | Department name. Has many Jobs. |
| `Location` | `int LocationId` | City name with foreign key to `Country`. Has many Jobs. |
| `Country` | `int CountryId` | Country name. Has many Locations. |
| `Education` | `int EducationId` | Education level (name, description, level number). Used by Jobs and Applicants. |
| `EmploymentType` | `int EmploymentTypeId` | Employment type name (Full-time, Part-time, etc.). Has many Jobs. |

### Feature Entities

| Entity | Key | Description |
|---|---|---|
| `SavedJob` | `Guid SavedJobId` | Bookmarked job by an applicant. Links Job ↔ Applicant. |
| `JobSubscription` | `Guid SubscriptionId` | Applicant's subscription preferences for job alerts: keywords, preferred departments/locations, unsubscribe token. |
| `JobRecommendation` | `Guid JobRecommendationId` | AI-generated job recommendations for an applicant. Stores recommended job IDs as JSON. |
| `NotificationApplicant` | `Guid NotificationId` | In-app notification: message, type, read status. Links to AppUser and JobApplication. |
| `TalentPool` | `Guid TalentPoolId` | Saved applicant for future opportunities. Links to JobApplication. Includes admin notes and cached skills. |
| `AuditTrail` | `Guid ActivityId` | Activity log entry: user name, role, action, job, location, timestamp. |
| `EmailTemplate` | `Guid EmailTemplateID` | Customizable templates for HR/EndUser interview emails, offering, and rejection. |

### Entity Relationship Diagram (Simplified)

```
Country 1──* Location 1──* Job *──1 Department
                               *──1 EmploymentType
                               *──1 Education
                               *──1 HrStaff ──1 AppUser (Identity)
                               *──1 EndUserStaff ──1 AppUser
                               1──* JobApplication *──1 Applicant ──1 AppUser
                                        │
                                        ├──1 HrInterview ──1 HrStaff
                                        ├──1 EndUserInterview ──1 EndUserStaff
                                        └──* NotificationApplicant

Applicant 1──* SavedJob *──1 Job
Applicant 1──* JobSubscription
Applicant 1──* JobRecommendation

JobApplication 1──* TalentPool
Any User Action ──→ AuditTrail
```

---

## 6. Recruitment Pipeline (Business Flow)

The application tracks applicants through a multi-stage hiring pipeline defined by the `ProcessType` enum:

```
  ┌──────────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────┐     ┌──────────┐
  │Administration │────▶│ HR Interview │────▶│ User Interview │────▶│ Offering │────▶│ Accepted │
  └──────────────┘     └──────────────┘     └────────────────┘     └──────────┘     └──────────┘
         │                    │                      │                    │
         │                    │                      │                    │
         ▼                    ▼                      ▼                    ▼
    ┌──────────┐        ┌──────────┐           ┌──────────┐        ┌──────────┐
    │ Rejected │        │ Rejected │           │ Rejected │        │ Rejected │
    └──────────┘        └──────────┘           └──────────┘        └──────────┘
         ▲                    ▲                      ▲
         │                    │                      │
         └──────── Can be moved to ──────────────────┘
                   Talent Pool at any stage
                   
  Additional states: Withdraw (applicant-initiated), Sourced, Closed
```

### Pipeline Stages

1. **Administration** — Initial review of application and CV. ATS scoring happens here.
2. **HR Interview** — Schedule and conduct HR interview (online via Jitsi or on-site).
3. **User Interview** — Schedule and conduct hiring manager / end-user interview.
4. **Offering** — Extend job offer to candidate.
5. **Accepted** — Candidate accepts the offer.
6. **Rejected** — Candidate is rejected (possible at any stage).
7. **Talent Pool** — Promising candidates saved for future opportunities.
8. **Withdraw** — Applicant voluntarily withdraws.
9. **Sourced** — Imported from talent pool into a new job application.

---

## 7. Roles & Permissions

| Role | Access | Key Capabilities |
|---|---|---|
| **Admin** | Full system access | All HR capabilities + system configuration, user management, bulk operations, hard-delete jobs |
| **HR** | Recruitment management | Create/edit/close jobs, manage applications, schedule interviews, send emails, manage talent pool, view audit trails |
| **EndUser** | Interview participation | View assigned jobs, manage end-user interview schedules, view applicant profiles, add interview notes |
| **Applicant** | Self-service | Browse jobs, apply, track application status, manage profile, save jobs, subscribe to job alerts, view AI recommendations |

---

## 8. Controllers

### `HomeController`
- **Public-facing** job listing page with search, filtering (department, location, country), sorting, and pagination.
- Job detail view.

### `AdminController` (62 actions, `[Authorize(Roles = "Admin")]`)
- **Dashboard**: Job statistics (active, closed, drafted, deleted).
- **Job Management**: Full CRUD, bulk operations (close, activate, soft-delete, hard-delete, restore), drafts.
- **Recruitment Process**: Administration, HR Interview, End-User Interview, Offering stages.
- **Interview Scheduling**: Assign interviewers, schedule HR and end-user interviews.
- **Email Management**: Save/send customizable email templates for each stage.
- **Talent Pool**: Save, delete, match talents to jobs, import from talent pool.
- **Audit Trail**: View and export activity logs to CSV.
- **Data Export**: CSV exports for jobs, applicants, job applications.

### `HRController` (56 actions, `[Authorize(Roles = "HR")]`)
- Nearly identical to AdminController with slightly reduced permissions (no hard-delete, no user management).
- Additional: Import candidates from talent pool, update talent notes.

### `ApplicantController` (22 actions, `[Authorize(Roles = "Applicant")]`)
- **Profile**: View and edit applicant profile (phone, major, GPA, education, image).
- **Applications**: Apply to jobs, withdraw applications, track application status.
- **Saved Jobs**: Save/remove jobs.
- **Recommendations**: AI-powered job recommendations.
- **Subscriptions**: Manage job alert preferences, subscribe/unsubscribe.

### `EndUserController` (13 actions, `[Authorize(Roles = "EndUser")]`)
- View active/closed jobs, schedules, applicant profiles.
- Add interview notes.

### `ConfigureController` (`[Authorize(Roles = "Admin")]`)
- CRUD operations for: **Departments**, **Countries**, **Locations**, **Education Levels**, **Employment Types**, **App Users**.

### `NotificationController` (`[Authorize]`)
- List notifications with pagination, mark read, mark all read, delete, get unread count, preview.

### `ErrorController`
- Custom error pages for HTTP status codes (404, 440, 5xx, etc.).

---

## 9. Services Layer

Services are organized into domain-specific folders under `RecruitmentTracking/Services/`:

### ATS (AI-Powered Applicant Tracking)
| Service | Purpose |
|---|---|
| `AiAtsService` | AI resume scoring — sends CV text and job requirements to DeepSeek, returns ATS score, pros/cons, matched/missing skills |
| `AiProcessAnalysisService` | AI analysis of recruitment process data |
| `TalentPoolService` | CRUD for talent pool entries, save applicants with skills caching |
| `TalentMatchingService` | AI-powered matching of talent pool candidates to open jobs |

### Applicant
| Service | Purpose |
|---|---|
| `ApplicantService` | Full applicant lifecycle: profile management, job applications, CV upload/download, saved jobs, subscriptions, ATS integration |

### Background
| Service | Purpose |
|---|---|
| `AiAnalysisWorker` | Background hosted service processing queued AI analysis tasks |
| `BackgroundTaskQueue` | Channel-based async task queue for AI processing |

### Configuration
| Service | Purpose |
|---|---|
| `AppUserService` | User management (CRUD, role assignment) |
| `CountryService` | Country CRUD with location dependency checks |
| `DepartmentService` | Department CRUD with job dependency checks |
| `EducationService` | Education level CRUD |
| `EmploymentTypeService` | Employment type CRUD with job dependency checks |
| `LocationService` | Location CRUD with country linkage |
| `AiClient` | Wrapper around OpenAI SDK for DeepSeek API calls |

### Core
| Service | Purpose |
|---|---|
| `FileService` | File upload/download for CVs and profile pictures with MIME validation |
| `ReferenceDataService` | Aggregates reference data (departments, locations, countries, etc.) for dropdowns |
| `ScheduleService` | HR and EndUser interview schedule management with Google Calendar integration |
| `ApplicantProfileService` | Applicant profile data aggregation for partial views |

### Jobs
| Service | Purpose |
|---|---|
| `JobService` | Full job CRUD, status management, preview, close, applicant listing |
| `JobSearchService` | Advanced job search with multi-filter, pagination, sorting |
| `JobRecommendationService` | AI-powered job recommendations using DeepSeek |
| `RecruitmentProcessService` | Manages the full recruitment pipeline: stage transitions, interview scheduling, email sending, accept/reject, audit trail logging |

### Notifications
| Service | Purpose |
|---|---|
| `NotificationApplicantService` | In-app notification CRUD, push via SignalR |
| `EmailService` | Sends emails via MailKit (interview invites, rejections, offers) |
| `EmailTemplateService` | CRUD for customizable email templates |

### Shared
| Sub-folder | Service | Purpose |
|---|---|---|
| `Audit` | `AuditService` | Creates audit trail entries for user actions |
| `Utilities` | `CsvService`, `PdfProcessingService`, `GoogleCalendarService`, `JitsiService`, `ViewRenderService` | CSV export, PDF text extraction (iText + Tesseract OCR), Google Calendar events, Jitsi meeting links, view-to-string rendering |

### Standalone
| Service | Purpose |
|---|---|
| `SubscriptionBackgroundService` | Hosted background service that periodically checks for new jobs matching subscriber preferences and sends email notifications |
| `ViewRenderService` | Renders Razor views to string for email templates |

---

## 10. Repositories Layer

All repositories follow the **Repository Pattern** with interface segregation.

### Generic Repository
- `IGenericRepository<T>` / `GenericRepository<T>` — Base CRUD operations for any entity.
- `IPaginationRepository<T>` / `PaginationRepository<T>` — Generic pagination support.
- `IRepository<T>` / `Repository<T>` — Base repository in DataAccess layer.

### Domain-Specific Repositories (22 total)
| Repository | Entity | Key Operations |
|---|---|---|
| `JobRepository` | `Job` | GetAll with includes, filter by status, search, pagination |
| `JobApplicationRepository` | `JobApplication` | Get by job/applicant, include deep navigation properties |
| `ApplicantRepository` | `Applicant` | Get by user ID with includes |
| `AppUserRepository` | `AppUser` | User lookups, role-based queries |
| `HrStaffRepository` | `HrStaff` | Staff lookups by user |
| `EndUserStaffRepository` | `EndUserStaff` | Staff lookups by user |
| `HrInterviewRepository` | `HrInterview` | Interview schedule CRUD |
| `EndUserInterviewRepository` | `EndUserInterview` | Interview schedule CRUD |
| `DepartmentRepository` | `Department` | CRUD with job count |
| `LocationRepository` | `Location` | CRUD with country include |
| `CountryRepository` | `Country` | CRUD with location include |
| `EducationRepository` | `Education` | CRUD |
| `EmploymentTypeRepository` | `EmploymentType` | CRUD |
| `EmailTemplateRepository` | `EmailTemplate` | CRUD |
| `SavedJobRepository` | `SavedJob` | Save/unsave jobs for applicants |
| `JobSubscriptionRepository` | `JobSubscription` | Subscription CRUD |
| `JobRecommendationRepository` | `JobRecommendation` | AI recommendation CRUD |
| `NotificationApplicantRepository` | `NotificationApplicant` | Notification CRUD with pagination |
| `AuditTrailRepository` | `AuditTrail` | Audit log storage |
| `TalentPoolRepository` | `TalentPool` | Talent pool CRUD with deep includes |

---

## 11. AutoMapper Profiles

Located in `RecruitmentTracking/Mapping/`. These map between entities and DTOs:

| Profile | Mappings |
|---|---|
| `JobProfile` | `Job` ↔ `JobDTO`, `CreateJobDTO` |
| `JobApplicationProfile` | `JobApplication` ↔ `ApplicantApplicationDTO`, `ApplicantApplicationStatusDTO`, etc. |
| `ApplicantProfile` | `Applicant` ↔ `ApplicantProfileDTO` |
| `InterviewerProfile` | `HrInterview`/`EndUserInterview` ↔ `InterviewerScheduleDTO` |
| `SavedJobProfile` | `SavedJob` ↔ `SavedJobsDTO` |
| `JobRecommendationProfile` | `JobRecommendation` ↔ `JobRecommendationDTO` |
| `TalentPoolProfile` | `TalentPool` ↔ `TalentPoolDTO`, `AdministrationTalentPoolDTO` |

---

## 12. Validation (FluentValidation)

Located in `RecruitmentTracking/Validator/`:

| Validator | DTO | Rules |
|---|---|---|
| `AtsRequestValidator` | `AtsRequestDTO` | Validates ATS analysis request fields |
| `ExtractCvTextValidator` | `ExtractCvTextDTO` | Validates CV text extraction input |
| `JobRecommendationValidator` | `JobRecommendationDTO` | Validates job recommendation requests |
| `CreateJobValidator` | `CreateJobDTO` | Validates job creation (required fields, date ranges, etc.) |

---

## 13. Real-Time Features (SignalR)

### NotificationHub (`/notificationHub`)
- **User groups**: Each user auto-joins `User_{userId}` group on connection.
- **Methods**:
  - `SendNotification(userId, message, type, id)` — Push notification to specific user.
  - `MarkNotificationAsRead(notificationId)` — Mark a notification as read and broadcast.
- **Client events**: `ReceiveNotification`, `NotificationMarkedAsRead`.

---

## 14. Identity & Authentication

Located in `RecruitmentTracking/Areas/Identity/`:

- **ASP.NET Core Identity** with `AppUser` (extends `IdentityUser`).
- **External Providers**: Google OAuth, Facebook OAuth.
- **Roles**: Seeded at startup via `SeedUserRole` and `SeedAppUser`.
- **Cookie Policy**: SameSite=Lax, HttpOnly, Secure.
- **Razor Pages**: Standard Identity scaffolded pages (Login, Register, Manage, etc.).

### Default Accounts (Development)

| Role | Email | Password |
|---|---|---|
| Admin | `admin@fmlx.com` | `Admin0!!` |
| HR | `hr@fmlx.com` | `HumanResource0!!` |
| EndUser | `enduser@fmlx.com` | `Enduser0!!` |
| Applicant | `applicant@fmlx.com` | `Applicant0!!` |

---

## 15. Database & Seeding

### Database
- **Engine**: SQLite (`Data Source=./Data/Database.db`)
- **ORM**: Entity Framework Core 8.0
- **Context**: `ApplicationDbContext` inherits `IdentityDbContext<AppUser>` with 20 `DbSet` properties.

### Seeders (13 classes in `DataAccess/Data/Seeder/`)
Seed data runs at startup via `DbInitializer`:

| Seeder | Seeds |
|---|---|
| `SeedUserRole` | Creates 4 roles: Admin, Applicant, EndUser, HR |
| `SeedAppUser` | Creates default users for each role |
| `DepartmentSeeder` | Departments |
| `CountrySeeder` | Countries |
| `LocationSeeder` | Locations |
| `EducationSeeder` | Education levels |
| `EmploymentTypeSeeder` | Employment types |
| `EmailTemplateSeeder` | Default email templates |
| `EndUserSeeder` | End-user staff records |
| `HumanResourceSeeder` | HR staff records |
| `JobSeeder` | Sample job postings |
| `JobApplicationSeeder` | Sample job applications |
| `JobRecommendationSeeder` | Sample AI recommendations |

---

## 16. AI Features

### ATS (Applicant Tracking System) Scoring
1. Applicant submits CV (PDF) with their job application.
2. `PdfProcessingService` extracts text from PDF using iText; falls back to Tesseract OCR for image-based PDFs.
3. `AiAtsService` sends the extracted CV text + job requirements to DeepSeek API.
4. AI returns: **ATS Score** (0–100), summary, pros/cons, matched/missing skills.
5. Results are stored on the `JobApplication` entity.
6. Processing runs asynchronously via `BackgroundTaskQueue` + `AiAnalysisWorker`.

### Job Recommendations
1. Applicant requests recommendations from their dashboard.
2. `JobRecommendationService` sends applicant profile (major, education, previous applications) + available jobs to DeepSeek.
3. AI returns ranked list of recommended job IDs with a summary message.
4. Results are cached in `JobRecommendation` entity.

### Talent Matching
1. Admin/HR views a talent pool candidate.
2. `TalentMatchingService` sends candidate's skills/experience + open jobs to DeepSeek.
3. AI returns matching jobs with compatibility scores.

---

## 17. Email System

- **Transport**: MailKit via SMTP (configured in `appsettings.json` under `MailSettings`).
- **Templates**: Stored in `EmailTemplate` entity. Customizable per-job templates for:
  - HR Interview invitation
  - End-User Interview invitation
  - Job offering
  - Rejection notification
- **Background Notifications**: `SubscriptionBackgroundService` periodically sends email alerts to subscribers when new matching jobs are posted.
- **View Rendering**: `ViewRenderService` renders Razor views to HTML strings for rich email content.

---

## 18. Data Export (CSV)

Three export models in `RecruitmentTracking.Models/ExportModels/`:

| Export Model | Contains |
|---|---|
| `JobCSV` | Job details: title, department, location, dates, status, applicant count |
| `JobApplicationCSV` | Application details: applicant info, job, status, ATS score, salary |
| `ApplicantCSV` | Applicant details: name, email, phone, education, GPA |

Export is handled by `CsvService` using CsvHelper.

---

## 19. Frontend

### Views Structure
```
Views/
├── Admin/          # 19 views — Dashboard, job management, recruitment pipeline
├── HR/             # 18 views — Similar to Admin
├── Applicant/      # 8 views — Profile, applications, saved jobs, recommendations
├── EndUser/        # 5 views — Schedules, job history
├── Configure/      # 24 views — CRUD for reference data
├── Home/           # 2 views — Job listing, job detail
├── Notification/   # 1 view — Notification center
├── Error/          # 1 view — Error page
└── Shared/         # 18 views — Layout, partials, components
```

### Static Assets (`wwwroot/`)
- **CSS**: 7 stylesheet files
- **JavaScript**: 33 script files (AJAX handlers, form validation, notifications, charts)
- **Libraries**: jQuery, Bootstrap, DataTables, SignalR client, etc.

---

## 20. Testing

The `RecruitmentTracking.Test` project contains:

| Folder | Content |
|---|---|
| `Controllers/` | 8 controller test classes |
| `Services/` | 26 service test classes |
| `Repositories/` | 22 repository test classes |
| `Converters/` | 1 converter test class |
| `Helpers/` | 1 test helper class |
| `TestData/` | 2 test data files |

### Testing Tools
- **NUnit** — Test framework
- **Moq** / **FakeItEasy** — Mocking
- **FluentAssertions** — Readable assertions
- **Bogus** — Fake data generation
- **EF Core InMemory** — In-memory database for integration tests

---

## 21. Configuration

### `appsettings.json` Key Sections

| Section | Purpose |
|---|---|
| `ConnectionStrings:DefaultConnection` | SQLite database path |
| `Authentication:Google` | Google OAuth client ID/secret |
| `Authentication:Facebook` | Facebook OAuth client ID/secret |
| `DeepsekApi:ApiKey` | DeepSeek AI API key |
| `MailSettings` | SMTP configuration (server, port, credentials) |
| `UserRole:Roles` | Available roles for seeding |
| `Admin` / `HR` / `EndUser` / `Applicant` | Default user credentials |
| `AllowedFileUploads` | File size and MIME type restrictions |
| `Jitsi:BaseUrl` | Jitsi Meet base URL for video interviews |

---

## 22. Getting Started

### Prerequisites
- .NET SDK 8.0.4+
- dotnet-ef tool 8.0.4+
- Git

### Setup Steps
```bash
# 1. Clone
git clone <repository-url> && cd recruitmenttracking

# 2. Build
dotnet build

# 3. Migrate database
cd RecruitmentTracking.DataAccess
dotnet ef --startup-project ../RecruitmentTracking database update

# 4. Run
cd ../RecruitmentTracking
dotnet run
```

The application will be available at `http://localhost:5186`.

### Optional: Configure AI Features
Set the DeepSeek API key in `appsettings.json` or user secrets:
```bash
cd RecruitmentTracking
dotnet user-secrets set "DeepSeek:ApiKey" "your-api-key-here"
```

---

## 23. Key Development Patterns

| Pattern | Implementation |
|---|---|
| **Repository Pattern** | Generic + domain-specific repositories with interfaces |
| **Service Pattern** | Business logic encapsulated in services, injected via DI |
| **Dependency Injection** | All services and repositories registered in `Program.cs` as Scoped |
| **AutoMapper** | Entity ↔ DTO mapping via profile classes |
| **FluentValidation** | Server-side validation with auto-validation middleware |
| **Background Services** | `IHostedService` for subscription emails and AI processing |
| **Result Pattern** | Services return result objects with `Success`/`Error` properties |
| **Audit Trail** | All significant actions logged via `AuditService` |
| **Soft Delete** | Jobs support soft-delete with `IsDeleted` flag |
| **Pagination** | Generic `PaginationRepository<T>` + `PagedResultDTO` |
