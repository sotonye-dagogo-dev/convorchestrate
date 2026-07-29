# Lessons Learned

> **Metadata**
> - last-updated-by: update-ai-system
> - last-verified-against-code: 2026-07-29
> - staleness-policy: each entry has its own staleness — check supersedes links

> **Overview:** Practical knowledge accumulated during development — things that worked well, things that didn't, and patterns worth repeating. Different from `repair-system.md` (tracks errors); this file tracks development process insights and architectural wisdom. Uses supersedes/superseded-by links for evolving practices.

---

## Entry Format

```
## [Lesson Title]

**Context:**
[What situation this came from]

**What We Learned:**
[The insight or pattern discovered]

**Apply When:**
[When future agents/developers should use this knowledge]

**Supersedes:** [link to any prior lesson this replaces, or None]
**Superseded by:** [link to any newer lesson that replaces this, or None]
```

---

## Lessons

## `import type` Breaks NestJS Dependency Injection

**Context:**
NestJS uses `reflect-metadata` at runtime to inspect constructor parameter types. When you use `import type { QueueService }`, TypeScript erases the import entirely from the compiled JS output. At runtime, the parameter type resolves to `Function` (or `undefined`), and Nest throws "can't resolve dependencies".

**What We Learned:**
Use `import { QueueService, type SomeType }` — regular import for classes (kept at runtime), `type` prefix for pure type-only exports.

**Apply When:**
Whenever importing injectable NestJS providers. The error shows `argument Function at index [N]` instead of the actual class name. If you see this, check for `import type` on injectable providers.

**Supersedes:** None
**Superseded by:** None

## Fastify Plugin Version Compatibility

**Context:**
NestJS 10 uses Fastify 4.x internally (`@nestjs/platform-fastify@10` -> `fastify@4`). The `@fastify/*` plugin ecosystem has SemVer-major versions that target Fastify 5.

**What We Learned:**
Always check `npm view @fastify/<name> versions` and install the latest version that targets Fastify 4. Known compatible versions: `@fastify/helmet@11`, `@fastify/rate-limit@9`, `@fastify/multipart@8`.

**Apply When:**
Adding or updating any `@fastify/*` plugin.

**Supersedes:** None
**Superseded by:** None

## `__dirname` Resolution in Monorepo with ts-node-dev

**Context:**
In a monorepo where source files live in `apps/api/src/` and the `.env` is at the repo root, `resolve(__dirname, '../../.env')` is WRONG. From `convorchestrate/apps/api/src/`, `../../` only goes to `convorchestrate/apps/`.

**What We Learned:**
Need `../../../.env` to reach the project root from `src/`. Both dev (ts-node) and prod (compiled) work with `../../../.env`.

**Apply When:**
Setting up relative path resolution in monorepo packages.

**Supersedes:** None
**Superseded by:** None

## typeorm:generate is Destructive in Shared-DB Migrations

**Context:**
Running `typeorm migration:generate` on a schema that's already partially synced produces a migration that drops and recreates every constraint, index, and column.

**What We Learned:**
For targeted schema changes (add a table, alter a column), write migrations manually using `queryRunner.query()`. Only use `migration:generate` for the initial schema.

**Apply When:**
Writing TypeORM migrations for an existing schema.

**Supersedes:** None
**Superseded by:** None

## TypeORM CLI Must Load Env Separately from Nest

**Context:**
`typeorm-ts-node-commonjs` runs `src/db/data-source.ts` directly, so Nest `ConfigModule.forRoot()` does not help migration commands.

**What We Learned:**
Load the repo-root `.env` inside the data source itself and validate the value before constructing `DataSource`.

**Apply When:**
Configuring TypeORM data source files for CLI use.

**Supersedes:** None
**Superseded by:** None

## Fastify Raw Body Capture for Webhook HMAC

**Context:**
Meta webhook delivery requires HMAC-SHA256 signature validation against the raw request body. Fastify parses JSON bodies by default, consuming the raw stream, so the raw body is lost before the controller can read it.

**What We Learned:**
Register a `preParsing` hook on the Fastify instance that captures raw chunks before body parsing:

```ts
instance.addHook("preParsing", function (request, _reply, payload, done) {
    const chunks: Buffer[] = [];
    payload.on("data", (chunk) => chunks.push(chunk));
    payload.on("end", () => { request.rawBody = Buffer.concat(chunks).toString("utf-8"); });
    done();
});
```

Access via `(req as any).rawBody` in the controller.

**Apply When:**
Any controller needs access to the raw HTTP body for signature verification or logging, especially with Fastify.

**Supersedes:** None
**Superseded by:** None

## TypeORM update() Does Not Accept Partial<T> With Relation Fields

**Context:**
TypeORM's `Repository.update(id, partialEntity)` method expects `_QueryDeepPartialEntity<T>`, not `Partial<T>`. When `T` has relations decorated with `@ManyToOne`, the `Partial<T>` type includes the relation property as an entity class type (`Campaign | undefined`), which is incompatible with `_QueryDeepPartialEntity` that expects a raw column value or `() => string` subquery.

**What We Learned:**
Use `Record<string, unknown>` and cast with `as any` for `update()` calls on entities with relations:

```ts
const updates: Record<string, unknown> = { status: "sent" };
await this.repo.update(id, updates as any);
```

Alternatively, only pass column-level properties and avoid spreading the full entity.

**Apply When:**
Calling `Repository.update()` on any TypeORM entity that has `@ManyToOne` or `@OneToMany` relation decorators. The error looks like: `Types of property 'template' are incompatible.`

**Supersedes:** None
**Superseded by:** None

## try/catch with throw Creates Silent Error Swallowing

**Context:**
In `packages/meta-api`, the `parseMetaError` function used a try/catch pattern that caught its own `throw new MetaApiError(...)` and re-threw it as `MetaApiHttpError`, silently converting a structured Meta error into a generic HTTP error.

**What We Learned:**
When using try/catch for error-handling flow control in TypeScript, ensure the catch block re-throws known error types rather than replacing them. Pattern:

```ts
try {
    // parse and validate
    if (errorCondition) throw new SpecificError(...)
} catch (err) {
    if (err instanceof SpecificError) throw err  // re-throw known types
    throw new GenericError(...)  // only convert unknown errors
}
```

**Apply When:**
Any function that parses API error responses inside try/catch. Check that structured errors are not being silently converted to generic errors.

**Supersedes:** None
**Superseded by:** None

## NPM Workspace Lockfile Skips Transitive Deps of Nested Packages

**Context:**
`@fastify/multipart@8.3.1` depends on `stream-wormhole@^1.1.0` and `@fastify/busboy@^3.x`. In an npm 11 workspace with `--install-strategy` default (isolated), these transitive deps are installed at `apps/api/node_modules/@fastify/multipart` but not hoisted to `apps/api/node_modules/` or root `node_modules/`. The module cannot resolve them at runtime because Node's module resolution finds the `@fastify/multipart` directory itself but not its nested deps.

**What We Learned:**
When using npm workspaces with Docker, there are three reliable approaches for handling transitive dependencies of nested packages:
1. Add missing transitive deps to the root `package.json` as explicit dependencies
2. Copy the full workspace `apps/api/node_modules/` into the Docker image alongside root `node_modules/`
3. Use `npm install --install-strategy hoisted` (but this changes the lockfile structure and can break other builds)

We used approaches 1 + 2. The Dockerfile copies `apps/api/node_modules/` from the builder stage, and root `package.json` declares the transitive deps explicitly.

**Apply When:**
Setting up Docker builds for npm workspace monorepos. If a workspace-only dependency fails at runtime with `MODULE_NOT_FOUND`, check whether its transitive deps are installed and available in the module resolution path.

**Supersedes:** None
**Superseded by:** None

## React 18/19 Version Conflict with npm 11 Peer Dep Resolution

**Context:**
npm 11 introduced stricter peer dependency resolution. When `@radix-ui/*` and Next.js 15 declare `react@^18.2.0 || ^19.0.0`, npm 11 resolves both React 19 and React 18 into different branches of the tree. The dashboard ends up with React 19 at the top level but React 18 installed as a nested dependency of some packages. At runtime, hooks and context fail because two different React copies are loaded.

**What We Learned:**
Pin React and React-DOM to a single major version (18.3.1) and pin their `@types/*` to matching versions in the workspace's `package.json`. After changing versions, delete both `node_modules` and `package-lock.json` before running `npm install` — npm 11 caches resolution decisions aggressively and won't "fix" a stale tree without a clean install.

**Apply When:**
Running npm 11 in a monorepo with mixed React 18/19 peer dependencies. The symptom is `Invalid hook call` errors or context mismatches at runtime, even though `npm ls react` shows the right version.

**Supersedes:** None
**Superseded by:** None

## Docker Multi-Stage Build Requires Explicit Workspace node_modules Copy

**Context:**
The root `Dockerfile` uses a multi-stage build. The `builder` stage runs `npm install` and `turbo run build`. The `runner` stage copies `node_modules` from the builder. In a workspace monorepo, the root `node_modules/` contains workspace symlinks and hoisted packages, but nested node_modules at `apps/api/node_modules/` contain packages that npm didn't hoist. If the runner stage only copies root `node_modules/`, nested transitive deps are missing at runtime.

**What We Learned:**
Always copy workspace-level node_modules directories in the runner stage:
```
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/apps/api/node_modules ./apps/api/node_modules
```
Without the second line, any package installed in a workspace's nested node_modules (due to version conflicts or isolation) will be missing at runtime.

**Apply When:**
Building Docker images for npm workspace monorepos using multi-stage builds. If a package works in `docker compose run api npm start` but fails in the built image, check whether its node_modules are being copied.

**Supersedes:** None
**Superseded by:** None
