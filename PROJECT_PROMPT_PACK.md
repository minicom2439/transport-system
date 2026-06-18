# Project Prompt Pack — Land Transport Management System

A set of deep, copy-paste prompts for continuing work in new chats.

**How to use this pack**
1. Always paste **PROMPT 0 — BASE CONTEXT** first (it's the shared foundation).
2. Then paste the ONE task prompt that matches your work (A–F).
3. In the same message, **upload `land-transport-system-final.zip`** (the full source
   of both projects + schema + docs).
4. Fill in the `>>> FILL IN <<<` block at the end of the task prompt.

Each prompt is delimited by `==== COPY FROM HERE ====` / `==== COPY TO HERE ====`.

---

# PROMPT 0 — BASE CONTEXT (always paste first)

```
==== COPY FROM HERE ====
You are helping me maintain and extend an existing system of TWO related web
applications that share ONE database and run fully OFFLINE on a local network. I will
upload `land-transport-system-final.zip` containing the full source of both projects,
the database schema, and docs. Read the ACTUAL current code before changing anything —
do not assume from memory; grep/inspect the real files.

THE TWO APPLICATIONS
- HQ Transport (folder `hq-backend/` + `hq-frontend/`, SYSTEM_CODE=hq): headquarters
  oversight — issues directives, reviews reports/statistics across all units, manages
  reference data.
- Unit Operations (folder `unit-backend/` + `unit-frontend/`, SYSTEM_CODE=unit):
  subordinate-unit operations — daily data entry, transport planning,
  employees/attendance, payments, reservations, file exchange, and
  receiving/responding to HQ directives.
They are two independent Node servers that NEVER call each other; they collaborate
ONLY through the shared database.

DEPLOYMENT TARGET (affects every decision)
- Windows, air-gapped/offline LAN. I cannot test deeply; aim for low-bug, verified work.
- DB: XAMPP 8.2.12 (bundles MariaDB 10.4.32). Only MariaDB is used; Apache/PHP are not.
- Runtime: Node.js 22 LTS (engines node >=18 <23; do not exceed 22).

STACK
- Frontend: React 18 + Vite 5, plain JavaScript/JSX, plain CSS with design tokens.
- Backend: Node + Express 4 (^4.22.2) + Knex 3. Sessions stored in the DB.
- DB in production: MariaDB/MySQL via the mysql2 driver. SQLite (better-sqlite3) is a
  DEV-ONLY stand-in. Canonical schema = `schema.mysql.sql` (86 tables), single source
  of truth shared by both systems.

KEY ARCHITECTURE
- Navigation lives in one self-referencing table `system_sections` (roots → groups →
  screens) with a `system_code` discriminator (hq|unit). Each backend reads SYSTEM_CODE
  and serves only its own sections. The menu is also the basis of permissions.
- Auth: username+password (bcryptjs, NOT JWT); server-side sessions in the DB; signed
  cookie. RBAC: roles grant actions (view/add/edit/remove/export) on sections;
  super-admin bypasses. Data scope (by unit/region) is applied in repositories and is
  FAIL-CLOSED (no scope → no rows unless super-admin).
- Backend layering is strict: routes → services → repositories → db/knex.js (the single
  connection). Only repositories build SQL.
- Dev/prod DB split lives ONLY in each backend's `knexfile.js`: development =
  better-sqlite3 (./dev.db or SQLITE_FILE); production = mysql2 reading MYSQL_HOST,
  MYSQL_PORT, MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE. App code never changes between.
- Env (src/config/index.js): NODE_ENV, SYSTEM_CODE, PORT, SERVE_CLIENT, CLIENT_DIST,
  SESSION_SECRET, COOKIE_SECURE, MAX_UPLOAD_MB (25), JSON_BODY_LIMIT.
- Files are stored as bytes in the DB (`file_blobs`: LONGBLOB + sha256 + byte_size),
  used by unit Data Transfer and HQ directive attachments. Limit MAX_UPLOAD_MB=25;
  MariaDB max_allowed_packet must be >=64M.
- Frontend: `api/client.js` uses RELATIVE paths + credentials:'include' (same-origin
  cookie auth). `AuthContext` exposes user/access/can(). `CrudScreen` is the main
  list/form screen. `Sidebar` renders nav from `/api/meta/nav` and maps codes to routes
  via SCREEN_ROUTES. In `App.jsx`, EVERY app page must be INSIDE
  `<Route element={<ProtectedRoute><Layout /></ProtectedRoute>}>`; only /login and
  /register are outside.

STANDING RULES (follow every time)
1. Review the real current code before editing; never trust memory about file contents.
2. Offline audit on every change: no CDNs, web fonts, map tiles, analytics, or external
   HTTP at build or run time. Keep dependencies pure-JS where possible.
3. Shared/schema changes apply to BOTH systems — do both and tell me explicitly.
4. Be honest about scope and deferrals; don't claim done what isn't.
5. Verify with builds + smoke tests before delivering.
6. Never suggest a phone call.
7. Re-package and hand back affected artifacts after changes.

VERIFY / TEST
- Reset+seed a backend (dev/SQLite): `node src/db/init-dev-db.js && node src/db/seed.js`.
- Build a frontend: `npm run build` (must pass).
- Backend smoke test: start app in-process on base `http://127.0.0.1:<port>` (NOT
  localhost), reseed first, login, GET endpoints. When sending a JSON body in a test,
  always include {'Content-Type':'application/json'} or login fails.
- Logins: super-admin admin/admin123 (both systems); sample users use test123.
- npm audit is currently 0 vulnerabilities — keep it there; never run
  `npm audit fix --force` casually (prefer safe in-major bumps).

CURRENT STATE: both projects are feature-complete and passed a deep review with no
outstanding bugs; express/express-session patched (0 vulnerabilities); docs written
(README, DEPLOYMENT_GUIDE, ARCHITECTURE).

Acknowledge you've read this, then wait for the task prompt that follows.
==== COPY TO HERE ====
```

---

# PROMPT A — BACKEND CHANGE / BUG FIX

```
==== COPY FROM HERE ====
TASK TYPE: Backend change or bug fix (Node + Express + Knex). Base context already
provided.

BACKEND MAP (per project, under <project>-backend/src/)
- routes/        HTTP endpoints; attach auth + permission + audit middleware. Mounted
                 in routes/index.js via router.use('/path', ...).
- services/      business rules, validation, data-scope decisions.
- repositories/  the ONLY layer that builds SQL (via Knex).
- db/knex.js     the single DB connection. db/seed.js + db/init-dev-db.js for dev data.
- config/        env parsing. middleware/ for auth, permissions, errors.
- list-query.js  parseListParams / applyFiltersAndSearch / applySort, driven by a
                 per-resource CFG {columns, sortable, filterable, searchable, defaultSort}.

REUSABLE FACTORIES (use these instead of hand-writing CRUD)
- _crud.factory(service, sectionCode, table): standard list/create/update/delete + the
  permission gate for sectionCode.
- _readonly.factory(service, sectionCode): a gated read-only list.
- _workflow.factory(...): draft → submitted → approved/rejected flows.
- _dailyByDept.factory(sectionCode, department): read dashboard over
  business_daily_reports filtered to one department.
- master-data.factory: makeService({scopeColumn, validate, createdBy, updatedBy}) +
  requireFields(...) + makeReadService.

CONVENTIONS YOU MUST RESPECT
- Permissions: any endpoint gated on requirePermission('CODE', ...) (or via a factory's
  sectionCode) REQUIRES that 'CODE' exists as a section in db/seed.js. A gate with no
  matching seeded section = unreachable for normal users. If you add a gated endpoint,
  add/confirm the section + grant it to the right roles in seed.js.
- Errors: throw ValidationError → 400 {fields}; ForbiddenError → 403. The error
  middleware honors err.status and err.publicMessage. For 404 use:
  throw Object.assign(new Error(msg), { status: 404, publicMessage: msg });
  (NotFoundError is NOT exported — do not import it.)
- Data scope is applied in repositories and is fail-closed. Keep new queries
  scope-aware where the resource is unit/region-scoped.
- Keep dependencies pure-JS; do not add native modules. Do not add external HTTP calls.

STEPS
1. Reproduce/understand: grep the relevant route → service → repository chain. Read the
   real code; confirm the current behavior.
2. Make the minimal correct change in the right layer (SQL only in repositories).
3. If it's gated, confirm the section exists in seed.js and is granted appropriately.
4. Reseed + smoke test: `node src/db/init-dev-db.js && node src/db/seed.js`, then start
   in-process on 127.0.0.1, login admin/admin123, GET the affected endpoint(s) and
   confirm no 5xx and correct payloads. If the change touches shared behavior, do the
   same in BOTH backends.
5. Re-package the affected backend (zip excluding node_modules/dev.db/.env/public/
   package-lock) and hand it back with a short note of what changed.

VERIFY BEFORE DELIVERING
- No 5xx on affected endpoints as super-admin; expected 4xx still behave.
- Every new/changed gate maps to a seeded section.
- npm audit still 0 vulnerabilities; no new native deps; offline audit clean.

>>> FILL IN <<<
- System: HQ / unit / both
- Endpoint(s) or feature:
- Current behavior (and any error text / stack):
- Desired behavior:
==== COPY TO HERE ====
```

---

# PROMPT B — FRONTEND CHANGE / BUG FIX

```
==== COPY FROM HERE ====
TASK TYPE: Frontend change or bug fix (React + Vite). Base context already provided.

FRONTEND MAP (per project, under <project>-frontend/src/)
- App.jsx               routing. EVERY app page must be INSIDE
                        <Route element={<ProtectedRoute><Layout /></ProtectedRoute>}>;
                        only /login and /register are outside. (A past bug was pages
                        placed outside this group → no sidebar + no auth.)
- api/client.js         fetch wrapper: RELATIVE paths + credentials:'include'. Use
                        api.get/post/put/del. Never hardcode a host/port.
- context/AuthContext   exposes user, access, and can(action, sectionCode).
- components/CrudScreen  main list/form screen. Props include: title, subtitle,
                        section, columns, fields, api, initialSort, toServerFilters,
                        lookups, entityLabel, readOnly, tree={filterKey,title,nodes},
                        and optional rowExtra=(row)=>JSX for extra row actions.
- components/WorkflowScreen, DeptDailyReportsPage(apiPath,section,title,subtitle),
  Modal, DataTable.
- components/Sidebar.jsx renders nav from /api/meta/nav and maps each section code to a
  route+icon via the SCREEN_ROUTES map. New leaf screens need a SCREEN_ROUTES entry AND
  a matching <Route> in App.jsx, or the menu item goes nowhere.
- hooks/useLookups: useLookups({units:true, regions:true, lookups:['name', ...]}). Each
  lookup name must be in the backend meta LOOKUPS whitelist
  (<backend>/src/services/meta.service.js) or the dropdown is empty.
- styles/tokens.css: system font stack only. NO web fonts, NO CDN assets, NO map tiles.

CONVENTIONS YOU MUST RESPECT
- Same-origin only; the client never calls an absolute URL.
- A field with optionsKey:'X' needs X provided in the screen's lookups object (via
  useLookups, or merged in like lookups={{ ...base, X: data }}).
- Keep components plain JS/JSX; no new external runtime libraries unless I ask.

STEPS
1. Reproduce: read the real page/component and trace data flow (page → CrudScreen →
   api/client → endpoint). Confirm current behavior.
2. Make the minimal change. If adding a screen: add the page, its <Route> INSIDE the
   Layout group, its SCREEN_ROUTES entry (route + icon), and confirm the backend
   endpoint + any lookups exist.
3. Build: `npm run build` must pass. Do an offline check (no fonts.google/cdn/unpkg/
   jsdelivr/tile./map references introduced).
4. Re-package the affected frontend (zip excluding node_modules/dist) and hand it back
   with a note. Remind me to rebuild and drop dist into the backend's public/.

VERIFY BEFORE DELIVERING
- Build passes; no dead sidebar links; new leaf screens mapped in SCREEN_ROUTES; routes
  inside the Layout/ProtectedRoute group.
- Any optionsKey used is actually provided; lookups requested are served by the backend.
- Offline audit clean.

>>> FILL IN <<<
- System: HQ / unit / both
- Screen/component:
- Current behavior (and any console error):
- Desired behavior:
==== COPY TO HERE ====
```

---

# PROMPT C — SCHEMA / DATABASE CHANGE

```
==== COPY FROM HERE ====
TASK TYPE: Database schema change. Base context already provided. The canonical schema
is `schema.mysql.sql` (86 tables) and it is the SINGLE SOURCE OF TRUTH shared by BOTH
systems.

WORKFLOW (do all of it — partial changes cause drift)
1. Edit `schema.mysql.sql` (the canonical MySQL/MariaDB schema).
2. Regenerate the SQLite dev schema: `python3 mysql_to_sqlite.py` → produces
   `schema.sqlite.sql`. Copy BOTH schema files into BOTH backends' `src/db/`
   (each backend keeps a .mysql.sql and a .sqlite.sql copy).
3. Update db/seed.js (and any repository/service) to match new columns/tables.
4. Reseed both backends (dev): `node src/db/init-dev-db.js && node src/db/seed.js`.
5. Smoke test both backends (login + affected endpoints) for 5xx.

MARIADB COMPATIBILITY RULES (target is MariaDB 10.4.32 via XAMPP)
- Use utf8mb4 / utf8mb4_unicode_ci (NOT the MySQL-8-only utf8mb4_0900_ai_ci).
- InnoDB, FOREIGN KEYs, BIGINT UNSIGNED keys. LONGBLOB for binary (file_blobs).
- Multiple DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP is fine
  (supported since MariaDB 10.2). Avoid any MySQL-8-only syntax.
- Conventions on most tables: soft-delete is_active; audit columns created_at,
  updated_at, created_by, updated_by.
- Large blobs require max_allowed_packet>=64M on the server (already documented).

SHARED-DB IMPLICATION
- Both systems read/write this one database. A new table/column may be used by HQ, unit,
  or both. Make sure seeding and any nav/permission additions cover the right
  system_code(s). If a new screen is needed, see PROMPT F.

STEPS / VERIFY BEFORE DELIVERING
- schema.mysql.sql edited; SQLite regenerated; BOTH copies updated in BOTH backends.
- Both seeds run clean; both backends smoke-test with no 5xx.
- Re-deliver schema.mysql.sql + both backends; tell me to re-import the schema on
  MariaDB (phpMyAdmin/CLI) and reseed.

>>> FILL IN <<<
- What data/structure to add or change (tables, columns, relationships):
- Which system(s) use it: HQ / unit / both
- Any rules (uniqueness, defaults, scoping by unit/region):
==== COPY TO HERE ====
```

---

# PROMPT D — DEEP REVIEW / QA PASS

```
==== COPY FROM HERE ====
TASK TYPE: Deep backend-to-frontend review of BOTH projects to find and fix bugs. Base
context already provided. Be thorough and evidence-based: run the checks, show results,
fix what's broken, re-verify, and re-package anything you change.

RUN THIS BATTERY (both projects)
1. Runtime smoke: reseed each backend, start in-process on 127.0.0.1, login
   admin/admin123, GET every mounted endpoint (parse router.use('/...') in
   routes/index.js). Flag any 5xx. Expected 4xx (param-required) are fine.
2. Permission gates vs seeded sections: every requirePermission/factory section code
   must exist in db/seed.js. Report gates with no matching section.
3. Dead sidebar links: every SCREEN_ROUTES `to:` must have a matching <Route> in
   App.jsx.
4. Leaf screens mapped: every leaf nav code in seed.js must have a SCREEN_ROUTES entry.
5. Lookups: every useLookups name must be in the backend meta LOOKUPS whitelist.
6. optionsKey coverage: every field optionsKey must be provided to its CrudScreen
   (via useLookups or a merged lookups object).
7. Routing: exactly one ProtectedRoute/Layout group; only /login + /register outside it.
8. api/client: relative paths + credentials:'include'; no hardcoded host.
9. SPA fallback in app.js serves index.html for non-/api, non-/health routes.
10. Duplicate section definitions; all grant() codes resolve to defined sections.
11. Dependencies pure-JS (only better-sqlite3 native, optional); npm audit = 0.
12. Offline audit: no CDNs/web fonts/map tiles/external HTTP.
13. Both frontends build (`npm run build`).

OUTPUT
- A concise report: each check + PASS/✅ or the exact finding.
- For any real bug: explain root cause, fix it in the correct layer, re-run the
  relevant check, and re-package the affected artifact(s). Distinguish real bugs from
  false positives and say which is which.

>>> FILL IN <<<
- Scope: full review, or focus area (e.g., a specific module/system):
==== COPY TO HERE ====
```

---

# PROMPT E — DEPLOYMENT / ENVIRONMENT HELP (XAMPP + Node, offline)

```
==== COPY FROM HERE ====
TASK TYPE: Help with installing/running/operating the system on XAMPP + Node, offline
on Windows LAN. Base context already provided.

GROUND TRUTH
- One MariaDB (XAMPP 8.2.12 → MariaDB 10.4.32) on port 3306, ONE shared database used by
  both systems. Apache/PHP not required; phpMyAdmin handy for import.
- Two Node backends: HQ on 4000, unit on 4001. Each serves its own built frontend
  (SERVE_CLIENT=true → drop the frontend dist into <backend>/public).
- Node 22 LTS (engines node >=18 <23). XAMPP's MariaDB uses mysql_native_password, so
  mysql2 connects with no auth workaround (the MySQL-8 caching_sha2 issue does not apply).

MUST-DO CONFIG
- my.ini [mysqld]: max_allowed_packet=64M (for 25MB file storage), then restart MariaDB.
- Create DB (e.g. transport_system, utf8mb4_unicode_ci) + a user; import
  schema.mysql.sql once (phpMyAdmin Import or `mysql ... < schema.mysql.sql`).
- Each backend .env: NODE_ENV=production, SYSTEM_CODE=hq|unit, PORT=4000|4001,
  SERVE_CLIENT=true, MYSQL_HOST=127.0.0.1, MYSQL_PORT=3306, MYSQL_USER, MYSQL_PASSWORD,
  MYSQL_DATABASE (same for both), SESSION_SECRET (long random, different per backend),
  COOKIE_SECURE=false (plain-HTTP LAN), MAX_UPLOAD_MB=25.
- Install deps (online npm install, or vendored node_modules offline). Seed BOTH
  (npm run db:seed each) so both nav trees exist. Build BOTH frontends; copy each dist
  into its backend's public/. Start both (npm start). Login admin/admin123, change it.

OFFLINE NOTE: `npm install` needs internet OR a vendored node_modules / local mirror.
better-sqlite3 is optional/dev-only; safe if absent in production.
npm audit guidance: it's currently 0; never use `npm audit fix --force` casually.

For any deployment question, answer from this ground truth and the DEPLOYMENT_GUIDE.md
in the zip. If something is version/build specific and might have changed, say so.

>>> FILL IN <<<
- The deployment question or error you hit (include exact message if any):
==== COPY TO HERE ====
```

---

# PROMPT F — ADD A NEW FEATURE / SCREEN (END-TO-END)

```
==== COPY FROM HERE ====
TASK TYPE: Add a new feature/screen end-to-end across DB → backend → frontend → nav →
permissions, in the right system(s). Base context already provided. Follow the existing
patterns exactly; do not invent new ones.

END-TO-END STEPS
1. DB (if new data): edit schema.mysql.sql, regenerate SQLite (python3
   mysql_to_sqlite.py), copy both schema files into both backends' src/db/. Follow
   MariaDB rules (utf8mb4_unicode_ci, InnoDB, BIGINT UNSIGNED, is_active + audit cols).
2. Backend: add repository (SQL only here) → service (rules/validation/scope) → route.
   Prefer a factory (_crud/_readonly/_workflow/_dailyByDept/master-data) over hand CRUD.
   Mount it in routes/index.js. Gate it with a section code.
3. Seed: add the new section(s) to db/seed.js under the correct system_code, place it in
   the nav tree (root/group/screen), and grant the right actions to the right roles.
   IMPORTANT gotcha: do NOT do a global string-replace of a code that's also a
   screen-definition line (it corrupts the seed). Add EXPLICIT grant() calls instead.
4. Frontend: add the page (usually a CrudScreen with columns/fields/lookups), add its
   <Route> INSIDE the ProtectedRoute/Layout group in App.jsx, add a SCREEN_ROUTES entry
   (route + icon). If it uses dropdowns, add the lookup to the backend meta LOOKUPS
   whitelist and request it via useLookups.
5. Verify: reseed both (if shared), smoke test the new endpoint (no 5xx), build the
   frontend, run the relevant Prompt D checks (gate↔section, dead links, leaf mapped,
   lookups served, optionsKey provided), offline audit, npm audit=0.
6. Re-package affected artifacts (schema if changed, the backend(s), the frontend(s),
   or the full bundle) and hand back with a note. Apply to BOTH systems if shared.

>>> FILL IN <<<
- System(s): HQ / unit / both
- Feature name and what it does:
- Data it stores (fields) and any rules (uniqueness, scope by unit/region, workflow):
- Where it should appear in the menu (which root/group):
- Who can see/edit it (which roles/profiles):
==== COPY TO HERE ====
```
