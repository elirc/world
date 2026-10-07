# Senior Engineering Explanation: `src/lib/auth.js`

## Ownership and Intent
This file provides shared infrastructure primitives. Its contracts are reused widely, so compatibility discipline is important.

## How the Implementation Works
The file is structured around explicit module responsibilities and clear entry points. Import dependencies define collaboration boundaries, while exported symbols provide the public contract consumed by other layers.

Detected imports:
- next/headers
- jose
- @/lib/db
- @/lib/errors

Detected exports / entry points:
- createSessionToken
- setSession
- clearSession
- readSessionFromRequest
- readSessionFromCookies
- normalizeUserContext
- getRequestUser
- getServerUser
- hasRole
- hasPermission
- requirePermission
- requireAuth



## Code-Level Structure
Approximate line count: 175

Top-level declarations (module/global scope candidates):
- const SESSION_COOKIE -> `const SESSION_COOKIE = "vg_session";`
- const MAX_AGE_SECONDS -> `const MAX_AGE_SECONDS = 60 * 60 * 8;`

Function-level structure:
- getAuthSecret() -> function declaration, internal
- async loadUserContext(userId) -> function declaration, internal
- async createSessionToken(payload) -> function declaration, exported
- async setSession(response, payload) -> function declaration, exported
- clearSession(response) -> function declaration, exported
- async readSessionFromRequest(request) -> function declaration, exported
- async readSessionFromCookies() -> function declaration, exported
- normalizeUserContext(user) -> function declaration, exported
- async getRequestUser(request) -> function declaration, exported
- async getServerUser() -> function declaration, exported
- hasRole(user, roleName) -> function declaration, exported
- hasPermission(user, permission) -> function declaration, exported
- requirePermission(user, permission) -> function declaration, exported
- requireAuth(user) -> function declaration, exported

## Scope and State Model
Scope analysis:
- Module scope declarations are used for reusable constants/helpers: SESSION_COOKIE, MAX_AGE_SECONDS.
- Session constants are module-scoped to keep auth contract names centralized and non-duplicated.

State concepts observed:
- request/session state via HTTP cookies
- configuration state sourced from environment variables

## Control Flow and Side Effects
Control-flow profile:
- Structured error boundaries are present (try: 2, catch: 2).
- Conditional branching is used to encode domain/path logic (if-count approx: 9).
- Iterative control flow is present for batch or aggregation behavior (loop-count approx: 2).
- Failures are surfaced with typed application errors, preserving stable API error semantics.

Observed side effects:
- database I/O through Prisma
- HTTP cookie mutation
- session cookie issuance/clear

## Why It Is Implemented This Way
Design choices in this file prioritize explicit contracts, predictable side effects, and maintainable layering. This helps the team evolve behavior without hidden coupling.

Cross-cutting concerns currently present:
- None detected in this file.

## Safe Extension Guidance
- Keep business rules in the owning layer (service layer for domain policy, route layer for transport policy, UI layer for interaction policy).
- Preserve existing exported contracts when possible; when changes are required, update all call sites in the same change set.
- Keep module-scope mutable state minimal and intentional; prefer explicit factories for complex lifecycle state.
- For stateful UI files, keep pending/error/success transitions explicit and deterministic.
- For backend files with side effects, maintain idempotency and transactional coherence to avoid partial writes.

## Verified Review Notes (read against the current source)

- **Secret fallback.** `getAuthSecret()` uses `process.env.AUTH_SECRET` or the constant `"dev_only_replace_me_32_characters_min"` (`:9-12`). With the variable unset, any party can mint a valid `vg_session` JWT.
- **What the token carries.** `createSessionToken` signs the payload with HS256 and an 8-hour expiry (`:36-42`). `getRequestUser` trusts only `sub` and reloads the user, roles and permissions from the database on each request (`:132-140`), so role changes apply immediately.
- **Status not enforced.** `normalizeUserContext` copies `user.status` (`:125`) but nothing in this file or in `withRoute` rejects inactive users.
- **Wildcard admin.** Any role named `PLATFORM_ADMIN` yields the `*` permission (`:109-112`), and `hasPermission` honours `*` (`:156-162`). The check is by role *name*, so an organization-scoped role created with that name would also be a platform admin.
- **Cookie clearing.** `clearSession` resets only name, value, path and `maxAge` (`:60-69`); it does not repeat `httpOnly`/`secure`, which is harmless for an empty value but inconsistent with `setSession` (`:44-58`).

Exercise: **Goal** reject deactivated users on every request. **Check** `getRequestUser` returns `null` (leading to `401` in `withRoute`) when `user.status` is not active, and you identify the status values from `prisma/schema.prisma`.
