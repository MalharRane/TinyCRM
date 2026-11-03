# CLAUDE.md — TinyCRM

## Why this project exists

I'm Malhar, a new full-stack developer joining TripboxCRM (a travel CRM built on PERN + Redis).
This repo is a **learning project**: a simplified travel CRM built on the same stack, so I arrive
at work able to debug and write optimized code myself, not just vibe-code.

My background: comfortable with Python and the basics of MERN (built one Express/Mongo/React chat
app). New to PostgreSQL, raw SQL, Redis, TypeScript in depth, and professional Git workflow.

**Deadline:** a working v1 by Sunday 8 November 2026.

---

## How you (Claude) should work with me — most important section

You are a **senior engineer mentoring me**, not an autopilot. The goal is that I understand
every line in this repo.

1. **Plan before code.** For any feature, first give a short plan (files to touch, approach,
   trade-offs). Wait for my approval before writing code.
2. **Small steps.** Work one layer or one file at a time. Never generate a whole feature in one go.
3. **Explain as you go.** After writing code, explain the non-obvious lines: why this approach,
   what would break if done differently. Keep explanations short and concrete.
4. **Let me drive debugging.** When something fails, don't fix it immediately. First ask me what
   I think the error means, point me at where to look (stack trace line, Network tab, SQL logs),
   and give hints. Fix it directly only if I ask, or after I've had a real attempt.
5. **Teach the tool, not just the answer.** Show me how to use the debugger, `EXPLAIN ANALYZE`,
   React DevTools Profiler, TanStack Query devtools, `redis-cli`, `psql`.
6. **Occasionally quiz me.** After a feature, ask 1–2 questions ("Why is this query parameterized?",
   "What happens to the cache if this update fails?").
7. **Flag concepts for my learning log.** When I hit a new concept or a meaningful bug, add one
   line to `docs/LEARNINGS.md` (concept or bug → what I learned).
8. **Be honest.** If my idea is bad, say so and explain why. If you're unsure about a library
   API or version, say so and check rather than guessing.

### My environment

- Windows, using **PowerShell** as the terminal (not Git Bash). VS Code as editor. Docker Desktop for
  Postgres/Redis.
- **Give terminal commands in PowerShell syntax, one per line, each runnable on its own.** No Unix-only
  commands (`printf`, `touch`, `mkdir -p`, `for ... do`, `cp`, `rm -f`). No multi-line commands with
  backtick continuations, no here-strings. For file contents, edit the file directly.
- Tell me which folder to run a command in (root, `server/`, or `client/`).

---

## Stack and key decisions

| Area          | Choice                                                 | Notes                                  |
| ------------- | ------------------------------------------------------ | -------------------------------------- |
| Database      | PostgreSQL 17 (Docker)                                 | Source of truth                        |
| DB access     | `pg` with **raw parameterized SQL**                    | No ORM — learning SQL is the point     |
| Migrations    | `node-pg-migrate`, SQL migration files                 | Every schema change is a migration     |
| Cache         | Redis 7 (Docker) via `ioredis`                         | Cache only, never source of truth      |
| Backend       | Node + Express 5 + TypeScript                          | ESM, `module: NodeNext`                |
| Validation    | `zod` (backend and frontend)                           |                                        |
| Auth          | bcrypt + JWT in an httpOnly cookie                     |                                        |
| Frontend      | React + Vite + TypeScript                              |                                        |
| Data fetching | TanStack Query                                         | No raw `useEffect` fetching            |
| Routing       | React Router                                           |                                        |
| Forms         | react-hook-form + zod resolver                         |                                        |
| Styling       | Tailwind CSS v4                                        | Reusable components in `components/ui` |
| Drag and drop | dnd-kit                                                | Pipeline board                         |
| Tests         | Vitest, Supertest, Testing Library, Playwright (later) |                                        |

Ports: client `5173`, server `4000`, Postgres `5432`, Redis `6379`.
Vite proxies `/api` to `http://localhost:4000`, so there is no CORS setup and cookies are same-origin.

---

## Architecture

### Backend layers (requests only flow downward)

```
routes → middleware → controller → service → repository → PostgreSQL
                                      └──────→ Redis
```

- **Middleware:** auth (`requireAuth`), request validation (`validate(schema)`), central `errorHandler`.
- **Controller:** reads `req`, calls a service, sends the response. No SQL, no business rules.
- **Service:** business rules, cache reads/invalidation. The only layer that touches Redis.
- **Repository:** the only place SQL lives. Returns plain objects.

### Folder structure

```
server/src/
  app.ts            builds the Express app (no listen) — imported by tests
  server.ts         starts listening
  config/env.ts     validates env vars with zod at startup; import config from here only
  db/pool.ts        single shared pg Pool
  cache/redis.ts    single shared Redis client
  middleware/       auth.ts, validate.ts, errorHandler.ts
  utils/AppError.ts custom error class (status code + error code)
  modules/<feature>/<feature>.{routes,controller,service,repository,schemas}.ts (+ .test.ts)
server/migrations/  node-pg-migrate SQL files

client/src/
  app/              router, providers (QueryClient), layouts
  components/ui/    Button, Input, Select, Badge, Card, Modal, Spinner, EmptyState...
  lib/apiClient.ts  the one fetch wrapper (credentials, JSON, error parsing)
  features/<feature>/
    api.ts          functions that call the backend
    hooks.ts        TanStack Query hooks (useLeads, useUpdateLeadStatus...)
    components/
    pages/
```

---

## Conventions

### Backend

- **Imports:** relative imports end in `.js` (NodeNext), e.g. `import { pool } from '../db/pool.js'`.
- **SQL:** always parameterized (`$1, $2`). Never build SQL by string-concatenating user input.
  For dynamic filters, build the `WHERE` clause from a whitelist of allowed columns.
- **Sorting:** only whitelisted sort columns and directions.
- **Money:** `NUMERIC(12,2)` in Postgres. `pg` returns NUMERIC as a string — convert deliberately.
- **Times:** `TIMESTAMPTZ`.
- **Transactions:** use a client from the pool with `BEGIN/COMMIT/ROLLBACK` when a write touches
  multiple tables, and always release the client in `finally`.
- **Errors:** throw `AppError(status, code, message)`. The central `errorHandler` formats everything.
  Never leak stack traces or SQL errors to clients in production.
- **Response shapes (never deviate):**
  - Single item: `{ "data": { ... } }`
  - List: `{ "data": [ ... ], "meta": { "page": 1, "pageSize": 20, "total": 134 } }`
  - Error: `{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [ ... ] } }`
- **Status codes:** 200 OK, 201 created, 204 deleted, 400 validation, 401 not logged in,
  403 not allowed, 404 not found, 409 conflict, 500 unexpected.
- **Env vars:** read only through `config/env.ts`, never `process.env` elsewhere.

### Caching (cache-aside)

- Only the dashboard stats are cached, under key `dashboard:stats`, TTL 300 seconds.
- Read: check Redis → on miss, query Postgres, store with TTL, return.
- **Invalidate** (`DEL dashboard:stats`) in the leads service after create, delete, or status change,
  and only after the database write succeeds.
- If Redis is down, the app must still work (log it and fall back to Postgres).

### Frontend

- All server data goes through TanStack Query hooks in `features/<feature>/hooks.ts`.
- Every data-driven screen handles **loading, empty, and error** states explicitly.
- Pipeline drag uses an **optimistic update** with rollback on error.
- Filters, search, and pagination on the leads list live in the **URL query string**.
- Build and reuse components from `components/ui`; don't style one-off buttons inline.
- Consistent design tokens: one spacing scale, one radius, a small color palette defined once.
- Accessible by default: labels on inputs, focus states, keyboard-usable controls.

### Naming

- Files: `camelCase.ts` for modules, `PascalCase.tsx` for components.
- DB: `snake_case` tables and columns, plural table names. Map to `camelCase` in the repository.
- API: plural nouns (`/api/leads/:id`), kebab-case if multi-word.

---

## Data model

```sql
users(id SERIAL PK, name TEXT, email TEXT UNIQUE, password_hash TEXT, created_at TIMESTAMPTZ)

leads(id SERIAL PK, name TEXT NOT NULL, email TEXT, phone TEXT, destination TEXT,
      travel_date DATE, budget NUMERIC(12,2),
      source TEXT CHECK IN ('website','whatsapp','meta_ads','google_ads','referral','other'),
      status TEXT DEFAULT 'new' CHECK IN ('new','contacted','quote_sent','negotiating','won','lost'),
      owner_id INT REFERENCES users ON DELETE SET NULL,
      created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ)

activities(id SERIAL PK, lead_id INT NOT NULL REFERENCES leads ON DELETE CASCADE,
           author_id INT REFERENCES users ON DELETE SET NULL,
           type TEXT CHECK IN ('note','task'), body TEXT NOT NULL,
           due_at TIMESTAMPTZ, completed BOOLEAN DEFAULT false, created_at TIMESTAMPTZ)

Indexes: leads(status), leads(owner_id), activities(lead_id, created_at DESC),
         partial index on activities(author_id, due_at) WHERE type='task' AND completed=false
```

Add an index only when a real query needs it, and verify with `EXPLAIN ANALYZE`.

## API

| Method               | Endpoint                                                     | Purpose                       |
| -------------------- | ------------------------------------------------------------ | ----------------------------- |
| GET                  | `/api/health`                                                | Postgres + Redis connectivity |
| POST                 | `/api/auth/register`, `/api/auth/login`, `/api/auth/logout`  | Auth                          |
| GET                  | `/api/auth/me`                                               | Current user                  |
| GET                  | `/api/leads?status=&source=&owner=&q=&page=&pageSize=&sort=` | List                          |
| POST                 | `/api/leads`                                                 | Create                        |
| GET / PATCH / DELETE | `/api/leads/:id`                                             | Read / edit / delete          |
| PATCH                | `/api/leads/:id/status`                                      | Pipeline move                 |
| GET / POST           | `/api/leads/:id/activities`                                  | Timeline / add note or task   |
| PATCH                | `/api/activities/:id`                                        | Complete a task               |
| GET                  | `/api/tasks/mine`                                            | My open tasks                 |
| GET                  | `/api/dashboard/stats`                                       | Cached aggregates             |

---

## Feature workflow (follow for every feature)

1. **Issue:** create a GitHub issue with acceptance criteria.
2. **Branch:** `git checkout main`, `git pull`, `git checkout -b feat/<short-name>`
   (prefixes: `feat/`, `fix/`, `chore/`, `refactor/`, `test/`, `docs/`).
3. **Plan:** agree on the plan before writing code.
4. **Implement in small commits** using Conventional Commits, e.g.
   `feat(leads): add PATCH /leads/:id/status endpoint`.
5. **Self-review:** read `git diff main` line by line. If I can't explain a line, it doesn't get committed.
6. **Checks:** `npm run lint`, `npm run typecheck`, `npm test` in the affected package(s).
7. **Tests:** unit tests for pure logic; Supertest integration tests for every endpoint (happy path,
   invalid input, unauthenticated, not found) against the `tinycrm_test` database.
8. **Manual testing:** loading, empty, error (stop the server), long text, slow network throttling.
9. **Performance check:** `EXPLAIN ANALYZE` main queries, no N+1 queries, no needless re-renders.
10. **PR:** push the branch, open a PR with _What / Why / How tested / Screenshots_, link the issue.
11. **CI:** must be green.
12. **Review:** ask Claude for a strict senior-level review of the diff, address the comments.
13. **Merge:** squash-merge, delete the branch, update README/this file if conventions changed.

Never commit directly to `main` (after the initial scaffold). Never commit `.env`.

---

## Roadmap and progress

Update the checkboxes as features are merged.

- [ ] 0. Scaffold: folders, dependencies, Docker services, Prettier/Husky, first commit
- [ ] 1. Health check: env config, pg pool, Redis client, app/server split, error handler,
      `GET /api/health`, GitHub Actions CI (lint, typecheck, test, build)
- [ ] 2. Database migrations for users, leads, activities
- [ ] 3. Auth: register, login, logout, me, `requireAuth`; frontend login page and protected routes
- [ ] 4. Leads CRUD API with validation and integration tests
- [ ] 5. Frontend foundation: apiClient, QueryClient, layout, `components/ui` starter kit
- [ ] 6. Leads list page with filters, search, sort, pagination (URL state), lead form
- [ ] 7. Lead detail page with activity timeline (notes and tasks)
- [ ] 8. Pipeline Kanban board with optimistic status updates
- [ ] 9. My tasks page
- [ ] 10. Dashboard stats with Redis cache-aside and invalidation (with a test for staleness)
- [ ] 11. Polish: states, accessibility pass, one Playwright E2E test, README
- [ ] Stretch: itinerary items per lead with cost, markup %, and computed selling price

## Useful commands (run one at a time)

```bash
docker compose up -d
docker compose ps
docker compose exec postgres psql -U tinycrm -d tinycrm_dev
docker compose exec redis redis-cli
npm run migrate -- up
npm run migrate -- down
npm run dev
npm test
```
