# Future Development: Sistem Manajemen Inventaris & Peminjaman Aset Perusahaan

## 1. Manajemen Item & Katalog Inventaris

### [1.1] Barcode & QR Code Tagging untuk Item

**Status saat ini:** Sistem hanya menggunakan `ItemCode` berupa string manual tanpa standar barcode. Tidak ada mekanisme pemindaian (scanning) untuk identifikasi item secara cepat. Proses pencarian item masih bergantung pada input teks manual di halaman katalog.

**Fitur baru:**
- Generasi otomatis barcode (Code-128) dan QR code unik untuk setiap item berdasarkan `ItemCode`, `CategoryCode`, dan `SubCategoryCode` yang sudah ada
- Fitur scan barcode/QR code via kamera perangkat (mobile/webcam) untuk pencarian item instan, proses peminjaman, dan verifikasi pengembalian
- Cetak label barcode dalam format batch (PDF) untuk keperluan pelabelan fisik aset

**Implementasi:**
- Tambah field `BarcodeData` dan `QrCodePath` pada entity `Item`
- Buat `BarcodeService` untuk generate barcode menggunakan library `ZXing.Net` atau `BarcodeLib`
- Buat `BarcodeController` di Area Admin dengan endpoint generate, print batch, dan scan
- Tambah endpoint API `/api/scan/{code}` untuk lookup item via scan
- Integrasi JavaScript library `html5-qrcode` pada View untuk fitur scan dari browser

---

### [1.2] Manajemen Lokasi & Rak Penyimpanan (Storage Location)

**Status saat ini:** Sistem hanya memiliki entitas `City` sebagai lokasi item, tanpa granularitas lebih lanjut untuk menentukan lokasi fisik penyimpanan item (gedung, lantai, rak, slot). Admin tidak dapat melacak di mana tepatnya sebuah item disimpan secara fisik.

**Fitur baru:**
- Hierarki lokasi penyimpanan: City → Building → Floor → Rack → Slot
- Penentuan lokasi penyimpanan default untuk setiap item baru
- Fitur pindah lokasi (relocation) item antar rak/gedung dengan pencatatan riwayat perpindahan
- Visualisasi peta lokasi item secara sederhana (grid layout per rak)

**Implementasi:**
- Buat entity baru: `Building`, `Floor`, `StorageRack`, `StorageSlot` dengan relasi hierarki ke `City`
- Tambah field `StorageSlotId` pada entity `Item`
- Buat `ItemRelocationHistory` entity untuk mencatat riwayat perpindahan
- Buat `StorageLocationController` dan `StorageLocationService` di Area Admin
- Buat View baru untuk manajemen hierarki lokasi dan peta visual rak

---

### [1.3] Manajemen Masa Pakai & Depresiasi Aset

**Status saat ini:** Entity `Item` tidak memiliki informasi mengenai tanggal pembelian, harga perolehan, masa pakai (useful life), maupun status depresiasi aset. Tidak ada mekanisme untuk menandai item yang sudah melewati masa pakai berguna.

**Fitur baru:**
- Pencatatan informasi aset: tanggal pembelian, harga perolehan, vendor/supplier, nomor faktur
- Kalkulasi otomatis depresiasi aset (metode garis lurus) berdasarkan masa pakai berguna
- Notifikasi otomatis ketika item mendekati atau melewati akhir masa pakai
- Status lifecycle aset: Active, Maintenance, Retired, Disposed

**Implementasi:**
- Tambah field pada entity `Item`: `PurchaseDate`, `PurchasePrice`, `SupplierName`, `InvoiceNumber`, `UsefulLifeMonths`, `AssetLifecycleStatus`
- Buat enum `AssetLifecycleStatusEnum` (Active, Maintenance, Retired, Disposed)
- Buat `AssetDepreciationService` untuk kalkulasi nilai buku aset
- Buat `AssetDepreciationDTO` untuk response kalkulasi
- Tambah endpoint di `ItemViewController` untuk laporan depresiasi
- Buat background job (Hangfire/hosted service) untuk cek masa pakai dan kirim notifikasi

---

### [1.4] Import Data Inventaris via File dengan Validasi Lanjutan

**Status saat ini:** Sistem sudah memiliki `CsvService`, `ExcelService`, dan `FileImportService` di Area Admin, namun kemampuan validasi saat import masih terbatas. Tidak ada mekanisme preview data sebelum commit, deteksi duplikasi, maupun error reporting yang komprehensif per baris.

**Fitur baru:**
- Preview data import sebelum commit ke database (staging area)
- Validasi lanjutan: deteksi duplikasi berdasarkan `ItemCode`, validasi `CategoryCode`/`SubCategoryCode` terhadap master data
- Error report per baris dengan detail kolom dan jenis error
- Template file download (CSV/Excel) dengan contoh data dan instruksi pengisian
- Rollback capability jika import gagal di tengah proses

**Implementasi:**
- Buat entity `ImportBatch` dan `ImportBatchItem` untuk staging area
- Tambah `ImportValidationService` dengan rule engine sederhana
- Modifikasi `FileImportService` untuk support staging → validate → commit flow
- Buat `ImportPreviewDTO` dan `ImportErrorDTO`
- Tambah View baru untuk halaman preview import dengan tabel interaktif
- Tambah endpoint download template di `ItemViewController`

---

### [1.5] Multi-Image & Dokumen Pendukung Item

**Status saat ini:** Entity `Item` hanya memiliki satu field `PicturePath` untuk satu gambar. Tidak ada dukungan untuk multiple foto (berbagai sudut aset) maupun lampiran dokumen pendukung seperti manual, garansi, atau sertifikat kalibrasi.

**Fitur baru:**
- Upload multiple gambar per item (foto depan, belakang, detail label, kondisi terkini)
- Upload dokumen pendukung (PDF manual, sertifikat garansi, dokumen kalibrasi)
- Gallery view untuk melihat semua gambar item
- Tracking masa berlaku dokumen (misal garansi, kalibrasi) dengan notifikasi expired

**Implementasi:**
- Buat entity `ItemImage` (ItemId, ImagePath, Caption, UploadDate, IsPrimary)
- Buat entity `ItemDocument` (ItemId, DocumentPath, DocumentType, ExpiryDate, UploadDate)
- Buat enum `DocumentTypeEnum` (Manual, Warranty, Calibration, Invoice, Other)
- Modifikasi `ItemViewAdminService` untuk handle multi-file upload
- Tambah `ItemImageDTO` dan `ItemDocumentDTO`
- Buat partial view gallery component dan document list component

---

## 2. Alur Peminjaman & Pengembalian (Request Workflow)

### [2.1] Multi-Item Request (Peminjaman Banyak Item Sekaligus)

**Status saat ini:** Entity `Request` hanya mendukung peminjaman satu item per request (`ItemId` tunggal). Karyawan yang ingin meminjam beberapa item sekaligus harus membuat request terpisah untuk masing-masing item, yang tidak efisien.

**Fitur baru:**
- Mekanisme keranjang peminjaman (borrow cart) dimana karyawan bisa menambahkan beberapa item sebelum submit request
- Submit satu request yang mencakup beberapa item sekaligus
- Admin dapat approve/reject per item dalam satu request atau approve/reject seluruh request batch

**Implementasi:**
- Buat entity `RequestGroup` sebagai parent grouping (RequestGroupId, UserId, RequestDate, GroupNote, Status)
- Tambah field `RequestGroupId` pada entity `Request` sebagai foreign key opsional
- Buat `BorrowCartDTO` dan `CartItemDTO` untuk sesi keranjang
- Modifikasi `RequestEmployeeService` untuk support batch submission
- Modifikasi `RequestAdminService` untuk approve/reject batch
- Buat View keranjang peminjaman di Area Employee
- Update `RequestController` di kedua Area untuk handle multi-item flow

---

### [2.2] Perpanjangan Masa Peminjaman (Extend Borrow Period)

**Status saat ini:** Entity `ActiveRequest` memiliki `RequestDays` dan `RemainingDays`, namun tidak ada fitur bagi karyawan untuk mengajukan perpanjangan masa peminjaman. Jika masa peminjaman habis, status langsung berubah ke `Late` tanpa opsi extend.

**Fitur baru:**
- Karyawan dapat mengajukan perpanjangan peminjaman sebelum atau sesudah jatuh tempo
- Admin menerima notifikasi dan dapat approve/reject perpanjangan
- Batas maksimum perpanjangan yang bisa dikonfigurasi oleh admin
- Riwayat perpanjangan tercatat untuk audit

**Implementasi:**
- Buat entity `ExtensionRequest` (ExtensionId, ActiveRequestId, RequestedDays, Status, RequestDate, ProcessedDate, AdminNote)
- Buat enum `ExtensionStatusEnum` (Pending, Approved, Rejected)
- Tambah `MaxExtensionDays` dan `MaxExtensionCount` sebagai configurable setting
- Buat `ExtensionRequestService` di Area Employee dan Admin
- Buat `ExtensionRequestController` di kedua Area
- Modifikasi `ActiveRequestService` untuk update `ReturnDate` dan `RemainingDays` setelah perpanjangan diapprove
- Kirim notifikasi via `NotificationHub` pada setiap perubahan status perpanjangan

---

### [2.3] Approval Bertingkat (Multi-Level Approval)

**Status saat ini:** Workflow approval hanya satu tingkat: karyawan submit → admin approve/reject. Tidak ada mekanisme approval chain, misalnya untuk item bernilai tinggi yang memerlukan persetujuan manager dan/atau kepala departemen.

**Fitur baru:**
- Konfigurasi approval chain berdasarkan kategori item, nilai aset, atau jumlah kuantitas
- Role tambahan: Supervisor/Manager sebagai approver tingkat pertama
- Eskalasi otomatis jika approver tidak merespons dalam batas waktu tertentu
- Dashboard approval queue per level

**Implementasi:**
- Buat entity `ApprovalChain` (ChainId, CategoryId, MinQuantity, MinAssetValue)
- Buat entity `ApprovalLevel` (LevelId, ChainId, ApproverRole, LevelOrder, TimeoutHours)
- Buat entity `ApprovalRecord` (RecordId, RequestId/ReservationId, LevelId, ApproverId, Status, ProcessedAt, Note)
- Tambah role baru via `AllRoles.cs`: "Supervisor", "Manager"
- Buat `ApprovalChainService` untuk routing request ke approver yang tepat
- Modifikasi `RequestAdminService` untuk cek approval chain sebelum finalize
- Implementasi background job untuk eskalasi timeout

---

### [2.4] Sistem Denda & Penalti Keterlambatan

**Status saat ini:** `ActiveRequestStatusEnum` memiliki status `Late` namun tidak ada konsekuensi atau mekanisme penalti untuk keterlambatan pengembalian. Sistem hanya menandai status tanpa tindak lanjut terukur.

**Fitur baru:**
- Konfigurasi tarif denda per hari keterlambatan (bisa per kategori item)
- Kalkulasi otomatis total denda berdasarkan jumlah hari terlambat
- Dashboard denda per karyawan
- Blokir hak peminjaman baru jika denda belum diselesaikan (threshold konfigurasi)
- Laporan keterlambatan dan denda untuk evaluasi manajemen

**Implementasi:**
- Buat entity `PenaltyConfig` (ConfigId, CategoryId, DailyRate, Currency, GracePeriodDays)
- Buat entity `PenaltyRecord` (RecordId, ActiveRequestId, UserId, Amount, CalculatedDays, IsSettled, SettledDate)
- Tambah field `TotalPenalty` dan `HasUnsettledPenalty` pada view model user
- Buat `PenaltyService` untuk kalkulasi dan manajemen denda
- Modifikasi `RequestEmployeeService` untuk cek status denda sebelum izinkan request baru
- Buat `PenaltyController` di Area Admin untuk manajemen dan settlement denda
- Buat View laporan denda

---

### [2.5] Peminjaman Item Konsumabel dengan Pelacakan Penggunaan

**Status saat ini:** `ItemTypeEnum` membedakan Borrowable dan Consumable, namun alur peminjaman (Request → ActiveRequest → Return) dirancang utama untuk item borrowable. Item consumable tidak memiliki alur khusus yang mencerminkan penggunaannya (konsumsi langsung, bukan dikembalikan).

**Fitur baru:**
- Alur request consumable terpisah: Request → Approved → Distributed (tanpa alur pengembalian)
- Pencatatan distribusi consumable: siapa, berapa banyak, kapan
- Auto-deduction stok setelah distribusi disetujui
- Statistik konsumsi per karyawan, per item, per periode
- Alert otomatis ketika pola konsumsi abnormal (melebihi rata-rata)

**Implementasi:**
- Buat entity `ConsumableDistribution` (DistributionId, ItemId, UserId, Quantity, DistributionDate, ApprovedBy)
- Modifikasi `RequestAdminService` untuk route ke alur distribusi jika item bertipe Consumable
- Buat `ConsumableDistributionService` dengan logic auto-deduct dari `Item.Quantity`
- Buat `ConsumableReportDTO` untuk statistik konsumsi
- Tambah endpoint di `RequestController` Area Admin untuk proses distribusi
- Tambah View distribusi consumable terpisah dari alur peminjaman

---

## 3. Sistem Reservasi Lanjutan

### [3.1] Kalender Ketersediaan Item (Availability Calendar)

**Status saat ini:** Entity `Reservation` memiliki `PickupDate` dan `RequestDays`, namun tidak ada visualisasi kalender ketersediaan item. Karyawan tidak bisa melihat jadwal ketersediaan item sebelum membuat reservasi, sehingga berpotensi terjadi tumpang tindih.

**Fitur baru:**
- Kalender visual interaktif yang menampilkan slot ketersediaan per item
- Warna berbeda untuk status: tersedia (hijau), direservasi (kuning), sedang dipinjam (merah)
- Karyawan memilih tanggal pickup langsung dari kalender
- Deteksi otomatis konflik jadwal saat submit reservasi

**Implementasi:**
- Buat `AvailabilityService` yang menghitung ketersediaan berdasarkan `Reservation` dan `ActiveRequest` yang aktif
- Buat `AvailabilitySlotDTO` (Date, AvailableQuantity, TotalQuantity, Status)
- Buat endpoint API `GET /api/items/{id}/availability?from=&to=` untuk data kalender
- Integrasi JavaScript library `FullCalendar` pada View reservasi
- Modifikasi `ReservationEmployeeService` untuk validasi konflik sebelum save
- Buat partial view komponen kalender yang bisa di-reuse

---

### [3.2] Reservasi Berulang (Recurring Reservation)

**Status saat ini:** Setiap reservasi bersifat one-time. Karyawan yang secara rutin membutuhkan item tertentu (misal setiap Senin) harus membuat reservasi baru setiap minggu secara manual.

**Fitur baru:**
- Opsi pembuatan reservasi berulang: harian, mingguan, bulanan
- Konfigurasi rentang pengulangan (misal setiap Senin selama 3 bulan)
- Admin dapat melihat dan mengelola seluruh seri reservasi berulang
- Pembatalan satu instance tanpa membatalkan seluruh seri

**Implementasi:**
- Buat entity `RecurringReservationPattern` (PatternId, ItemId, UserId, FrequencyType, Interval, StartDate, EndDate, DayOfWeek)
- Tambah field `RecurringPatternId` pada entity `Reservation` sebagai foreign key opsional
- Buat enum `RecurrenceFrequencyEnum` (Daily, Weekly, BiWeekly, Monthly)
- Buat `RecurringReservationService` untuk generate instance reservasi berdasarkan pola
- Modifikasi `ReservationController` di Area Employee untuk opsi recurring
- Buat View form reservasi berulang dengan preview jadwal

---

### [3.3] Waitlist & Antrian Otomatis

**Status saat ini:** Jika item sudah tidak tersedia (stok habis atau semua unit sedang dipinjam), karyawan tidak bisa mendaftar antrian. Mereka harus terus mengecek manual kapan item tersedia kembali.

**Fitur baru:**
- Tombol "Masuk Antrian" ketika item tidak tersedia
- Notifikasi otomatis ke karyawan pertama di antrian saat item tersedia kembali
- Batas waktu klaim (misal 24 jam) sebelum dialihkan ke antrian berikutnya
- Dashboard antrian untuk admin dan karyawan

**Implementasi:**
- Buat entity `Waitlist` (WaitlistId, ItemId, UserId, JoinDate, Position, Status, NotifiedAt, ClaimDeadline)
- Buat enum `WaitlistStatusEnum` (Waiting, Notified, Claimed, Expired, Cancelled)
- Buat `WaitlistService` untuk manajemen antrian dengan logic FIFO
- Hook ke event pengembalian item di `ActiveRequestService` untuk trigger notifikasi waitlist
- Buat `WaitlistController` di Area Employee dan Admin
- Kirim notifikasi via `NotificationHub` dan email via `EmailSender`

---

## 4. Manajemen Restock & Supply Chain

### [4.1] Restock Otomatis Berdasarkan Threshold

**Status saat ini:** Entity `Item` memiliki field `QuantityLow` sebagai batas minimum stok, namun tidak ada mekanisme otomatis untuk membuat restock request ketika stok mencapai batas minimum. Entity `Restock` ada namun `RestockService` sangat minim (hanya 797 bytes).

**Fitur baru:**
- Monitoring stok real-time yang membandingkan `Quantity` dengan `QuantityLow`
- Auto-generate `Restock` request ketika stok jatuh di bawah `QuantityLow`
- Konfigurasi reorder quantity (jumlah optimal untuk di-restock)
- Dashboard monitoring stok dengan indikator visual (hijau/kuning/merah)
- Email alert ke admin inventory saat stok kritis

**Implementasi:**
- Tambah field `ReorderQuantity` dan `AutoRestockEnabled` pada entity `Item`
- Buat `StockMonitoringService` dengan background job (IHostedService) untuk periodic check
- Modifikasi `RestockService` untuk auto-create restock request
- Buat `StockAlertDTO` dan `StockDashboardDTO`
- Tambah endpoint `GET /admin/stock-monitor` di `RestockController`
- Buat View dashboard monitoring stok dengan chart dan color-coded indicators
- Kirim email alert via `EmailSender` dan notifikasi via `NotificationHub`

---

### [4.2] Manajemen Supplier & Purchase Order

**Status saat ini:** Tidak ada entity atau mekanisme untuk mengelola data supplier/vendor. Proses restock tidak memiliki tracking purchase order, sehingga admin tidak bisa melacak status pesanan ke supplier.

**Fitur baru:**
- Master data supplier: nama, kontak, alamat, kategori item yang di-supply
- Pembuatan Purchase Order (PO) dari restock request
- Tracking status PO: Draft → Submitted → Confirmed → Shipped → Received
- Riwayat pembelian per supplier dan per item
- Evaluasi performa supplier berdasarkan lead time dan fill rate

**Implementasi:**
- Buat entity `Supplier` (SupplierId, Name, ContactPerson, Email, Phone, Address, IsActive)
- Buat entity `SupplierItem` (SupplierId, ItemId, UnitPrice, LeadTimeDays) untuk relasi many-to-many
- Buat entity `PurchaseOrder` (POId, SupplierId, PODate, Status, TotalAmount, ExpectedDeliveryDate, ReceivedDate)
- Buat entity `PurchaseOrderItem` (POItemId, POId, ItemId, Quantity, UnitPrice)
- Buat enum `PurchaseOrderStatusEnum` (Draft, Submitted, Confirmed, Shipped, Received, Cancelled)
- Tambah field `SupplierId` opsional pada entity `Restock`
- Buat `SupplierController`, `PurchaseOrderController`, dan service terkait di Area Admin
- Buat View CRUD supplier dan manajemen PO

---

### [4.3] Stock Opname / Stock Taking Digital

**Status saat ini:** Tidak ada fitur stock opname digital. Proses pengecekan stok fisik vs stok sistem harus dilakukan secara manual tanpa alat bantu pencatatan selisih dan rekonsiliasi dalam sistem.

**Fitur baru:**
- Pembuatan sesi stock opname per lokasi/kategori dengan tanggal jadwal
- Input hasil hitung fisik per item (bisa via scan barcode jika fitur 1.1 tersedia)
- Kalkulasi otomatis selisih (variance) antara stok sistem dan stok fisik
- Approval adjustment oleh admin untuk rekonsiliasi
- Laporan hasil stock opname dengan detail selisih dan penyesuaian

**Implementasi:**
- Buat entity `StockOpnameSession` (SessionId, SessionDate, LocationId, Status, CreatedBy, CompletedAt)
- Buat entity `StockOpnameItem` (Id, SessionId, ItemId, SystemQuantity, PhysicalQuantity, Variance, AdjustmentApproved, Note)
- Buat enum `StockOpnameStatusEnum` (Planned, InProgress, PendingReview, Completed)
- Buat `StockOpnameService` untuk generate daftar item per sesi dan kalkulasi variance
- Buat `StockOpnameController` di Area Admin
- Modifikasi `ItemRepository` untuk update quantity setelah adjustment disetujui
- Buat View form input stock opname dan laporan hasil

---

## 5. Riwayat & Pelacakan Kondisi Item

### [5.1] Pelacakan Kondisi Item Secara Berkala (Condition Tracking)

**Status saat ini:** `ItemHistory` hanya mencatat status saat pengembalian melalui `ItemStatusEnum` (WellReturned, Damaged, Lost, Consumed). Tidak ada mekanisme pencatatan kondisi item secara berkala di luar event pengembalian, maupun grading kondisi yang lebih granular.

**Fitur baru:**
- Penilaian kondisi item secara berkala oleh admin (misal setiap 3 bulan)
- Skala kondisi: Excellent, Good, Fair, Poor, Unserviceable
- Foto bukti kondisi terkini saat assessment
- Riwayat perubahan kondisi item dari waktu ke waktu (timeline view)
- Auto-flag item dengan kondisi Poor/Unserviceable untuk review disposal

**Implementasi:**
- Buat entity `ConditionAssessment` (AssessmentId, ItemId, AssessedBy, AssessmentDate, ConditionGrade, PhotoPath, Notes)
- Buat enum `ConditionGradeEnum` (Excellent, Good, Fair, Poor, Unserviceable)
- Buat `ConditionAssessmentService` di Area Admin
- Buat `ConditionAssessmentController` di Area Admin
- Buat View timeline kondisi item dengan foto
- Tambah background job reminder untuk assessment berkala

---

### [5.2] Laporan Kerusakan & Maintenance Request

**Status saat ini:** `ItemStatusEnum` memiliki status `Damaged` namun tidak ada alur tindak lanjut setelah item dilaporkan rusak. Tidak ada mekanisme untuk membuat maintenance request, melacak perbaikan, atau mencatat biaya perbaikan.

**Fitur baru:**
- Karyawan dan admin dapat membuat laporan kerusakan (damage report) dengan foto bukti
- Alur maintenance request: Reported → Assigned → InProgress → Completed → Verified
- Pencatatan biaya perbaikan dan vendor yang mengerjakan
- Item secara otomatis ditarik dari ketersediaan selama proses maintenance
- Riwayat seluruh maintenance per item

**Implementasi:**
- Buat entity `DamageReport` (ReportId, ItemId, ReportedBy, ReportDate, Description, PhotoPath, Severity)
- Buat entity `MaintenanceRequest` (MaintenanceId, DamageReportId, ItemId, AssignedTo, Status, StartDate, CompletedDate, Cost, VendorName, Notes)
- Buat enum `MaintenancePriorityEnum` (Low, Medium, High, Critical)
- Buat enum `MaintenanceStatusEnum` (Reported, Assigned, InProgress, Completed, Verified)
- Buat `MaintenanceService` untuk manajemen lifecycle maintenance
- Modifikasi `Item.Availability` secara otomatis saat item masuk maintenance
- Buat `DamageReportController` di Area Employee dan `MaintenanceController` di Area Admin
- Buat View form laporan kerusakan dan dashboard maintenance

---

### [5.3] Disposal & Penghapusan Aset (Asset Write-Off)

**Status saat ini:** Entity `Item` memiliki field `IsDeleted` sebagai soft delete, namun tidak ada proses formal disposal/penghapusan aset yang mencakup alasan, bukti, approval, dan pencatatan legal.

**Fitur baru:**
- Alur disposal formal: Propose Disposal → Review → Approve → Execute → DocumentRecord
- Pencatatan alasan disposal (rusak tidak ekonomis, hilang, obsolete, expired)
- Upload bukti dokumentasi (foto kondisi akhir, berita acara pemusnahan)
- Approval disposal oleh authorized personnel
- Laporan aset yang di-dispose untuk keperluan akuntansi dan audit

**Implementasi:**
- Buat entity `DisposalRequest` (DisposalId, ItemId, RequestedBy, Reason, DisposalMethod, DocumentPath, Status, ApprovedBy, ExecutedDate)
- Buat enum `DisposalReasonEnum` (Damaged, Obsolete, Lost, Expired, Other)
- Buat enum `DisposalMethodEnum` (Sold, Donated, Scrapped, Returned)
- Buat enum `DisposalStatusEnum` (Proposed, Reviewed, Approved, Executed, Cancelled)
- Buat `DisposalService` dan `DisposalController` di Area Admin
- Modifikasi `Item.IsDeleted` dan `Item.Availability` setelah disposal executed
- Buat View form disposal dan laporan disposal

---

## 6. Pengajuan Item Baru (Item Propose) Lanjutan

### [6.1] Template Pengajuan berdasarkan Kategori

**Status saat ini:** Entity `ItemPropose` memiliki field umum untuk semua jenis pengajuan. Tidak ada perbedaan template atau field khusus berdasarkan kategori item yang diajukan (misal IT equipment vs Office Supplies memiliki kebutuhan informasi berbeda).

**Fitur baru:**
- Template pengajuan dinamis berdasarkan Category yang dipilih
- Field tambahan spesifik per kategori (misal untuk IT: spesifikasi teknis, OS requirement; untuk furniture: dimensi, material)
- Panduan pengisian yang berbeda per template kategori
- Estimasi harga otomatis dari data supplier (jika terintegrasi fitur 4.2)

**Implementasi:**
- Buat entity `ProposeTemplate` (TemplateId, CategoryId, TemplateName, IsActive)
- Buat entity `ProposeTemplateField` (FieldId, TemplateId, FieldName, FieldType, IsRequired, DisplayOrder)
- Buat entity `ProposeFieldValue` (ValueId, ItemProposeId, FieldId, Value)
- Modifikasi `ItemProposeService` di Area Employee untuk dynamic form rendering
- Modifikasi `ItemProposeAdminService` untuk display field tambahan saat review
- Buat `ProposeTemplateController` di Area Admin untuk manajemen template
- Buat View form pengajuan dinamis dengan JavaScript template switching

---

### [6.2] Voting & Prioritas Pengajuan oleh Karyawan

**Status saat ini:** `ItemPropose` dengan status `WaitingApproval` hanya menunggu keputusan admin tanpa insight apa pun mengenai seberapa banyak karyawan lain yang juga membutuhkan item tersebut. `ProposeLimit` membatasi jumlah pengajuan per karyawan, namun tidak ada mekanisme kolaboratif.

**Fitur baru:**
- Karyawan dapat memberikan upvote pada pengajuan item karyawan lain yang relevan
- Dashboard admin menampilkan pengajuan diurutkan berdasarkan jumlah vote
- Karyawan melihat daftar pengajuan aktif dan memberikan komentar/alasan kebutuhan
- Notifikasi ke proposer saat pengajuannya mendapat vote

**Implementasi:**
- Buat entity `ProposeVote` (VoteId, ItemProposeId, UserId, VoteDate)
- Buat entity `ProposeComment` (CommentId, ItemProposeId, UserId, Content, CreatedAt)
- Modifikasi `ItemProposeAdminService` untuk include vote count dan sort
- Buat `ProposeVoteController` di Area Employee dengan endpoint vote/unvote
- Modifikasi View pengajuan di Area Employee untuk tampilkan daftar dan tombol vote
- Modifikasi View review pengajuan di Area Admin untuk tampilkan vote count dan komentar
- Notifikasi via `NotificationHub`

---

### [6.3] Budget Tracking untuk Pengajuan

**Status saat ini:** Proses pengajuan item baru (`ItemPropose`) tidak terintegrasi dengan informasi budget. Admin menyetujui/menolak pengajuan tanpa visibility terhadap alokasi anggaran departemen atau perusahaan.

**Fitur baru:**
- Konfigurasi budget inventaris per periode (bulanan/kuartalan/tahunan)
- Estimasi biaya pada setiap pengajuan
- Tracking penggunaan budget secara real-time (budget used vs remaining)
- Warning saat pengajuan akan melebihi sisa budget
- Laporan penggunaan budget per periode dan per kategori

**Implementasi:**
- Buat entity `BudgetPeriod` (BudgetId, PeriodStart, PeriodEnd, TotalBudget, UsedBudget)
- Buat entity `BudgetCategory` (Id, BudgetId, CategoryId, AllocatedAmount, UsedAmount)
- Tambah field `EstimatedCost` pada entity `ItemPropose`
- Buat `BudgetService` untuk tracking dan validasi budget
- Modifikasi `ItemProposeAdminService` untuk cek budget saat approve
- Buat `BudgetController` di Area Admin
- Buat View dashboard budget dengan chart penggunaan

---

## 7. Dashboard & Reporting Analytics

### [7.1] Dashboard Analitik yang Diperkaya

**Status saat ini:** Sistem memiliki `ChartView` entity dengan konfigurasi MonthCount dan `ChartViewService` serta `HomeAdminService`/`HomeEmployeeService`, namun analitik masih terbatas pada kustomisasi rentang bulan. Belum ada insight mendalam seperti tren peminjaman, item paling populer, utilization rate, atau prediktif analytics.

**Fitur baru:**
- Widget dashboard: Top Borrowed Items, Category Distribution, Request Trend, Utilization Rate
- Filter analitik per City, Category, SubCategory, dan rentang waktu kustom
- Grafik tren peminjaman harian/mingguan/bulanan
- Item utilization rate (persentase waktu item dipinjam vs tersedia)
- Highlight anomali: item yang sering rusak, karyawan dengan overdue terbanyak

**Implementasi:**
- Modifikasi `HomeAdminService` untuk kalkulasi metrik baru
- Buat `AnalyticsDashboardDTO` dengan sub-DTO untuk tiap widget
- Buat `ItemUtilizationDTO` (ItemId, ItemName, TotalBorrowDays, TotalAvailableDays, UtilizationRate)
- Modifikasi `ChartViewService` untuk support multiple chart types
- Integrasi JavaScript charting library `Chart.js` atau upgrade ke `ApexCharts`
- Buat partial views untuk masing-masing widget dashboard
- Tambah endpoint API untuk data masing-masing widget

---

### [7.2] Laporan Export yang Komprehensif

**Status saat ini:** `CsvService` dan `ExcelService` sudah ada tetapi kemampuan reporting terbatas. Belum ada fitur generate laporan dengan filter kustom, laporan periodik otomatis, atau format PDF untuk keperluan formal.

**Fitur baru:**
- Report builder dengan filter kustom (tanggal, kategori, lokasi, status)
- Export laporan dalam format CSV, Excel, dan PDF
- Template laporan pre-built: Inventory Summary, Borrow Activity, Overdue Items, Restock History, Asset Depreciation
- Scheduled report (penjadwalan otomatis laporan periodik via email)
- Laporan audit trail dari `AuditLog`

**Implementasi:**
- Buat `ReportBuilderService` dengan template pattern
- Buat `ReportFilterDTO` (DateFrom, DateTo, CategoryIds, CityIds, StatusFilter, ReportType)
- Integrasi library `QuestPDF` atau `DinkToPdf` untuk generate PDF
- Modifikasi `CsvService` dan `ExcelService` untuk accept custom filter
- Buat `ReportController` di Area Admin dengan endpoint per report type
- Buat `ScheduledReportConfig` entity dan background job untuk scheduled reports
- Buat View report builder dengan form filter dinamis

---

### [7.3] Real-Time Dashboard Metrics dengan SignalR

**Status saat ini:** `NotificationHub` sudah ada tetapi hanya digunakan untuk notifikasi. Tidak ada penggunaan SignalR untuk update dashboard real-time saat terjadi perubahan data (misal item baru dipinjam, stok berkurang, request masuk).

**Fitur baru:**
- Update dashboard admin secara real-time tanpa refresh halaman
- Counter live: total request pending, active borrow, overdue items, low stock alerts
- Feed aktivitas terkini (live activity feed) di dashboard
- Notifikasi visual (toast/badge) untuk event penting

**Implementasi:**
- Buat `DashboardHub` extends Hub untuk broadcast data dashboard
- Buat `DashboardUpdateDTO` dengan counter dan activity feed data
- Modifikasi service-service utama (`RequestAdminService`, `ActiveRequestService`, `RestockService`) untuk broadcast ke `DashboardHub` setelah setiap operasi penting
- Buat JavaScript client yang subscribe ke `DashboardHub` dan update DOM secara real-time
- Implementasi debounce mechanism agar tidak overload broadcast
- Buat partial view komponen live counter dan activity feed

---

## 8. Notifikasi & Komunikasi Lanjutan

### [8.1] Template Email yang Dikustomisasi per Event

**Status saat ini:** Direktori `EmailTemplates` ada dengan 3 template. `EmailSender` di Utilities mengirim email namun belum ada mekanisme kustomisasi template email tanpa mengubah kode. Template terbatas untuk beberapa event saja.

**Fitur baru:**
- Template email untuk setiap event bisnis: request approved/rejected, reservation confirmed, overdue reminder, restock alert, propose status update
- Admin dapat mengkustomisasi subject dan body template dari UI (tanpa ubah kode)
- Placeholder dinamis: {{userName}}, {{itemName}}, {{dueDate}}, {{adminNote}}
- Preview template sebelum simpan

**Implementasi:**
- Buat entity `EmailTemplate` (TemplateId, EventType, Subject, BodyHtml, IsActive, LastModifiedAt)
- Buat enum `EmailEventTypeEnum` (RequestApproved, RequestRejected, ReservationConfirmed, OverdueReminder, RestockAlert, ProposeApproved, ProposeRejected, dll.)
- Buat `EmailTemplateService` untuk render template dengan placeholder replacement
- Modifikasi `EmailSender` untuk lookup template dari database
- Buat `EmailTemplateController` di Area Admin dengan CRUD dan preview
- Buat View editor template email dengan rich text editor (TinyMCE/CKEditor)

---

### [8.2] Preferensi Notifikasi per Pengguna

**Status saat ini:** Entity `Notification` mengirim notifikasi ke user tanpa opsi apapun bagi user untuk menentukan jenis notifikasi apa yang ingin diterima atau channel preferensi (in-app vs email).

**Fitur baru:**
- Halaman pengaturan notifikasi per karyawan
- Toggle on/off per jenis event (request updates, reservation reminders, restock alerts, propose updates)
- Pilihan channel: in-app only, email only, atau keduanya
- Quiet hours: periode waktu dimana notifikasi email tidak dikirim (misal di luar jam kerja)

**Implementasi:**
- Buat entity `NotificationPreference` (Id, UserId, EventType, InAppEnabled, EmailEnabled, QuietHoursStart, QuietHoursEnd)
- Modifikasi `NotificationRepository` untuk cek preferensi sebelum create notification
- Modifikasi `EmailSender` untuk cek preferensi dan quiet hours sebelum kirim
- Buat `NotificationPreferenceController` di Area Employee (dan Admin untuk default settings)
- Tambah halaman pengaturan notifikasi di profil user
- Modifikasi `UserProfileService` untuk include notification preferences

---

### [8.3] Reminder Otomatis Jatuh Tempo Peminjaman

**Status saat ini:** Sistem tidak memiliki mekanisme reminder proaktif sebelum jatuh tempo peminjaman. Karyawan hanya mengetahui statusnya berubah menjadi `Late` setelah sudah terlambat. `RemainingDays` di `ActiveRequest` dihitung tetapi tidak digunakan untuk trigger reminder.

**Fitur baru:**
- Reminder otomatis H-3, H-1, dan H+0 (hari jatuh tempo) melalui notifikasi in-app dan email
- Reminder eskalasi ke admin jika H+1 dan belum dikembalikan
- Konfigurasi jadwal reminder oleh admin (berapa hari sebelum jatuh tempo)
- Ringkasan harian untuk admin: daftar item yang jatuh tempo hari ini dan yang overdue

**Implementasi:**
- Buat entity `ReminderConfig` (ConfigId, DaysBeforeDue, SendToUser, SendToAdmin, IsActive)
- Buat `ReminderSchedulerService` sebagai IHostedService (background job harian)
- Modifikasi `NotificationRepository` untuk create reminder notifications
- Buat `ReminderEmailDTO` dengan detail item, tanggal jatuh tempo, dan remaining days
- Modifikasi `EmailSender` untuk support reminder email template
- Tambah konfigurasi reminder di halaman admin settings
- Buat View ringkasan harian overdue untuk admin dashboard

---

## 9. Manajemen Pengguna & Keamanan Lanjutan

### [9.1] Manajemen Departemen & Hierarki Organisasi

**Status saat ini:** Entity `User` extends `IdentityUser` dengan `EmployeeCode`, `FirstName`, `LastName` tanpa informasi departemen. Tidak ada fitur untuk mengelompokkan karyawan berdasarkan departemen atau unit organisasi, sehingga analitik dan kontrol akses tidak bisa dilakukan per departemen.

**Fitur baru:**
- Master data departemen dan hierarki organisasi
- Assign setiap user ke departemen
- Filter dan reporting per departemen
- Kontrol akses item: item tertentu hanya bisa dipinjam oleh departemen tertentu
- Dashboard admin per departemen

**Implementasi:**
- Buat entity `Department` (DepartmentId, DepartmentName, ParentDepartmentId, ManagerUserId, IsActive)
- Tambah field `DepartmentId` pada entity `User`
- Buat entity `ItemDepartmentRestriction` (ItemId, DepartmentId) untuk access control per item
- Modifikasi `UserViewService` untuk include department info
- Modifikasi `ItemViewEmployeeService` untuk filter item berdasarkan department restriction
- Buat `DepartmentController` di Area Admin
- Buat View CRUD departemen dan assign user ke departemen

---

### [9.2] Audit Log yang Diperkaya dengan Detail Perubahan

**Status saat ini:** Entity `AuditLog` hanya memiliki field `Action` (string), `Role`, `Name`, dan `TimeStamp`. Tidak ada detail tentang data apa yang berubah (before/after values), entity apa yang terdampak, atau IP address pelaku.

**Fitur baru:**
- Pencatatan detail perubahan: entity yang berubah, field yang berubah, nilai sebelum dan sesudah
- Pencatatan metadata: IP address, User-Agent, session info
- Filter audit log berdasarkan entity type, user, action type, dan rentang waktu
- Export audit log untuk keperluan compliance

**Implementasi:**
- Tambah field pada entity `AuditLog`: `EntityType`, `EntityId`, `OldValues` (JSON), `NewValues` (JSON), `IpAddress`, `UserAgent`
- Buat `AuditInterceptor` sebagai EF Core SaveChanges interceptor untuk otomatis capture perubahan
- Modifikasi `AuditLogRepository` untuk support query filter lanjutan
- Buat `AuditLogDetailDTO` untuk response yang diperkaya
- Modifikasi `AuditLogController` di kedua Area untuk support filter
- Buat View audit log dengan filter panel dan detail modal (before/after comparison)

---

### [9.3] Two-Factor Authentication & Kebijakan Password

**Status saat ini:** Sistem menggunakan ASP.NET Identity standar untuk autentikasi. Tidak ada implementasi Two-Factor Authentication (2FA) atau kebijakan password yang ketat (minimum length, complexity, expiration).

**Fitur baru:**
- Aktivasi 2FA via email OTP atau authenticator app (TOTP)
- Kebijakan password: minimum length, harus ada uppercase/lowercase/angka/simbol
- Password expiration policy: force change password setiap N hari
- Account lockout setelah N kali percobaan gagal
- Riwayat login per user

**Implementasi:**
- Aktifkan konfigurasi 2FA bawaan ASP.NET Identity di `Program.cs`
- Konfigurasi `IdentityOptions` untuk password policy di `Program.cs`
- Buat entity `LoginHistory` (HistoryId, UserId, LoginTime, IpAddress, UserAgent, IsSuccessful)
- Buat `SecurityPolicyService` untuk manage password expiration check
- Modifikasi halaman login/register di Area Identity untuk 2FA flow
- Buat View pengaturan keamanan di profil user
- Tambah middleware untuk cek password expiration pada setiap request

---

## 10. Integrasi & Interoperabilitas

### [10.1] RESTful API Layer untuk Integrasi Eksternal

**Status saat ini:** Sistem adalah MVC web application dengan controller yang return View. Tidak ada API layer terpisah yang memungkinkan integrasi dengan aplikasi eksternal (mobile app, ERP, sistem lain).

**Fitur baru:**
- API controller terpisah dengan endpoint RESTful untuk semua operasi utama
- Autentikasi API via JWT Bearer Token
- API versioning untuk backward compatibility
- Rate limiting untuk mencegah penyalahgunaan
- Swagger/OpenAPI documentation

**Implementasi:**
- Buat folder `Controllers/Api/V1/` dengan API controllers: `ItemsApiController`, `RequestsApiController`, `ReservationsApiController`, `RestockApiController`
- Implementasi JWT authentication dengan `Microsoft.AspNetCore.Authentication.JwtBearer`
- Buat `ApiAuthController` untuk login dan generate token
- Konfigurasi API versioning via `Asp.Versioning.Mvc`
- Konfigurasi rate limiting via `AspNetCoreRateLimit`
- Konfigurasi Swagger via `Swashbuckle.AspNetCore` di `Program.cs`
- Buat `ApiResponseDTO<T>` sebagai standar response wrapper

---

### [10.2] Integrasi Google Calendar untuk Jadwal Peminjaman

**Status saat ini:** File `client_secret_841964895929-...apps.googleusercontent.com.json` menunjukkan adanya setup OAuth Google, namun belum ada integrasi aktif dengan Google Calendar. Jadwal peminjaman dan reservasi tidak tersinkronisasi dengan kalender personal karyawan.

**Fitur baru:**
- Sinkronisasi otomatis jadwal peminjaman dan reservasi ke Google Calendar karyawan
- Event calendar: tanggal pickup, tanggal return, reminder
- Update otomatis event calendar saat status berubah (approve, cancel, extend)
- Link kembali ke detail request/reservasi dari event calendar

**Implementasi:**
- Integrasi `Google.Apis.Calendar.v3` NuGet package
- Buat `GoogleCalendarService` untuk CRUD calendar events
- Implementasi OAuth2 consent flow menggunakan client secret yang sudah ada
- Tambah field `GoogleCalendarEventId` pada entity `ActiveRequest` dan `Reservation`
- Modifikasi `ActiveRequestService` dan `ReservationAdminService` untuk sync event pada setiap status change
- Buat halaman pengaturan koneksi Google Account di profil user
- Handle token refresh dan error recovery

---

### [10.3] Webhook & Event Notification untuk Sistem Eksternal

**Status saat ini:** Tidak ada mekanisme webhook untuk memberi tahu sistem eksternal tentang event yang terjadi di sistem inventaris (misal stok habis, item baru ditambahkan, request disetujui).

**Fitur baru:**
- Konfigurasi webhook URL per event type
- Pengiriman HTTP POST dengan payload JSON ke webhook URL saat event terjadi
- Retry mechanism dengan exponential backoff untuk webhook yang gagal
- Log pengiriman webhook untuk monitoring dan debugging
- Test webhook endpoint dari halaman konfigurasi

**Implementasi:**
- Buat entity `WebhookConfig` (ConfigId, EventType, Url, SecretKey, IsActive, CreatedAt)
- Buat entity `WebhookDeliveryLog` (LogId, ConfigId, EventType, Payload, ResponseCode, SentAt, RetryCount)
- Buat `WebhookService` untuk dispatch event ke registered webhooks
- Buat `WebhookDispatcherBackgroundService` sebagai IHostedService untuk async delivery dengan retry
- Hook `WebhookService` ke service-service utama (Request, Reservation, Restock, ItemPropose)
- Buat `WebhookController` di Area Admin untuk CRUD konfigurasi dan test
- Buat View halaman konfigurasi webhook dan delivery log

---

## 11. Mobile & Aksesibilitas

### [11.1] Progressive Web App (PWA) Support

**Status saat ini:** Aplikasi adalah web MVC standar tanpa kemampuan offline atau fitur PWA. Karyawan di lapangan atau gudang mungkin mengalami koneksi internet yang tidak stabil saat perlu mengakses data inventaris.

**Fitur baru:**
- Service Worker untuk caching halaman dan data penting
- Manifest file untuk install sebagai PWA di perangkat mobile
- Offline mode: view katalog item, riwayat peminjaman, dan status request tanpa koneksi
- Background sync: request yang dibuat offline otomatis terkirim saat koneksi kembali
- Push notification native via Web Push API

**Implementasi:**
- Buat file `manifest.json` di `wwwroot` dengan konfigurasi PWA
- Buat `service-worker.js` dengan caching strategy (Cache First untuk aset statis, Network First untuk API)
- Implementasi IndexedDB untuk penyimpanan data offline di browser
- Buat `OfflineSyncService` JavaScript untuk queue dan sync offline requests
- Integrasi Web Push API dengan `WebPush` NuGet package
- Tambah meta tag PWA di layout View
- Buat `PushSubscriptionController` untuk manage device subscriptions

---

### [11.2] Responsive Design & Mobile-First UI Enhancement

**Status saat ini:** Halaman-halaman View di `wwwroot` dan `Views` ada tetapi belum terverifikasi untuk pengalaman mobile yang optimal. Tabel data, form input, dan navigasi mungkin tidak responsive untuk layar kecil.

**Fitur baru:**
- Redesign layout utama untuk mobile-first approach
- Navigasi bottom tab bar untuk mobile
- Swipe gesture untuk approve/reject request (mobile)
- Touch-friendly UI elements: ukuran tombol, spacing, dan input yang optimal untuk touch
- Quick action floating button untuk operasi sering (request item, scan barcode)

**Implementasi:**
- Modifikasi CSS/Sass di `wwwroot` dengan breakpoints mobile-first
- Modifikasi `_Layout.cshtml` untuk responsive navigation (sidebar → bottom tab pada mobile)
- Buat `_MobileLayout.cshtml` sebagai alternatif layout untuk mobile detection
- Implementasi Hammer.js atau CSS touch gestures untuk swipe actions
- Modifikasi View tables dengan responsive table pattern (card view pada mobile)
- Audit dan modifikasi semua `ViewComponents` untuk responsive rendering

---

## 12. Konfigurasi & Pengaturan Sistem

### [12.1] Halaman Pengaturan Sistem Terpusat (System Settings)

**Status saat ini:** `ProposeLimit` adalah satu-satunya configurable setting yang tersimpan di database. Pengaturan lain (connection string, email config) ada di `appsettings.json` dan tidak bisa diubah tanpa restart aplikasi atau akses ke server.

**Fitur baru:**
- Halaman admin settings terpusat untuk konfigurasi sistem yang bisa diubah runtime
- Kategori settings: General, Borrow Policy, Notification, Security, Integration
- Contoh settings: max borrow days, max quantity per request, auto-approve threshold, email SMTP config
- Riwayat perubahan settings untuk audit
- Reset ke default values

**Implementasi:**
- Buat entity `SystemSetting` (SettingId, Key, Value, DataType, Category, Description, DefaultValue, LastModifiedBy, LastModifiedAt)
- Buat `SystemSettingService` dengan caching (IMemoryCache) untuk akses cepat
- Modifikasi service-service yang saat ini hardcode policy values untuk baca dari `SystemSettingService`
- Buat `SystemSettingController` di Area Admin
- Buat View halaman settings dengan grouping per kategori dan input sesuai data type
- Buat `SystemSettingChangeLog` entity untuk audit perubahan setting

---

### [12.2] Manajemen Hari Libur & Kalender Kerja

**Status saat ini:** Kalkulasi `RequestDays` dan `RemainingDays` di sistem tidak mempertimbangkan hari libur atau hari non-kerja. Semua hari dihitung sama, sehingga perhitungan durasi peminjaman dan denda bisa tidak akurat.

**Fitur baru:**
- Kalender hari libur nasional dan hari libur perusahaan
- Opsi hitung `RequestDays` hanya hari kerja (exclude weekend dan hari libur)
- Konfigurasi hari kerja perusahaan (misal Senin-Jumat atau Senin-Sabtu)
- Import hari libur dari sumber eksternal

**Implementasi:**
- Buat entity `Holiday` (HolidayId, Date, Name, IsRecurring, Type)
- Buat entity `WorkingDayConfig` (ConfigId, DayOfWeek, IsWorkingDay)
- Buat `WorkingDayService` untuk kalkulasi hari kerja
- Modifikasi `ActiveRequestService` dan `ReservationAdminService` untuk gunakan `WorkingDayService` saat hitung remaining days
- Buat `HolidayController` di Area Admin untuk CRUD hari libur
- Buat View kalender hari libur dan konfigurasi hari kerja

---

### [12.3] Backup & Restore Ekspor Data Terjadwal

**Status saat ini:** Sistem belum memiliki mekanisme backup atau ekspor data berkala yang diatur dari dalam aplikasi. Keamanan data bergantung sepenuhnya pada manajemen database eksternal.

**Fitur baru:**
- Penjadwalan backup/ekspor data master inventaris secara otomatis (harian/mingguan)
- Penyimpanan file hasil ekspor ke lokasi yang aman atau cloud storage
- Retensi otomatis (menghapus file backup lama)
- Notifikasi email bila proses backup otomatis gagal

**Implementasi:**
- Buat entity `BackupConfig` (ConfigId, Frequency, StoragePath, RetentionDays, IsActive)
- Buat entity `BackupLog` (LogId, BackupDate, FileSize, Status, ErrorMessage)
- Buat `DataBackupService` sebagai background job (IHostedService)
- Integrasikan dengan storage provider via `IStorageService`
- Buat `BackupController` di Area Admin untuk konfigurasi dan manual trigger
- Buat View untuk konfigurasi jadwal ekspor dan riwayat ekspor data

---

## 13. Integrasi Perangkat Keras & IoT (Hardware & IoT Integration)

### [13.1] Pelacakan Aset Menggunakan RFID

**Status saat ini:** Sistem belum memiliki integrasi ke perangkat keras pemindai otomatis. Identifikasi item bergantung pada admin yang memasukkan data secara manual ke dalam sistem.

**Fitur baru:**
- Modul integrasi dengan RFID reader (fixed reader di pintu gudang/ruangan atau handheld scanner)
- Pembaruan status peminjaman dan pengembalian secara otomatis saat aset dengan tag RFID melewati gantry/reader
- Audit fisik instan (stock opname) dengan memindai seluruh ruangan menggunakan handheld RFID reader
- Alert sekuriti jika item yang belum di-approve (status belum dipinjam) dibawa keluar dari zona penyimpanan

**Implementasi:**
- Buat entity `RfidTag` (TagId, ItemId, EpcCode, RegisteredAt)
- Buat entity `RfidReaderEvent` (EventId, ReaderIp, EpcCode, Timestamp, Direction)
- Buat `RfidIntegrationService` yang menerima data dari RFID reader (melalui protokol MQTT atau webhook HTTP POST)
- Modifikasi `ActiveRequestService` untuk otomatis mengubah status menjadi "Dipinjam" atau "Dikembalikan" berdasarkan *reader event*
- Tambah background job untuk memantau "unauthorized item movement" dan trigger alert

---

### [13.2] Integrasi Loker Pengambilan Mandiri (Smart Locker)

**Status saat ini:** Proses pengambilan (pickup) dan pengembalian (return) item memerlukan interaksi fisik dengan admin atau staf gudang. Karyawan tidak bisa mengambil barang di luar jam kerja admin.

**Fitur baru:**
- API integrasi dengan sistem loker pintar (smart locker) ber-PIN atau berbasis QR code
- Saat admin menyetujui request, sistem menempatkan item di smart locker dan mengirim PIN satu kali pakai (OTP) ke karyawan
- Karyawan mengambil item dari loker kapan saja menggunakan PIN/QR Code
- Karyawan juga dapat mengembalikan item ke loker pintar tanpa kehadiran admin

**Implementasi:**
- Buat entity `SmartLockerSystem` (LockerId, LocationId, TotalCompartments, IpAddress, ApiEndpoint)
- Buat entity `LockerCompartment` (CompartmentId, LockerId, Name, CurrentItemId, Status)
- Buat `SmartLockerIntegrationService` untuk berkomunikasi dengan API vendor loker
- Tambah field `PickupPinCode` dan `ReturnPinCode` pada entity `ActiveRequest`
- Buat endpoint `POST /api/webhooks/locker-events` untuk menerima konfirmasi pintu loker dibuka/ditutup dari hardware

---

## 14. Kepatuhan & Asuransi Aset (Asset Compliance & Insurance)

### [14.1] Manajemen Polis Asuransi Aset

**Status saat ini:** Tidak ada pelacakan asuransi untuk aset bernilai tinggi (seperti kendaraan operasional, laptop high-end, atau server). Jika aset hilang atau rusak, proses klaim tidak terintegrasi dengan data inventaris.

**Fitur baru:**
- Pencatatan polis asuransi per aset atau kategori aset
- Pengingat otomatis perpanjangan polis asuransi yang akan kedaluwarsa
- Pencatatan nilai pertanggungan asuransi dan premi
- Integrasi riwayat laporan kerusakan (damage report) dengan log klaim asuransi

**Implementasi:**
- Buat entity `AssetInsurance` (InsuranceId, ItemId, ProviderName, PolicyNumber, CoverageAmount, StartDate, EndDate, PremiumCost)
- Buat entity `InsuranceClaim` (ClaimId, InsuranceId, DamageReportId, ClaimDate, ClaimAmount, Status)
- Buat enum `ClaimStatusEnum` (Submitted, UnderReview, Approved, Rejected, PayoutReceived)
- Buat `InsuranceManagementService` di Area Admin
- Modifikasi background job reminder untuk mengecek `EndDate` polis asuransi (pengingat H-30)
- Buat View CRUD asuransi dan form pengajuan klaim

---

### [14.2] Pencatatan Pajak & Nilai Buku Aset (Tax Compliance)

**Status saat ini:** Meskipun nilai depresiasi dicatat secara sederhana (jika fitur 1.3 diterapkan), perhitungan teknis untuk kepatuhan pajak perusahaan belum tersedia.

**Fitur baru:**
- Pengelompokan aset berdasarkan golongan pajak aset tetap (sesuai peraturan perpajakan yang berlaku)
- Laporan nilai buku fiskal vs komersial pada akhir tahun
- Pencetakan laporan daftar penyusutan harta berwujud untuk lampiran SPT Tahunan

**Implementasi:**
- Buat entity `TaxAssetCategory` (CategoryId, Name, DepreciationRate, UsefulLifeYearsFiskal)
- Relasikan `TaxAssetCategory` dengan entitas master `Category` atau secara spesifik per `Item`
- Modifikasi `AssetDepreciationService` untuk membedakan kalkulasi *Commercial Depreciation* dan *Fiscal Depreciation*
- Buat `TaxReportDTO` dan tambahkan fungsionalitas di `ReportBuilderService` untuk format PDF lampiran SPT
- Tambah View khusus laporan nilai buku fiskal di module reporting Admin

---

## 15. Keamanan Fisik & Logikal (Physical & Logical Security)

### [15.1] Tanda Tangan Digital (E-Signature) untuk Serah Terima

**Status saat ini:** Proses serah terima item (pickup dan return) hanya berupa klik tombol "Approve" atau "Receive" oleh admin. Tidak ada bukti fisik atau otentikasi biometrik/kriptografik bahwa karyawan yang bersangkutan benar-benar menerima atau mengembalikan barang tersebut.

**Fitur baru:**
- Modul *electronic signature* saat serah terima barang secara langsung
- Penempatan *signature pad* digital di dashboard admin saat karyawan mengambil atau mengembalikan item
- Otomatisasi pembuatan Berita Acara Serah Terima (BAST) dalam format PDF yang ditandatangani secara digital (mengandung *hash/timestamp*)
- Penyimpanan bukti otentikasi (IP, *browser fingerprint*, atau *geolocation*) pada saat tanda tangan dilakukan

**Implementasi:**
- Integrasi library JavaScript `signature_pad` pada View `ActiveRequest` untuk menangkap input *stylus* / *touchscreen*
- Buat entity `HandoverDocument` (DocumentId, RequestId, DocumentType, SignatureDataUrl, GeneratedPdfPath, GeneratedAt)
- Buat enum `HandoverTypeEnum` (Pickup, Return)
- Modifikasi `ActiveRequestService` untuk memvalidasi *signature data* sebelum memproses pergantian status menjadi `Dipinjam` atau `Dikembalikan`
- Buat endpoint untuk menghasilkan dan mengunduh PDF BAST menggunakan `QuestPDF`

---

### [15.2] Gate Pass & Surat Jalan Eksternal

**Status saat ini:** Admin tidak dapat menerbitkan izin resmi bila karyawan perlu membawa aset keluar dari premis perusahaan (misalnya membawa laptop dan proyektor untuk presentasi di luar kota).

**Fitur baru:**
- Penerbitan *Gate Pass* (Surat Jalan/Izin Keluar Gedung) yang dilengkapi dengan QR Code otentikasi
- Satpam gedung (Security) dapat melakukan *scan* QR code pada Gate Pass untuk memverifikasi bahwa item yang dibawa keluar berstatus "Approved" untuk kegiatan eksternal
- Pelacakan waktu *check-out* dan *check-in* dari gedung

**Implementasi:**
- Buat entity `GatePass` (PassId, ActiveRequestId, Destination, TargetReturnDate, Status, QrCodePath, SecurityCheckOut, SecurityCheckIn)
- Buat enum `GatePassStatusEnum` (Issued, ScannedOut, ScannedIn, Invalidated)
- Tambah Role baru: `SecurityGuard`
- Buat Area atau View khusus `SecurityController` yang berisi pemindai QR untuk memvalidasi *Gate Pass*
- Modifikasi `ActiveRequestService` untuk men-generate otomatis dokumen *Gate Pass* bila karyawan mengindikasikan tujuan peminjaman adalah "Eksternal"

---

## 16. Manajemen Multi-Lokasi Ekstensif (Enterprise Resource Structuring)

### [16.1] Inter-Branch Transfer (Transfer Aset Antar Cabang)

**Status saat ini:** Hanya ada entitas `City` untuk menandai lokasi spesifik. Tidak ada proses terstruktur untuk memindahkan stok atau aset secara permanen dari satu kota (cabang) ke kota lain.

**Fitur baru:**
- Alur persetujuan perpindahan antar cabang (Pihak Cabang A mengirim *Transfer Request* -> Pihak Cabang B menyetujui penerimaan)
- *In-Transit Status* untuk melacak aset yang sedang berada dalam perjalanan logistik
- Pencatatan biaya ekspedisi/kurir

**Implementasi:**
- Buat entity `InterBranchTransfer` (TransferId, ItemId, SourceCityId, DestinationCityId, InitiatedBy, ApprovedBy, DispatchDate, ReceivedDate, CourierName, TrackingNumber, Status)
- Buat enum `TransferStatusEnum` (Draft, PendingApproval, InTransit, Received, Cancelled)
- Buat `InterBranchService` dan `InterBranchController` di Area Admin
- Modifikasi `Item.Availability` dan ganti filter lokasi sehingga item hilang sementara dari katalog reguler selama statusnya `InTransit`

---

### [16.2] Hierarki Kategori Tanpa Batas (Infinite Category Tree)

**Status saat ini:** Sistem hanya mendukung dua level kategori hierarki (`Category` dan `SubCategory`). Hal ini membatasi skalabilitas bila inventaris perusahaan mencapai ratusan ribu jenis barang bervariasi.

**Fitur baru:**
- Struktur hierarki tree (`ParentId`) untuk kategori, memungkinkan N-level sub-kategori (misal: IT -> Hardware -> Computer -> Laptop -> Ultrabook)
- Sistem kategori yang lebih terpusat tanpa harus membuat entity `SubCategory`, `SubSubCategory` yang terpisah
- *Breadcrumb navigation* yang dinamis di katalog berdasarkan path kategori item

**Implementasi:**
- Restrukturisasi drastis Entity `Category`: (CategoryId, Name, ParentCategoryId, Description, IconPath)
- Hapus entity `SubCategory` dan migrasikan relasinya langsung ke `ParentCategoryId`
- Modifikasi `CategoryService` di seluruh aplikasi agar menggunakan *Recursive CTE (Common Table Expressions)* saat *query* database untuk memuat seluruh cabangnya
- Buat komponen View `CategoryTreeViewCore` untuk merender hierarki tanpa batas di menu sidebar karyawan

---

## 17. Pengelolaan Barang Habis Pakai (Consumable Inventory & Kitting)

### [17.1] Kitting & Bundling Item (Perakitan Paket Inventaris)

**Status saat ini:** Item bersifat murni independen. Tidak ada sistem untuk merakit beberapa *Item* menjadi satu *Bundle* peminjaman. Contoh kasus: "Paket Presentasi" yang terdiri dari 1 Proyektor, 1 Layar, 1 Pointer, dan 3 Kabel HDMI.

**Fitur baru:**
- Admin dapat membuat "Kit" atau "Bundle" yang berisi berbagai item (borrowable & consumable)
- Karyawan hanya mengklik "Pinjam Kit Presentasi", sistem secara otomatis mem-booking seluruh sub-item yang ada di dalam kit tersebut secara serentak
- Logika validasi *Stock-out*: Jika salah satu kabel HDMI hilang/habis, maka sistem melarang peminjaman seluruh Kit tersebut atau menggantinya dengan kabel HDMI subsitusi

**Implementasi:**
- Buat entity `ItemBundle` (BundleId, Name, Description, IsActive)
- Buat entity `BundleComponent` (ComponentId, BundleId, ItemId, RequiredQuantity)
- Modifikasi logic peminjaman (`RequestEmployeeService`) agar saat karyawan menambah sebuah `ItemBundle`, layanan secara loop membuat rincian `Request` untuk seluruh komponen terkait
- Buat `BundleController` di Area Admin untuk meracik paket

---

### [17.2] BOM (Bill of Materials) untuk Modifikasi Aset

**Status saat ini:** Tidak ada pelacakan untuk aset yang dimodifikasi. (Misal RAM ditambahkan ke laptop, stok RAM terpotong, laptop menjadi spesifikasi baru).

**Fitur baru:**
- Menautkan *Consumable* yang melekat/diinstal pada aset tetap (*Borrowable*)
- Menambahkan riwayat "Upgrade Aset" (contoh: Pemasangan SSD 1TB pada Laptop Dell XPS)
- Penghitungan nilai buku yang otomatis disesuaikan (Nilai Laptop + Nilai SSD)

**Implementasi:**
- Buat entity `AssetDependency/AssetModification` (ModificationId, ParentItemId, ConsumableItemId, QuantityUsed, InstallationDate, InstalledBy)
- Saat `Modifikasi` di-approve, deduct stok `ConsumableItemId`, dan catat sejarah di log Parent Item
- Modifikasi model DTO `ItemView` untuk menampilkan tabel "Komponen Terpasang" bagi aset berjenis elektronik/hardware

---

## 18. Analitik Tingkat Lanjut & Machine Learning (Advanced Analytics & ML)

### [18.1] Prediksi Kebutuhan Restock Berbasis AI (Demand Forecasting)

**Status saat ini:** Sistem hanya memiliki *QuantityLow* statis sebagai batas *restock*. Tidak ada kemampuan prediktif untuk memproyeksikan kapan barang akan habis berdasarkan tren historis peminjaman.

**Fitur baru:**
- Algoritma pemodelan deret waktu (Time Series Analysis / ARIMA / Prophet) untuk memprediksi lonjakan permintaan barang di bulan-bulan tertentu (misalnya, laptop dan mouse sering dipinjam/dikonsumsi di bulan rilis proyek besar)
- Rekomendasi kuantitas pembelian (Reorder Quantity) di masa depan agar tidak terjadi *overstock* atau *stockout*
- *Lead time forecasting* (prediksi keterlambatan vendor berdasarkan data PO historis)

**Implementasi:**
- Integrasi Library / Eksternal Service (contoh: ML.NET, atau Python microservice via API) untuk memproses histori `ActiveRequest` dan `Restock`
- Buat entity `PredictionLogs` untuk menyimpan riwayat hasil kalkulasi model AI 
- Modifikasi `RestockService` untuk menampilkan UI "Rekomendasi Restock AI" ke Admin saat membuat *Draft* PO baru

---

### [18.2] Deteksi Anomali Pemakaian (Fraud/Abuse Detection)

**Status saat ini:** Tidak ada pemantauan otomatis pada perilaku aneh (misal: satu karyawan meminjam 5 laptop sekaligus, atau barang *consumable* selalu habis dua kali lebih cepat dari departemen lain). 

**Fitur baru:**
- Modul pendeteksi anomali pada peminjaman dan konsumsi barang berdasarkan *baseline* perilaku departemen/pengguna
- Pemberitahuan otomatis (Alert/Flagging) ke Admin/HR jika ada indikasi *hoarding* (penimbunan aset) atau *misuse*
- *Auto-rejection* untuk request yang melebihi batas "Normalitas" yang dinamis (bukan sekedar `ProposeLimit` statis)

**Implementasi:**
- Pembuatan filter deteksi anomali (berdasarkan *standard deviation* historikal) yang dijalankan oleh Background Service / `IHostedService`
- Modifikasi `RequestAdminService` dengan menambahkan visual *flag* "Potensi Anomali" pada baris antrean approval
- Penyusunan `AnomalyReportDTO` untuk dikirimkan melalui notifikasi ke auditor / manajemen tingkat atas

---

## 19. Migrasi & Pengelolaan Multi-Tenant (SaaS Readiness)

### [19.1] Arsitektur Multi-Tenant (Tenant Isolation)

**Status saat ini:** Sistem dirancang untuk digunakan oleh satu entitas bisnis tunggal (Single Tenant). Seluruh database hanya melayani satu perusahaan tanpa isolasi data akun perusahaan lain.

**Fitur baru:**
- Kemampuan sistem bertindak sebagai SaaS (Software as a Service) 
- Pendaftaran akun Perusahaan baru secara mandiri 
- Perusahaan A tidak bisa melihat Asset, Pegawai, atau transaksi Perusahaan B
- URL kustom atau subdomain (*companyA.inventory.com*) 

**Implementasi:**
- Tambah field `TenantId` pada setiap entitas dasar (`User`, `Item`, `Category`, `City`, `Request`, `Reservation`, dll.)
- Modifikasi *EF Core DbContext* dengan Global Query Filter `HasQueryFilter(e => e.TenantId == _currentTenantId)` supaya semua *query* otomatis dibatasi dengan tenant per sesi
- Buat `TenantIdentificationMiddleware` untuk mendeteksi `TenantId` berdasarkan token autentikasi atau domain Header
- Buat `TenantController` master (Super Admin) untuk manajemen langganan (Subscription Mgt) per perusahaan

---

## 20. Ekosistem Pihak Ketiga & Otomatisasi (Third-Party Ecosystem)

### [20.1] Integrasi ERP (Enterprise Resource Planning) (SAP / Oracle / Odoo)

**Status saat ini:** Data inventaris dan aset tidak memengaruhi buku besar perusahaan secara otomatis. 

**Fitur baru:**
- API dua arah antara sistem inventaris dengan layanan ERP Pusat (SAP, Odoo, Dynamics)
- Ketika *Purchase Order* dibuat dalam sistem, otomatis menjadi *Draft* AP (Account Payable) di ERP
- Sinkronisasi master data pegawai langsung dari *Human Resource Management System* (HRMS) pusat

**Implementasi:**
- Pembuatan modul *Event Bus* terdistribusi (contoh: RabbitMQ / Kafka)
- Buat layanan pekerja (*Worker Services*) independen untuk memproses *message queue* integrasi 
- Penyusunan standard payload *JSON* yang bisa di-mapping untuk modul *Finance* di *third party software*

---

### [20.2] Single Sign-On (SSO) Enterprise

**Status saat ini:** Autentikasi sistem hanya menggunakan ASP.NET Core Identity lokal. Karyawan harus mengingat *username/password* khusus untuk portal inventaris yang terpisah dari akun kerja utama mereka.

**Fitur baru:**
- Modul *Single Sign-On* menggunakan akun Microsoft Entra ID (sebelumnya Azure AD), Google Workspace, atau Okta
- *Auto-provisioning* dan de-aktivasi akun (saat karyawan keluar dari perusahaan, akses inventaris otomatis terputus)

**Implementasi:**
- Registrasi aplikasi di portal penyedia identitas (OAuth2 / OpenID Connect)
- Implementasi `Microsoft.AspNetCore.Authentication.OpenIdConnect` pada `Program.cs`
- Sinkronisasi otomatis *Roles* (contoh: AD Group `InventoryAdmins` di-map secara dinamis ke Role `Admin` di aplikasi internal)

---

## 21. Pengelolaan Kontrak & Dokumen Aset Pihak Ketiga (Contract Management)

### [21.1] Service Level Agreement (SLA) & Vendor Management

**Status saat ini:** Sistem tidak melacak kontrak layanan teknis dari vendor luar. Jika aset IT rusak, admin harus mencari *hardcopy* surat garansi di lemari.

**Fitur baru:**
- Penyimpanan detail kontrak perawatan (SLA) dengan pihak ketiga
- Menautkan aset ke dokumen kontrak tertentu yang masih berlaku
- Peringatan terminasi kontrak atau pembaruan SLA sebelum aset *End-of-Life*

**Implementasi:**
- Buat entity `VendorContract` (ContractId, VendorId, ContractNumber, SlaDetails, StartDate, EndDate, DocumentPath, Status)
- Relasi antara `Item` dan `VendorContract` (misal 50 unit laptop ditautkan pada 1 kontrak Service Dell ProSupport)
- Modifikasi *Dashboard Admin* dengan menambahkan *Widget* "Kontrak Aktif vs Kedaluwarsa"

---

## 22. Manajemen Lingkungan & Keberlanjutan (Eco-Sustainability)

### [22.1] E-Waste Tracking & Carbon Footprint

**Status saat ini:** Sistem tidak memiliki pelaporan atas limbah elektronik dari pembuangan aset dan tingkat emisi/pemakaian energi barang (misal: AC Portabel, Server).

**Fitur baru:**
- Perhitungan estimasi konsumsi energi aset IT dan elektronik
- Log terpusat untuk mendokumentasikan pembuangan limbah elektronik (*E-Waste*) ke lembaga *recycle* resmi
- *Carbon Footprint Report* perusahaan terkait investasi perangkat keras

**Implementasi:**
- Tambah atribut teknis tambahan `EstimatedWattage` dan `EcoRating` (seperti *Energy Star*) pada `Category` atau `Item`
- Buat entitas `EwasteDisposal` sebagai ekstensi dari Alur `DisposalRequest` untuk memisahkan sumbangan meja kayu vs pemusnahan baterai Lithium
- Buat analitik di dashboard untuk menampilkan Total Jejak Karbon dari seluruh aset operasional aktif

---

## 23. Gamifikasi & Motivasi Karyawan (User Gamification)

### [23.1] Poin Reward Berdasarkan Perilaku Peminjaman yang Baik

**Status saat ini:** Karyawan tidak mendapatkan insentif apapun untuk merawat atau mengembalikan perangkat tepat waktu, sementara `Late` / Overdue hanya berbuah teguran.

**Fitur baru:**
- Implementasi sistem poin (contoh: *AssetCare Points*) 
- Karyawan mendapat +10 poin untuk setiap pengembalian tepat waktu tanpa kerusakan. Karyawan dikurangi -20 poin jika terlambat.
- Papan peringkat (*Leaderboard*) bulanan departemen dengan persentase peminjaman tersukses
- Integrasi ke *HR System* di mana poin ini bisa ditukar dengan *merchandise* kantor kecil (misal: tumbler baru)

**Implementasi:**
- Buat entitas `UserRewardPoint` (PointId, UserId, Points, Reason, LogDate)
- Modifikasi `ActiveRequestService` untuk otomatis menambah/mengurangi poin saat status berubah menjadi `WellReturned` / `Late` / `Damaged`
- Buat View `Leaderboard` pada dashboard karyawan

---

## 24. Peminjaman Antar Pengguna (Peer-to-Peer Borrowing)

### [24.1] Serah Terima Sementara Antar Karyawan

**Status saat ini:** Jika Karyawan A dan Karyawan B sama-sama berada di lapangan, Karyawan A yang telah selesai menggunakan proyektor tidak bisa langsung menyerahkannya ke Karyawan B. Karyawan A harus kembali ke kantor, menyerahkan ke Admin, lalu Admin meminjamkan ke Karyawan B.

**Fitur baru:**
- *Peer-to-Peer Transfer* untuk item yang sedang dipinjam
- Karyawan A mengajukan "Transfer to B" dari aplikasinya.
- Karyawan B menerima notifikasi dan memverifikasi kondisi barang via aplikasi (termasuk *electronic signature* P2P).
- Kepemilikan (Liability) atas `ActiveRequest` tersebut langsung berpindah dari A ke B secara digital, tanpa campur tangan Admin.

**Implementasi:**
- Buat entitas `PeerTransferRequest` (TransferId, ActiveRequestId, FromUserId, ToUserId, HandoverNotes, ToUserSignatureData, Status)
- Modifikasi model kepemilikan di `ActiveRequest.UserId` setelah transfer berstatus `Accepted`
- Tambah API endpoint untuk validasi pemindaian QR antar gawai karyawan (`FromUser` memunculkan QR Code -> `ToUser` men-scan QR)

---

## 25. Penjadwalan Pemeliharaan Preventif (Preventive Maintenance Scheduling)

### [25.1] Kalender Servis Rutin Otomatis

**Status saat ini:** Maintenance hanya terjadi secara reaktif melalui `DamageReport`. Tidak ada penjadwalan pemeliharaan preventif (misalnya: perawatan AC setiap 6 bulan, kalibrasi alat ukur tiap tahun, tune-up kendaraan).

**Fitur baru:**
- Pembuatan jadwal pemeliharaan rutin (*Preventive Maintenance*) berdasarkan rentang waktu tempuh rata-rata atau interval bulan/tahun.
- Notifikasi ke Vendor atau Teknisi Internal (PIC) seminggu sebelum jadwal tiba.
- Sistem akan merubah status *Item* menjadi "Dalam Perawatan" sehingga tidak dapat direservasi selama jendela pemeliharaan.
- Log pemeliharaan rutin yang mencatat apa saja yang diservis, biaya suku cadang, dan pengunggah bukti *service sheet*.

**Implementasi:**
- Buat entitas `MaintenanceSchedule` (ScheduleId, ItemId, IntervalDays, LastServiceDate, NextServiceDate, AssignedVendorId, IsActive)
- Buat entitas `MaintenanceExecution` (ExecutionId, ScheduleId, ActualServiceDate, Cost, ReplacedParts, TechnicianName, ReceiptPath)
- Gunakan `Hangfire` atau `Quartz.NET` sebagai *cron job* harian yang mengecek selisih tanggal `NextServiceDate` dan hari ini.
- Modifikasi UI *Calendar* admin untuk menambahkan lapisan visual *Maintenance Blocks*.

---

## 26. Logistik & Dispatch Assets (Delivery Fleet Management)

### [26.1] Penugasan Kurir Internal & Bukti Pengiriman

**Status saat ini:** Jika karyawan meminta pengiriman aset berukuran besar atau perpindahan antar cabang (Inter-Branch Transfer), tidak ada sistem pencatatan armada/kurir pengantar.

**Fitur baru:**
- Manajemen armada angkut inventaris (Mobil Box Operasional, Kurir Internal).
- *Delivery Dashboard* untuk mengatur pengiriman beberapa aset ke cabang atau ke lokasi *site project* karyawan dalam satu rute.
- Modul aplikasi *driver/courier* untuk mengubah status *In-Transit* menjadi *Delivered* disertai dengan bukti foto di lokasi.

**Implementasi:**
- Buat entitas `FleetVehicle` (VehicleId, PlateNumber, DriverName, Capacity)
- Buat entitas `DeliveryManifest` (ManifestId, VehicleId, DispatchDate, RouteDescription, Status)
- Relasikan `DeliveryManifest` ke banyak ID `ActiveRequest` (yang berjenis antar barang) atau `InterBranchTransfer`.
- Modifikasi portal *Employee* dengan tipe role baru `Driver` yang hanya melihat antrean pengirimannya sendiri dan tombol *Upload E-POD* (Electronic Proof of Delivery).

---

## 27. Kalkulasi & Visualisasi Ruang (Space Capacity Management)

### [27.1] Analisis Kapasitas Gudang (Warehouse Volumetrics)

**Status saat ini:** Entitas lokasi (City, Building, Room) tidak memahami kapasitas fisik ruangan. Admin bisa saja memindahkan 100 lemari ke sebuah ruangan sempit di dalam sistem, padahal aktualnya tidak muat.

**Fitur baru:**
- Sistem pemetaan gudang berbasis *Volume* (Panjang x Lebar x Tinggi) per lokasi/rak.
- Penambahan dimensi volumetrik pada setiap master `Item` (seberapa besar tempat yang dihabiskan satu buah kotak printer).
- Peringatan (*Warning*) ke Admin apabila `StorageLocation` sudah mencapai batas maksimal 90% kapasitas.
- Visualisasi warna kapasitas (Heatmap: Merah jika padat, Hijau jika kosong) pada antarmuka *Smart Storage location*.

**Implementasi:**
- Tambah field `LengthCm`, `WidthCm`, `HeightCm`, dan `WeightKg` pada `Item`.
- Tambah field `MaxVolumeCc` dan `MaxWeightCapacityKg` pada `StorageLocation` / `StorageRack`.
- Buat `SpaceKalkulatorService` yang akan mendeteksi (Sigma Volume Seluruh Item) di dalam satu rak/ruangan setiap kali terjadi *Receive*, *Restock*, atau *Relocation*.
- Penambahan *UI Chart Heatmap* 2D di dashboard *Storage Location* admin.

---

## 28. Pengambilan Keputusan Tingkat Lanjut (Advanced Business Intelligence)

### [28.1] ROI & Total Cost of Ownership (TCO) Calculator

**Status saat ini:** Sistem hanya dapat memproses item keluar-masuk. Manajemen perusahaan tidak mengetahui mana merk/vendor perangkat yang paling menguntungkan (Cost-Effective).

**Fitur baru:**
- Modul perhitungan seberapa menguntungkan memiliki/menyewa suatu aset keras di kantor.
- Menggabungkan *Purchase Price*, *Maintenance Cost* historikal, *Depreciation Value*, *Consumables Used* (tinta printer/baterai), hingga biaya listrik menjadi metrik TCO.
- Pemberian predikat (Vendor A vs Vendor B) dalam hal siklus hidup komponen. Manajer pengadaan dapat memutuskan "Di masa depan, kita tidak akan beli Macbook, karena TCO Macbook lebih mahal 40% dari Thinkpad di departemen yang sama".

**Implementasi:**
- Buat entitas *Analytic Dashboard* baru murni untuk TCO per *Category* maupun grup tipe (*ModelName*).
- Kalkulasi Service akan merayapi log *MaintenanceExecution*, agregasi *ItemHistory* dan mengkomparasinya dengan entitas `TaxAssetCategory`. 

---

## 29. Interaksi Pegawai Berbasis AI (AI Chatbot & Virtual Assistant)

### [29.1] Asisten Inventaris Cerdas (Smart Procurement Bot)

**Status saat ini:** Karyawan harus melakukan pencarian manual (mengetik `query` di kolom *search*) untuk menemukan *item* atau *category* apa yang mereka butuhkan.

**Fitur baru:**
- Integrasi *Chatbot* Gen-AI bertenaga RAG (*Retrieval-Augmented Generation*) berbasis katalog inventaris spesifik perusahaan.
- Karyawan bisa bertanya dengan NLP (Natural Language): *"Saya besok dinas luar ke Surabaya selama seminggu untuk seminar. Pinjamkan saya alat-alat presentasi."*
- *Chatbot* secara cerdas menganalisis maksud karyawan, mengecek kalender `AvailabilitySlotDTO`, mencari di database untuk merekomendasikan: 1 Laptop Ringan, 1 Pointer, dan 1 Proyektor Portabel yang sedang 'Tersedia' dari Gudang Utama besok, lalu menaruhnya otomatis dalam `BorrowCart`.

**Implementasi:**
- Registrasi API NLP dari OpenAI atau Azure Cognitive Services. 
- Buat Vector DB / Embeddings sederhana untuk kolom deskripsi dari `Item` dan histori permintaan yang mirip.
- Integrasi UI di *Layout Employee* menggunakan balon ngobrol (chat bubble) dengan library React/Vue mini-app yang berkomunikasi ke Controller `/Api/Bot/Assist`.

---

## 30. Ekstensibilitas Arsitektur (Architecture & Plugin Webhooks)

### [30.1] Plugin System (Marketplace Internal Module)

**Status saat ini:** Aplikasi berbentuk monolit dengan *services* yang mengikat. Sangat suit bila ada cabang perusahaan di negara lain yang ingin fitur regulasi unik, tanpa harus merubah *Core App*.

**Fitur baru:**
- Arsitektur berbasis Modul (Modular Monolith / Plugin). Admin bisa menghidup-matikan (Toggle *Enable/Disable*) modul fitur seperti "Reservasi Lanjutan", "Kitting BOM", "E-Signature" selayaknya toko *App*.
- Hanya memuat memori (Dependency Injection) fitur yang aktif sehingga mengurangi *footprint* bagi server yang dipakai perusahaan skala kecil.

**Implementasi:**
- Penggunaan library `Prism`, Oqtane, atau arsitektur `.NET MEF` (Managed Extensibility Framework).
- Pemindahan setiap folder `Services` untuk modul baru dipecah menjadi file DLL (Class Libraries) terpisah (contoh: `InventoryMgmt.Plugins.ESignature.dll`).
- Buat dashboard Super Admin: *Plugin Manager*.

---

## 31. Analitik Anggaran Operasional (Financial Opex Management)

### [31.1] Prediksi Beban Operasional Tahunan (Opex Forecasting)

**Status saat ini:** Biaya pemeliharaan yang terintegrasi (seperti di modul 5.2 dan 28.1) masih sebatas pelacakan per item. Manajemen tidak mendapatkan visibilitas untuk penganggaran OPEX Inventaris tahun depan.

**Fitur baru:**
- Modul *Financial Planner* khusus inventaris. Sistem memproyeksikan total estimasi biaya yang harus dikeluarkan tahun depan.
- Prediksi mempertimbangkan: jadwal penggantian aset yang *End-of-Life*, kenaikan biaya langganan SLA Vendor otomatis, dan rata-rata pembelian *consumable* 3 tahun terakhir.
- Kemampuan *export/sync* "Tabel Rencana OPEX Inventaris 2027" ke modul ERP Keuangan.

**Implementasi:**
- Buat entitas `OpexBudgetProjection` (ProjectionId, Year, ProjectedTotal, BreakdownJson)
- Buat skrip *Aggregator* yang membaca relasi `AssetInsurance`, `VendorContract`, `MaintenanceExecution`, dan `PurchaseOrder`.
- Buat laporan grafik Pareto di modul *Finance Dashboard* untuk menentukan Kategori apa yang paling membebani Operasional.

---

## 32. Standar Manajemen Mutu (ISO Compliance & Audit)

### [32.1] Rekaman Audit Mutu ISO 9001/27001 (QMS Document Control)

**Status saat ini:** Dokumentasi dan proses kalibrasi (ISO) tidak dicatat dalam kerangka formal yang bisa memuaskan auditor eksternal. (Misal: bukti bahwa semua alat ukur gudang masih valid kalibrasinya).

**Fitur baru:**
- Alur *Document Control* khusus ISO. Semua pengunggahan manual/sertifikat (*ItemDocument*) akan dimintai tinjauan ulang tahunan oleh *Quality Manager*.
- Laporan pra-cetak "Daftar Induk Inventaris Terkalibrasi" yang disesuaikan persis dengan template auditor ISO.
- Pembekuan (*Freezing/Quarantine*) secara otomatis terhadap alat yang sertifikat kalibrasinya terlambat diperbarui, agar alat tersebut tidak bisa dipesan/dirilis.

**Implementasi:**
- Buat entitas `IsoCertificationRecord` (RecordId, ItemDocumentId, EvaluatedBy, NextEvaluationDate, IsCompliant).
- Relasikan role `QualityManager` untuk menyetujui log `ItemDocument` berjenis *Calibration Certificate*.
- Cegat logika di `ReservationEmployeeService` jika status `IsoCertificationRecord` = `Expired/Quarantined`, berikan notifikasi: "Item ini sedang dikarantina untuk verifikasi Mutu/ISO".
