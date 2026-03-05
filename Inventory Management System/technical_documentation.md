# Inventory Management System — Technical Documentation

> **Purpose**: This document provides a comprehensive, module-by-module technical reference for new programmers joining the project. Each module section explains the architecture, data flow, key classes, methods, and how they interact.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Module: Models & DTOs](#2-module-models--dtos)
3. [Module: Persistence (Data Access Layer)](#3-module-persistence-data-access-layer)
4. [Module: Item Management](#4-module-item-management)
5. [Module: Borrow Request Lifecycle](#5-module-borrow-request-lifecycle)
6. [Module: Active Request Management](#6-module-active-request-management)
7. [Module: Reservation System](#7-module-reservation-system)
8. [Module: Item Propose (New Item Proposals)](#8-module-item-propose-new-item-proposals)
9. [Module: Category & SubCategory Management](#9-module-category--subcategory-management)
10. [Module: User Management](#10-module-user-management)
11. [Module: Notification & Real-Time (SignalR)](#11-module-notification--real-time-signalr)
12. [Module: Dashboard & Analytics](#12-module-dashboard--analytics)
13. [Module: Audit Log](#13-module-audit-log)
14. [Module: Authentication & Authorization](#14-module-authentication--authorization)
15. [Module: File Import/Export (CSV & Excel)](#15-module-file-importexport-csv--excel)
16. [Module: Restock Management](#16-module-restock-management)
17. [Module: Utilities & Shared Services](#17-module-utilities--shared-services)
18. [Dependency Injection Registration](#18-dependency-injection-registration)
19. [Routing & Middleware Pipeline](#19-routing--middleware-pipeline)

---

## 1. Architecture Overview

### Solution Structure

```
InventoryManagementSystem.sln
├── InventoryManagementSystem/             ← Main MVC Web App
├── InventoryManagementSystem.Models/      ← Entity Models, Enums, DTOs
├── InventoryManagementSystem.Persistence/ ← DbContext, Migrations, Repositories
├── InventoryManagementSystem.Tests/       ← Unit Tests
└── InventoryManagementSystem.Utilities/   ← Helper/Utility classes
```

### Dependency Graph

```
Main App ──→ Models
Main App ──→ Persistence ──→ Models
Main App ──→ Utilities
Tests    ──→ All projects
```

### Layered Architecture Pattern

Every feature follows this **strict layered pattern**:

```
Controller (HTTP handling, routing, TempData)
    ↓ calls
Service (business logic, validation, mapping)
    ↓ calls
Repository (data access via EF Core)
    ↓ queries
ApplicationDbContext (EF Core DbSets)
    ↓ connects to
SQLite Database
```

**Key Rule**: Controllers NEVER access repositories directly. All business logic lives in services.

### Area-Based Separation

The application uses ASP.NET Areas to separate features by role:

```
Areas/
├── Admin/       ← [Authorize(Roles = "Admin")] on all controllers
│   ├── Controllers/  (16 controllers)
│   ├── Services/     (21 services + 21 interfaces)
│   ├── ViewComponents/
│   └── Views/
├── Employee/    ← [Authorize(Roles = "Employee")] on all controllers
│   ├── Controllers/  (10 controllers)
│   ├── Services/     (9 services + 9 interfaces)
│   └── Views/
└── Identity/    ← Authentication (Login, Register)
    └── Pages/
```

---

## 2. Module: Models & DTOs

### 2.1 Entity Models

Located in `InventoryManagementSystem.Models/Models/`.

#### Core Entities

| Entity | PK | Description |
|---|---|---|
| `Item` | `ItemId` (Guid) | Core inventory item with code, name, quantity, type, category, city |
| `Category` | `CategoryId` (Guid) | E.g., Electronics, Stationery |
| `SubCategory` | `SubCategoryId` (Guid) | Child of Category |
| `ItemType` | `Id` (int) | 1 = Borrowable, 2 = Consumable |
| `City` | `CityId` (Guid) | Location where item resides |
| `User` | `Id` (string) | Extends `IdentityUser` with EmployeeCode, FirstName, LastName, IsDeleted |

#### Request Lifecycle Entities

| Entity | PK | Description |
|---|---|---|
| `Request` | `RequestId` (Guid) | Borrow request submitted by employee |
| `ActiveRequest` | `ActiveRequestId` (Guid) | Approved request currently being tracked |
| `ItemHistory` | `ItemHistoryId` (Guid) | Record after item is returned |
| `Reservation` | `ReservationId` (Guid) | Future date reservation for an item |

#### Supporting Entities

| Entity | PK | Description |
|---|---|---|
| `ItemView` | `ItemViewId` (Guid) | Custom item view configuration |
| `ItemPropose` | `ItemProposeId` (Guid) | Employee proposal for new item |
| `ItemStatus` | `Id` (int) | Lookup: WellReturned(1), Damaged(2), Lost(3), Consumed(4) |
| `Restock` | `RestockId` (Guid) | Restock request record |
| `RestockStatus` | `Id` (int) | Restock status lookup |
| `Notification` | `NotificationId` (int) | Push notification record |
| `AuditLog` | `ActivityId` (Guid) | Activity log entry |
| `ProposeLimit` | — | Max proposals per employee |
| `ChartView` | `ChartViewId` | Dashboard chart config |

### 2.2 Item Entity — Deep Dive

```csharp
public class Item
{
    Guid ItemId              // Primary key
    string ItemCode          // Auto-generated unique code (e.g., "ELC-LAP-001")
    string? ItemName
    string? Description
    DateTime CreateAt
    string? PicturePath      // Item photo file path
    bool Availability        // Is currently available
    int? Quantity            // Stock count (for consumables)
    int? QuantityLow         // Low-stock threshold for restock alerts
    bool IsDeleted           // Soft delete flag
    int TypeId               // FK → ItemType (1=Borrowable, 2=Consumable)
    Guid CategoryId          // FK → Category
    Guid SubCategoryId       // FK → SubCategory
    Guid CityId              // FK → City

    // Navigation Properties
    ItemType Type
    Category Category
    SubCategory SubCategory
    City City
    ICollection<ItemView> ItemViews
    ICollection<Request> Requests
    ICollection<ActiveRequest> ActiveRequests
    ICollection<ItemHistory> ItemHistories
    ICollection<Restock> Restocks
    ICollection<Reservation> Reservations
}
```

### 2.3 Enums

Located in `InventoryManagementSystem.Models/Enum/`.

| Enum | Values | Usage |
|---|---|---|
| `ItemTypeEnum` | All(0), Borrowable(1), Consumable(2) | Filter items by type |
| `RequestStatusEnum` | WaitingApproval(0), Approved(1), Rejected(2), Returned(3) | Track request state |
| `ActiveRequestStatusEnum` | WaitingPickUp(0), InUser(1), Late(2), Returned(3), Cancelled(4) | Track active borrow state |
| `ReservationStatusEnum` | WaitingApproval(0), Approved(1), Rejected(2), Cancelled(3) | Track reservation state |
| `ProposeStatusEnum` | WaitingApproval(0), Approved(1), Rejected(2) | Track proposal state |
| `ItemStatusEnum` | WellReturned(1), Damaged(2), Lost(3), Consumed(4) | Item condition after return |
| `InputSourceEnum` | MainPage(1), DetailsPage(2) | Identify which page submitted data |

### 2.4 DTOs (Data Transfer Objects)

Located in `InventoryManagementSystem.Models/ModelsDTO/`. DTOs prevent leaking entity details to views.

| DTO Folder | Count | Purpose |
|---|---|---|
| `ActiveRequestDTOs/` | 11 | Filter, list, detail DTOs for active requests |
| `AuditLogDTOs/` | 2 | Display audit log entries |
| `CategoryDTOs/` | 7 | CRUD categories |
| `ChartDTOs/` | 1 | Dashboard chart data |
| `ItemViewDTOs/` | 8 | CRUD item views, filters |
| `ItemHistoryDTOs/` | 8 | Return history records |
| `ItemProposeDTOs/` | 12 | CRUD proposals |
| `NotificationDTOs/` | 3 | Notification payloads |
| `ProposeLimitDTOs/` | 1 | Proposal limit config |
| `RequestDTOs/` | 9 | Request CRUD & filters |
| `ReservationDTOs/` | 4 | Reservation CRUD |
| `SubCategoryDTOs/` | 10 | CRUD sub-categories |
| `UserViewDTOs/` | 6 | User management views |

### 2.5 AutoMapper Profiles

Located in `InventoryManagementSystem/Mappers/`. 16 profiles bridge Entity ↔ DTO:

```
ActiveRequestMappingProfile    ItemViewMappingProfile
AdminRequestMappingProfile     NotificationMappingProfile
AuditLogMappingProfile         ProposeLimitMappingProfile
CategoryMappingProfile         RequestMappingProfile
CityMappingProfile             ReservationMappingProfile
ItemHistoryMappingProfile      SubCategoryMappingProfile
ItemProposeMappingProfile      UserViewMappingProfile
ItemTypeMappingProfile         Mapper (manual helper)
```

**Convention**: Each mapper is registered in `Program.cs` via:
```csharp
builder.Services.AddAutoMapper(typeof(XxxMappingProfile));
```

---

## 3. Module: Persistence (Data Access Layer)

### 3.1 ApplicationDbContext

**File**: `InventoryManagementSystem.Persistence/Data/ApplicationDbContext.cs`

Inherits `IdentityDbContext<User>` (ASP.NET Identity). Contains **18 DbSets**:

```csharp
DbSet<Item> Items
DbSet<ItemType> Types
DbSet<Category> Categories
DbSet<SubCategory> SubCategories
DbSet<City> Cities
DbSet<ItemView> ItemViews
DbSet<ItemStatus> ItemStatuses
DbSet<RestockStatus> RestockStatuses
DbSet<Request> Requests
DbSet<ActiveRequest> ActiveRequests
DbSet<ItemHistory> ItemHistories
DbSet<Restock> Restocks
DbSet<Notification> Notifications
DbSet<Reservation> Reservations
DbSet<ItemPropose> ItemProposes
DbSet<ProposeLimit> ProposeLimit
DbSet<AuditLog> Auditlogs
DbSet<ChartView> ChartViews
```

**Fluent API Configuration** in `OnModelCreating()`:
- Primary keys, unique indexes (`ItemCode`, `CategoryCode`, `SubCategoryCode`, `EmployeeCode`, `CityName`)
- Foreign key relationships with cascade behaviors
- Default values and nullable property settings

### 3.2 Generic Repository Pattern

**File**: `InventoryManagementSystem.Persistence/Repository/Repository.cs`

```csharp
public class Repository<T> : IRepository<T> where T : class
{
    // Core Methods:
    Task AddAsync(T entity)              // Insert + SaveChanges
    Task AddRangeAsync(IEnumerable<T>)   // Bulk insert
    Task<T?> GetByIdAsync(Guid id)       // Find by PK
    Task<T?> GetFirstAsync(Expression<Func<T, bool>>, string? includeProperties)
    Task<IEnumerable<T>> GetAllAsync(string? includeProperties)
    Task<IEnumerable<T>> GetWhereAsync(Expression<Func<T, bool>>, string? includeProperties)
    Task UpdateAsync(T entity)           // Update + SaveChanges
    Task SaveAsync()                     // SaveChanges only
    Task RemoveAsync(T entity)           // Delete + SaveChanges
    Task RemoveRangeAsync(IEnumerable<T>)
}
```

**Include Properties Pattern**: Navigation properties are loaded via comma-separated string:
```csharp
// Example usage in services:
await _repo.GetFirstAsync(r => r.Id == id, includeProperties: "Item,User,Category");
```

### 3.3 Entity-Specific Repositories

Each extends `Repository<T>` with custom query methods:

| Repository | Key Custom Methods |
|---|---|
| `ItemRepository` | `ReduceItemQuantityAsync()`, `RestoreItemQuantityAsync()`, item code generation queries |
| `RequestRepository` | `GetRequestsByStatusAsync()`, `GetRequestWithDetailsAsync()`, `ApplyFilters()` |
| `ActiveRequestRepository` | `GetAllActiveRequestsAsync()`, `CreateActiveRequestAsync()`, `GetMonthlyCountsAsync()` |
| `ItemViewRepository` | `GetItemViewWithDetailsAsync()`, filtered queries with navigation properties |
| `CategoryRepository` | `GetActiveCategoriesAsync()`, `GetCategoryWithSubCategoriesAsync()` |
| `SubCategoryRepository` | `GetSubCategoriesByCategoryAsync()`, code update methods |
| `ReservationRepository` | `GetWaitingApprovalAsync()`, `GetMonthlyCountsAsync()` |
| `ItemProposeRepository` | `GetWaitingApprovalAsync()`, `CountAllProposesAsync()`, `GetMonthlyCountsAsync()` |
| `NotificationRepository` | `MarkAsReadAsync()`, `MarkAllAsReadAsync()`, `GetPagedNotificationsAsync()` |
| `AuditLogRepository` | `GetAuditLogsForUserAsync()`, `GetAllAuditLogsAsync()` |
| `ItemHistoryRepository` | `GetItemHistoryAsync()`, `GetLostItemsAsync()`, `CountAllReturnAsync()` |
| `UserViewRepository` | User-specific view queries |

### 3.4 Database Seeding

**`DbInitializer`**: Seeds initial data on startup:
- Roles: Admin, Employee
- Default admin account (from `appsettings.json` → `AdminDetails`)
- Item types (Borrowable, Consumable)
- Item statuses (WellReturned, Damaged, Lost, Consumed)
- Default cities

**`SeedDataService`**: Admin-accessible URL endpoints to generate/clear dummy data:
```
GET /Admin/SeedData/GenerateDummyData
GET /Admin/SeedData/ClearDummyData
```

---

## 4. Module: Item Management

### 4.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                   ADMIN SIDE                         │
│ ItemViewController → IItemViewAdminService           │
│   (CRUD, Upload,     → ItemViewAdminService          │
│    Soft Delete,        → IItemViewRepository         │
│    Activate)           → IItemRepository             │
│                        → ICategoryRepository         │
│                        → ISubCategoryRepository      │
│                        → ICityRepository             │
│                        → IItemTypeRepository         │
│                        → ICsvService / IExcelService  │
├─────────────────────────────────────────────────────┤
│                  EMPLOYEE SIDE                       │
│ ItemViewController → IItemViewEmployeeService        │
│   (Browse, Request,   → ItemViewEmployeeService      │
│    Reserve)            → IItemViewRepository         │
│                        → IRequestRepository          │
│                        → IReservationRepository      │
│                        → INotificationModule         │
└─────────────────────────────────────────────────────┘
```

### 4.2 Admin — ItemViewController

**File**: `Areas/Admin/Controllers/ItemViewController.cs`
**Authorization**: `[Authorize(Roles = "Admin")]`

| Action | HTTP | Description |
|---|---|---|
| `Index` | GET | List all active items with filtering & pagination |
| `IndexDeleted` | GET | List soft-deleted items |
| `Details` | GET | View item details |
| `Create` | GET/POST | Create new item with image upload |
| `Edit` | GET/POST | Edit existing item |
| `Delete` / `DeleteAsync` | GET/POST | Soft delete an item (sets `IsDeleted = true`) |
| `Activate` / `ActivateAsync` | GET/POST | Restore a soft-deleted item |
| `GetSubcategoriesByCategory` | GET | AJAX endpoint: returns subcategories JSON for a category |
| `Upload` / `UploadFile` | GET/POST | Import items via CSV/Excel file |

### 4.3 Admin — ItemViewAdminService

**File**: `Areas/Admin/Services/ItemViewAdminService.cs` (640 lines)

**Key Methods**:

| Method | What It Does |
|---|---|
| `GetItemViewIndexAsync()` | Builds the index page model: applies filters (type, category, search, city), fetches dropdown data, paginates results |
| `GetDeletedItemViewIndexAsync()` | Same as above but for soft-deleted items |
| `CreateItemAsync()` | Validates via FluentValidation → generates ItemCode → creates Item + ItemView → saves image file |
| `UpdateItemAsync()` | Validates → updates Item + ItemView → handles image replacement |
| `DeleteItemAsync()` | Soft deletes (sets `IsDeleted = true` on both Item and ItemView) |
| `ActivateItemAsync()` | Restores soft-deleted item |
| `ProcessFileUploadAsync()` | Delegates to `CsvService` or `ExcelService` based on file type |
| `SetupViewBagAsync()` | Populates ViewBag with dropdown data (categories, types, cities, pagination) |
| `CheckLowStockAsync()` | Checks if item quantity is below `QuantityLow` threshold |

**Data Flow for Creating an Item**:
```
1. Controller receives ItemViewCreateDTO
2. Service validates with FluentValidation
3. If valid:
   a. ItemCodeService generates unique code (e.g., "ELC-LAP-001")
   b. AutoMapper maps DTO → Item entity
   c. Image file saved to wwwroot/images/
   d. ItemRepository.AddAsync() persists to DB
   e. ItemViewRepository.AddAsync() creates view record
4. If invalid: returns model with validation errors
```

### 4.4 Employee — ItemViewEmployeeService

**File**: `Areas/Employee/Services/ItemViewEmployeeService.cs` (353 lines)

| Method | What It Does |
|---|---|
| `GetItemViewIndexAsync()` | Browse available items with filters |
| `GetItemViewDetailsAsync()` | View single item details |
| `GetRequestModelAsync()` | Prepare request form model |
| `CreateRequestAsync()` | Check existing requests/reservations → create new Request → send notification to admins → log audit |
| `GetReserveModelAsync()` | Prepare reservation form model |
| `CreateReservationAsync()` | Validate dates → create Reservation → notify admins |
| `CheckExistingRequestAsync()` | Prevent duplicate requests for same item |
| `CheckExistingReservationAsync()` | Prevent duplicate reservations |

### 4.5 Item Code Generation

**File**: `Areas/Admin/Services/ItemCodeService.cs`

Auto-generates unique item codes based on Category + SubCategory abbreviations:
```
Format: {CategoryCode}-{SubCategoryCode}-{SequenceNumber}
Example: ELC-LAP-001 (Electronics → Laptop → 1st item)
```

---

## 5. Module: Borrow Request Lifecycle

### 5.1 Complete Lifecycle Flow

```
Employee browses items (ItemViewEmployeeService)
         ↓
Employee creates Request (status: WaitingApproval)
         ↓ notification sent to Admin
Admin reviews request (RequestAdminService)
         ↓
    ┌────┴────┐
    ↓         ↓
 APPROVE    REJECT
    ↓         ↓
 Creates    Email sent
 ActiveRequest  to employee
 (WaitingPickUp) with reason
    ↓
 Item quantity
 reduced
    ↓
 Email sent
 to employee
```

### 5.2 Admin — RequestController

**File**: `Areas/Admin/Controllers/RequestController.cs`

| Action | HTTP | Description |
|---|---|---|
| `WaitingApproval` | GET | List pending requests with pagination |
| `Approved` | GET | List approved requests |
| `Rejected` | GET | List rejected requests |
| `Details` | GET | View request details |
| `Approve` | POST | Approve a request → creates ActiveRequest |
| `AdminNoteRejection` | POST | Reject with admin note |
| `Cancel` | GET | Cancel a request |
| `RemoveExpiredRejectedRequests` | POST | Bulk cleanup of old rejected requests |
| `RedirectItemView` | GET | Navigate to the requested item's details |

### 5.3 Admin — RequestAdminService

**File**: `Areas/Admin/Services/RequestAdminService.cs` (372 lines)

**Dependencies**: `IRequestRepository`, `IActiveRequestRepository`, `IItemRepository`, `INotificationModule`, `IEmailSender`, `IAuditLogAdminService`, `IPaginationService`

**Key Methods**:

| Method | Flow |
|---|---|
| `ApproveRequestAsync(requestId, adminUserId)` | 1. Fetch request with item details → 2. Set status = Approved → 3. Reduce item quantity → 4. Call `CreateNewActiveRequest()` → 5. Send approval email → 6. Create notification → 7. Check low stock alert → 8. Log audit |
| `RejectRequestAsync(requestId, adminNote, adminUserId)` | 1. Fetch request → 2. Set status = Rejected, save admin note → 3. Send rejection email → 4. Create notification → 5. Log audit |
| `CancelRequestAsync(requestId, status)` | 1. Fetch request → 2. If was Approved, restore item quantity → 3. Delete associated ActiveRequest → 4. Delete the Request → 5. Notify user |
| `CreateNewActiveRequest(request)` | Maps Request → ActiveRequest with status WaitingPickUp |
| `CheckAndNotifyLowStock(item)` | If quantity ≤ QuantityLow → send alert notification to admins |
| `GetRequestsByStatusAsync(status, filter)` | Fetch + filter + paginate requests by status |

### 5.4 Employee — RequestController & Service

**Employee Controller Actions**: `WaitingApproval`, `Rejected`, `Approved`, `Details`

**RequestEmployeeService** methods:
- `GetWaitingApprovalRequestsAsync()` — Employee's own pending requests
- `GetRejectedRequestsAsync()` — Employee's rejected requests
- `GetApprovedRequestsAsync()` — Employee's approved requests
- `GetRequestDetailsAsync()` — Detailed view with item info
- `GetRequestsByStatusAsync()` — Core filter + paginate logic

**Key Difference**: Employee services always filter by `userId` to show only their own data.

---

## 6. Module: Active Request Management

### 6.1 Active Request State Machine

```
WaitingPickUp ──→ InUser ──→ Returned
     │               │          ↓
     ↓               ↓      ItemHistory created
  Cancelled        Late       (WellReturned/Damaged/Lost/Consumed)
                     │
                     ↓
                  Returned
```

### 6.2 Admin — ActiveRequestController

**File**: `Areas/Admin/Controllers/ActiveRequestController.cs` (203 lines)

| Action | HTTP | Description |
|---|---|---|
| `Index` | GET | List active requests filtered by status |
| `Update` | GET | Transition status (e.g., WaitingPickUp → InUser) |
| `Details` | GET/POST | View/update pickup notes |
| `NotifyUser` | GET | Send email notification to late borrower |
| `PickUpDeskIndex` | GET | QR-code based pickup desk view |
| `ReturnDeskIndex` | GET | QR-code based return desk view |
| `BackToList` | GET | Navigate back with preserved filters |

### 6.3 Admin — ActiveRequestService

**File**: `Areas/Admin/Services/ActiveRequestService.cs` (413 lines)

**State Transition Handlers**:

| Method | Transition | What Happens |
|---|---|---|
| `HandleInUserStatus()` | WaitingPickUp → InUser | Set pickup date, mark as in user, update item availability |
| `HandleLateStatus()` | InUser → Late | Flag as overdue, potential email notification |
| `HandleReturnedStatus()` | InUser/Late → Returned | Create `ItemHistory` record, restore item quantity, set return date |
| `HandleCancelledStatus()` | Any → Cancelled | Restore item quantity, cancel the active request |

**QR Code Desk Operations**:
- `FindActiveRequestForPickupAsync(scannedData)` — Lookup by QR code for pickup
- `FindActiveRequestForReturnAsync(scannedData)` — Lookup by QR code for return
- `ValidateScannedData(scannedData, found)` — Validate QR scan result

**Other Key Methods**:
- `GetActiveRequestsAsync()` — Filter + paginate with search
- `UpdateActiveRequestStatusAsync()` — Dispatch to appropriate handler based on target status
- `CreateNewItemHistoryAsync()` — Records item condition after return
- `SendNotificationEmailAsync()` — Send late-return emails

### 6.4 Employee — EmployeeActiveRequestService

**File**: `Areas/Employee/Services/EmployeeActiveRequestService.cs` (284 lines)

Similar to admin service but scoped to the logged-in employee. Key addition:
- `GenerateQRCode()` — Generates QR code for the active request using `QRCoder` library

---

## 7. Module: Reservation System

### 7.1 Reservation Flow

```
Employee selects item + date (ItemViewEmployeeService.CreateReservationAsync)
         ↓
Reservation created (status: WaitingApproval)
         ↓ notification to admins
Admin reviews (ReservationAdminService)
         ↓
    ┌────┴────┐
    ↓         ↓
 APPROVE    REJECT
    ↓         ↓
 Creates    Admin note
 ActiveRequest  saved, user
 (WaitingPickUp) notified
    ↓
 Item quantity reduced
```

### 7.2 Admin — ReservationAdminService

**File**: `Areas/Admin/Services/ReservationAdminService.cs` (163 lines)

| Method | What It Does |
|---|---|
| `GetReservationsAsync(status, filter)` | Fetch reservations by status with search + pagination |
| `ApproveReservationAsync(id)` | Validate stock → Set Approved → Create ActiveRequest → Reduce quantity → Notify user with pickup date |
| `RejectReservationAsync(id, adminNote)` | Set Rejected → Save reason → Notify user |
| `CreateNewActiveRequestAsync(reservation)` | Maps Reservation → ActiveRequest with pickup date from reservation |

### 7.3 Employee — ReservationEmployeeService

**File**: `Areas/Employee/Services/ReservationEmployeeService.cs` (116 lines)

| Method | What It Does |
|---|---|
| `GetReservationsAsync(filter, userId)` | Employee's own reservations with search + pagination |
| `CancelReservationAsync(id)` | Only if WaitingApproval → set Cancelled → notify → audit log |

---

## 8. Module: Item Propose (New Item Proposals)

### 8.1 Proposal Flow

```
Employee submits proposal (ItemProposeService.CreateItemProposeAsync)
  • Checks propose limit (ProposeLimit entity)
  • Uploads image
  • Creates ItemPropose (status: WaitingApproval)
  • Notifies admins
         ↓
Admin reviews (ItemProposeAdminService)
         ↓
    ┌────┴────┐
    ↓         ↓
 APPROVE    REJECT
    ↓         ↓
 Creates    Admin note
 new Item   saved, user
 + ItemView notified
    ↓
 Image copied from
 proposal to item
```

### 8.2 Admin — ItemProposeAdminService

**File**: `Areas/Admin/Services/ItemProposeAdminService.cs` (448 lines)

| Method | What It Does |
|---|---|
| `GetAdminProposePagedAsync()` | List proposals with filters (status, search) and pagination |
| `GetItemProposeDetailAsync()` | Detailed view of a proposal |
| `ApproveItemProposeAsync()` | Set Approved → call `ApproveItemPropose()` to create actual Item |
| `ApproveItemPropose()` | Generate ItemCode → create Item + ItemView → handle image copy |
| `HandleImageApprovalAsync()` | Copy proposal image to item's image folder |
| `RejectItemProposeAsync()` | Set Rejected → save admin note → notify employee |
| `GetProposeInformatationAsync()` | Get proposal info with category/subcategory details |

### 8.3 Employee — ItemProposeService

**File**: `Areas/Employee/Services/ItemProposeService.cs` (313 lines)

| Method | What It Does |
|---|---|
| `CreateItemProposeAsync()` | Check limit → build entity → upload image → save → notify admins → audit |
| `CanUserProposeAsync(userId)` | Check if user hasn't exceeded `ProposeLimit` |
| `GetUserProposalsPagedAsync()` | Employee's own proposals with pagination |
| `GetUserItemProposeDetailAsync()` | Detailed view of own proposal |
| `UploadImageAsync()` | Save image to `wwwroot/pictures/` with unique filename |

---

## 9. Module: Category & SubCategory Management

### 9.1 Admin — CategoryAdminService

**File**: `Areas/Admin/Services/CategoryAdminService.cs` (24,878 bytes)

Handles CRUD + soft delete for Categories. Also manages the Category ↔ SubCategory relationship.

| Method | What It Does |
|---|---|
| `GetCategoriesWithModelAsync()` | List categories with pagination, supports deleted/active filter |
| `GetSubCategoriesWithModelAsync()` | List subcategories under categories |
| `CreateCategoryAsync()` | Validate uniqueness → create Category with auto-generated code |
| `UpdateCategoryAsync()` | Update name/code → cascade update item codes |
| `DeleteCategoryAsync()` | Soft delete category + all child subcategories |
| `ActivateCategoryAsync()` | Restore soft-deleted category |
| `GetCategoryForEditAsync()` / `GetCategoryForDeleteAsync()` | Fetch entity for forms |

### 9.2 Admin — SubCategoryAdminService

**File**: `Areas/Admin/Services/SubCategoryAdminService.cs` (395 lines)

| Method | What It Does |
|---|---|
| `GetSubCategoriesWithPaginationAsync()` | List subcategories with pagination |
| `CreateSubCategoryAsync()` | Validate → create with auto code → link to parent category |
| `EditSubCategoryAsync()` | Update → cascade update item codes via `UpdateSubCategoryCodeInItemCodeAsync()` |
| `DeleteSubCategoryAsync()` | Soft delete |
| `ActivateSubCategoryAsync()` | Restore if parent category is also active |
| `ValidateSubCategoryAsync()` | Check name uniqueness within category |

**Item Code Cascading**: When a category or subcategory code changes, all associated item codes are updated:
```
UpdateSubCategoryCodeInItemCodeAsync():
  1. Find all items with old subcategory code
  2. Replace the subcategory portion of ItemCode
  3. Bulk update items
```

---

## 10. Module: User Management

### 10.1 Admin — UserViewService

**File**: `Areas/Admin/Services/UserViewService.cs` (622 lines)

**Dependencies**: `UserManager<User>`, `IUserViewRepository`, `IEmailSender`, `IValidator<UserCreateDto>`, `IValidator<UserEditDto>`

| Method | What It Does |
|---|---|
| `CreateUserWithEmailConfirmationAsync()` | Validate → create user via Identity → assign role → send confirmation email |
| `CreateUserAsync()` | Core user creation with Identity's `CreateAsync()` + role assignment |
| `UpdateUserAsync()` | Update user fields + handle role changes + email changes |
| `DeleteUserAsync()` | Soft delete (set `IsDeleted = true`) |
| `ActivateUserAsync()` | Restore soft-deleted user |
| `PermanentDeleteUserAsync()` | Hard delete from database |
| `GetPaginatedUsersAsync()` | List users with search + role filter + pagination |
| `GetFilteredUsersListAsync()` | Apply search string + role filter to user query |
| `SendEmailConfirmationAsync()` | Generate confirmation token → build URL → send via SMTP |
| `IsValidEmailAsync()` | Check email uniqueness and format |

**User Creation Flow**:
```
1. FluentValidation validates UserCreateDto
2. Check email uniqueness
3. UserManager.CreateAsync() creates Identity user
4. Set employee-specific fields (EmployeeCode, FirstName, LastName)
5. UserManager.AddToRoleAsync() assigns role
6. Generate email confirmation token
7. Build confirmation URL
8. Send confirmation email via IEmailSender
```

---

## 11. Module: Notification & Real-Time (SignalR)

### 11.1 Architecture

```
Service Layer (any service)
    ↓ calls
INotificationModule.Create(message, receiverId)
    ↓
NotificationModule
    ├── Saves to DB (Notification entity)
    └── Pushes via SignalR (IHubContext<NotificationHub>)
            ↓
        Client browser receives "ReceiveNotification" event
```

### 11.2 NotificationModule

**File**: `Module/NotificationModule.cs` (71 lines)

```csharp
public class NotificationModule : INotificationModule
{
    // Create notification + push via SignalR
    async Task Create(string message, string? receiverId)
    {
        // 1. Save to database
        var notification = new Notification { Message, UserId, TimeStamp, IsRead = false };
        _context.Notifications.Add(notification);
        await _context.SaveChangesAsync();

        // 2. Push via SignalR
        if (receiverId != null)
            // Send to specific user
            await _hubContext.Clients.User(receiverId).SendAsync("ReceiveNotification", message);
        else
            // Send to all admins (for employee-initiated actions)
            foreach admin: await _hubContext.Clients.User(admin.Id).SendAsync(...)
    }

    // Query methods
    Task<IEnumerable<Notification>> GetAllNotificationsByUserId(string? userId)
    Task<IEnumerable<Notification>> GetAllNotifications()
    Task<int> GetUnreadNotificationCountAsync(string? userId)
}
```

### 11.3 NotificationHub

**File**: `Hubs/NotificationHub.cs`

Simple SignalR hub that clients connect to for receiving real-time notifications. Mapped in `Program.cs`:
```csharp
app.MapHub<NotificationHub>("/notificationHub");
```

### 11.4 NotificationBellViewComponent

**File**: `ViewComponents/NotificationBellViewComponent.cs`

Renders the bell icon with unread count badge. Invoked in the layout:
```html
@await Component.InvokeAsync("NotificationBell")
```

---

## 12. Module: Dashboard & Analytics

### 12.1 Admin Dashboard — HomeAdminService

**File**: `Areas/Admin/Services/HomeAdminService.cs` (144 lines)

`GetViewModelsAsync()` aggregates data from multiple repositories:

| Data Point | Source |
|---|---|
| Total proposals + waiting approval | `IItemProposeRepository` |
| Total reservations + waiting approval | `IReservationRepository` |
| Total requests + waiting approval | `IRequestAdminRepository` |
| Total active requests | `IActiveRequestRepository` |
| Total returned + lost items | `IItemHistoryRepository` |
| Monthly trend chart data | `GetChartDataAsync()` |

**Chart Data** (`GetChartDataAsync(monthCount)`):
- Queries monthly counts for proposals, reservations, requests, active requests
- Returns labels (month names) + value arrays for chart rendering
- Default period configured via `ChartView` entity

### 12.2 Employee Dashboard — HomeEmployeeService

Similar but scoped to the logged-in employee's data.

### 12.3 ChartViewService

**File**: `Areas/Admin/Services/ChartViewService.cs`
Manages dashboard chart configuration (how many months to display).

---

## 13. Module: Audit Log

### 13.1 How Audit Logging Works

Audit logs are created by services when significant actions occur.

**Admin side** — `AuditLogAdminService`:
```csharp
await _auditLogAdminService.AddAdminAuditLogAsync(
    "Approved Request: Laptop for John Doe"
);
```

**Employee side** — `EmployeeAuditLogService`:
```csharp
await _auditLogService.AddEmployeeAuditLogAsync(
    "Cancelled reservation for item Laptop"
);
```

### 13.2 AuditLog Entity

```csharp
public class AuditLog
{
    Guid ActivityId     // PK
    string Activity     // Description of the action
    string UserId       // Who performed it
    DateTime Timestamp  // When
    User User           // Navigation property
}
```

### 13.3 Viewing Audit Logs

- **Admin**: `AuditLogController` → `AuditLogAdminService` → sees ALL logs with search + pagination
- **Employee**: `AuditLogController` → `EmployeeAuditLogService` → sees only THEIR OWN logs

---

## 14. Module: Authentication & Authorization

### 14.1 Identity Setup

- **Framework**: ASP.NET Core Identity with `IdentityDbContext<User>`
- **Database**: SQLite backend
- **Roles**: `Admin` and `Employee` (defined in `AllRoles` constants)

### 14.2 Google OAuth

Configured in `Program.cs`:
```csharp
builder.Services.AddAuthentication()
    .AddGoogle(options => {
        options.ClientId = config["Authentication:Google:ClientId"];
        options.ClientSecret = config["Authentication:Google:ClientSecret"];
    });
```

### 14.3 Authorization Enforcement

Every controller is protected with area + role attributes:
```csharp
[Area(AllRoles.Role_Admin)]
[Authorize(Roles = AllRoles.Role_Admin)]
public class ItemViewController : Controller { ... }
```

### 14.4 Session Configuration

```csharp
builder.Services.AddSession(options => {
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.MaxAge = TimeSpan.FromMinutes(60);
});
```

### 14.5 Email Confirmation

On registration, a confirmation email is sent:
1. `UserManager.GenerateEmailConfirmationTokenAsync()` creates token
2. URL is built with `Url.Page("/Account/ConfirmEmail", ...)`
3. `IEmailSender.SendEmailAsync()` sends via SMTP
4. HTML template: `EmailTemplates/EmailConfirmation.html`

### 14.6 Validators

| Validator | File | Purpose |
|---|---|---|
| `UserCreateValidator` | `Validators/UserCreateValidator.cs` | Validate new user fields |
| `UserEditValidator` | `Validators/UserEditValidator.cs` | Validate user edit fields |
| `RegisterInputModelValidator` | `Validators/RegisterInputModelValidator.cs` | Validate registration form |
| `ItemViewValidators` | `Validators/ItemViewValidators/` | Validate item CRUD (3 validators) |
| `ItemProposeValidators` | `Validators/ItemProposeValidators/` | Validate proposals (2 validators) |
| `ProposeLimitValidators` | `Validators/ProposeLimitValidators/` | Validate limit settings |
| `ReservationViewValidators` | `Validators/ReservationViewValidators/` | Validate reservations |

---

## 15. Module: File Import/Export (CSV & Excel)

### 15.1 CsvService

**File**: `Areas/Admin/Services/CsvService.cs`
- Uses **CsvHelper** library
- Parses CSV file → maps columns to Item properties → bulk creates items

### 15.2 ExcelService

**File**: `Areas/Admin/Services/ExcelService.cs`
- Uses **ExcelDataReader** library
- Reads .xlsx/.xls files → extracts rows → maps to Item properties → bulk creates items

### 15.3 FileImportService

**File**: `Areas/Admin/Services/FileImportService.cs`
- Orchestrates file upload processing
- Determines file type (CSV vs Excel) → delegates to appropriate service

### 15.4 Import Flow

```
ItemViewController.UploadFile(file, fileType)
    ↓
ItemViewAdminService.ProcessFileUploadAsync(file, fileType)
    ↓
FileImportService determines type
    ↓
CsvService.ImportAsync() or ExcelService.ImportAsync()
    ↓
Items bulk created in database
```

---

## 16. Module: Restock Management

### 16.1 RestockController & RestockService

**Files**: `Areas/Admin/Controllers/RestockController.cs`, `Areas/Admin/Services/RestockService.cs`

Handles restock requests for items that have low quantity.

**Low Stock Alert Flow**:
```
When an admin approves a borrow request (RequestAdminService.ApproveRequestAsync):
    ↓
CheckAndNotifyLowStock(item) is called
    ↓
If item.Quantity <= item.QuantityLow:
    ↓
Notification sent to all admins: "Low stock alert for {item.ItemName}"
```

---

## 17. Module: Utilities & Shared Services

### 17.1 Utility Classes

**Location**: `InventoryManagementSystem.Utilities/`

| Class | Purpose |
|---|---|
| `AllRoles` | Role name constants: `Role_Admin = "Admin"`, `Role_Employee = "Employee"` |
| `EmailSender` | Implements `IEmailSender` using Gmail SMTP. Config from `appsettings.json` → `SenderEmailDetails` |
| `OperationResult` | Simple result wrapper: `{ Success, Message }` |
| `ServiceResult` | Generic result with data: `{ Success, Message, Data }`. Static factory: `Success()`, `Failure()` |
| `TimeToWib` | Converts UTC time to WIB (Western Indonesian Time, UTC+7) |

### 17.2 Shared Services

| Service | File | Purpose |
|---|---|---|
| `CurrentUserService` | `Services/CurrentUserService.cs` | Get logged-in user info (userId, name, role) |
| `PaginationService` | `Services/PaginationService.cs` | Reusable pagination logic. `Paginate<T>(items, pageSize, currentPage, navbarLimit)` |
| `UserProfileService` | `Services/UserProfileService.cs` | Profile management with image upload for both Admin and Employee |

### 17.3 ViewComponents

| Component | File | Purpose |
|---|---|---|
| `HeaderViewComponent` | `ViewComponents/HeaderViewComponent.cs` | Navigation header bar |
| `FooterViewComponent` | `ViewComponents/FooterViewComponent.cs` | Page footer |
| `NavigationViewComponent` | `ViewComponents/NavigationViewComponent.cs` | Sidebar menu (role-based) |
| `NotificationBellViewComponent` | `ViewComponents/NotificationBellViewComponent.cs` | Bell icon with unread count |
| `PaginationViewComponent` | `ViewComponents/PaginationViewComponent.cs` | Reusable pagination UI |

### 17.4 Email Templates

Located in `EmailTemplates/`:
- `EmailConfirmation.html` — Registration email confirmation
- `RequestApproved.html` — Notification when request is approved
- `RequestRejected.html` — Notification when request is rejected

---

## 18. Dependency Injection Registration

All services and repositories are registered in `Program.cs` using **Scoped** lifetime:

### Repositories
```csharp
builder.Services.AddScoped<IItemRepository, ItemRepository>();
builder.Services.AddScoped<IItemViewRepository, ItemViewRepository>();
builder.Services.AddScoped<ICategoryRepository, CategoryRepository>();
builder.Services.AddScoped<ISubCategoryRepository, SubCategoryRepository>();
builder.Services.AddScoped<IRequestRepository, RequestRepository>();
builder.Services.AddScoped<IActiveRequestRepository, ActiveRequestRepository>();
builder.Services.AddScoped<IReservationRepository, ReservationRepository>();
builder.Services.AddScoped<INotificationRepository, NotificationRepository>();
builder.Services.AddScoped<IAuditLogRepository, AuditLogRepository>();
// ... and 13 more repositories
```

### Services
```csharp
// Admin Services
builder.Services.AddScoped<IItemViewAdminService, ItemViewAdminService>();
builder.Services.AddScoped<IRequestAdminService, RequestAdminService>();
builder.Services.AddScoped<IActiveRequestService, ActiveRequestService>();
builder.Services.AddScoped<ICategoryAdminService, CategoryAdminService>();
builder.Services.AddScoped<IUserViewService, UserViewService>();
builder.Services.AddScoped<IReservationAdminService, ReservationAdminService>();
// ... and more

// Employee Services
builder.Services.AddScoped<IItemViewEmployeeService, ItemViewEmployeeService>();
builder.Services.AddScoped<IRequestEmployeeService, RequestEmployeeService>();
builder.Services.AddScoped<IEmployeeActiveRequestService, EmployeeActiveRequestService>();
// ... and more

// Shared Services
builder.Services.AddScoped<IPaginationService, PaginationService>();
builder.Services.AddScoped<INotificationModule, NotificationModule>();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
```

### External Libraries
```csharp
builder.Services.AddAutoMapper(typeof(XxxMappingProfile));  // 16 profiles
builder.Services.AddFluentValidationClientsideAdapters();
builder.Services.AddTransient<IValidator<UserCreateDto>, UserCreateValidator>();
builder.Services.AddSignalR();
```

---

## 19. Routing & Middleware Pipeline

### 19.1 Route Configuration

```csharp
// Area routes (Admin & Employee controllers)
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");

// Default route (Landing page for unauthenticated users)
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Landing}/{action=Index}/{id?}");

// SignalR hub
app.MapHub<NotificationHub>("/notificationHub");
```

### 19.2 Middleware Order

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseSession();            // Session middleware (before auth)
app.UseAuthentication();     // Identity authentication
app.UseAuthorization();      // Role-based authorization
app.MapRazorPages();         // Identity UI pages
```

### 19.3 URL Patterns

| URL Pattern | Handler |
|---|---|
| `/` | `LandingController.Index` (public landing page) |
| `/Admin/Home` | `Admin/HomeController.Index` (admin dashboard) |
| `/Admin/ItemView` | `Admin/ItemViewController.Index` (manage items) |
| `/Admin/Request/WaitingApproval` | `Admin/RequestController.WaitingApproval` |
| `/Employee/Home` | `Employee/HomeController.Index` (employee dashboard) |
| `/Employee/ItemView` | `Employee/ItemViewController.Index` (browse items) |
| `/Employee/Request/WaitingApproval` | `Employee/RequestController.WaitingApproval` |
| `/Identity/Account/Login` | ASP.NET Identity login page |
| `/Identity/Account/Register` | ASP.NET Identity register page |

---

## Quick Reference: Code Location Guide

| Want to change... | Look in... |
|---|---|
| Entity / database models | `InventoryManagementSystem.Models/Models/` |
| DTOs / data transfer objects | `InventoryManagementSystem.Models/ModelsDTO/` |
| Enums / constants | `InventoryManagementSystem.Models/Enum/` |
| Database queries / data access | `InventoryManagementSystem.Persistence/Repository/` |
| DbContext / EF configuration | `InventoryManagementSystem.Persistence/Data/` |
| Database migrations | `InventoryManagementSystem.Persistence/Migrations/` |
| Admin business logic | `InventoryManagementSystem/Areas/Admin/Services/` |
| Employee business logic | `InventoryManagementSystem/Areas/Employee/Services/` |
| Admin controllers (HTTP) | `InventoryManagementSystem/Areas/Admin/Controllers/` |
| Employee controllers (HTTP) | `InventoryManagementSystem/Areas/Employee/Controllers/` |
| Admin views (Razor) | `InventoryManagementSystem/Areas/Admin/Views/` |
| Employee views (Razor) | `InventoryManagementSystem/Areas/Employee/Views/` |
| Login/register pages | `InventoryManagementSystem/Areas/Identity/Pages/` |
| Entity ↔ DTO mapping | `InventoryManagementSystem/Mappers/` |
| Input validation | `InventoryManagementSystem/Validators/` |
| Reusable UI components | `InventoryManagementSystem/ViewComponents/` |
| Email templates | `InventoryManagementSystem/EmailTemplates/` |
| Startup / DI configuration | `InventoryManagementSystem/Program.cs` |
| App settings | `InventoryManagementSystem/appsettings.json` |
| Unit tests | `InventoryManagementSystem.Tests/` |
| Utility helpers | `InventoryManagementSystem.Utilities/` |
| Real-time notifications | `InventoryManagementSystem/Module/NotificationModule.cs` |
| SignalR hub | `InventoryManagementSystem/Hubs/NotificationHub.cs` |
