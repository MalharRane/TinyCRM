# SETUP.md — TinyCRM one-time scaffold

Reference for what the finished scaffold should look like.

**Rules for running these commands**

- Commands in this guide are written for Git Bash. In PowerShell, Claude gives you the equivalents.
- Run **one line at a time** and read the output before running the next.
- Each section says which folder to be in.

**Prerequisites:** Node.js 20+ (LTS), Git, Docker Desktop (open and running).

```bash
node -v
npm -v
git --version
docker --version
```

---

## 1. Root folder and Git

```bash
git init
npm init -y
printf "node_modules/\ndist/\ncoverage/\n.env\n*.log\n.DS_Store\n" > .gitignore
mkdir -p docs
touch docs/LEARNINGS.md
```

---

## 2. Backend (start in the project root)

```bash
mkdir server
cd server
npm init -y
npm install express pg ioredis zod bcrypt jsonwebtoken cookie-parser helmet dotenv
npm install -D typescript tsx @types/node @types/express @types/pg @types/bcrypt @types/jsonwebtoken @types/cookie-parser vitest supertest @types/supertest node-pg-migrate eslint @eslint/js typescript-eslint globals
npm pkg set type=module
npm pkg set scripts.dev="tsx watch src/server.ts"
npm pkg set scripts.build="tsc"
npm pkg set scripts.start="node dist/server.js"
npm pkg set scripts.typecheck="tsc --noEmit"
npm pkg set scripts.lint="eslint ."
npm pkg set scripts.test="vitest run"
npm pkg set scripts.migrate="node-pg-migrate"
printf "NODE_ENV=development\nPORT=4000\nDATABASE_URL=postgres://tinycrm:tinycrm@localhost:5432/tinycrm_dev\nREDIS_URL=redis://localhost:6379\nJWT_SECRET=replace-with-a-long-random-string\n" > .env.example
cp .env.example .env
mkdir -p src/config src/db src/cache src/middleware src/utils migrations
mkdir -p src/modules/auth src/modules/leads src/modules/activities src/modules/dashboard
touch src/app.ts src/server.ts src/config/env.ts src/db/pool.ts src/cache/redis.ts src/utils/AppError.ts
touch src/middleware/auth.ts src/middleware/validate.ts src/middleware/errorHandler.ts
for m in auth leads activities dashboard; do for f in routes controller service repository schemas; do touch src/modules/$m/$m.$f.ts; done; done
touch tsconfig.json eslint.config.js
cd ..
```

**If `bcrypt` fails to install** (native build issue on Windows), run these in `server/` instead:

```bash
npm uninstall bcrypt @types/bcrypt
npm install bcryptjs
```

### `server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "sourceMap": true
  },
  "include": ["src"]
}
```

### `server/eslint.config.js`

```js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import globals from 'globals';
import { defineConfig } from 'eslint/config';

export default defineConfig([
  { ignores: ['dist'] },
  js.configs.recommended,
  tseslint.configs.recommended,
  { languageOptions: { globals: globals.node } },
]);
```

---

## 3. Frontend (start in the project root)

```bash
npm create vite@latest client -- --template react-ts
cd client
npm install
npm install react-router @tanstack/react-query @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities react-hook-form @hookform/resolvers zod tailwindcss @tailwindcss/vite
npm install -D @tanstack/react-query-devtools vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
npm pkg set scripts.test="vitest run"
mkdir -p src/app src/lib src/components/ui
for f in auth leads pipeline tasks dashboard; do mkdir -p src/features/$f/components src/features/$f/pages && touch src/features/$f/api.ts src/features/$f/hooks.ts; done
touch src/lib/apiClient.ts
echo '@import "tailwindcss";' > src/index.css
rm -f src/App.css
cd ..
```

- If `npm create vite` asks whether to install and start now, choose **No**.
- Open `client/src/App.tsx` and delete the line `import './App.css'`.

### `client/vite.config.ts`

If the original file imported `@vitejs/plugin-react-swc`, keep that import instead.

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      '/api': 'http://localhost:4000',
    },
  },
});
```

---

## 4. Docker, Prettier, Husky (project root)

```bash
touch docker-compose.yml
npm install -D prettier husky lint-staged
npx husky init
echo "npx lint-staged" > .husky/pre-commit
echo '{ "singleQuote": true, "semi": true, "printWidth": 100 }' > .prettierrc
echo '{ "*.{ts,tsx,js,json,css,md,yml}": "prettier --write" }' > .lintstagedrc.json
```

### `docker-compose.yml` (YAML: keep the indentation exactly)

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: tinycrm
      POSTGRES_PASSWORD: tinycrm
      POSTGRES_DB: tinycrm_dev
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U tinycrm']
      interval: 5s
      retries: 5

  redis:
    image: redis:7
    ports:
      - '6379:6379'

volumes:
  pgdata:
```

If port 5432 is already in use (local Postgres installed), change it to `"5433:5432"` and use port
5433 in `DATABASE_URL`.

---

## 5. Start services and verify (project root, Docker Desktop running)

```bash
docker compose up -d
docker compose ps
docker compose exec postgres createdb -U tinycrm tinycrm_test
docker compose exec postgres psql -U tinycrm -d tinycrm_dev -c "select version();"
docker compose exec redis redis-cli ping
cd client
npm run dev
```

Expected: both containers running, Postgres prints its version, Redis replies `PONG`, the frontend
opens at http://localhost:5173. The backend won't run yet; `server.ts` is empty on purpose
(that's roadmap item 1).

---

## 6. First commit and GitHub (project root; stop Vite with Ctrl+C first)

```bash
cd ..
git add .
git commit -m "chore: scaffold project structure, tooling and docker services"
git branch -M main
git push -u origin main
```

(`origin` is already set to `https://github.com/MalharRane/TinyCRM.git`.)

On GitHub: **Settings → Branches → add a rule for `main` → require a pull request before merging.**
From now on, all work goes through branches and PRs.

---

## Expected final structure

```
TinyCRM/
├── .husky/pre-commit
├── .gitignore  .prettierrc  .lintstagedrc.json  package.json
├── CLAUDE.md
├── docker-compose.yml
├── docs/  SETUP.md  LEARNINGS.md
├── server/
│   ├── .env  .env.example  package.json  tsconfig.json  eslint.config.js
│   ├── migrations/
│   └── src/
│       ├── app.ts  server.ts
│       ├── config/env.ts  db/pool.ts  cache/redis.ts  utils/AppError.ts
│       ├── middleware/  auth.ts  validate.ts  errorHandler.ts
│       └── modules/{auth,leads,activities,dashboard}/<module>.{routes,controller,service,repository,schemas}.ts
└── client/
    ├── package.json  vite.config.ts
    └── src/
        ├── main.tsx  App.tsx  index.css
        ├── app/  lib/apiClient.ts  components/ui/
        └── features/{auth,leads,pipeline,tasks,dashboard}/{api.ts,hooks.ts,components/,pages/}
```
