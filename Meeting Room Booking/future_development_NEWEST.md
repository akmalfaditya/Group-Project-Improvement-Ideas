# Future Development: Sistem Reservasi & Manajemen Ruang Rapat

## 1. Manajemen Reservasi Lanjutan

### [1.1] Sistem Approval Reservasi Bertingkat

**Status saat ini:** Reservasi langsung berstatus `Active` saat dibuat. Enum `ReservationStatus` memiliki `Pending` dan `Rejected`, namun tidak ada workflow approval yang terimplementasi di `ReservationService`. Semua user dapat langsung membooking tanpa persetujuan.

**Fitur baru:**
- Workflow approval multi-level: reservasi otomatis berstatus `Pending` dan memerlukan persetujuan dari Admin atau PIC ruangan sebelum berstatus `Active`
- Konfigurasi aturan approval per ruangan (ruangan tertentu butuh approval, ruangan lain langsung approve)
- Notifikasi otomatis ke approver saat ada reservasi baru yang menunggu persetujuan
- Dashboard khusus approver untuk melihat dan mengelola antrian approval

**Implementasi:**
- Tambah entity `ApprovalRule` (RoomId, RequiresApproval, ApproverId) dan `ApprovalHistory` (ReservationId, ApproverId, Action, Timestamp, Comment)
- Modifikasi `ReservationService.CreateReservationAsync()` untuk mengecek `ApprovalRule` sebelum set status
- Tambah endpoint `PUT api/reservations/{id}/approve` dan `PUT api/reservations/{id}/reject` di `ReservationController`
- Tambah `NotificationType.ReservationPendingApproval` di enum `NotificationType`
- Integrasikan dengan `NotificationService` dan `FirebaseMessagingService` untuk push notification ke approver

---

### [1.2] Deteksi & Resolusi Konflik Jadwal Otomatis

**Status saat ini:** Pencarian ruangan tersedia menggunakan `FindAvailableRoomAsync()` di `RoomService` dengan parameter `startDate`, `endDate`, dan `capacity`. Namun tidak ada mekanisme saran alternatif saat terjadi konflik jadwal.

**Fitur baru:**
- Deteksi konflik jadwal secara real-time saat user membuat atau mengubah reservasi
- Saran slot waktu alternatif terdekat yang tersedia di ruangan yang sama
- Saran ruangan alternatif yang tersedia pada waktu yang diminta (berdasarkan kapasitas dan fasilitas serupa)
- Visualisasi timeline konflik jadwal untuk memudahkan user memilih waktu lain

**Implementasi:**
- Tambah method `FindAlternativeSlots(Guid roomId, DateTime start, DateTime end, int suggestionCount)` di `IReservationService`
- Tambah method `FindAlternativeRooms(DateTime start, DateTime end, int capacity, List<Guid> facilityIds)` di `IRoomService`
- Tambah endpoint `GET api/reservations/conflicts?roomId=&start=&end=` di `ReservationController`
- Tambah response DTO `ConflictResolutionResponse` (ConflictingReservations, AlternativeSlots, AlternativeRooms)
- Modifikasi `AvailableRoomRequest` untuk menyertakan `List<Guid>? FacilityIds` sebagai filter tambahan

---

### [1.3] Check-in & Check-out Reservasi dengan QR Code

**Status saat ini:** Tidak ada mekanisme konfirmasi kehadiran untuk reservasi. Ruangan bisa di-booking tapi tidak ada cara mengetahui apakah pengguna benar-benar menggunakan ruangan tersebut.

**Fitur baru:**
- Generate QR Code unik untuk setiap reservasi yang telah `Active`
- Sistem check-in via scan QR Code saat peserta tiba di ruangan
- Sistem check-out otomatis atau manual saat rapat selesai
- Auto-release ruangan jika tidak ada check-in dalam batas waktu tertentu (misal 15 menit setelah waktu mulai)
- Dashboard statistik tingkat kehadiran per ruangan dan per user

**Implementasi:**
- Tambah entity `CheckIn` (CheckInId, ReservationId, UserId, CheckInTime, CheckOutTime, Method: QRCode/Manual)
- Tambah field `QRCodeToken` (string, unique) pada entity `Reservation`
- Tambah `CheckInRepository` dan `ICheckInRepository` di layer Database
- Tambah `CheckInService` dengan method `GenerateQRToken()`, `ProcessCheckIn()`, `ProcessCheckOut()`, `AutoReleaseExpiredReservations()`
- Tambah `CheckInController` dengan endpoint `POST api/checkin/scan`, `POST api/checkout/{reservationId}`, `GET api/checkin/status/{reservationId}`
- Library tambahan: `QRCoder` untuk generate QR Code image
- Background service (`IHostedService`) untuk auto-release reservasi tanpa check-in

---

### [1.4] Reservasi Berulang (Recurring) yang Lebih Canggih

**Status saat ini:** Entity `Reservation` memiliki field `Recurrence` (string, nullable) dan `ReservationRequest` memiliki `IEnumerable<string> Recurrence`. Integrasi dengan Google Calendar sudah mendukung recurrence rule (RRULE). Namun tidak ada pengelolaan individual occurrence dari sisi internal.

**Fitur baru:**
- Kemampuan mengedit atau membatalkan single occurrence dari reservasi berulang tanpa mempengaruhi occurrence lainnya
- Preview daftar semua occurrence yang akan dibuat sebelum konfirmasi
- Penanganan konflik khusus untuk reservasi berulang (jika satu occurrence konflik, tawarkan skip atau pindah ruangan hanya untuk occurrence tersebut)
- Batasan maksimal recurring period (misal: maksimal 3 bulan ke depan)

**Implementasi:**
- Tambah entity `RecurringReservationGroup` (GroupId, MasterReservationId, RecurrenceRule, StartBound, EndBound)
- Tambah field `RecurringGroupId` pada entity `Reservation` untuk mengelompokkan occurrence
- Tambah method `ExpandRecurrence()` dan `ModifySingleOccurrence()` di `ReservationService`
- Tambah endpoint `GET api/reservations/recurring/{groupId}/occurrences` dan `PUT api/reservations/recurring/{groupId}/occurrence/{date}`
- Modifikasi `GoogleCalendarService` untuk sinkronisasi per-instance exception

---

### [1.5] Waitlist & Notifikasi Ketersediaan

**Status saat ini:** Jika ruangan sudah dibooking pada waktu tertentu, user hanya bisa mencari ruangan lain via `FindAvailableRoom`. Tidak ada mekanisme antrian tunggu.

**Fitur baru:**
- User dapat mendaftar ke waitlist saat ruangan yang diinginkan tidak tersedia
- Notifikasi otomatis via push notification dan email saat ruangan menjadi tersedia (reservasi dibatalkan/diubah)
- Prioritas waitlist berdasarkan waktu pendaftaran
- Auto-booking opsional: jika user mengaktifkan opsi ini, reservasi otomatis dibuat saat ruangan tersedia

**Implementasi:**
- Tambah entity `Waitlist` (WaitlistId, UserId, RoomId, DesiredStart, DesiredEnd, Priority, AutoBook, DateCreated, Status: Waiting/Notified/Fulfilled/Expired)
- Tambah `WaitlistRepository` dan `WaitlistService` dengan method `AddToWaitlist()`, `NotifyWaitlistUsers()`, `AutoBookForWaitlist()`
- Tambah `WaitlistController` dengan endpoint `POST api/waitlist`, `GET api/waitlist/user/{userId}`, `DELETE api/waitlist/{id}`
- Hook ke `ReservationService.DeleteReservationAsync()` dan `UpdateReservationAsync()` untuk trigger pengecekan waitlist
- Tambah `NotificationType.WaitlistAvailable` di enum `NotificationType`

---

## 2. Manajemen Ruangan & Fasilitas Lanjutan

### [2.1] Sistem Lokasi & Peta Interaktif Ruangan

**Status saat ini:** Entity `Room` menyimpan `RoomName`, `Capacity`, `Description`, `RoomImageUrl`, dan `AvailabilityStatus`. Tidak ada informasi lokasi fisik (lantai, gedung, area) yang tersimpan.

**Fitur baru:**
- Informasi lokasi ruangan meliputi gedung, lantai, dan zona/area
- Peta interaktif (floor plan) yang menunjukkan posisi ruangan di setiap lantai
- Indikator status real-time pada peta (Available/hijau, In-Use/merah, Issues/kuning)
- Navigasi dari peta langsung ke form reservasi ruangan yang dipilih

**Implementasi:**
- Tambah entity `Building` (BuildingId, Name, Address) dan `Floor` (FloorId, BuildingId, FloorNumber, FloorPlanImageUrl)
- Tambah field `FloorId`, `PositionX`, `PositionY` pada entity `Room` untuk penempatan pada peta
- Tambah `BuildingController` dan `FloorController` dengan CRUD endpoint
- Modifikasi `RoomResponse` untuk menyertakan informasi lokasi (BuildingName, FloorNumber, FloorPlanUrl)
- Modifikasi `GetRoomsRequest` untuk menambah filter `buildingId` dan `floorId`

---

### [2.2] Manajemen Inventaris Fasilitas dengan Riwayat Kondisi

**Status saat ini:** Entity `Facility` hanya menyimpan `Name` dan `IsActive`. Relasi `RoomFacility` menyimpan `Condition` (Good/MinorIssue/Broken/Maintenance). Tidak ada tracking riwayat perubahan kondisi atau jumlah unit fasilitas per ruangan.

**Fitur baru:**
- Tracking riwayat perubahan kondisi fasilitas per ruangan (siapa yang mengubah, kapan, dari kondisi apa ke kondisi apa)
- Jumlah unit fasilitas per ruangan (misal: 5 kursi, 2 whiteboard)
- Jadwal maintenance preventif otomatis berdasarkan umur pakai atau interval tertentu
- Peringatan otomatis ke Admin saat kondisi fasilitas berubah menjadi `Broken` atau `Maintenance`

**Implementasi:**
- Tambah entity `FacilityConditionHistory` (HistoryId, RoomId, FacilityId, OldCondition, NewCondition, ChangedByUserId, DateChanged, Notes)
- Tambah field `Quantity` (int) dan `LastMaintenanceDate` (DateTime?) pada entity `RoomFacility`
- Tambah entity `MaintenanceSchedule` (ScheduleId, RoomFacilityId, IntervalDays, NextMaintenanceDate, IsActive)
- Modifikasi `FacilityService` untuk mencatat riwayat setiap perubahan kondisi
- Tambah endpoint `GET api/facilities/{facilityId}/history` di `FacilityController`
- Tambah `NotificationType.FacilityBroken` pada enum `NotificationType`
- Background service untuk pengecekan jadwal maintenance dan kirim reminder

---

### [2.3] Sistem Kapasitas Dinamis & Konfigurasi Layout Ruangan

**Status saat ini:** Entity `Room` memiliki field `Capacity` (int) yang bersifat statis. Tidak ada kemampuan untuk mendefinisikan layout atau konfigurasi ruangan yang berbeda.

**Fitur baru:**
- Multiple layout per ruangan (Theater, U-Shape, Boardroom, Classroom) dengan kapasitas berbeda per layout
- User dapat memilih layout saat membuat reservasi
- Informasi visual layout tersedia untuk user sebagai referensi
- Rekomendasi layout berdasarkan jumlah peserta yang terdaftar

**Implementasi:**
- Tambah entity `RoomLayout` (LayoutId, RoomId, LayoutType: enum, Capacity, LayoutImageUrl, IsDefault)
- Tambah enum `LayoutType` (Theater, UShape, Boardroom, Classroom, RoundTable, Standing)
- Tambah field `LayoutId` (Guid?, optional) pada entity `Reservation`
- Modifikasi `ReservationRequest` untuk menambah field `LayoutId`
- Modifikasi `RoomResponse` untuk menyertakan `List<RoomLayoutResponse>`
- Tambah endpoint `GET api/rooms/{roomId}/layouts` dan `POST api/rooms/{roomId}/layouts` di `RoomController`

---

### [2.4] Sistem Rating & Review Ruangan

**Status saat ini:** Tidak ada mekanisme feedback dari pengguna setelah menggunakan ruangan. Data kualitas pengalaman pengguna tidak tercatat.

**Fitur baru:**
- User dapat memberikan rating (1-5 bintang) dan review setelah reservasi berstatus `Completed`
- Rating kategorikal: kebersihan, kenyamanan, kelengkapan fasilitas, kualitas AV equipment
- Skor rata-rata rating ditampilkan pada detail ruangan
- Trigger review otomatis setelah waktu reservasi berakhir via notifikasi

**Implementasi:**
- Tambah entity `RoomReview` (ReviewId, ReservationId, UserId, RoomId, OverallRating, CleanlinessRating, ComfortRating, FacilityRating, Comment, DateCreated)
- Tambah `RoomReviewRepository` dan `RoomReviewService`
- Tambah `RoomReviewController` dengan endpoint `POST api/rooms/{roomId}/reviews`, `GET api/rooms/{roomId}/reviews`
- Modifikasi `RoomResponse` untuk menyertakan `AverageRating` dan `TotalReviews`
- Tambah `NotificationType.ReviewReminder` di enum `NotificationType`
- Hook ke `ReservationService` untuk trigger notifikasi review saat reservasi selesai

---

## 3. Dashboard & Analitik

### [3.1] Dashboard Analitik Penggunaan Ruangan

**Status saat ini:** Terdapat endpoint `GET api/reservations/room-usage` yang mengembalikan `RoomUsagePerDay` dengan response `RoomUsageDailyResponse` dan `RoomUsageReservation`. Namun analitik masih sangat terbatas, hanya berupa raw data usage per hari.

**Fitur baru:**
- Dashboard visual dengan grafik tren penggunaan ruangan (harian, mingguan, bulanan)
- Metrik kunci: occupancy rate per ruangan, peak hours, rata-rata durasi rapat, ruangan paling/paling jarang digunakan
- Heatmap penggunaan ruangan per jam dalam seminggu
- Perbandingan utilisasi antar ruangan dalam periode tertentu
- Export laporan analitik ke PDF dan Excel

**Implementasi:**
- Tambah `AnalyticsService` dengan method `GetOccupancyRate()`, `GetPeakHours()`, `GetUsageTrend()`, `GetRoomComparison()`
- Tambah response DTO: `OccupancyRateResponse`, `PeakHoursResponse`, `UsageTrendResponse`, `RoomComparisonResponse`
- Tambah `AnalyticsController` dengan endpoint `GET api/analytics/occupancy`, `GET api/analytics/peak-hours`, `GET api/analytics/trends`, `GET api/analytics/comparison`
- Modifikasi `ExportService` untuk mendukung export analitik ke PDF (library: `QuestPDF` atau `iTextSharp`) dan Excel (`EPPlus`)
- Manfaatkan data dari `ReservationRepository` dan `CheckIn` (jika fitur 1.3 terimplementasi) untuk kalkulasi

---

### [3.2] Laporan Biaya & Utilisasi Sumber Daya

**Status saat ini:** Tidak ada tracking biaya operasional ruangan atau alokasi biaya penggunaan ke departemen/tim. Entity `Report` digunakan khusus untuk laporan masalah/kerusakan, bukan laporan keuangan/utilisasi.

**Fitur baru:**
- Konfigurasi tarif per jam per ruangan (opsional, untuk organisasi yang menerapkan charge-back antar departemen)
- Kalkulasi estimasi biaya penggunaan ruangan per departemen/tim per periode
- Tracking biaya maintenance dan perbaikan fasilitas
- Laporan perbandingan biaya vs utilisasi sebagai dasar keputusan investasi ruangan baru

**Implementasi:**
- Tambah entity `RoomPricing` (PricingId, RoomId, HourlyRate, EffectiveFrom, EffectiveTo)
- Tambah entity `MaintenanceCost` (CostId, RoomId, FacilityId?, Description, Amount, DateIncurred)
- Tambah field `Department` (string?) pada entity `User` untuk pengelompokan biaya
- Tambah `CostReportService` dengan method `GetDepartmentUsageCost()`, `GetMaintenanceCostSummary()`, `GetROIReport()`
- Tambah `CostReportController` dengan endpoint `GET api/reports/cost/department`, `GET api/reports/cost/maintenance`, `GET api/reports/cost/roi`

---

## 4. Integrasi Kalender & Komunikasi

### [4.1] Sinkronisasi Kalender Dua Arah yang Lebih Robust

**Status saat ini:** `GoogleCalendarService` sudah mengimplementasi full CRUD untuk Calendar dan Event via Google Calendar API. `GoogleCalendarController` memiliki 10 endpoint. Sinkronisasi menggunakan `syncToken` pada `GetEventList`. Namun tidak ada mekanisme webhook untuk real-time sync dari sisi Google ke aplikasi.

**Fitur baru:**
- Webhook/Push Notification dari Google Calendar untuk mendeteksi perubahan event secara real-time (bukan polling)
- Auto-sync: perubahan di Google Calendar otomatis ter-refleksi di sistem internal
- Conflict resolution saat perubahan bersamaan dari kedua sisi
- Support kalender tambahan: Microsoft Outlook Calendar (via Microsoft Graph API)

**Implementasi:**
- Tambah endpoint `POST api/google/calendar/webhook` di `GoogleCalendarController` sebagai receiver Google Calendar push notification
- Tambah entity `CalendarSyncLog` (LogId, CalendarId, EventId, SyncDirection, SyncStatus, Timestamp, ErrorMessage?)
- Tambah `CalendarWebhookService` untuk memproses incoming push notification dan melakukan reconciliation
- Modifikasi `GoogleCalendarService` untuk meregistrasikan watch channel via `Events.watch()` API
- Tambah `IOutlookCalendarService` dan `OutlookCalendarService` untuk integrasi Microsoft Outlook
- Library tambahan: `Microsoft.Graph` untuk Microsoft Outlook Calendar API

---

### [4.2] Integrasi Meeting Platform (Video Conference)

**Status saat ini:** Tidak ada integrasi dengan platform video conference. Reservasi hanya untuk ruangan fisik tanpa kemampuan membuat meeting link otomatis.

**Fitur baru:**
- Auto-generate link meeting (Google Meet, Zoom, Microsoft Teams) saat membuat reservasi
- Opsi hybrid meeting: ruangan fisik + link virtual untuk peserta remote
- Link meeting otomatis disertakan di undangan Google Calendar event
- Informasi meeting link tersedia di detail reservasi

**Implementasi:**
- Tambah enum `MeetingPlatform` (None, GoogleMeet, Zoom, MicrosoftTeams)
- Tambah field `MeetingPlatform` dan `MeetingLink` pada entity `Reservation`
- Modifikasi `ReservationRequest` untuk menambah field `MeetingPlatform`
- Modifikasi `ReservationResponse` untuk menyertakan `MeetingLink`
- Tambah `IMeetingLinkService` dengan implementasi `GoogleMeetService`, `ZoomService`, `TeamsService`
- Modifikasi `ReservationService.CreateReservationAsync()` untuk auto-generate link berdasarkan platform yang dipilih
- Modifikasi `GoogleCalendarService.InsertEventAsync()` untuk menyertakan `ConferenceData` pada Google Calendar event
- Library tambahan: `Zoom API SDK`, `Microsoft.Graph`

---

### [4.3] Notifikasi Multi-Channel yang Lebih Kaya

**Status saat ini:** Sistem notifikasi memiliki 7 `NotificationType` (ReservationApproved, ReservationRejected, UpcomingMeeting, ReservationCancelled, MaintenanceAlert, NewReport, GeneralAnnouncement). Implementasi menggunakan `NotificationService`, `NotificationPushService` via Firebase, dan `EmailService` (43KB, sangat komprehensif). `BrowserNotificationTrigger` entity tersedia untuk trigger notifikasi browser.

**Fitur baru:**
- Notifikasi reminder bertingkat sebelum meeting dimulai (misal: 24 jam, 1 jam, 15 menit sebelum)
- Konfigurasi preferensi notifikasi per user (channel mana yang aktif: email, push, in-app)
- Notifikasi digest harian: ringkasan jadwal rapat hari ini dikirim setiap pagi
- Notifikasi saat ada perubahan attendee di reservasi yang diikuti

**Implementasi:**
- Tambah entity `NotificationPreference` (UserId, Channel: enum Email/Push/InApp, IsEnabled, ReminderMinutesBefore: List)
- Tambah enum `NotificationChannel` (Email, Push, InApp)
- Modifikasi `NotificationTriggerService` untuk mengecek preferensi user sebelum mengirim notifikasi
- Tambah method `SendDailyDigest()` di `EmailService` dan `NotificationService`
- Background service (`IHostedService`) untuk trigger reminder bertingkat dan daily digest
- Tambah endpoint `GET/PUT api/notifications/preferences` di `NotificationController`

---

## 5. AI Chatbot Lanjutan

### [5.1] AI Chatbot dengan Kemampuan Booking Langsung

**Status saat ini:** `AiChatService` memiliki `ProcessMessageAsync()` dan `AiActionType` enum hanya mendukung `None`, `AvailableRooms`, dan `Faq`. AI hanya bisa menjawab pertanyaan FAQ dan mencari ruangan tersedia, belum bisa melakukan aksi booking.

**Fitur baru:**
- Chatbot dapat membuat reservasi langsung melalui percakapan natural language (misal: "Booking ruangan Orchid besok jam 10 pagi untuk 5 orang")
- Chatbot dapat membatalkan atau mengubah reservasi existing
- Chatbot memberikan rekomendasi ruangan berdasarkan kebutuhan yang di-describe user
- Chatbot memahami konteks percakapan multi-turn (conversation history)

**Implementasi:**
- Tambah `AiActionType`: `CreateReservation = 3`, `CancelReservation = 4`, `UpdateReservation = 5`, `RecommendRoom = 6`
- Modifikasi `AiChatService.ProcessMessageAsync()` untuk menangani action baru
- Tambah method `ParseBookingIntent()`, `ParseCancellationIntent()`, `ParseUpdateIntent()` di service AI
- Tambah `AiChatRequest` field `ConversationHistory` (List<ChatMessage>) untuk konteks multi-turn
- Modifikasi folder `Services/AI/` untuk menambahkan plugin baru: `BookingPlugin`, `CancellationPlugin`, `RecommendationPlugin`
- Integrasi dengan `ReservationService` dan `RoomService` untuk eksekusi aksi

---

### [5.2] AI-Powered Smart Scheduling Suggestion

**Status saat ini:** Pencarian ruangan tersedia (`FindAvailableRoomAsync`) hanya berdasarkan waktu dan kapasitas. Tidak ada rekomendasi cerdas berdasarkan pola penggunaan.

**Fitur baru:**
- Rekomendasi waktu meeting optimal berdasarkan ketersediaan semua attendee (calendar cross-check)
- Rekomendasi ruangan berdasarkan pola penggunaan historis user dan tim
- Prediksi kemungkinan no-show berdasarkan histori kehadiran, sehingga double-booking kontekstual bisa ditawarkan
- AI menyarankan durasi meeting berdasarkan jenis rapat (standup: 15 min, weekly: 1 jam, workshop: 2 jam)

**Implementasi:**
- Tambah `SmartSchedulingService` dengan method `SuggestOptimalTime()`, `SuggestRoom()`, `PredictNoShow()`, `SuggestDuration()`
- Tambah response DTO `SmartSuggestionResponse` (SuggestedTimes, SuggestedRooms, NoShowProbability, SuggestedDuration, Reasoning)
- Tambah endpoint `POST api/chat/smart-schedule` di `AiChatController`
- Manfaatkan data dari `ReservationRepository`, `CheckIn` (jika ada), dan `GoogleCalendarService` untuk analisis
- Integrasi dengan AI model untuk natural language understanding dan pattern recognition

---

## 6. Laporan Masalah & Pemeliharaan

### [6.1] Workflow Laporan Masalah yang Lebih Lengkap

**Status saat ini:** Entity `Report` memiliki `Status` (Pending/InProgress/Resolved/Closed) dan `Priority` (Low/Medium/High/Critical). `ReportController` menyediakan CRUD dan endpoint khusus `UpdateReportStatus` (Admin only). Namun tidak ada tracking riwayat status, lampiran foto, atau assignment ke teknisi.

**Fitur baru:**
- Lampiran foto/video kerusakan pada laporan (evidence upload)
- Assignment laporan ke teknisi/PIC tertentu oleh Admin
- Riwayat perubahan status laporan (audit trail)
- Estimasi waktu penyelesaian dan tracking SLA (Service Level Agreement)
- Komentar/diskusi pada laporan antara pelapor, admin, dan teknisi

**Implementasi:**
- Tambah entity `ReportAttachment` (AttachmentId, ReportId, FileUrl, FileType, UploadedByUserId, DateUploaded)
- Tambah entity `ReportStatusHistory` (HistoryId, ReportId, OldStatus, NewStatus, ChangedByUserId, DateChanged, Comment)
- Tambah entity `ReportComment` (CommentId, ReportId, UserId, Content, DateCreated)
- Tambah field `AssignedToUserId` (string?) dan `EstimatedCompletionDate` (DateTime?) pada entity `Report`
- Modifikasi `ReportService` untuk mencatat riwayat setiap perubahan status
- Tambah endpoint `POST api/reports/{id}/attachments`, `GET api/reports/{id}/history`, `POST api/reports/{id}/comments` di `ReportController`
- Tambah `NotificationType.ReportAssigned` dan `ReportStatusChanged` di enum

---

### [6.2] Auto-Block Ruangan Berdasarkan Laporan Kritis

**Status saat ini:** Status ruangan (`RoomAvailabilityStatus`) dapat diubah manual oleh admin menjadi `Issues` atau `Unavailable`. Laporan masalah dengan priority `Critical` tidak otomatis mempengaruhi ketersediaan ruangan.

**Fitur baru:**
- Ruangan otomatis berubah status menjadi `Issues` atau `Unavailable` saat ada laporan berstatus `Critical` yang belum resolved
- Notifikasi otomatis ke semua user yang memiliki reservasi aktif di ruangan tersebut
- Opsi admin untuk override dan tetap mengizinkan booking meskipun ada laporan kritis
- Ruangan otomatis kembali `Available` saat semua laporan kritis telah `Resolved` atau `Closed`

**Implementasi:**
- Modifikasi `ReportService.CreateReportAsync()` untuk mengecek priority dan otomatis update `Room.AvailabilityStatus`
- Modifikasi `ReportService.UpdateReportStatusAsync()` untuk re-evaluate status ruangan saat laporan di-resolve
- Tambah method `NotifyAffectedReservations()` di `NotificationService` untuk mengirim notifikasi ke user terdampak
- Tambah field `AdminOverride` (bool) pada entity `Room` untuk override admin
- Modifikasi `RoomService` untuk mengecek laporan aktif saat menentukan ketersediaan

---

## 7. Keamanan & Manajemen Pengguna

### [7.1] Sistem Delegasi & Booking Atas Nama Orang Lain

**Status saat ini:** `ReservationRequest` memiliki field `UserId` yang wajib diisi. User bisa membuat reservasi untuk dirinya sendiri. Tidak ada mekanisme formal untuk sekretaris atau asisten yang mem-booking atas nama atasan.

**Fitur baru:**
- Role "Delegate" yang memungkinkan user tertentu membuat reservasi atas nama user lain
- Konfigurasi delegasi: siapa yang boleh mem-booking atas nama siapa
- Audit trail yang jelas mencatat siapa yang membuat booking dan untuk siapa
- Notifikasi ke pemilik reservasi saat delegate membuat/mengubah booking atas nama mereka

**Implementasi:**
- Tambah entity `BookingDelegation` (DelegationId, DelegateUserId, OnBehalfOfUserId, IsActive, DateCreated, DateExpired?)
- Tambah field `CreatedByUserId` (string?) pada entity `Reservation` untuk membedakan pembuat dan pemilik reservasi
- Tambah `DelegationService` dengan method `GrantDelegation()`, `RevokeDelegation()`, `GetDelegationsForUser()`
- Tambah `DelegationController` dengan endpoint `POST api/delegations`, `GET api/delegations/user/{userId}`, `DELETE api/delegations/{id}`
- Modifikasi `ReservationService.CreateReservationAsync()` untuk validasi hak delegasi

---

### [7.2] Audit Log yang Lebih Komprehensif

**Status saat ini:** Entity `Activity` tracking aksi `Add/Update/Delete` pada target `User/Room/Reservation` dengan informasi `AuthorName`, `TargetName`, dan navigasi ke entity terkait. Namun tidak ada detail perubahan spesifik (field apa yang berubah, dari nilai apa ke nilai apa).

**Fitur baru:**
- Detail perubahan per field (old value vs new value) pada setiap activity log
- Tambahan target tracking: Facility, Report, Notification, Role, CalendarSync
- Filtering dan pencarian advanced pada activity log (berdasarkan tanggal, user, tipe aksi, target)
- Export activity log untuk keperluan compliance/audit

**Implementasi:**
- Tambah entity `ActivityDetail` (DetailId, ActivityId, FieldName, OldValue, NewValue)
- Tambah value pada enum `ActivityTarget`: `Facility`, `Report`, `Notification`, `Role`, `Settings`
- Modifikasi `ActivityService` untuk menerima dictionary perubahan field dan menyimpannya ke `ActivityDetail`
- Modifikasi semua service (RoomService, FacilityService, ReportService, UserService) untuk mengirim detail perubahan saat mencatat activity
- Tambah endpoint `GET api/activities/export` di `ActivityController` dengan dukungan format CSV/PDF
- Modifikasi `GetRoomsRequest`-style filtering pada `ActivityController.GetActivities()` untuk advanced search

---

### [7.3] Rate Limiting & Kebijakan Reservasi Per User

**Status saat ini:** Tidak ada batasan jumlah reservasi yang bisa dibuat oleh seorang user dalam periode tertentu. User berpotensi mendominasi ketersediaan ruangan.

**Fitur baru:**
- Konfigurasi batas maksimal reservasi aktif per user (misal: maks 5 reservasi aktif)
- Konfigurasi batas maksimal jam booking per user per minggu (misal: maks 10 jam/minggu)
- Kebijakan berbeda per role (Admin tidak dibatasi, Regular User dibatasi)
- Dashboard admin untuk melihat penggunaan kuota per user

**Implementasi:**
- Tambah entity `BookingPolicy` (PolicyId, RoleName, MaxActiveReservations, MaxWeeklyHours, MaxAdvanceBookingDays, MinBookingDurationMinutes, MaxBookingDurationMinutes, IsActive)
- Tambah `BookingPolicyService` dengan method `ValidateUserQuota()`, `GetUserUsageSummary()`
- Modifikasi `ReservationService.CreateReservationAsync()` untuk memanggil `ValidateUserQuota()` sebelum membuat reservasi
- Tambah `BookingPolicyController` dengan endpoint `GET/PUT api/policies` (Admin only)
- Tambah endpoint `GET api/users/{userId}/booking-quota` di `UserController`

---

## 8. Pengalaman Pengguna & Aksesibilitas

### [8.1] Tampilan Kalender Multi-View untuk Frontend

**Status saat ini:** Data reservasi di-return sebagai list flat dari API (`GET api/reservations`). Frontend harus mengolah sendiri data untuk tampilan kalender. Tidak ada endpoint yang dioptimalkan untuk rendering kalender.

**Fitur baru:**
- Endpoint khusus yang mengembalikan data terstruktur untuk tampilan kalender harian, mingguan, dan bulanan
- Grouping reservasi per ruangan untuk tampilan multi-room timeline
- Color-coding otomatis berdasarkan status reservasi dan kategori ruangan
- Data ringkas (summary) untuk performa rendering kalender yang optimal

**Implementasi:**
- Tambah response DTO `CalendarViewResponse` (Date, TimeSlots: List<CalendarSlot>) dan `CalendarSlot` (Start, End, ReservationSummary, RoomName, Color, Status)
- Tambah method `GetCalendarView()` di `ReservationService` dengan parameter viewType (Day/Week/Month), startDate, roomIds
- Tambah endpoint `GET api/reservations/calendar-view?type=week&start=2026-03-01&roomIds=...` di `ReservationController`
- Optimisasi query di `ReservationRepository` untuk mengurangi data transfer (minimal projection)

---

### [8.2] Sistem Template Reservasi (Quick Booking)

**Status saat ini:** User harus mengisi semua parameter (waktu, ruangan, deskripsi, attendee) setiap kali membuat reservasi. Tidak ada kemampuan menyimpan template untuk rapat rutin.

**Fitur baru:**
- User dapat menyimpan template reservasi (misal: "Weekly Standup" dengan preset ruangan, durasi, attendee)
- Quick booking: satu klik untuk membuat reservasi dari template, hanya perlu pilih tanggal/waktu
- Template bisa di-share antar anggota tim
- Admin dapat membuat template organisasi yang tersedia untuk semua user

**Implementasi:**
- Tambah entity `ReservationTemplate` (TemplateId, UserId, Name, RoomId?, Duration, Description, AttendeesEmail: JSON, Visibility: Private/Team/Organization, IsActive)
- Tambah `ReservationTemplateRepository` dan `ReservationTemplateService`
- Tambah `ReservationTemplateController` dengan endpoint `POST api/templates`, `GET api/templates`, `POST api/templates/{id}/book`
- Modifikasi `ReservationService` untuk mendukung pembuatan reservasi dari template

---

### [8.3] Tampilan Display Ruangan (Room Signage Digital)

**Status saat ini:** Tidak ada fitur tampilan digital di depan ruangan yang menunjukkan status booking secara real-time. Informasi ketersediaan hanya bisa diakses melalui web application.

**Fitur baru:**
- Halaman khusus (endpoint terpisah tanpa login) untuk ditampilkan pada tablet/layar di depan pintu ruangan
- Menampilkan status ruangan saat ini (Available/In Use/Upcoming), nama meeting yang sedang berlangsung, dan jadwal selanjutnya
- Tombol quick-booking langsung dari display untuk slot yang kosong (walk-in booking)
- Warna background otomatis berubah berdasarkan status (hijau: Available, merah: In Use, kuning: Starting Soon)

**Implementasi:**
- Tambah `RoomDisplayController` dengan endpoint `GET api/display/{roomId}` (tanpa `[Authorize]`)
- Tambah response DTO `RoomDisplayResponse` (RoomName, CurrentStatus, CurrentMeeting?, NextMeetings: List, WalkInAvailableSlots)
- Tambah authentication khusus untuk display menggunakan API Key per ruangan (`RoomApiKey` field pada entity `Room`)
- Modifikasi `RoomService` untuk menyediakan method `GetRoomDisplayInfo(Guid roomId)` yang mengambil data real-time
- Tambah endpoint `POST api/display/{roomId}/walk-in` untuk quick booking dari display

---

### [8.4] Aksesibilitas Multi-Bahasa (i18n)

**Status saat ini:** Semua response dan pesan error dari API menggunakan bahasa Inggris yang di-hardcode di dalam service dan controller. Tidak ada mekanisme internasionalisasi.

**Fitur baru:**
- Dukungan multi-bahasa untuk semua response message API (Bahasa Indonesia dan English sebagai default)
- Template email dan notifikasi dalam bahasa yang sesuai preferensi user
- Pilihan bahasa tersimpan di profil user
- Resource file terpisah untuk memudahkan penambahan bahasa baru

**Implementasi:**
- Tambah field `PreferredLanguage` (string, default: "id") pada entity `User`
- Implementasi `IStringLocalizer` dan resource files (`.resx`) untuk setiap bahasa: `Resources/Messages.id.resx`, `Resources/Messages.en.resx`
- Modifikasi semua service untuk menggunakan `IStringLocalizer` alih-alih string hardcoded
- Modifikasi `EmailService` untuk memilih template email berdasarkan bahasa user
- Modifikasi `NotificationService` untuk menghasilkan pesan notifikasi sesuai bahasa user
- Konfigurasi `RequestLocalizationMiddleware` di `Program.cs`

---

## 9. Real-time & Komunikasi Live

### [9.1] Real-time Status Update via SignalR

**Status saat ini:** Tidak ada implementasi WebSocket atau SignalR di API. Semua data diambil melalui HTTP request/response tradisional (REST). Frontend harus melakukan polling untuk mendapatkan data terbaru.

**Fitur baru:**
- Update status ruangan secara real-time di frontend tanpa perlu refresh halaman
- Live notification saat ada reservasi baru, perubahan, atau pembatalan yang mempengaruhi kalender user
- Real-time update pada peta interaktif (fitur 2.1) saat status ruangan berubah
- Countdown timer live untuk reservasi yang sedang berlangsung

**Implementasi:**
- Tambah package `Microsoft.AspNetCore.SignalR`
- Tambah `RoomStatusHub` di folder baru `Hubs/` dengan method `SendRoomStatusUpdate()`, `SendReservationUpdate()`, `SendNotification()`
- Konfigurasi SignalR di `Program.cs` dengan endpoint `app.MapHub<RoomStatusHub>("/hubs/room-status")`
- Modifikasi `ReservationService` untuk memanggil `RoomStatusHub` setiap kali ada create/update/delete reservasi
- Modifikasi `RoomService` untuk broadcast perubahan status ruangan via hub
- Modifikasi `NotificationService` untuk push real-time notification via SignalR selain Firebase

---

### [9.2] Live Occupancy Tracking (Sensor IoT Integration)

**Status saat ini:** Tidak ada data real-time tentang apakah ruangan benar-benar sedang digunakan secara fisik. Status ruangan hanya berdasarkan data reservasi di database.

**Fitur baru:**
- Integrasi dengan sensor kehadiran (occupancy sensor) untuk mendeteksi apakah ada orang di ruangan
- Auto-release ruangan jika sensor mendeteksi ruangan kosong selama durasi tertentu meskipun ada reservasi aktif
- Dashboard real-time menunjukkan actual occupancy vs booked occupancy
- Notifikasi ke user jika ruangan yang di-booking terdeteksi kosong (memungkinkan orang lain untuk menggunakannya)

**Implementasi:**
- Tambah entity `OccupancySensor` (SensorId, RoomId, SensorType, ApiEndpoint, ApiKey, IsActive, LastHeartbeat)
- Tambah entity `OccupancyLog` (LogId, RoomId, SensorId, IsOccupied, PeopleCount, Timestamp)
- Tambah `OccupancySensorService` dengan method `ProcessSensorData()`, `GetCurrentOccupancy()`, `CheckGhostBookings()`
- Tambah `OccupancyController` dengan endpoint `POST api/occupancy/data` (untuk sensor push), `GET api/occupancy/{roomId}/current`
- Background service untuk mendeteksi "ghost bookings" (ruangan di-booking tapi kosong) dan kirim notifikasi/auto-release
- Integrasi dengan `RoomStatusHub` (fitur 9.1) untuk broadcast data occupancy real-time

---

### [9.3] Sistem Chat Internal Antar Peserta Meeting

**Status saat ini:** `AiChatController` dan `AiChatService` hanya menangani percakapan antara user dan AI chatbot. Tidak ada fitur messaging antar user terkait konteks reservasi.

**Fitur baru:**
- Chat internal antara peserta meeting dalam konteks reservasi tertentu
- Fitur share file/dokumen agenda meeting sebelum rapat dimulai
- Notifikasi saat ada pesan baru di chat reservasi yang diikuti
- Riwayat chat tersimpan dan bisa diakses dari detail reservasi setelah rapat selesai

**Implementasi:**
- Tambah entity `ReservationChat` (ChatId, ReservationId, UserId, Message, FileUrl?, DateCreated)
- Tambah `ReservationChatHub` (SignalR Hub) untuk real-time messaging dalam konteks reservasi
- Tambah `ReservationChatService` dengan method `SendMessage()`, `GetChatHistory()`, `UploadFile()`
- Tambah endpoint `GET api/reservations/{id}/chat`, `POST api/reservations/{id}/chat` di `ReservationController`
- Konfigurasi SignalR group per reservationId untuk isolasi pesan
- Tambah `NotificationType.NewChatMessage` pada enum `NotificationType`

---

## 10. Integrasi Layanan Pendukung Rapat

### [10.1] Sistem Pemesanan Katering & Konsumsi

**Status saat ini:** Tidak ada fitur pemesanan konsumsi yang terintegrasi dengan reservasi ruangan. Pengguna harus mengurus kebutuhan konsumsi secara terpisah di luar sistem.

**Fitur baru:**
- Katalog menu katering yang tersedia (snack box, makan siang, coffee break, dll.)
- Pemesanan konsumsi langsung saat membuat reservasi atau sebagai add-on setelahnya
- Estimasi biaya konsumsi otomatis berdasarkan jumlah peserta dan menu yang dipilih
- Notifikasi ke vendor katering saat ada pesanan baru
- Status tracking pesanan konsumsi (Ordered, Preparing, Delivered)

**Implementasi:**
- Tambah entity `CateringMenu` (MenuId, Name, Description, Price, Category: enum Snack/Lunch/CoffeeBreak/Dinner, ImageUrl, IsAvailable)
- Tambah entity `CateringOrder` (OrderId, ReservationId, UserId, TotalAmount, Status: Ordered/Preparing/Delivered/Cancelled, DateCreated, Notes)
- Tambah entity `CateringOrderItem` (ItemId, OrderId, MenuId, Quantity, Subtotal)
- Tambah `CateringService` dan `CateringController` dengan endpoint `GET api/catering/menu`, `POST api/catering/order`, `GET api/catering/order/{id}`, `PUT api/catering/order/{id}/status`
- Modifikasi `ReservationResponse` untuk menyertakan `CateringOrder?` jika ada pesanan terkait
- Tambah `NotificationType.CateringOrderUpdate` pada enum

---

### [10.2] Pemesanan Peralatan Tambahan untuk Meeting

**Status saat ini:** Entity `Facility` merepresentasikan fasilitas permanen yang ter-install di ruangan (via `RoomFacility`). Tidak ada mekanisme meminjam peralatan tambahan (portable equipment) untuk meeting tertentu.

**Fitur baru:**
- Katalog peralatan portabel yang dapat dipinjam (projector tambahan, flipchart, mic wireless, kamera video, dll.)
- Pemesanan peralatan saat membuat reservasi
- Tracking ketersediaan peralatan portabel (untuk menghindari double-booking peralatan)
- Reminder pengembalian peralatan setelah meeting selesai

**Implementasi:**
- Tambah entity `PortableEquipment` (EquipmentId, Name, Description, TotalQuantity, AvailableQuantity, ImageUrl, IsActive)
- Tambah entity `EquipmentBooking` (BookingId, ReservationId, EquipmentId, Quantity, Status: Reserved/CheckedOut/Returned/Overdue, CheckedOutAt?, ReturnedAt?)
- Tambah `EquipmentService` dengan method `CheckAvailability()`, `ReserveEquipment()`, `CheckOutEquipment()`, `ReturnEquipment()`
- Tambah `EquipmentController` dengan endpoint `GET api/equipment`, `POST api/equipment/reserve`, `PUT api/equipment/{id}/checkout`, `PUT api/equipment/{id}/return`
- Modifikasi `ReservationRequest` untuk menambah `List<EquipmentBookingRequest>?`
- Background service untuk mendeteksi peralatan overdue dan kirim reminder via `NotificationService`

---

### [10.3] Manajemen Agenda & Notulensi Rapat

**Status saat ini:** Entity `Reservation` hanya menyimpan `Description` (string nullable) sebagai informasi tentang rapat. Tidak ada fitur pengelolaan agenda terstruktur atau pencatatan notulensi.

**Fitur baru:**
- Agenda meeting terstruktur yang bisa dibuat sebelum rapat (judul topik, durasi per topik, PIC per topik)
- Template agenda yang bisa disimpan dan digunakan ulang
- Pencatatan notulensi saat rapat berlangsung (tempel ke reservasi)
- Tracking action items dari notulensi dengan assignee dan deadline
- Distribusi notulensi otomatis ke semua attendee setelah rapat selesai via email

**Implementasi:**
- Tambah entity `MeetingAgenda` (AgendaId, ReservationId, OrderIndex, Topic, Duration, PICUserId, Notes)
- Tambah entity `MeetingMinutes` (MinutesId, ReservationId, Content, CreatedByUserId, DateCreated, DateModified)
- Tambah entity `ActionItem` (ActionItemId, MinutesId, ReservationId, Description, AssigneeUserId, Deadline, Status: Open/InProgress/Done)
- Tambah `MeetingAgendaService` dan `MeetingMinutesService`
- Tambah endpoint `POST/GET api/reservations/{id}/agenda`, `POST/GET api/reservations/{id}/minutes`, `POST/GET api/reservations/{id}/action-items`
- Modifikasi `EmailService` untuk mendukung distribusi notulensi ke attendee
- Tambah `NotificationType.ActionItemAssigned` dan `MinutesShared` pada enum

---

## 11. Multi-Tenant & Skalabilitas

### [11.1] Dukungan Multi-Cabang / Multi-Lokasi

**Status saat ini:** Aplikasi didesain untuk satu lokasi/organisasi. Tidak ada konsep cabang, kantor, atau lokasi berbeda. Semua ruangan berada dalam satu pool global.

**Fitur baru:**
- Dukungan multiple lokasi/cabang dalam satu instance aplikasi
- Setiap user ditetapkan ke lokasi utama, namun bisa melihat dan booking ruangan di lokasi lain
- Dashboard admin per lokasi dengan data yang di-filter sesuai lokasi
- Konfigurasi kebijakan booking yang berbeda per lokasi

**Implementasi:**
- Tambah entity `Location` (LocationId, Name, Address, City, TimeZone, IsActive)
- Tambah field `LocationId` (Guid) pada entity `Room`
- Tambah field `PrimaryLocationId` (Guid?) pada entity `User`
- Modifikasi semua repository dan service yang terkait Room untuk menyertakan filter location
- Modifikasi `GetRoomsRequest` untuk menambah filter `locationId`
- Modifikasi `RoomResponse` untuk menyertakan informasi lokasi
- Tambah `LocationController` dengan CRUD endpoint (Admin only)
- Modifikasi `BookingPolicy` (fitur 7.3) untuk mendukung kebijakan per lokasi

---

### [11.2] Sistem Visitor Management Terintegrasi

**Status saat ini:** `ReservationAttendee` hanya menyimpan referensi ke `User` internal yang terdaftar di sistem (via `UserId`). Reservasi tidak memiliki mekanisme untuk mengundang tamu eksternal secara terkelola.

**Fitur baru:**
- Registrasi tamu eksternal (nama, email, perusahaan, nomor identitas) saat membuat reservasi
- Generate visitor pass/badge otomatis untuk tamu
- Notifikasi keamanan/resepsionis saat tamu terdaftar dijadwalkan hadir
- Check-in tamu di lobby yang ter-link ke reservasi
- Log kunjungan tamu untuk keperluan keamanan dan compliance

**Implementasi:**
- Tambah entity `Visitor` (VisitorId, FullName, Email, Company, PhoneNumber, IdentityNumber?, DateCreated)
- Tambah entity `ReservationVisitor` (Id, ReservationId, VisitorId, VisitorPassCode, CheckInTime?, CheckOutTime?, Status: Expected/CheckedIn/CheckedOut/NoShow)
- Tambah `VisitorService` dengan method `RegisterVisitor()`, `GeneratePassCode()`, `CheckInVisitor()`, `CheckOutVisitor()`
- Tambah `VisitorController` dengan endpoint `POST api/visitors/register`, `POST api/visitors/check-in`, `GET api/reservations/{id}/visitors`
- Modifikasi `ReservationRequest` untuk menambah `List<VisitorRequest>?` (Name, Email, Company)
- Tambah `NotificationType.VisitorArrival` untuk notifikasi ke host saat tamu check-in
- Modifikasi `EmailService` untuk kirim undangan ke email tamu eksternal

---

## 12. Konfigurasi Sistem & Performa

### [12.1] Panel Konfigurasi Sistem Terpusat (Admin Settings)

**Status saat ini:** Konfigurasi aplikasi tersimpan di `appsettings.json` (koneksi database, JWT settings, Google API keys). Tidak ada panel admin untuk mengubah konfigurasi operasional tanpa mengubah file dan restart server.

**Fitur baru:**
- Panel admin web untuk mengubah konfigurasi operasional secara dinamis (tanpa restart server)
- Konfigurasi yang dapat diubah: durasi minimum/maksimum booking, jam operasional ruangan, batas advance booking, pesan pengumuman sistem
- Riwayat perubahan konfigurasi (siapa mengubah apa dan kapan)
- Fitur maintenance mode: menonaktifkan semua booking sementara untuk maintenance system

**Implementasi:**
- Tambah entity `SystemSetting` (SettingId, Key, Value, ValueType: string/int/bool/json, Category, Description, LastModifiedBy, LastModifiedAt)
- Tambah `SystemSettingRepository` dan `SystemSettingService` dengan method `GetSetting()`, `UpdateSetting()`, `GetSettingsByCategory()`
- Tambah `SystemSettingController` dengan endpoint `GET api/settings`, `PUT api/settings/{key}`, `GET api/settings/history` (Admin only)
- Implementasi `IOptionsMonitor<T>` pattern untuk hot-reload konfigurasi tanpa restart
- Modifikasi service-service terkait (ReservationService, RoomService) untuk membaca konfigurasi dari `SystemSetting` alih-alih hardcoded values

---

### [12.2] Caching & Optimisasi Performa API

**Status saat ini:** Semua request ke API langsung mengakses database (SQLite via Entity Framework Core). Tidak ada layer caching. Untuk data yang jarang berubah (daftar ruangan, fasilitas), ini menyebabkan query berulang yang tidak perlu.

**Fitur baru:**
- Response caching untuk endpoint yang data-nya jarang berubah (daftar ruangan, fasilitas, konfigurasi)
- Distributed caching untuk data yang sering diakses (status ketersediaan ruangan saat ini)
- Cache invalidation otomatis saat data terkait berubah
- Rate limiting per endpoint untuk melindungi API dari abuse

**Implementasi:**
- Implementasi `IMemoryCache` atau `IDistributedCache` (Redis) untuk caching layer
- Tambah `CacheService` wrapper dengan method `GetOrSetAsync<T>()`, `InvalidateAsync()`
- Modifikasi `RoomService.GetRoomsAsync()` dan `FacilityService` untuk menggunakan cache
- Implementasi cache invalidation pattern: saat room/facility di-update, cache terkait di-invalidate
- Tambah `ResponseCacheAttribute` pada endpoint tertentu di controller
- Implementasi rate limiting menggunakan `AspNetCoreRateLimit` middleware di `Program.cs`
- Tambah `RateLimitingMiddleware` atau konfigurasi di `Middlewares/`

---

### [12.3] Health Check & Monitoring Sistem

**Status saat ini:** Tidak ada endpoint health check untuk monitoring ketersediaan service. `Middlewares/` folder hanya berisi satu middleware. Logging menggunakan `NLog` (sesuai `nlog.config`), namun tidak ada monitoring proaktif.

**Fitur baru:**
- Health check endpoint untuk monitoring status semua komponen (database, Google Calendar API, Firebase, Redis cache)
- Dashboard status sistem untuk admin
- Alert otomatis (email/push) ke admin saat komponen mengalami masalah
- Metrik performa API (response time, error rate, request count) via structured logging

**Implementasi:**
- Tambah package `AspNetCore.HealthChecks.Sqlite`, `AspNetCore.HealthChecks.Redis` (jika Redis dipakai)
- Konfigurasi `builder.Services.AddHealthChecks()` di `Program.cs` dengan check untuk DbContext, Google API connectivity, Firebase
- Tambah endpoint `/health` dan `/health/detail` yang mengembalikan status per komponen
- Tambah custom `IHealthCheck` implementation: `GoogleCalendarHealthCheck`, `FirebaseHealthCheck`
- Tambah `SystemMonitorService` dengan method `GetSystemMetrics()`, `CheckComponentHealth()`
- Integrasikan dengan `NotificationService` untuk kirim alert saat health check gagal
- Tambah middleware untuk logging response time per request ke structured log (NLog)

---

## 13. Pelaporan Kepatuhan & Kebijakan (Compliance)

### [13.1] Sistem Penalti & Peringatan untuk No-Show

**Status saat ini:** Pengguna dapat membuat reservasi tetapi tidak ada konsekuensi jika mereka tidak hadir (no-show) atau menggunakan ruangan melebihi waktu (overbooking). Sistem tidak memonitor pelanggaran.

**Fitur baru:**
- Tracking otomatis untuk event "No-Show" (jika modul 1.3 Check-in terimplementasi)
- Sistem peringatan bertingkat otomatis via email/notifikasi kepada user yang melakukan "No-Show" berulang kali
- Penalti otomatis: penangguhan hak booking ruangan selama periode tertentu jika melewati batas pelanggaran
- Dashboard kepatuhan user untuk dilihat oleh HR atau Admin

**Implementasi:**
- Tambah entity `ComplianceInfraction` (InfractionId, UserId, ReservationId, Type: NoShow/LateCancellation/Overstay, DateOccurred)
- Tambah entity `PenaltyProfile` (PenaltyId, UserId, StartDate, EndDate, Reason, IsActive)
- Tambah `ComplianceService` dengan method `LogInfraction()`, `EvaluateUserStanding()`, `ApplyPenalty()`
- Modifikasi `ReservationService.CreateReservationAsync()` untuk memblokir user yang memiliki `PenaltyProfile` aktif
- Background service untuk me-log infraction otomatis jika reservasi expired tanpa check-in
- Tambah endpoint `GET api/compliance/users/{userId}` di `UserController` untuk admin

---

### [13.2] Persetujuan Syarat & Ketentuan Ruangan Dinamis

**Status saat ini:** Saat membuat reservasi, user tidak diharuskan menyetujui aturan khusus penggunaan ruangan. Beberapa ruangan mungkin memiliki aturan khusus (contoh: auditorium, lab komputer, ruang direksi).

**Fitur baru:**
- Admin dapat menetapkan dokumen Syarat & Ketentuan (T&C) yang berbeda untuk setiap ruangan khusus
- Pengguna wajib mencentang persetujuan (digital consent) terhadap aturan spesifik tersebut sebelum reservasi berhasil
- Log persetujuan digital untuk keperluan legal atau pertanggungjawaban jika terjadi kerusakan
- Peringatan khusus jika ruangan memerlukan persiapan ekstra (misal: dilarang makan/minum)

**Implementasi:**
- Tambah entity `RoomTerms` (TermId, RoomId, Content, Version, IsActive, DateCreated)
- Tambah entity `ReservationConsent` (ConsentId, ReservationId, UserId, TermId, ConsentTimestamp, IPAddress)
- Modifikasi `ReservationRequest` untuk menyertakan `AcceptedTermVersion`
- Tambah `ConsentService` dengan method `RecordConsent()`, `GetLatestTermsForRoom()`
- Modifikasi `ReservationService` untuk memvalidasi `AcceptedTermVersion` dengan versi T&C aktif ruangan
- Tambah endpoint `GET api/rooms/{roomId}/terms` di `RoomController`

---

## 14. Smart Building & Manajemen Energi

### [14.1] Integrasi HVAC & Lighting Bersama Reservasi

**Status saat ini:** Aplikasi hanya mencatat reservasi perangkat lunak. Tidak ada kontrol ke infrastruktur fisik gedung seperti AC atau lampu berdasarkan jadwal booking.

**Fitur baru:**
- Integrasi dengan Building Management System (BMS) untuk mengontrol AC dan lampu berdasarkan jadwal reservasi
- Lampu dan AC otomatis menyala 15 menit sebelum reservasi dimulai (Pre-cooling)
- Mode hemat energi: AC dan lampu otomatis mati setelah check-out atau jika ruangan kosong
- Analisis estimasi penghematan energi (kilowatt-hour) yang dicapai dari sistem ini

**Implementasi:**
- Tambah entity `BmsIntegrationSettings` (SettingId, RoomId, BmsDeviceId, IpAddress, Protocol: MQTT/BACnet/HTTP, IsEnabled)
- Tambah entity `EnergySavingLog` (LogId, RoomId, DateTime, EstimatedKwhSaved)
- Tambah `IBmsConnectorService` (interface untuk komunikasi ke hardware/BMS) dengan method `TurnOnHvac()`, `TurnOffHvac()`, `SetLighting()`
- Tambah background worker `SmartBuildingWorkerService` yang terus memantau jadwal di `ReservationRepository` dan mengeksekusi command BMS
- Tambah dashboard analitik penghematan energi (endpoint `GET api/analytics/energy-savings` di `AnalyticsController`)

---

## 15. Enterprise Integrations & SSO

### [15.1] Single Sign-On (SSO) Enterprise (SAML/OIDC)

**Status saat ini:** `AuthController` menangani registrasi dan login lokal (dengan JWT) dan ada draft untuk `GoogleSigninRequest`. Namun tidak ada dukungan formal untuk autentikasi level Enterprise (Corporate Active Directory / Okta / Azure AD).

**Fitur baru:**
- Autentikasi transparan menggunakan akun korporat (Microsoft Entra ID / Azure AD, Okta, atau SAML 2.0 provider)
- Auto-provisioning akun: user yang sukses login SSO otomatis dibuatkan akun dengan role default jika belum ada
- Sinkronisasi role otomatis dari group SSO ke role lokal di aplikasi
- Penarikan foto profil otomatis dari direktori korporat saat login

**Implementasi:**
- Tambah package `Microsoft.AspNetCore.Authentication.OpenIdConnect` dan/atau `Sustainsys.Saml2`
- Konfigurasi Authentication handler di `Program.cs` untuk AzureAD / OIDC
- Modifikasi `AuthController` dengan endpoint khusus `GET api/auth/sso-login`, `GET api/auth/sso-callback`
- Tambah `SsoProvisioningService` untuk menangani mapping claims dari token external ke entity `User` lokal
- Sinkronisasi `ProfilePictureUrl` dengan Microsoft Graph API (jika Azure AD) pada saat login berhasil

---

### [15.2] Sinkronisasi HRIS Otomatis (Departemen & Struktur Organisasi)

**Status saat ini:** Entity `User` berdiri sendiri. Tidak ada konsep departemen, manajer, atau struktur organisasi.

**Fitur baru:**
- Sinkronisasi harian dari sistem HRIS perusahaan (Workday, SAP SuccessFactors, atau BambooHR) untuk update departemen dan atasan langsung
- Fitur approval reservasi yang dirouting secara otomatis ke atasan langsung (Line Manager) dari requester
- Laporan utilisasi ruangan yang otomatis dipecah berdasarkan struktur organisasi terbaru

**Implementasi:**
- Tambah field `DepartmentId`, `LineManagerId` (self-referencing), `JobTitle` pada entity `User`
- Tambah entity `Department` (DepartmentId, Name, CostCenter, HeadUserId)
- Tambah background job `HrisSyncWorker` (menggunakan Quartz.NET atau Hangfire)
- Tambah `IHrisAdapterService` untuk koneksi ke API HRIS eksternal
- Modifikasi logika Approval (fitur 1.1) untuk mendukung "LineManagerApproval" selain "RoomAdminApproval"

---

## 16. Gamifikasi & Keterlibatan Karyawan

### [16.1] Sistem Poin Etiket Rapat (Meeting Etiquette Score)

**Status saat ini:** Tidak ada visibilitas perilaku atau etiket pengguna dalam menggunakan fasilitas bersama perusahaan.

**Fitur baru:**
- Sistem poin gamifikasi untuk karyawan berdasarkan perilaku booking mereka
- Penambahan poin untuk perilaku baik: Check-in tepat waktu, check-out/merilis ruangan lebih awal ("Early Release"), mengisi fasilitas review
- Pengurangan poin untuk perilaku buruk: No-show, telat, cancel mendadak (kurang dari 1 jam)
- Tampilan skor etiket pada profil user (Private) dan badges (misal: "Top Organizer")

**Implementasi:**
- Tambah entity `EtiquetteScore` (ScoreId, UserId, ReservationId, PointsAwarded, Reason, TargetBehavior: EarlyRelease/OnTime/NoShow, DateCreated)
- Tambah field `TotalEtiquetteScore` (int) pada entity `User`
- Tambah `GamificationService` dengan method `AwardPoints()`, `DeductPoints()`, `CalculateUserLevel()`
- Hook pada event check-in/check-out di modul check-in (fitur 1.3) untuk mentrigger kalkulasi poin
- Tambah endpoint `GET api/users/{userId}/etiquette-score` dan `GET api/gamification/leaderboard` (ber-anonim untuk privacy atau per departemen)
- UI Badges yang bisa di-fetch pada `UserResponse` untuk ditambahkan ke UI avatar profil

---

## 17. Fitur B2B & Co-Working Space (Komersialisasi)

### [17.1] Sistem Penagihan & Integrasi Payment Gateway

**Status saat ini:** Sistem sepenuhnya didesain untuk penggunaan internal organisasi di mana ruangan digratiskan. Tidak ada konsep harga, tagihan, atau pembayaran.

**Fitur baru:**
- Monetisasi ruangan untuk model bisnis Co-Working Space atau penyewaan eksternal
- Dukungan dynamic pricing (harga berbeda untuk jam sibuk vs jam sepi, weekend vs weekday)
- Integrasi Payment Gateway (Xendit, Midtrans, atau Stripe) untuk pembayaran online
- Auto-generate Invoice PDF yang dikirim ke email pemesan setelah reservasi
- Reservasi dengan status `Pending Payment` yang otomatis dibatalkan jika melewati batas waktu pembayaran

**Implementasi:**
- Tambah entity `Invoice` (InvoiceId, ReservationId, InvoiceNumber, TotalAmount, TaxAmount, Status: Paid/Unpaid/Cancelled, DueDate, PaymentUrl)
- Tambah entity `PaymentTransaction` (TransactionId, InvoiceId, PaymentGatewayId, Amount, PaymentMethod, Status, Timestamp)
- Modifikasi `ReservationStatus` enum untuk menambah status `PendingPayment`
- Tambah `IPaymentAdapterService` dan implementasi spesifik (misal: `XenditAdapterService`)
- Tambah `InvoiceController` dengan endpoint `GET api/invoices/{id}`, `POST api/webhooks/payment` (untuk menerima callback dari gateway)
- Background service untuk auto-cancel reservasi yang unpaid melebihi durasi batas waktu

---

### [17.2] Paket Membership & Token Booking

**Status saat ini:** Setiap user internal yang terautentikasi dapat melakukan booking (sekalipun dibatasi kuota pada fitur 7.3). Tidak ada konsep sistem prabayar atau keanggotaan.

**Fitur baru:**
- Paket berlangganan bulanan/tahunan (Membership Tiers) untuk tim atau individu
- Sistem "Booking Credits" atau Token: user membeli paket token yang akan dikurangi setiap kali membooking ruangan (berdasarkan rate ruangan)
- Alokasi token bulanan otomatis untuk corporate member
- Warning notifikasi jika saldo token tersisa sedikit

**Implementasi:**
- Tambah entity `MembershipTier` (TierId, Name, MonthlyPrice, MonthlyTokenAllowance, Perks)
- Tambah entity `UserWallet` (WalletId, UserId, CurrentBalance, LastUpdated)
- Tambah entity `WalletTransaction` (TransactionId, WalletId, Amount, TransactionType: Debit/Credit, Description, ReservationId?)
- Modifikasi `ReservationService.CreateReservationAsync()` untuk mengecek `UserWallet` dan memotong saldo jika model bisnis token diaktifkan
- Tambah `WalletService` dan `WalletController` dengan endpoint `GET api/wallet`, `POST api/wallet/topup`

---

## 18. Integrasi Workspace & Kolaborasi

### [18.1] Integrasi Slack & Microsoft Teams (Chatbot Booking)

**Status saat ini:** Interaksi hanya bisa dilakukan melalui Web Application atau secara langsung ke API. Fitur 5.1 menambahkan NLP Chatbot internal, namun tidak dapat diakses dari luar aplikasi.

**Fitur baru:**
- Bot aplikasi resmi untuk Slack dan Microsoft Teams perusahaan
- Pengguna dapat mengetik `/book-room besok jam 10 untuk 5 orang` langsung dari Slack/Teams
- Approval reservasi (bagi manager/admin) langsung dilakukan melalui tombol interaktif di dalam chat Slack/Teams tanpa harus pindah ke web
- Notifikasi meeting reminder dikirim sebagai Direct Message oleh bot

**Implementasi:**
- Tambah `SlackIntegrationService` dan `TeamsIntegrationService` yang mengimplementasikan API chatbot platform masing-masing
- Tambah `WebhookController` dengan endpoint `POST api/webhooks/slack` dan `POST api/webhooks/teams` untuk menerima slash commands dan event interaktif
- Integrasi logic parsing dari `AiChatService` (fitur 5.1) ke handler pesan webhook
- Modifikasi `NotificationTriggerService` untuk push notifikasi ke channel/DM Teams/Slack sesuai `NotificationPreference` user

---

### [18.2] Auto-Update Status Kehadiran (Presence Sync)

**Status saat ini:** Data bahwa user sedang mengikuti rapat di ruangan tertentu tidak terintegrasi ke platform kolaborasi lain.

**Fitur baru:**
- Status custom Slack/Teams user (misal: tulisan "In a Meeting" dan icon 🗓️) otomatis berubah saat jam reservasi dimulai
- Status kembali "Available" secara otomatis saat jam reservasi selesai atau user melakukan Check-out
- Opsi "Do Not Disturb" otomatis saat rapat untuk peserta VIP atau jenis ruangan tertentu (misal: Boardroom)

**Implementasi:**
- Tambah field `SlackUserId` dan `TeamsUserId` pada entity `User` (bisa diisi manual di profil atau auto-mapped via SSO)
- Tambah method `UpdateUserPresence()` pada `SlackIntegrationService` dan `TeamsIntegrationService`
- Background service (Quartz.NET) yang men-scan `ReservationRepository` menit by menit, mencari reservasi yang *Started* atau *Ended* untuk mentrigger perubahan status
- Opsi toggle "Sync Status" di pengaturan profil user

---

## 19. Manajemen Layanan Tambahan (Ancillary Services)

### [19.1] Request Pembersihan Ekstra (On-Demand Cleaning)

**Status saat ini:** Tidak ada alur kerja operasional untuk tim fasilitas (Facility / Cleaning Service). Pembersihan diasumsikan terjadi di luar sistem secara terjadwal.

**Fitur baru:**
- Fitur bagi partisipan rapat untuk request "Pembersihan Ruangan Segera" baik sebelum atau sesudah meeting jika kondisi ruangan berantakan
- Fitur bagi admin ruangan untuk men-set "Mandatory Buffer-Time" untuk pembersihan khusus (misal jika ada pesanan makanan/katering)
- Dashboard/App khusus untuk tim Cleaning Service melihat task list pembersihan per ruangan

**Implementasi:**
- Tambah entity `CleaningTask` (TaskId, RoomId, ReservationId?, Status: Request/InProgress/Done, RequestedBy, Priority)
- Tambah method `RequestCleaning()` di `FacilityService` atau `RoomService`
- Modifikasi logika pengecekan konflik reservasi (`ReservationService`) agar menyisipkan automatically added buffer time (+15 menit) setelah reservasi yang memesan Katering
- Tambah `CleaningController` dengan endpoint `POST api/rooms/{roomId}/clean`, `GET api/cleaning-tasks`
- Tambah role baru `CleaningStaff` di sistem User

---

### [19.2] Manajemen Akses Parkir Tamu Rapat

**Status saat ini:** Fitur 11.2 (Sistem Visitor Management) mendata identitas tamu, namun tidak terhubung dengan fasilitas gedung lainnya seperti parkir.

**Fitur baru:**
- Kuota slot parkir VIP/Tamu khusus yang bisa dibooking secara terintegrasi dengan reservasi ruangan
- Input pelat nomor kendaraan tamu saat membuat reservasi
- Barcode/QR Code akses parkir otomatis dikirimkan ke email tamu
- Integrasi data ke sistem parkir gedung (jika ada API) atau dashboard khusus sekuriti/petugas parkir

**Implementasi:**
- Tambah entity `ParkingSlot` (SlotId, SlotNumber, IsVip)
- Tambah entity `ParkingBooking` (BookingId, ReservationId, VisitorId?, VehicleLicensePlate, SlotId, Status)
- Modifikasi `ReservationRequest` dengan field `VehicleLicensePlates: List<string>`
- Tambah `ParkingService` dengan method `ReserveParking()`, `ReleaseParking()`
- Tambah `SecurityController` (role Security) endpoint `GET api/parking/expected-vehicles` untuk sekuriti lobby mengecek plat kendaraan tamu rapat pada hari itu
