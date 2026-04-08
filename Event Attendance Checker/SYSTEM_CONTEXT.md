# EAC System - Complete System Flow Reference
# Event Attendance Checker - ASIATECH CEITE
# Stack: Google Apps Script + Google Sheets + HTML/CSS/JS

## Architecture
- Frontend: Vanilla HTML/CSS/JS
- Hosting pattern: local HTML pages via Live Server or file system during development
- Backend: Google Apps Script Web App using `doGet(e)` and `doPost(e)`
- Database: Google Sheets spreadsheet named `EAC_Database`
- External APIs/services:
  - QuickChart.io for QR code image generation
  - Google Forms for event registration
  - Gmail `MailApp` for confirmation and OTP email delivery
- Version control: GitHub

## Deployment URL
- `scriptURL = 'https://script.google.com/macros/s/AKfycbzYdDr94uC7aByOQG7l2DNXUuIVfUxmn5aHBdl5_ZHvCabQoz0wHNSAtTdnRPVwepRK4g/exec'`

## System Purpose
Event Attendance Checker is a web-based attendance workflow for ASIATECH CEITE events.

Main objective:
- Faculty can create an event.
- The system automatically creates a Google Form for student registration.
- Students receive a confirmation email with a unique QR code.
- Faculty scan the QR code during the event for check-in and again after the event for checkout.
- Attendance status is updated inside Google Sheets.

## Roles
- Faculty
- Student
- Admin

Admin note:
- Faculty accounts are manually managed by admin.
- Faculty self-registration is not implemented.

## Database Sheets

### Events Sheet
Columns:
- A = Timestamp
- B = EventName
- C = Date
- D = Location
- E = Notes
- F = FormURL
- G = FacultyEmail
- H = FormID
- I = ResponseSheetName

### Faculty Sheet
Columns:
- A = Email
- B = Password
- C = Role
- D = Name

Notes:
- Managed manually by admin
- No self-registration for faculty

### Students Sheet
Columns:
- A = Email
- B = Password
- C = Role
- D = Name
- E = StudentID
- F = Course
- G = YearLevel
- H = Section

### Responses - EventName Sheet
One response sheet exists per event.

Columns:
- A = Timestamp
- B = Email
- C = FullName
- D = StudentID
- E = Course
- F = YearLevel
- G = Section
- H = UniqueCode
- I = QRCodeURL
- J = AttendanceStatus
- K = CheckInTime
- L = CheckOutTime

Status flow:
- `REGISTERED`
- `PRESENT`
- `COMPLETED (OUT)`

### OTP Sheet
Columns:
- A = Email
- B = OTPCode
- C = ExpiryTimestamp

Notes:
- Auto-created on first forgot password request
- OTP row is deleted after successful use or expiry

## API Surface

### GET: `doGet(e)` Read Operations
- `?action=getEvents` -> fetch all events
- `?action=getMyEvents&faculty=EMAIL` -> fetch faculty-specific events
- `?action=getAttendance&event=NAME` -> fetch attendance for a selected event
- `?action=getStudentAttendance&email=EMAIL` -> fetch student attendance across events
- `?action=createEvent&name=&date=&location=&notes=&faculty=` -> create a new event

### POST: `doPost(e)` Write and Sensitive Operations
All POST requests use:
- `Content-Type: application/x-www-form-urlencoded`
- body format: `data=JSON.stringify({ action, ...params })`

Supported actions:
- `login` -> `{ email, password }`
- `register` -> `{ email, password, name, studentId, course, yearLevel, section }`
- `scan` -> `{ code, event }`
- `deleteEvent` -> `{ name, faculty }`
- `changePassword` -> `{ email, currentPassword, newPassword, role }`
- `sendOTP` -> `{ email }`
- `verifyOTP` -> `{ email, otp, newPassword }`

## Backend Functions

### `doGet(e)`
Handles read actions and event creation request flow.

### `doPost(e)`
Handles writes, authentication, scanning, OTP, and other sensitive actions.

### `postNewEvent(name, date, loc, notes, facultyEmail)`
- Creates a Google Form with fields:
  - Full Name
  - Student ID
  - Course
  - Year Level
  - Section
- Uses `setCollectEmail(true)` to collect institutional email
- Uses `setLimitOneResponsePerUser(false)`
- Uses `setDestination` to link the form to the `EAC_Database` spreadsheet
- Saves event metadata to the `Events` sheet
- Stores `FormID` in column H

### `onFormSubmit(e)`
Installed spreadsheet trigger that fires on Google Form submission.

Flow:
- Reads `email`, `name`, `studentId`, `course`, `yearLevel`, and `section` from `e.values`
- Generates a unique code in the format `CEITE-XXXXXX`
- Generates QR image URL through QuickChart.io
- Writes the following into the response sheet:
  - unique code in column H
  - QR URL in column I
  - `REGISTERED` in column J
- Ensures headers for columns H to L exist
- Finds the matching event using `FormID` stored in `Events` column H
- Renames default response sheet from `Form Responses X` to `Responses - EventName`
- Saves the final response sheet name into `Events` column I
- Sends a QR confirmation email through `MailApp`

### `scanQrCode(uniqueCode, eventName)`
- Resolves the proper response sheet using `resolveResponseSheetByEventName(eventName)`
- Searches column H for the matching unique code
- First scan:
  - status becomes `PRESENT`
  - check-in timestamp is saved in column K
- Second scan:
  - status becomes `COMPLETED (OUT)`
  - check-out timestamp is saved in column L
- Third scan or later:
  - returns `ALREADY_SCANNED`
- Sends attendance confirmation email

### `deleteEvent(eventName, facultyEmail)`
- Matches using normalized event name and faculty email
- Response sheet resolution fallback order:
  - saved response sheet name from `Events` column I
  - standard `Responses - EventName` sheet name
  - `FormID`
  - loose match
- Deletes the event response sheet
- Trashes the Google Form using `DriveApp.getFileById(formId).setTrashed(true)`
- Deletes the event row last

### `getStudentAttendance(email)`
- Loops through all sheets whose name starts with `Responses - `
- Matches the email in column B using trim and normalization
- Returns:
  - eventName
  - name
  - studentId
  - course
  - yearLevel
  - section
  - status
  - checkIn
  - checkOut

### `login(email, password)`
- Checks the `Faculty` sheet first
- Checks the `Students` sheet next
- Returns:
  - `success`
  - `role`
  - `name`
  - `email`
  - student details when the user is a student

### `registerStudent(email, password, name, studentId, course, yearLevel, section)`
- Allows student self-registration
- Validates institutional email format using `@asiatech.edu.ph`
- Checks for duplicate email
- Saves the record in the `Students` sheet

### `changePassword(email, currentPassword, newPassword, role)`
- Chooses `Faculty` or `Students` sheet using the provided role
- Verifies the current password
- Updates password in column B

### `sendOTP(email)`
- Verifies the email exists in either `Faculty` or `Students`
- Generates a 6-digit OTP
- Sets a 5-minute expiry timestamp
- Saves or updates the `OTP` sheet row
- Sends the OTP via email

### `verifyOTPandReset(email, otp, newPassword)`
- Finds the email row in the `OTP` sheet
- Verifies OTP code match
- Verifies expiry has not passed
- Updates the password in `Faculty` or `Students`
- Deletes the OTP row after successful reset

### Helper Functions
- `sanitizeSheetName(name)` -> removes invalid Google Sheet name characters
- `_norm(v)` -> lowercase + trim normalization helper
- `findResponseSheetByFormId(ss, formId)` -> finds response sheet from form URL linkage
- `getEventMetaByName(eventName)` -> returns event row metadata
- `resolveResponseSheetByEventName(eventName)` -> resolves the proper event response sheet

## End-to-End Flow
1. Faculty creates an event.
2. Google Apps Script creates a Google Form and stores event metadata in the `Events` sheet.
3. Student submits the Google Form.
4. `onFormSubmit(e)` writes registration metadata, generates the QR, sets status to `REGISTERED`, and sends the email confirmation.
5. Faculty opens the scan page and scans the QR code during the event.
6. First scan changes status to `PRESENT` and stores check-in time.
7. Second scan after the event changes status to `COMPLETED (OUT)` and stores check-out time.
8. Further scans are blocked as `ALREADY_SCANNED`.

## Frontend Pages

### Faculty Pages
Current repo file mapping:
- [faculty-dashboard.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-dashboard.html) -> overview, totals, active events, quick links
- [faculty-create-event.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-create-event.html) -> create event form, calls GET `createEvent`
- [faculty-my-events.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-my-events.html) -> faculty event list and delete, calls POST `deleteEvent`
- [faculty-scan-qr.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-scan-qr.html) -> camera QR scanner, calls POST `scan`
- [faculty-reports.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-reports.html) -> attendance records, modal view, CSV download
- [faculty-profile.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-profile.html) -> faculty profile from session storage
- [faculty-settings.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\faculty-settings.html) -> settings and logout

Logical names used in documentation or discussion:
- `create-event.html`
- `my-events.html`
- `scan-qr.html`
- `view-records.html`

### Student Pages
Current repo file mapping:
- [student-dashboard.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\student-dashboard.html) -> student portal/dashboard, active events table, registered count
- [student-profile.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\student-profile.html) -> profile from session storage
- [student-attendance.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\student-attendance.html) -> attendance records and stats
- [student-settings.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\student-settings.html) -> password change page, calls POST `changePassword`
- [student-register.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\student-register.html) -> student self-registration, calls POST `register`

Logical names used in documentation or discussion:
- `student-portal.html`

### Auth Pages
- [loginpage.html](D:\dayle_personal\Documents\Practice Code\WebDev Practice\Random\official test eac\Event Attendance Checker\loginpage.html) -> login for faculty and students plus forgot password OTP flow

## Session Management
- Login stores session data using:
  - `sessionStorage.setItem('user', JSON.stringify(result))`
- User object shape:
  - `{ success, role, name, email, studentId, course, yearLevel, section }`
- Each protected page checks session storage and redirects to `loginpage.html` if user is missing or role mismatches
- Logout flow:
  - `sessionStorage.removeItem('user')`
  - `confirm()` dialog before logout

## Security Rules
- All sensitive or mutating operations should use POST
- Read-only operations use GET
- Institutional email is enforced with `@asiatech.edu.ph`
- Password policy:
  - minimum 8 characters
  - uppercase
  - lowercase
  - number
  - special character
- OTP rules:
  - 6-digit code
  - 5-minute expiry
  - single-use deletion after reset
- Role-based page protection exists on every page
- `sessionStorage` is cleared on logout and browser close

## Attendance Status Flow
- `REGISTERED` -> default after form submission
- `PRESENT` -> first QR scan, with check-in time
- `COMPLETED (OUT)` -> second QR scan, with check-out time
- `ALREADY_SCANNED` -> third scan or later, blocked

## Mobile Responsiveness
All pages are expected to support:
- Hamburger button on mobile
- Dark overlay
- Sidebar slide-in on mobile
- CSS media queries for `max-width: 768px` and `max-width: 480px`
- Horizontally scrollable tables on mobile

## Known Limitations
- Passwords are stored in plain text
- Production recommendation: migrate to hashed passwords such as SHA-256 or stronger server-side hashing
- `createEvent` still uses GET and can be migrated to POST for consistency
- Student portal announcements are currently static
- Faculty self-registration is not implemented

## Maintenance Rule
This file is the repo-level source of truth for architecture, flow, and shared terminology.

When the codebase changes:
- Update this file first or alongside the feature
- Keep file names, API behavior, sheet schema, and attendance statuses aligned with implementation
