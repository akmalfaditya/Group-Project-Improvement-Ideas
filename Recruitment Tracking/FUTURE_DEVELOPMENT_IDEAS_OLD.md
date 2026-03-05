# Future Development — RecruitmentTracking


---

## Daftar Isi

1. [AI & Machine Learning Enhancements](#1-ai--machine-learning-enhancements)
2. [Recruitment Pipeline Improvements](#2-recruitment-pipeline-improvements)
3. [Advanced Search & Discovery](#3-advanced-search--discovery)
4. [Analytics, Reporting & Dashboard](#4-analytics-reporting--dashboard)
5. [Interview & Scheduling](#5-interview--scheduling)
6. [Communication & Notification](#6-communication--notification)
7. [Applicant Experience](#7-applicant-experience)
8. [Talent Pool & CRM](#8-talent-pool--crm)
9. [Integration & Interoperability](#9-integration--interoperability)
10. [Security & Compliance](#10-security--compliance)
11. [Administration & Configuration](#11-administration--configuration)
12. [Performance & Scalability](#12-performance--scalability)
13. [Mobile & Responsive](#13-mobile--responsive)
14. [Export, Import & Data Management](#14-export-import--data-management)
15. [Collaboration & Workflow](#15-collaboration--workflow)
16. [Employer Branding & Career Page](#16-employer-branding--career-page)
17. [Onboarding Integration](#17-onboarding-integration)
18. [Gamification & Engagement](#18-gamification--engagement)
19. [Accessibility & Internationalization](#19-accessibility--internationalization)
20. [DevOps & Infrastructure](#20-devops--infrastructure)

---

## 1. AI & Machine Learning Enhancements

### Konteks Saat Ini
Sistem sudah memiliki AI ATS scoring via DeepSeek API (`AiAtsService`), AI job recommendations (`JobRecommendationService`), talent matching (`TalentMatchingService` — keyword-based), dan CV text extraction via Tesseract OCR (`PdfProcessingService`).

### Fitur yang Bisa Dikembangkan

#### 1.1 — AI Resume Parser yang Lebih Cerdas
**Deskripsi:** Saat ini `PdfProcessingService` mengekstrak teks CV dengan regex rigid yang hanya mengenali heading tertentu (SKILLS, EXPERIENCE, EDUCATION, dll.). Parser sering gagal pada CV dengan format non-standar.

**Ide Pengembangan:**
- Gunakan AI (LLM) untuk parsing CV secara semantik, bukan regex — kirim teks mentah ke AI dan minta structured output (JSON) untuk nama, kontak, pendidikan, pengalaman, skill, sertifikasi
- Support parsing CV dalam format **DOCX**, **RTF**, dan **plain text** selain PDF
- Deteksi bahasa otomatis (saat ini hanya `eng+ind`) dan tambahkan dukungan multi-bahasa OCR
- Parsing link LinkedIn/GitHub/Portfolio dari CV
- Ekstraksi **years of experience** secara otomatis
- Parsing **expected salary range** dari CV jika tersedia
- Confidence score untuk setiap field yang diekstrak

**Estimasi Kompleksitas:** Medium-High

---

#### 1.2 — AI Skill Taxonomy & Normalization
**Deskripsi:** Saat ini skill disimpan sebagai semicolon-separated string (`AtsMatchedSkills`, `AtsMissingSkills`, `CachedSkills`). Tidak ada normalisasi — "React.js", "ReactJS", "React" dianggap skill berbeda.

**Ide Pengembangan:**
- Buat **Skill Taxonomy Database** — master list of skills dengan synonym mapping (React.js = ReactJS = React)
- AI-powered auto-categorization: Programming Languages, Frameworks, Soft Skills, Tools, Certifications
- Skill proficiency level detection (Beginner/Intermediate/Advanced/Expert)
- **Skill gap analysis** — bandingkan skill applicant vs. job requirement, tampilkan gap secara visual
- Skill trending analytics — skill mana yang paling banyak dicari vs. paling sedikit dimiliki applicant
- Auto-suggest related skills ("Jika menguasai React, mungkin juga perlu TypeScript, Redux")

**Estimasi Kompleksitas:** High

---

#### 1.3 — AI-Powered Interview Question Generator
**Deskripsi:** Saat ini tidak ada fitur untuk membantu interviewer menyiapkan pertanyaan wawancara.

**Ide Pengembangan:**
- Generate pertanyaan interview berdasarkan job description + CV applicant
- Kategorisasi pertanyaan: Technical, Behavioral, Situational, Culture Fit
- Difficulty level per pertanyaan (Easy/Medium/Hard)
- Suggested follow-up questions berdasarkan jawaban
- Question bank yang bisa di-save dan di-reuse per department/role
- Include rubrik penilaian (scoring criteria) untuk setiap pertanyaan

**Estimasi Kompleksitas:** Medium

---

#### 1.4 — AI Candidate Ranking & Scoring Dashboard
**Deskripsi:** Saat ini ATS score (0-10) sudah ada per applicant, tapi tidak ada ranking komparatif atau scoring dashboard.

**Ide Pengembangan:**
- **Ranked applicant list** per job — sort by composite score (ATS score + interview notes + experience years)
- **Radar chart / spider diagram** per applicant — visual perbandingan skill match
- **Comparative view** — side-by-side comparison 2-3 applicant
- Weighted scoring — HR bisa set bobot per kriteria (skill 40%, education 20%, experience 30%, culture 10%)
- AI-generated **hiring recommendation** — "Strongly Recommend", "Recommend", "Consider", "Not Recommended"
- Score trend tracking across pipeline stages

**Estimasi Kompleksitas:** Medium-High

---

#### 1.5 — AI Job Description Generator & Optimizer
**Deskripsi:** Saat ini HR/Admin menulis job description secara manual di `CreateJob`.

**Ide Pengembangan:**
- AI auto-generate job description dari input minimal (job title + department + beberapa keyword)
- **Bias detector** — scan job description untuk gender-biased, age-biased, atau exclusionary language
- **Readability score** — pastikan JD mudah dipahami (Flesch-Kincaid score)
- SEO optimization — suggest keyword untuk meningkatkan visibility di job board
- Benchmark comparison — bandingkan JD dengan similar positions di industri
- Template library — simpan JD templates per department/role yang bisa direuse

**Estimasi Kompleksitas:** Medium

---

#### 1.6 — AI Chatbot untuk Applicant
**Deskripsi:** Saat ini tidak ada fitur chatbot. Applicant harus navigasi manual untuk informasi.

**Ide Pengembangan:**
- **FAQ Chatbot** — jawab pertanyaan umum tentang proses rekrutmen, status lamaran, benefit perusahaan
- **Application Assistant** — bantu applicant mengisi form lamaran step-by-step
- **Job Finder Chatbot** — "Saya software engineer dengan 3 tahun pengalaman React, job apa yang cocok?"
- **Interview Prep Bot** — berikan tips persiapan interview berdasarkan job yang dilamar
- **Status Checker** — "Apa status lamaran saya untuk posisi X?"
- Integrasi dengan WhatsApp / Telegram untuk reach yang lebih luas

**Estimasi Kompleksitas:** High

---

#### 1.7 — Predictive Analytics & Hiring Intelligence
**Deskripsi:** Tidak ada predictive analytics di sistem saat ini.

**Ide Pengembangan:**
- **Time-to-hire prediction** — prediksi berapa lama proses rekrutmen akan berlangsung per job/department
- **Offer acceptance probability** — prediksi kemungkinan applicant menerima offering berdasarkan salary gap, market data
- **Candidate dropout prediction** — identifikasi applicant yang kemungkinan besar withdraw
- **Pipeline bottleneck detection** — AI identifikasi tahap mana yang paling lambat dan mengapa
- **Hiring success prediction** — berdasarkan historical data, prediksi quality of hire
- **Seasonal hiring patterns** — kapan waktu terbaik untuk posting job tertentu

**Estimasi Kompleksitas:** Very High

---

#### 1.8 — AI-Powered Email Content Generation
**Deskripsi:** Saat ini email template sudah ada dalam `EmailTemplate` entity (rejection, HR interview, EndUser interview, offering), tapi kontennya ditulis manual.

**Ide Pengembangan:**
- AI generate email body yang personalized per applicant (menggunakan nama, posisi, detail interview)
- Tone selector — Formal, Semi-formal, Friendly
- Multi-language email generation otomatis
- A/B testing email subject lines untuk engagement rate terbaik
- Auto-suggest optimal send time berdasarkan timezone applicant

**Estimasi Kompleksitas:** Medium

---

#### 1.9 — Semantic Job Recommendation Enhancement
**Deskripsi:** Saat ini `JobRecommendationService` hanya menggunakan `Major` dan `Education` level untuk recommendation. Mengirim seluruh job list ke AI per request.

**Ide Pengembangan:**
- Gunakan **full CV content** (skills, experience, education) bukan hanya Major
- **Embedding-based matching** — konversi CV dan job description ke vector embeddings, hitung cosine similarity
- **Collaborative filtering** — "Applicant yang melamar job X juga melamar job Y"
- **Location-based recommendation** — pertimbangkan preferensi lokasi applicant
- **Salary-aware recommendation** — filter job berdasarkan expected salary range
- **Real-time recommendation update** — push notification ketika ada job baru yang cocok
- Pre-filter jobs di server sebelum kirim ke AI untuk mengurangi token cost

**Estimasi Kompleksitas:** High

---

#### 1.10 — AI Interview Sentiment Analysis
**Deskripsi:** Saat ini interview notes (`HrInterviewNotes`, `EndUserInterviewNotes`) hanya plain text tanpa analysis.

**Ide Pengembangan:**
- Analisis sentiment dari interview notes (positif/negatif/netral)
- Extract key points dan concern otomatis dari notes
- Detect red flags (e.g., "not a team player", "salary too high")
- Summarisasi interview notes panjang menjadi bullet points
- Scoring berdasarkan konteks notes

**Estimasi Kompleksitas:** Medium

---

## 2. Recruitment Pipeline Improvements

### Konteks Saat Ini
Pipeline 5 tahap: Administration → HR Interview → EndUser Interview → Offering → Accepted/Rejected. Semua tahap fix dan tidak bisa dikustomisasi.

---

#### 2.1 — Customizable Pipeline Stages
**Deskripsi:** Pipeline saat ini hardcoded 5 tahap. Beberapa job mungkin memerlukan technical test, atau assessment center, atau multiple interview rounds.

**Ide Pengembangan:**
- Admin bisa **define custom pipeline stages** per job atau per department
- Drag-and-drop pipeline stage editor
- Support stages: Screening, Phone Interview, Technical Test, Take-Home Assignment, Panel Interview, Reference Check, Background Check, Offering, Onboarding
- Each stage bisa memiliki custom form/checklist
- Conditional branching — jika gagal di stage X, bisa redirect ke stage Y (bukan langsung reject)
- Optional stages — beberapa stage bisa di-skip

**Estimasi Kompleksitas:** Very High

---

#### 2.2 — Kanban Board untuk Recruitment Pipeline
**Deskripsi:** Saat ini pipeline ditampilkan per-stage di view terpisah (Administration.cshtml, HRInterview.cshtml, dll.).

**Ide Pengembangan:**
- **Kanban board visual** — drag-and-drop applicant card antar stage
- Real-time update (pakai SignalR yang sudah ada)
- Quick actions dari card: view CV, add notes, schedule interview, accept/reject
- Color coding berdasarkan urgency (days in stage, ATS score)
- Filter kanban by department, location, date range
- Mini applicant profile preview on hover

**Estimasi Kompleksitas:** Medium-High

---

#### 2.3 — Automated Stage Transitions & Triggers
**Deskripsi:** Saat ini semua transisi stage dilakukan manual oleh HR/Admin.

**Ide Pengembangan:**
- **Auto-advance** — setelah interview ter-schedule + notes diisi, otomatis move ke stage berikutnya
- **Auto-reject** — applicant dengan ATS score < threshold otomatis ditolak
- **Deadline-based** — jika applicant tidak merespons dalam X hari, auto-reject/auto-remind
- **Bulk auto-screening** — filter applicant berdasarkan minimum education, experience years, skill match
- **Configurable rules engine** — Admin set rules per job (e.g., "ATS < 5 → auto reject", "ATS ≥ 8 → fast-track to HR interview")
- Email/notification trigger otomatis saat stage berubah

**Estimasi Kompleksitas:** High

---

#### 2.4 — Assessment & Technical Test Integration
**Deskripsi:** Tidak ada fitur assessment atau technical test di sistem saat ini.

**Ide Pengembangan:**
- Built-in **quiz/assessment builder** untuk technical screening
- Support question types: Multiple Choice, Code Challenge, Essay, File Upload
- Timer-based assessment dengan time tracking
- Auto-grading untuk Multiple Choice dan simple code challenges
- Integration dengan external assessment platforms (HackerRank, Codility, TestGorilla)
- Score otomatis masuk ke applicant profile di pipeline

**Estimasi Kompleksitas:** Very High

---

#### 2.5 — Reference Check Module
**Deskripsi:** Tidak ada fitur reference check.

**Ide Pengembangan:**
- Form untuk input referensi (nama, hubungan, email, telepon)
- Automated reference check email — kirim questionnaire ke reference
- Online reference form yang bisa diisi oleh reference secara anonim
- Scoring dari reference responses
- Status tracking: Pending, Completed, Flagged

**Estimasi Kompleksitas:** Medium

---

#### 2.6 — Background Check Integration
**Deskripsi:** Tidak ada fitur background check.

**Ide Pengembangan:**
- Integration dengan background check providers (Sterling, Checkr, First Advantage)
- Track status: Initiated, In Progress, Clear, Flagged
- Automated consent form generation
- Document upload untuk background check results
- Flag system untuk highlighting issues

**Estimasi Kompleksitas:** Medium

---

#### 2.7 — Multi-Job Application Management
**Deskripsi:** Applicant bisa melamar ke beberapa job, dan `AutoCloseOtherApplicationsAsync` otomatis menutup lamaran lain saat satu accepted.

**Ide Pengembangan:**
- **Cross-job applicant view** — lihat semua posisi yang dilamar seorang applicant sekaligus
- **Conflict detection** — alert jika applicant di-interview untuk 2 job pada waktu yang sama
- **Priority ranking** — applicant bisa set preferensi urutan job
- **Transfer between jobs** — HR bisa transfer applicant dari satu job opening ke job lain tanpa harus re-apply
- **Application limit** — configurable max concurrent applications per applicant

**Estimasi Kompleksitas:** Medium

---

## 3. Advanced Search & Discovery

### Konteks Saat Ini
Job search menggunakan substring `Contains()` via EF Core. Filter: department, location, country. Sort: title, posted date, expired date. Tidak ada full-text search, fuzzy matching, atau search analytics.

---

#### 3.1 — Full-Text Search Engine
**Deskripsi:** Search saat ini hanya substring match pada title/description.

**Ide Pengembangan:**
- Integrasi **Elasticsearch** atau **Azure Cognitive Search** untuk full-text search
- Boolean search operators (AND, OR, NOT, quotes for exact phrase)
- Fuzzy matching — "sofware enginer" tetap menemukan "Software Engineer"
- Stemming dan lemmatization — "programming" juga match "programmer", "programs"
- Relevance scoring — tampilkan most relevant results first
- Search across all fields: title, description, requirement, skills, department, location

**Estimasi Kompleksitas:** High

---

#### 3.2 — Semantic Search / Natural Language Search
**Deskripsi:** Pencarian berbasis AI yang memahami intent.

**Ide Pengembangan:**
- "Cari pekerjaan yang butuh pengalaman React 3 tahun di Jakarta" → AI memahami dan filter
- "Remote job untuk fresh graduate" → Match dengan employment type + education level
- Voice search / speech-to-text
- Natural language filter builder — konversi query bahasa natural ke filter terstruktur

**Estimasi Kompleksitas:** High

---

#### 3.3 — Advanced Applicant Search untuk HR
**Deskripsi:** HR saat ini hanya bisa melihat applicant per-job di pipeline. Tidak ada global applicant search.

**Ide Pengembangan:**
- **Boolean resume search** — cari applicant berdasarkan skill, education, experience, lokasi
- Saved searches — HR simpan criteria pencarian untuk digunakan ulang
- **Talent pipeline** — proactively search existing applicants untuk job baru
- Filter: ATS score range, education level, years of experience, current status, application date
- Export search results ke CSV/Excel

**Estimasi Kompleksitas:** Medium-High

---

#### 3.4 — Job Alert Enhancement
**Deskripsi:** Saat ini `JobSubscription` memiliki `Keywords`, `PreferredDepartments`, `PreferredLocations`. Matching dilakukan 24 jam sekali oleh `SubscriptionBackgroundService`.

**Ide Pengembangan:**
- **Instant alerts** — push notification via SignalR saat job baru posting yang match
- Frequency options: Instant, Daily, Weekly digest
- More criteria: salary range, employment type, education level, experience level
- Smart alerting — hanya kirim alert untuk job yang substantially different dari sebelumnya
- In-app alert management — enable/disable tanpa unsubscribe completely
- **SMS alerts** selain email

**Estimasi Kompleksitas:** Medium

---

#### 3.5 — Search Analytics & Insights
**Deskripsi:** Tidak ada tracking search behavior saat ini.

**Ide Pengembangan:**
- Track most searched keywords, most viewed jobs, conversion rate (view → apply)
- **Zero-result search tracking** — identifikasi pencarian yang tidak membuahkan hasil
- Heatmap: jam/hari apa applicant paling aktif mencari
- A/B testing job titles untuk melihat mana yang lebih menarik
- Suggest popular searches ("People also searched for...")

**Estimasi Kompleksitas:** Medium

---

## 4. Analytics, Reporting & Dashboard

### Konteks Saat Ini
HR Dashboard (`HR/Index`) menampilkan total applicant, open/closed jobs, top 5 popular jobs. Admin punya Audit Trail dengan CSV export. Tidak ada reporting atau analytics mendalam.

---

#### 4.1 — Comprehensive Analytics Dashboard
**Deskripsi:** Dashboard analytics yang kaya data untuk decision-making.

**Ide Pengembangan:**
- **Recruitment funnel visualization** — konversi rate setiap stage (Application → Administration → HR Interview → EndUser Interview → Offering → Hired)
- **Time-to-fill metrics** — berapa hari rata-rata dari posting job sampai hire, per department/role
- **Time-to-hire** — dari application sampai acceptance
- **Source of hire** — dari mana applicant tahu job posting (direct, referral, social media)
- **Pipeline velocity** — seberapa cepat applicant bergerak antar stage
- **Diversity metrics** — (jika data tersedia) gender, education background distribution
- **Cost-per-hire tracking** — biaya rekrutmen per posisi
- Interactive charts: line, bar, pie, funnel, sankey diagram

**Estimasi Kompleksitas:** High

---

#### 4.2 — Custom Report Builder
**Deskripsi:** Saat ini hanya ada CSV export untuk jobs, applications, applicants, dan audit trails.

**Ide Pengembangan:**
- **Drag-and-drop report builder** — pilih columns, filters, grouping, sorting
- Scheduled reports — otomatis generate dan kirim via email (weekly/monthly)
- Report templates: Hiring Summary, Department Headcount, Interview Utilization, recruiter productivity
- Export ke PDF, Excel (XLSX), dan CSV
- Chart embedding ke dalam report
- Real-time dashboards yang bisa di-pin ke homepage

**Estimasi Kompleksitas:** High

---

#### 4.3 — Recruiter Performance Metrics
**Deskripsi:** Tidak ada tracking performa individual HR/recruiter.

**Ide Pengembangan:**
- Jobs managed per recruiter
- Applicants processed per week/month
- Average review time (waktu dari application masuk sampai di-review)
- Interview-to-offer ratio
- Offer acceptance rate per recruiter
- Leaderboard (gamification, lihat Section 18)

**Estimasi Kompleksitas:** Medium

---

#### 4.4 — Department Hiring Dashboard
**Deskripsi:** EndUser/Hiring Manager hanya melihat list job dan schedule.

**Ide Pengembangan:**
- Department-specific dashboard: open positions, pending interviews, headcount vs. target
- Budget tracking: allocated vs. spent per department
- Position aging — posisi yang sudah lama terbuka tanpa hire
- Headcount planning — forecast kebutuhan hiring berdasarkan growth/attrition

**Estimasi Kompleksitas:** Medium-High

---

#### 4.5 — Audit Trail Enhancement
**Deskripsi:** Audit trail saat ini hanya mencatat `Name`, `Role`, `Action`, `Job`, `Timestamp`. Tidak ada detail tentang apa yang berubah.

**Ide Pengembangan:**
- **Before/after diff** — tampilkan field apa yang berubah, dari value apa ke value apa
- **IP address logging** dan device info
- Filter audit trail by: date range, user, action type, entity type
- **Compliance report** — generate report untuk audit kepatuhan (GDPR, SOX)
- Alert pada suspicious activity (mass deletion, off-hours access)
- Retention policy — auto-archive audit trail setelah X bulan

**Estimasi Kompleksitas:** Medium

---

## 5. Interview & Scheduling

### Konteks Saat Ini
Interview scheduling dengan date/time/location, Jitsi Meet URL generation (public, tanpa auth), Google Calendar integration (commented out di EndUser), interviewer assignment, interview notes.

---

#### 5.1 — Advanced Calendar Integration
**Deskripsi:** Google Calendar integration sudah disiapkan di `Program.cs` tapi belum fully active (partial class di EndUserController.Calendar.cs semua commented out).

**Ide Pengembangan:**
- **Aktifkan dan perbaiki** Google Calendar integration for all roles
- Sync dua arah: buat calendar event saat schedule interview, update saat reschedule
- **Outlook/Microsoft 365 Calendar** integration
- **iCal file generation** — download .ics file untuk ditambahkan ke calendar manapun
- **Availability checker** — cek jadwal interviewer dan applicant sebelum scheduling
- Timezone-aware scheduling (saat ini hardcoded "SE Asia Standard Time")
- **Calendar view** di dalam app — weekly/monthly view semua interview

**Estimasi Kompleksitas:** Medium-High

---

#### 5.2 — Self-Service Interview Scheduling
**Deskripsi:** Saat ini HR meng-schedule interview sepihak. Applicant tidak bisa memilih waktu.

**Ide Pengembangan:**
- HR set **available time slots** (e.g., "Senin-Jumat, 10:00-16:00")
- Applicant **memilih slot** yang tersedia dari interface
- Auto-confirmation email setelah applicant memilih
- Reschedule request flow — applicant bisa minta reschedule dengan alasan
- **Buffer time** otomatis antar interview (e.g., 15 menit)
- Waitlist — jika preferred slot penuh, applicant masuk waiting list

**Estimasi Kompleksitas:** High

---

#### 5.3 — Video Interview Enhancement
**Deskripsi:** Saat ini Jitsi Meet hanya generate URL public tanpa auth, recording, atau room management.

**Ide Pengembangan:**
- **Jitsi JWT Authentication** — setiap room dilindungi dengan token, hanya peserta yang diundang bisa masuk
- **One-way video interview (async)** — applicant record jawaban atas pertanyaan preset, HR review nanti
- **Interview recording** — rekam interview dengan consent, simpan di server
- **Live coding interview** — embedded code editor (Monaco/CodeMirror) di samping video
- Integration dengan platform lain: Zoom, Google Meet, Microsoft Teams
- **Lobby/waiting room** — applicant menunggu sampai interviewer admit
- **Screen sharing** untuk technical presentation
- Post-interview evaluation form yang muncul otomatis setelah call berakhir

**Estimasi Kompleksitas:** Very High

---

#### 5.4 — Panel Interview Support
**Deskripsi:** Saat ini hanya 1 HR interviewer dan 1 EndUser interviewer per application.

**Ide Pengembangan:**
- **Multiple interviewers** — assign lebih dari 1 interviewer per stage
- Panel scoring — setiap interviewer memberikan score independen
- Consolidated score — rata-rata atau weighted average dari semua panel
- Interview debrief — shared notes area dimana semua interviewer bisa diskusi
- **Interview role assignment** — Lead Interviewer, Technical Assessor, Culture Fit Evaluator

**Estimasi Kompleksitas:** Medium-High

---

#### 5.5 — Interview Feedback & Scorecard
**Deskripsi:** Interview notes saat ini hanya free-text (`HrInterviewNotes`, `EndUserInterviewNotes`).

**Ide Pengembangan:**
- **Structured scorecard** — rating per kriteria (Communication, Technical Knowledge, Problem Solving, Culture Fit, dll.)
- Likert scale (1-5) untuk setiap kriteria
- Required fields — interviewer harus mengisi semua kriteria sebelum submit
- Comment per kriteria
- Overall recommendation: Strong Hire, Hire, No Hire, Strong No Hire
- Scorecard template bisa dikustomisasi per job/department
- Historical scorecard comparison antar applicant

**Estimasi Kompleksitas:** Medium

---

#### 5.6 — Interview Reminder System
**Deskripsi:** Tidak ada reminder otomatis untuk interview.

**Ide Pengembangan:**
- **Email reminder** ke applicant: H-1 hari dan H-1 jam sebelum interview
- Email reminder ke interviewer: H-1 hari
- **In-app notification** reminder via SignalR
- **SMS reminder** (opsional)
- Reminder jika interviewer belum submit feedback setelah interview selesai
- Configurable reminder timing per company/department

**Estimasi Kompleksitas:** Medium

---

## 6. Communication & Notification

### Konteks Saat Ini
Email via MailKit/Gmail SMTP, 6 email templates (rejection, HR interview, EndUser interview, offering, plus detail versions), SignalR real-time push notifications, notification inbox dengan mark-as-read dan bulk delete.

---

#### 6.1 — In-App Messaging / Chat
**Deskripsi:** Tidak ada messaging langsung antara HR, interviewer, dan applicant.

**Ide Pengembangan:**
- **Direct message** antara HR ↔ Applicant (per application context)
- **Internal chat** antara HR ↔ EndUser mengenai applicant tertentu
- Threaded conversations per applicant/job
- File sharing dalam chat (CV, portfolio, assignment)
- Chat history tersimpan dan bisa di-search
- Typing indicator dan read receipts via SignalR

**Estimasi Kompleksitas:** High

---

#### 6.2 — SMS / WhatsApp Notification
**Deskripsi:** Saat ini hanya email dan in-app notification.

**Ide Pengembangan:**
- **SMS gateway integration** (Twilio, Vonage) untuk notifikasi penting
- **WhatsApp Business API** — kirim update status, interview reminder, offering via WhatsApp
- Applicant bisa pilih preferred notification channel (Email, SMS, WhatsApp, In-App)
- Two-way SMS — applicant bisa reply untuk confirm/reschedule
- Template message approval flow (sesuai kebijakan WhatsApp Business)

**Estimasi Kompleksitas:** Medium-High

---

#### 6.3 — Email Campaign & Bulk Communication
**Deskripsi:** Saat ini email dikirim satu-satu per applicant per action.

**Ide Pengembangan:**
- **Bulk email** ke semua applicant di stage tertentu
- Email campaign untuk talent pool re-engagement ("Ada posisi baru yang cocok untuk Anda")
- Email sequence automation — series of nurture emails untuk applicant yang sudah di shortlist
- Open/click tracking per email
- Unsubscribe management yang lebih sophisticated
- Email template versioning dan preview

**Estimasi Kompleksitas:** Medium

---

#### 6.4 — Notification Preferences
**Deskripsi:** Saat ini semua user menerima semua notifikasi tanpa kontrol.

**Ide Pengembangan:**
- Applicant bisa pilih notifikasi mana yang ingin diterima: Application Status, Interview Schedule, New Matching Jobs, System Updates
- HR/Admin bisa set notification untuk: New Application, Pipeline Stage Changes, Interview Scheduled, Assessment Completed
- Per-channel control: Email ON, In-App ON, SMS OFF
- Quiet hours — jangan kirim notifikasi di luar jam kerja
- Digest mode — kumpulkan semua notifikasi dan kirim summary 1x sehari

**Estimasi Kompleksitas:** Medium

---

#### 6.5 — Rich Email Templates with Visual Editor
**Deskripsi:** Email template saat ini adalah string field di database (`EmailTemplate` entity) tanpa visual editor.

**Ide Pengembangan:**
- **WYSIWYG email editor** (drag-and-drop, seperti Mailchimp editor)
- Template variables/placeholders: `{{applicant_name}}`, `{{job_title}}`, `{{interview_date}}`
- Preview dan test send sebelum publish
- Multi-language template support
- Version history — rollback ke versi sebelumnya
- Responsive email design (mobile-friendly)
- Brand theming — company logo, colors

**Estimasi Kompleksitas:** Medium-High

---

## 7. Applicant Experience

### Konteks Saat Ini
Applicant bisa: view profile, edit profile, apply job, track applications, save jobs, get AI recommendations, manage subscription preferences, download CV.

---

#### 7.1 — Applicant Portal / Dashboard yang Lebih Kaya
**Deskripsi:** Applicant dashboard saat ini hanya TrackJob list.

**Ide Pengembangan:**
- **Visual pipeline tracker** — lihat di stage mana setiap lamaran (progress bar visual)
- Upcoming events: interview schedules, deadlines
- Personalized tips: "Complete your profile for better recommendations"
- Activity feed: "Your application for X has been moved to HR Interview"
- Profile completeness score / progress bar
- Quick stats: total applied, total saved, interviews scheduled, offers received

**Estimasi Kompleksitas:** Medium

---

#### 7.2 — Application Status Timeline
**Deskripsi:** Saat ini status hanya ditampilkan sebagai text (`StatusInJob` string).

**Ide Pengembangan:**
- **Visual timeline** per application — setiap stage change ditampilkan sebagai timeline event
- Timestamps untuk setiap transisi
- Estimated next step dan timeline expectations
- Notes/messages dari HR di setiap stage transition (jika ada)
- Status history yang bisa di-scroll

**Estimasi Kompleksitas:** Low-Medium

---

#### 7.3 — One-Click Apply & Apply with LinkedIn
**Deskripsi:** Saat ini applicant harus mengisi form dan upload CV setiap kali melamar.

**Ide Pengembangan:**
- **One-click apply** — gunakan CV dan profile yang sudah tersimpan
- **Apply with LinkedIn** — parse LinkedIn profile untuk auto-fill form
- **Apply with Indeed / JobStreet** profile
- Pre-filled form — jika sudah pernah apply, auto-fill dari data sebelumnya
- **Quick apply** — hanya perlu confirm, tanpa re-upload file

**Estimasi Kompleksitas:** Medium

---

#### 7.4 — Portfolio & Work Sample Upload
**Deskripsi:** Saat ini hanya CV (PDF) dan profile image yang bisa di-upload.

**Ide Pengembangan:**
- Upload **multiple documents**: cover letter, portfolio, certificates, transcripts
- Link ke portfolio online: GitHub, Behance, Dribbble, personal website
- **Video introduction** — applicant record short intro video
- Work sample gallery — preview images/PDFs dalam profil
- Document tagging dan categorization

**Estimasi Kompleksitas:** Medium

---

#### 7.5 — Application Draft & Multi-Step Form
**Deskripsi:** Saat ini form apply job adalah single-page.

**Ide Pengembangan:**
- **Multi-step application form** — break down ke beberapa step (Personal Info → Education → Experience → Upload → Review)
- **Auto-save draft** — applicant bisa mulai dan lanjutkan nanti
- Progress indicator di setiap step
- Validation per step (bukan semua di akhir)
- "Save and Continue Later" button

**Estimasi Kompleksitas:** Medium

---

#### 7.6 — Feedback untuk Rejected Applicants
**Deskripsi:** Applicant yang ditolak saat ini hanya menerima email rejection tanpa detail.

**Ide Pengembangan:**
- **Optional constructive feedback** — HR bisa include general feedback saat reject (e.g., "We suggest gaining more experience in X")
- Skill improvement suggestions berdasarkan ATS analysis
- Recommended courses/certifications
- "Apply again" — link ke posisi lain yang mungkin cocok
- Survey: "How was your application experience?" (NPS score)

**Estimasi Kompleksitas:** Low-Medium

---

#### 7.7 — Applicant Profile Visibility & Privacy Control
**Deskripsi:** Saat ini semua HR dan Admin bisa melihat semua applicant profiles.

**Ide Pengembangan:**
- Applicant bisa set profile visibility: Public (semua recruiter bisa lihat) vs. Private (hanya job yang dilamar)
- **Incognito mode** — apply tanpa menampilkan nama/foto ke hiring manager sampai HR interview
- GDPR-style data control: download semua data saya, hapus akun dan semua data
- Consent management — applicant approve/reject data usage untuk talent pool

**Estimasi Kompleksitas:** Medium

---

## 8. Talent Pool & CRM

### Konteks Saat Ini
Talent Pool: save applicant dari pipeline, admin notes, keyword-based talent←→job matching, import kembali ke job. Soft/hard delete.

---

#### 8.1 — Proactive Talent Sourcing
**Deskripsi:** Talent pool saat ini hanya diisi dari applicant yang sudah melamar.

**Ide Pengembangan:**
- **Manual candidate entry** — HR bisa add candidate tanpa mereka harus melamar (e.g., dari event, referral)
- **LinkedIn scraping/import** — import candidate dari LinkedIn profile URL
- **Resume parsing from email** — auto-detect dan parse CV yang masuk via email
- Bulk import dari spreadsheet (CSV/Excel)
- **Sourcing campaign tracking** — dari channel mana kandidat terbaik didapat

**Estimasi Kompleksitas:** High

---

#### 8.2 — Talent Pool Segmentation & Tagging
**Deskripsi:** Saat ini talent pool adalah flat list dengan basic search.

**Ide Pengembangan:**
- **Custom tags/labels** — "Senior", "High Potential", "Passive Candidate", "Bootcamp Graduate"
- **Smart segments** — auto-group berdasarkan skills, experience, education
- Talent pool folders/categories per department/skill area
- Pipeline per talent pool segment
- Aging indicator — berapa lama kandidat ada di pool tanpa engagement

**Estimasi Kompleksitas:** Medium

---

#### 8.3 — Talent Nurturing & Engagement
**Deskripsi:** Setelah di-save ke talent pool, tidak ada engagement follow-up.

**Ide Pengembangan:**
- **Automated nurture campaigns** — periodic check-in emails ("Are you still interested?")
- Content sharing — kirim company news, blog posts, event invitations
- Talent engagement score — seberapa responsif kandidat terhadap outreach
- Re-engagement triggers — alert saat kandidat update LinkedIn, atau skill match dengan new job
- Birthday/anniversary greetings automation

**Estimasi Kompleksitas:** Medium-High

---

#### 8.4 — Enhanced Talent Matching
**Deskripsi:** `TalentMatchingService` saat ini menggunakan primitive keyword matching (strip trailing 's' for pluralization).

**Ide Pengembangan:**
- **AI-powered semantic matching** — gunakan embeddings bukan keyword match
- Weighted matching — skill 50%, experience 30%, education 20%
- Experience-aware matching — tahun pengalaman vs. job requirement
- Location preference matching
- Salary expectation matching
- **Match score with explanation** — bukan hanya percentage, tapi penjelasan mengapa match/tidak
- **Reverse matching** — saat job baru dibuat, otomatis cari talent yang cocok dan notify HR

**Estimasi Kompleksitas:** High

---

## 9. Integration & Interoperability

### Konteks Saat Ini
Integrasi: Google OAuth, Facebook OAuth, Google Calendar (partial), DeepSeek AI, Jitsi Meet, Gmail SMTP.

---

#### 9.1 — Job Board Multiposting
**Deskripsi:** Saat ini job hanya posting di internal career page.

**Ide Pengembangan:**
- Posting otomatis ke **LinkedIn Jobs**, **Indeed**, **JobStreet**, **Glassdoor**, **Kalibrr**
- Status tracking per channel: mana yang menghasilkan applicant terbanyak
- One-click publish/unpublish ke semua channel
- Budget management per channel (jika paid posting)
- Application aggregation — applicant dari semua channel masuk ke satu pipeline

**Estimasi Kompleksitas:** High

---

#### 9.2 — HRIS / HR System Integration
**Deskripsi:** Setelah applicant di-hire, tidak ada handoff ke HR system.

**Ide Pengembangan:**
- Integration dengan HRIS populer: **SAP SuccessFactors**, **Workday**, **BambooHR**, **Talenta**
- Auto-create employee record dari hired applicant data
- Sync department data antara ATS dan HRIS
- Headcount requisition approval workflow dari HRIS
- Org chart integration — posisi di ATS map ke HRIS org structure

**Estimasi Kompleksitas:** Very High

---

#### 9.3 — REST API / GraphQL untuk Third-Party Integration
**Deskripsi:** Saat ini semua endpoint adalah MVC actions yang return Views. Tidak ada public API.

**Ide Pengembangan:**
- **RESTful API** dengan Swagger documentation untuk semua fitur
- **Webhook system** — notify external systems saat events terjadi (new application, status change, hired)
- OAuth2 / API key authentication untuk API consumers
- Rate limiting per API consumer
- GraphQL endpoint untuk flexible querying
- SDK generation (C#, JavaScript, Python) dari OpenAPI spec

**Estimasi Kompleksitas:** High

---

#### 9.4 — Social Media Integration
**Deskripsi:** Saat ini hanya OAuth login untuk Google dan Facebook. Tidak ada social sharing.

**Ide Pengembangan:**
- **Share job** ke LinkedIn, Twitter/X, Facebook, WhatsApp dari job detail page
- Social media referral tracking — track clicks dari shared links
- **LinkedIn Recruiter integration** — search LinkedIn profiles dari dalam ATS
- **Employee advocacy** — karyawan share job openings dengan tracking
- Instagram integration untuk employer branding content

**Estimasi Kompleksitas:** Medium

---

#### 9.5 — Slack / Microsoft Teams Integration
**Deskripsi:** Tidak ada integrasi dengan collaboration tools.

**Ide Pengembangan:**
- Slack bot: "New application for Software Engineer from John Doe (ATS: 8.5/10)"
- Teams notification channel untuk hiring updates
- Slash commands: `/hire status [job-id]`, `/hire approve [applicant-id]`
- Interactive approval buttons di Slack/Teams message
- Daily/weekly hiring summary di designated channel

**Estimasi Kompleksitas:** Medium

---

#### 9.6 — Single Sign-On (SSO)
**Deskripsi:** Saat ini menggunakan ASP.NET Identity + Google/Facebook OAuth.

**Ide Pengembangan:**
- **SAML 2.0** SSO untuk enterprise customers
- **Azure Active Directory** integration
- **Okta / Auth0** integration
- LDAP integration untuk on-premise organizations
- Role mapping dari SSO provider ke ATS roles

**Estimasi Kompleksitas:** Medium-High

---

#### 9.7 — Document Signing Integration
**Deskripsi:** Setelah offering, tidak ada digital document signing.

**Ide Pengembangan:**
- Integration dengan **DocuSign**, **Adobe Sign**, atau **HelloSign**
- Generate offer letter dari template
- Applicant sign offer letter secara digital
- Track signing status: Sent, Opened, Signed, Declined
- Auto-advance pipeline setelah document signed

**Estimasi Kompleksitas:** Medium

---

## 10. Security & Compliance

### Konteks Saat Ini
ASP.NET Identity, role-based authorization, partial CSRF protection, basic audit trail. Secrets di appsettings.json.

---

#### 10.1 — Two-Factor Authentication (2FA)
**Deskripsi:** ASP.NET Identity mendukung 2FA tapi belum diaktifkan.

**Ide Pengembangan:**
- Enable TOTP-based 2FA (Google Authenticator, Authy)
- SMS-based 2FA sebagai fallback
- Enforce 2FA untuk Admin dan HR roles
- Recovery codes untuk backup
- Remember device — skip 2FA untuk trusted devices

**Estimasi Kompleksitas:** Low-Medium

---

#### 10.2 — GDPR / Data Privacy Compliance
**Deskripsi:** Tidak ada fitur data privacy management.

**Ide Pengembangan:**
- **Data retention policy** — auto-delete applicant data setelah X bulan jika tidak hired
- **Right to be forgotten** — applicant bisa request complete data deletion
- **Data export (portability)** — applicant download semua data mereka dalam format standard (JSON/CSV)
- **Consent management** — explicit opt-in untuk data processing, talent pool, marketing emails
- **Privacy dashboard** — applicant lihat dan manage semua consent mereka
- Cookie consent banner
- Data Processing Agreement (DPA) template
- Anonymization of rejected applicant data after retention period

**Estimasi Kompleksitas:** High

---

#### 10.3 — Row-Level Security & Data Isolation
**Deskripsi:** Saat ini HR dan Admin bisa melihat semua data. Tidak ada pembatasan per department.

**Ide Pengembangan:**
- HR hanya bisa melihat applicant untuk job di department mereka
- EndUser hanya bisa melihat applicant yang assigned ke mereka
- Multi-tenant support — jika digunakan untuk multiple companies
- Admin bisa assign "department scope" ke HR users
- Data encryption at rest untuk sensitive fields (salary, contacts)

**Estimasi Kompleksitas:** High

---

#### 10.4 — Advanced Audit & Compliance
**Deskripsi:** Audit trail saat ini hanya basic (who, what, when).

**Ide Pengembangan:**
- **IP address logging** per action
- **Device fingerprinting** — detect suspicious login dari device baru
- **Session management** — view active sessions, force logout
- **Login history** dengan geolocation
- **Data access logging** — siapa yang view CV applicant tertentu
- Compliance reports: EEO (Equal Employment Opportunity), OFCCP
- Automated alerts untuk unusual activity patterns

**Estimasi Kompleksitas:** Medium-High

---

#### 10.5 — Anti-Fraud & Duplicate Detection
**Deskripsi:** Tidak ada deteksi duplicate applicant atau fraud.

**Ide Pengembangan:**
- **Duplicate applicant detection** — deteksi applicant yang mendaftar multiple kali dengan email berbeda
- CV plagiarism check — detect CV yang di-copy dari template/orang lain
- AI fake detection — identifikasi AI-generated CVs
- Identity verification — verify applicant identity via document upload (KTP, passport)
- Blacklist management — block applicant yang pernah di-flag

**Estimasi Kompleksitas:** High

---

## 11. Administration & Configuration

### Konteks Saat Ini
CRUD: Departments, Locations, Countries, Education Levels, Employment Types, App Users (dengan role assignment).

---

#### 11.1 — Company Settings & Branding
**Deskripsi:** Tidak ada company-level settings.

**Ide Pengembangan:**
- Company profile: logo, nama, tagline, deskripsi, website
- **Career page theming** — customize colors, fonts, hero image
- Company social media links
- About us / Culture page builder
- Company photo/video gallery untuk employer branding
- Custom domain support untuk career page

**Estimasi Kompleksitas:** Medium

---

#### 11.2 — Role & Permission Management yang Lebih Granular
**Deskripsi:** Saat ini hanya 4 role fix: Admin, HR, EndUser, Applicant.

**Ide Pengembangan:**
- **Custom roles** — Admin bisa buat role baru (e.g., "Junior HR", "Department Head", "Recruitment Lead")
- **Granular permissions** — per-feature permission (Can Create Job, Can View Salary, Can Export Data, Can Manage Users, dll.)
- **Permission groups/templates** — set of permissions yang bisa di-assign ke role
- Department-scoped permissions — HR bisa access hanya job/applicant di department tertentu
- Temporary access — berikan akses sementara ke interviewer eksternal

**Estimasi Kompleksitas:** High

---

#### 11.3 — Approval Workflows
**Deskripsi:** Saat ini semua aksi (create job, accept, reject) langsung execute tanpa approval.

**Ide Pengembangan:**
- **Job posting approval** — HR create job → Manager approve → publish
- **Offer approval** — HR buat offering → Finance/Manager approve budget → kirim ke applicant
- **Requisition workflow** — department submit hiring request → Admin approve → HR create job
- Multi-level approval chain
- Configurable approval rules per job level/salary band
- Auto-escalation jika approver tidak respond dalam X hari

**Estimasi Kompleksitas:** High

---

#### 11.4 — System Health & Monitoring Dashboard
**Deskripsi:** Tidak ada monitoring dashboard.

**Ide Pengembangan:**
- Background service health: AI worker status, subscription service status
- Email delivery rates dan bounces
- API health checks (DeepSeek, Google, SMTP)
- Database size monitoring
- Active user count dan concurrent sessions
- Error rate tracking
- Performance metrics (response time, throughput)

**Estimasi Kompleksitas:** Medium

---

#### 11.5 — Configurable Business Rules
**Deskripsi:** Business rules saat ini hardcoded.

**Ide Pengembangan:**
- Admin bisa set: max applications per applicant, max CV file size, allowed file types
- Auto-rejection rules: minimum ATS score, education level, experience years
- Job posting rules: mandatory fields, maximum posting duration
- Interview rules: minimum time between interviews, max interviews per day per interviewer
- Salary band configuration per role level
- Working hours configuration untuk scheduling

**Estimasi Kompleksitas:** Medium-High

---

## 12. Performance & Scalability

---

#### 12.1 — Database Migration ke PostgreSQL / SQL Server
**Deskripsi:** Saat ini menggunakan SQLite, cocok untuk development tapi tidak ideal untuk production.

**Ide Pengembangan:**
- Migrasi ke **PostgreSQL** (open-source, production-ready) atau **SQL Server**
- Connection pooling
- Read replicas untuk reporting queries
- Database backups automation
- Proper indexing strategy

**Estimasi Kompleksitas:** Medium

---

#### 12.2 — Caching Layer
**Deskripsi:** Tidak ada caching di sistem saat ini.

**Ide Pengembangan:**
- **Redis** atau **MemoryCache** untuk:
  - Department/Location/Country/Education/EmploymentType select lists (jarang berubah)
  - Job search results dengan short TTL
  - User session data
  - AI response caching (identik request → cached response)
- **Output caching** untuk public job board pages
- Cache invalidation strategy saat data berubah

**Estimasi Kompleksitas:** Medium

---

#### 12.3 — File Storage ke Cloud
**Deskripsi:** CV dan profile images disimpan di local filesystem (`Data/DataCV/`, `wwwroot/img/foto/`).

**Ide Pengembangan:**
- **Azure Blob Storage** atau **AWS S3** untuk file storage
- CDN integration untuk profile images
- Virus scanning pada file upload
- Automatic file compression/optimization
- File versioning — keep history CV uploads

**Estimasi Kompleksitas:** Medium

---

#### 12.4 — Distributed Background Jobs
**Deskripsi:** Background services menggunakan in-process `BackgroundService` dan `PeriodicTimer`.

**Ide Pengembangan:**
- **Hangfire** atau **Quartz.NET** untuk distributed job scheduling
- Job retry dengan configurable policy
- Job monitoring dashboard — lihat semua scheduled/running/failed jobs
- Horizontal scaling — multiple app instances bisa process jobs
- Dead letter queue untuk permanently failed jobs

**Estimasi Kompleksitas:** Medium

---

#### 12.5 — API Response Pagination & Optimization
**Deskripsi:** Beberapa API load semua data ke memory.

**Ide Pengembangan:**
- Cursor-based pagination untuk infinite scroll
- Server-side projection (select hanya field yang diperlukan)
- Lazy loading dengan proper includes
- Response compression (gzip/brotli)
- API response caching dengan ETags

**Estimasi Kompleksitas:** Medium

---

## 13. Mobile & Responsive

---

#### 13.1 — Progressive Web App (PWA)
**Deskripsi:** Saat ini hanya web app standar.

**Ide Pengembangan:**
- **Service worker** untuk offline access ke saved jobs dan application status
- **Push notifications** via Web Push API (bukan hanya SignalR)
- **Install to home screen** — app-like experience di mobile
- Offline-first — cache recent data, sync saat online kembali
- Background sync — submit application offline, auto-submit saat online

**Estimasi Kompleksitas:** Medium

---

#### 13.2 — Native Mobile App
**Deskripsi:** Tidak ada mobile app.

**Ide Pengembangan:**
- **React Native** atau **.NET MAUI** mobile app
- Applicant features: browse jobs, apply, track, notifications
- HR features: review applicants, approve/reject, schedule interviews
- Push notifications via FCM/APNs
- Camera integration — upload profile photo langsung dari kamera
- Document scanner — scan CV/documents dari kamera

**Estimasi Kompleksitas:** Very High

---

#### 13.3 — Responsive Design Improvements
**Deskripsi:** UI saat ini memiliki sidebar navigation per role.

**Ide Pengembangan:**
- Mobile-first responsive redesign
- Touch-friendly interface (larger tap targets, swipe gestures)
- Bottom navigation bar untuk mobile
- Collapsible sidebar
- Responsive tables dengan horizontal scroll atau card view
- Mobile-optimized CV viewer

**Estimasi Kompleksitas:** Medium

---

## 14. Export, Import & Data Management

### Konteks Saat Ini
CSV export untuk Jobs, Applications, Applicants, Audit Trails.

---

#### 14.1 — Advanced Export Options
**Ide Pengembangan:**
- **Excel (XLSX) export** dengan formatting, multiple sheets, header styling
- **PDF report export** — formatted reports, bukan hanya data dump
- Custom column selection — user pilih kolom mana yang di-export
- Scheduled export — otomatis generate dan email report setiap minggu/bulan
- Export template — simpan export configuration untuk reuse
- Large dataset streaming — export ribuan records tanpa memory issue

**Estimasi Kompleksitas:** Medium

---

#### 14.2 — Data Import
**Deskripsi:** Tidak ada fitur import data.

**Ide Pengembangan:**
- Import applicants dari CSV/Excel (bulk)
- Import jobs dari spreadsheet
- Import dari ATS lain (data migration tool)
- Import validation — preview data, highlight errors sebelum commit
- Duplicate detection saat import
- Field mapping UI — map columns dari CSV ke database fields

**Estimasi Kompleksitas:** Medium-High

---

#### 14.3 — Data Archival & Cleanup
**Ide Pengembangan:**
- **Auto-archive** closed jobs setelah X bulan
- Archive old applications yang tidak active
- Database cleanup scheduler — remove orphaned records
- Storage usage dashboard
- Export-before-archive — auto-export data sebelum archival

**Estimasi Kompleksitas:** Medium

---

## 15. Collaboration & Workflow

---

#### 15.1 — Team Collaboration & Comments
**Deskripsi:** Saat ini hanya interview notes yang bisa ditambahkan. Tidak ada collaboration tool.

**Ide Pengembangan:**
- **Comments/discussion thread** per applicant — HR, interviewer, manager diskusi
- **@mention** — tag team member untuk notifikasi
- **Reactions** — quick reactions (thumbs up, flag, star)
- Comment visibility control — internal only vs. shared with applicant
- Kanban-style sticky notes
- Activity feed — "John accepted this applicant", "Jane added interview notes"

**Estimasi Kompleksitas:** Medium-High

---

#### 15.2 — Employee Referral Program
**Deskripsi:** Tidak ada fitur referral.

**Ide Pengembangan:**
- **Referral portal** — karyawan submit referral dengan link personal
- Referral tracking — siapa yang refer, status referral
- **Referral bonus management** — track bonus eligibility, payout status
- Referral leaderboard
- **Social sharing dengan tracking links** — unique link per employee
- Auto-notify referrer saat referred candidate di-hire
- Referral analytics — source quality comparison

**Estimasi Kompleksitas:** Medium-High

---

#### 15.3 — Hiring Committee / Evaluation Panel
**Deskripsi:** Decision making saat ini ad-hoc tanpa structured process.

**Ide Pengembangan:**
- **Evaluation round** — semua dalam hiring committee submit rating independen
- Blind evaluation — evaluator tidak bisa lihat rating orang lain sebelum submit
- **Consensus meeting mode** — setelah semua submit, tampilkan comparison view
- Voting system — hire/no-hire vote per committee member
- Decision recording — record final decision dengan justification
- Meeting notes & action items

**Estimasi Kompleksitas:** Medium

---

## 16. Employer Branding & Career Page

---

#### 16.1 — Custom Career Page Builder
**Ide Pengembangan:**
- **Drag-and-drop page builder** untuk career/landing page
- Sections: Hero banner, About Us, Culture, Benefits & Perks, Open Positions, Testimonials, Office Gallery
- SEO optimization — meta tags, structured data (Schema.org JobPosting)
- Custom URL / subdomain
- Responsive design templates
- Analytics — page views, bounce rate, apply conversion

**Estimasi Kompleksitas:** High

---

#### 16.2 — Employee Testimonials & Company Culture
**Ide Pengembangan:**
- Employee testimonial cards dengan foto dan quote
- Team/department showcase pages
- Company culture values display
- "Day in the Life" content — teks/video tentang daily experience
- Social proof: awards, certifications, ratings dari Glassdoor/Great Place to Work
- Virtual office tour (360° images / video)

**Estimasi Kompleksitas:** Medium

---

#### 16.3 — Job Category & Landing Pages
**Ide Pengembangan:**
- Category pages: "Engineering Jobs", "Marketing Jobs", "Remote Jobs"
- Location-specific landing pages: "Jobs in Jakarta", "Jobs in Surabaya"
- SEO-friendly URLs (`/careers/engineering`, `/careers/jakarta`)
- Blog/content section — recruiting tips, company news
- Event page — upcoming career fairs, tech talks, meetups

**Estimasi Kompleksitas:** Medium

---

## 17. Onboarding Integration

---

#### 17.1 — Pre-Onboarding Module
**Deskripsi:** Saat ini setelah applicant accepted, tidak ada proses selanjutnya di sistem.

**Ide Pengembangan:**
- **Onboarding checklist** — auto-generate checklist saat applicant status = Accepted
- Document collection: KTP, NPWP, ijazah, SKCK, foto, bank account details
- **Online form completion** — new hire isi semua data sebelum Day 1
- Equipment request form
- Buddy assignment — assign mentor/buddy dari existing employees
- Pre-onboarding email sequence: welcome, documents needed, first day info
- Countdown to start date

**Estimasi Kompleksitas:** High

---

#### 17.2 — Onboarding Task Management
**Ide Pengembangan:**
- Task assignment untuk HR, IT, Manager per new hire
- Checklist progress tracking — percentage complete
- Automated reminders jika tasks overdue
- Digital signature untuk company policies, NDA, contracts
- IT provisioning request automation (email, laptop, access)
- New hire feedback survey setelah week 1 / month 1

**Estimasi Kompleksitas:** High

---

## 18. Gamification & Engagement

---

#### 18.1 — Applicant Engagement Gamification
**Ide Pengembangan:**
- **Profile completeness badge** — motivasi applicant melengkapi profil
- **Achievement badges** — "First Application", "5 Applications Sent", "Profile Star"
- Application streak — "Apply 3 days in a row"
- Points system — earn points for profile completion, quiz, referrals
- Leaderboard (opt-in) — top applicants berdasarkan engagement

**Estimasi Kompleksitas:** Low-Medium

---

#### 18.2 — Recruiter Gamification
**Ide Pengembangan:**
- **Weekly/monthly hiring goals** — target per recruiter
- Achievement tracking: "Fastest Review Time", "Most Hires This Month"
- Recruiter leaderboard — internal competition (fun & optional)
- Productivity streaks dan badges
- Team-based challenges

**Estimasi Kompleksitas:** Low-Medium

---

## 19. Accessibility & Internationalization

---

#### 19.1 — Multi-Language Support (i18n)
**Deskripsi:** Saat ini UI semua dalam bahasa campuran (English + beberapa Indonesian).

**Ide Pengembangan:**
- **Resource files (.resx)** untuk semua UI strings
- Language switcher — Bahasa Indonesia, English, dan extensible ke bahasa lain
- Multi-language job postings — satu job di-post dalam multiple bahasa
- Date/number/currency formatting sesuai locale
- RTL (Right-to-Left) support untuk bahasa Arab (future expansion)

**Estimasi Kompleksitas:** Medium-High

---

#### 19.2 — WCAG Accessibility Compliance
**Ide Pengembangan:**
- **WCAG 2.1 AA compliance** audit dan fixes
- Screen reader support — proper ARIA labels di semua komponen
- Keyboard navigation — semua fitur accessible tanpa mouse
- High contrast mode
- Font size adjustment
- Alt text untuk semua images
- Focus indicators yang jelas
- Color-blind friendly design (jangan rely hanya pada color untuk information)

**Estimasi Kompleksitas:** Medium

---

#### 19.3 — Multi-Currency Support
**Ide Pengembangan:**
- Salary fields support multiple currencies (IDR, USD, SGD, etc.)
- Auto-conversion display berdasarkan user locale
- Currency selection per job posting
- Salary range display normalization

**Estimasi Kompleksitas:** Low-Medium

---

#### 19.4 — Multi-Timezone Support
**Deskripsi:** Saat ini timezone hardcoded "SE Asia Standard Time" di beberapa tempat.

**Ide Pengembangan:**
- Auto-detect user timezone dari browser
- Timezone selector di user preferences
- Semua datetime disimpan sebagai UTC di database
- Display conversion per-user timezone
- Interview scheduling dengan timezone awareness — "10:00 AM WIB (03:00 AM UTC)"
- Timezone conflict detection saat scheduling international interviews

**Estimasi Kompleksitas:** Medium

---

## 20. DevOps & Infrastructure

---

#### 20.1 — CI/CD Pipeline
**Ide Pengembangan:**
- **GitHub Actions** atau **Azure DevOps** pipeline
- Automated build, test, lint on every PR
- Automated deployment ke staging dan production
- Code coverage tracking (integrate dengan existing 56 test files)
- Static analysis (SonarQube, CodeQL)
- Dependency vulnerability scanning (Dependabot, Snyk)

**Estimasi Kompleksitas:** Medium

---

#### 20.2 — Containerization & Orchestration
**Ide Pengembangan:**
- **Docker** containerization — Dockerfile untuk app + dependencies (Tesseract, tessdata)
- **Docker Compose** untuk local development (app + database + Redis)
- **Kubernetes** deployment manifests untuk production
- Health check endpoints (`/health`, `/readiness`)
- Horizontal scaling — multiple app instances behind load balancer

**Estimasi Kompleksitas:** Medium

---

#### 20.3 — Monitoring & Observability
**Ide Pengembangan:**
- **Application Performance Monitoring** (APM) — Datadog, New Relic, atau Application Insights
- **Structured logging** — Serilog dengan JSON output, correlation IDs
- **Distributed tracing** — track request flow across services
- Error tracking — Sentry atau Elmah
- Metric dashboards — Grafana + Prometheus
- Alerting — PagerDuty / Slack notifications untuk errors, high latency, downtime

**Estimasi Kompleksitas:** Medium

---

#### 20.4 — Feature Flags / Feature Toggle
**Ide Pengembangan:**
- Implement **feature flags** (LaunchDarkly, Azure App Configuration, atau custom)
- Gradual rollout — enable features untuk subset of users
- A/B testing infrastructure
- Kill switch — disable problematic features without deployment
- Environment-specific flags (dev, staging, production)

**Estimasi Kompleksitas:** Low-Medium

---

#### 20.5 — Automated Database Backups & Disaster Recovery
**Ide Pengembangan:**
- Scheduled database backups (daily/hourly)
- Backup to cloud storage (S3, Azure Blob)
- Point-in-time recovery
- Disaster recovery runbook
- Backup verification — periodic restore tests
- File storage backup (CVs, images)

**Estimasi Kompleksitas:** Medium

---

## Ringkasan Total Ide Fitur

| Kategori | Jumlah Fitur |
|----------|-------------|
| 1. AI & Machine Learning | 10 |
| 2. Recruitment Pipeline | 7 |
| 3. Advanced Search & Discovery | 5 |
| 4. Analytics & Reporting | 5 |
| 5. Interview & Scheduling | 6 |
| 6. Communication & Notification | 5 |
| 7. Applicant Experience | 7 |
| 8. Talent Pool & CRM | 4 |
| 9. Integration & Interoperability | 7 |
| 10. Security & Compliance | 5 |
| 11. Administration & Configuration | 5 |
| 12. Performance & Scalability | 5 |
| 13. Mobile & Responsive | 3 |
| 14. Export, Import & Data | 3 |
| 15. Collaboration & Workflow | 3 |
| 16. Employer Branding | 3 |
| 17. Onboarding Integration | 2 |
| 18. Gamification & Engagement | 2 |
| 19. Accessibility & i18n | 4 |
| 20. DevOps & Infrastructure | 5 |
| **TOTAL** | **96 ide fitur** |

---

## Priority Matrix

### Quick Wins (Low Effort, High Impact)
| # | Fitur | Estimasi |
|---|-------|----------|
| 7.2 | Application Status Timeline | Low-Medium |
| 7.6 | Feedback untuk Rejected Applicants | Low-Medium |
| 18.1 | Applicant Engagement Gamification | Low-Medium |
| 19.3 | Multi-Currency Support | Low-Medium |
| 20.4 | Feature Flags | Low-Medium |
| 10.1 | Two-Factor Authentication | Low-Medium |

### Strategic Investments (High Effort, High Impact)
| # | Fitur | Estimasi |
|---|-------|----------|
| 2.1 | Customizable Pipeline Stages | Very High |
| 2.2 | Kanban Board Pipeline | Medium-High |
| 4.1 | Comprehensive Analytics Dashboard | High |
| 1.4 | AI Candidate Ranking Dashboard | Medium-High |
| 9.3 | REST API / GraphQL | High |
| 5.2 | Self-Service Interview Scheduling | High |
| 3.1 | Full-Text Search Engine | High |

### Innovation Differentiators (High Effort, Differentiation)
| # | Fitur | Estimasi |
|---|-------|----------|
| 1.6 | AI Chatbot untuk Applicant | High |
| 1.7 | Predictive Analytics & Hiring Intelligence | Very High |
| 5.3 | Advanced Video Interview (async, recording, live coding) | Very High |
| 2.4 | Assessment & Technical Test Integration | Very High |
| 9.1 | Job Board Multiposting | High |

### Foundation Building (Medium Effort, Essential)
| # | Fitur | Estimasi |
|---|-------|----------|
| 12.1 | Database Migration ke PostgreSQL | Medium |
| 12.2 | Caching Layer (Redis) | Medium |
| 12.3 | Cloud File Storage | Medium |
| 20.1 | CI/CD Pipeline | Medium |
| 20.2 | Docker Containerization | Medium |
| 19.1 | Multi-Language Support | Medium-High |

---
