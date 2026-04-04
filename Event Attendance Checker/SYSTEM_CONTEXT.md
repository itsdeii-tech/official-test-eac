# Event Attendance Checker - System Context

## Purpose
Web-based event attendance system for students and faculty.

Core goal:
- Let faculty create events and track real attendance with QR code scanning.

## Primary Roles
- Faculty
- Student

## Current Tech Stack
- Frontend: HTML pages (faculty and student views)
- Backend: Google Apps Script
- Registration intake: Google Forms (auto-created per event)
- Database: Google Sheets
- QR image generation: QuickChart.io API

## Core Business Flow
1. Faculty creates an event.
2. System automatically creates a Google Form for that event.
3. Students register through the form.
4. System sends confirmation email with each student's unique QR code.
5. During check-in at the event, faculty scans the student's QR code.
6. Attendance status changes from `Registered` to `Present`.
7. After the event, faculty scans the same QR code again for checkout.
8. System records checkout in the database.

## Attendance States
- `Registered`: Student completed event registration.
- `Present`: Student scanned on arrival/check-in.
- `Completed` (or equivalent checkout flag): Student scanned again after event.



## Data/Integration Notes
- Google Form captures registration data.
- Google Sheet acts as the central attendance data store.
- Google Apps Script coordinates:
  - event setup
  - form creation
  - email confirmation dispatch
  - attendance update logic
  - checkout handling
- QuickChart.io is used to generate QR code images tied to unique attendee identifiers.

## Operational Rule
- Same student QR is used for both check-in and checkout.
- Second valid scan should be interpreted as checkout, not duplicate check-in.

## Maintenance Rule
Whenever workflow, statuses, integrations, or schema change:
- Update this file first.
- Keep this file as source of truth for new sessions and future development.
