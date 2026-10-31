# SOFTWARE REQUIREMENTS SPECIFICATION (SRS)
## Project: Remote Lab Reservation System (RLRS)
**Standard:** IEEE Std 830-1998 Compliant  
**Version:** 1.0.0-PROD  
**Date:** October 27, 2026  
**Context:** University Software Engineering Mini-Project (Team size: 4 members)  
**Timeline:** Sprint 1 (Oct 27–31, 2026) & Sprint 2 (Nov 02–07, 2026)  

---

## 1. INTRODUCTION

### 1.1 Purpose
This Software Requirements Specification (SRS) establishes the complete, authoritative functional and non-functional requirements for the **Remote Lab Reservation System (RLRS)**. The system provides an automated booking portal enabling university students to reserve physical and digital laboratory equipment timeslots, inspect real-time calendar availability, resolve scheduling contention via an atomic conflict-detection engine, and receive automated transactional and reminder emails.

### 1.2 Scope of the Software Product
The RLRS platform encompasses:
- A responsive client-side Single Page Application (SPA) featuring an interactive calendar widget.
- RESTful micro-endpoints governing authentication, inventory cataloging, timeslot validation, and administrative overrides.
- An atomic scheduling engine executing mathematical interval collision detection to prevent double-booking.
- An asynchronous cron daemon delivering automated email confirmations and 24-hour / 2-hour reminders.
- Administrative reporting tools for lab technicians and university faculty.

### 1.3 Definitions, Acronyms, and Abbreviations
- **RLRS:** Remote Lab Reservation System.
- **SRS:** Software Requirements Specification (IEEE 830 standard).
- **RBAC:** Role-Based Access Control (Student, Lab Technician, Administrator, Faculty).
- **JWT:** JSON Web Token (RFC 7519 standard for stateless authentication).
- **Interval Collision:** Mathematical overlap where $(Start_{new} < End_{existing}) \land (End_{new} > Start_{existing})$.
- **ICS:** iCalendar standard data format (RFC 5545) for calendar synchronization.
- **MoSCoW:** Prioritization taxonomy (Must have, Should have, Could have, Won't have).

### 1.4 References
- IEEE Std 830-1998: IEEE Recommended Practice for Software Requirements Specifications.
- RFC 7519: JSON Web Token (JWT).
- RFC 5545: Internet Calendaring and Scheduling Core Object Specification (iCalendar).

---

## 2. OVERALL DESCRIPTION

### 2.1 Product Perspective
RLRS is an independent, web-based reservation solution designed to replace manual whiteboard and paper sign-up logs in university engineering laboratories. It interfaces with institutional email gateways via SMTP and integrates with existing student authorization credentials.

### 2.2 User Classes and Characteristics
1. **Student:** Primary consumer. Seeks available equipment timeslots, submits reservations for class projects, views personal booking history, and receives email alerts. Requires an intuitive, zero-training interface.
2. **Lab Technician / Administrator:** Facility manager. Oversees equipment operational states, executes calibration maintenance lockouts, force-cancels bookings in emergencies with logged justification, and reviews utilization reports.
3. **Faculty (Optional / Future Scope):** Academic instructor. Requests semester-wide weekly recurring timeslot reservations for lab course cohorts.

### 2.3 Operating Environment
- **Client Tier:** Modern HTML5/CSS3 browsers (Chrome 110+, Firefox 115+, Safari 16+, Edge) on desktop and mobile viewports (minimum 375px width).
- **Application Server Tier:** Node.js (v20+ LTS) running Express REST framework with connection pooling.
- **Data Tier:** Relational SQL Database (MySQL 8.0+ or PostgreSQL 15+) supporting ACID transactions with `SERIALIZABLE` isolation level.
- **Mail Gateway:** University SMTP Relay server supporting TLS 1.3 encryption.

### 2.4 Design and Implementation Constraints
- **Team Size & Delivery:** University mini-project developed in 2 Sprints:
  - *Sprint 1 (Oct 27–31, 2026):* Core Auth, Equipment Catalog, Basic Slot Booking & Conflict Engine.
  - *Sprint 2 (Nov 02–07, 2026):* Calendar Widget Integration, Email Reminders, Admin Overrides, Metrics & Reports.
- **Tooling:** GitHub monorepo with CI/CD automation via GitHub Actions; Jira for backlog user stories and burndown tracking.
- **Zero Double-Booking Tolerance:** The scheduling engine must guarantee atomic interval locking to prevent race conditions during simultaneous user checkouts.

### 2.5 Assumptions and Dependencies
- University students possess active institutional `@university.edu` email inboxes.
- Lab operating hours are standardized between 08:00 and 22:00 local time.
- Physical lab equipment remains fixed to assigned room bench stations.

---

## 3. SPECIFIC REQUIREMENTS

### 3.1 Functional Requirements (FR)

| Req ID | Title | Priority | Category | Description | Acceptance Criteria |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FR-01** | User Authentication and Session Management | Must Have | Auth & Roles | The system shall authenticate users (Students, Lab Technicians/Admins, Faculty) via university institutional email and password credentials, issuing secure stateless JWT tokens containing the user's role. | • Tokens expire after 60 minutes of inactivity and are stored in secure httpOnly cookies or authorization headers.<br>• Failed login attempts exceed 5 times within 10 minutes trigger a 15-minute lock on the account.<br>• Passwords must meet institutional complexity standards (minimum 8 characters, 1 uppercase, 1 symbol, 1 digit). |
| **FR-02** | Role-Based Access Control (RBAC) | Must Have | Auth & Roles | The system shall enforce role-based authorization for all protected REST endpoints, restricting administrative actions (equipment provisioning, maintenance toggling, global overrides) to Lab Admins/Technicians. | • Students cannot access endpoints under `/api/admin/*` (HTTP 403 Forbidden returned).<br>• Technicians and Admins have privileges to manage all inventory, override bookings, and download audit logs.<br>• Faculty role (Optional/future scope) can request semester-long recurring laboratory reservations. |
| **FR-03** | Equipment Catalog Browsing and Categorization | Must Have | Equipment Catalog | The system shall display a searchable, filterable catalog of lab equipment with metadata including name, model number, laboratory room/bench ID, operating specs, reservation constraints, and current operational state. | • Filtering supports category (e.g., Oscilloscopes, Spectrum Analyzers, GPU Compute Nodes, Robotic Arms).<br>• Real-time operational badge indicates status: 'Available', 'Maintenance Required', or 'Decommissioned'.<br>• Equipment cards display max continuous booking duration (e.g., 2 hours for standard bench, 4 hours for compute nodes). |
| **FR-04** | Equipment Inventory Management | Must Have | Equipment Catalog | The system shall enable Lab Technicians and Admins to add, edit, flag, or remove equipment records, including updating hardware maintenance schedules and downtime alerts. | • Updating status to 'Maintenance' automatically marks all existing overlapping bookings during the maintenance window as 'Cancelled - Admin Overridden' with email notifications dispatched.<br>• Equipment cannot be deleted if active future bookings exist without explicit administrative confirmation.<br>• Changes to equipment specifications update immediately across the catalog cache. |
| **FR-05** | Interactive Calendar Availability View | Must Have | Calendar & Availability | The system shall provide an interactive calendar widget displaying booked, available, and maintenance slots across Day, Week, and Month views for any selected equipment item. | • Color-coded slots: Green (Available), Red/Amber (Booked), Striped Gray (Maintenance/Lab Closed).<br>• Time slots are rendered in configurable 30-minute or 60-minute increments between lab operating hours (08:00 to 22:00 local time).<br>• Client UI prevents selecting past date-times or dates beyond the 14-day advance booking horizon. |
| **FR-06** | Equipment Timeslot Reservation Request | Must Have | Booking & Conflict Engine | The system shall permit authenticated students to submit a booking request for an available timeslot on a specific piece of equipment with project/course rationale. | • Booking requires: Equipment ID, Start Timestamp, End Timestamp, Course Code, and Purpose Summary.<br>• System validates student has not exceeded the maximum active reservation quota (maximum 2 active future reservations per student).<br>• Booking confirmation generates a unique reservation reference code (e.g., `RES-2026-X892`). |
| **FR-07** | Atomic Scheduling Engine with Conflict Detection | Must Have | Booking & Conflict Engine | The scheduling engine shall atomically evaluate incoming reservation requests against existing bookings to guarantee zero overlapping bookings on identical hardware units. | • Uses mathematical interval overlap condition: `(NewStart < ExistingEnd) AND (NewEnd > ExistingStart)` for the same equipment where status is 'CONFIRMED'.<br>• Database transaction isolation (SERIALIZABLE or explicit row-level locking `SELECT ... FOR UPDATE`) prevents race conditions during simultaneous user submissions.<br>• If conflict is detected, the transaction rolls back, returns HTTP 409 Conflict, and displays exact conflicting time boundaries. |
| **FR-08** | Booking Quota and Duration Validation | Must Have | Booking & Conflict Engine | The scheduling engine shall enforce maximum continuous slot duration (e.g., max 120 minutes per single session) and a daily allowance ceiling per student. | • Rejects reservation if `(End - Start) > MaxAllowedDuration` with HTTP 422 Unprocessable Entity.<br>• Rejects requests where Start Time is within 15 minutes of the current timestamp (minimum advance notice buffer).<br>• Enforces minimum slot duration of 30 minutes. |
| **FR-09** | Reservation Rescheduling | Should Have | Reschedule & Cancellation | The system shall allow users to modify the start or end time of an active, upcoming reservation without losing the reservation if the new slot is available. | • Runs atomic conflict check excluding the current booking ID.<br>• Rescheduling is permitted up to 2 hours prior to scheduled start time.<br>• On successful reschedule, audit log record is created and an updated calendar invitation/email is queued. |
| **FR-10** | Self-Service Reservation Cancellation | Must Have | Reschedule & Cancellation | The system shall allow students to cancel their own future reservations at least 1 hour prior to start, immediately freeing the timeslot on the public calendar. | • Reservation state changes to 'CANCELLED_BY_USER' with timestamp.<br>• Freed timeslot transitions immediately to 'Available' in the calendar UI for other students.<br>• Email cancellation confirmation dispatched to student within 60 seconds. |
| **FR-11** | Automated Email Reservation Confirmation | Must Have | Email Notifications | The system shall automatically dispatch a structured confirmation email upon successful reservation creation, detailing lab room, bench number, equipment manual links, and an iCalendar (.ics) invite. | • Email is dispatched via SMTP/mail service within 30 seconds of database commit.<br>• Contains equipment details, check-in instructions, and cancellation link.<br>• Includes `.ics` calendar attachment for Google Calendar / Microsoft Outlook auto-sync. |
| **FR-12** | Scheduled Automated Email Reminders | Must Have | Email Notifications | A scheduled background daemon (cron service) shall scan upcoming reservations and dispatch an email reminder to students exactly 24 hours and 2 hours prior to reservation start time. | • Reminder cron triggers every 15 minutes, processing pending reminders idempotently (`reminder_sent_24h` and `reminder_sent_2h` boolean flags).<br>• Includes direct button to confirm attendance or cancel if no longer required.<br>• Does not trigger if reservation has already been cancelled or overridden. |
| **FR-13** | Lab Administrator Reservation Override and Force Cancellation | Must Have | Admin & Technician | The system shall empower Lab Administrators to force-cancel or reassign reservations in the event of emergency hardware malfunctions or urgent faculty research priorities. | • Admin must provide an mandatory override reason (minimum 10 characters).<br>• Affected student receives priority notification email titled 'URGENT: Lab Equipment Reservation Rescheduled/Cancelled'.<br>• Action is permanently logged in the security audit ledger. |
| **FR-14** | Student Personal Booking History and Dashboard | Must Have | Reporting & History | The system shall render a personal dashboard for students displaying Upcoming Bookings, Past Completed Sessions, and Cancelled Bookings with filterable status tabs. | • Upcoming bookings display countdown timer to start time.<br>• Allows 1-click 'Add to Calendar' (.ics export) and 'Cancel Booking' directly from the card.<br>• Historical bookings show total hours utilized in the current academic semester. |
| **FR-15** | Lab Utilization Analytics and Reporting | Should Have | Reporting & History | The system shall generate aggregate utilization reports for Lab Administrators, calculating equipment peak hours, utilization percentages, cancellation rates, and no-show metrics. | • Calculates Equipment Utilization Rate: `(Total Booked Hours / Total Lab Operating Hours) * 100%`.<br>• Exportable summary to CSV and PDF formats for academic departmental accreditation audits.<br>• Filterable by date range (e.g., Sprint 1, Sprint 2, Semester-to-Date) and lab room. |
| **FR-16** | Equipment Maintenance Mode Flagging | Must Have | Admin & Technician | The system shall permit technicians to mark specific units as under calibration or repair, setting an estimated return-to-service date. | • Equipment cannot be reserved while maintenance flag is active.<br>• Public calendar widget displays striped unavailable block with tooltip 'Under Calibration'.<br>• Returning unit to service automatically restores it to the public catalog. |
| **FR-17** | No-Show Detection and Slot Reclamation (Optional / Future Scope) | Could Have | Booking & Conflict Engine | If a student does not physically or digitally check in within 15 minutes of reservation start, the system shall mark the slot as 'NO_SHOW' and release the equipment for walk-in students. | • Marks student record with 1 strike; 3 strikes limit booking privileges for 14 days.<br>• Automated webhook notification to on-duty lab technician desk. |
| **FR-18** | Faculty Semester Recurring Group Reservations (Optional / Future Scope) | Won't Have (Sprint 1/2) | Admin & Technician | Faculty can batch-reserve lab benches for weekly 3-hour laboratory course practicals spanning 14 weeks of the university term. | • Validates all 14 weekly instances in a single atomic transaction.<br>• Identifies specific individual date conflicts before committing the batch. |

---

### 3.2 Non-Functional Requirements (NFR)

| Req ID | Category | Metric | Target Metric | Description & Verification |
| :--- | :--- | :--- | :--- | :--- |
| **NFR-01** | Performance | Scheduling Conflict Query Latency | `P95 < 150ms under 50 concurrent users` | **Description:** The atomic conflict-checking SQL query and API response for slot booking shall execute in under 150 milliseconds at the 95th percentile.<br>**Verification:** Automated k6 load test simulating 50 virtual users booking overlapping slots. |
| **NFR-02** | Performance | Calendar Grid Render & Hydration Time | `< 500ms initial load, < 100ms month/week switch` | **Description:** The client calendar widget shall load and render 30 days of slot data within 500ms on standard 4G broadband connections.<br>**Verification:** Lighthouse CI audit on React production build. |
| **NFR-03** | Security | Authentication & Role Integrity | `100% RBAC coverage; OWASP Top 10 compliance` | **Description:** All API endpoints must enforce role-based access validation. Passwords hashed using bcrypt (salt rounds = 12). Tokens signed with RS256/HS256 with 256-bit entropy secret.<br>**Verification:** Automated OWASP ZAP penetration scan and Postman RBAC integration suite. |
| **NFR-04** | Security | Data Protection & Injection Immunity | `Zero SQL injection vulnerabilities; TLS 1.3 enforced` | **Description:** All database queries must utilize parameterized prepared statements or ORM binding. All communication over HTTPS.<br>**Verification:** Static code analysis (SonarQube/GitHub CodeQL) and SSL Labs grade A+. |
| **NFR-05** | Usability | System Usability Scale (SUS) Score | `SUS Score >= 80 (Grade A); Mobile Responsive` | **Description:** First-time students must be able to discover an oscilloscope and complete a reservation in under 3 minutes without training. Fully responsive on viewports 375px to 2560px.<br>**Verification:** Cognitive walkthrough with 10 engineering students; WCAG 2.1 AA accessibility audit. |
| **NFR-06** | Reliability | Scheduling Engine Consistency & Uptime | `0% Double-Booking tolerance; 99.5% service uptime during sprint window` | **Description:** The scheduling engine must mathematically guarantee zero overlapping reservations for any single piece of physical equipment under any concurrency level.<br>**Verification:** Stress testing concurrent race condition scripts with 100 identical simultaneous requests. |
| **NFR-07** | Scalability | Data Volume & Concurrency Capacity | `Supports 500 active students, 50 lab stations, 10,000 reservation records without performance degradation` | **Description:** Database indexes on `(equipment_id, start_time, end_time, status)` ensure O(log N) lookup speed as reservation archives grow.<br>**Verification:** Database seeding with 20,000 synthetic rows and query execution plan `EXPLAIN ANALYZE`. |
| **NFR-08** | Maintainability | Test Coverage & CI/CD Pipeline Automation | `Unit & Integration test code coverage >= 80%; Automated CI pass on main` | **Description:** All business logic (conflict detector, date range validators, quota calculators) must have automated unit tests executed via GitHub Actions on every pull request.<br>**Verification:** Jest/Vitest coverage reports uploaded to CI workflow. |
| **NFR-09** | Reliability | Email Dispatch Fault Tolerance | `99% successful delivery within 120s; automatic 3x retry on transient SMTP error` | **Description:** The email notification daemon must employ exponential backoff retry logic if the SMTP gateway experiences transient failures.<br>**Verification:** Chaos test simulating simulated SMTP connection dropouts. |
| **NFR-10** | Maintainability | API Documentation & Contract Clarity | `100% OpenAPI 3.0 / Swagger specification conformance` | **Description:** All RESTful endpoints, request payloads, response bodies, and HTTP status codes (200, 201, 400, 401, 403, 404, 409, 422, 500) strictly documented.<br>**Verification:** Spectral OpenAPI linter validation in build pipeline. |

---

## 4. EXTERNAL INTERFACE REQUIREMENTS

### 4.1 User Interfaces
- **Responsive Dashboard:** Dual-tier layout with dark/light themes, breadcrumbs, and accessible typography.
- **Equipment Catalog Widget:** Grid and list displays with status badges and search/filter controls.
- **Interactive Calendar Widget:** 7-day grid showing 30-minute intervals from 08:00 to 22:00 with green (available), red (booked), and striped gray (maintenance) states.

### 4.2 Hardware Interfaces
- The web client communicates over standard TCP/IP network interfaces (Ethernet / 802.11 Wi-Fi).
- Future integration interfaces for smart bench IoT relays and RFID card readers are reserved via REST webhook hooks.

### 4.3 Software & API Interfaces
- **Database Interface:** SQL queries executed via parameterized prepared statements and connection pooling.
- **Email Interface:** Standard SMTP / STARTTLS on port 587 using MIME multipart templates with embedded `.ics` calendar payloads.

### 4.4 Communications Interfaces
- All client-server traffic strictly encrypted via HTTPS / TLS 1.3 using modern cipher suites.
- REST payloads encoded using standard JSON (`application/json`).

---

## 5. USE CASES

### 5.UC-01: Reserve Lab Equipment Timeslot
- **Actors:** Student
- **Preconditions:** Student is authenticated and has fewer than 2 active future reservations.
- **Postconditions:** Reservation record created with status 'CONFIRMED', calendar slot locked, confirmation email dispatched.
- **Main Flow:**
  1. Student navigates to Equipment Catalog and selects 'Tektronix MDO3024 Oscilloscope'.
  2. System displays interactive calendar with green/red/gray timeslot overlays.
  3. Student selects an available 2-hour window on tomorrow's date (14:00 - 16:00).
  4. System displays confirmation modal prompting for Course Code (EE301) and Project Purpose.
  5. Student clicks 'Confirm Reservation'.
  6. Scheduling engine executes atomic conflict check within transaction.
  7. System commits booking, issues Reservation ID `RES-2026-X892`, and renders success badge.
  8. Notification service enqueues confirmation email with `.ics` calendar attachment.
- **Alternative / Exceptional Flows:**
  4a. Student attempts to reserve > 2 hours: System displays error 'Maximum continuous slot is 120 minutes' and blocks submission.
  6a. Race condition: Another user booked overlapping slot 1 second earlier: Transaction rolls back, returns HTTP 409 Conflict with message 'This slot was just claimed. Please select another time window.', and calendar refreshes immediately.


### 5.UC-02: Cancel Active Reservation
- **Actors:** Student, Lab Admin
- **Preconditions:** A confirmed reservation exists with start time > 1 hour from current timestamp.
- **Postconditions:** Reservation status updated to 'CANCELLED_BY_USER' or 'CANCELLED_BY_ADMIN', calendar slot instantly available.
- **Main Flow:**
  1. User navigates to 'My Bookings' tab.
  2. User locates upcoming reservation and clicks 'Cancel Booking'.
  3. System prompts for confirmation.
  4. User confirms cancellation.
  5. System updates reservation state to 'CANCELLED_BY_USER' and records cancellation timestamp.
  6. Calendar availability cache is invalidated; timeslot turns Green for other users.
  7. Cancellation acknowledgement email is sent.
- **Alternative / Exceptional Flows:**
  2a. Reservation start time is within 60 minutes: System disables cancellation button with tooltip 'Cancellations locked within 1 hour of scheduled start. Contact lab technician desk.'


### 5.UC-03: Automated Reminder Cron Dispatch
- **Actors:** System Scheduler / Cron Daemon
- **Preconditions:** Active confirmed reservations exist within the 24-hour or 2-hour window where reminder flag is false.
- **Postconditions:** Emails transmitted, reminder flags set to true.
- **Main Flow:**
  1. Background cron worker wakes up on 15-minute interval.
  2. Queries database for `status = 'CONFIRMED' AND start_time BETWEEN NOW() AND NOW() + 24 HOURS AND reminder_sent_24h = FALSE`.
  3. Dispatches customized HTML reminder email containing reservation summary and cancellation link.
  4. Updates `reminder_sent_24h = TRUE`.
  5. Repeats query for 2-hour window and updates `reminder_sent_2h = TRUE`.
  6. Logs audit count of processed notifications.
- **Alternative / Exceptional Flows:**
  3a. SMTP server timeout: Worker logs warning, leaves flag as false, and retries on next 15-minute cycle.


### 5.UC-04: Mark Equipment for Emergency Maintenance
- **Actors:** Lab Technician / Administrator
- **Preconditions:** Technician is authenticated with 'ADMIN' or 'TECHNICIAN' role.
- **Postconditions:** Equipment status set to 'MAINTENANCE', overlapping bookings cancelled with admin override notices.
- **Main Flow:**
  1. Technician opens Admin Equipment Console.
  2. Selects hardware unit (e.g. 'Keysight N9020B Spectrum Analyzer').
  3. Clicks 'Set Maintenance Window' and specifies Start Date, End Date, and Cause ('Annual RF calibration').
  4. System detects 3 existing student reservations in the specified window.
  5. Technician confirms override.
  6. System updates equipment status and cancels affected student reservations with reason 'Hardware Maintenance'.
  7. Automated priority apology emails dispatched to all 3 students advising them to rebook on alternate units.
- **Alternative / Exceptional Flows:**
  4a. No reservations exist in window: System marks maintenance mode immediately without requiring cancellation confirmations.


---
*End of IEEE 830 Software Requirements Specification.*
