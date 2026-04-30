# Development Environment Setup Guide

This guide explains, end to end, how to set up a local development
environment for Atlas CMMS and how to use it to implement and verify a
new feature such as a **Persian calendar view** in the work order
calendar (UI rendered in Jalali / Shamsi while the data continues to be
stored as Gregorian/ISO on the API).

It is written so that someone unfamiliar with the codebase can:

1. Run the full stack locally.
2. Make code changes in the right places.
3. See those changes immediately in a browser (and on the API).
4. Test them safely before deploying anything to production.

The repository has four sub-projects:

| Path        | Stack                       | Purpose                                                |
|-------------|-----------------------------|--------------------------------------------------------|
| `api/`      | Java 17, Spring Boot 3.2    | REST backend, Postgres, Liquibase, JWT auth, Quartz    |
| `frontend/` | React 17 + TypeScript (CRA) | Main authenticated web app (work orders, settings, …) |
| `mobile/`   | React Native + Expo         | Mobile app                                             |
| `home/`     | Next.js 16                  | Marketing / public site                                |

For a Persian *calendar view* the main work happens in `frontend/`,
with a small contract change in `api/`. `mobile/` and `home/` are
optional follow-ups.

---

## 1. Prerequisites

Install the following on your workstation. Versions below are the ones
the project has been verified against (matching `docker-compose.yml`,
the Dockerfiles, and `package.json` files).

### Required for any workflow

- **Git** ≥ 2.30
- **Docker Engine** ≥ 24 and **Docker Compose** v2 (`docker compose …`)
- **A POSIX shell** (bash/zsh on macOS/Linux, or WSL2 / Git Bash on
  Windows)
- 8 GB+ RAM available to Docker (Postgres + MinIO + API + frontend)
- Free TCP ports `3000`, `8080`, `5432`, `9000`, `9001` (see
  [`dev-docs/Change Ports.md`](../../dev-docs/Change%20Ports.md) if
  any of them are taken)

### Required only when running services natively (no Docker)

- **JDK 17** (Temurin/Corretto). Spring Boot 3.2 requires Java 17.
- **Maven** is *not* required because `api/` ships `mvnw` (use
  `./mvnw …`).
- **Node.js 21.x** and **npm 10+** for `frontend/` and `home/`.
  The frontend Dockerfile pins `node:21.6.1`.
- **Postgres 16** if you do not want to run the DB in Docker.
- **MinIO** (or a GCS bucket) if you want to test file uploads.

### Recommended tooling

- An IDE with good Java + TypeScript support
  (IntelliJ IDEA / VS Code / Cursor).
- The Cursor / VS Code extensions for ESLint, Prettier, and Lombok.
- `psql` client for ad-hoc database queries (see
  [`dev-docs/Run SQL command.md`](../../dev-docs/Run%20SQL%20command.md)).
- `curl` or HTTPie / Postman / Insomnia for hitting the API directly.

---

## 2. Clone the repository

```bash
git clone https://github.com/grashjs/cmms.git atlas-cmms
cd atlas-cmms
```

Create a feature branch before you change anything. For Persian
calendar view work, a sensible name is:

```bash
git checkout -b feat/persian-calendar-view
```

This keeps your changes isolated and makes it trivial to open a pull
request later.

---

## 3. Choose a development mode

There are three realistic ways to run the stack locally. Each has
trade-offs. Pick one and stick with it for a given task.

### Mode A – Full Docker stack (closest to production)

You run **everything** with `docker compose`: Postgres, MinIO, the
backend, and the frontend. Best for:

- A first-time setup / smoke test.
- Reproducing a production-only bug.
- Testing the final image you would push.

Trade-off: hot-reload is **off** for both `api/` and `frontend/`,
because the images are production builds. You rebuild and restart
containers to see changes.

### Mode B – Hybrid (recommended for feature work)

Postgres + MinIO run in Docker. The backend and the frontend run
natively on your machine with hot-reload. Best for:

- Day-to-day feature development.
- Iterating on UI changes (sub-second feedback in the browser).
- Iterating on backend changes (Spring DevTools / restart on save).

This is the recommended mode for the **Persian calendar view**
feature, because most of the work is in the React app and you need
fast UI feedback.

### Mode C – Fully native

Everything runs on the host (Postgres native install, etc.). Most
flexible but slowest to set up. Only use this if Docker is not
available.

The remainder of this guide focuses on Mode A and Mode B.

---

## 4. Configure environment variables

The repo ships an example `.env` at the root for the Docker stack:

```bash
cp .env.example .env
```

Open `.env` and adjust at least:

| Variable                 | Why                                                                  |
|--------------------------|----------------------------------------------------------------------|
| `POSTGRES_USER`          | DB user. Default `rootUser` is fine for local.                       |
| `POSTGRES_PWD`           | DB password. Default `mypassword` is fine for local.                 |
| `JWT_SECRET_KEY`         | JWT signing secret. Generate one with `openssl rand -base64 32`.     |
| `MINIO_USER` / `_PWD`    | Object storage credentials. Defaults `minio` / `minio123`.            |
| `PUBLIC_FRONT_URL`       | Leave as `http://localhost:3000` for local dev.                       |
| `PUBLIC_API_URL`         | Leave as `http://localhost:8080`.                                     |
| `PUBLIC_MINIO_ENDPOINT`  | Leave as `http://localhost:9000`.                                     |
| `STORAGE_TYPE`           | `minio` (matches the bundled MinIO container).                        |
| `ENABLE_EMAIL_NOTIFICATIONS` | `false` for local. SMTP is not needed to test the UI.            |
| `ENABLE_CORS`            | Set to `true` so the local frontend can call the local backend.       |

The full table of variables lives in the root
[`README.MD`](../../README.MD#set-environment-variables) and is the
authoritative reference; do not duplicate it in feature branches.

> Notes specific to this branch:
> - `docker-compose.yml` references `MINIO_PASSWORD`, while `.env.example`
>   uses `MINIO_PWD`. Add a `MINIO_PASSWORD=<same value as MINIO_PWD>`
>   line to your `.env` until that mismatch is fixed, otherwise the
>   `api` container will not authenticate to MinIO.
> - The frontend image reads runtime config from `.env` at container
>   startup via `runtime-env-cra`. When developing in Mode B you instead
>   set `REACT_APP_API_URL` and friends through your shell.

### Frontend native development (Mode B)

`frontend/` consumes runtime config produced by `runtime-env-cra`.
For `npm start` you can either:

- Export `REACT_APP_API_URL=http://localhost:8080` in your shell, or
- Copy `frontend/.env.example` to `frontend/.env` and set
  `API_URL=http://localhost:8080`.

```bash
cd frontend
cp .env.example .env
# Edit .env so API_URL points at your local API
```

### Backend native development (Mode B)

The backend reads its DB connection from environment variables
(`DB_URL`, `DB_USER`, `DB_PWD`). For local Mode B, point them at
the Dockerized Postgres:

```bash
export DB_URL=localhost:5432/atlas
export DB_USER=rootUser
export DB_PWD=mypassword
export JWT_SECRET_KEY=$(openssl rand -base64 32)
export MINIO_ENDPOINT=http://localhost:9000
export MINIO_BUCKET=atlas-bucket
export MINIO_ACCESS_KEY=minio
export MINIO_SECRET_KEY=minio123
export STORAGE_TYPE=minio
export PUBLIC_API_URL=http://localhost:8080
export PUBLIC_FRONT_URL=http://localhost:3000
export ENABLE_CORS=true
export ENABLE_EMAIL_NOTIFICATIONS=false
export INVITATION_VIA_EMAIL=false
```

You can save these into an `api/.env.local` file and `source` it
each time, or use IntelliJ's "Run Configuration → Environment
variables".

---

## 5. Mode A – Run the full stack with Docker

From the repository root:

```bash
docker compose up -d
```

Compose will:

1. Pull `postgres:16-alpine`, `minio/minio`, the published
   `intelloop/atlas-cmms-backend`, and `intelloop/atlas-cmms-frontend`
   images.
2. Create persistent volumes for Postgres and MinIO data.
3. Run Liquibase migrations on first boot.
4. Expose:
   - Frontend: <http://localhost:3000>
   - Backend:  <http://localhost:8080>
   - Postgres: `localhost:5432`
   - MinIO API:     <http://localhost:9000>
   - MinIO Console: <http://localhost:9001>

Check container health and logs:

```bash
docker compose ps
docker compose logs -f api
docker compose logs -f frontend
```

To stop:

```bash
docker compose down            # keeps the data volumes
docker compose down -v         # also wipes Postgres + MinIO data (factory reset)
```

If Liquibase ever gets stuck with a "Waiting for changelog lock"
error, follow
[`dev-docs/Fix Liquibase lock.md`](../../dev-docs/Fix%20Liquibase%20lock.md).

### Build images from your local sources (instead of pulling)

The published images do not contain your local changes. To test your
own backend or frontend in the full Docker stack, override the image
with a `build:` directive. Create a
`docker-compose.override.yml` next to `docker-compose.yml`:

```yaml
services:
  api:
    image: atlas-cmms-backend-local:dev
    build:
      context: ./api
  frontend:
    image: atlas-cmms-frontend-local:dev
    build:
      context: ./frontend
```

Then:

```bash
docker compose build api frontend
docker compose up -d
```

This is the closest you can get to "what production will run", but it
is also the slowest feedback loop because every change requires a
rebuild.

---

## 6. Mode B – Hybrid setup (recommended for feature work)

### 6.1 Start only the infrastructure containers

```bash
docker compose up -d postgres minio
```

This gives you:

- Postgres on `localhost:5432`
- MinIO on `localhost:9000` (console on `9001`)

Verify:

```bash
docker compose ps
psql "host=localhost port=5432 dbname=atlas user=rootUser password=mypassword" -c "select 1;"
```

### 6.2 Run the backend natively with hot-reload

```bash
cd api
./mvnw spring-boot:run
```

Useful behaviors:

- The default profile is `dev` (set in `application.yml`).
- Liquibase migrates the schema on startup against `DB_URL`.
- The Spring Boot **DevTools** dependency is *not* on the classpath
  by default; restart the process to pick up Java changes (a few
  seconds). For pure resource changes (e.g.
  `messages_fa_IR.properties`, Thymeleaf templates) Spring will
  re-read them on subsequent requests because resources are scanned
  from `target/classes`. If you want true hot-reload, add
  `spring-boot-devtools` to your local `pom.xml` while you work — do
  not commit it.
- API will be reachable at <http://localhost:8080>.

Quick health checks:

```bash
curl -s http://localhost:8080/actuator/health || true
# OpenAPI / Swagger UI is exposed via springdoc:
open http://localhost:8080/swagger-ui/index.html
```

### 6.3 Run the frontend natively with hot-reload

```bash
cd frontend
npm install --legacy-peer-deps
npm start
```

`react-app-rewired` plus CRA gives you hot module reloading:

- Edits to `.tsx`/`.ts` files refresh the browser in <1 s.
- Edits under `frontend/src/i18n/translations/*` propagate the same
  way.
- Errors show up as a red overlay in the browser and in the terminal.

The dev server runs on <http://localhost:3000> and proxies API calls
to whatever you set in `REACT_APP_API_URL` / `API_URL`.

### 6.4 Optional: run the home and mobile projects

Most Persian calendar view work does **not** need these.

- Home (Next.js):

  ```bash
  cd home
  npm install
  npm run dev    # http://localhost:4000
  ```

- Mobile (Expo / React Native):

  ```bash
  cd mobile
  npm install
  npm run android   # or: npm run ios / npm run web
  ```

  See `mobile/README.md` for Firebase + EAS setup. Skip until your
  web feature is stable.

### 6.5 Common Mode B problems

| Symptom                                                                    | Likely cause                                       | Fix                                                                                         |
|----------------------------------------------------------------------------|----------------------------------------------------|---------------------------------------------------------------------------------------------|
| Frontend shows CORS errors                                                  | `ENABLE_CORS=false` on the API                     | Set `ENABLE_CORS=true` and restart the backend.                                              |
| API logs `LiquibaseException: Waiting for changelog lock`                   | Previous run was killed mid-migration              | Run the unlock SQL in `dev-docs/Fix Liquibase lock.md`.                                      |
| API fails to start with `column "calendar_system" of relation … does not exist` | New entity field has no migration yet           | Add a Liquibase changelog under `api/src/main/resources/db/changelog/` (see §10).            |
| Frontend cannot upload attachments                                          | MinIO bucket missing or wrong endpoint              | Open <http://localhost:9001>, create an `atlas-bucket`, set `MINIO_BUCKET=atlas-bucket`.     |
| `npm install` fails with peer dependency errors                             | React 17 + newer transitives                        | Always use `npm install --legacy-peer-deps` in `frontend/`.                                  |
| Browser shows stale runtime env (`API_URL` stuck)                           | `runtime-env-cra` cached `runtime-env.js`          | Stop `npm start`, delete `frontend/public/runtime-env.js`, run `npm start` again.           |

---

## 7. First-run application bootstrap

The first time the API connects to a fresh Postgres, it seeds:

- A super-admin user (default credentials documented in
  [`dev-docs/SuperAdmin password update guide.md`](../../dev-docs/SuperAdmin%20password%20update%20guide.md)).
- Default subscription plans, roles, and permissions.

Steps:

1. Open <http://localhost:3000>.
2. Either log in with the super-admin credentials, or register a new
   account (the `ALLOWED_ORGANIZATION_ADMINS` env variable controls
   self-signup).
3. Navigate to **Settings → General**. Confirm you can change the
   language and the date format. This is the screen where the new
   *calendar system* preference will live for the Persian calendar
   view.

### Reset to a known-good state

If your data gets into a weird state during development:

```bash
docker compose down -v          # removes postgres_data and minio_data
docker compose up -d postgres minio
# then re-run the API; Liquibase will recreate the schema from scratch
```

For more granular resets, see
[`dev-docs/Factory Reset.md`](../../dev-docs/Factory%20Reset.md).

---

## 8. Where to make changes for a Persian calendar view

The companion docs in this folder describe the full design. Below is
the minimal map you need to actually open files and start editing.

### 8.1 Backend (`api/`)

You only need backend work if the **calendar system** has to be a
persisted, per-company preference (recommended). For a pure
"frontend-only Jalali display" prototype you can skip the backend
changes initially.

Edit:

- `api/src/main/java/com/grash/model/enums/Language.java`
  Add `FA` at the end of the enum.
- `api/src/main/java/com/grash/utils/Helper.java`
  Map `FA` → `new Locale("fa", "IR")`.
- `api/src/main/resources/messages_fa_IR.properties` (new file)
  Translate every key from `messages.properties`.
- `api/src/main/java/com/grash/model/enums/CalendarSystem.java` (new
  file) with `GREGORIAN`, `JALALI`.
- `api/src/main/java/com/grash/model/GeneralPreferences.java`
  Add a `private CalendarSystem calendarSystem = CalendarSystem.GREGORIAN;`
  field.
- `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`
  Add the same field so the frontend can PATCH it.
- `api/src/main/resources/db/changelog/…` Add a Liquibase changelog
  (see §10) that adds a `calendar_system` column with a `GREGORIAN`
  default and backfills existing rows.

The detailed reasoning lives in
[`backend-guide.md`](./backend-guide.md).

### 8.2 Main frontend (`frontend/`)

The calendar view itself is here:

- `frontend/src/content/own/WorkOrders/Calendar/index.tsx`
  Hosts the FullCalendar component.
- `frontend/src/content/own/WorkOrders/Calendar/Actions.tsx`
  Headings, range pickers, and labels.
- `frontend/src/contexts/CompanySettingsContext.tsx`
  Central `getFormattedDate` helper. **Most date strings in the app
  go through this.**
- `frontend/src/i18n/i18n.ts`
  Registers languages and FullCalendar / date-fns locales. Add a
  `fa` entry with `IR` flag.
- `frontend/src/i18n/translations/fa.ts` (new file).
- `frontend/src/models/owns/generalPreferences.ts`
  Add `calendarSystem: 'GREGORIAN' | 'JALALI'`.
- `frontend/src/utils/calendarSystem.ts` (new file, recommended)
  Centralize Jalali ↔ Gregorian conversion, formatting, and
  parsing. Pick **one** dependency and confine it to this file:
  - `date-fns-jalali` (closest to current `date-fns` usage), or
  - `dayjs-jalaali` (smaller, fits existing `dayjs` usage), or
  - `moment-jalaali` (only if you accept Moment in this project).
- `frontend/src/hooks/useCompanyDateFormatter.ts` (new file,
  recommended) Reads `generalPreferences.calendarSystem` and
  delegates to the helper above.

For a *display-only* Persian calendar view (no Jalali grid math):

1. Add `fa` to `i18n.ts` and create `translations/fa.ts`.
2. Add the FullCalendar Persian locale loader.
3. In the calendar component, pass `locale="fa"` (or the dynamic
   value coming from `getCalendarLocale`) when the user has selected
   `calendarSystem === 'JALALI'`. Be aware: the locale only
   translates labels — the grid is still Gregorian.

For a *true* Jalali grid (months named فروردین/اردیبهشت/…, weeks
starting Saturday, correct month lengths):

1. Render dates through `formatCompanyDate(value, { calendarSystem:
   'JALALI', timeZone, locale })` from your new helper.
2. Replace or wrap the FullCalendar grid header with Jalali
   month/year labels, or evaluate alternative calendar libraries
   that support Jalali natively. Treat this as a larger task and
   keep API payloads ISO/Gregorian throughout.

The detailed reasoning lives in
[`frontend-guide.md`](./frontend-guide.md).

### 8.3 Optional surfaces

- `mobile/` — see the mobile section in
  [`implementation-guide.md`](./implementation-guide.md). RTL and
  date pickers need explicit work that is out of scope for a
  web-only first release.
- `home/` — only needed if the marketing site should advertise
  Persian support.

---

## 9. The inner development loop

This is the loop you should aim for when building the calendar view.

### 9.1 Backend change → see it on the API

```bash
# 1. Edit Java/SQL/properties
# 2. Stop Spring Boot (Ctrl-C in the api/ terminal)
# 3. Restart:
cd api && ./mvnw spring-boot:run
# 4. Hit the endpoint:
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/general-preferences/<companyId> | jq .
```

If you only changed `messages_fa_IR.properties`, you usually do not
need to restart — Spring `MessageSource` re-reads on each request as
long as the file is on the classpath of the running process.

### 9.2 Frontend change → see it in the browser

With `npm start` running you simply save the file. The dev server:

1. Recompiles the affected module.
2. Pushes the update over WebSocket to the open tab.
3. Re-renders the React tree without losing state.

For language work specifically:

- Use `?lang=fa` in the URL to bypass language detection and load
  Persian directly.
- Open DevTools → Application → Local Storage and clear the `lang`
  key when you want to retest detection from scratch.
- The Settings → General page will show Persian once it is
  registered in `supportedLanguages`.

### 9.3 Database change → keep it reproducible

Never run ad-hoc `ALTER TABLE` against the local DB and call it
done; that only works on your machine. Always express schema
changes as Liquibase changelogs:

```text
api/src/main/resources/db/changelog/<area>/<NN>-add-calendar-system.xml
```

…and reference them from `db/master.xml` (or the area-specific
master file) following the existing pattern.

For day-to-day inspection:

```bash
docker exec -it atlas_db psql -U rootUser -d atlas
\d general_preferences
select id, language, date_format, calendar_system from general_preferences limit 5;
```

### 9.4 Useful debugging entry points

- Spring Boot logs in your terminal — increase verbosity by setting
  `LOGGING_LEVEL_COM_GRASH=DEBUG` in the env.
- `http://localhost:8080/swagger-ui/index.html` — quick way to call
  any endpoint.
- React DevTools and Redux DevTools browser extensions — inspect
  `companySettings`, language state, and date helpers.
- `localStorage` keys: `accessToken`, `lang`. Useful to script
  authenticated requests (`document.cookie`/`localStorage` →
  `Authorization: Bearer …`).

---

## 10. Database migrations are not optional

Atlas CMMS uses Liquibase with `spring.jpa.hibernate.ddl-auto:
validate`. Translation: **Hibernate will refuse to start** if the
schema doesn't match the JPA entities. Whenever you add a column to
an entity (for example `calendar_system` on `GeneralPreferences`):

1. Add the field on the entity *and* on any DTO that exposes/patches
   it.
2. Add a Liquibase changelog file under
   `api/src/main/resources/db/changelog/`.
3. Reference it from the master changelog so Liquibase picks it up.
4. Restart the API. On startup it will:
   - Run the new changeset against your local DB.
   - Validate the schema against the entity model.

If you forget step 2, the API will fail to start with a Hibernate
schema validation error. That is a feature, not a bug — it prevents
schema drift between dev and production.

There is a helper script in `api/scripts/generate-liquibase.js` for
scaffolding new changelog files; use it as a starting point and
adjust it to match the existing style of nearby changelogs.

---

## 11. Testing strategy before deploying

Production-readiness for a Persian calendar view should include all of
the following, layered from cheap to expensive.

### 11.1 Backend

- **Unit tests** (none ship today; add focused ones around new code):
  - `Language.fromString("FA")` → `Language.FA`.
  - `Helper.getLocale(...)` for `Language.FA` → `Locale("fa","IR")`.
  - `CompanyDateFormatter.format(...)` returns the expected Jalali
    string for a known Gregorian instant.
- **Schema sanity**: start the API against a fresh DB
  (`docker compose down -v && docker compose up -d postgres && ./mvnw spring-boot:run`).
  If Liquibase + Hibernate validation passes, your migration is
  correct.
- **Smoke**: PATCH `generalPreferences` with
  `{"calendarSystem":"JALALI"}` and re-fetch. The change must
  round-trip.
- **Translation parity**: every key in
  `api/src/main/resources/messages.properties` must exist in
  `messages_fa_IR.properties` with placeholders intact.

### 11.2 Frontend

- **Type checks**: `npm run build` (or `tsc --noEmit`) catches model
  mismatches between API and TypeScript.
- **Lint**: `npm run lint` and `npm run format` keep diffs small.
- **Unit tests** (the project does not currently use Jest in
  `frontend/`; if you add a calendar helper, add tests with whatever
  framework you set up locally):
  - `formatCompanyDate(<known instant>, { calendarSystem: 'JALALI', timeZone: 'Asia/Tehran' })`
    returns the expected `۱۴۰۵/۰۲/۰۸` style string.
  - `parseJalaliInput("۱۴۰۵/۰۲/۰۸")` returns the correct UTC `Date`.
  - Round-trip: `parse(format(date)) === date` for representative
    timezones (UTC, Asia/Tehran, America/Los_Angeles).
- **Manual UI walk-through**, in this order:
  1. Switch language to Persian. Confirm `<html dir="rtl" lang="fa">`.
  2. Switch `calendarSystem` to `JALALI` in Settings → General.
  3. Navigate to Work Orders → Calendar:
     - Headings/labels are Persian.
     - Days/months display Jalali if a true Jalali grid is
       implemented; otherwise verify Persian-labeled Gregorian
       behavior is intentional.
     - "Today", "Next month", "Previous month" navigation works.
     - Clicking a cell pre-fills the create-work-order form with the
       expected ISO/Gregorian datetime.
  4. Repeat with Gregorian + English to confirm no regression.

### 11.3 Cross-cutting

- **Timezone correctness**: pick three timezones and one date close
  to local midnight. Verify the same instant displays correctly in
  each. Off-by-one-day bugs are the #1 risk in calendar work.
- **DST boundaries**: pick a date right around the spring/fall DST
  transitions (Asia/Tehran historically, plus your local zone). The
  display must not jump.
- **Persistence**: every value the user picks must round-trip
  through the API as ISO/Gregorian. Inspect the network tab and
  confirm the JSON payload is not Jalali.
- **Existing tenants**: log in as a non-Persian company and confirm
  nothing changed. The default for `calendarSystem` must stay
  `GREGORIAN`.

### 11.4 Pre-PR checklist

- [ ] `./mvnw -pl api -am verify` passes (or at least
      `./mvnw spring-boot:run` starts cleanly against a fresh DB).
- [ ] `npm run build` in `frontend/` passes.
- [ ] `npm run lint` in `frontend/` passes.
- [ ] No commented-out code, no `console.log` left in `frontend/`,
      no `System.out.println` left in `api/`.
- [ ] Translation parity script run for `messages_fa_IR.properties`
      and `frontend/src/i18n/translations/fa.ts`.
- [ ] Manual smoke through Mode A (full Docker) using locally built
      images: see §5 "Build images from your local sources".

---

## 12. Best practices

These are the conventions that keep this codebase healthy. They are
especially relevant for calendar work because date/calendar bugs are
hard to spot and easy to ship.

### Branching and commits

- Branch from the latest `main` (or the assigned base branch).
- One logical change per commit. Use imperative subject lines:
  `Add CalendarSystem enum and migration`, not
  `updates`.
- The repo uses Husky + commitlint + lint-staged in `frontend/`.
  Format and lint will run automatically on commit. Don't bypass
  with `--no-verify` unless you know what you're doing.
- Open the PR as **draft** until the manual smoke pass in §11 is
  green.

### Code organization

- **Don't scatter calendar logic.** Put every Jalali ↔ Gregorian
  conversion behind a single helper module on each side
  (`api/.../utils/CompanyDateFormatter.java`,
  `frontend/src/utils/calendarSystem.ts`). Components and
  controllers should never call Jalali libraries directly.
- **Keep API transport ISO/Gregorian.** The calendar system is a
  *display* concern. JSON payloads, query parameters, database
  columns, and reports machine-consumed by integrations stay
  Gregorian.
- **Treat language and calendar as independent preferences.** Some
  Persian users want Gregorian; some non-Persian users want Jalali.
  Don't infer one from the other.

### Translations

- Always copy `en.ts` / `messages.properties` first, then translate
  values. Never invent new keys in a non-English file.
- Preserve placeholders (`{{name}}`, `{0}`, `{1}`) and any HTML
  fragments exactly.
- Save files as UTF-8 with a BOM-less encoding. `.properties` files
  in particular: do not paste rich-text quotes; they break message
  formatting.

### Database

- Every entity change ships with a Liquibase changelog in the same
  PR.
- Defaults must keep existing tenants working: `GREGORIAN` for
  calendar system, English fallback for languages.
- Do not add new enum values in the *middle* of an enum. Append at
  the end. The codebase comment in `Language.java` calls this out
  explicitly.

### Performance and footprint

- The frontend bundle is already large. Prefer **lazy-loading** new
  locales (the existing `dateLocaleLoaders` and
  `calendarLocaleLoaders` patterns) instead of static imports.
- Pick **one** Jalali library and stick with it. Adding both
  `dayjs-jalaali` and `date-fns-jalali` doubles the bundle for no
  benefit.

### Security

- Never commit a real `.env` or any value of `JWT_SECRET_KEY`,
  `MINIO_PWD`, etc. The repo's `.gitignore` already excludes
  `.env`; keep it that way.
- Don't bypass the super-admin password rotation in production
  builds. Defaults are for local only.

### Documentation

- Update `cursor-docs/persian-language-calendar-support/` as the
  design evolves. Keep this development environment guide accurate;
  it is the entry point for the next contributor.

---

## 13. Going to production (high level)

You said you want to verify changes locally before deploying. The
deployment itself is out of scope for this guide, but the *path* is:

1. Local Mode B → feature works against real Postgres + MinIO with
   hot-reload.
2. Local Mode A with `docker-compose.override.yml` building from
   sources → confirms the production images would behave the same.
3. Push the branch, open a PR, wait for review and CI.
4. After merge, the published images
   (`intelloop/atlas-cmms-backend`, `intelloop/atlas-cmms-frontend`)
   are rebuilt and tagged. See
   [`Docker image build guide.md`](../Docker%20image%20build%20guide.md).
5. Roll out to staging first if you have one. For the calendar work
   specifically, run the manual checklist in §11 against staging
   before promoting to production.
6. Production rollout is the same `docker compose up -d` against the
   new tag. Existing tenants default to `GREGORIAN`, so the rollout
   is non-breaking.

---

## 14. Where to ask for help

- Project Discord: <https://discord.gg/cHqyVRYpkA>.
- Issue tracker on GitHub.
- Companion docs in this folder:
  - [`README.md`](./README.md)
  - [`implementation-guide.md`](./implementation-guide.md)
  - [`backend-guide.md`](./backend-guide.md)
  - [`frontend-guide.md`](./frontend-guide.md)
  - [`backend-architecture-assessment.md`](./backend-architecture-assessment.md)
  - [`frontend-architecture-assessment.md`](./frontend-architecture-assessment.md)
- Operations / super-admin docs in [`dev-docs/`](../../dev-docs):
  - [`Change Ports.md`](../../dev-docs/Change%20Ports.md)
  - [`Run SQL command.md`](../../dev-docs/Run%20SQL%20command.md)
  - [`Fix Liquibase lock.md`](../../dev-docs/Fix%20Liquibase%20lock.md)
  - [`Factory Reset.md`](../../dev-docs/Factory%20Reset.md)
  - [`SuperAdmin password update guide.md`](../../dev-docs/SuperAdmin%20password%20update%20guide.md)
  - [`Add translation.md`](../../dev-docs/Add%20translation.md)

If a step in this guide drifts from reality, fix the guide in the
same PR as the code change. The goal is for the next person to read
this once and be running in under 30 minutes.
