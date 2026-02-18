# Senior Engineering Explanation: `prisma/seed.js`

## Ownership and Intent
This file defines database-level contracts and persistence workflows. It is foundational because service logic and API behavior depend on these schema and data-shaping decisions.

## How the Implementation Works
The implementation follows a clear separation of concerns and keeps responsibilities explicit.

The database definition codifies invariants as close to storage as possible (types, relations, enums, uniqueness/index strategy). These contracts reduce ambiguity and prevent divergent behavior across services.

Detected imports:
None detected in this file.

Detected exports / entry points:
None detected in this file.

## Why It Is Implemented This Way
Design choices in this file favor maintainability over short-term convenience. The code is structured so that behavior changes can be made in one layer without causing cascading edits across unrelated modules.

Cross-cutting concerns currently visible in this file include: database client lifecycle management, background processing, money math using minor units.

Schema decisions here are long-lived and expensive to reverse, so explicit enums and relational constraints are preferred over loosely typed JSON-only persistence for core entities.

## Operational and Maintenance Considerations
This file currently has approximately 373 lines, which is manageable but should still be monitored for responsibility creep.

Operationally, future changes should preserve backward-compatible behavior at public interfaces (routes, exported service functions, and schema contracts).

## Safe Extension Guidance
- 1. Add or change behavior in the owning layer only; avoid bypassing abstractions for convenience.
- 2. Keep input/output contracts explicit and update validators/types/route expectations together.
- 3. Preserve auditability for mutating workflows; if data changes materially, record who/when/what changed.
- 4. Add regression coverage (or at minimum reproducible manual verification steps) for critical workflow paths.
