# Senior Engineering Explanation: `src/services/billing.service.js`

## Ownership and Intent
This file owns business logic for a bounded domain concern. It is designed to be invoked by routes and potentially background workers, so it centralizes invariants, transaction boundaries, and cross-cutting side effects such as audit logging and domain event publication.

## How the Implementation Works
The implementation follows a clear separation of concerns and keeps responsibilities explicit.

The service module centralizes domain invariants and transaction scope. Mutating flows are intentionally grouped so persistence changes, audit entries, and domain events can be committed atomically when needed.

This is the right place to evolve policy rules because callers (API routes or workers) should not duplicate domain logic.

Key dependencies imported here indicate coupling points: date-fns, @/lib/db, @/services/context, @/lib/audit.

Primary exported entry points: listPricingPlans, listInvoices, generateMonthlyInvoice.

Detected imports:
- date-fns
- @/lib/db
- @/services/context
- @/lib/audit

Detected exports / entry points:
- listPricingPlans
- listInvoices
- generateMonthlyInvoice

## Why It Is Implemented This Way
Design choices in this file favor maintainability over short-term convenience. The code is structured so that behavior changes can be made in one layer without causing cascading edits across unrelated modules.

Cross-cutting concerns currently visible in this file include: transactional consistency, audit logging, money math using minor units.

The service-first pattern enables consistent business outcomes regardless of invocation source (UI, admin tools, scheduled workers, or future integrations). This is a critical scaling property for enterprise workflows.

## Operational and Maintenance Considerations
This file currently has approximately 115 lines, which is manageable but should still be monitored for responsibility creep.

Operationally, future changes should preserve backward-compatible behavior at public interfaces (routes, exported service functions, and schema contracts).

When introducing new side effects, ensure they remain transactional or explicitly compensatable. Partial writes across core HR/payroll flows create expensive reconciliation work.

## Safe Extension Guidance
- 1. Add or change behavior in the owning layer only; avoid bypassing abstractions for convenience.
- 2. Keep input/output contracts explicit and update validators/types/route expectations together.
- 3. Preserve auditability for mutating workflows; if data changes materially, record who/when/what changed.
- 4. Add regression coverage (or at minimum reproducible manual verification steps) for critical workflow paths.
- 5. Maintain idempotency and clear transaction boundaries for workflows that can be retried.
