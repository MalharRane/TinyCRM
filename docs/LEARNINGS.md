# Learnings

One line per concept or bug: what it was → what I learned.

## 2026-11-04

- **PowerShell vs Git Bash:** `printf: not recognized` means PowerShell (`PS C:\>` prompt). Unix commands (`printf`, `touch`, `mkdir -p`, `for ... do`) only exist in Git Bash. This project uses PowerShell syntax.
- **`.gitignore` before the first `git add .`:** without it, `node_modules/` gets committed. `.env` is ignored; `.env.example` (fake values) is committed.
- **Husky default hook:** `npx husky init` writes `npm test` into `.husky/pre-commit`. With no root tests that blocks every commit, so it is replaced with `npx lint-staged`.
- **lint-staged:** runs a command (here `prettier --write`) only on the files staged for commit.
- **Peer dependency ranges:** `typescript-eslint` supports `typescript <6.1.0`, so TypeScript 7 in the client was unsupported. Both packages are pinned to 6.0.x. Check with `npm view <pkg> peerDependencies`.
- **`module: NodeNext`:** Node's ESM loader needs real file extensions, so relative imports in `.ts` files end in `.js`.
- **`noUncheckedIndexedAccess`:** `arr[0]` is typed `T | undefined`, which forces a check.
- **`npm run build` in the client:** `tsc -b && vite build`. Type errors stop the build before Vite runs.
- **Docker client vs engine:** `docker compose` is only a client. "cannot find the file specified ... dockerDesktopLinuxEngine" means Docker Desktop (the engine) is not running.
- **Healthcheck:** `Up (healthy)` means Postgres passed `pg_isready`, not just that the container started.
- **Two databases:** `tinycrm_dev` for development and `tinycrm_test` for integration tests, so tests never touch dev data.
