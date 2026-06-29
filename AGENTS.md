# Repository guide

## Worktrees and ownership

- There is no root package or task runner. `InvoHydra-Frontend/` and `InvoHydra-Backend/` are independent npm projects with independent lockfiles.
- The root repository records both applications as gitlinks (`160000`), but has no `.gitmodules`; each directory has its own `.git`. Run status, diff, branch, and commit commands inside the application being changed. Update the root gitlink only after intentionally committing the child repository.
- `MEMORY/` and `OCSession1.md` are historical session notes, not current sources of truth. Verify their claims against manifests, Prisma schema, and runtime code.

## Setup and local runtime

- Use `npm ci` in each application. The frontend is Next 16 and requires Node `>=20.9`; the backend production image uses Node 22.
- Frontend: `cd InvoHydra-Frontend && npm run dev` (port 3000). Set `NEXT_PUBLIC_API_URL=http://localhost:5000` in the ignored `.env.local`; Firebase `NEXT_PUBLIC_FIREBASE_*` values are required for login. There is no checked-in env template.
- Backend: `cd InvoHydra-Backend && npm run dev` (nodemon) or `npm start` (plain Node), default port 5000. It is ESM (`"type": "module"`) and loads the ignored `.env`; at minimum, database-backed/authenticated flows need `DATABASE_URL` and `JWT_SECRET`.
- Do not assume another backend port works: several frontend paths fall back to, or directly contain, `http://localhost:5000` (notably `app/Pos/page.tsx` and PDF proxies).

## Runtime wiring

- `InvoHydra-Backend/server.js` is the composition root. It mounts Express routers under `/api`; routers apply auth and call `src/controllers/`, whose active persistence layer is PostgreSQL through Prisma. Files under `src/models/` are legacy and are not imported by the current controllers.
- Prisma generates to the non-default tracked directory `InvoHydra-Backend/generated/prisma/`, which `src/config/db.js` imports directly. After editing `prisma/schema.prisma`, run `npx prisma generate` from the backend and include the regenerated client with the schema change.
- The frontend uses the App Router. Most client pages call the Express API directly through `NEXT_PUBLIC_API_URL`; `app/api/proxy/` is used selectively for reports, PDFs, and snapshot upload/presign.
- Authentication starts with Firebase, then stores the backend JWT as `sessionToken` in local storage and a cookie. Prefer `apiHelper.authenticatedFetch` from `lib/authService.ts` for browser-side authenticated calls because it retries once after an expired-token refresh. Next route handlers read the cookie or forwarded `Authorization` header.
- Invoice template IDs are duplicated between `lib/templates/registry.ts` (catalog metadata) and the `templateMap` in `components/templates/TemplateRenderer.tsx`; update both when adding or renaming a template.
- Route spellings such as `/Pos`, `/pro-forma-invocies`, and `/reports/expense/recrring` are the actual App Router paths. Renaming them requires updating links/bookmarks, not just directories.

## Database and operational safety

- Treat the backend `.env` database as potentially shared or production. Before starting the backend or running any database command, inspect only the target identity/host and confirm it is safe; never print secret values.
- Starting `server.js` schedules `src/jobs/accountCleanupJob.js`, which permanently deletes expired soft-deleted users daily at midnight. Do not leave a backend connected to an unverified database running.
- `npx prisma validate` and `npx prisma generate` are non-mutating checks. Do not run `prisma db push`, `prisma migrate*`, raw SQL scripts, restore scripts, or endpoint smoke tests that write data without explicit confirmation of the target database and rollout plan. Historical notes document a `db push` attempt that proposed dropping live columns.
- Deploy CI does not apply migrations. Schema changes need an explicit migration/deployment plan in addition to updating the schema and generated client.

## Verification

- Frontend focused lint: `cd InvoHydra-Frontend && npx eslint path/to/file.tsx`; full lint: `npm run lint`.
- Frontend typecheck: `npx tsc --noEmit`. The current repository has many pre-existing type errors, while `next.config.ts` sets `typescript.ignoreBuildErrors: true`; record whether changed files add errors instead of treating a successful build as type safety.
- Frontend production build: `npm run build` (explicitly uses webpack). It fetches Poppins through `next/font`, so it requires network access; it also currently warns that the `api` key in `next.config.ts` is not a valid Next 16 option.
- Backend static checks: `cd InvoHydra-Backend && npx prisma validate`, then `node --check server.js` and `find src -type f -name '*.js' -exec node --check {} \;`.
- Neither application defines an automated test script or contains a test suite. Backend CI only builds and deploys Docker on pushes to the backend `testing` branch; it does not lint, test, typecheck, or migrate.
