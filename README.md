# Velocity Grid

Employer-of-record (EOR) platform MVP built with Next.js, Node.js, and PostgreSQL.

## What Is Included
- Multi-tenant architecture with RBAC and org isolation
- Authentication with secure JWT cookie sessions
- Employee + contractor lifecycle modules
- Employee CSV automation (autofill first row + bulk import)
- Contract template rendering and e-signature workflow
- Payroll engine with tax rules, approval, processing, and payslip PDF generation
- Leave policies, requests, approvals, and balances
- Billing plans and monthly invoice generation
- Notifications, audit logs, reporting, and domain-event outbox
- Admin controls for tax rules/templates/organizations

## Tech Stack
- Next.js App Router (JavaScript)
- PostgreSQL + Prisma
- Redis + BullMQ (fallback inline mode when Redis is unavailable)
- Tailwind CSS

## Quick Start
1. Install dependencies:
   - `npm install`
2. Create `.env` (there is no `.env.example` in the repository: `.gitignore` excludes every `.env*` file). The code reads:
   - `DATABASE_URL` (Prisma datasource)
   - `AUTH_SECRET`: JWT signing key. If unset, `src/lib/auth.js:10` falls back to a hard-coded development string, so anyone can forge a session. Always set it.
   - `FIELD_ENCRYPTION_KEY`: 64 hex characters (32 bytes) for AES-256-GCM field encryption. If missing or the wrong length, `src/lib/crypto.js:23-30` stores fields in plaintext without any warning.
   - `REDIS_URL`: optional; without it jobs run inline (`src/lib/queue.js`).
3. Start infrastructure:
   - `docker compose up -d`
4. Apply schema:
   - `npm run db:push`
5. Seed data:
   - `npm run db:seed`
6. Start app:
   - `npm run dev`

Optional worker:
- `npm run worker:events -- --loop`

## Seed Accounts
- Platform admin:
  - `platform.admin@example.com`
  - `ChangeMe!123`
- Client admin:
  - `client.admin@acme-global.example`
  - `ChangeMe!123`

## Scripts
- `npm run dev`
- `npm run build`
- `npm run start`
- `npm run lint`
- `npm run db:generate`
- `npm run db:migrate`
- `npm run db:push`
- `npm run db:seed`
- `npm run db:reset`
- `npm run worker:events`

## CSV Import (Employees)
- Open `/employees`
- Use `Download CSV Template`
- Upload CSV and either:
  - `Autofill Form` (first row to form fields), or
  - `Import CSV Rows` (bulk create employees with row-level validation results)

## Documentation
- Improved plan: `docs/improved-implementation-plan.md`
- Codebase architecture: `docs/architecture.md`

## How a request flows

Every API route is a `withRoute(options, handler)` call (`src/lib/api.js:16-61`):

1. Read the `vg_session` JWT cookie, verify it with `jose` (`src/lib/auth.js:71-83`), then reload the user with roles and permissions from the database (`:14-34`, `:132-140`). Permissions therefore change immediately when roles change.
2. `requireAuth` → `401`; `requirePermission(user, options.permission)` → `403` (`src/lib/auth.js:156-174`). `PLATFORM_ADMIN` holds the wildcard `*` (`:109-112`).
3. If a `bodySchema` is set, parse JSON (`400 Invalid JSON body`) and validate with Zod (`400` with flattened issues) (`src/lib/api.js:30-42`).
4. Call the service with `{ request, params, query, user, body }` and wrap the result as `{ ok: true, data }` (`:44-56`).

Tenant isolation lives in services, not routes: `resolveOrganizationId(user, explicitId)` (`src/services/context.js:18-36`) returns the caller's own organization, rejects a different explicit id with `403`, and requires platform admins to name one.

## Known gaps (verified by reading the code)

- Payroll endpoints do not check the run's current status before approving, processing or recalculating it (`src/services/payroll.service.js:254-299`, `:525-542`, `:570-585`). See `docs/architecture.md` for the consequences.
- `GET /api/documents/payslips/[id]` serves any payslip file to any user with `payroll.view`, with no organization check (`src/app/api/documents/payslips/[id]/route.js:5-19`).
- User `status` is loaded with the session user but never re-checked on later requests (`src/lib/auth.js:100-140`), so deactivating a user does not end an existing 8-hour session.
- There are no automated tests and no `test` script in `package.json`.
