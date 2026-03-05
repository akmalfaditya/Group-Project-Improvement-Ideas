# Future Development: Sistem Pelacakan Rekrutmen (Applicant Tracking System)

## 1. Manajemen Pipeline Rekrutmen Lanjutan

### [1.1] Kustomisasi Tahapan Pipeline Rekrutmen

**Status saat ini:** Pipeline rekrutmen bersifat statis dengan tahapan yang di-hardcode dalam enum `ProcessType` (Administration, HRInterview, UserInterview, Offering, Accepted, Rejected, Withdraw, TalentPool, Sourced, Closed). Tidak ada mekanisme bagi Admin/HR untuk menambah, menghapus, atau mengubah urutan tahapan pipeline sesuai kebutuhan posisi tertentu.

**Fitur baru:**

- Kemampuan membuat pipeline rekrutmen kustom per lowongan kerja (misalnya: menambahkan tahap "Technical Test" atau "Psychotest" di antara HR Interview dan User Interview)
- Drag-and-drop interface untuk mengatur ulang urutan tahapan pipeline
- Template pipeline yang dapat digunakan ulang untuk berbagai jenis posisi (Engineering, Marketing, Finance, dll.)
- Validasi otomatis agar kandidat tidak dapat melompati tahapan yang diwajibkan

**Implementasi:**

- Model baru: `RecruitmentPipeline`, `PipelineStage` dengan field `StageOrder`, `StageName`, `IsRequired`, `StageDuration`
- DTO baru: `PipelineTemplateDTO`, `PipelineStageDTO`, `CustomPipelineCreateDTO`
- Service baru: `IPipelineService` / `PipelineService` untuk CRUD pipeline dan validasi transisi tahapan
- Modifikasi `JobService` dan `RecruitmentProcessService` untuk mendukung pipeline dinamis
- Modifikasi `CreateJobDTO` untuk menyertakan `PipelineTemplateId`
- Endpoint baru pada `AdminController` dan `HRController`: `ManagePipeline`, `CreatePipelineTemplate`, `ReorderStages`

---

### [1.2] SLA (Service Level Agreement) dan Deadline Otomatis per Tahapan

**Status saat ini:** Tidak ada pelacakan waktu atau deadline untuk setiap tahapan rekrutmen. Tidak ada peringatan ketika kandidat terlalu lama berada di satu tahapan, sehingga proses rekrutmen bisa terhambat tanpa terdeteksi.

**Fitur baru:**

- Konfigurasi SLA per tahapan pipeline (misalnya: maksimal 3 hari di tahap Administration, 7 hari di tahap HR Interview)
- Notifikasi otomatis ke HR/Admin ketika SLA hampir atau sudah terlewati
- Dashboard "Bottleneck Detector" yang menampilkan tahapan mana yang paling sering melebihi SLA
- Eskalasi otomatis ke supervisor jika SLA kritis terlewati

**Implementasi:**

- Model baru: `StageSLA` dengan field `MaxDurationHours`, `WarningThresholdHours`, `EscalationEmail`
- Field baru pada `JobApplication`: `StageEnteredAt`, `SLADeadline`, `IsOverdue`
- Service baru: `ISLAMonitoringService` / `SLAMonitoringService` dengan background job menggunakan `IHostedService`
- Modifikasi `NotificationApplicantService` untuk mengirim notifikasi SLA warning
- Endpoint baru pada `AdminController`: `SLADashboard`, `ConfigureSLA`
- Library tambahan: Hangfire atau Quartz.NET untuk scheduled background jobs

---

### [1.3] Bulk Action dan Mass Processing pada Kandidat

**Status saat ini:** Sistem sudah mendukung bulk action untuk job (BulkCloseJobs, BulkActivateJobs, BulkSoftDeleteJobs, BulkHardDeleteJobs), namun belum ada bulk action untuk memproses kandidat secara massal di dalam pipeline rekrutmen.

**Fitur baru:**

- Bulk move kandidat antar tahapan (misalnya: memindahkan 10 kandidat sekaligus dari Administration ke HR Interview)
- Bulk reject kandidat yang tidak memenuhi kriteria minimum
- Bulk send email (undangan interview, penolakan, offering) ke beberapa kandidat sekaligus
- Bulk download CV dari kandidat terpilih dalam format ZIP

**Implementasi:**

- DTO baru: `BulkCandidateActionDTO` dengan field `List<Guid> JobApplicationIds`, `TargetStatus`, `EmailTemplateId`
- Modifikasi `IRecruitmentProcessService`: tambah method `BulkAcceptApplicantsAsync`, `BulkRejectApplicantsAsync`, `BulkMoveStageAsync`
- Modifikasi `IEmailService`: tambah method `SendBulkEmailAsync`
- Modifikasi `IFileService`: tambah method `DownloadBulkCVAsync` yang mengembalikan file ZIP
- Endpoint baru pada `AdminController` dan `HRController`: `BulkMoveStage`, `BulkReject`, `BulkSendEmail`, `BulkDownloadCV`
- Library tambahan: `System.IO.Compression` untuk fitur ZIP download

---

### [1.4] Kanban Board Visual untuk Pipeline Rekrutmen

**Status saat ini:** Tampilan pipeline rekrutmen menggunakan tabel/list standar per tahapan (Administration, HRInterview, EndUserInterview, Offering). Tidak ada visualisasi drag-and-drop yang interaktif untuk memindahkan kandidat antar tahapan.

**Fitur baru:**

- Kanban board dengan kolom per tahapan pipeline, menampilkan card kandidat yang bisa di-drag antar kolom
- Quick preview card kandidat (foto, nama, ATS score, tanggal apply) tanpa perlu membuka halaman detail
- Filter dan sort pada kanban board berdasarkan ATS score, tanggal apply, departemen
- Real-time update menggunakan SignalR ketika kandidat dipindahkan oleh user lain

**Implementasi:**

- Frontend: Implementasi library JavaScript drag-and-drop (SortableJS atau serupa)
- Service baru: `IKanbanService` / `KanbanService` untuk mendapatkan data pipeline dalam format kolom
- DTO baru: `KanbanBoardDTO`, `KanbanColumnDTO`, `KanbanCardDTO`
- SignalR Hub baru: `RecruitmentKanbanHub` untuk real-time collaboration
- Endpoint baru pada `AdminController` dan `HRController`: `KanbanView`, `MoveCandidate` (API endpoint untuk drag-drop)
- Modifikasi `RecruitmentProcessService` untuk mengirim broadcast SignalR saat ada perubahan status

---

## 2. AI dan Otomasi Screening Kandidat

### [2.1] Penyempurnaan AI ATS Scoring dengan Multi-Criteria Analysis

**Status saat ini:** Sistem sudah memiliki AI ATS scoring dasar melalui `AiAtsService` yang menganalisis resume dan menghasilkan `AtsScore`, `AtsSummary`, `AtsPros`, `AtsCons`, `AtsMatchedSkills`, `AtsMissingSkills`. Namun scoring masih berbasis single-pass analysis tanpa bobot kriteria yang dapat dikonfigurasi.

**Fitur baru:**

- Konfigurasi bobot scoring per kriteria (misalnya: Skill Match 40%, Education 20%, Experience 30%, GPA 10%)
- Breakdown score per kategori yang ditampilkan dalam radar/spider chart
- Perbandingan side-by-side ATS score antara beberapa kandidat untuk satu posisi
- Historical accuracy tracking: melacak korelasi antara ATS score dan hasil akhir rekrutmen (accepted/rejected) untuk meningkatkan akurasi model

**Implementasi:**

- Model baru: `AtsScoreBreakdown` dengan field `SkillScore`, `EducationScore`, `ExperienceScore`, `GPAScore`, `WeightConfig`
- Model baru: `AtsScoringConfig` dengan field `JobId`, `SkillWeight`, `EducationWeight`, `ExperienceWeight`
- Modifikasi `AtsRequestDTO` untuk menyertakan bobot kustom per job
- Modifikasi `AtsResponseDTO` untuk menyertakan breakdown score per kategori
- Modifikasi `IAiAtsService`: tambah method `AnalyzeResumeWithWeightsAsync`, `GetScoringAccuracyReportAsync`
- Endpoint baru pada `AdminController`: `ConfigureAtsWeights`, `CompareCandidate`, `AtsAccuracyReport`

---

### [2.2] Auto-Screening dan Auto-Rejection Berdasarkan Kriteria Minimum

**Status saat ini:** Proses screening kandidat di tahap Administration dilakukan secara manual oleh HR. Tidak ada mekanisme otomatis untuk menyaring kandidat berdasarkan kriteria minimum yang sudah didefinisikan di lowongan kerja (Education, GPA minimum, dll.).

**Fitur baru:**

- Konfigurasi auto-screening rules per lowongan (minimum GPA, minimum education level, required major, minimum ATS score)
- Otomatis memindahkan kandidat yang memenuhi semua kriteria ke tahap berikutnya
- Otomatis menolak kandidat yang tidak memenuhi kriteria minimum dengan email penolakan otomatis
- Dashboard "Auto-Screening Results" yang menampilkan statistik pass/fail per kriteria

**Implementasi:**

- Model baru: `ScreeningRule` dengan field `JobId`, `RuleType` (GPA, Education, Major, AtsScore), `MinimumValue`, `IsAutoReject`
- DTO baru: `ScreeningRuleDTO`, `AutoScreeningResultDTO`
- Service baru: `IAutoScreeningService` / `AutoScreeningService`
- Modifikasi `ApplicantService.ApplyJobAsync` untuk menjalankan auto-screening setelah ATS scoring
- Modifikasi `IEmailService` untuk mengirim auto-rejection email
- Endpoint baru pada `AdminController` dan `HRController`: `ConfigureScreeningRules`, `AutoScreeningDashboard`

---

### [2.3] AI-Powered Interview Question Generator

**Status saat ini:** Proses persiapan interview dilakukan secara manual. HR dan EndUser interviewer tidak mendapatkan rekomendasi pertanyaan berdasarkan profil kandidat, job requirement, atau hasil ATS analysis.

**Fitur baru:**

- Generate pertanyaan interview otomatis berdasarkan job requirement dan gap analysis dari ATS (AtsMissingSkills, AtsCons)
- Kategori pertanyaan: Technical, Behavioral, Situational, Culture Fit
- Interviewer dapat menandai pertanyaan yang sudah diajukan dan mencatat jawaban langsung di sistem
- Template bank soal per departemen yang dapat di-reuse

**Implementasi:**

- Model baru: `InterviewQuestion` dengan field `QuestionText`, `Category`, `DifficultyLevel`, `SourceType` (AI-Generated, Manual, Template)
- Model baru: `InterviewQuestionBank` dengan field `DepartmentId`, `Questions`
- DTO baru: `InterviewQuestionDTO`, `InterviewPrepDTO`
- Service baru: `IInterviewPrepService` / `InterviewPrepService` yang terintegrasi dengan AI provider
- Modifikasi `HrInterview` dan `EndUserInterview`: tambah relasi ke `InterviewQuestion`
- Endpoint baru pada `HRController` dan `EndUserController`: `GenerateInterviewQuestions`, `ManageQuestionBank`

---

### [2.4] AI Process Analysis dan Recruitment Insights

**Status saat ini:** Sistem sudah memiliki `AiProcessAnalysisService` dan `IAiProcessAnalysisService`, namun fitur ini masih terbatas. Belum ada analisis mendalam terhadap keseluruhan proses rekrutmen untuk mengidentifikasi pola, bottleneck, dan prediksi keberhasilan.

**Fitur baru:**

- Analisis prediktif: prediksi kemungkinan kandidat menerima offering berdasarkan data historis
- Analisis durasi rata-rata per tahapan pipeline dan identifikasi anomali
- Rekomendasi perbaikan proses berdasarkan data (misalnya: "Tahap User Interview rata-rata memakan waktu 15 hari, 2x lebih lama dari benchmark")
- Analisis kualitas sumber kandidat (dari job portal mana kandidat terbaik berasal)

**Implementasi:**

- Model baru: `RecruitmentAnalytics` dengan field `MetricType`, `Value`, `Period`, `JobId`
- DTO baru: `ProcessAnalyticsDTO`, `PredictionResultDTO`, `BottleneckReportDTO`
- Modifikasi `IAiProcessAnalysisService`: tambah method `PredictOfferingAcceptanceAsync`, `AnalyzeProcessEfficiencyAsync`, `GetSourceQualityReportAsync`
- Endpoint baru pada `AdminController`: `RecruitmentInsights`, `ProcessAnalytics`, `PredictiveReport`
- Library tambahan: ML.NET untuk model prediktif lokal (opsional)

---

## 3. Manajemen Lowongan Kerja (Job Posting) Lanjutan

### [3.1] Job Posting Multi-Channel dan Integrasi Job Board

**Status saat ini:** Lowongan kerja hanya diposting secara internal pada aplikasi. Tidak ada integrasi dengan job board eksternal (LinkedIn, JobStreet, Glints, Kalibrr, dll.) untuk memperluas jangkauan distribusi lowongan.

**Fitur baru:**

- Integrasi API dengan job board eksternal untuk auto-publish lowongan
- Dashboard tracking performa posting per channel (jumlah view, jumlah apply, conversion rate)
- Konfigurasi channel distribusi per lowongan (pilih channel mana yang aktif)
- Auto-sync status lowongan (ketika job ditutup di sistem, otomatis ditarik dari job board)

**Implementasi:**

- Model baru: `JobPostingChannel` dengan field `ChannelName`, `ApiEndpoint`, `ApiKey`, `IsActive`
- Model baru: `JobChannelPosting` dengan field `JobId`, `ChannelId`, `ExternalPostingId`, `Status`, `ViewCount`, `ApplyCount`
- DTO baru: `JobChannelPostingDTO`, `ChannelPerformanceDTO`
- Service baru: `IJobDistributionService` / `JobDistributionService`
- Modifikasi `JobService.CreateJobAsync` dan `CloseJobAsync` untuk trigger publish/unpublish ke channel
- Endpoint baru pada `AdminController` dan `HRController`: `ManageChannels`, `ChannelPerformance`, `PublishToChannel`
- Library tambahan: `HttpClient` factory untuk API call ke external job boards

---

### [3.2] Job Approval Workflow

**Status saat ini:** Pembuatan lowongan kerja melalui `CreateJobDTO` langsung masuk ke sistem dengan status `IsDraft` atau langsung aktif. Tidak ada workflow persetujuan (approval) multi-level sebelum lowongan dipublikasikan.

**Fitur baru:**

- Workflow approval bertingkat: Creator → HR Manager → Department Head → Published
- Kemampuan reviewer memberikan komentar dan meminta revisi sebelum approve
- Notifikasi email otomatis ke approver ketika ada lowongan yang menunggu persetujuan
- Audit trail lengkap untuk setiap approval/rejection beserta komentar

**Implementasi:**

- Model baru: `JobApproval` dengan field `JobId`, `ApproverId`, `ApprovalLevel`, `Status` (Pending, Approved, Rejected, RevisionRequested), `Comments`, `ApprovedAt`
- Model baru: `ApprovalWorkflow` dengan field `WorkflowName`, `Levels` (collection of approval levels)
- DTO baru: `JobApprovalDTO`, `ApprovalActionDTO`
- Service baru: `IJobApprovalService` / `JobApprovalService`
- Modifikasi `JobService.CreateJobAsync` untuk memasukkan job ke approval queue, bukan langsung publish
- Modifikasi `IEmailService`: tambah method `SendApprovalRequestEmailAsync`
- Endpoint baru pada `AdminController` dan `HRController`: `ApprovalQueue`, `ApproveJob`, `RejectJob`, `RequestRevision`

---

### [3.3] Job Cloning dan Template Lowongan

**Status saat ini:** Setiap lowongan kerja dibuat dari awal melalui `CreateJobDTO`. Tidak ada fitur untuk menduplikasi lowongan yang sudah ada atau menggunakan template standar, sehingga HR harus mengisi ulang data yang sama berulang kali.

**Fitur baru:**

- Clone lowongan yang sudah ada (semua data di-copy kecuali tanggal dan status)
- Simpan lowongan sebagai template yang dapat digunakan ulang
- Library template lowongan per departemen
- Bulk create jobs dari template (misalnya: buka 5 posisi Software Engineer sekaligus)

**Implementasi:**

- Model baru: `JobTemplate` dengan field `TemplateName`, `DepartmentId`, `TemplateData` (JSON), `CreatedBy`, `IsActive`
- DTO baru: `JobTemplateDTO`, `CloneJobDTO`, `BulkCreateJobDTO`
- Modifikasi `IJobService`: tambah method `CloneJobAsync`, `CreateFromTemplateAsync`, `BulkCreateJobsAsync`
- Modifikasi `IJobService`: tambah method `SaveAsTemplateAsync`, `GetTemplatesAsync`
- Endpoint baru pada `HRController`: `CloneJob`, `SaveAsTemplate`, `TemplateLibrary`, `BulkCreateFromTemplate`

---

### [3.4] Salary Benchmarking dan Range Management

**Status saat ini:** Field `Salary` pada `JobApplication` hanya menyimpan nilai tunggal. Tidak ada fitur salary range pada job posting, dan tidak ada perbandingan salary antar kandidat atau benchmark terhadap market rate.

**Fitur baru:**

- Salary range (min-max) pada setiap job posting
- Benchmark salary berdasarkan data internal (rata-rata salary yang diterima per departemen, lokasi, level pendidikan)
- Alert otomatis ketika expected salary kandidat melebihi budget range
- Analisis competitiveness salary offer terhadap pasar

**Implementasi:**

- Field baru pada `Job`: `SalaryRangeMin`, `SalaryRangeMax`, `SalaryCurrency`
- Model baru: `SalaryBenchmark` dengan field `DepartmentId`, `LocationId`, `EducationId`, `MedianSalary`, `Period`
- DTO baru: `SalaryBenchmarkDTO`, `SalaryComparisonDTO`
- Modifikasi `CreateJobDTO`: tambah field `SalaryRangeMin`, `SalaryRangeMax`
- Service baru: `ISalaryBenchmarkService` / `SalaryBenchmarkService`
- Endpoint baru pada `AdminController`: `SalaryBenchmark`, `SalaryAnalysis`

---

## 4. Manajemen Talent Pool Lanjutan

### [4.1] Talent Pool Segmentasi dan Kategorisasi Otomatis

**Status saat ini:** TalentPool menyimpan kandidat dengan field `AdminNotes` dan `CachedSkills`. Tidak ada segmentasi atau kategorisasi otomatis kandidat berdasarkan keahlian, level, atau departemen yang relevan.

**Fitur baru:**

- Auto-tagging kandidat di talent pool berdasarkan skills yang diekstrak dari CV (AtsMatchedSkills)
- Segmentasi otomatis berdasarkan departemen, level pendidikan, salary range, dan area keahlian
- Filter dan pencarian advanced di talent pool (berdasarkan skill, lokasi, availability, last updated)
- Scoring/ranking kandidat di talent pool berdasarkan relevansi terhadap lowongan aktif

**Implementasi:**

- Model baru: `TalentPoolTag` dengan field `TagName`, `TagCategory` (Skill, Department, Level)
- Model baru: `TalentPoolTagMapping` dengan field `TalentPoolId`, `TagId`
- DTO baru: `TalentPoolFilterDTO`, `TalentPoolSegmentDTO`
- Modifikasi `ITalentPoolService`: tambah method `AutoTagTalentAsync`, `GetTalentsBySegmentAsync`, `SearchTalentPoolAsync`, `RankTalentsForJobAsync`
- Modifikasi `TalentPoolDTO`: tambah field `Tags`, `RelevanceScore`
- Endpoint baru pada `AdminController` dan `HRController`: `TalentPoolSearch`, `TalentPoolSegments`, `RankTalentsForJob`

---

### [4.2] Talent Pool Engagement dan Re-recruitment Campaign

**Status saat ini:** Kandidat yang disimpan di TalentPool hanya tersimpan secara pasif. Tidak ada fitur untuk secara proaktif menghubungi kembali kandidat di talent pool ketika ada lowongan baru yang relevan.

**Fitur baru:**

- Auto-matching: ketika lowongan baru dibuat, sistem otomatis mencocokkan dengan kandidat di talent pool dan mengirim notifikasi ke HR
- Email campaign ke kandidat talent pool yang relevan untuk mengundang mereka melamar kembali
- Tracking engagement: apakah kandidat membuka email, klik link, atau melamar kembali
- Reminder otomatis ke HR untuk me-review talent pool secara berkala

**Implementasi:**

- Model baru: `TalentEngagement` dengan field `TalentPoolId`, `CampaignType`, `EmailSentAt`, `EmailOpenedAt`, `LinkClickedAt`, `ReappliedAt`
- DTO baru: `TalentEngagementDTO`, `TalentCampaignDTO`
- Service baru: `ITalentEngagementService` / `TalentEngagementService`
- Modifikasi `JobService.CreateJobAsync` untuk trigger auto-matching dengan talent pool
- Modifikasi `ITalentMatchingService`: tambah method `MatchTalentPoolWithNewJobAsync`
- Endpoint baru pada `HRController`: `LaunchTalentCampaign`, `TalentEngagementReport`

---

### [4.3] Talent Pool Expiry dan Data Retention Management

**Status saat ini:** Kandidat di talent pool memiliki field `SavedAt` dan `DeletedAt` (soft delete), tapi tidak ada mekanisme otomatis untuk menangani data yang sudah usang atau kebijakan retensi data sesuai regulasi privasi.

**Fitur baru:**

- Konfigurasi masa berlaku data talent pool (misalnya: otomatis expired setelah 12 bulan)
- Notifikasi ke kandidat sebelum data di-expire untuk memberikan opsi perpanjangan (consent renewal)
- Auto-archive kandidat yang sudah expired
- Compliance report untuk audit data retention (GDPR/UU PDP compliance)

**Implementasi:**

- Field baru pada `TalentPool`: `ExpiryDate`, `ConsentRenewedAt`, `IsArchived`
- Model baru: `DataRetentionPolicy` dengan field `EntityType`, `RetentionPeriodMonths`, `NotifyBeforeDays`
- Service baru: `IDataRetentionService` / `DataRetentionService` dengan scheduled background job
- Modifikasi `ITalentPoolService`: tambah method `ArchiveExpiredTalentsAsync`, `SendConsentRenewalAsync`
- Endpoint baru pada `AdminController`: `DataRetentionPolicy`, `ComplianceReport`, `ArchiveManagement`

---

## 5. Interview Scheduling dan Evaluasi Lanjutan

### [5.1] Self-Service Interview Scheduling untuk Kandidat

**Status saat ini:** Jadwal interview (`HrInterview`, `EndUserInterview`) diatur secara manual oleh HR/Admin. Kandidat tidak memiliki kemampuan untuk memilih slot waktu interview yang tersedia atau mengajukan reschedule.

**Fitur baru:**

- Kandidat dapat melihat slot waktu interview yang tersedia dan memilih sendiri
- Interviewer dapat mengatur availability slot mereka di sistem
- Automatic conflict detection: cegah double-booking interviewer
- Kemampuan kandidat mengajukan reschedule dengan alasan (approval required)
- Reminder otomatis H-1 dan H-1jam sebelum interview via email dan in-app notification

**Implementasi:**

- Model baru: `InterviewerAvailability` dengan field `InterviewerId`, `AvailableDate`, `StartTime`, `EndTime`, `IsBooked`
- Model baru: `RescheduleRequest` dengan field `InterviewId`, `RequestedBy`, `NewDate`, `Reason`, `Status`
- DTO baru: `AvailabilitySlotDTO`, `RescheduleRequestDTO`
- Service baru: `IInterviewSchedulingService` / `InterviewSchedulingService`
- Modifikasi `ApplicantController`: tambah method `SelectInterviewSlot`, `RequestReschedule`
- Modifikasi `IScheduleService`: tambah method `GetAvailableSlotsAsync`, `BookSlotAsync`, `CheckConflictAsync`
- Endpoint baru pada `ApplicantController`: `AvailableSlots`, `BookSlot`, `Reschedule`

---

### [5.2] Interview Scorecard dan Structured Evaluation

**Status saat ini:** Evaluasi interview hanya berupa free-text notes (`HrInterviewNotes`, `EndUserInterviewNotes` pada `JobApplication`). Tidak ada format evaluasi terstruktur atau scoring yang konsisten antar interviewer.

**Fitur baru:**

- Scorecard template dengan kriteria evaluasi terstruktur (Communication, Technical Skills, Problem Solving, Culture Fit, dll.)
- Rating scale per kriteria (1-5) dengan panduan scoring
- Kalkulasi total score otomatis dengan bobot per kriteria
- Perbandingan scorecard antar interviewer untuk satu kandidat (inter-rater comparison)
- Agregasi score dari semua tahapan interview untuk menghasilkan "Overall Interview Score"

**Implementasi:**

- Model baru: `InterviewScorecard` dengan field `JobApplicationId`, `InterviewerId`, `InterviewType` (HR/EndUser), `TotalScore`, `Recommendation`
- Model baru: `ScorecardCriteria` dengan field `CriteriaName`, `Weight`, `MaxScore`
- Model baru: `ScorecardRating` dengan field `ScorecardId`, `CriteriaId`, `Score`, `Comments`
- DTO baru: `ScorecardDTO`, `ScorecardCriteriaDTO`, `ScorecardComparisonDTO`
- Service baru: `IScorecardService` / `ScorecardService`
- Modifikasi controller `HRController` dan `EndUserController`: tambah method `FillScorecard`, `ViewScorecardComparison`
- Endpoint baru: `SubmitScorecard`, `GetScorecardSummary`, `CompareInterviewerScores`

---

### [5.3] Panel Interview dan Multi-Interviewer Support

**Status saat ini:** Setiap tahapan interview (`HrInterview`, `EndUserInterview`) hanya terhubung dengan satu interviewer (`HrStaffId`, `EndUserStaffId`). Komentar pada kode `JobApplication` menunjukkan bahwa multi-interviewer behavior sudah dipertimbangkan tapi belum diimplementasikan (`// for multiple UserInterview behaviour`).

**Fitur baru:**

- Dukungan panel interview dengan beberapa interviewer dalam satu sesi
- Setiap interviewer dalam panel dapat mengisi scorecard secara independen
- Lead interviewer dapat menyimpulkan hasil panel interview
- Calendar invite otomatis ke semua panel member

**Implementasi:**

- Model baru: `InterviewPanel` dengan field `InterviewId`, `InterviewType`, `LeadInterviewerId`
- Model baru: `InterviewPanelMember` dengan field `PanelId`, `InterviewerId`, `HasSubmittedScorecard`
- Modifikasi `HrInterview` dan `EndUserInterview`: tambah relasi `ICollection<InterviewPanelMember>`
- DTO baru: `InterviewPanelDTO`, `PanelMemberDTO`
- Modifikasi `IRecruitmentProcessService`: tambah method `CreatePanelInterviewAsync`, `AddPanelMemberAsync`
- Modifikasi `IGoogleCalendarService` untuk mengirim invite ke semua panel member
- Endpoint baru pada `HRController`: `CreatePanelInterview`, `ManagePanelMembers`

---

### [5.4] Video Interview Recording dan Playback

**Status saat ini:** Interview mendukung mode online dengan `MeetingLink` dan `IsOnline` pada `HrInterview` dan `EndUserInterview`. Namun tidak ada fitur recording, playback, atau timestamped notes yang terintegrasi langsung di sistem.

**Fitur baru:**

- Integrasi dengan platform video meeting untuk auto-record interview
- Penyimpanan recording interview yang terhubung langsung ke job application
- Timestamped notes: interviewer dapat menambahkan catatan yang terhubung ke timestamp tertentu dalam recording
- Share recording dengan interviewer/approver lain untuk review asynchronous

**Implementasi:**

- Model baru: `InterviewRecording` dengan field `InterviewId`, `RecordingUrl`, `StoragePath`, `Duration`, `RecordedAt`
- Model baru: `TimestampedNote` dengan field `RecordingId`, `Timestamp`, `Note`, `AuthorId`
- DTO baru: `InterviewRecordingDTO`, `TimestampedNoteDTO`
- Service baru: `IInterviewRecordingService` / `InterviewRecordingService`
- Modifikasi `HrInterview` dan `EndUserInterview`: tambah relasi ke `InterviewRecording`
- Endpoint baru pada `HRController` dan `EndUserController`: `UploadRecording`, `ViewRecording`, `AddTimestampedNote`
- Library tambahan: Azure Blob Storage atau AWS S3 SDK untuk penyimpanan video

---

## 6. Notifikasi dan Komunikasi Lanjutan

### [6.1] Notifikasi Real-Time dengan SignalR

**Status saat ini:** Notifikasi kepada kandidat disimpan di `NotificationApplicant` dengan field `IsRead`, namun pengiriman bersifat passive (kandidat harus refresh halaman untuk melihat notifikasi baru). Email dikirim melalui `EmailService` secara terpisah.

**Fitur baru:**

- Push notification real-time ke dashboard kandidat dan HR menggunakan SignalR
- Badge counter notifikasi yang live-update tanpa refresh halaman
- Toast notification untuk event penting (ada aplikasi baru masuk, interview dijadwalkan, status berubah)
- Notification preferences: kandidat dan HR dapat memilih jenis notifikasi yang ingin diterima

**Implementasi:**

- SignalR Hub baru: `NotificationHub` untuk broadcast real-time
- Model baru: `NotificationPreference` dengan field `UserId`, `NotificationType`, `IsEmailEnabled`, `IsPushEnabled`
- DTO baru: `NotificationPreferenceDTO`, `RealTimeNotificationDTO`
- Modifikasi `INotificationApplicantService`: tambah method `SendRealTimeNotificationAsync`, `GetNotificationPreferencesAsync`
- Modifikasi `NotificationApplicant`: tambah field `NotificationType` validation enum
- Endpoint baru pada `ApplicantController`: `NotificationPreferences`, `UpdatePreferences`
- JavaScript client: implementasi SignalR client di layout page untuk semua role

---

### [6.2] Multi-Channel Communication (WhatsApp, SMS)

**Status saat ini:** Komunikasi dengan kandidat hanya melalui email (`EmailService`, `EmailTemplateService`). Tidak ada channel komunikasi alternatif yang terintegrasi.

**Fitur baru:**

- Integrasi WhatsApp Business API untuk mengirim notifikasi dan undangan interview
- Integrasi SMS gateway untuk notifikasi urgent (reminder interview H-1, update status)
- Kandidat dapat memilih channel komunikasi preferensi mereka
- Template pesan per channel (email, WhatsApp, SMS) yang dapat dikustomisasi

**Implementasi:**

- Model baru: `CommunicationChannel` dengan field `ChannelType` (Email, WhatsApp, SMS), `ProviderConfig`, `IsActive`
- Model baru: `MessageTemplate` dengan field `ChannelType`, `TemplateType`, `Content`, `Variables`
- Modifikasi `Applicant`: tambah field `PreferredChannel`, `WhatsAppNumber`
- DTO baru: `MessageTemplateDTO`, `CommunicationPreferenceDTO`
- Service baru: `IWhatsAppService` / `WhatsAppService`, `ISmsService` / `SmsService`
- Service baru: `ICommunicationOrchestratorService` yang memilih channel berdasarkan preferensi kandidat
- Endpoint baru pada `ConfigureController`: `ManageCommunicationChannels`, `ManageMessageTemplates`
- Library tambahan: Twilio SDK atau WABLAS SDK untuk WhatsApp/SMS

---

### [6.3] In-App Messaging antara HR dan Kandidat

**Status saat ini:** Tidak ada fitur komunikasi langsung antara HR dan kandidat di dalam aplikasi. Semua komunikasi dilakukan melalui email di luar sistem, sehingga tidak ada riwayat percakapan yang terlacak.

**Fitur baru:**

- Fitur chat/messaging dalam aplikasi antara HR dan kandidat per job application
- Riwayat percakapan yang terhubung langsung ke setiap job application
- File attachment dalam pesan (misalnya: kirim dokumen tambahan, portofolio)
- Notifikasi real-time untuk pesan baru menggunakan SignalR

**Implementasi:**

- Model baru: `Message` dengan field `SenderId`, `ReceiverId`, `JobApplicationId`, `Content`, `AttachmentUrl`, `SentAt`, `IsRead`
- Model baru: `Conversation` dengan field `JobApplicationId`, `ParticipantIds`, `LastMessageAt`
- DTO baru: `MessageDTO`, `ConversationDTO`
- Service baru: `IMessagingService` / `MessagingService`
- SignalR Hub baru: `ChatHub` untuk real-time messaging
- Endpoint baru pada `ApplicantController` dan `HRController`: `Messages`, `SendMessage`, `GetConversation`

---

## 7. Reporting dan Analytics Rekrutmen

### [7.1] Dashboard Analitik Rekrutmen Komprehensif

**Status saat ini:** Sistem memiliki fitur export CSV (`ExportJobsToCSVAsync`, `ExportJobApplicationsToCSVAsync`, `ExportAllJobApplicationsToCSVAsync`) dan audit trail. Namun belum ada dashboard analitik visual dengan KPI dan metrik rekrutmen yang real-time.

**Fitur baru:**

- Dashboard dengan KPI utama: Time-to-Hire, Cost-per-Hire, Source Effectiveness, Offer Acceptance Rate
- Chart visual: Funnel conversion rate per tahapan pipeline, trend aplikasi per bulan, distribusi kandidat per departemen
- Filter dinamis berdasarkan periode, departemen, lokasi, employment type
- Perbandingan performa rekrutmen antar periode (month-over-month, year-over-year)

**Implementasi:**

- Model baru: `RecruitmentKPI` dengan field `MetricName`, `Value`, `Period`, `DepartmentId`, `LocationId`
- DTO baru: `DashboardDTO`, `FunnelDataDTO`, `TrendDataDTO`, `KPIComparisonDTO`
- Service baru: `IRecruitmentAnalyticsService` / `RecruitmentAnalyticsService`
- Modifikasi `AuditTrail`: tambah field `DurationMinutes`, `MetricCategory` untuk kalkulasi KPI
- Endpoint baru pada `AdminController`: `AnalyticsDashboard`, `GetFunnelData`, `GetTrendData`, `GetKPIComparison`
- Library tambahan: Chart.js atau ApexCharts di frontend untuk visualisasi

---

### [7.2] Diversity dan Inclusion Reporting

**Status saat ini:** Tidak ada tracking atau laporan terkait keberagaman (diversity) dalam proses rekrutmen. Data demografis kandidat seperti gender, usia, atau disabilitas tidak dilacak.

**Fitur baru:**

- Tracking opsional data demografis kandidat (dengan consent) untuk analisis diversity
- Laporan diversity per tahapan pipeline (apakah ada bias di tahapan tertentu)
- Goal setting untuk target diversity per departemen
- Anonymized reporting untuk compliance tanpa mengekspos data personal

**Implementasi:**

- Model baru: `ApplicantDemographics` dengan field `ApplicantId`, `Gender`, `AgeRange`, `DisabilityStatus`, `ConsentGiven` (semua opsional)
- Model baru: `DiversityGoal` dengan field `DepartmentId`, `MetricType`, `TargetPercentage`, `Period`
- DTO baru: `DiversityReportDTO`, `DiversityGoalDTO`, `AnonymizedDiversityDTO`
- Service baru: `IDiversityReportingService` / `DiversityReportingService`
- Endpoint baru pada `AdminController`: `DiversityReport`, `SetDiversityGoals`
- Modifikasi `ApplicantController.EditProfile`: tambah form opsional untuk data demografis

---

### [7.3] Laporan Performa Recruiter dan Interviewer

**Status saat ini:** `AuditTrail` mencatat aktivitas user (Name, Role, Action, Job, Timestamp), namun tidak ada analisis performa individual recruiter atau interviewer. Tidak ada metrik untuk mengukur efektivitas setiap HR atau interviewer.

**Fitur baru:**

- Laporan performa per HR recruiter: jumlah job yang di-handle, rata-rata time-to-fill, conversion rate
- Laporan performa per interviewer: jumlah interview yang dilakukan, rata-rata score yang diberikan, feedback quality
- Leaderboard recruiter berdasarkan KPI yang dapat dikonfigurasi
- Identifikasi interviewer dengan pola scoring yang anomali (terlalu tinggi/rendah dibanding rata-rata)

**Implementasi:**

- Model baru: `RecruiterPerformance` dengan field `RecruiterId`, `Period`, `JobsHandled`, `AverageTimeToFill`, `ConversionRate`
- DTO baru: `RecruiterPerformanceDTO`, `InterviewerPerformanceDTO`, `LeaderboardDTO`
- Service baru: `IPerformanceReportingService` / `PerformanceReportingService`
- Modifikasi `AuditService` untuk menghitung metrik performa secara berkala
- Endpoint baru pada `AdminController`: `RecruiterPerformance`, `InterviewerPerformance`, `RecruiterLeaderboard`

---

## 8. Pengalaman Kandidat (Candidate Experience)

### [8.1] Portal Karir Publik yang Enhanced

**Status saat ini:** `HomeController` menampilkan halaman beranda dengan daftar lowongan. Kandidat bisa melakukan pencarian dan filter melalui `JobSearchService`. Namun portal karir masih sederhana tanpa fitur engagement seperti company profile, culture page, atau employee testimonial.

**Fitur baru:**

- Halaman company profile dengan visi, misi, culture values, dan foto/video kantor
- Section "Life at [Company]" dengan employee testimonial dan behind-the-scene
- FAQ page untuk pertanyaan umum seputar proses rekrutmen
- Landing page per departemen yang menampilkan lowongan spesifik dan informasi tim

**Implementasi:**

- Model baru: `CompanyProfile` dengan field `VisionMission`, `CultureValues`, `MediaGallery`, `EmployeeTestimonials`
- Model baru: `CareerPageContent` dengan field `SectionType`, `Title`, `Content`, `MediaUrl`, `SortOrder`
- DTO baru: `CompanyProfileDTO`, `CareerPageContentDTO`
- Service baru: `ICareerPortalService` / `CareerPortalService`
- Modifikasi `HomeController`: tambah method `CompanyProfile`, `DepartmentLanding`, `FAQ`
- Endpoint baru pada `ConfigureController`: `ManageCareerPageContent`, `ManageCompanyProfile`

---

### [8.2] Applicant Tracking Progress yang Transparan

**Status saat ini:** Kandidat dapat melacak status lamaran melalui `TrackJob` di `ApplicantController` yang menampilkan `ApplicantApplicationStatusDTO`. Namun progress tracking masih berupa status teks sederhana tanpa visual timeline atau estimasi waktu.

**Fitur baru:**

- Visual timeline/stepper yang menampilkan posisi kandidat di pipeline rekrutmen
- Estimasi waktu untuk setiap tahapan berdasarkan data historis
- Feedback loop: kandidat mendapatkan notifikasi setiap kali status berubah beserta penjelasan
- Kemampuan kandidat melihat "next steps" yang harus dilakukan setelah setiap tahapan

**Implementasi:**

- DTO baru: `ApplicationTimelineDTO` dengan field `Stages[]` (nama, status, entered date, estimated completion)
- DTO baru: `StageGuidanceDTO` dengan field `StageName`, `WhatToExpect`, `NextSteps`, `AverageWaitDays`
- Modifikasi `IApplicantService`: tambah method `GetApplicationTimelineAsync`, `GetStageGuidanceAsync`
- Modifikasi `ApplicantApplicationStatusDTO`: tambah field `TimelineStages`, `EstimatedCompletionDate`
- Endpoint baru pada `ApplicantController`: `ApplicationTimeline`, `StageGuidance`

---

### [8.3] Penyempurnaan Job Recommendation Engine

**Status saat ini:** Sistem sudah memiliki `JobRecommendation` model dan `JobRecommendationService` yang menyimpan `RecommendedJobIdsJson` dan `MajorSnapshot`. Namun rekomendasi masih berbasis major/jurusan dan belum memanfaatkan data enriched dari riwayat lamaran, skills, dan preferensi kandidat.

**Fitur baru:**

- Rekomendasi berbasis multi-signal: skills dari CV, riwayat lamaran, saved jobs, lokasi preferensi, salary expectation
- "Similar jobs" feature: menampilkan lowongan serupa ketika kandidat melihat detail lowongan
- Personalized job alerts berdasarkan profil dan perilaku kandidat (bukan hanya keyword subscription)
- Collaborative filtering: "Kandidat yang melamar posisi X juga melamar posisi Y"

**Implementasi:**

- Model baru: `ApplicantBehavior` dengan field `ApplicantId`, `ViewedJobId`, `ViewDuration`, `AppliedAt`, `SavedAt`
- Modifikasi `JobRecommendation`: tambah field `SignalSources`, `ConfidenceScore`, `RecommendationType` (Content-Based, Collaborative, Hybrid)
- DTO baru: `EnhancedRecommendationDTO`, `SimilarJobDTO`
- Modifikasi `IJobRecommendationService`: tambah method `GetEnhancedRecommendationsAsync`, `GetSimilarJobsAsync`, `TrackJobViewAsync`
- Modifikasi `JobSubscription`: tambah field `SkillBasedKeywords`, `SalaryRange`
- Endpoint baru pada `ApplicantController`: `SimilarJobs`, `EnhancedRecommendations`
- Middleware baru untuk tracking job view behavior

---

### [8.4] Resume Builder Terintegrasi

**Status saat ini:** Kandidat mengupload CV dalam format file melalui field `CV` pada `JobApplication`. Sistem hanya menyimpan file dan melakukan ekstraksi teks (`ExtractedCvText`). Tidak ada fitur untuk membantu kandidat membuat atau menyempurnakan CV mereka.

**Fitur baru:**

- Resume builder dalam aplikasi dengan template profesional
- Auto-populate resume dari profil kandidat (Applicant data, Education, GPA, Major)
- Tips dan saran AI untuk menyempurnakan resume berdasarkan job requirement
- Export resume ke PDF dengan formatting profesional
- Penyimpanan multiple versi resume per kandidat

**Implementasi:**

- Model baru: `ResumeVersion` dengan field `ApplicantId`, `VersionName`, `ResumeData` (JSON), `TemplateId`, `CreatedAt`, `IsDefault`
- Model baru: `ResumeTemplate` dengan field `TemplateName`, `TemplateHtml`, `PreviewImageUrl`, `Category`
- DTO baru: `ResumeBuilderDTO`, `ResumeVersionDTO`, `ResumeSuggestionDTO`
- Service baru: `IResumeBuilderService` / `ResumeBuilderService`
- Endpoint baru pada `ApplicantController`: `ResumeBuilder`, `SaveResumeVersion`, `ExportResumePDF`, `GetResumeSuggestions`
- Library tambahan: iTextSharp atau QuestPDF untuk PDF generation

---

## 9. Keamanan dan Audit Lanjutan

### [9.1] Role-Based Access Control (RBAC) Granular

**Status saat ini:** Sistem menggunakan ASP.NET Identity dengan role dasar: Admin, HR, EndUser, Applicant. Otorisasi menggunakan `[Authorize(Roles = "...")]` secara statis di controller. Tidak ada permission-level granular atau role kustom.

**Fitur baru:**

- Permission-based authorization (misalnya: "CanCreateJob", "CanViewAllApplications", "CanExportData")
- Role kustom yang dapat dibuat oleh Admin dengan kombinasi permission yang fleksibel
- Permission per scope (misalnya: HR hanya bisa melihat job di departemen-nya sendiri)
- Audit log khusus untuk perubahan permission dan role assignment

**Implementasi:**

- Model baru: `Permission` dengan field `PermissionName`, `Description`, `Category`
- Model baru: `RolePermission` dengan field `RoleId`, `PermissionId`
- Modifikasi `AppUser`: tambah relasi ke custom permission
- DTO baru: `PermissionDTO`, `RolePermissionDTO`, `CustomRoleDTO`
- Service baru: `IPermissionService` / `PermissionService`
- Custom authorization handler: `PermissionAuthorizationHandler` yang memeriksa permission per endpoint
- Modifikasi `ConfigureController.AppUsers`: tambah fitur manage permission per user/role
- Endpoint baru pada `ConfigureController`: `ManagePermissions`, `CreateCustomRole`, `AssignPermissions`

---

### [9.2] Audit Trail yang Comprehensive dan Tamper-Proof

**Status saat ini:** `AuditTrail` saat ini hanya menyimpan field dasar: `Name`, `Role`, `Action`, `Job`, `JobLocation`, `Timestamp`. Tidak ada tracking detail perubahan data (before/after values), IP address, atau device information.

**Fitur baru:**

- Detailed change tracking: simpan old value dan new value untuk setiap perubahan data
- Capture IP address, user agent, dan session information di setiap audit entry
- Immutable audit log (hashed chain) untuk mencegah manipulasi riwayat
- Advanced audit search dan filtering (berdasarkan entity, user, action type, date range)
- Export audit trail ke format yang compliance-ready (PDF, Excel)

**Implementasi:**

- Modifikasi `AuditTrail`: tambah field `EntityType`, `EntityId`, `OldValues` (JSON), `NewValues` (JSON), `IpAddress`, `UserAgent`, `PreviousHash`, `CurrentHash`
- DTO baru: `DetailedAuditTrailDTO`, `AuditSearchDTO`, `AuditExportDTO`
- Modifikasi `IAuditService`: tambah method `SearchAuditTrailAsync`, `ExportAuditTrailAsync`, `CalculateHashAsync`
- Implementasi `SaveChangesInterceptor` di EF Core untuk auto-capture perubahan pada semua entity
- Endpoint baru pada `AdminController`: `AuditSearch`, `AuditExport`, `AuditIntegrity` (untuk verifikasi integritas hash)

---

### [9.3] Data Encryption dan Privacy Protection

**Status saat ini:** Data sensitif kandidat (Phone, CV, Salary, GPA) disimpan tanpa enkripsi khusus di database. Tidak ada mekanisme data masking atau anonymization untuk compliance privasi data.

**Fitur baru:**

- Enkripsi field sensitif di database (Phone, Salary, CurrentSalary, ExpectedSalary)
- Data masking pada API response berdasarkan role (misalnya: HR hanya lihat 4 digit terakhir nomor telepon)
- Right to be forgotten: mekanisme untuk menghapus seluruh data kandidat secara permanen atas permintaan
- Anonymization otomatis data kandidat yang sudah lebih dari tahun tertentu untuk analisis historical

**Implementasi:**

- Service baru: `IDataProtectionService` / `DataProtectionService` menggunakan ASP.NET Data Protection API
- Value converter EF Core untuk enkripsi/dekripsi field sensitif secara transparan
- DTO baru: `DataDeletionRequestDTO`, `AnonymizationDTO`
- Service baru: `IPrivacyService` / `PrivacyService` untuk handle right-to-be-forgotten request
- Endpoint baru pada `AdminController`: `ProcessDeletionRequest`, `AnonymizeOldData`
- Modifikasi semua DTO yang mengandung data sensitif: tambah logic data masking berdasarkan role

---

## 10. Onboarding Pre-Hire Integration

### [10.1] Pre-Onboarding Checklist untuk Kandidat yang Diterima

**Status saat ini:** Setelah kandidat diterima (`ProcessType.Accepted`), tidak ada flow lanjutan di sistem. Proses onboarding dilakukan secara manual di luar aplikasi.

**Fitur baru:**

- Checklist pre-onboarding otomatis yang dikirim ke kandidat setelah diterima (upload dokumen KTP, NPWP, ijazah, rekening, dll.)
- Tracking completion status setiap item checklist
- Deadline dan reminder untuk dokumen yang belum disubmit
- Document verification oleh HR sebelum hari pertama masuk kerja

**Implementasi:**

- Model baru: `OnboardingChecklist` dengan field `JobApplicationId`, `ItemName`, `ItemType` (Document, Form, Task), `IsRequired`, `Deadline`
- Model baru: `OnboardingSubmission` dengan field `ChecklistItemId`, `SubmittedFileUrl`, `VerifiedByHR`, `VerifiedAt`, `Status`
- DTO baru: `OnboardingChecklistDTO`, `OnboardingSubmissionDTO`, `OnboardingProgressDTO`
- Service baru: `IPreOnboardingService` / `PreOnboardingService`
- Modifikasi `IRecruitmentProcessService.AcceptApplicantAsync` untuk otomatis generate checklist
- Endpoint baru pada `ApplicantController`: `OnboardingChecklist`, `SubmitOnboardingDocument`
- Endpoint baru pada `HRController`: `VerifyOnboardingDocument`, `OnboardingProgress`

---

### [10.2] Digital Offer Letter dan E-Signature

**Status saat ini:** Offering dikirim melalui email (`EmailOffering` di `EmailTemplate`) dengan template sederhana. Tidak ada fitur offer letter formal yang dapat ditandatangani secara digital oleh kandidat.

**Fitur baru:**

- Generate offer letter formal dari template dengan merge fields (nama, posisi, salary, start date, terms)
- E-signature integration: kandidat dapat menandatangani offer letter secara digital di dalam aplikasi
- Tracking status offer letter: sent, viewed, signed, declined, expired
- Counter-offer mechanism: kandidat dapat mengajukan negosiasi sebelum menandatangani
- Penyimpanan offer letter yang sudah ditandatangani sebagai dokumen legal

**Implementasi:**

- Model baru: `OfferLetter` dengan field `JobApplicationId`, `TemplateData`, `GeneratedPdfUrl`, `Status` (Draft, Sent, Viewed, Signed, Declined, Expired), `SignedAt`, `SignatureImageUrl`
- Model baru: `OfferNegotiation` dengan field `OfferLetterId`, `NegotiationType` (Salary, StartDate, Benefits), `ProposedValue`, `Status`
- DTO baru: `OfferLetterDTO`, `OfferLetterCreateDTO`, `OfferNegotiationDTO`
- Service baru: `IOfferLetterService` / `OfferLetterService`
- Endpoint baru pada `HRController`: `GenerateOfferLetter`, `SendOfferLetter`, `ViewOfferLetterStatus`
- Endpoint baru pada `ApplicantController`: `ViewOfferLetter`, `SignOfferLetter`, `NegotiateOffer`
- Library tambahan: QuestPDF atau iTextSharp untuk PDF generation, SignatureJS untuk e-signature frontend

---

## 11. Referral dan Employee Sourcing

### [11.1] Employee Referral Program

**Status saat ini:** Semua kandidat masuk melalui self-application di portal karir. Tidak ada mekanisme referral dari karyawan internal yang sudah ada, dan sumbernya tidak dilacak.

**Fitur baru:**

- Karyawan internal dapat mereferensikan kandidat untuk lowongan tertentu
- Tracking referral source: dari siapa kandidat dirujuk, kapan, dan status di pipeline
- Dashboard referral untuk karyawan: melihat status semua referral yang sudah disubmit
- Referral analytics: conversion rate referral vs non-referral, time-to-hire comparison

**Implementasi:**

- Model baru: `Referral` dengan field `ReferredBy` (AppUserId), `ReferredApplicantId`, `JobId`, `ReferralDate`, `Status`, `Notes`
- Modifikasi `JobApplication`: tambah field `SourceType` (Direct, Referral, JobBoard, TalentPool), `ReferralId`
- DTO baru: `ReferralDTO`, `ReferralDashboardDTO`, `ReferralAnalyticsDTO`
- Service baru: `IReferralService` / `ReferralService`
- Controller baru: `ReferralController` untuk endpoint referral karyawan internal
- Endpoint baru: `SubmitReferral`, `MyReferrals`, `ReferralAnalytics`

---

### [11.2] Proactive Sourcing dan Candidate Pipeline Building

**Status saat ini:** `TalentMatchingService` hanya mencocokkan kandidat dari talent pool yang sudah ada dengan lowongan. Tidak ada fitur sourcing proaktif untuk membangun pipeline kandidat sebelum ada lowongan.

**Fitur baru:**

- Sourcing campaign: HR dapat membuat campaign untuk mengumpulkan kandidat potensial bahkan sebelum ada lowongan resmi
- Integration readiness untuk LinkedIn Recruiter dan platform sourcing lainnya
- Candidate pipeline per departemen: pool kandidat potensial yang dikelompokkan berdasarkan keahlian yang dibutuhkan departemen
- Sourcing analytics: efektivitas setiap channel sourcing

**Implementasi:**

- Model baru: `SourcingCampaign` dengan field `CampaignName`, `TargetDepartmentId`, `TargetSkills`, `Status`, `StartDate`, `EndDate`
- Model baru: `SourcedCandidate` dengan field `CampaignId`, `ApplicantId`, `SourceChannel`, `SourcedBy`, `EngagementStatus`
- DTO baru: `SourcingCampaignDTO`, `SourcedCandidateDTO`, `SourcingAnalyticsDTO`
- Service baru: `ISourcingService` / `SourcingService`
- Endpoint baru pada `HRController`: `CreateSourcingCampaign`, `AddSourcedCandidate`, `SourcingAnalytics`
- Modifikasi `ProcessType.Sourced` untuk terintegrasi dengan sourcing campaign flow

---

## 12. Integrasi Kalender dan Scheduling Lanjutan

### [12.1] Penyempurnaan Google Calendar Integration

**Status saat ini:** Sistem sudah memiliki `GoogleCalendarService`, `IGoogleCalendarService`, dan `CalendarServiceBuilder` di area Identity/Services. Namun integrasi ini belum mencakup fitur lengkap seperti auto-create event, status sync, dan multi-calendar support.

**Fitur baru:**

- Auto-create Google Calendar event ketika jadwal interview disimpan
- Two-way sync: perubahan di Google Calendar tercermin di sistem dan sebaliknya
- Support Outlook/Office 365 Calendar sebagai alternatif
- Tampilkan availability interviewer dari calendar untuk smart scheduling
- iCalendar (.ics) file attachment di email undangan interview

**Implementasi:**

- Modifikasi `IGoogleCalendarService`: tambah method `CreateInterviewEventAsync`, `SyncCalendarAsync`, `GetAvailabilityAsync`
- Service baru: `IOutlookCalendarService` / `OutlookCalendarService` untuk Microsoft Graph API
- Service baru: `ICalendarAbstraction` sebagai interface shared antara Google dan Outlook calendar
- Modifikasi `IRecruitmentProcessService.SaveHRInterviewScheduleAsync` dan `SaveEndUserInterviewScheduleAsync` untuk auto-create calendar event
- DTO baru: `CalendarEventDTO`, `AvailabilityDTO`, `CalendarSyncDTO`
- Library tambahan: Microsoft Graph SDK untuk Outlook Calendar, iCal.NET untuk .ics file generation

---

### [12.2] Smart Scheduling dengan Conflict Detection

**Status saat ini:** Penjadwalan interview dilakukan secara manual tanpa pengecekan konflik. Jika interviewer sudah memiliki jadwal di waktu yang sama, sistem tidak memberikan peringatan.

**Fitur baru:**

- Auto-detection konflik jadwal interviewer sebelum menyimpan schedule
- Suggest time slots alternatif berdasarkan availability semua pihak (interviewer + kandidat)
- Buffer time otomatis antar interview (misalnya: 15 menit gap)
- Timezone awareness untuk interview cross-timezone
- Batch scheduling: jadwalkan beberapa kandidat sekaligus dengan time slots berurutan

**Implementasi:**

- Model baru: `ScheduleConflict` dengan field `InterviewerId`, `ConflictingEventA`, `ConflictingEventB`, `SuggestedAlternatives`
- DTO baru: `SmartScheduleRequestDTO`, `ScheduleSuggestionDTO`, `BatchScheduleDTO`
- Modifikasi `IScheduleService`: tambah method `DetectConflictsAsync`, `SuggestTimeSlotsAsync`, `BatchScheduleAsync`
- Modifikasi `IRecruitmentProcessService`: integrasi conflict detection sebelum `SaveHRInterviewScheduleAsync`
- Endpoint baru pada `HRController`: `CheckConflicts`, `GetScheduleSuggestions`, `BatchScheduleInterviews`
- Konfigurasi baru: `BufferTimeMinutes`, `DefaultTimezone` di `appsettings.json`

---

## 13. Assessment dan Technical Testing Terintegrasi

### [13.1] Online Assessment Platform dalam Rekrutmen

**Status saat ini:** Proses rekrutmen tidak memiliki mekanisme assessment atau tes kemampuan terintegrasi. Evaluasi teknis kandidat bergantung sepenuhnya pada review CV (ATS scoring) dan interview verbal. Tidak ada pengujian skill secara objektif di dalam sistem.

**Fitur baru:**

- Modul assessment online yang dapat dikonfigurasi per lowongan (misalnya: tes logika, tes bahasa Inggris, tes teknis coding)
- Bank soal per kategori yang dapat dikelola oleh HR dan departemen
- Timer dan anti-cheating measures (screen lock, tab detection, randomized questions)
- Auto-grading untuk soal pilihan ganda dan scoring rubric untuk soal esai
- Integrasi skor assessment ke dalam pipeline rekrutmen sebagai tahapan yang terukur

**Implementasi:**

- Model baru: `Assessment` dengan field `AssessmentName`, `JobId`, `DurationMinutes`, `PassingScore`, `QuestionCount`, `IsRandomized`
- Model baru: `AssessmentQuestion` dengan field `AssessmentId`, `QuestionText`, `QuestionType` (MultipleChoice, Essay, Coding), `CorrectAnswer`, `Points`, `Category`
- Model baru: `AssessmentAttempt` dengan field `AssessmentId`, `JobApplicationId`, `StartedAt`, `CompletedAt`, `Score`, `Status` (InProgress, Completed, TimedOut)
- Model baru: `AssessmentAnswer` dengan field `AttemptId`, `QuestionId`, `AnswerText`, `IsCorrect`, `PointsEarned`
- DTO baru: `AssessmentDTO`, `AssessmentQuestionDTO`, `AssessmentAttemptDTO`, `AssessmentResultDTO`
- Service baru: `IAssessmentService` / `AssessmentService`
- Controller baru: `AssessmentController` dengan endpoint `TakeAssessment`, `SubmitAssessment`, `ViewResults`
- Endpoint baru pada `HRController`: `CreateAssessment`, `ManageQuestionBank`, `AssessmentResults`
- Modifikasi `ProcessType`: tambah nilai `Assessment` sebagai tahapan opsional dalam pipeline

---

### [13.2] Coding Challenge dan Live Coding Interview

**Status saat ini:** Tidak ada fitur technical assessment untuk posisi engineering/IT. Evaluasi kemampuan coding kandidat dilakukan sepenuhnya di luar sistem (misalnya: HackerRank, LeetCode) tanpa tracking hasil di dalam aplikasi.

**Fitur baru:**

- Built-in code editor dengan syntax highlighting dan support multi-bahasa (C#, Python, JavaScript, Java, SQL)
- Predefined coding challenges per level (Junior, Mid, Senior) yang bisa dikustomisasi per job
- Auto-evaluation: compile dan run kode kandidat terhadap test cases predefined
- Live coding interview mode: interviewer dan kandidat dapat berkolaborasi di editor yang sama secara real-time
- Tracking metrics: completion time, test cases passed, code quality score

**Implementasi:**

- Model baru: `CodingChallenge` dengan field `Title`, `Description`, `Difficulty`, `LanguageSupport`, `TimeLimit`, `TestCases` (JSON)
- Model baru: `CodingSubmission` dengan field `ChallengeId`, `JobApplicationId`, `SourceCode`, `Language`, `TestCasesPassed`, `TotalTestCases`, `ExecutionTimeMs`, `SubmittedAt`
- DTO baru: `CodingChallengeDTO`, `CodingSubmissionDTO`, `CodeExecutionResultDTO`
- Service baru: `ICodingChallengeService` / `CodingChallengeService`
- SignalR Hub baru: `LiveCodingHub` untuk real-time collaborative editing
- Endpoint baru pada `HRController`: `CreateCodingChallenge`, `ViewCodingResults`
- Endpoint baru pada `ApplicantController`: `StartCodingChallenge`, `SubmitCode`, `RunTestCases`
- Library tambahan: Monaco Editor (frontend), Roslyn Compiler (backend C# execution), Docker SDK untuk sandboxed code execution

---

### [13.3] Psychometric dan Personality Assessment

**Status saat ini:** Tidak ada evaluasi kepribadian atau psychometric test terintegrasi. Penilaian culture fit dan soft skills kandidat sepenuhnya bergantung pada penilaian subjektif interviewer.

**Fitur baru:**

- Template psychometric test standar (DISC, MBTI-inspired, Big Five Personality traits)
- Custom personality assessment yang dapat disesuaikan dengan culture values perusahaan
- Visualisasi profil kepribadian kandidat dalam bentuk radar chart
- Perbandingan profil kepribadian kandidat dengan profil ideal untuk posisi tertentu
- Aggregate personality insights per departemen untuk membantu hiring decision

**Implementasi:**

- Model baru: `PsychometricTest` dengan field `TestName`, `TestType` (DISC, BigFive, Custom), `Questions` (JSON), `ScoringRubric`
- Model baru: `PsychometricResult` dengan field `TestId`, `JobApplicationId`, `Scores` (JSON), `ProfileType`, `FitScore`, `CompletedAt`
- DTO baru: `PsychometricTestDTO`, `PsychometricResultDTO`, `PersonalityProfileDTO`, `FitComparisonDTO`
- Service baru: `IPsychometricService` / `PsychometricService`
- Endpoint baru pada `HRController`: `AssignPsychometricTest`, `ViewPsychometricResults`, `ConfigureIdealProfile`
- Endpoint baru pada `ApplicantController`: `TakePsychometricTest`, `SubmitPsychometricTest`

---

## 14. Manajemen Email Template Lanjutan

### [14.1] Dynamic Email Template Builder dengan Variable Placeholders

**Status saat ini:** `EmailTemplate` model memiliki field statis: `EmailHR`, `EmailEndUser`, `InterviewHR`, `InterviewEndUser`, `EmailOffering`, `EmailReject`. Template disimpan sebagai string HTML sederhana tanpa variabel dinamis atau preview kemampuan.

**Fitur baru:**

- Visual email template editor (WYSIWYG) dengan drag-and-drop components
- Dynamic variable placeholders (misalnya: {{CandidateName}}, {{JobTitle}}, {{InterviewDate}}, {{InterviewLocation}}, {{MeetingLink}})
- Preview template dengan data contoh sebelum mengirim
- Versioning template: simpan riwayat perubahan template dan kemampuan rollback
- Kategori template yang lebih fleksibel (bukan hanya 6 jenis fixed)

**Implementasi:**

- Modifikasi `EmailTemplate`: refactor menjadi model yang lebih fleksibel dengan field `TemplateName`, `TemplateCategory`, `Subject`, `BodyHtml`, `Variables` (JSON), `Version`, `IsActive`, `CreatedBy`, `CreatedAt`, `UpdatedAt`
- Model baru: `EmailTemplateVersion` dengan field `EmailTemplateId`, `VersionNumber`, `BodyHtml`, `ModifiedBy`, `ModifiedAt`
- DTO baru: `DynamicEmailTemplateDTO`, `EmailPreviewDTO`, `TemplateVariableDTO`
- Modifikasi `IEmailTemplateService`: tambah method `RenderTemplateAsync`, `PreviewTemplateAsync`, `GetTemplateVersionsAsync`, `RollbackTemplateAsync`
- Modifikasi `IEmailService`: update seluruh method pengiriman email untuk menggunakan dynamic variable rendering
- Endpoint baru pada `AdminController` dan `HRController`: `EmailTemplateEditor`, `PreviewEmail`, `TemplateVersionHistory`, `RollbackTemplate`

---

### [14.2] Email Scheduling dan Automated Email Sequences

**Status saat ini:** Email dikirim secara manual satu per satu melalui action di controller (`SendHRInterview`, `SendEndUserInterview`, `SendOffering`, `SendEmailRejection`). Tidak ada fitur scheduling pengiriman email atau email sequence otomatis.

**Fitur baru:**

- Schedule email untuk dikirim di waktu tertentu (misalnya: kirim email rejection di akhir hari kerja, bukan langsung)
- Automated email sequences: serangkaian email yang otomatis terkirim berdasarkan trigger/timeline (misalnya: email follow-up 3 hari setelah offering dikirim jika belum ada respons)
- Email delivery tracking: status sent, delivered, opened, bounced
- Retry mechanism untuk email yang gagal terkirim

**Implementasi:**

- Model baru: `ScheduledEmail` dengan field `RecipientEmail`, `Subject`, `Body`, `ScheduledAt`, `SentAt`, `Status` (Pending, Sent, Failed, Cancelled), `RetryCount`
- Model baru: `EmailSequence` dengan field `SequenceName`, `TriggerEvent`, `Steps` (JSON: delay, template, condition)
- Model baru: `EmailTrackingEvent` dengan field `EmailId`, `EventType` (Sent, Delivered, Opened, Bounced), `EventAt`
- DTO baru: `ScheduledEmailDTO`, `EmailSequenceDTO`, `EmailTrackingDTO`
- Service baru: `IEmailSchedulingService` / `EmailSchedulingService`
- Background job: `EmailSchedulerJob` menggunakan `IHostedService` untuk memproses scheduled emails
- Endpoint baru pada `HRController`: `ScheduleEmail`, `CreateEmailSequence`, `EmailTrackingReport`
- Library tambahan: Hangfire atau Quartz.NET untuk job scheduling

---

### [14.3] Email Analytics dan Engagement Tracking

**Status saat ini:** Field `EmailSent` pada `JobApplication` hanya mencatat apakah email sudah dikirim (boolean). Tidak ada tracking apakah email dibaca oleh kandidat, di-klik, atau berakhir di spam.

**Fitur baru:**

- Open rate tracking: apakah kandidat membuka email (menggunakan tracking pixel)
- Click rate tracking: apakah kandidat mengklik link di dalam email (meeting link, portal link)
- Bounce rate monitoring: identifikasi email yang gagal terkirim
- Dashboard email analytics: overview performa email per jenis (invitation, rejection, offering)
- A/B testing support: kirim 2 versi template ke subset kandidat dan lihat mana yang lebih efektif

**Implementasi:**

- Model baru: `EmailAnalytics` dengan field `EmailId`, `OpenCount`, `ClickCount`, `FirstOpenedAt`, `LastOpenedAt`, `IsBounced`
- DTO baru: `EmailAnalyticsDashboardDTO`, `EmailPerformanceDTO`, `ABTestResultDTO`
- Modifikasi `IEmailService`: integrasi tracking pixel dan link wrapper
- Service baru: `IEmailAnalyticsService` / `EmailAnalyticsService`
- Endpoint baru (API): `TrackEmailOpen` (GET endpoint untuk tracking pixel callback), `TrackEmailClick`
- Endpoint baru pada `AdminController`: `EmailAnalyticsDashboard`, `CreateABTest`, `ABTestResults`
- Middleware: Email tracking pixel middleware untuk menangani callback

---

## 15. Workforce Planning dan Headcount Management

### [15.1] Headcount Request dan Manpower Planning

**Status saat ini:** Pembuatan lowongan dilakukan secara langsung oleh HR/Admin tanpa proses formal manpower request dari departemen. Tidak ada tracking kebutuhan headcount per departemen atau planning jangka panjang.

**Fitur baru:**

- Departemen/EndUser dapat mengajukan headcount request dengan justifikasi bisnis
- Approval workflow untuk headcount request: Department Head → HR → Finance/Management
- Dashboard manpower planning: kebutuhan headcount vs posisi yang sedang direkrut vs yang sudah terisi
- Forecasting: estimasi kebutuhan rekrutmen berdasarkan growth plan dan attrition rate
- Budget tracking per headcount request

**Implementasi:**

- Model baru: `HeadcountRequest` dengan field `DepartmentId`, `RequestedBy`, `PositionTitle`, `Justification`, `Quantity`, `Priority` (Critical, High, Medium, Low), `TargetStartDate`, `BudgetAmount`, `Status` (Draft, Pending, Approved, Rejected), `ApprovedBy`, `ApprovedAt`
- Model baru: `ManpowerPlan` dengan field `DepartmentId`, `Period`, `PlannedHeadcount`, `CurrentHeadcount`, `InRecruitment`, `Filled`
- DTO baru: `HeadcountRequestDTO`, `ManpowerPlanDTO`, `HeadcountForecastDTO`
- Service baru: `IWorkforcePlanningService` / `WorkforcePlanningService`
- Modifikasi `JobService.CreateJobAsync`: opsional link ke `HeadcountRequestId`
- Endpoint baru pada `EndUserController`: `SubmitHeadcountRequest`, `MyHeadcountRequests`
- Endpoint baru pada `AdminController`: `HeadcountApprovalQueue`, `ManpowerDashboard`, `HeadcountForecast`

---

### [15.2] Recruitment Budget Tracking dan Cost Analytics

**Status saat ini:** Tidak ada tracking biaya rekrutmen di dalam sistem. Tidak ada visibilitas terhadap cost-per-hire, biaya per channel, atau total pengeluaran rekrutmen per periode.

**Fitur baru:**

- Tracking biaya per lowongan: biaya posting di job board, biaya assessment tools, biaya perjalanan interview, dll.
- Kalkulasi otomatis Cost-per-Hire berdasarkan data yang tercatat
- Budget allocation per departemen dan monitoring pengeluaran vs budget
- Laporan ROI per channel rekrutmen (biaya yang dikeluarkan vs jumlah hire dari channel tersebut)

**Implementasi:**

- Model baru: `RecruitmentExpense` dengan field `JobId`, `ExpenseCategory` (JobBoard, Assessment, Travel, Agency, Other), `Amount`, `Currency`, `ExpenseDate`, `Description`, `RecordedBy`
- Model baru: `RecruitmentBudget` dengan field `DepartmentId`, `Period`, `AllocatedBudget`, `SpentAmount`
- DTO baru: `RecruitmentExpenseDTO`, `CostPerHireDTO`, `BudgetReportDTO`, `ChannelROIDTO`
- Service baru: `IBudgetTrackingService` / `BudgetTrackingService`
- Endpoint baru pada `AdminController`: `RecordExpense`, `BudgetDashboard`, `CostPerHireReport`, `ChannelROIReport`

---

## 16. Manajemen Blacklist dan Compliance Kandidat

### [16.1] Candidate Blacklist dan Do-Not-Hire Registry

**Status saat ini:** Kandidat yang ditolak (`ProcessType.Rejected`) atau melakukan withdraw (`ProcessType.Withdraw`) dapat mendaftar kembali tanpa batasan. Tidak ada mekanisme blacklist untuk kandidat yang bermasalah atau do-not-hire policy.

**Fitur baru:**

- Blacklist database untuk kandidat yang tidak boleh direkrut kembali (fraud, falsifikasi dokumen, perilaku tidak etis)
- Auto-check saat kandidat baru mendaftar atau melamar: apakah email/phone terdaftar di blacklist
- Blacklist dengan alasan dan periode (permanent atau temporary)
- Hanya Admin yang dapat menambah/menghapus dari blacklist dengan audit trail lengkap

**Implementasi:**

- Model baru: `CandidateBlacklist` dengan field `ApplicantId`, `Email`, `Phone`, `Reason`, `BlacklistedBy`, `BlacklistedAt`, `ExpiryDate` (null = permanent), `IsActive`
- DTO baru: `BlacklistDTO`, `BlacklistCheckDTO`
- Service baru: `IBlacklistService` / `BlacklistService`
- Modifikasi `IApplicantService.ApplyJobAsync`: integrasi blacklist check sebelum allow apply
- Modifikasi `AuditService`: log semua blacklist operations
- Endpoint baru pada `AdminController`: `ManageBlacklist`, `AddToBlacklist`, `RemoveFromBlacklist`, `CheckBlacklist`

---

### [16.2] Document Verification dan Background Check Tracking

**Status saat ini:** Dokumen kandidat (CV) di-upload tanpa proses verifikasi formal. Tidak ada tracking untuk background check atau verifikasi keaslian dokumen (ijazah, sertifikasi, referensi).

**Fitur baru:**

- Checklist verifikasi dokumen per kandidat: ijazah, transkrip nilai, KTP, sertifikasi professional
- Status verifikasi: Pending, Verified, Flagged, Rejected
- Tracking background check: reference check, criminal record check, education verification
- Integration readiness dengan provider background check pihak ketiga
- Notifikasi ke HR ketika verifikasi selesai atau ada dokumen yang ditandai bermasalah

**Implementasi:**

- Model baru: `DocumentVerification` dengan field `JobApplicationId`, `DocumentType` (Ijazah, Transkrip, KTP, Sertifikasi), `DocumentUrl`, `VerificationStatus`, `VerifiedBy`, `VerifiedAt`, `Notes`
- Model baru: `BackgroundCheck` dengan field `JobApplicationId`, `CheckType` (Reference, Criminal, Education, Employment), `Provider`, `Status`, `Result`, `CompletedAt`
- DTO baru: `DocumentVerificationDTO`, `BackgroundCheckDTO`, `VerificationSummaryDTO`
- Service baru: `IVerificationService` / `VerificationService`
- Endpoint baru pada `HRController`: `ManageDocumentVerification`, `InitiateBackgroundCheck`, `VerificationSummary`
- Endpoint baru pada `ApplicantController`: `UploadVerificationDocument`, `VerificationStatus`

---

### [16.3] Duplicate Applicant Detection dan Merge

**Status saat ini:** Tidak ada mekanisme untuk mendeteksi kandidat duplikat. Seorang kandidat bisa mendaftar dengan email berbeda dan memiliki beberapa profil `Applicant` yang terpisah, menyebabkan data tidak konsisten.

**Fitur baru:**

- Auto-detection kandidat duplikat berdasarkan nama, email, nomor telepon, atau kombinasi data
- Flagging otomatis saat registrasi jika terdeteksi potensial duplikat
- Merge tool: HR dapat menggabungkan dua profil kandidat menjadi satu (memilih data yang dipertahankan)
- Riwayat lamaran dari semua profil yang di-merge tetap terjaga

**Implementasi:**

- Model baru: `DuplicateDetectionLog` dengan field `PrimaryApplicantId`, `DuplicateApplicantId`, `MatchScore`, `MatchCriteria` (Email, Phone, Name), `Status` (Pending, Merged, Dismissed), `ResolvedBy`
- DTO baru: `DuplicateDetectionDTO`, `MergeApplicantDTO`, `MergPreviewDTO`
- Service baru: `IDuplicateDetectionService` / `DuplicateDetectionService`
- Modifikasi registrasi Applicant: integrasi duplicate check saat sign-up
- Endpoint baru pada `AdminController` dan `HRController`: `DuplicateApplicants`, `PreviewMerge`, `MergeApplicants`, `DismissDuplicate`

---

## 17. Multi-Bahasa dan Aksesibilitas

### [17.1] Multi-Language Support untuk Portal Karir

**Status saat ini:** Seluruh antarmuka aplikasi (views, email templates, notifikasi) menggunakan satu bahasa saja. Tidak ada mekanisme internationalization (i18n) atau localization (l10n) untuk mendukung kandidat dari berbagai negara.

**Fitur baru:**

- Dukungan multi-bahasa pada portal karir publik (Bahasa Indonesia, English, minimal)
- Job posting yang dapat dibuat dalam beberapa bahasa
- Email template multi-bahasa: otomatis pilih bahasa berdasarkan preferensi kandidat
- Language switcher di portal karir
- Localization konten dinamis (notifikasi, pesan status, guidance text)

**Implementasi:**

- Model baru: `Language` dengan field `LanguageCode`, `LanguageName`, `IsDefault`, `IsActive`
- Model baru: `TranslationResource` dengan field `ResourceKey`, `LanguageCode`, `Value`
- Model baru: `JobTranslation` dengan field `JobId`, `LanguageCode`, `TranslatedTitle`, `TranslatedDescription`, `TranslatedRequirement`
- Modifikasi `Applicant`: tambah field `PreferredLanguage`
- DTO baru: `LanguageDTO`, `TranslationResourceDTO`, `JobTranslationDTO`
- Service baru: `ILocalizationService` / `LocalizationService`
- Modifikasi `IEmailService`: pilih template bahasa berdasarkan `PreferredLanguage` kandidat
- Endpoint baru pada `ConfigureController`: `ManageLanguages`, `ManageTranslations`
- Implementasi ASP.NET Core Resource localization middleware

---

### [17.2] Aksesibilitas (WCAG Compliance) pada Portal Karir

**Status saat ini:** Portal karir belum dioptimasi untuk aksesibilitas. Kandidat dengan disabilitas visual, motorik, atau kognitif mungkin mengalami kesulitan menggunakan aplikasi.

**Fitur baru:**

- Audit dan perbaikan aksesibilitas sesuai standar WCAG 2.1 Level AA
- Screen reader compatibility: semua elemen interaktif memiliki proper ARIA labels
- Keyboard navigation penuh: seluruh fitur dapat diakses tanpa mouse
- High contrast mode dan font size adjustment untuk kandidat dengan low vision
- Form accessibility: proper error messaging, label association, dan focus management

**Implementasi:**

- Modifikasi seluruh Views/Layouts: tambah ARIA roles, labels, dan live regions
- Modifikasi CSS: tambah high contrast theme dan responsive font sizing
- JavaScript utility baru: `accessibility.js` untuk keyboard trap management, focus management, dan screen reader announcements
- Modifikasi form validation: update semua form untuk menggunakan `aria-describedby` dan `aria-invalid`
- Testing: integrasi axe-core untuk automated accessibility testing
- Endpoint baru pada `HomeController`: `AccessibilityStatement`
- Library tambahan: axe-core untuk automated WCAG compliance testing

---

## 18. API Layer dan Integrasi Sistem Eksternal

### [18.1] RESTful API Layer untuk Integrasi Pihak Ketiga

**Status saat ini:** Seluruh endpoint menggunakan MVC pattern yang mengembalikan Views (HTML). Beberapa endpoint API (`GetAll`, `GetAllAuditTrails`) tersebar di controller yang berbeda tanpa konsistensi format atau dokumentasi. Tidak ada dedicated API layer untuk integrasi dengan sistem eksternal.

**Fitur baru:**

- Dedicated API controller layer (`/api/v1/`) dengan format response JSON yang konsisten
- API versioning untuk backward compatibility
- Swagger/OpenAPI documentation otomatis untuk semua endpoint
- API key authentication untuk integrasi machine-to-machine
- Rate limiting untuk mencegah abuse

**Implementasi:**

- Controller baru: `ApiJobsController`, `ApiApplicantsController`, `ApiRecruitmentController`, `ApiAnalyticsController` di namespace `Controllers/Api/V1`
- Model baru: `ApiKey` dengan field `Key`, `ClientName`, `Permissions`, `IsActive`, `CreatedAt`, `ExpiresAt`, `RateLimit`
- DTO baru: `ApiResponseDTO<T>` dengan field `Success`, `Data`, `Error`, `Pagination`
- Service baru: `IApiKeyService` / `ApiKeyService`
- Middleware baru: `ApiKeyAuthenticationMiddleware`, `RateLimitingMiddleware`
- Endpoint baru pada `ConfigureController`: `ManageApiKeys`, `ApiKeyUsageReport`
- Library tambahan: Swashbuckle.AspNetCore untuk Swagger, AspNetCoreRateLimit untuk rate limiting

---

### [18.2] Webhook System untuk Event-Driven Integration

**Status saat ini:** Tidak ada mekanisme untuk memberi tahu sistem eksternal ketika terjadi event penting dalam proses rekrutmen (misalnya: kandidat baru apply, status berubah, offering dikirim).

**Fitur baru:**

- Konfigurasi webhook endpoint untuk event tertentu (ApplicationReceived, StatusChanged, OfferSent, CandidateAccepted, dll.)
- Retry mechanism dengan exponential backoff untuk webhook delivery yang gagal
- Webhook delivery log untuk debugging
- Signature verification (HMAC) untuk keamanan webhook
- Test webhook: kemampuan mengirim test payload ke endpoint yang dikonfigurasi

**Implementasi:**

- Model baru: `WebhookSubscription` dengan field `Url`, `Events` (JSON array), `SecretKey`, `IsActive`, `CreatedBy`, `Headers` (JSON)
- Model baru: `WebhookDeliveryLog` dengan field `SubscriptionId`, `Event`, `Payload`, `ResponseCode`, `ResponseBody`, `DeliveredAt`, `AttemptCount`, `Status` (Success, Failed, Retrying)
- DTO baru: `WebhookSubscriptionDTO`, `WebhookDeliveryLogDTO`, `WebhookTestDTO`
- Service baru: `IWebhookService` / `WebhookService`
- Background job: `WebhookDeliveryJob` untuk async delivery dan retry
- Modifikasi seluruh service yang trigger event (JobService, RecruitmentProcessService, ApplicantService): dispatch webhook event
- Endpoint baru pada `ConfigureController`: `ManageWebhooks`, `WebhookDeliveryLogs`, `TestWebhook`

---

### [18.3] HRIS Integration Bridge

**Status saat ini:** Setelah kandidat diterima (`ProcessType.Accepted`), tidak ada mekanisme otomatis untuk mentransfer data ke sistem HRIS (Human Resource Information System) perusahaan. Data harus dipindahkan secara manual.

**Fitur baru:**

- Export data kandidat yang diterima dalam format yang compatible dengan HRIS populer
- Auto-sync data ke HRIS ketika kandidat berstatus Accepted dan semua dokumen terverifikasi
- Mapping field antara RecruitmentTracking dan target HRIS yang dapat dikonfigurasi
- Status tracking: apakah data sudah berhasil disinkronkan ke HRIS atau gagal
- Support multiple HRIS integration (SAP SuccessFactors, Oracle HCM, BambooHR, custom)

**Implementasi:**

- Model baru: `HRISIntegration` dengan field `SystemName`, `ApiEndpoint`, `AuthConfig` (encrypted JSON), `FieldMapping` (JSON), `IsActive`
- Model baru: `HRISSyncLog` dengan field `IntegrationId`, `JobApplicationId`, `SyncStatus` (Pending, InProgress, Success, Failed), `ErrorMessage`, `SyncedAt`, `SyncedData` (JSON)
- DTO baru: `HRISIntegrationDTO`, `FieldMappingDTO`, `HRISSyncLogDTO`, `HRISExportDTO`
- Service baru: `IHRISIntegrationService` / `HRISIntegrationService`
- Modifikasi `IRecruitmentProcessService.AcceptApplicantAsync`: trigger HRIS sync setelah acceptance
- Endpoint baru pada `AdminController`: `ConfigureHRISIntegration`, `SyncToHRIS`, `HRISSyncStatus`, `HRISFieldMapping`
- Library tambahan: HttpClientFactory untuk API calls ke target HRIS

---

## 19. Internal Mobility dan Talent Marketplace

### [19.1] Portal Lowongan Internal (Internal Job Board)

**Status saat ini:** Semua lowongan kerja bersifat publik dan dapat dilamar oleh siapa saja dari luar perusahaan (`IsDraft: false, IsOpen: true`). Tidak ada pemisahan antara lowongan untuk kandidat eksternal dan lowongan khusus untuk promosi atau mutasi karyawan internal.

**Fitur baru:**

- Tipe lowongan `InternalOnly` yang hanya bisa dilihat dan dilamar oleh `EndUserStaff` atau `HrStaff` yang sudah otentikasi
- Notifikasi prioritas ke karyawan internal ketika ada lowongan departemen terkait
- Workflow persetujuan manajer saat ini (current manager approval) sebelum karyawan internal bisa melamar lowongan di departemen lain
- Profil karyawan internal terhubung otomatis dengan data HRIS, sehingga tidak perlu upload CV dari awal

**Implementasi:**

- Modifikasi `Job`: tambah field `JobVisibility` (Public, Internal, Confidential)
- Model baru: `InternalApplicationApproval` dengan field `JobApplicationId`, `CurrentManagerId`, `ApprovalStatus`, `Comments`, `ApprovedAt`
- Modifikasi `ApplicantService`: tambah validasi khusus untuk lowongan internal
- Endpoint baru pada `HomeController`: `InternalJobsBoard`
- Endpoint baru pada `EndUserController`: `ApproveInternalTransfer`

---

### [19.2] Sistem Alumni (Corporate Alumni Network)

**Status saat ini:** Kandidat yang lolos (`ProcessType.Accepted`) akan berhenti terlacak di sistem rekrutmen. Demikian juga mantan karyawan yang resign (alumni) tidak memiliki wadah di sistem untuk re-hire (boomerang employees).

**Fitur baru:**

- Talent pool khusus tipe "Alumni" untuk mantan karyawan yang berkinerja baik
- Portal alumni dengan akses terbatas untuk melihat lowongan prioritas
- Kampanye "Boomerang Employee": email otomatis ke alumni di ulang tahun masa kerja mereka dengan tawaran posisi senior
- Pelacakan sumber "Re-hire" untuk metrik rekrutmen

**Implementasi:**

- Modifikasi `TalentPool`: tambah field `PoolType` (General, Alumni, SilverMedalist)
- Model baru: `AlumniProfile` terhubung dengan `AppUser`
- DTO baru: `AlumniTalentDTO`, `BoomerangCampaignDTO`
- Service baru: `IAlumniNetworkService` / `AlumniNetworkService`
- Endpoint baru pada `HRController`: `AlumniDirectory`, `LaunchBoomerangCampaign`

---

### [19.3] Silver Medalist Auto-Matching

**Status saat ini:** Kandidat yang sampai ke tahap akhir (Offering/UserInterview) tapi tidak terpilih, biasanya masuk ke `TalentPool` secara pasif. Tidak ada matching otomatis ketika posisi serupa dibuka di masa depan.

**Fitur baru:**

- Auto-tag "Silver Medalist" untuk kandidat yang lolos hingga UserInterview tapi berstatus Rejected/Pool karena kalah bersaing tipis
- Ketika posisi serupa dibuka, sistem otomatis memunculkan pop-up ke HR dengan daftar Silver Medalists tersebut
- Fast-track pipeline: kemampuan mengundang Silver Medalist langsung ke tahap User Interview melewati tahap Administrasi dan HR Interview

**Implementasi:**

- Modifikasi `JobApplication`: tambah flag `IsSilverMedalist`
- Modifikasi `ITalentMatchingService`: tambah method `GetSilverMedalistsForJobAsync`
- Modifikasi `CreateJobAsync`: trigger cek Silver Medalist saat lowongan baru di-publish
- Endpoint baru pada `HRController`: `FastTrackCandidate` yang memodifikasi status langsung ke `ProcessType.UserInterview`

---

## 20. Advanced Offer Management dan Negosiasi

### [20.1] Compensation Package Calculator

**Status saat ini:** Field `Salary` hanya nominal integer/double. Tidak ada kalkulator untuk menghitung total compensation package (Base, Allowance, Bonus, Stock Options) yang sering menjadi poin negosiasi di tahap Offering.

**Fitur baru:**

- Kalkulator kompensasi interaktif di halaman HR saat membuat penawaran
- Visualisasi "Total Rewards" (Chart breakdown Base vs Benefits) yang bisa disertakan di Offer Letter
- Kalkulasi PPh 21 dan Take Home Pay estimasi untuk meyakinkan kandidat
- Versioning untuk setiap revisi penawaran (Offer V1, Offer V2)

**Implementasi:**

- Model baru: `CompensationPackage` dengan field `JobApplicationId`, `BaseSalary`, `TransportAllowance`, `MealAllowance`, `HealthInsuranceValue`, `AnnualBonusTarget`, `TotalAnnualValue`, `VersionNumber`
- DTO baru: `CompensationBreakdownDTO`, `GenerateTotalRewardsDTO`
- Service baru: `ICompensationCalculatorService` / `CompensationCalculatorService`
- Endpoint baru pada `HRController`: `CalculateCompensation`, `ViewOfferVersions`

---

### [20.2] Approval Chain Dinamis untuk Offering di Luar Budget

**Status saat ini:** Penawaran (Offering) bisa dikirim langsung tanpa persetujuan finansial, terlepas dari apakah `Salary` yang ditawarkan melebihi budget departemen.

**Fitur baru:**

- Validasi penawaran terhadap `SalaryRangeMax` dari lowongan terkait
- Jika penawaran berada di atas range, otomatis trigger approval flow ke Department Head dan Finance Director
- Notifikasi Slack/Teams bot ke approver untuk slow-approval prevention
- Penawaran terkunci (tidak bisa dikirim ke kandidat) sebelum persetujuan final diberikan

**Implementasi:**

- Model baru: `ExceptionApproval` dengan field `EntityId`, `ExceptionType` (OverBudgetOffer), `RequestedBy`, `RequiredApprovers` (JSON list), `Status`
- Modifikasi `HRController.SendOffering`: cegah pengiriman jika status approval `Pending`
- Service baru: `IOfferApprovalService` / `OfferApprovalService`
- Endpoint API internal untul webhook Slack/Teams integration

---

### [20.3] Expiry dan Deadline Management pada Offer

**Status saat ini:** Status `Offering` bersifat statis. Kandidat bisa menggantung penawaran selama berminggu-minggu tanpa sistem memberikan peringatan.

**Fitur baru:**

- Configurable Offer Deadline (misalnya: penawaran hangus dalam 3 hari)
- Countdown timer di sisi portal kandidat "Offer expires in: 48h 12m"
- Auto-revoke status menjadi `Rejected` atau `Expired` ketika waktu habis
- Auto-email H-1 sebelum penawaran hangus sebagai reminder (Urgency mechanism)

**Implementasi:**

- Modifikasi `JobApplication`: tambah field `OfferSentAt`, `OfferExpiresAt`
- Background job `OfferExpiryMonitorJob` berjalan setiap jam menggunakan `IHostedService`
- Modifikasi `ApplicantController.TrackJob`: tampilkan countdown jika status = Offering
- Endpoint `RevokeOffer` pada `HRController` untuk revokasi manual

---

## 21. Candidate Background Services Pihak Ketiga

### [21.1] Integrasi Cek Kriminal dan Finansial (Slik OJK/BI Checking)

**Status saat ini:** Tidak tercantum integrasi untuk memverifikasi latar belakang legal/finansial kandidat, yang sering kali krusial untuk posisi perbankan atau finance.

**Fitur baru:**

- API trigger ke vendor background checking lokal (misal: AsliRI, Verihubs, atau lembaga terkait)
- Dashboard consent: kandidat menyetujui secara digital agar datanya digunakan untuk pengecekan
- Laporan background check berformat PDF yang otomatis terlampir di profil institusi

**Implementasi:**

- Model baru: `BackgroundCheckConsent` dengan field `ApplicantId`, `CheckType`, `IPAddress`, `ConsentDate`, `DigitalSignature`
- Service API caller: `IVendorBackgroundCheckService` khusus untuk vendor Indonesia
- Modifikasi UI HR: tombol "Initiate BI Checking" yang hanya muncul jika consent = true

---

### [21.2] Verifikasi Identitas menggunakan Liveness Detection (KYC)

**Status saat ini:** Sistem tidak memverifikasi apakah kandidat yang melamar dan submit CV adalah orang asli (anti-fraud).

**Fitur baru:**

- Modul e-KYC saat tahap awal interview: scan KTP dan foto selfie wajah (Liveness Check)
- Integrasi dengan AI Face Match provider untuk mencocokkan wajah saat online test/assessment dengan e-KYC ID
- Flagging akun fake atau joki (fraud detection)

**Implementasi:**

- Model baru: `ApplicantIdentity` dengan field `KtpNumber`, `KtpImageUrl`, `FaceImageUrl`, `LivenessScore`, `MatchScore`, `VerificationStatus`
- DTO baru: `KycSubmissionDTO`, `FaceMatchResultDTO`
- Integrasi vendor OCR dan Face Recognition (contoh: AWS Rekognition, Azure Face API, atau vendor ID verification)
- Endpoint `SubmitKyc` di `ApplicantController`

---

## 22. Machine Learning Analytics (Advanced)

### [22.1] Prediksi Turnover / Retention Kandidat

**Status saat ini:** Rekrutmen hanya berfokus sampai tahap `Accepted`. Tidak ada analisis apakah kandidat yang di-hire tersebut akan bertahan lama (retention) berdasarkan profil rekrutmennya.

**Fitur baru:**

- Model ML yang memprediksi resiko churn/turnover (High, Medium, Low Risk) berdasarkan data demografis historis, skor ATS, commute distance (lokasi kandidat vs kantor), dan riwayat pekerjaan di CV.
- "Flight Risk" indicator di dashboard HR saat me-review pelamar untuk posisi strategis
- Insight pengurang resiko (misal: "Kandidat beresiko tinggi karena jarak tempuh > 30km, pertimbangkan opsi Work From Home")

**Implementasi:**

- Service `IRetentionPredictionService` menggunakan model ML.NET tersendiri
- Algoritma menghitung `CommuteScore` membandingkan koordinat alamat rumah (jika ada) via Google Maps API Service
- DTO `RetentionRiskDTO` terlampir di halaman profil pelamar khusus view HR/Admin

---

### [22.2] Bias Detection Engine (DEI Enforcer)

**Status saat ini:** Human Resource (HRStaff/EndUserStaff) dapat mem-filter kandidat secara manual (Sourced, Rejected) tanpa diaudit terkait kemungkinan bias implisit (gender bias, age bias, university bias).

**Fitur baru:**

- Sistem AI berjalan di background memeriksa rasio penolakan. Jika interviewer A selalu menolak kandidat dari universitas tertentu atau demografi tertentu dengan nilai ATS tinggi, sistem akan memberikan flag anomali
- Dashboard DEI (Diversity, Equity, Inclusion) Health Score per recruiter
- Fitur "Blind Recruitment Mode": Menyembunyikan nama, foto, umur, dan nama institusi pendidikan saat screening tahap Administrasi, hanya menampilkan skill dan skor ATS.

**Implementasi:**

- Mode toggle di aplikasi: `IsBlindScreeningEnabled` pada model `Job`
- Service `IBiasAnalyticsService` memproses data log `AuditTrail` dan `JobApplication`
- Anonymizer Helper di Controller untuk mem-masking data pada `ApplicantProfileDTO` ketika mode Blind aktif.

---

## 23. Pengumpulan Feedback dan NPS

### [23.1] Candidate Experience NPS (Net Promoter Score)

**Status saat ini:** Tidak ada fitur untuk menanyakan pendapat kandidat tentang proses rekrutmen perusahaan.

**Fitur baru:**

- Automated Survey Email dikirim ke kandidat setelah tahapan akhir mereka (baik diterima maupun ditolak)
- Pertanyaan rating 1-10: "Seberapa transparan proses komunikasi kami?" atau "Seberapa profesional interviewer kami?"
- Analisis sentimen terhadap komentar teks kandidat menggunakan AI
- Dashboard NPS per departemen atau per recruiter untuk KPI

**Implementasi:**

- Model baru: `CandidateFeedback` dengan field `JobApplicationId`, `Rating`, `Comments`, `SentimentScore` (Positive/Negative/Neutral), `SubmittedAt`
- Trigger di `IRecruitmentProcessService` saat status berubah menjadi Closed, Rejected, Accepted
- Dashboard khusus `CandidateNPSResults` di `AdminController`

---

### [23.2] Hiring Manager Satisfaction (HM-SAT)

**Status saat ini:** Tidak ada wadah untuk EndUserStaff (User) memberikan rating kepada HR terkait kualitas kandidat yang diberikan.

**Fitur baru:**

- Pop-up otomatis di dashboard aplikasi EndUser 30 hari setelah kandidat mulai bekerja: "Apakah kandidat [Nama] memenuhi ekspektasi Anda?"
- Metrik "Quality of Hire" yang menjadi KPI utama tim HR Recruitment
- Loopback mechanism: jika user memberi score rendah, algoritma ATS akan menyesuaikan bobot scoring untuk lowongan berikutnya dari departemen tersebut.

**Implementasi:**

- Model `QualityOfHireScore` dengan field `ApplicantId`, `EvaluatorId` (EndUserStaff), `Score`, `FeedbackDetails`
- Background Worker `QualityOfHireSurveyJob` untuk trigger 30 hari pasca hiring
- Endpoint di `EndUserController` untuk submit rating

---

## 24. Modul Rekrutmen Kampus & Event Management

### [24.1] Campus Hiring & Job Fair Kiosk Mode

**Status saat ini:** Proses lamaran mengharuskan buka web desktop/mobile dan sign-up panjang. Ini tidak praktis untuk skenario massal seperti Job Fair atau kunjungan kampus.

**Fitur baru:**

- "Kiosk Mode" UI: Tampilan web tablet-friendly yang ringkas. Tanpa password, hanya Quick Apply menggunakan Scan QR Code profil LinkedIn atau upload PDF cepat.
- Event tracking: Lowongan spesifik dilabeli dengan `EventId` (Misal: UI Career Fair 2026)
- Analisis ROI per event: Berapa banyak hire sukses yang bersumber dari spesifik job fair tersebut.

**Implementasi:**

- Model baru: `RecruitmentEvent` dengan field `EventName`, `Location`, `StartDate`, `EndDate`, `CostBudget`
- Modifikasi `Job`: relation opsional `EventId`
- Controller khusus `KioskController` dengan DTO mini `QuickApplyDTO` (Email, Phone, CV Upload) tanpa registrasi rumit
- Fitur auto-create `AppUser` di background dengan password generate otomatis yang diemailkan.

---

### [24.2] Walk-In Interview Queue Management

**Status saat ini:** `ScheduleService` lebih cocok untuk interview terjadwal (online/offline). Tidak mendistribusikan traffic "Walk-In" (kandidat datang kapan saja dalam jendela waktu hari itu).

**Fitur baru:**

- Modul Antrean (Queue) Digital. Pelamar scan QR Code di lobi kantor untuk mengambil nomor antrean.
- Monitor dashboard (Live Screen) menampilkan antrean saat ini yang sedang dipanggil masuk ke ruangan interviewer.
- Automatic routing: Kandidat A di-assign secara round-robin ke Interviewer 1, 2, atau 3 yang sedang kosong (tersedia).

**Implementasi:**

- Model baru: `WalkInSession` (mirip Job application tapi transient)
- SignalR Hub `QueueMonitorHub` untuk sinkronisasi Live Screen TV di Lobby dengan dashboard HR
- Tombol "Next Candidate" pada dashboard HR/Interviewer yang otomatis mengubah status antrean dan membunyikan notifikasi audio.
- Integrasi Controller: `QueueController` untuk mengelola flow walk-in ini secara independen dari proses ATS panjang.
