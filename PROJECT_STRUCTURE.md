# Pay365 / Odoo Payroll System — Codebase & Architecture Guide

This document provides a comprehensive map of the entire codebase, including directory trees, module architectures, and a file-by-file explanation of what each file does and how it operates within the system.

---

## 1. System Architecture Overview

The application is a full-stack, enterprise-grade HR, Attendance, Time-Off, Contract, and Rule-Based Payroll Management System designed following modular, layered domain architecture.

### Technology Stack
- **Backend:** Node.js (ESM), Express.js, Prisma ORM, PostgreSQL, Decimal.js, JWT, Zod, Vitest.
- **Engine:** Custom Abstract Syntax Tree (AST) Math & Logic Expression Parser with Topological Dependency Graph Sorting.
- **Frontend:** React 19, Vite, React Router v7, Lucide Icons, SCSS / Modular CSS Design System.
- **Infrastructure:** Docker, Docker Compose, PostgreSQL Container.

---

## 2. Root Project Structure

```
odoo-final/
├── docker-compose.yml       # Multi-container orchestration for PostgreSQL and services
├── package.json             # Root monorepo scripts & dependencies
├── README.md                # High-level product overview and setup instructions
├── PROJECT_STRUCTURE.md     # Detailed directory and file-by-file documentation (this file)
├── .env.example             # Template for global environment variables
├── .gitignore               # Monorepo git ignore definitions
├── docs/                    # Technical specs, SRS, and architecture documentation
├── backend/                 # Node.js + Express + Prisma REST API & Payroll Engine
└── frontend/                # React + Vite Enterprise HR & Payroll Web Application
```

### Root Files
- **`docker-compose.yml`**: Defines the PostgreSQL 16 database service container, persistent volume mappings, network configuration, and environment credentials.
- **`package.json`**: Root package definition managing concurrent development scripts (`npm run dev`, `npm run test`, monorepo linting).
- **`README.md`**: Master onboarding guide, product capabilities, installation steps, and API summary.

---

## 3. Backend Architecture (`/backend`)

The backend follows a modular domain structure where each feature is encapsulated with its own routes, controller, service, schema validation, and utilities.

```
backend/
├── Dockerfile               # Backend container image build definition
├── package.json             # Backend dependencies and execution scripts
├── vitest.config.js         # Vitest test suite configuration
├── prisma/
│   ├── schema.prisma        # Database schema models, enums, indexes, and relations
│   ├── seed.js              # Comprehensive database seeder with realistic test data
│   └── migrations/          # Version-controlled SQL migration history
├── src/
│   ├── server.js            # HTTP server startup and graceful shutdown lifecycle
│   ├── app.js               # Express app initialization, middleware pipeline, route mounting
│   ├── config/
│   │   └── env.js           # Validated environment variable loader (JWT secrets, DB URLs, ports)
│   ├── shared/
│   │   ├── prisma.js        # Global Prisma client singleton instance
│   │   ├── errors.js        # Standardized AppError classes and HTTP error codes
│   │   ├── logger.js        # Structured JSON application logger
│   │   └── constants.js     # Global constants and enum mappings
│   ├── middleware/
│   │   ├── auth.js          # JWT authentication and bearer token verification
│   │   ├── rbac.js          # Role-based access control (Admin, HR Manager, Officer, Employee)
│   │   ├── validate.js      # Zod request body/query/params validation middleware
│   │   ├── errorHandler.js  # Global error catch-all formatting uniform JSON responses
│   │   └── audit.js         # Automated HTTP request audit logging middleware
│   ├── engine/              # High-Performance Payroll Calculation Engine
│   │   ├── index.js         # Engine facade and batch execution entry point
│   │   ├── parser/          # Expression tokenization and AST parsing
│   │   ├── executor/        # Rule evaluation context, variable resolution, and math execution
│   │   ├── graph/           # Directed Acyclic Graph (DAG) for salary rule dependency sorting
│   │   └── engine.test.js   # Unit tests for rule parsing, execution, and edge conditions
│   ├── modules/             # Business Domain Modules
│   │   ├── auth/            # Login, token refresh, password reset, session management
│   │   ├── users/           # User accounts, role assignment, profile management
│   │   ├── employees/       # Employee master data, personal info, bank details, job associations
│   │   ├── contracts/       # Employment contracts, wage rates, date tracking, active state toggles
│   │   ├── schedules/       # Working schedules, shifts, weekly work hours
│   │   ├── attendance/      # Check-in/check-out, daily attendance tracking, overtime, anomalies
│   │   ├── timeoff/         # Leave allocation, leave requests, approval flows, balance calculation
│   │   ├── payroll-config/  # Salary structures, salary rules, calculation formulas, contribution rules
│   │   ├── payroll-run/     # Payrun batch management, computation orchestrator, payslip generator
│   │   ├── reports/         # Real-time dynamic reporting, analytics, CSV exports, KPI queries
│   │   ├── notifications/   # System alerts, email dispatch, notification logs
│   │   └── audit/           # Audit logs inspection and compliance trail
│   └── docs/
│       └── swagger.js       # OpenAPI / Swagger API specification
├── scripts/                 # Maintenance, verification, and migration scripts
└── tests/                   # End-to-end integration and regression test suites
```

---

### Backend File-by-File Breakdown

#### Root & Configuration
- **`backend/src/server.js`**: Initializes the Node.js HTTP server on configured port, connects to Prisma, and handles graceful termination signals (`SIGTERM`, `SIGINT`).
- **`backend/src/app.js`**: Configures Express application middleware (CORS, Helmet, JSON body parser, request logger, rate limiting), mounts all API v1 routes (`/api/v1/*`), and attaches the global error handler.
- **`backend/src/config/env.js`**: Reads and validates environment variables (Database URL, JWT Secret, Token expiry, Server Port) using Zod.
- **`backend/src/shared/prisma.js`**: Instantiates and exports the shared `PrismaClient` singleton with query logging in development mode.
- **`backend/src/shared/errors.js`**: Implements custom error classes (`AppError`, `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `ConflictError`).
- **`backend/src/shared/logger.js`**: Lightweight logging utility with log levels (`info`, `warn`, `error`, `debug`).

#### Middleware
- **`backend/src/middleware/auth.js`**: Intercepts incoming requests, verifies `Authorization: Bearer <token>`, decodes user identity, and attaches `req.user` to the request context.
- **`backend/src/middleware/rbac.js`**: Role-based access control guard. Enforces permission boundaries across roles (`ADMIN`, `HR_MANAGER`, `HR_PAYROLL_MANAGER`, `HR_PAYROLL_USER`, `EMPLOYEE`).
- **`backend/src/middleware/validate.js`**: Validates request `body`, `query`, and `params` against Zod schemas before reaching controllers, returning structured 400/422 responses on failure.
- **`backend/src/middleware/errorHandler.js`**: Intercepts all thrown exceptions, Prisma errors, and validation issues, returning clean `{ success: false, error: { code, message, details } }` responses.

#### Payroll Engine (`backend/src/engine/`)
- **`backend/src/engine/index.js`**: Engine interface orchestrating rule execution over an employee contract, attendance records, and leave inputs.
- **`backend/src/engine/parser/tokenizer.js`**: Lexical analyzer breaking math/logical formula strings into tokens (identifiers, operators, numbers, parentheses).
- **`backend/src/engine/parser/ast.js`**: Recursive descent parser converting token streams into Abstract Syntax Trees (AST).
- **`backend/src/engine/executor/evaluator.js`**: Evaluates AST expressions using `Decimal.js` for exact financial precision without IEEE-754 floating-point rounding errors.
- **`backend/src/engine/executor/index.js`**: Executes evaluated salary rules in sequence, maintaining intermediate variables (`BASIC`, `GROSS`, `HRA`, `PF_EE`, `PT`, `ESI_EE`, `TDS`, `NET`), handling conditional evaluations safely.
- **`backend/src/engine/graph/dag.js`**: Constructs a Directed Acyclic Graph of rule dependencies and performs Topological Sorting to guarantee variables are calculated in exact required dependency order.

#### Business Modules (`backend/src/modules/`)
- **`auth/`**:
  - `auth.routes.js`: Defines routes for `/login`, `/register`, `/refresh`, `/logout`, `/me`.
  - `auth.controller.js`: Handles authentication requests and HTTP cookies.
  - `auth.service.js`: Validates passwords via bcrypt, issues JWT access/refresh tokens.
  - `auth.schema.js`: Zod schemas for login and registration payloads.
- **`employees/`**:
  - `employees.routes.js`: Routes for CRUD operations on employee profiles.
  - `employees.controller.js`: Controller layer for employee queries and mutations.
  - `employees.service.js`: Business logic for employee onboarding, department/job assignment, and cascade deletion.
  - `employees.schema.js`: Validation for employee creation, updates, and query filters.
- **`contracts/`**:
  - `contracts.routes.js`: Endpoints for contract lifecycles, wage structures, and `/activate-now`.
  - `contracts.controller.js`: Handles contract creation, modification, and activation requests.
  - `contracts.service.js`: Enforces contract uniqueness (single active contract per employee), auto-links salary structures, and manages effective date transitions.
  - `contracts.schema.js`: Zod validation for wage amounts, contract types, dates, and status transitions.
- **`schedules/`**:
  - `schedules.routes.js` & `schedules.service.js`: Manages working hours, daily shift schedules, and weekend rules.
- **`attendance/`**:
  - `attendance.routes.js`, `attendance.controller.js`, `attendance.service.js`: Manages employee check-in/out, biometric logs, manual corrections, overtime hours calculation, and anomaly detection (late arrivals, missing checkouts).
- **`timeoff/`**:
  - `timeoff.routes.js`, `timeoff.controller.js`, `timeoff.service.js`: Manages leave types (Paid, Casual, Sick), allocation quotas, leave requests, manager approval/rejection workflows, and automated balance deductions.
- **`payroll-config/`**:
  - `salary-structure.service.js` & `salary-rule.service.js`: Manages customizable salary structures, rule sequences, fixed amounts, percentage rates, and formula-based rules.
- **`payroll-run/`**:
  - `payrun.routes.js`, `payrun.controller.js`, `payrun.service.js`: Manages payrun batch lifecycles (`DRAFT` -> `COMPUTED` -> `VALIDATED` -> `PAID`).
  - `orchestrator.js`: Core batch processor that collects employee contracts, calendar schedules, attendance, and approved leaves, evaluates salary rules through the Engine, and persists payslips.
  - `payslip.service.js`: Handles payslip generation, serialization, public format conversions, and PDF generation.
  - `preview.service.js`: Allows real-time calculation preview before committing payrun batches.
- **`reports/`**:
  - `analytics.routes.js`, `analytics.controller.js`, `analytics.service.js`: Real-time dynamic reporting engine querying live database tables for Department Payroll, Job Payroll, Monthly Trends, Statutory Compliance, Leave Utilization, Attendance Exceptions, and KPI summaries.
  - `csv.js`: High-performance streaming CSV generator for financial exports.
- **`audit/`**:
  - `audit.service.js`: Immutable audit logging recording who changed what, when, old values, and new values.

---

## 4. Frontend Architecture (`/frontend`)

The frontend is built with React 19 and Vite, following a feature-based architecture with reusable component libraries and SCSS design tokens.

```
frontend/
├── index.html               # Single-page application HTML entry point
├── package.json             # Frontend dependencies and Vite build scripts
├── vite.config.js           # Vite build, proxy, and alias configuration
├── src/
│   ├── main.jsx             # React DOM root render and global provider wrapping
│   ├── app/
│   │   ├── router.jsx       # Route definitions, protected routes, and lazy loading
│   │   ├── guards/
│   │   │   ├── RequireAuth.jsx   # Redirects unauthenticated users to /login
│   │   │   └── RequireRole.jsx   # Restricts views based on user roles
│   │   └── layout/
│   │       ├── AppShell.jsx      # Master layout container with responsive Sidebar and Header
│   │       ├── Navbar.jsx        # Top navigation bar with user profile and quick actions
│   │       ├── Sidebar.jsx       # Left navigation menu with route links and active indicators
│   │       └── AppShell.scss     # App layout styles
│   ├── lib/
│   │   ├── api.js           # Axios HTTP client instance with JWT auto-injection & 401 refresh
│   │   └── formatters.js    # Currency (₹ INR), date, percentage, and number formatting helpers
│   ├── components/
│   │   ├── ui/
│   │   │   ├── ToastContext.jsx  # Global toast notifications context provider
│   │   │   ├── Modal.jsx         # Accessible, animated modal dialog component
│   │   │   ├── Skeleton.jsx      # Content loading placeholder animations
│   │   │   ├── EmptyState.jsx    # Standard empty state graphic and messaging
│   │   │   └── ConfirmDialog.jsx # Confirmation dialog for destructive actions
│   ├── styles/
│   │   ├── _variables.scss  # Color palette, spacing tokens, typography, shadows, breakpoints
│   │   └── index.scss       # Global CSS reset, typography, and utility classes
│   └── features/            # Feature-Driven Pages & Components
│       ├── auth/
│       │   ├── pages/LoginPage.jsx       # Clean, secure login interface
│       │   └── context/AuthContext.jsx   # Global user state, login, logout, and token persistence
│       ├── landing/
│       │   └── pages/LandingPage.jsx     # Modern marketing landing page showcasing product capabilities
│       ├── dashboard/
│       │   └── pages/DashboardPage.jsx   # Real-time executive & employee self-service dashboard
│       ├── employees/
│       │   ├── pages/EmployeesPage.jsx       # Employee directory with search, filter, and pagination
│       │   ├── pages/EmployeeProfilePage.jsx # Comprehensive employee 360° profile view
│       │   └── components/CreateEmployeeModal.jsx # New employee onboarding modal
│       ├── contracts/
│       │   ├── pages/ContractsPage.jsx       # Contract list with status pills and quick actions
│       │   └── components/ContractModal.jsx  # Contract editor with wage rules and "Activate NOW" toggle
│       ├── attendance/
│       │   └── pages/AttendancePage.jsx      # Daily attendance logs, check-in kiosk, overtime manager
│       ├── timeoff/
│       │   └── pages/TimeOffPage.jsx         # Leave calendar, quota balance cards, request approvals
│       ├── payroll-config/
│       │   └── pages/SalaryConfigPage.jsx    # Salary structure builder, rule ordering, formula simulator
│       ├── payroll-run/
│       │   └── pages/PayrunsPage.jsx         # Payrun stepper (Draft->Compute->Validate->Paid) & payslips
│       ├── reports/
│       │   ├── pages/ReportsPage.jsx         # Live dynamic reporting suite with 8 interactive tabs
│       │   └── pages/ReportsPage.scss        # Styling for reporting tabs, KPI ribbon, and progress bars
│       └── audit/
│           └── pages/AuditLogsPage.jsx       # System audit logs and compliance history viewer
```

---

### Frontend File-by-File Breakdown

#### Core Setup & Layout
- **`frontend/src/main.jsx`**: Bootstraps the React root application inside `#root`, mounting `AuthContext`, `ToastProvider`, and `RouterProvider`.
- **`frontend/src/app/router.jsx`**: Central routing table mapping URLs to page components with `RequireAuth` and `RequireRole` security wrappers.
- **`frontend/src/app/layout/AppShell.jsx`**: Wraps authenticated pages with the persistent top Navbar and collapsible Sidebar navigation.
- **`frontend/src/lib/api.js`**: Configured Axios client with base URL `/api/v1`, automatic `Bearer` token header injection, and global 401 response interceptor for session expiration.
- **`frontend/src/styles/_variables.scss`**: Design system tokens for colors (`$color-primary`, `$color-success`, `$color-danger`, `$color-bg-app`), typography, borders, and shadows.

#### Feature Pages
- **`features/dashboard/pages/DashboardPage.jsx`**: Live executive dashboard showing organization payroll totals, department cost donut charts, monthly trend graphs, and employee self-service widgets (latest payslip, leave balance, attendance status).
- **`features/employees/pages/EmployeesPage.jsx`**: Employee directory featuring search, department filtering, status filters, card/table view toggle, and onboarding modal.
- **`features/employees/pages/EmployeeProfilePage.jsx`**: Tabbed employee profile displaying personal info, active contracts, job assignments, leave balances, attendance logs, and past payslips.
- **`features/contracts/pages/ContractsPage.jsx`**: Lists all employment contracts, displaying status (`ACTIVE`, `DRAFT`, `EXPIRED`), wage breakdown, and allows direct editing and instant activation.
- **`features/attendance/pages/AttendancePage.jsx`**: Attendance management interface with check-in/out kiosk, daily presence log, anomaly flags (`LATE`, `MISSING_CHECKOUT`, `MANUAL_EDIT`), and overtime tracker.
- **`features/timeoff/pages/TimeOffPage.jsx`**: Time-off management hub displaying available leave quotas, leave calendar, submission form, and manager approval/refusal queue.
- **`features/payroll-config/pages/SalaryConfigPage.jsx`**: Salary structure and rule editor. Allows configuring basic wages, allowances (`HRA`, `LTA`, `SPECIAL`), statutory deductions (`PF`, `ESI`, `PT`, `TDS`), rule sequences, and testing calculations live in the simulator.
- **`features/payroll-run/pages/PayrunsPage.jsx`**: Payroll execution center. Guides HR through the 4-step payroll lifecycle (`Draft` -> `Compute` -> `Validate` -> `Paid`), displays batch warning alerts, lists computed employee payslips, and enables downloading PDF payslips.
- **`features/reports/pages/ReportsPage.jsx`**: Real-time dynamic analytics suite featuring 6 KPI metric cards, date range presets, department filter, and 8 interactive sortable report tabs (Department Cost, Job Role Payroll, Monthly Trend, Statutory Deductions, Leave Utilization, Attendance Exceptions, and Employee Payslip Details).
- **`features/audit/pages/AuditLogsPage.jsx`**: Security audit log explorer tracking user actions, IP addresses, entity mutations, and timestamps.

---

## 5. End-to-End Data Flow Lifecycle

```
[User Action in Frontend UI]
       │
       ▼
[Axios API Client (`api.js`)] ── Injects JWT Token ──► [Express Route (`/api/v1/*`)]
                                                              │
                                                              ▼
                                                   [Auth & RBAC Middleware]
                                                              │
                                                              ▼
                                                   [Zod Schema Validation]
                                                              │
                                                              ▼
                                                   [Module Controller]
                                                              │
                                                              ▼
                                                   [Module Service Layer]
                                                              │
                     ┌────────────────────────────────────────┴────────────────────────────────────────┐
                     ▼                                                                                 ▼
          [Database Query (Prisma)]                                                        [Payroll Engine AST Evaluator]
                     │                                                                                 │
                     ▼                                                                                 ▼
           [PostgreSQL Database]                                                            [Topological Rule Sorter]
                     │                                                                                 │
                     └────────────────────────────────────────┬────────────────────────────────────────┘
                                                              ▼
                                                   [Standardized JSON / CSV]
                                                              │
                                                              ▼
                                                   [Dynamic React UI Render]
```

1. **Client Interaction:** User triggers an action (e.g. clicks "Compute Payslips" or adjusts date filters in Reports).
2. **API Request:** Frontend calls Axios `/api/v1/...` with current authentication credentials and query parameters.
3. **Authentication & Authorization:** Express middleware validates JWT claims and verifies that the user's role has permission to access the endpoint.
4. **Validation:** Zod schemas inspect request payload and reject invalid or malformed data before hitting database services.
5. **Business Logic & Computation:** 
   - For Payroll: Orchestrator queries employee contracts, attendance days, and approved time-offs, passing inputs into the Expression Engine to compute salary rules in dependency order using exact Decimal math.
   - For Reports: Queries live database tables aggregating gross wages, net pay, deductions, leave days, and anomalies.
6. **Response & Rendering:** Clean, structured JSON or CSV stream is returned to the client, updating UI tables, KPI ribbons, and charts in real time.

---
*Document automatically generated and aligned with codebase state.*
