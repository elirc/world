# Senior Engineering Explanation: `src/services/payroll.service.js`

## Ownership and Intent
This file owns core domain behavior. It is where business invariants, transaction boundaries, and side effects must be coordinated coherently.

## How the Implementation Works
The file is structured around explicit module responsibilities and clear entry points. Import dependencies define collaboration boundaries, while exported symbols provide the public contract consumed by other layers.

Detected imports:
- node:fs
- node:path
- pdfkit
- @/lib/db
- @/lib/errors
- @/lib/money
- @/lib/audit
- @/lib/events
- @/services/context
- @/services/notification.service

Detected exports / entry points:
- listPayrollRuns
- getPayrollRun
- listPayrollItems
- createPayrollRun
- calculatePayrollRun
- approvePayrollRun
- processPayrollRun



## Code-Level Structure
Approximate line count: 656

Top-level declarations (module/global scope candidates):
- None detected in this file.

Function-level structure:
- asBigInt(value) -> function declaration, internal
- toMajor(minor) -> function declaration, internal
- computeProgressiveTax(brackets, taxableMajor) -> function declaration, internal
- computeTaxForRule(rule, taxableMinor, ytdMinor) -> function declaration, internal
- async loadCurrentCompensation(employeeId, periodEnd) -> function declaration, internal
- async loadYtd(employeeId, organizationId, periodStart) -> function declaration, internal
- ensureStorageDir() -> function declaration, internal
- async generatePayslip(item, employee, run) -> function declaration, internal
- async listPayrollRuns(user, organizationId = null) -> function declaration, exported
- async getPayrollRun(user, id) -> function declaration, exported
- async listPayrollItems(user, runId) -> function declaration, exported
- async createPayrollRun(user, payload, requestMeta = {}) -> function declaration, exported
- async calculatePayrollRun(user, runId, requestMeta = {}) -> function declaration, exported
- async approvePayrollRun(user, runId, requestMeta = {}) -> function declaration, exported
- async processPayrollRun(user, runId, requestMeta = {}) -> function declaration, exported

## Scope and State Model
Scope analysis:
- Scope usage is conventional: constants and helpers in module scope, request-specific values inside function scope.

State concepts observed:
- None detected in this file.

## Control Flow and Side Effects
Control-flow profile:
- Structured error boundaries are present (try: 1, catch: 1).
- Conditional branching is used to encode domain/path logic (if-count approx: 15).
- Iterative control flow is present for batch or aggregation behavior (loop-count approx: 4).
- Failures are surfaced with typed application errors, preserving stable API error semantics.

Observed side effects:
- database I/O through Prisma
- filesystem reads/writes
- user-facing notification side effects
- domain event outbox side effects

## Why It Is Implemented This Way
Design choices in this file prioritize explicit contracts, predictable side effects, and maintainable layering. This helps the team evolve behavior without hidden coupling.

Cross-cutting concerns currently present:
- transactional consistency
- audit logging
- domain event emission
- notification dispatch
- money math using minor units

## Safe Extension Guidance
- Keep business rules in the owning layer (service layer for domain policy, route layer for transport policy, UI layer for interaction policy).
- Preserve existing exported contracts when possible; when changes are required, update all call sites in the same change set.
- Keep module-scope mutable state minimal and intentional; prefer explicit factories for complex lifecycle state.
- For stateful UI files, keep pending/error/success transitions explicit and deterministic.
- For backend files with side effects, maintain idempotency and transactional coherence to avoid partial writes.

## Verified Review Notes (read against the current source)

The sections above are generated from the file's structure. These notes come from reading the code line by line:

- **State machine not enforced.** `calculatePayrollRun` (`:254-523`), `approvePayrollRun` (`:525-568`) and `processPayrollRun` (`:570-655`) never compare `run.status` with an expected value before writing. Recalculating a completed run resets `PAID` items to `CALCULATED` (`:380-399`).
- **Reads outside the transaction.** `loadCurrentCompensation` (`:86-99`) and `loadYtd` (`:101-128`) use the global `db` client although they are called inside `db.$transaction` (`:295`, `:303`, `:350`).
- **Ineffective per-employee catch.** After a failed statement PostgreSQL aborts the transaction, so the `FAILED` upsert in the `catch` (`:426-457`) cannot succeed for database errors.
- **File side effects in a transaction.** `generatePayslip` (`:136-167`) writes PDFs to `storage/payslips` from inside the processing transaction (`:597-607`); a rollback leaves the files behind.
- **YTD includes unapproved work.** `loadYtd` counts `CALCULATED` items (`:114-116`) and computes the year start in server local time (`:102`).
- **Tax rule details.** `payFrequency` is never consulted; progressive brackets add `flatAmount` per bracket reached (`:36-40`); an unrecognised `paidBy` charges both sides (`:83`).
- **Notifications outside the transaction.** Admin notifications after calculation (`:496-520`) run after commit and are not retried if they fail.

Exercise: **Goal** make approval safe against a run that is not `PENDING_APPROVAL`. **Check** your change is a conditional `updateMany` inside the transaction at `:534-542` and a `409` when nothing matched.
