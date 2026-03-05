# Meeting Room Booking System — Project Overview

> A full-stack web application for managing meeting room reservations, built with a **.NET 8 Web API** backend and a **React 18 (TypeScript)** frontend. The system integrates with **Google Calendar**, **Firebase Cloud Messaging**, **Gmail API**, and **AI assistants** (Gemini / DeepSeek) to deliver a rich, enterprise-grade experience.

---

## Table of Contents

1. [Solution Structure](#1-solution-structure)
2. [Technology Stack](#2-technology-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [Project: MeetingRoomBooking.Database](#4-project-meetingroombookingdatabase)
5. [Project: MeetingRoomBooking.Api](#5-project-meetingroombookingapi)
6. [Project: MeetingRoomBooking.Web](#6-project-meetingroombookingweb)
7. [Project: MeetingRoomBooking.Tests](#7-project-meetingroombootkingtests)
8. [Key Features](#8-key-features)
9. [External Integrations](#9-external-integrations)
10. [Authentication & Authorization](#10-authentication--authorization)
11. [Background Jobs](#11-background-jobs)
12. [Error Handling & Logging](#12-error-handling--logging)
13. [Data Flow Summary](#13-data-flow-summary)
14. [Getting Started](#14-getting-started)

---

## 1. Solution Structure

```
MeetingRoomBooking.sln
├── MeetingRoomBooking.Database/     # Data layer — Entities, DbContext, Repositories
├── MeetingRoomBooking.Api/          # Backend — Controllers, Services, Middleware
├── MeetingRoomBooking.Web/          # Frontend host + React SPA (ClientApp/)
└── MeetingRoomBooking.Tests/        # Unit tests for Api and Database layers
```

| Project | Type | Target |
|---------|------|--------|
| `MeetingRoomBooking.Database` | Class Library | .NET 8 |
| `MeetingRoomBooking.Api` | ASP.NET Core Web API | .NET 8 |
| `MeetingRoomBooking.Web` | ASP.NET Core + React SPA | .NET 8 + Vite |
| `MeetingRoomBooking.Tests` | xUnit Test Project | .NET 8 |

---

## 2. Technology Stack

### Backend
| Component | Technology |
|-----------|------------|
| Framework | ASP.NET Core 8 (Web API) |
| Database | SQLite (via Entity Framework Core 8) |
| Authentication | ASP.NET Identity + JWT Bearer Tokens |
| Object Mapping | AutoMapper 13 |
| Validation | FluentValidation 11 |
| Background Jobs | Hangfire (SQLite storage) |
| Logging | NLog |
| API Documentation | Swagger / Swashbuckle |
| PDF Generation | iText7 |
| AI Integration | Microsoft Semantic Kernel (Gemini, DeepSeek, Ollama connectors) |
| Email | Google Gmail API |
| Calendar Sync | Google Calendar API |
| Push Notifications | Firebase Admin SDK |
| Password Hashing | BCrypt.Net |

### Frontend
| Component | Technology |
|-----------|------------|
| Framework | React 18 |
| Language | TypeScript |
| Build Tool | Vite 7 |
| Styling | TailwindCSS 3 |
| State Management | Preact Signals + Redux Toolkit |
| Routing | React Router DOM 6 |
| HTTP Client | Axios |
| Charts | Chart.js + react-chartjs-2 |
| Animations | Framer Motion |
| Icons | FontAwesome + Heroicons + React Icons |
| Date Logic | Luxon + date-fns |
| PDF Export | jsPDF |
| Excel Export | xlsx |
| Google Auth | @react-oauth/google |
| Push Notifications | Firebase (client SDK) |

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     React SPA (Vite)                     │
│  Pages → Components → Signals (state) → Services (API)  │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTPS (Axios)
                         ▼
┌──────────────────────────────────────────────────────────┐
│               ASP.NET Core Web API (.NET 8)              │
│  Controllers → Services → Repositories → DbContext       │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │ Middleware   │ │ Hangfire     │ │ Semantic Kernel   │  │
│  │ (Exception)  │ │ (Background) │ │ (AI Chat)         │  │
│  └─────────────┘ └──────────────┘ └──────────────────┘  │
└────────────────────────┬─────────────────────────────────┘
                         │ EF Core
                         ▼
                ┌──────────────────┐
                │   SQLite (App.db) │
                └──────────────────┘

External Services:
  • Google Calendar API  → Calendar sync
  • Gmail API            → Email notifications
  • Firebase FCM         → Browser push notifications
  • Gemini / DeepSeek    → AI chatbot
```

The application follows a **layered architecture**:

1. **Presentation Layer** — React SPA (frontend) and API Controllers (backend)
2. **Business Logic Layer** — Service classes with interface abstractions
3. **Data Access Layer** — Repository pattern over EF Core DbContext

---

## 4. Project: MeetingRoomBooking.Database

This project contains the data model hierarchy, entity configurations, and all data access logic.

### 4.1 Entities

| Entity | Description | Key Fields |
|--------|-------------|------------|
| `User` | Extends `IdentityUser`. Represents a system user. | `FirstName`, `LastName`, `FullName`, `Status` (ApprovalStatus), `IsDeleted`, `ProfilePictureUrl` |
| `Room` | A bookable meeting room. | `RoomId`, `RoomName`, `Capacity`, `CalendarId`, `CalendarColor`, `RoomImageUrl`, `AvailabilityStatus`, `Description`, `IsDeleted` |
| `Reservation` | A booking for a room with start/end times. | `ReservationId`, `EventId`, `Status`, `DateStart`, `DateEnd`, `Description`, `Recurrence`, `ReminderMinutes`, `Visibility`, `RoomId`, `UserId` |
| `ReservationAttendee` | Junction table linking users to reservations. | `Id`, `ReservationId`, `UserId` |
| `Report` | An issue/maintenance report for a room. | `ReportId`, `RoomId`, `UserId`, `Title`, `Description`, `Status`, `Priority`, `IsDeleted` |
| `Notification` | In-app notification for a user. | `NotificationId`, `UserId`, `Title`, `Message`, `Type`, `Status`, `DateRead`, optional FKs to `Reservation`, `Room`, `Report` |
| `NotificationBrowser` | Stores FCM tokens per user for push notifications. | `Id`, `UserId`, `Token`, `DateCreated` |
| `BrowserNotificationTrigger` | Scheduled notification trigger processed by Hangfire. | `Id`, `ReservationId`, `EventType`, `ExecuteAt`, `Status`, `CreatedAt` |
| `Activity` | Audit log entry for user actions. | `ActivityId`, `Action`, `Target`, `AuthorName`, `AuthorId`, `TargetName`, optional FKs to `User`, `Room`, `Reservation` |
| `Facility` | A piece of equipment/amenity (e.g., projector, whiteboard). | `FacilityId`, `Name`, `IsActive` |
| `RoomFacility` | Junction table linking facilities to rooms with condition. | `RoomId`, `FacilityId`, `Condition` |
| `PasswordResetToken` | Token for password reset flow. | `Id`, `UserId`, `Token`, `CreatedAt`, `ExpiresAt` |

### 4.2 Enums (Database Layer)

| Enum | Values |
|------|--------|
| `ActivityAction` | Create, Update, Delete, etc. |
| `ActivityTarget` | User, Room, Reservation, etc. |
| `ApprovalStatus` | Pending, Approved, Rejected |
| `FacilityCondition` | Good, NeedRepair, Broken, UnderMaintenance |
| `NotificationEventType` | Reservation-related event types |
| `NotificationStatus` | Unread, Read |
| `NotificationType` | Reservation, Report, Room, Maintenance, Announcement, etc. |
| `ReportPriority` | Low, Medium, High, Critical |
| `ReportStatus` | Pending, InProgress, Resolved, Closed |
| `ReservationStatus` | Active, Cancelled, Completed |
| `RoomAvailabilityStatus` | Available, Unavailable, Maintenance |
| `TriggerStatus` | Pending, Processed, Failed |
| `Visibility` | Public, Private |

### 4.3 DbContext

`MeetingRoomBookingDbContext` extends `IdentityDbContext<User>` and configures:
- All entity tables with Fluent API (keys, required fields, relationships, cascade deletes)
- ASP.NET Identity tables (`AspNetUsers`, `AspNetRoles`, `AspNetUserClaims`, etc.)
- Composite keys for `RoomFacility`
- Unique constraints on `NotificationBrowser` (UserId + Token)

### 4.4 Repositories

Each entity has a dedicated repository with an interface. All repositories use `MeetingRoomBookingDbContext` for data access.

| Repository | Interface | Purpose |
|-----------|-----------|---------|
| `RoomRepository` | `IRoomRepository` | CRUD for rooms |
| `ReservationRepository` | `IReservationRepository` | CRUD for reservations |
| `ReservationAttendeeRepository` | `IReservationAttendeeRepository` | Manage attendees |
| `UserRepository` | `IUserRepository` | User queries |
| `ReportRepository` | `IReportRepository` | CRUD for reports |
| `NotificationRepository` | `INotificationRepository` | CRUD for notifications |
| `NotificationBrowserRepository` | `INotificationBrowserRepository` | FCM token storage |
| `BrowserNotificationTriggerRepository` | `IBrowserNotificationTriggerRepository` | Scheduled triggers |
| `ActivityRepository` | `IActivityRepository` | Activity audit log |
| `FacilityRepository` | `IFacilityRepository` | CRUD for facilities |
| `RoomFacilityRepository` | `IRoomFacilityRepository` | Room-facility linkage |
| `PasswordResetTokenRepository` | `IPasswordResetTokenRepository` | Password reset tokens |

---

## 5. Project: MeetingRoomBooking.Api

The backend Web API project. Handles HTTP requests, business logic, external integrations, and data orchestration.

### 5.1 Controllers (14 total)

| Controller | Route Prefix | Purpose |
|-----------|-------------|---------|
| `AuthController` | `/api/auth` | Login, register, Google Sign-in, logout, token refresh, email confirmation |
| `ReservationController` | `/api/reservation` | CRUD for reservations, availability checks |
| `RoomController` | `/api/room` | CRUD for rooms, room images, room listing with filters |
| `UserController` | `/api/user` | User management (list, update, delete) |
| `RoleController` | `/api/role` | Role management (create, assign, remove) |
| `FacilityController` | `/api/facility` | CRUD for room facilities |
| `NotificationController` | `/api/notification` | Notification listing, marking as read |
| `ReportController` | `/api/report` | CRUD for room issue reports |
| `ActivityController` | `/api/activity` | Activity log queries |
| `GoogleCalendarController` | `/api/calendar` | Google Calendar sync operations |
| `AiChatController` | `/api/ai` | AI chatbot endpoint |
| `ChangePasswordController` | `/api/changepassword` | Password change flow |
| `BrowserNotificationTokenController` | `/api/notification-token` | FCM token registration |
| `BrowserNotificationPushController` | `/api/notification-push` | Manual push notification trigger |

### 5.2 Services (25+ services)

Services contain all business logic and are injected via ASP.NET Core DI.

#### Core Business Services

| Service | Interface | Description |
|---------|-----------|-------------|
| `AuthService` | `IAuthService` | Authentication logic: login, registration, Google OAuth, token generation, email confirmation |
| `ReservationService` | `IReservationService` | Reservation CRUD, availability validation, Google Calendar sync, attendee management |
| `RoomService` | `IRoomService` | Room CRUD, image upload, filtering, availability status management |
| `UserService` | `IUserService` | User CRUD, profile management, approval workflow |
| `RoleService` | `IRoleService` | Role CRUD, role assignment to users |
| `FacilityService` | `IFacilityService` | Facility CRUD |
| `ReportService` | `IReportService` | Report CRUD, status updates, PDF export |
| `ActivityService` | `IActivityService` | Activity log CRUD and queries |
| `ReservationAttendeeService` | `IReservationAttendeeService` | Attendee management for reservations |

#### Notification Services

| Service | Interface | Description |
|---------|-----------|-------------|
| `NotificationService` | `INotificationService` | In-app notification CRUD, mark as read |
| `NotificationPushService` | `INotificationPushService` | Sends push notifications via Firebase FCM |
| `NotificationTokenService` | `INotificationTokenService` | Manages browser FCM tokens |
| `NotificationTriggerService` | `INotificationTriggerService` | Processes scheduled notification triggers (Hangfire job) |
| `FirebaseMessagingService` | `IFirebaseMessagingService` | Firebase SDK wrapper |

#### External Integration Services

| Service | Interface | Description |
|---------|-----------|-------------|
| `GoogleCalendarService` | `IGoogleCalendarService` | Google Calendar API wrapper (create/update/delete events) |
| `CalendarServiceBuilder` | — | Builds authenticated `CalendarService` instance |
| `GmailAPIService` / `EmailService` | `IEmailService` | Sends emails via Gmail API (confirmations, notifications, password reset) |
| `GmailAPIServiceBuilder` | — | Builds authenticated `GmailService` instance |

#### Auth & Security Services

| Service | Interface | Description |
|---------|-----------|-------------|
| `TokenService` | `ITokenService` | JWT token generation and validation |
| `ChangePasswordService` | `IChangePasswordService` | Password change logic |
| `PasswordResetTokenService` | `IPasswordResetTokenService` | Password reset token generation/validation |

#### AI Services

| Service | Interface | Description |
|---------|-----------|-------------|
| `AiChatService` | `IAiChatService` | AI chatbot service using Semantic Kernel |
| `KernelProvider` | `IKernelProvider` | Configures Semantic Kernel with Gemini or DeepSeek |
| `AiContext` | `IAiContext` | Provides system context/prompt for AI conversations |
| `FaqPlugin` | — (SK Plugin) | Semantic Kernel plugin to answer FAQ questions |
| `RoomAvailabilityPlugin` | — (SK Plugin) | Semantic Kernel plugin to check room availability |
| `FaqRepository` | — | Loads FAQ data from `Data/faqs.json` |

#### Utility Services

| Service | Description |
|---------|-------------|
| `AutoMapperService` | AutoMapper profile for entity ↔ DTO mapping |
| `ExportService` | Data export (PDF via iText7) |

### 5.3 Models (DTOs)

#### Request DTOs (20 files)
Used for incoming API request payloads:

| DTO | Purpose |
|-----|---------|
| `SigninRequest` | Email + password login |
| `GoogleSigninRequest` | Google OAuth token login |
| `RegisRequest` | User registration |
| `ReservationRequest` | Create/update reservation |
| `RoomRequest` | Create/update room |
| `ReportRequest` | Create/update report |
| `FacilityRequest` | Create/update facility |
| `RoomFacilityRequest` | Link facility to room |
| `UserRequest` | Update user profile |
| `RoleRequest` | Role operations |
| `ChangePasswordRequest` | Password change |
| `NotificationTokenRequest` | Register FCM token |
| `NotificationPushRequest` | Trigger push notification |
| `AnnouncementRequest` | Send announcement |
| `MaintenanceAlertRequest` | Send maintenance alert |
| `AiChatRequest` | AI chat message |
| `AvailableRoomRequest` | Check room availability |
| `GetRoomsRequest` | Room query with filters/pagination |
| `LogoutRequest` | Logout |
| `NewUserRequest` | Admin create new user |

#### Response DTOs (16 files)
Used for API response payloads:

`LoginResponse`, `UserResponse`, `RoomResponse`, `ReservationResponse`, `ReservationAttendeesResponse`, `ReportResponse`, `NotificationResponse`, `NotificationTokenResponse`, `FacilityResponse`, `RoomFacilityResponse`, `RoleResponse`, `AiChatResponse`, `ChangePasswordResponse`, `RoomUsageResponse`, `RoomUsageDailyResponse`, `RoomUsageReservation`

#### Enums (API Layer)

| Enum | Purpose |
|------|---------|
| `UserRole` | Administrator, User |
| `ResultType` | Success, ValidationError, NotFound, Unauthorized, Conflict, ServerError |
| `AiActionType` | Chat, FAQ, RoomAvailability, etc. |
| `AiResponseType` | Text, RoomList |
| `ActivityOrderOptions` | Sort options for activities |
| `ReservationOrderOptions` | Sort options for reservations |
| `RoomOrderOptions` | Sort options for rooms |
| `UserOrderOptions` | Sort options for users |

### 5.4 Middleware

| Middleware | Purpose |
|-----------|---------|
| `ExceptionMiddleware` | Global exception handler that maps exceptions to HTTP status codes (e.g., `ArgumentException` → 400, `KeyNotFoundException` → 404, `UnauthorizedAccessException` → 401) and returns structured JSON error responses |

### 5.5 Validation

| Validator | Purpose |
|-----------|---------|
| `RegisRequestValidations` | Validates registration request fields |
| `ReservationRequestValidation` | Validates reservation request fields |
| `NewPasswordRequestValidation` | Validates new password requirements |

### 5.6 Data Seeding

`DbSeed.Seed()` runs at startup to:
- Create default roles (`Administrator`, `User`)
- Create a default admin account
- Seed initial rooms, reservations, reports, and notifications if the database is empty

---

## 6. Project: MeetingRoomBooking.Web

The frontend project has two parts:

### 6.1 ASP.NET Host
A minimal ASP.NET Core app that serves the React SPA (static files). Configuration in `Program.cs` is minimal.

### 6.2 React SPA (`ClientApp/`)

Built with **Vite + React 18 + TypeScript + TailwindCSS**.

#### Pages (15 pages)

| Page | Route | Access | Description |
|------|-------|--------|-------------|
| `Home` | `/` | All authenticated | Dashboard with recent reservations, activity feed, reservation chart |
| `Login` | `/login` | Public | Email/password + Google OAuth login |
| `Register` | `/register` | Public | User registration form |
| `ForgetPassword` | `/forgot-password` | Public | Request password reset email |
| `ResetPassword` | `/reset-password` | Public | Set new password via reset token |
| `Reservations` | `/reservations` | All authenticated | Full reservation list with create/edit/delete |
| `Rooms` | `/rooms` | All authenticated | Room listing (card or list view) |
| `RoomDetail` | `/rooms/:id` | All authenticated | Detail view with calendar, facilities |
| `Notifications` | `/notifications` | All authenticated | Notification list |
| `NotificationDetail` | `/notifications/:id` | All authenticated | Single notification detail |
| `Profile` | `/profile` | All authenticated | User profile card |
| `Users` | `/users` | Admin only | User management (approve, edit, delete) |
| `Activities` | `/activities` | Admin only | Activity audit log |
| `Report` | `/report` | Admin only | Room issue reports management |
| `NotFound` | `*` | All | 404 page |

#### Key Components (59+ components)

**Calendar Components:**
- `CalendarDay`, `CalendarWeek`, `CalendarMonth`, `CalendarYear` — Multi-view calendar for reservations

**Reservation Components:**
- `ReservationCreate`, `ReservationCreateModal`, `ReservationCreateForm` — Create new reservations
- `ReservationEditModal` — Edit existing reservations
- `ReservationViewModal` — View reservation details
- `ReservationDeleteModal` — Delete confirmation
- `ReservationList` — Table/list view
- `ReservationRecent` — Recent reservation cards
- `ReservationChart` — Chart.js visualization

**Room Components:**
- `RoomCard`, `RoomCardList` — Card view
- `RoomList` — Table view
- `RoomCreateModal`, `ExistingRoomCreateModal` — Create rooms
- `RoomDeleteModal` — Delete confirmation

**User Components:**
- `UserList`, `UserCreateModal`, `UserDeleteModal` — Admin user management

**Facility Components:**
- `FacilityCard`, `FacilityCardList`, `FacilityList` — Facility display
- `FacilityCreateModal`, `FacilityDeleteModal` — Facility management

**Report Components:**
- `ReportItemList`, `ReportRoomModel`, `ReportUpdateModal`, `ReportDeleteModal`

**Help Desk / AI Chat:**
- `chatWidget`, `chatBox`, `chatBubble`, `chatInput`, `chatLoading`, `chatMini` — AI chatbot widget
- `chatRoomCard`, `chatRoomCardList` — Room availability cards in chat

**Layout & Navigation:**
- `Header`, `Navigation`, `NavigationItem`, `BaseLayout`
- `Pagination`, `Dropdown`, `LogoutModal`
- `ProfileCard`, `ProfileCardItem`

#### State Management — Signals (`signals/`)

The app uses **Preact Signals** as the primary state management (reactive state). Each signal manager handles API calls and state for a specific domain:

| Signal Manager | Domain |
|---------------|--------|
| `AuthManager` | Authentication state, tokens, current user |
| `ReservationManager` | Reservation CRUD state |
| `RoomManager` | Room CRUD state |
| `UserManager` | User CRUD state |
| `NotificationManager` | Notification state |
| `ReportManager` | Report state |
| `FacilityManager` | Facility state |
| `ActivityManager` | Activity log state |
| `RoleManager` | Role state |
| `ProfileManager` | Profile state |
| `DropdownManager` | UI dropdown state |

#### Frontend Models (`models/`)

TypeScript interfaces/types for API requests, responses, and event models:
- `requests/` — 8 request type definitions
- `responses/` — 9 response type definitions
- `events/` — 9 event/model definitions
- Root enums: `ActivityAction`, `ActivityTarget`, `ApprovalStatus`, `Visibility`

#### Firebase Client (`services/firebase/`)

Firebase SDK integration for:
- FCM token registration
- Push notification handling
- Service worker for background notifications

---

## 7. Project: MeetingRoomBooking.Tests

Unit test project using **xUnit**.

```
MeetingRoomBooking.Tests/
├── Api/           # Tests for API services (44 files)
├── Database/      # Tests for repository layer (19 files)
└── GlobalUsings.cs
```

Covers both service-layer and repository-layer unit tests.

---

## 8. Key Features

### 8.1 Room Management
- CRUD operations for meeting rooms
- Room image upload
- Room availability status (Available / Unavailable / Maintenance)
- Facility management per room (with condition tracking)
- Calendar color coding
- Google Calendar integration per room

### 8.2 Reservation System
- Create, edit, cancel reservations with date/time range
- Recurrence support
- Attendee management (add/remove users)
- Conflict detection (prevent double-booking)
- Visibility settings (Public / Private)
- Reminder configuration (minutes before start)
- Automatic sync with Google Calendar (creates events)
- Multi-view calendar (Day, Week, Month, Year)

### 8.3 User Management
- Registration with email confirmation (via Gmail API)
- Admin approval workflow (Pending → Approved / Rejected)
- Role-based access control (Administrator, User)
- Google OAuth sign-in support
- Profile management with avatar upload
- Password change and reset (with token-based email flow)
- Soft delete support

### 8.4 Notification System
- **In-app notifications** — Stored in database, displayed in UI
- **Browser push notifications** — Via Firebase Cloud Messaging (FCM)
- **Scheduled notifications** — Via Hangfire recurring job (minutely)
- Notification types: Reservation, Report, Room, Maintenance, Announcement
- Read/unread status tracking
- Linked to related entities (Reservation, Room, Report)

### 8.5 Reporting & Issue Tracking
- Users can report room issues
- Reports have status (Pending → InProgress → Resolved → Closed) and priority levels
- Admin can manage and update report status
- PDF export of reports

### 8.6 AI Chatbot (Help Desk)
- AI-powered chatbot integrated into the frontend as a widget
- Supports **Gemini** and **DeepSeek** as AI providers (configurable)
- Uses **Microsoft Semantic Kernel** with plugins:
  - `FaqPlugin` — Answers predefined FAQ questions from `Data/faqs.json`
  - `RoomAvailabilityPlugin` — Checks room availability in real-time
- Displays room availability cards in chat responses

### 8.7 Activity Logging
- Tracks all CRUD actions across the system
- Records: who (author), what (action), on what (target type + name)
- Admin-only activity log page

### 8.8 Dashboard
- Home page with:
  - Recent reservations
  - Recent activity feed
  - Reservation statistics chart (Chart.js)

### 8.9 Data Export
- PDF export (iText7 for backend, jsPDF for frontend)
- Excel export (xlsx library on frontend)

---

## 9. External Integrations

### Google Calendar API
- Each room has a `CalendarId` for a corresponding Google Calendar
- Reservations are synced as Google Calendar events
- Requires a Google Cloud service account with Calendar API enabled
- Configuration: `Secrets/service_secret.json`

### Gmail API
- Sends transactional emails: confirmation, password reset, notifications
- Uses domain-wide delegation via service account
- Configuration: same service account as Calendar

### Firebase Cloud Messaging (FCM)
- Backend uses **Firebase Admin SDK** to send push notifications
- Frontend registers FCM tokens and handles notifications
- Backend config: `firebase-adminsdk.json`
- Frontend config: `.env` file with Firebase client config

### AI (Gemini / DeepSeek)
- Configurable in `appsettings.json` under the `AI` section
- Provider switching: set `AI:Provider` to `"Gemini"` or `"DeepSeek"`
- Gemini uses Google AI connector; DeepSeek uses OpenAI-compatible connector
- Semantic Kernel orchestrates AI calls and plugin routing

---

## 10. Authentication & Authorization

```
User Registration ──→ Email Confirmation ──→ Admin Approval ──→ Active User
                                                                     │
                                                          Login (Email/Password or Google)
                                                                     │
                                                               JWT Token Issued
                                                                     │
                                                         Token sent in Authorization header
```

- **ASP.NET Identity** manages users, roles, and passwords
- **JWT Bearer tokens** secure all API endpoints
- **Role-based authorization**: `Administrator` and `User` roles
- Token contains: user ID, email, role, issued-at, expiry
- Frontend stores token in `sessionStorage` and checks role for route protection
- Google OAuth: frontend sends `idToken` → backend validates with Google → issues JWT

---

## 11. Background Jobs

**Hangfire** is configured with SQLite storage. Current jobs:

| Job | Schedule | Description |
|-----|----------|-------------|
| `process-notification-triggers` | Every minute | Processes pending `BrowserNotificationTrigger` records and sends FCM push notifications |

Hangfire Dashboard is available at `/hangfire` in development.

---

## 12. Error Handling & Logging

### Global Exception Handling
The `ExceptionMiddleware` catches all unhandled exceptions and maps them:

| Exception Type | HTTP Status | Log Level |
|---------------|-------------|-----------|
| `ArgumentException` / `ArgumentOutOfRangeException` | 400 Bad Request | Warning |
| `KeyNotFoundException` | 404 Not Found | Warning |
| `UnauthorizedAccessException` | 401 Unauthorized | Warning |
| `InvalidOperationException` | 409 Conflict | Warning |
| All others | 500 Internal Server Error | Error |

### Logging
- NLog is configured via `nlog.config`
- Logs are written to `structured-log.json` (not displayed in terminal)
- All services use `ILogger<T>` for structured logging

---

## 13. Data Flow Summary

### Creating a Reservation (Example)

```
1. User fills reservation form in React SPA
2. Frontend calls POST /api/reservation with ReservationRequest DTO
3. AuthController middleware validates JWT token
4. ReservationController receives request
5. ReservationService:
   a. Validates availability (checks for conflicts)
   b. Creates Google Calendar event via GoogleCalendarService
   c. Saves Reservation entity via ReservationRepository
   d. Creates ReservationAttendees
   e. Creates Notification for attendees via NotificationService
   f. Schedules BrowserNotificationTrigger for reminders
   g. Logs Activity via ActivityService
6. Response returns ReservationResponse DTO to frontend
7. Hangfire job later processes triggers → sends FCM push notifications
```

---

## 14. Getting Started

### Prerequisites
- .NET 8 SDK
- Node.js (for React frontend)
- Google Cloud service account (Calendar + Gmail APIs)
- Firebase project (for push notifications)
- AI API key (Gemini or DeepSeek, optional for chatbot)

### Quick Start

```bash
# 1. Backend
cd MeetingRoomBooking.Api
dotnet restore
dotnet build
dotnet ef database update    # Creates/updates SQLite database
dotnet watch                 # Starts the API at https://localhost:7098

# 2. Frontend
cd MeetingRoomBooking.Web/ClientApp
npm install
npm start                    # Starts Vite dev server
```

### Configuration Files

| File | Purpose |
|------|---------|
| `MeetingRoomBooking.Api/appsettings.json` | JWT config, DB connection, AI provider settings, FAQ path |
| `MeetingRoomBooking.Api/Secrets/service_secret.json` | Google service account credentials |
| `MeetingRoomBooking.Api/firebase-adminsdk.json` | Firebase Admin SDK credentials |
| `MeetingRoomBooking.Web/ClientApp/.env` | Firebase client config for frontend |

### API Documentation
When the backend is running in development mode, Swagger UI is available at:
```
https://localhost:7098/swagger/index.html
```

---

> **Note:** This project is a collaborative bootcamp project that has evolved across multiple batch cycles (Batch 6–16). Each batch has contributed features and improvements. Technical documentation for each batch is linked in the main `README.md`.
