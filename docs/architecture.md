# Architecture Guide

## Stack
- Runtime: Node.js
- Framework: Next.js App Router (JavaScript)
- Database: PostgreSQL via Prisma ORM
- Queue/Jobs: BullMQ + Redis (with inline fallback)
- Auth: JWT cookie sessions

## Codebase Layout
- `prisma/schema.prisma`
  - Full multi-tenant EOR schema (core, payroll, leave, billing, compliance, integrations, audit, outbox).
- `src/lib`
  - Infrastructure primitives: DB client, auth/session, API wrapper, errors, money, crypto, queue, audit, events.
- `src/services`
  - Domain business logic:
    - `auth.service.js`
    - `org.service.js`
    - `employee.service.js`
    - `contract.service.js`
    - `payroll.service.js`
    - `leave.service.js`
    - `contractor.service.js`
    - `billing.service.js`
    - `report.service.js`
    - `dashboard.service.js`
    - `tax.service.js`
- `src/app/api`
  - REST API handlers; thin layer calling services with validation + permission checks.
- `src/app/(app)`
  - Authenticated UI pages (dashboard, employees, contracts, payroll, leave, contractors, billing, reports, notifications, admin).
- `src/components/panels`
  - Client-side operational panels/forms per domain.
- `src/workers/domain-events-worker.cjs`
  - Outbox event processor.
- `docs/improved-implementation-plan.md`
  - Updated plan and architecture improvements over the original document.

## Request Lifecycle
1. Route receives request in `src/app/api/**/route.js`.
2. `withRoute` wrapper (`src/lib/api.js`) handles:
   - session resolution
   - permission checks
   - zod body validation
   - normalized success/error responses
3. Domain service executes and writes:
   - business state
   - `AuditLog`
   - `DomainEvent` (when applicable)
4. Response is returned to UI/API client.

## Multi-Tenancy Model
- Tenant key: `organizationId` on domain tables.
- Tenant control at service layer:
  - `resolveOrganizationId(...)`
  - `requireOrganizationAccess(...)`
- Platform admins can operate cross-tenant; all other roles are scoped to their org.

## RBAC Model
- Dynamic RBAC tables: `Role`, `Permission`, `RolePermission`, `UserRole`.
- Permissions use `resource.action` naming.
- API route guards specify required permission, for example:
  - `employees.view`
  - `payroll.approve`
  - `tax_rules.manage`

## Payroll Engine Design
- Main logic in `src/services/payroll.service.js`.
- Flow:
  1. Create run (`DRAFT`)
  2. Calculate:
     - load active employees + current compensation
     - apply tax rules by country
     - compute gross/tax/net per item
     - persist `PayrollItem` with YTD snapshots
     - set run `PENDING_APPROVAL`
  3. Approve run (`APPROVED`)
  4. Process run:
     - generate PDF payslip per item
     - mark items `PAID`
     - set run `COMPLETED`

## Contracts and Templates
- Templates are versioned in `ContractTemplate`.
- Contract rendering resolves merge tags from org/worker context.
- Lifecycle endpoints support create/update/send/sign and amendment chaining.

## Leave and Balances
- Policies in `LeavePolicy`.
- Requests in `LeaveRequest`.
- Running balances in `LeaveBalance`.
- Approval decisions update the balance and the request in one transaction (`src/services/leave.service.js:242-300`). The `PENDING` check runs before the transaction (`:233-235`) and both writes are by id only, so two concurrent approvals of the same request can both pass the check and add the days to `usedDays` twice.

## Billing
- Plans in `PricingPlan`.
- Invoices in `ClientInvoice`.
- Monthly invoice generation includes:
  - base fee
  - per-employee fee
  - payroll-derived service fee

## Events and Worker
- Mutating services can write `DomainEvent`.
- Worker (`npm run worker:events`) processes pending events and records `IntegrationLog`.
- Failed events transition through retry states and eventually `DEAD_LETTER`.

## Local Runbook
1. Create `.env` by hand (no `.env.example` is committed; `.gitignore` excludes `.env*`). Set `DATABASE_URL`, `AUTH_SECRET`, `FIELD_ENCRYPTION_KEY` (64 hex chars) and optionally `REDIS_URL`; see the README for what happens when each is missing.
2. Start infra:
   - `docker compose up -d`
3. Apply schema:
   - `npm run db:push`
4. Seed data:
   - `npm run db:seed`
5. Start app:
   - `npm run dev`
6. Optional worker:
   - `npm run worker:events -- --loop`

## Seeded Accounts
- Platform admin:
  - `platform.admin@example.com`
  - `ChangeMe!123`
- Client admin:
  - `client.admin@acme-global.example`
  - `ChangeMe!123`

## Verified Walkthrough: Session And Permission Resolution

- `withRoute` (`src/lib/api.js:16-61`) resolves the user on every request unless `auth: false`, then checks one permission string, then validates the body with Zod.
- The JWT (`HS256`, 8-hour expiry, `src/lib/auth.js:36-42`) carries only the subject; `getRequestUser()` reloads the user, organization, roles and role permissions from the database on each request (`:14-34`, `:132-140`) and flattens them into a permission list, adding `*` for `PLATFORM_ADMIN` (`:100-130`).
- Role defaults come from `DEFAULT_ROLE_PERMISSIONS` (`src/lib/permissions.js:31-74`) and are materialised per organization by `ensureBaseRbac()` (`src/services/rbac.service.js:69-113`). `ensureRole()` only ever adds missing permissions (`:41-63`); removing a permission from the defaults never removes it from existing roles.
- `EMPLOYEE` has no `payroll.view`, so employees cannot download their own payslips even though `processPayrollRun` notifies them with `actionUrl: "/my/payslips"` (`src/services/payroll.service.js:609-621`), a page that does not exist under `src/app/(app)`.

## Verified Walkthrough: Payroll Run

| Step | Route and permission | Service | What it actually checks |
|---|---|---|---|
| Create | `POST /api/payroll`, `payroll.process` | `createPayrollRun` (`payroll.service.js:222-252`) | tenant via `resolveOrganizationId`; writes `DRAFT` + audit in one transaction |
| Calculate | `POST /api/payroll/[id]/calculate`, `payroll.process` | `calculatePayrollRun` (`:254-523`) | tenant only, **no status check** |
| Approve | `POST /api/payroll/[id]/approve`, `payroll.approve` | `approvePayrollRun` (`:525-568`) | tenant only, **no status check** |
| Process | `POST /api/payroll/[id]/process`, `payroll.process` | `processPayrollRun` (`:570-655`) | tenant only, **no status check** |

Inside calculation, for each `ACTIVE`, `ONBOARDING` or `ON_LEAVE` employee (`:265-279`): load the current compensation (`:86-99`), take `amountMinor` as the gross for the run (`:340-346`; overtime, bonus, commission and allowances are hard-coded to zero), load year-to-date totals (`:101-128`), apply every active tax rule for the employee's country (`:352-368`, rule math at `:46-84`), and upsert one `PayrollItem` per employee (`:373-421`).

## Senior Review Findings

1. **The payroll state machine is not enforced.** None of calculate, approve or process reads `run.status` before writing. Consequences: a `DRAFT` run can be approved with zero items; processing an unapproved run pays nothing but still marks it `COMPLETED` (`:625-631`); recalculating a `COMPLETED` run upserts every item back to `CALCULATED` with `errorReason: null` (`:380-399`), discarding the `PAID` status while the payslip PDFs stay on disk. Fix: conditional `updateMany` on the expected status, checked inside the transaction.
2. **Reads escape the transaction.** `loadCurrentCompensation` and `loadYtd` use the global `db` client (`:87`, `:104`) inside the `db.$transaction` callback that started at `:295`. Also, in PostgreSQL a failed statement aborts the whole transaction, so the per-employee `catch` that upserts a `FAILED` item (`:426-457`) cannot recover from a database error; it will fail too and roll back the run.
3. **Payslip files are written inside the database transaction** (`generatePayslip` at `:598`, called from the transaction opened at `:579`). If a later step fails the rows roll back but the PDFs remain. Large runs also risk exceeding Prisma's default interactive-transaction timeout.
4. **Payslip download has no tenant check.** `src/app/api/documents/payslips/[id]/route.js:5-19` reads `storage/payslips/<id>.pdf` for any caller holding `payroll.view`, in any organization.
5. **Year-to-date double counting.** `loadYtd` sums items in `APPROVED`, `PAID` *and* `CALCULATED` status (`:114-116`), so a calculated-but-abandoned earlier run inflates YTD and therefore wage-base-capped taxes (`:66-71`). The year boundary uses server local time (`:102`).
6. **Tax math.** `payFrequency` (`prisma/schema.prisma:473`) is never read by the calculation, so brackets are applied to a single period's pay as if it were the whole taxable base. In `computeProgressiveTax` a bracket's `flatAmount` is added for every bracket the income reaches (`:36-40`). A rule with an unrecognised `paidBy` charges both employee and employer (`:83`).
7. **No four-eyes control.** `CLIENT_ADMIN` holds `payroll.process` and `payroll.approve` (`src/lib/permissions.js:38-40`), and nothing prevents the same user from calculating, approving and processing a run.
8. **Silent security fallbacks.** `AUTH_SECRET` defaults to a constant (`src/lib/auth.js:10`); missing or malformed `FIELD_ENCRYPTION_KEY` silently disables field encryption (`src/lib/crypto.js:5-30`), and a decryption failure returns the ciphertext as if it were the value (`:63-65`).
9. **Leave double approval** (see Leave and Balances above): check outside the transaction, id-only writes.
10. **Route params on Next 16.** `withRoute` passes `context.params` through un-awaited (`src/lib/api.js:46`) and handlers read `params.id` directly. Since Next.js 15, route `params` is a Promise; this project pins `next` 16.1.6 (`package.json`). Verify that dynamic routes receive their id before relying on any `[id]` endpoint; if they do not, await `context.params` in `withRoute`.

## Exercises

- **Goal:** prove the payroll state machine is unenforced. **Check:** you name the three functions and point to the first write in each (`:296`, `:535`, `:580`), none preceded by a status comparison.
- **Goal:** design the fix for finding 1. **Check:** your design replaces each `payrollRun.update({ where: { id } })` with `updateMany({ where: { id, status: <expected> } })`, throws `409` when `count === 0`, and moves the `findUnique` inside the transaction.
- **Goal:** close the payslip IDOR. **Check:** the route loads the `PayrollItem` with its run, calls `resolveOrganizationId(user, run.organizationId)`, and additionally lets an employee fetch only items whose `employee.userId` matches the caller.
- **Goal:** explain why the per-employee `catch` cannot record a database failure. **Check:** your answer mentions PostgreSQL's "current transaction is aborted" state after any failed statement inside `db.$transaction`.
