# Meeting Room Booking System — Technical Documentation

---

## Table of Contents

1. [Authentication & Authorization Module](#1-authentication--authorization-module)
2. [User Management Module](#2-user-management-module)
3. [Room Management Module](#3-room-management-module)
4. [Reservation Management Module](#4-reservation-management-module)
5. [Facility Management Module](#5-facility-management-module)
6. [Notification System Module](#6-notification-system-module)
7. [Report / Issue Tracking Module](#7-report--issue-tracking-module)
8. [Activity Logging Module](#8-activity-logging-module)
9. [AI Chatbot (Help Desk) Module](#9-ai-chatbot-help-desk-module)
10. [Google Calendar Integration Module](#10-google-calendar-integration-module)
11. [Email Service Module](#11-email-service-module)
12. [Push Notification Module](#12-push-notification-module)
13. [Shared Patterns & Utilities](#13-shared-patterns--utilities)

---

## 1. Authentication & Authorization Module

### Overview
Handles user registration, login (email/password and Google OAuth), logout, token management, email verification, password reset, and role-based access control.

### Architecture

```
Frontend (React)                           Backend (.NET 8)
─────────────────                          ────────────────
Login/Register Pages                       AuthController
     │                                          │
     │ POST /api/auth/*                         │
     ▼                                          ▼
AuthManager.ts ──(Axios)──►             AuthService
(Preact Signal)                              │
                                 ┌───────────┼───────────────┐
                                 ▼           ▼               ▼
                          TokenService  EmailService  NotificationService
                                 │
                                 ▼
                          UserManager<User> (ASP.NET Identity)
                                 │
                                 ▼
                          MeetingRoomBookingDbContext (SQLite)
```

### API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/auth/register` | No | Register a new user |
| `POST` | `/api/auth/login` | No | Login with email/username + password |
| `POST` | `/api/auth/google/login` | No | Login/register via Google OAuth |
| `POST` | `/api/auth/logout` | Yes | Logout current user |
| `POST` | `/api/auth/refresh-token` | No | Refresh JWT token |
| `GET`  | `/api/auth/email/verify` | No | Verify email via token link |
| `POST` | `/api/auth/forgot-password` | No | Initiate password reset |
| `POST` | `/api/auth/reset-password` | No | Confirm password reset |

### Detailed Flows

#### Registration Flow
```
1.  User submits RegisRequest { Username, Email, Password, FirstName, LastName }
2.  AuthController.Register() validates ModelState
3.  AuthService.RegisterAsync():
    a. Checks if username or email already exists → returns error if duplicate
    b. Maps RegisRequest → User entity via AutoMapper
    c. Creates user via UserManager.CreateAsync() (password is hashed by Identity)
    d. Assigns "Employee" role via UserManager.AddToRolesAsync()
    e. Creates notifications for all Administrators ("New User Registration")
    f. Generates email confirmation token via UserManager.GenerateEmailConfirmationTokenAsync()
    g. Sends confirmation email via EmailService.SendConfirmationEmail()
       - Link format: {ApiBaseUrl}/api/auth/email/verify?userId={id}&emailToken={token}
4.  User receives email → clicks verification link
5.  AuthController.VerifyEmail():
    a. Finds user by ID
    b. Confirms email via UserManager.ConfirmEmailAsync()
    c. Sends notification to user ("Email Verified")
    d. Sends notification to all admins ("User Email Verified - ready for approval")
    e. Redirects to frontend login page
6.  Admin must then approve user (via User Management Module) before they can login
```

#### Login Flow (Email/Password)
```
1.  User submits SigninRequest { Email?, Username?, Password }
2.  AuthService.AttemptLoginAsync():
    a. Validates that email or username is provided
    b. Finds user by email or username
    c. Checks user.Status == ApprovalStatus.Accepted
       - Pending → "account is still under admin review"
       - NotVerified → "set to not verified by administrator"
       - Rejected → "account has been banned"
    d. Validates password via SignInManager.PasswordSignInAsync()
3.  AuthService.LoginAsync():
    a. Gets user roles via UserManager.GetRolesAsync()
    b. Generates JWT via TokenService.GenerateToken()
    c. Generates refresh token (GUID) via TokenService.GenerateRefreshToken()
    d. Maps User → LoginResponse { UserId, UserName, AccessToken, RefreshToken }
4.  Controller sets Cache-Control: no-cache headers and 30-min Expires
5.  Frontend stores accessToken in sessionStorage
```

#### Google OAuth Flow
```
1.  Frontend uses @react-oauth/google to get idToken
2.  POST /api/auth/google/login with GoogleSigninRequest { idToken }
3.  AuthController validates token via GoogleJsonWebSignature.ValidateAsync()
4.  Validates audience matches application client ID
5.  If user does NOT exist:
    a. Creates new User with Google profile data (status = Pending)
    b. Returns 401 "account is under review"
6.  If user EXISTS:
    a. Checks ApprovalStatus (Pending/NotVerified/Rejected → 401)
    b. If Accepted → generates JWT token via LoginAsync()
    c. Sets user-id cookie (HttpOnly, Secure, 30-day expiry)
```

#### JWT Token Structure
```
Claims:
  - ClaimTypes.Name = username
  - ClaimTypes.NameIdentifier = user ID
  - ClaimTypes.Email = email
  - ClaimTypes.Role = role (e.g., "Administrator", "Employee")

Configuration:
  - Algorithm: HMAC-SHA512
  - Expiry: 7 days
  - Issuer & Audience: from appsettings.json "Jwt" section
```

#### Password Reset Flow
```
1.  POST /api/auth/forgot-password with { Email }
2.  AuthService.InitiatePasswordResetAsync():
    a. Validates email format
    b. Finds user → verifies EmailConfirmed and Status == Accepted
    c. Generates ASP.NET Identity password reset token
    d. Creates PasswordResetToken entity (expires in 15 minutes)
    e. Saves token to database via PasswordResetTokenService
    f. Sends reset email with link: {FrontendBaseUrl}/reset-password?token={encodedToken}
3.  User clicks link → frontend ResetPassword page
4.  POST /api/auth/reset-password with ResetPasswordRequest { ResetCode, NewPassword }
5.  AuthService.ConfirmPasswordResetAsync():
    a. Validates token via PasswordResetTokenService.CheckResetCode()
    b. Finds associated user
    c. Resets password via UserManager.ResetPasswordAsync()
    d. Removes used token from database
    e. Creates notification "Password Reset Complete"
```

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/AuthController.cs` | All auth endpoints (354 lines) |
| `Services/AuthService.cs` | Auth business logic (454 lines) |
| `Services/TokenService.cs` | JWT generation/decoding (68 lines) |
| `Services/ChangePasswordService.cs` | In-app password change |
| `Services/PasswordResetTokenService.cs` | Token lifecycle management |
| `Validations/RegisRequestValidations.cs` | Registration input validation |
| `Validations/NewPasswordRequestValidation.cs` | Password validation rules |
| `Models/Requests/SigninRequest.cs` | Login DTO |
| `Models/Requests/RegisRequest.cs` | Registration DTO |
| `Models/Requests/GoogleSigninRequest.cs` | Google OAuth DTO |
| `Models/Responses/LoginResponse.cs` | Login response DTO |

### Frontend State (signals/)
- `AuthManager.ts` — Manages login/logout/register/token refresh state, stores tokens in `sessionStorage`

---

## 2. User Management Module

### Overview
Admin-managed CRUD operations for users, including approval workflow, role management, and profile management. Users can update their own profile and profile picture.

### API Endpoints

| Method | Endpoint | Auth | Role | Description |
|--------|----------|------|------|-------------|
| `GET` | `/api/users` | Yes | Admin | List users (filter by role, status; sort, paginate) |
| `GET` | `/api/users/{userId}` | Yes | Any | Get user by ID |
| `GET` | `/api/users/me` | Yes | Any | Get current authenticated user |
| `POST` | `/api/users` | Yes | Admin | Create new user (admin) |
| `PUT` | `/api/users/{userId}` | Yes | Any | Update user (admin can change status/role) |
| `PUT` | `/api/users/profilePicture` | Yes | Any | Upload profile picture |
| `DELETE` | `/api/users/{userId}` | Yes | Admin | Soft-delete user |

### User Approval Workflow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Register   │────►│   Pending    │────►│   Accepted   │
│              │     │              │     │  (can login)  │
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │
                     Admin decision
                            │
                     ┌──────▼───────┐
                     │   Rejected   │
                     │  (banned)    │
                     └──────────────┘
```

**ApprovalStatus enum values:** `Pending`, `Accepted`, `NotVerified`, `Rejected`

### User Entity (Database)
```csharp
public class User : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string FullName { get; set; }
    public ApprovalStatus Status { get; set; } = ApprovalStatus.Pending;
    public bool IsDeleted { get; set; } = false;
    public string? ProfilePictureUrl { get; set; }
}
```
> Inherits from `IdentityUser`, which provides `Id`, `UserName`, `Email`, `PasswordHash`, `EmailConfirmed`, `PhoneNumber`, etc.

### Profile Picture Upload
- Endpoint: `PUT /api/users/profilePicture` with `IFormFile`
- Max file size: 1MB (configured in `Program.cs` via `FormOptions`)
- Stored as static file in `wwwroot/` directory
- URL stored in `User.ProfilePictureUrl`

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/UserController.cs` | User CRUD endpoints (179 lines) |
| `Services/UserService.cs` | User business logic (33,270 bytes) |
| `Controllers/RoleController.cs` | Role CRUD endpoints (5,590 bytes) |
| `Services/RoleService.cs` | Role business logic (10,369 bytes) |
| `Controllers/ChangePasswordController.cs` | Password change endpoint |

### Side Effects on User Update
When an admin updates a user's status:
- **Notification** created for the user ("Account Approved" / "Account Rejected")
- **Email** sent to the user about status change
- **Activity** logged (who changed what)

---

## 3. Room Management Module

### Overview
Admin-managed CRUD for meeting rooms. Each room has a linked Google Calendar, a capacity, facilities, an availability status, and an image. Rooms use **soft delete** (`IsDeleted` flag).

### API Endpoints

| Method | Endpoint | Auth | Role | Description |
|--------|----------|------|------|-------------|
| `GET` | `/api/rooms` | Yes | Any | List rooms (filter, sort, paginate) |
| `GET` | `/api/rooms/{roomId}` | Yes | Any | Get room by ID |
| `POST` | `/api/rooms` | Yes | Admin | Create room |
| `PUT` | `/api/rooms/{roomId}` | Yes | Admin | Update room |
| `DELETE` | `/api/rooms/{roomId}` | Yes | Admin | Soft-delete room |
| `GET` | `/api/rooms/find` | Yes | Any | Find available rooms for a time slot |

### Room Entity
```csharp
public class Room
{
    public Guid RoomId { get; set; }
    public string CalendarId { get; set; }         // Google Calendar ID
    public string RoomName { get; set; }
    public int Capacity { get; set; }
    public string? Description { get; set; }
    public bool IsDeleted { get; set; } = false;
    public string? CalendarColor { get; set; }     // Hex color for UI
    public string? RoomImageUrl { get; set; }
    public RoomAvailabilityStatus AvailabilityStatus { get; set; }
    public ICollection<RoomFacility> RoomFacilities { get; set; }
}
```

**RoomAvailabilityStatus:** `Available`, `Unavailable`, `Maintenance`

### Room Create Flow
```
1.  Admin submits RoomRequest { RoomName, Capacity, Description, CalendarColor, ... }
2.  RoomService.CreateRoomAsync():
    a. Creates new Google Calendar via GoogleCalendarService.InsertCalendarAsync()
    b. Sets ACL rules for the calendar
    c. Maps request → Room entity
    d. Saves to database via RoomRepository
    e. If RoomFacilities provided, creates RoomFacility link records
    f. Logs Activity (action: Create, target: Room)
3.  Returns RoomResponse
```

### Room Delete Flow (Critical)
```
1.  Admin calls DELETE /api/rooms/{roomId}
2.  RoomService.DeleteRoomAsync():
    a. Sets room.IsDeleted = true (soft delete)
    b. Sets room.AvailabilityStatus = Unavailable
    c. Cancels ALL active reservations for this room:
       - For each reservation:
         • Sets status to Canceled
         • Sends email to user via EmailService.SendUnavailableRoomEmail()
         • Creates notification "Reservation Cancelled - Room No Longer Available"
    d. Deletes Google Calendar via GoogleCalendarService.DeleteCalendarAsync()
    e. Logs Activity (action: Delete, target: Room)
```

### Find Available Rooms
- `GET /api/rooms/find?startDate=...&endDate=...&capacity=...`
- Checks for rooms that have no conflicting reservations in the given time range
- Filters by minimum capacity

### Room Query Parameters (GetRoomsRequest)
The `GET /api/rooms` endpoint supports rich filtering:
- `orderBy` — Sort field (RoomOrderOptions enum)
- `descending` — Sort direction
- `startIndex`, `count` — Pagination
- `search` — Search term
- `status` — Filter by availability status

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/RoomController.cs` | Room CRUD endpoints (150 lines) |
| `Services/RoomService.cs` | Room business logic (25,337 bytes) |

---

## 4. Reservation Management Module

### Overview
The core module of the system. Users can create, view, update, and cancel reservations. Every reservation is synced with Google Calendar, triggers notifications, sends emails, and creates activity logs. Supports recurrence, attendees, reminders, and visibility settings.

### API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/reservations` | Yes | List reservations (rich filtering/sorting) |
| `GET` | `/api/reservations/{id}` | Yes | Get reservation by ID |
| `POST` | `/api/reservations` | Yes | Create reservation |
| `PUT` | `/api/reservations/{id}` | Yes | Update reservation |
| `DELETE` | `/api/reservations/{id}` | Yes | Cancel reservation (soft) |
| `GET` | `/api/reservations/room-usage` | Yes | Get room usage statistics |

### Reservation Entity
```csharp
public class Reservation
{
    public Guid ReservationId { get; set; }
    public string EventId { get; set; }              // Google Calendar Event ID
    public ReservationStatus Status { get; set; }     // Active, Canceled, Completed, Pending, Rejected
    public DateTime DateStart { get; set; }
    public DateTime DateEnd { get; set; }
    public string? Description { get; set; }
    public string? Recurrence { get; set; }           // JSON array of RRULE strings
    public int? ReminderMinutes { get; set; }
    public Visibility Visibility { get; set; }        // Public, Private
    public Guid RoomId { get; set; }
    public Room Room { get; set; }
    public string UserId { get; set; }
    public User User { get; set; }
    public ICollection<ReservationAttendee> Attendees { get; set; }
}
```

### Create Reservation Flow (Most Complex)

```
1.  User submits ReservationRequest:
    { UserId, RoomId, Start, End, Description, Recurrence[], AttendeesEmail[],
      ReminderMinutes, Visibility }
2.  Controller validates ModelState (FluentValidation: ReservationRequestValidation)
3.  ReservationService.CreateReservationAsync():

    Step A: Validation
    a. Validates recurrence rules (no empty RRULE)
    b. Validates start/end dates are not default
    c. Verifies Room exists via RoomRepository
    d. Verifies User exists via UserManager

    Step B: Google Calendar Sync
    e. Creates Google Calendar Event:
       - Summary = Description
       - Start/End with "Asia/Jakarta" timezone
       - Recurrence rules attached
    f. Inserts event via GoogleCalendarService.InsertEventAsync()
    g. Stores returned EventId on the reservation

    Step C: Database Save
    h. Maps ReservationRequest → Reservation entity
    i. Serializes recurrence rules to JSON
    j. Saves reservation via ReservationRepository.AddAsync()

    Step D: Attendee Management
    k. Resolves attendee emails → user IDs via UserManager.FindByEmailAsync()
    l. Creates ReservationAttendee records via ReservationAttendeeService

    Step E: Email Notification
    m. Sends confirmation email via EmailService.SendCreatedReservationEmail()

    Step F: In-App Notification
    n. Creates notification "New Reservation Created" via NotificationService

    Step G: Browser Push Notification Triggers
    o. If ReminderMinutes > 0:
       - Creates BrowserNotificationTrigger for reminder (ExecuteAt = DateStart - ReminderMinutes)
       - Creates BrowserNotificationTrigger for meeting start (ExecuteAt = DateStart)
       - These are processed by Hangfire recurring job every minute
```

### Update Reservation Flow
```
1.  PUT /api/reservations/{id} with ReservationRequest
2.  ReservationService.UpdateReservationAsync():
    a. Validates request and finds existing reservation
    b. Finds associated room and user
    c. Records previous status
    d. Maps updated fields onto existing entity
    e. Updates Google Calendar event via GoogleCalendarService.UpdateEventAsync()
    f. Updates attendees (clears and re-adds)
    g. If status changed → sends notification via NotifyReservationStatusChangeAsync()
    h. Sends update email to user
    i. Logs Activity (action: Update)
```

### Delete (Cancel) Reservation Flow
```
1.  DELETE /api/reservations/{id}
2.  ReservationService.DeleteReservationAsync():
    a. Sets reservation.Status = Canceled (NOT hard delete)
    b. Updates database
    c. Sends status change notification
    d. Deletes Google Calendar event (if EventId and CalendarId present)
    e. Logs Activity (action: Delete)
```

### Query Parameters for GET /api/reservations
| Parameter | Type | Description |
|-----------|------|-------------|
| `startIndex` | int | Pagination offset |
| `count` | int | Page size |
| `userId` | Guid | Filter by user |
| `roomId` | Guid | Filter by room |
| `eventId` | string | Filter by Google Calendar event ID |
| `startDate` | string | Filter: DateStart >= value |
| `endDate` | string | Filter: DateStart <= value |
| `orderBy` | enum | DateCreated, DateStart, DateEnd, Visibility |
| `descending` | bool | Sort direction |
| `visibility` | enum | Public, Private |
| `status` | enum | Active, Canceled, Completed |

### Room Usage Statistics
- `GET /api/reservations/room-usage?startDate=...&endDate=...`
- Returns list of `RoomUsageReservation` with room and user details
- Used by the dashboard chart (Chart.js on frontend)

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/ReservationController.cs` | Reservation endpoints (257 lines) |
| `Services/ReservationService.cs` | Core reservation logic (827 lines) |
| `Services/ReservationAttendeeService.cs` | Attendee management |
| `Validations/ReservationRequestValidation.cs` | Input validation |

### Frontend Components
- `ReservationCreateModal.tsx` (23,746 bytes) — Large create form with room picker, attendee selector, recurrence, reminders
- `ReservationEditModal.tsx` (16,629 bytes) — Edit modal
- `ReservationViewModal.tsx` (23,634 bytes) — Detail view
- `ReservationList.tsx` — Table view
- `CalendarDay/Week/Month/Year.tsx` — Multi-view calendar components

---

## 5. Facility Management Module

### Overview
Admin-managed inventory of equipment and amenities (e.g., projector, whiteboard, AC). Facilities can be linked to rooms via a many-to-many relationship (`RoomFacility`) with a condition rating.

### API Endpoints (Admin only)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/facilities` | Admin | List all facilities |
| `GET` | `/api/facilities/by-{facilityId}` | Admin | Get by ID |
| `GET` | `/api/facilities/by-{facilityName}` | Admin | Get by name |
| `POST` | `/api/facilities` | Admin | Create facility |
| `PUT` | `/api/facilities/{facilityId}` | Admin | Update facility |
| `PATCH` | `/api/facilities/disable/{facilityId}` | Admin | Disable facility |

### Entities

**Facility:**
```csharp
public class Facility
{
    public Guid FacilityId { get; set; }
    public string Name { get; set; }
    public bool IsActive { get; set; } = true;
    public ICollection<RoomFacility> RoomFacilities { get; set; }
}
```

**RoomFacility (Junction Table):**
```csharp
public class RoomFacility
{
    public Guid RoomId { get; set; }        // Composite PK
    public Guid FacilityId { get; set; }    // Composite PK
    public FacilityCondition Condition { get; set; }
    public Room Room { get; set; }
    public Facility Facility { get; set; }
}
```

**FacilityCondition:** `Good`, `NeedRepair`, `Broken`, `UnderMaintenance`

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/FacilityController.cs` | 192 lines, all admin-gated |
| `Services/FacilityService.cs` | CRUD logic |

---

## 6. Notification System Module

### Overview
Multi-channel notification system with in-app notifications, browser push notifications (Firebase FCM), and scheduled triggers via Hangfire. Notifications can be linked to reservations, rooms, and reports.

### Architecture
```
                    ┌────────────────────────┐
                    │  NotificationService   │
                    │  (In-App Database)     │
                    └────────┬───────────────┘
                             │
              ┌──────────────┼──────────────────┐
              ▼              ▼                  ▼
   NotificationPush   NotificationTrigger  NotificationToken
   Service            Service               Service
   (immediate FCM)   (scheduled via        (FCM token CRUD)
                      Hangfire)
              │              │                  │
              ▼              ▼                  ▼
     Firebase Admin       Hangfire          NotificationBrowser
     SDK (FCM)           Recurring Job      Repository (DB)
                        (every minute)
```

### In-App Notification API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/notifications` | Yes | Get user's notifications (filter read/unread, paginate) |
| `GET` | `/api/notifications/count` | Yes | Get unread notification count |
| `POST` | `/api/notifications/{id}/read` | Yes | Mark notification as read |
| `POST` | `/api/notifications/mark-all-read` | Yes | Mark all as read |
| `POST` | `/api/notifications/send-announcement` | Admin | Send announcement to users |

### Notification Entity
```csharp
public class Notification
{
    public Guid NotificationId { get; set; }
    public string UserId { get; set; }
    public string Title { get; set; }
    public string Message { get; set; }
    public DateTime DateCreated { get; set; }
    public DateTime? DateRead { get; set; }
    public NotificationType Type { get; set; }
    public NotificationStatus Status { get; set; }  // Unread, Read
    // Optional related entities
    public Guid? ReservationId { get; set; }
    public Guid? RoomId { get; set; }
    public Guid? ReportId { get; set; }
}
```

**NotificationType enum:**
`UpcomingMeeting`, `ReservationCancelled`, `ReservationApproved`, `ReservationRejected`, `RoomMaintenance`, `GeneralAnnouncement`, `ReportCreated`, `ReportUpdated`

### Notification Triggers
Notifications are created as **side effects** of other operations:

| Trigger Event | Notification Created For | Type |
|---------------|------------------------|------|
| User registers | All Admins | GeneralAnnouncement |
| Email verified | User + All Admins | GeneralAnnouncement |
| Reservation created | Reservation owner | UpcomingMeeting |
| Reservation status changes | Reservation owner | Cancelled/Approved/Rejected |
| Room deleted | All affected reservation owners | ReservationCancelled |
| Password reset complete | User | GeneralAnnouncement |
| User approved/rejected | User | GeneralAnnouncement |
| Admin sends announcement | Targeted users/roles/all | GeneralAnnouncement |

### Announcement System
The `SendAnnouncementAsync()` method supports three targeting modes:
1. **Specific users** — Send to a list of user IDs
2. **By role** — Send to all users in specified roles
3. **Broadcast** — Send to all users (when no filter specified)

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/NotificationController.cs` | In-app notification endpoints (135 lines) |
| `Services/NotificationService.cs` | Notification CRUD + announcement logic (361 lines) |
| `Services/NotificationPushService.cs` | FCM push service |
| `Services/NotificationTokenService.cs` | FCM token management |
| `Services/NotificationTriggerService.cs` | Scheduled trigger processing |

---

## 7. Report / Issue Tracking Module

### Overview
Users can report issues with meeting rooms (e.g., broken equipment, cleanliness). Reports have a lifecycle with status and priority. Admins can manage report status.

### API Endpoints

| Method | Endpoint | Auth | Role | Description |
|--------|----------|------|------|-------------|
| `GET` | `/api/reports` | Yes | Any | List reports (filter by date, paginate, sort) |
| `GET` | `/api/reports/{id}` | Yes | Any | Get report by ID |
| `POST` | `/api/reports` | Yes | Any | Create report (userId from JWT) |
| `PUT` | `/api/reports/{id}` | Yes | Any* | Update report (* admins can update any, users own only) |
| `PUT` | `/api/reports/{id}/status` | Yes | Admin | Update report status only |
| `DELETE` | `/api/reports/{id}` | Yes | Any* | Delete report (* same ownership rule) |

### Report Entity
```csharp
public class Report
{
    public Guid ReportId { get; set; }
    public Guid RoomId { get; set; }
    public string UserId { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public ReportStatus Status { get; set; }
    public ReportPriority Priority { get; set; }
    public bool IsDeleted { get; set; } = false;
    public Room Room { get; set; }
    public User User { get; set; }
}
```

**ReportStatus:** `Pending` → `InProgress` → `Resolved` → `Closed`

**ReportPriority:** `Low`, `Medium`, `High`, `Critical`

### Authorization Logic
- **Create**: Any authenticated user can create reports
- **Update/Delete**: Users can only update/delete their own reports. Admins can update/delete any report.
- Enforced by checking `userId == reportOwner || isAdmin` in the service layer

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/ReportController.cs` | Report endpoints (176 lines) |
| `Services/ReportService.cs` | Report business logic (17,184 bytes) |
| Frontend: `components/report/` | 4 components: list, create, update, delete modals |

---

## 8. Activity Logging Module

### Overview
Automatic audit trail that records all significant user actions in the system. Activities track who performed what action on which resource. Admin-only visibility.

### API Endpoints (Admin only)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/activity` | Admin | List activities (filter by author, user, room; sort, paginate) |
| `GET` | `/api/activity/{id}` | Admin | Get activity by ID |
| `GET` | `/api/activity/export` | Admin | Export activities as CSV |

### Activity Entity
```csharp
public class Activity
{
    public Guid ActivityId { get; set; }
    public DateTime DateCreated { get; set; }
    public ActivityAction Action { get; set; }     // Create, Update, Delete
    public ActivityTarget Target { get; set; }     // User, Room, Reservation
    public string AuthorName { get; set; }         // Who performed the action
    public string TargetName { get; set; }         // Name of affected resource
    public string AuthorId { get; set; }
    public User Author { get; set; }
    // Optional related entities
    public string? UserId { get; set; }
    public Guid? RoomId { get; set; }
    public Guid? ReservationId { get; set; }
}
```

**ActivityAction:** `Create`, `Update`, `Delete`

**ActivityTarget:** `User`, `Room`, `Reservation`

### How Activities Are Created
Activities are NOT created directly by the ActivityController. Instead, they are created as **side effects** by other services via `IActivityService`:

```csharp
// Example: In ReservationService after creating a reservation
await _activityService.CreateReservationActivityAsync(
    ActivityAction.Create, user, newReservation);

// Example: In RoomService after deleting a room
await _activityService.CreateRoomActivityAsync(
    ActivityAction.Delete, user, room);
```

### CSV Export
- `GET /api/activity/export` returns a downloadable CSV file
- Uses `ExportService.ExportActivitiesToCsvAsync()`
- Content-Type: `text/csv`, filename: `activities.csv`

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/ActivityController.cs` | Activity endpoints (108 lines) |
| `Services/ActivityService.cs` | Activity CRUD + creation helpers |
| `Services/ExportService.cs` | CSV export logic |

---

## 9. AI Chatbot (Help Desk) Module

### Overview
An AI-powered chatbot integrated as a floating widget in the frontend. Uses **Microsoft Semantic Kernel** to orchestrate AI conversations with **plugin** support for FAQ lookup and room availability checking. Supports multiple LLM providers (Google Gemini, DeepSeek).

### Architecture
```
Frontend Chat Widget                    Backend
───────────────────                     ──────────────────────────────
chatWidget.tsx                          AiChatController
  chatBox.tsx                                │
    chatBubble.tsx                      AiChatService
    chatInput.tsx                            │
    chatLoading.tsx                     ┌────┴────────┐
    chatRoomCard(s).tsx                 │             │
                                    KernelProvider  AiContext
                                   (Semantic Kernel)  (side-channel)
                                        │
                            ┌───────────┼──────────────┐
                            ▼                          ▼
                      FaqPlugin              RoomAvailabilityPlugin
                         │                          │
                    FaqRepository              RoomRepository +
                   (faqs.json)              ReservationRepository
```

### API Endpoint

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/chat` | No | Process AI chat message |

### Request/Response

**AiChatRequest:**
```json
{
    "Message": "I need a room for 10 people tomorrow at 2pm",
    "History": [
        { "Content": "Hello", "isUser": true },
        { "Content": "Hi! How can I help?", "isUser": false }
    ]
}
```

**AiChatResponse:**
```json
{
    "Message": "Here are the available rooms...",
    "ActionType": "AvailableRooms",
    "Data": "{ ... serialized room data ... }",
    "RedirectedUrl": "/rooms"
}
```

**AiActionType enum:** `None`, `Faq`, `AvailableRooms`, `RedirectToPage`

### Processing Flow
```
1.  POST /api/chat with AiChatRequest
2.  AiChatService.ProcessMessageAsync():
    a. Validates message is not empty
    b. Builds system prompt (includes FAQ list + current WIB time)
    c. Reconstructs chat history from request.History
    d. Adds current user message
    e. Gets IChatCompletionService from Semantic Kernel
    f. Sets GeminiPromptExecutionSettings with auto-invoke plugins
    g. Calls chatCompletionService.GetChatMessageContentAsync()
       → Semantic Kernel may auto-invoke:
         * FaqPlugin.get_faq_answer(faqId) → returns FAQ answer
         * RoomAvailabilityPlugin.find_available_rooms(startDate, endDate, capacity)
           → returns available rooms as JSON
    h. Checks AiContext for plugin results (side-channel data)
    i. Builds AiChatResponse with message + optional ActionType + Data
```

### System Prompt Design
The AI system prompt is carefully designed to:
1. **Scope restriction** — ONLY answers meeting room booking questions
2. **Reject out-of-scope** — Politely refuses general knowledge questions
3. **FAQ awareness** — Lists all FAQ IDs and questions for matching
4. **Date/time context** — Includes current WIB time for relative date calculations
5. **Function calling** — Guides AI on when to call each plugin

### Multi-Provider Support
```
appsettings.json:
{
    "AI": {
        "Provider": "DeepSeek",    // or "Gemini"
        "Gemini": { "ApiKey": "...", "Model": "gemini-2.5-flash-lite" },
        "DeepSeek": { "ApiKey": "...", "Model": "deepseek-chat", "BaseUrl": "..." }
    }
}
```
- `KernelProvider` reads `AI:Provider` and configures the appropriate connector
- Gemini uses `AddGoogleAIGeminiChatCompletion()`
- DeepSeek uses `AddOpenAIChatCompletion()` (OpenAI-compatible API)

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/AiChatController.cs` | Chat endpoint (46 lines) |
| `Services/AiChatService.cs` | AI chat orchestration (196 lines) |
| `Services/AI/KernelProvider.cs` | Semantic Kernel configuration (96 lines) |
| `Services/AI/AiContext.cs` | Side-channel for plugin results |
| `Services/AI/Plugins/FaqPlugin.cs` | FAQ lookup plugin |
| `Services/AI/Plugins/RoomAvailabilityPlugin.cs` | Room search plugin |
| `Services/AI/FaqRepository.cs` | Loads FAQ data from JSON |
| `Data/faqs.json` | FAQ content |

### Frontend Chat Components
Located in `components/helpDesk/`:
- `chatWidget.tsx` — Floating widget button + container
- `chatBox.tsx` — Chat message display area
- `chatBubble.tsx` — Individual message bubble (user/AI)
- `chatInput.tsx` — Text input field
- `chatLoading.tsx` — Loading animation during AI processing
- `chatRoomCard.tsx` — Room card shown when AI returns room availability
- `chatRoomCardList.tsx` — List of room cards

---

## 10. Google Calendar Integration Module

### Overview
Provides two-way synchronization between the booking system and Google Calendar. Each meeting room has its own Google Calendar. Reservations are created/updated/deleted as Calendar events.

### Architecture
```
                      CalendarServiceBuilder
                        (Authentication)
                              │
                              ▼
                      Google CalendarService (API Client)
                              │
                              ▼
                      GoogleCalendarService (Wrapper)
                        │         │         │
                  InsertEvent  UpdateEvent  DeleteEvent
                  InsertCalendar  ...    DeleteCalendar
```

### Authentication
- Uses a **Google Cloud Service Account** with domain-wide delegation
- Credentials stored in `Secrets/service_secret.json`
- `CalendarServiceBuilder` builds an authenticated `CalendarService` instance at startup
- Registered as `Singleton` in DI container

### Available Operations

| Method | Description |
|--------|-------------|
| `InsertCalendarAsync()` | Create a new Google Calendar (used when creating a room) |
| `DeleteCalendarAsync()` | Delete a Google Calendar (used when deleting a room) |
| `UpdateCalendarAsync()` | Update calendar metadata |
| `GetCalendarAsync()` | Get calendar details |
| `GetCalendarListAsync()` | List all calendars |
| `InsertEventAsync()` | Create an event (used when creating a reservation) |
| `UpdateEventAsync()` | Update an event (used when updating a reservation) |
| `DeleteEventAsync()` | Delete an event (used when canceling a reservation) |
| `GetEventAsync()` | Get event details |
| `GetEventsList()` | List events with sync token support |
| `GetFreeBusyAsync()` | Check free/busy status for multiple calendars |
| `InsertAclAsync()` | Set access control rules on a calendar |

### Calendar API Controller

The `GoogleCalendarController` (`/api/calendar`) provides direct API endpoints for calendar operations, primarily used for debugging and manual sync operations.

### Key Configuration
```json
// Secrets/service_secret.json (structure)
{
    "type": "service_account",
    "client_email": "service-account@project.iam.gserviceaccount.com",
    "private_key": "...",
    "client_id": "...",
    ...
}
```

### Key Files
| File | Purpose |
|------|---------|
| `Services/GoogleCalendarService.cs` | Google Calendar wrapper (267 lines) |
| `Services/CalendarServiceBuilder.cs` | Auth configuration |
| `Controllers/GoogleCalendarController.cs` | Direct calendar API endpoints |

---

## 11. Email Service Module

### Overview
Sends transactional emails using the **Google Gmail API** with a service account. Email templates are HTML-based with inline styling. All email sending is fire-and-forget (non-blocking to the main request).

### Email Types Sent

| Email Type | Trigger | Recipient |
|-----------|---------|-----------|
| Confirmation Email | User registration | New user |
| Reservation Created | New reservation | Reservation owner |
| Reservation Updated | Reservation modified | Reservation owner |
| Room Unavailable | Room deleted with active reservations | Affected users |
| Password Reset | Forgot password request | User |
| Maintenance Alert | Admin action | Targeted users |

### Architecture
```
IEmailService (Interface)
    │
    ├── EmailService (Implementation - uses Gmail API)
    │       │
    │       ▼
    │   GmailService (Google API Client)
    │       │
    │       ▼
    │   GmailAPIServiceBuilder (Auth)
    │
    └── Services that use it:
        ├── AuthService (confirmation, password reset)
        ├── ReservationService (created, updated, room unavailable)
        └── NotificationService (announcements)
```

### Authentication
- Uses the same Google Service Account as Calendar
- `GmailAPIServiceBuilder` builds authenticated `GmailService`
- Uses `GmailService.Scope.GmailSend` scope
- Registered as `Singleton` in DI container

### Key Files
| File | Purpose |
|------|---------|
| `Services/EmailService.cs` | All email sending logic (43,424 bytes — large due to HTML templates) |
| `Services/GmailAPIService.cs` | Gmail API wrapper (22,045 bytes) |
| `Services/GmailAPIServiceBuilder.cs` | Auth configuration |

---

## 12. Push Notification Module

### Overview
Browser push notifications powered by **Firebase Cloud Messaging (FCM)**. The system supports both immediate push and scheduled push via Hangfire background jobs.

### Architecture

```
Frontend (React)                          Backend (.NET 8)
─────────────────                         ──────────────────
Firebase Client SDK                       Firebase Admin SDK
  │                                            │
  ├── Requests FCM token            BrowserNotificationTokenController
  ├── Registers token ───────────►  /api/notification-token/register
  │                                            │
  ├── Receives push notification    NotificationPushService
  │                                    │
  │                            ┌───────┴──────────┐
  │                            │                  │
  │                     Immediate Push    Hangfire Scheduled Job
  │                     (manual trigger)  (NotificationTriggerService)
  │                            │                  │
  │                            ▼                  ▼
  │                     FirebaseMessaging   BrowserNotificationTrigger
  │                     Service             Repository
  │                            │
  └── firebase-messaging-sw.js │
      (Service Worker)         ▼
                         Firebase Cloud Messaging (FCM)
```

### API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/notification-token/register` | Yes | Register FCM token for current user |
| `GET` | `/api/notification-token/user/{userId}` | Yes | Get user's FCM tokens |
| `DELETE` | `/api/notification-token/{tokenId}` | Yes | Remove FCM token |
| `POST` | `/api/notification-push/send` | Yes | Send immediate push notification |

### Scheduled Notification Flow
```
1.  When a reservation is created with ReminderMinutes:
    a. Two BrowserNotificationTrigger records are created:
       - MeetingReminder: ExecuteAt = DateStart - ReminderMinutes
       - MeetingStarted: ExecuteAt = DateStart
       - Status: Pending

2.  Hangfire recurring job runs every minute:
    → NotificationTriggerService.ProcessPendingTriggersAsync()
    a. Queries triggers where Status == Pending AND ExecuteAt <= now
    b. For each trigger:
       - Finds associated reservation and user
       - Gets user's FCM tokens from NotificationBrowser table
       - Sends push notification via Firebase Admin SDK
       - Updates trigger Status to Processed (or Failed)
```

### Key Files
| File | Purpose |
|------|---------|
| `Controllers/BrowserNotificationTokenController.cs` | Token management |
| `Controllers/BrowserNotificationPushController.cs` | Manual push |
| `Services/NotificationPushService.cs` | FCM send logic |
| `Services/NotificationTokenService.cs` | Token CRUD |
| `Services/NotificationTriggerService.cs` | Hangfire job processor |
| `Services/FirebaseMessagingService.cs` | Firebase SDK wrapper |
| `Firebase/FirebaseInitializer.cs` | Firebase Admin SDK init |
| `Helpers/NotificationMessageBuilder.cs` | Push message builder |

---

## 13. Shared Patterns & Utilities

### ServiceResult Pattern
All services return `ServiceResult<T>`, a generic wrapper that standardizes success/failure handling:

```csharp
// Success
return ServiceResult<RoomResponse>.Ok(response);

// Failure
return ServiceResult<RoomResponse>.Fail(ResultType.NotFound, "Room not found");
return ServiceResult<RoomResponse>.Fail(500, "Internal error");
```

**ResultType enum:** `Success`, `BadRequest`, `NotFound`, `Unauthorized`, `Forbid`, `Conflict`, `InternalServerError`, `StatusCode`

Controllers use pattern matching to map `ResultType` to HTTP status codes:
```csharp
if (!result.Success)
    return result.Type switch
    {
        ResultType.BadRequest => BadRequest(result.Message),
        ResultType.NotFound => NotFound(result.Message),
        ResultType.Unauthorized => Unauthorized(result.Message),
        _ => StatusCode(500, result.Message)
    };
return Ok(result.Response);
```

### AutoMapper Configuration
`AutoMapperService` defines all entity-to-DTO mappings:
- `User` ↔ `UserResponse`
- `Room` ↔ `RoomResponse`
- `Reservation` ↔ `ReservationResponse`
- `ReservationRequest` → `Reservation`
- `RegisRequest` → `User`
- `Report` ↔ `ReportResponse`
- `Notification` ↔ `NotificationResponse`

### Dependency Injection Registration (Program.cs)
All services follow this pattern:
```csharp
builder.Services.AddScoped<IReservationRepository, ReservationRepository>();
builder.Services.AddScoped<IReservationService, ReservationService>();
```
- **Repositories**: Scoped
- **Services**: Scoped
- **Google Services**: Singleton (CalendarService, GmailService)
- **Email Service**: Singleton

### Exception Middleware
`ExceptionMiddleware` provides global exception handling:

| Exception | HTTP Status |
|-----------|-------------|
| `ArgumentException` / `ArgumentOutOfRangeException` | 400 Bad Request |
| `KeyNotFoundException` | 404 Not Found |
| `UnauthorizedAccessException` | 401 Unauthorized |
| `InvalidOperationException` | 409 Conflict |
| Any other | 500 Internal Server Error |

### Data Seeding (DbSeed)
On application startup, `DbSeed.Seed()` ensures:
1. Default roles exist (`Administrator`, `Employee`)
2. Default admin user exists
3. Sample rooms, reservations, reports, and notifications exist

### Logging
- **NLog** configured via `nlog.config`
- All logs written to `structured-log.json` in output directory
- Structured logging used throughout: `_logger.LogInformation("Event {@Data}", new { ... })`

### Frontend State Management
Preact Signals are used as reactive state containers. Each manager:
1. Exports `signal()` instances for state
2. Contains async methods that call the API via Axios
3. Updates signals on success

Example pattern:
```typescript
// signals/RoomManager.ts
export const rooms = signal<RoomResponse[]>([]);
export const isLoading = signal<boolean>(false);

export const fetchRooms = async () => {
    isLoading.value = true;
    const response = await axios.get('/api/rooms');
    rooms.value = response.data;
    isLoading.value = false;
};
```

### Frontend Routing & Guards
- `AppRoutes.tsx` defines all routes with `createBrowserRouter`
- `authLoader` — Redirects unauthenticated users to `/login`
- `adminLoader` — Redirects non-admin users to `/` (home)
- Token stored in `sessionStorage.accessToken`
- JWT decoded client-side via `jwt-decode` for role checking

---

