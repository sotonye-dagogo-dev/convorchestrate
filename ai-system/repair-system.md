# Repair System — Error Knowledge Base

> **Metadata**
> - last-updated-by: update-ai-system
> - last-verified-against-code: 2026-07-29
> - staleness-policy: individual entries may be stale if the code has changed around them — verify fix still applies before reusing

> **Overview:** Living knowledge base of errors encountered during development, their root causes, and how they were fixed. Agents must search this before diagnosing new errors and log every fixed bug to prevent recurrence.

---

## How to Use

- **Before debugging:** Search this file for patterns matching the current error
- **After fixing a bug:** Add an entry using the template below
- **If a fix no longer applies:** Mark the entry as `[SUPERSEDED]` and link to the new entry

---

## Error Log

### TypeORM CLI Does Not Inherit Nest Config Loading

**Symptom:**
`npm run migration:run` fails with `SASL: SCRAM-SERVER-FIRST-MESSAGE: client password must be a string` or `DATABASE_URL is required for TypeORM migrations`.

**Root Cause:**
The TypeORM CLI loads `src/db/data-source.ts` directly and does not run Nest `ConfigModule.forRoot()`. If the data source depends on environment variables but does not load `.env` itself, `process.env.DATABASE_URL` can be undefined.

**Fix Applied:**
Load the repo-root `.env` inside `apps/api/src/db/data-source.ts` before creating the `DataSource`, and fail fast if `DATABASE_URL` is missing.

**Prevention:**
Treat TypeORM CLI entrypoints as standalone boot paths. Any env-dependent data source should load or validate its own configuration instead of relying on Nest app startup.

**Files Affected:**
- apps/api/src/db/data-source.ts

**Date:** 2026-06-24
**Status:** Active

### API Demo Route Missing Global Prefix

**Symptom:**
`POST /api/demo/seed` returns `404` even though the demo controller exists and the dashboard documentation calls that URL.

**Root Cause:**
The Nest API did not register a global `api` prefix, while the dashboard and setup guide already assumed all API routes lived under `/api`.

**Fix Applied:**
Add `app.setGlobalPrefix("api")` in `apps/api/src/main.ts` so the runtime route shape matches the dashboard and docs.

**Prevention:**
Keep one canonical route prefix convention for the API and use it consistently in the server bootstrap, dashboard client, and setup docs.

**Files Affected:**
- apps/api/src/main.ts

**Date:** 2026-06-24
**Status:** Active

### [SUPERSEDED] WhatsApp Browser Lock Blocks API Startup

**Symptom:**
API startup fails with `The browser is already running for ...\\wa-sessions\\session. Use a different userDataDir or stop the running browser first.`

**Root Cause:**
`MessagingService.onModuleInit()` awaited `WwjsAdapter.initialize()`. A stale `whatsapp-web.js` LocalAuth profile lock turned a WhatsApp runtime problem into a fatal API bootstrap failure.

**Fix Applied (Original):**
Catch adapter initialization errors in `MessagingService` so the API can boot without WhatsApp. Log the failure and continue serving demo/admin routes.

**Status:** SUPERSEDED — whatsapp-web.js adapter has been removed entirely (R1 rebase). Meta Cloud API does not use browser sessions. This error will not occur.

**Files Affected (Original):**
- apps/api/src/modules/messaging/messaging.service.ts

**Date:** 2026-06-24
**Status:** SUPERSEDED by wa-manager rebase (R1)

### parseMetaError Swallows Structured Errors with try/catch

**Symptom:**
When the Meta API returns a structured error (with `error.code`, `error.type`, etc.), the `parseMetaError` function in `packages/meta-api` was converting it to a generic `MetaApiHttpError` instead of a rich `MetaApiError`.

**Root Cause:**
The parseMetaError function used a try/catch block that caught its own `throw new MetaApiError(...)` and re-threw it as `MetaApiHttpError`:

```ts
function parseMetaError(body: string, status: number): never {
  try {
    const parsed: MetaErrorResponse = JSON.parse(body)
    if (parsed.error?.code) {
      throw new MetaApiError(...)  // caught by catch block below
    }
  } catch {
    if (status >= 300) {
      throw new MetaApiHttpError(status, body)  // replaces MetaApiError
    }
  }
  throw new MetaApiHttpError(status, body)
}
```

**Fix Applied:**
Restructured the function with the catch block re-throwing known types:

```ts
function parseMetaError(body: string, status: number): never {
  if (status < 300) throw new MetaApiHttpError(status, body)
  try {
    const parsed: MetaErrorResponse = JSON.parse(body)
    if (parsed.error?.code) {
      throw new MetaApiError(...)
    }
  } catch (err) {
    if (err instanceof MetaApiError) throw err
    throw new MetaApiHttpError(status, body)
  }
  throw new MetaApiHttpError(status, body)
}
```

**Prevention:**
When using try/catch for flow control, ensure the catch block re-throws known error types rather than silently converting them.

**Files Affected:**
- packages/meta-api/src/meta-api.client.ts

**Date:** 2026-07-01
**Status:** Active

### Nested Workspace Transitive Deps Missing at Docker Runtime

**Symptom:**
API container crashes at startup with:
```
Error: Cannot find module '@lukeed/ms'
Require stack:
- /app/apps/api/node_modules/@fastify/rate-limit/index.js
```
Or:
```
Error: Cannot find module '@fastify/multipart'
Require stack:
- /app/apps/api/dist/main.js
```

**Root Cause:**
`@fastify/multipart@8.3.1` and `@fastify/rate-limit@9.1.0` are installed in `apps/api/node_modules/` (nested). Their transitive deps (`@fastify/busboy`, `@lukeed/ms`, `stream-wormhole`, `@fastify/deepmerge`, `@fastify/error`) are either nested inside `apps/api/node_modules/@fastify/multipart/` or not hoisted to the resolution path. The Docker runner stage only copied root `node_modules/`, not workspace nested ones.

**Fix Applied:**
1. Added missing transitive deps to root `package.json` under `dependencies`:
   - `@fastify/busboy@^3.2.0`
   - `@lukeed/ms@^2.0.2`
   - `stream-wormhole@^2.0.1`
2. Added `COPY --from=builder /app/apps/api/node_modules ./apps/api/node_modules` to the Dockerfile runner stage

**Prevention:**
When using npm workspaces with multi-stage Docker builds, explicitly copy workspace-level node_modules directories and hoist any transitive deps that are version-locked to specific ranges.

**Files Affected:**
- Dockerfile — added COPY for apps/api/node_modules
- package.json — added @fastify/busboy, @lukeed/ms, stream-wormhole as root deps
- docker-compose.yml — added Meta env vars

**Date:** 2026-07-29
**Status:** Active

### Dashboard Build Fails — React 18/19 Version Conflict

**Symptom:**
Dashboard build produces React "Invalid hook call" errors at runtime, or `Module not found: Can't resolve 'react'` during build.

**Root Cause:**
npm 11 resolves the `react@^18.2.0 || ^19.0.0` peer dependency range from `@radix-ui/*` and Next.js 15 by installing both React 18 and 19 into different parts of the node_modules tree. The workspace symlinked `react` resolves to the root's React 19, while some packages resolve their own copy of React 18.

**Fix Applied:**
Downgraded `react` from `19.0.0` to `18.3.1` and `react-dom` to the same in `apps/dashboard/package.json`. Pinned `@types/react` and `@types/react-dom` to matching 18.x versions. Did a clean install (deleted lockfile + node_modules).

**Prevention:**
Pin React to a single major version (18.x) when using npm 11 with packages that declare `^18.2.0 || ^19.0.0` peer deps. Avoid letting npm resolve both majors.

**Files Affected:**
- apps/dashboard/package.json — react/react-dom pinned to 18.3.1, types pinned to exact 18.x

**Date:** 2026-07-29
**Status:** Active

### Dashboard Dockerfile — Turbo Workspace Duplication Error

**Symptom:**
Dashboard Docker build fails during `npm install` stage with:
```
npm ERR! Invalid workspace configuration: package "convorchestrate" has workspace "apps/*"...
Duplicate workspace "packages/*" found
```

**Root Cause:**
The Dockerfile used `COPY packages/*/package.json packages/*/package.json` which is not a valid glob pattern for the COPY instruction in Docker. It copies all package.json files to the `packages/*/package.json` path, creating overlapping directory structures.

**Fix Applied:**
Changed to explicit per-package COPY lines (similar to what the root Dockerfile does):
```
COPY packages/core/package.json packages/core/package.json
COPY packages/meta-api/package.json packages/meta-api/package.json
COPY packages/schemas/package.json packages/schemas/package.json
COPY packages/utils/package.json packages/utils/package.json
```

**Prevention:**
Docker COPY does not support `*` glob expansion in the destination path like shell globbing does. Always use explicit COPY lines for monorepo workspace packages.

**Files Affected:**
- apps/dashboard/Dockerfile

**Date:** 2026-07-29
**Status:** Active

### Dashboard Dockerfile — Standalone Output Path Mismatch

**Symptom:**
Dashboard container starts but immediately exits with:
```
node:internal/modules/cjs/loader: MODULE_NOT_FOUND: Cannot find module '/app/server.js'
```

**Root Cause:**
Next.js standalone output mode generates the server at `apps/dashboard/server.js` (matching the monorepo structure), but the CMD was `node server.js` (relative to WORKDIR /app).

**Fix Applied:**
Changed CMD to `node apps/dashboard/server.js`.

**Prevention:**
When Next.js `output: "standalone"` is used in a monorepo, the built server.js path preserves the monorepo directory structure. Check the output path in the final standalone `.next` directory.

**Files Affected:**
- apps/dashboard/Dockerfile

**Date:** 2026-07-29
**Status:** Active

### Missing Meta Environment Variables in Docker Compose

**Symptom:**
API container crashes at startup after successful module resolution:
```
Configuration key "META_PHONE_NUMBER_ID" does not exist
```

**Root Cause:**
The `MessagingModule` uses `ConfigService.getOrThrow<string>("META_PHONE_NUMBER_ID")` for the MetaApiClient factory. The `.env` file defines these vars, but `docker-compose.yml` was not passing them to the api service container.

**Fix Applied:**
Added all Meta environment variables to the api service's environment block in `docker-compose.yml`, with `${VAR:-placeholder}` defaults:
- `META_PHONE_NUMBER_ID`, `META_ACCESS_TOKEN`, `META_APP_SECRET`, `META_APP_ID`, `META_WABA_ID`

**Prevention:**
All env vars used by NestJS `ConfigModule` must be explicitly declared in `docker-compose.yml` or loaded via `env_file`. Use `getOrThrow` sparingly — consider `get` with fallback for non-critical config.

**Files Affected:**
- docker-compose.yml

**Date:** 2026-07-29
**Status:** Active

### Stale infrastructure/docker-compose.yml — References Missing Dockerfile

**Symptom:**
No explicit error (not actively used), but `infrastructure/docker-compose.yml` points to `apps/api/Dockerfile` which was moved to root `Dockerfile`. Building from this file would fail with `failed to solve: file not found`.

**Root Cause:**
During the R1 wa-manager rebase, the API Dockerfile was moved from `apps/api/Dockerfile` to the root `Dockerfile`, and the active docker-compose.yml was moved to the root directory. The stale copy in `infrastructure/` was never updated or deleted.

**Fix Applied:**
Deleted `infrastructure/docker-compose.yml` and the now-empty `infrastructure/` directory. Updated `SETUP.md` and `README.md` to use root `docker-compose.yml` instead.

**Prevention:**
When relocating build configuration files, always check for stale copies or aliases in related directories. After the move, `grep` for the old path across the repo.

**Files Affected:**
- infrastructure/docker-compose.yml — deleted
- SETUP.md — replaced `docker compose -f infrastructure/docker-compose.yml` with `docker compose`
- README.md — updated structure layout
- ai-system/planning/task-queue.md — updated repo layout

**Date:** 2026-07-29
**Status:** Active

---

## Known Error Patterns

### React / Next.js

**Hydration Mismatch**
- Symptom: `Hydration failed because the initial UI does not match what was rendered on the server`
- Cause: Browser-only logic (window, localStorage, Date.now()) running during server render
- Fix: Wrap in `useEffect` or use `dynamic(() => import(...), { ssr: false })`
- Prevention: Never access browser APIs outside useEffect in components

**Missing Key Prop**
- Symptom: `Each child in a list should have a unique "key" prop`
- Cause: `.map()` rendering without a stable unique key
- Fix: Add `key={item.id}` — use a stable unique ID, not the array index

### Node.js / Backend

**Unhandled Promise Rejection**
- Symptom: Server crashes silently or logs `UnhandledPromiseRejectionWarning`
- Cause: async function missing try/catch or `.catch()` not attached to promise
- Fix: Wrap async route handlers in try/catch; use a global async error wrapper
- Prevention: Always release DB connections in finally, not just success path

**Database Connection Pool Exhausted**
- Symptom: Requests hang indefinitely under load
- Cause: Connection pool limit too low or connections not released
- Fix: Increase pool size; ensure `client.release()` in finally blocks
- Prevention: Always release connections in finally

### Configuration / Environment

**Missing Environment Variable**
- Symptom: `undefined` values in production, features silently broken
- Cause: Variable defined in `.env.local` but not in production environment
- Fix: Add to deployment environment variables
- Prevention: Add a startup validation check that throws if required env vars are missing
