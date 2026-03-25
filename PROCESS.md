# PROCESS.md

This document explains how the WinForms/VB.NET attendance system is built and how the main components work together (based on your `.vb` code only; no designer code, `.resx`, or project files).

## 1) External NuGets / libraries used (inferred from `Imports` + usage)

1. `Npgsql`
   - PostgreSQL driver used everywhere for DB access (`dbConnect`, queries in all screens).
2. `BCrypt.Net` (`BCrypt.Net.BCrypt`)
   - Password hashing/verification for student login (`Form1.vb`) and admin login (`adminlogin.vb`).
   - Used during some admin super-account updates (e.g., `super_admin_panel.vb`).
3. `QRCoder`
   - Generates QR images for students (`register.vb`) and for regeneration after approvals (`arequest.vb`).
4. `OpenCvSharp` + `OpenCvSharp.Extensions`
   - Camera/video frames + QR decoding (`qscan.vb`) via `VideoCapture` and `QRCodeDetector`.
5. `AForge.Video.DirectShow`
   - Enumerates camera devices (`FilterInfoCollection` in `qscan.vb`).
6. `LiveCharts` / `LiveCharts.Wpf`
   - Admin dashboard charts in `adminform.vb` (WPF charts hosted in WinForms via `ElementHost`).

## 2) Overall coding/process structure (major modules)

1. **Startup / infrastructure connection selection**
   - `SchoolSelector.vb` loads available schools from `qrsys_admin.registered_schools` and builds a connection string into `modConfig`.
   - `DevLogin.vb` + `DevManagement.vb` enable a developer-only admin screen to add/update/delete "registered school nodes".
2. **Database connection plumbing**
   - `modConfig.vb` holds:
     - `SelectedSchoolName`
     - `CurrentConnString`
     - `IsDeveloperMode`
   - `dbConnect.vb` uses `modConfig.CurrentConnString` if set; otherwise falls back to a hardcoded connection string. It also owns the single shared `NpgsqlConnection` instance:
     - `OpenConnection()` opens and returns the connection
     - `CloseConnection()` closes/disposes it
3. **Session / RBAC state**
   - `SessionModule.vb` holds the currently logged-in identity for the app runtime:
     - `LoggedInUserId` (integer)
     - `LoggedInUsername` (string)
     - `IsSuperAdmin` (boolean)
4. **Student flow screens**
   - `Form1.vb` = student login + update check from GitHub releases.
   - `register.vb` = student registration + QR creation + registration email.
   - `mmenu.vb` = student menu (announcements, show QR, attendance).
   - `qview.vb` = display/export the student's QR image from DB.
   - `mattendance.vb` = show student's attendance history from DB.
   - `settings.vb` = submit profile/email change requests + change student password.
   - `info.vb` = "digital identification" panel for the student (shows status based on today's scans).
   - `forgotpass.vb` = password reset request submission (creates a row in `password_requests`).
5. **Admin flow screens**
   - `adminlogin.vb` = admin login (admin_accounts table) + "remember me" fields.
   - `adminform.vb` = main admin dashboard (charts + report + navigation).
   - `qscan.vb` = camera QR scanning; verifies QR payload and writes attendance records.
   - `arequest.vb` = admin approval/rejection of:
     - profile update requests (`change_requests`)
     - password reset requests (`password_requests`)
   - `manage_announcements.vb` = CRUD announcements.
   - `sfind.vb` = attendance finder/search (by student name/ID).
   - `aviewing.vb` = attendance viewer (all attendance).
   - `settingsform.vb` = admin "QR lookup" for a student (loads student's stored QR).
   - `ManageOrgs.vb` = grade level list maintenance (uses `sections` table as a combined grade/section store).
   - `section_manager.vb` = CRUD sections for grade levels (also uses `sections` table).
   - `aediting.vb` = edit late cutoff times (table `late_cutoffs`).
   - `adminlogs.vb` = view admin action audit logs (`admin_logs`).
   - `achangepass.vb` = admin password change screen (admin_accounts table).
6. **Super Admin flow screens**
   - `super_admin_panel.vb`:
     - loads admin_accounts list (showing who is super admin)
     - updates admin accounts' `uname`, `email`, and `is_superadmin`
     - blocks super admins from modifying their own `is_superadmin` status while logged in
   - `cregister.vb`:
     - superadmin-only "create admin account" screen (blocks non-super-admins)

## 3) Core data models (tables your code expects)

These are the DB tables referenced by the `.vb` files you provided:

1. `users`
   - student identity + attributes used for registration/login and QR payload building
2. `qcreate`
   - stores generated QR data + QR image (BLOB) per user
3. `Attendance`
   - attendance events written by `qscan.vb`
   - queried by reports, student attendance screens, and admin find/view screens
4. `announcements`
   - global announcements displayed on student `mmenu`
   - managed by admins
5. `change_requests`
   - pending/approved profile updates (grade/section/email where supported)
6. `password_requests`
   - pending/approved password reset requests
7. `admin_accounts`
   - admin login accounts + `is_superadmin` flag
8. `admin_logs`
   - audit log inserted via `LoggingModule.vb`
9. `sections`
   - grade levels and sections (code treats grade and section under the same table)
10. `late_cutoffs`
   - cutoff times used to classify late vs on time in `qscan.vb`
11. `qrsys_admin.registered_schools`
   - infrastructure registration table for developer mode and school selection
12. `qrsys_admin.dev_accounts`
   - developer authentication for DevManagement screens

## 4) End-to-end process logic (how the app "runs")

### 4.1 Startup and database routing
1. User lands on `SchoolSelector` (after splash/loader).
2. `SchoolSelector` loads active schools, and on "Proceed":
   - selects a school row
   - builds a connection string
   - writes it into `modConfig.CurrentConnString`
   - shows `Form1`
3. `Form1` uses `dbConnect.OpenConnection()` during student login.

### 4.2 Student registration + QR creation
1. `register.vb` loads organizations/grade levels from `sections`.
2. When registering:
   - it checks duplicate user by fullname OR student_id
   - hashes the password with BCrypt
   - inserts a row into `users`
   - builds a QR payload string with user info (UserID/Name/StudentID/Grade/Section)
   - generates a QR image (`QRCoder`) and stores:
     - the QR text
     - the QR image bytes
     - into `qcreate`
   - sends a registration email via `EmailService.EnqueueRegistrationEmail`

### 4.3 Student login + session
1. `Form1.vb` verifies student credentials using:
   - SQL by fullname match
   - BCrypt Verify on the returned password hash
2. On success:
   - sets `SessionModule.LoggedInUserId`
   - sets `SessionModule.LoggedInUsername`
   - shows `mmenu`

### 4.4 Admin QR scanning + attendance marking
1. Admin logs in (`adminlogin.vb`) and opens `adminform`.
2. Admin opens `qscan.vb` and starts the camera stream.
3. `qscan.vb` runs:
   - detects QR payload
   - extracts `ACCOUNTID:` or `STUDENTID:` from the scanned text
4. Attendance verification rules in `VerifyAndLogAttendance`:
   - If attendance already exists today:
     - if status begins with `O` -> "ALREADY MARKED OUT"
     - otherwise compute checkout status:
       - `L` -> `O(L)`
       - otherwise -> `O(E)`
   - If no record exists today:
     - compute late cutoff using `late_cutoffs` table (day-of-week row)
     - set status to:
       - `L` if late
       - `P` otherwise
5. It writes `Attendance` rows (INSERT for first scan, UPDATE for checkout).
6. It sends an email notification for the student using `EmailService.EnqueueEmail`.

### 4.5 Admin approvals (profile update + password reset)
1. Admin opens `arequest.vb`.
2. It loads pending items by UNION:
   - profile update rows from `change_requests` (pending)
   - password reset rows from `password_requests` (pending)
3. For approvals:
   - updates `users` (grade/section/email or password depending on request type)
   - updates the relevant request row to `Approved`
   - regenerates the student QR in `qcreate` when profile changes
4. For rejections:
   - updates the relevant request row to `Rejected`.

### 4.6 Student review screens
1. `mmenu` shows announcements from `announcements`.
2. Student can view:
   - QR code (`qview.vb`) from `qcreate`
   - attendance history (`mattendance.vb`) from `Attendance`
   - daily "digital identification" status (`info.vb`) which checks today's scans.
3. Student can submit requests through `settings.vb`:
   - profile update request (grade/section)
   - email update request
   - change password action (note: see security notes below)

## 5) RBAC enforcement points (what the code actually gates)

1. Super admin only "super admin panel"
   - In `adminform.vb` the super panel button is only visible when `IsSuperAdmin = True`.
   - If a click happens anyway, `btnSuperPanel_Click` checks `IsSuperAdmin` and shows:
     - `ACCESS DENIED: Super Admin only.`
2. Super admin only "create admin account"
   - In `cregister.vb` `cregister_Load` checks `IsSuperAdmin` and closes the screen if false.
3. Super admin cannot modify their own `is_superadmin`
   - In `super_admin_panel.vb` `btnUpdate_Click` blocks updates when the selected admin `id` matches `LoggedInUserId`.

## 6) Security notes (important issues found in the `.vb` code)

These are not guesses; they are based on how values are passed/hashed in the code:

1. Hardcoded Gmail SMTP credentials
   - `EmailService.vb` hardcodes both Gmail sender address and an app password constant.
2. Password update hashing is inconsistent
   - `Form1.vb` and `adminlogin.vb` use BCrypt verification (so DB passwords are expected to be BCrypt hashes).
   - But some update screens appear to write the new password "as entered" without hashing:
     - `settings.vb` updates `users.password = @p` using raw `txtNewPass.Text`
     - `achangepass.vb` updates `admin_accounts.password = @n` using raw `passtxt.Text`
   - Only some flows hash before writing (example: `super_admin_panel.vb` hashes on update; student registration hashes in `register.vb`).
3. QR payload schema inconsistency risk (minor)
   - `register.vb` generates `StudentID:` in QR text.
   - `arequest.vb` regenerates QR with `AccountID:` in QR text.
   - `qscan.vb` supports both by extracting either `ACCOUNTID:` or `STUDENTID:`, so it likely works, but the payload format is "two variants".

