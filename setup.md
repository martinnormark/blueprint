# Project Setup Blueprint

This document describes the **end state** of the repository so it can be used as a blueprint for new projects. It is intentionally not a step-by-step file-by-file recipe — Better-T-Stack's scaffolded output drifts over time, so instead this spec describes the shape, boundaries, and rules a coding agent should drive the project toward, starting from whatever the BTS CLI produces.

The agent's job is:

1. Run the BTS command below to get a starting point.
2. Diff the starting point against the end state in this document.
3. Apply only the transformations that are missing.

---

## Origin: Better-T-Stack

Scaffold with [Better-T-Stack](https://better-t-stack.dev) (`create-better-t-stack` by @AmanVarshney01). The canonical stack choices are recorded in `bts.jsonc` at the repo root:

```sh
bun create better-t-stack@latest <project> \
  --frontend tanstack-start --backend hono --runtime bun \
  --database sqlite --orm drizzle --api orpc --auth better-auth \
  --addons turborepo --examples none \
  --git --package-manager bun --install
```

- Stack builder UI: <https://better-t-stack.dev/new>
- Config schema: `https://r2.better-t-stack.dev/schema.json` (repo pinned to version `3.27.0`)
- Add stack addons later: `bun create better-t-stack add`

BTS produces a full Bun/Turborepo monorepo with `apps/server`, `apps/web`, and the packages `api`, `auth`, `config`, `db`, `env`, `ui`. It does **not** produce the conventions, REST surface, shadcn additions, linting rules, or agent guardrails below — those are the end-state deltas this document specifies.

---

## End-State Architecture

### Apps and packages

```
apps/
├── server   Hono HTTP shell. Mounts oRPC handlers, /api/auth/* for better-auth.
│            Has NO business logic; all procedures live in packages/api.
└── web      TanStack Start + Vite + React 19 + TanStack Query + oRPC client.

packages/
├── config   Shared tsconfig.base.json. No code.
├── env      t3-env + Zod. Two exports: "./server" and "./web" (Vite-prefixed).
├── db       Drizzle schema + libSQL client + drizzle-kit scripts + migrations.
├── auth     better-auth instance wired to db + env. Exports `auth`.
├── api      oRPC router, procedures, context. Framework-agnostic (no Hono import).
└── ui       Shared shadcn/ui primitives, tailwind globals, useMountEffect hook.
```

### Dependency graph (enforced)

```
config  ← (all TS packages extend it)
env     → (none)
db      → env
auth    → db, env
api     → auth, db, env
ui      → (none; pure React/Tailwind)

apps/server → api, auth, db, env          (+ hono, orpc server + openapi)
apps/web    → api (types only), auth, env, ui  (+ tanstack-*, orpc client)
```

**Boundaries that must hold:**

- `packages/api` **never imports Hono**. The server is a thin shell; swapping HTTP frameworks should not require touching `packages/api`.
- `packages/env` splits `server.ts` and `web.ts` so server-only secrets never leak into the Vite client bundle. Web code imports `@<scope>/env/web`, server code imports `@<scope>/env/server`.
- `packages/ui` is the only place allowed to import `useEffect` directly (in its `use-mount-effect.ts` helper). See [useEffect ban](#useeffect-ban).
- `apps/web` imports `AppRouter` from `@<scope>/api` as a **type only** — the client never pulls in server runtime code.
- Workspace packages use `workspace:*`. Shared third-party versions use `"catalog:"`, pinned in root `package.json` under `workspaces.catalog`.

### Runtime surface

`apps/server` mounts three oRPC handlers against one `appRouter`, plus the better-auth handler. This is the expected runtime shape:

| Route | Handler | Purpose |
|---|---|---|
| `/api/auth/*` | `auth.handler` | better-auth endpoints |
| `/rpc/*` | `RPCHandler` | Binary typed protocol — used by the web app |
| `/api-docs/*` | `OpenAPIHandler` + `OpenAPIReferencePlugin` | Scalar docs UI + OpenAPI spec |
| `/v1/*` | `OpenAPIHandler` | Public REST API |

The three oRPC handlers live in a single catch-all middleware that sets `c.res = c.newResponse(...)` then calls `await next()` (never early-return) so Hono response-phase middleware (compress, etag, rate-limiting) still runs.

### Router convention (`packages/api/src/routers/*`)

Each resource ships as a trio exported from one file:

- `const <name>Tag = { name, description }`
- `const <name>Router = o.tag(tag.name).prefix("/v1/<name>").router({ ... })` with procedures defined using `.route({ method, path })` with **relative** paths (`/`, `/{id}`)
- Path params require a matching `.input(z.object({ id: z.coerce.number()... }))` — OpenAPI generation fails without this

`packages/api/src/routers/index.ts` assembles `appRouter`, exports `apiTags: Tag[]`, and exports `type AppRouter = typeof appRouter`. The `widgets` router in the repo is the canonical reference example.

### Procedure builders (`packages/api/src/index.ts`)

- `publicProcedure` — no auth
- `protectedProcedure` — applies a middleware that throws `ORPCError("UNAUTHORIZED")` when `context.session?.user` is absent
- Context is created per request from the Hono `Context` (reads headers, calls `auth.api.getSession`)

---

## End-State Deltas (what to add on top of BTS)

These are the transformations that distinguish this repo from a vanilla BTS scaffold. A coding agent should check each one against the current starting point and apply only what is missing or different.

### 1. Public REST surface + OpenAPI docs

**End state:** three oRPC handlers mounted as described above, with `/api-docs` split out from `/v1/*` so the docs UI has its own prefix, and `apiTags` fed into the OpenAPI spec `tags` array.

BTS may only give you `/rpc` and a combined `/api-reference`. Reshape the server catch-all to the three-handler layout, and pass `apiTags` through to `OpenAPIReferencePlugin.specGenerateOptions`.

### 2. REST router pattern with `.tag()` / `.prefix()` / `.route()`

**End state:** every resource router uses the tag/prefix/route convention (see `packages/api/src/routers/widgets.ts` for the canonical shape). BTS provides the `appRouter` skeleton with `healthCheck` and `privateData` but no example resource — add the widgets router or an equivalent to establish the pattern before the first real resource.

### 3. Shared UI with shadcn targeting `packages/ui`

**End state:** shared primitives live in `packages/ui/src/components/*`. Both `packages/ui/components.json` and `apps/web/components.json` exist, pointing at the same `globals.css` in the UI package. `apps/web/src/index.css` contains one line: `@import "@<scope>/ui/globals.css";`.

Add primitives shared across apps from the repo root into the UI package:

```sh
bunx shadcn@latest add <component…> -c packages/ui
```

Add app-local blocks (not shared) from `apps/web` without `-c`.

shadcn CLI: <https://ui.shadcn.com/docs/cli>

### 4. Ultracite (Biome preset) + husky + VSCode

**End state:**

- `biome.jsonc` extends `ultracite/biome/core`, `ultracite/biome/react`, `ultracite/biome/vitest`, `ultracite/biome/remix` plus the `noRestrictedImports` override for `useEffect`.
- `.husky/pre-commit` runs `bun x ultracite fix` on staged files.
- `.vscode/settings.json` sets Biome as default formatter for JS/TS/JSON/CSS/MD.
- Root `package.json` has `"check": "ultracite check"` and `"fix": "ultracite fix"` scripts.
- `packages/db/biome.json` is a non-root Biome config that excludes `src/migrations/**` from linting.
- All tsconfigs have strict flags on: `strict`, `noUncheckedIndexedAccess`, `noUnusedLocals`, `noUnusedParameters`, `verbatimModuleSyntax`, `isolatedModules`.

Run once to bootstrap all of this:

```sh
bunx ultracite@latest init
```

Then add the `noRestrictedImports` override (see next section) and the `packages/db/biome.json` migrations exclusion manually. Ultracite docs: <https://ultracite.ai>.

### 5. useEffect ban

**End state:** direct `useEffect` imports from `"react"` are a lint error everywhere except `packages/ui/src/hooks/use-mount-effect.ts`.

- `biome.jsonc` declares `style.noRestrictedImports` blocking `useEffect` from `react` with a message pointing at `useMountEffect` and `AGENTS.md`.
- `packages/ui/src/hooks/use-mount-effect.ts` is a thin wrapper around `useEffect(effect, [])` with `biome-ignore` comments for the lint rule and `useExhaustiveDependencies`.
- Any existing `useEffect` in scaffolded code (typically `sidebar.tsx`, `use-mobile.ts`) is migrated.

**Rules for writing React code in this repo** (also captured in `AGENTS.md`):

- Prefer **derived state** — compute values inline instead of writing `useEffect(() => setX(f(y)), [y])`.
- Put work triggered by user action in **event handlers**, not in effects keyed off a flag.
- Use **TanStack Query / oRPC client utils** for fetching (never `useEffect` + `fetch` + `setState`).
- Use **`useMountEffect`** only for genuine one-time external sync (DOM APIs, third-party widget lifecycle, browser subscriptions). Import from `@<scope>/ui/hooks/use-mount-effect`.
- Use `key={id}` to **reset state by remount** instead of effects that reset on ID change.
- If a callback needs the latest prop/state but should subscribe only once, put the callback in a `useRef` and read `ref.current` inside a `useMountEffect`.

### 6. AGENTS.md

**End state:** a root `AGENTS.md` documenting the stack, commands, the three-handler architecture, resource-adding recipe, package boundaries, gotchas, and the useEffect rules. This is the primary onboarding doc for coding agents working in the repo.

### 7. Skills (skills.sh)

**End state:** `.agents/skills/<name>/` directories with installed skills, plus a reproducible `skills-lock.json` at the repo root. The Ultracite skill is installed and used by agents to apply the lint/format rules consistently. See [Agent Skills](#agent-skills) below.

### 8. Environment variables

No `.env.example` ships with BTS. Create:

**`apps/server/.env`**

```env
DATABASE_URL=file:../../local.db        # or libSQL URL + auth token
BETTER_AUTH_SECRET=<32+ char random>
BETTER_AUTH_URL=http://localhost:3000
CORS_ORIGIN=http://localhost:3001
NODE_ENV=development
```

**`apps/web/.env`**

```env
VITE_SERVER_URL=http://localhost:3000
```

`packages/db/drizzle.config.ts` reads `apps/server/.env` via `dotenv.config({ path: "../../apps/server/.env" })` so drizzle-kit commands work from the package directory.

---

## Agent Skills

Skills are managed by the **`skills` CLI** from [skills.sh](https://skills.sh) — a package manager for AI agent skills (Claude Code, Codex, OpenCode, etc). Installed skills live in `.agents/skills/<name>/` and are recorded (source + content hash) in `skills-lock.json` at the repo root.

Docs: <https://skills.sh/docs> · CLI reference: <https://skills.sh/docs/cli>

### Installed in this repo

| Skill | Source | Purpose |
|---|---|---|
| `ultracite` | `haydenbleasel/ultracite` (GitHub) | Guides the Ultracite/Biome lint + format workflow and enforces the code standards documented in `AGENTS.md`. |

### Using the CLI

No install required — run via `bunx` (or `npx`):

```sh
bunx skills add <owner>/<skill>
```

Source formats accepted:

- `owner/skill` — GitHub shorthand (e.g. `haydenbleasel/ultracite`)
- Full git URL
- Registry name from the [skills.sh](https://skills.sh) directory

### Re-adding skills after clone

```sh
bunx skills add haydenbleasel/ultracite
```

Writes into `.agents/skills/ultracite/` and updates `skills-lock.json` with the computed content hash so the install is reproducible.

### Telemetry opt-out

```sh
DISABLE_TELEMETRY=1 bunx skills add <source>
```

### Finding more skills

Browse <https://skills.sh> for the official registry. Add any that match the stack (e.g. Drizzle, TanStack, Hono, better-auth), for example:

```sh
bunx skills add vercel-labs/agent-skills
```

---

## Suggested transformation order

The deltas are independent enough to apply in any order, but this sequence minimizes churn:

1. **Bootstrap linting first** (`bunx ultracite@latest init`) so subsequent edits format automatically.
2. **Apply strict tsconfig flags** across all packages (extend `@<scope>/config/tsconfig.base.json`).
3. **Wire the three-handler server surface** in `apps/server/src/index.ts`.
4. **Add the router convention** and a reference resource (the `widgets` router is the template).
5. **Set up shadcn shared package** and add any needed primitives into `packages/ui`.
6. **Add the useEffect ban** (biome rule + `useMountEffect` hook) and migrate any scaffolded `useEffect` usages.
7. **Write `AGENTS.md`** capturing the above as the canonical rules for the repo.
8. **Install agent skills** (`bunx skills add haydenbleasel/ultracite`, plus any others).

---

## Daily commands (end state)

```sh
bun install                 # first-time / after pulls
bun run db:local            # optional: local Turso dev server
bun run db:push             # apply Drizzle schema
bun run dev                 # runs server (3000) + web (3001) via turbo
bun run check-types         # typecheck all packages
bun run fix                 # ultracite format + auto-fix
bun run build               # turbo build (tsdown for server, vite for web)
```

Turbo filters for per-package work: `turbo -F web dev`, `turbo -F @<scope>/db db:studio`, etc.

---

## References

- Better-T-Stack: <https://better-t-stack.dev>
- TanStack Start: <https://tanstack.com/start>
- Hono: <https://hono.dev>
- oRPC: <https://orpc.unnoq.com>
- better-auth: <https://www.better-auth.com>
- Drizzle ORM / Kit: <https://orm.drizzle.team>
- libSQL / Turso: <https://docs.turso.tech>
- t3-env: <https://env.t3.gg>
- shadcn/ui CLI: <https://ui.shadcn.com/docs/cli>
- Ultracite: <https://ultracite.ai>
- Biome: <https://biomejs.dev>
- Turborepo: <https://turbo.build/repo/docs>
- Skills: <https://skills.sh> · <https://skills.sh/docs/cli>
