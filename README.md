# Agentic Engineering Blueprint

Point your coding agent at `setup.md` to scaffold a new web app with an opinionated stack, ready for agentic coding.

Technology choices:
- **Monorepo**: Bun + Turborepo for unified dev experience and fast builds
- **Server**: Hono for minimal HTTP shell, mounting oRPC handlers and serving API docs
- **Web**: TanStack Start with Vite and React 19 for a modern frontend
- **API**: oRPC for type-safe APIs with a clear separation from the HTTP layer
- **Database**: Drizzle ORM with libSQL (SQLite) for a simple, file-based database
- **Auth**: better-auth for flexible authentication and authorization
- **Env**: t3-env with Zod for robust environment variable management
- **UI**: shadcn/ui for a set of pre-built, customizable components

**Depends on Bun to be installed**

## Capability packs
Better T Stack provides the baseline. Your coding agent adjust towards a desired end-state by adding capability packs on top. You can define your own packs, but here are some examples:
- **REST API pack**: Add OpenAPIHandler to server, generate OpenAPI spec from oRPC router, and serve API docs with Swagger UI or Redoc.
- **GraphQL pack**: Add GraphQL server (e.g. Apollo Server) to the server app, generate schema from oRPC router, and serve GraphQL Playground.
- **Testing pack**: Add testing frameworks (e.g. Jest, Testing Library), set up test structure and example tests for server and web apps.
- **CI/CD pack**: Add configuration files for CI/CD pipelines (e.g. GitHub Actions) to run tests, build apps, and deploy to hosting platforms.


## Project structure

```
blueprint/  (Bun + Turborepo monorepo)
│
├── apps/
│   │
│   ├── server/                    Hono HTTP shell, port 3000 mounts oRPC handlers:
│   │   │                            /rpc       → RPCHandler (binary, for web)
│   │   │                            /api-docs  → Scalar UI + OpenAPI spec
│   │   │                            /v1/*      → OpenAPIHandler (REST)
│   │   └── (thin wrapper; no business logic)
│   │
│   └── web/                       TanStack Start + Vite + React 19, port 3001
│
└── packages/
    │
    ├── config/                    Shared tsconfig.base.json
    │
    ├── env/                       t3-env + Zod
    │                                server.ts → DATABASE_URL, BETTER_AUTH_*
    │                                web.ts    → VITE_* vars
    │
    ├── db/                        Drizzle ORM + libSQL (SQLite)
    │                                schema, migrations, client
    │
    ├── auth/                      better-auth config
    │
    ├── api/                       oRPC — source of truth for the API
    │                                routers/<resource>.ts
    │                                publicProcedure / protectedProcedure
    │                                (knows nothing about Hono)
    │
    └── ui/                        Shared shadcn/ui components
                                     hooks/use-mount-effect.ts
                                     (only file allowed to import useEffect)
```

## Semantic dependency tree

```
                          ┌──────────────┐
                          │    config    │  (tsconfig only)
                          └──────────────┘
                          ┌──────────────┐
                          │     env      │  ← leaf: Zod-validated env
                          └──────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
              ┌──────────┐              ┌──────────┐
              │    db    │              │   ui     │ (standalone)
              └────┬─────┘              └────┬─────┘
                   │                         │
                   ▼                         │
              ┌──────────┐                   │
              │   auth   │                   │
              └────┬─────┘                   │
                   │                         │
                   ▼                         │
              ┌──────────┐                   │
              │   api    │  (oRPC router)    │
              └────┬─────┘                   │
                   │                         │
         ┌─────────┴──────────┐              │
         ▼                    ▼              │
   ┌───────────┐         ┌─────────┐         │
   │  server   │         │   web   │ ◄───────┘
   │  (Hono)   │         │ (TSS)   │
   └───────────┘         └─────────┘
         ▲                    ▲
         │                    │
         └────── depends on db, auth, env directly too
```


## Data flow at runtime

```
  Browser ──► apps/web ──► oRPC client
                            │
                            ▼
                    apps/server (Hono)
                            │  routes /rpc, /v1/*, /api-docs
                            ▼
                    packages/api (appRouter)
                       │        │
                       ▼        ▼
                  auth mw   procedure handler
                       │        │
                       └────┬───┘
                            ▼
                       packages/db (Drizzle)
                            │
                            ▼
                         libSQL
```