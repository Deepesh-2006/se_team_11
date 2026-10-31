# SYSTEM ARCHITECTURE DOCUMENT (SAD)
## Project: Remote Lab Reservation System (RLRS)
**Standard:** Kruchten 4+1 Architectural Views  
**Version:** 1.0.0-PROD  
**Date:** October 27, 2026  
**Architectural Style:** Layered Client-Server Architecture with RESTful Services & Asynchronous Background Workers  
**Context:** University Software Engineering Mini-Project  

---

## 1. ARCHITECTURAL GOALS, CONSTRAINTS & STYLE

### 1.1 Architectural Goals
- **Zero Double-Booking Guarantee:** The scheduling engine must guarantee atomic interval locking to prevent race conditions during simultaneous user checkouts.
- **Sub-150ms Query Latency:** High throughput interval indexing to permit rapid calendar rendering and conflict validation.
- **Stateless Application Server:** Node/Express API cluster capable of horizontal scaling behind a load balancer.
- **Asynchronous Reliability:** Decoupling user-facing reservation transactions from SMTP email transmissions via background worker queues.

### 1.2 Architectural Constraints
- Single GitHub repository for collaborative university mini-project.
- Multi-environment CI/CD pipeline using GitHub Actions for automated linting, test execution, and deployment.
- Strict RBAC authorization enforcement at the routing middleware layer.

---

## 2. 4+1 ARCHITECTURAL VIEWS

### 2.1 Logical View (Component Diagram)
```mermaid
graph TD
    subgraph Client Tier [Client Tier - React SPA]
      UI_Auth[Auth & Session Manager]
      UI_Catalog[Equipment Catalog Browser]
      UI_Calendar[Interactive Calendar Widget]
      UI_Booking[Booking & Reschedule Modal]
      UI_Admin[Admin Equipment & Overrides]
      UI_History[Student Dashboard & History]
    end

    subgraph API Tier [API Gateway & Application Server - Node / Express]
      MW_Auth[JWT Auth & RBAC Middleware]
      Ctrl_Equip[Equipment Controller]
      Ctrl_Book[Reservation Controller]
      Ctrl_Admin[Admin Controller]
      Engine_Sched[Scheduling & Conflict Detection Engine]
      Worker_Cron[Reminder Cron Daemon - 15m interval]
      Svc_Email[Email Notification Service / Nodemailer]
    end

    subgraph Data Tier [Relational Database - MySQL / PostgreSQL]
      DB_Users[(Users & Roles)]
      DB_Equip[(Equipment Inventory)]
      DB_Reservations[(Reservations & Intervals)]
      DB_Audit[(Audit & Maintenance Logs)]
    end

    subgraph External Services [External Gateway]
      SMTP[University SMTP Server / SendGrid]
    end

    UI_Calendar -->|REST / JSON| MW_Auth
    UI_Catalog -->|REST / JSON| MW_Auth
    UI_Booking -->|REST / JSON| MW_Auth
    UI_Admin -->|REST / JSON| MW_Auth

    MW_Auth --> Ctrl_Equip
    MW_Auth --> Ctrl_Book
    MW_Auth --> Ctrl_Admin

    Ctrl_Book --> Engine_Sched
    Engine_Sched -->|SELECT ... FOR UPDATE / SERIALIZABLE| DB_Reservations
    Ctrl_Equip --> DB_Equip
    Ctrl_Admin --> DB_Audit
    Ctrl_Admin --> DB_Equip

    Worker_Cron -->|Poll 24h/2h pending reminders| DB_Reservations
    Worker_Cron --> Svc_Email
    Ctrl_Book -->|Trigger Async Confirmation| Svc_Email
    Svc_Email -->|TLS / SMTP| SMTP
```

### 2.2 Process View: Booking & Conflict Detection Sequence
```mermaid
sequenceDiagram
    autonumber
    actor Student as Student (Browser)
    participant UI as React Calendar UI
    participant API as Express API Server
    participant Auth as RBAC Middleware
    participant Sched as Scheduling Engine
    participant DB as Relational Database
    participant Email as Email Worker

    Student->>UI: Selects slot (e.g. Oct 28, 14:00 - 16:00) & Submits Form
    UI->>API: POST /api/reservations (Bearer JWT, equipment_id, start_time, end_time)
    API->>Auth: Verify JWT & role permissions
    Auth-->>API: Authorized (user_id = 42, role = STUDENT)
    API->>Sched: evaluateBookingRequest(equipment_id, start_time, end_time, user_id)
    
    Sched->>DB: BEGIN TRANSACTION (ISOLATION LEVEL SERIALIZABLE)
    Sched->>DB: Check student active reservation quota (< 2 active)
    DB-->>Sched: Active quota = 1 (PASS)

    Sched->>DB: SELECT * FROM reservations WHERE equipment_id = ? AND status = 'CONFIRMED' AND (start_time < ?) AND (end_time > ?) FOR UPDATE
    
    alt Overlapping Slot Found (Conflict Detected)
        DB-->>Sched: Conflicting row returned (RES-710)
        Sched->>DB: ROLLBACK
        Sched-->>API: Conflict Error (Slot Unavailable)
        API-->>UI: HTTP 409 Conflict { error: "Slot already booked", conflictRange: [...] }
        UI-->>Student: Display red alert & refresh calendar availability
    else No Overlap (Slot Clear)
        DB-->>Sched: 0 rows returned
        Sched->>DB: INSERT INTO reservations (booking_ref, user_id, equipment_id, start_time, end_time, status) VALUES (...)
        Sched->>DB: COMMIT TRANSACTION
        Sched-->>API: Success (reservation_id = 902, booking_ref = "RES-2026-X892")
        API->>Email: enqueueConfirmationEmail(reservation_id)
        API-->>UI: HTTP 201 Created { booking_ref: "RES-2026-X892", status: "CONFIRMED" }
        UI-->>Student: Display green confirmation toast & update calendar grid
        Email-->>Student: Asynchronously dispatch confirmation email with .ics invite
    end
```

### 2.3 Process View: Automated Email Reminder Flow (24h / 2h Cron)
```mermaid
sequenceDiagram
    autonumber
    actor Cron as Scheduled Cron Daemon (Every 15m)
    participant API as Background Worker
    participant DB as Relational Database
    participant SMTP as University Mail Gateway
    actor Student as Student Inbox

    Cron->>API: Trigger scheduled tick: processReminders()
    API->>DB: SELECT r.*, u.email, u.first_name, e.name AS equip_name FROM reservations r JOIN users u ON r.user_id = u.user_id JOIN equipment e ON r.equipment_id = e.equipment_id WHERE r.status = 'CONFIRMED' AND r.reminder_sent_24h = FALSE AND r.start_time BETWEEN NOW() AND NOW() + INTERVAL '24 HOURS'
    DB-->>API: Return pending 24h reminder list [Reservation #902, #905]

    loop For each pending 24h reminder
        API->>SMTP: Send MIME HTML email (Subject: "Reminder: Lab Session in 24 Hours", details, cancel link)
        alt SMTP Delivery OK
            SMTP-->>API: 250 Message Accepted
            API->>DB: UPDATE reservations SET reminder_sent_24h = TRUE WHERE reservation_id = 902
            SMTP-->>Student: Deliver reminder email
        else SMTP Transient Error
            SMTP-->>API: 421 Service Unavailable
            API->>API: Log warning & leave reminder_sent_24h = FALSE for next retry
        end
    end

    API->>DB: Query for 2h reminders (reminder_sent_2h = FALSE AND start_time BETWEEN NOW() AND NOW() + INTERVAL '2 HOURS')
    DB-->>API: Return pending 2h reminder list
    API->>SMTP: Send final reminder email
    API->>DB: UPDATE reservations SET reminder_sent_2h = TRUE
```

### 2.4 Physical & Deployment View
```mermaid
graph LR
    subgraph Client Device
      Browser[Client Web Browser - Desktop/Mobile Chrome, Firefox, Safari]
    end

    subgraph Cloud Infrastructure - Cloud Run / Container Cluster
      subgraph Frontend Web Server
        Nginx[Vite Static Build / CDN Cache]
      end

      subgraph Backend App Services
        NodeExpress[Express Node.js REST API Server]
        CronService[Scheduled Cron Background Daemon]
      end

      subgraph Managed Data Store
        SQL[(PostgreSQL / MySQL Managed Database)]
        Redis[(Redis Cache - Optional / Future Session Store)]
      end
    end

    subgraph External University Infrastructure
      SMTP_Svc[University SMTP Mail Relay]
      SSO_IdP[University SSO / Identity Provider - Optional]
    end

    Browser -->|HTTPS / TLS 1.3 Port 443| Nginx
    Browser -->|REST API Requests / JSON| NodeExpress
    NodeExpress -->|Connection Pool TCP 5432| SQL
    CronService -->|Query & Lock Rows| SQL
    NodeExpress -->|Queue Jobs| Redis
    NodeExpress -->|SMTP Port 587| SMTP_Svc
    CronService -->|SMTP Port 587| SMTP_Svc
```

### 2.5 Data View: Entity-Relationship Diagram (ERD)
```mermaid
erDiagram
    USERS ||--o{ RESERVATIONS : places
    EQUIPMENT ||--o{ RESERVATIONS : has
    EQUIPMENT ||--o{ MAINTENANCE_LOGS : undergoes
    USERS ||--o{ AUDIT_LOGS : triggers

    USERS {
        int user_id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        enum role "STUDENT, TECHNICIAN, ADMIN"
        datetime created_at
        datetime last_login
    }

    EQUIPMENT {
        int equipment_id PK
        string name
        string model_number
        string category "OSCILLOSCOPE, SPECTRUM_ANALYZER, GPU_NODE, ROBOTIC_ARM"
        string room_bench
        int max_duration_minutes
        enum status "AVAILABLE, MAINTENANCE, DECOMMISSIONED"
        text specifications
        datetime created_at
    }

    RESERVATIONS {
        int reservation_id PK
        string booking_ref UK
        int user_id FK
        int equipment_id FK
        datetime start_time
        datetime end_time
        string course_code
        text purpose
        enum status "CONFIRMED, CANCELLED_BY_USER, CANCELLED_BY_ADMIN, COMPLETED"
        boolean reminder_sent_24h
        boolean reminder_sent_2h
        datetime created_at
        datetime updated_at
    }

    MAINTENANCE_LOGS {
        int log_id PK
        int equipment_id FK
        int logged_by_user_id FK
        datetime start_time
        datetime end_time
        string reason
        datetime created_at
    }

    AUDIT_LOGS {
        int audit_id PK
        int user_id FK
        string action_type
        string target_resource
        text details
        string ip_address
        datetime timestamp
    }
```

---

## 3. SCHEDULING ENGINE & CONFLICT DETECTION ALGORITHM

### 3.1 Mathematical Interval Formulation
Two time intervals $A = [Start_A, End_A)$ and $B = [Start_B, End_B)$ overlap if and only if:
$$(Start_A < End_B) \land (End_A > Start_B)$$

For an incoming reservation request for equipment $E$ spanning $[T_{reqStart}, T_{reqEnd})$:
```sql
SELECT reservation_id, booking_ref, start_time, end_time 
FROM reservations 
WHERE equipment_id = :equipment_id 
  AND status = 'CONFIRMED'
  AND start_time < :req_end_time 
  AND end_time > :req_start_time
FOR UPDATE;
```

### 3.2 Concurrency & Transaction Isolation
- **Isolation Level:** `SERIALIZABLE` or `READ COMMITTED` with explicit pessimistic row locks (`SELECT ... FOR UPDATE`).
- **Race Condition Prevention:** Any concurrent thread attempting to book the identical unit is blocked at the database engine until the prior transaction commits or rolls back, guaranteeing zero overlapping slots.

---

## 4. RESTFUL API SPECIFICATION

| Method | Path | Auth / Role | Description | Response & Error Codes |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/login` | All | Authenticates user with university email and password, returning JWT bearer token. | 200 OK, 400 Bad Request, 401 Unauthorized |
| `GET` | `/api/equipment` | Student, Technician, Admin, Faculty | Returns paginated and categorized list of lab equipment with operational status. | 200 OK, 401 Unauthorized |
| `GET` | `/api/equipment/:id/availability` | Student, Technician, Admin, Faculty | Retrieves booked and maintenance timeslots for a given equipment ID within a date window. | 200 OK, 400 Invalid Date Range, 404 Equipment Not Found |
| `POST` | `/api/reservations` | Student, Technician, Admin | Creates a new reservation after executing atomic interval conflict check. | 201 Created, 400 Bad Request, 409 Conflict (Overlap), 422 Quota/Duration Exceeded |
| `GET` | `/api/reservations/me` | Student, Technician, Admin | Retrieves current student's upcoming, past, and cancelled reservations. | 200 OK, 401 Unauthorized |
| `PATCH` | `/api/reservations/:id/cancel` | Student, Technician, Admin | Cancels an upcoming reservation and releases the slot for other users. | 200 OK, 400 Too Close to Start Time, 403 Forbidden, 404 Not Found |
| `PATCH` | `/api/reservations/:id/reschedule` | Student, Technician, Admin | Reschedules an existing reservation to a new non-conflicting time window. | 200 OK, 409 Conflict on New Slot, 422 Invalid Duration |
| `POST` | `/api/admin/equipment` | Technician, Admin | Provisions a new lab hardware unit into the institutional inventory catalog. | 201 Created, 400 Invalid Payload, 403 Forbidden |
| `PUT` | `/api/admin/equipment/:id/maintenance` | Technician, Admin | Flags equipment for maintenance and overrides conflicting student bookings. | 200 OK, 403 Forbidden, 404 Not Found |
| `GET` | `/api/admin/reports/utilization` | Technician, Admin | Computes lab equipment utilization percentage and booking metrics over specified period. | 200 OK, 403 Forbidden |

---

## 5. CI/CD PIPELINE DESIGN (GitHub Actions)

```yaml
name: CI/CD Pipeline - Remote Lab Reservation System
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - name: Install Dependencies
        run: npm ci
      - name: Code Quality & Lint
        run: npm run lint
      - name: Execute Vitest Suite
        run: npm test -- --coverage
      - name: Stress Test Conflict Engine
        run: npm run test:concurrency
```

---

## 6. REQUIREMENTS TRACEABILITY MATRIX (SAD)

| Requirement ID | Requirement Title | Architectural Component | Tier | Verification Artifact |
| :--- | :--- | :--- | :--- | :--- |
| **FR-01** | User Auth & JWT | Auth Middleware & User Controller | API Gateway | Unit & Integration Tests (TC-01..03) |
| **FR-02** | RBAC Authorization | RoleGuard Middleware | API Gateway | Security Matrix Tests (TC-04..06) |
| **FR-03** | Equipment Catalog | CatalogBrowser & Equipment Controller | Frontend UI | Catalog UI Tests (TC-07..08) |
| **FR-04** | Equipment Inventory | AdminEquipmentModal & DB_Equip | Persistence (DB) | Admin Suite (TC-09..10) |
| **FR-05** | Calendar Availability Widget | CalendarWidget & Availability API | Frontend UI | Calendar Functional Tests (TC-11..13) |
| **FR-06** | Timeslot Booking Request | BookingModal & Reservation Controller | Frontend UI | Booking Happy Path (TC-14..15) |
| **FR-07** | Conflict Detection Engine | SchedulingEngine & Transaction Manager | Scheduling Engine | Concurrency & Overlap Tests (TC-16..20) |
| **FR-08** | Quota & Duration Validator | BookingPolicyValidator | Scheduling Engine | Boundary Tests (TC-21..23) |
| **FR-09** | Reservation Rescheduling | RescheduleController | API Gateway | Reschedule Tests (TC-24..25) |
| **FR-10** | Self-Service Cancellation | CancelBookingController & DB | API Gateway | Cancellation Lockout Tests (TC-26..27) |
| **FR-11** | Confirmation Email + .ics | Nodemailer Service & ICS Generator | API Gateway | Email Mock Tests (TC-28) |
| **FR-12** | 24h & 2h Reminder Cron | ReminderCronDaemon | Cron Worker | Cron Daemon Simulation (TC-29..30) |
| **FR-13** | Admin Force Override | AdminOverrideController | API Gateway | Admin Override Suite (TC-31) |
| **FR-14** | Student Dashboard & History | StudentDashboardView | Frontend UI | Dashboard Render Tests (TC-32) |
| **NFR-01** | Conflict Latency < 150ms | Indexed DB Interval Query | Scheduling Engine | k6 Performance Load Tests |
| **NFR-06** | Zero Double-Booking | SERIALIZABLE Row-lock Engine | Scheduling Engine | Race Condition Stress Suite |

---
*End of System Architecture Document.*
