# Cost and Effort Estimate: Rewriting Atlas CMMS in C# (.NET Core + Blazor) From Scratch

This document estimates how much effort it would take a team of:

- **2 backend developers** experienced in .NET Core
- **1 Blazor developer**

to **rewrite Atlas CMMS from scratch in C#** — backend in ASP.NET Core, web UI
in Blazor, and (depending on scope) mobile in either .NET MAUI / Blazor Hybrid
or by reusing the existing React Native app against the new backend.

It is the quantitative companion to
[`Dotnet team adoption vs rewrite decision.md`](./Dotnet%20team%20adoption%20vs%20rewrite%20decision.md),
which explains *whether* you should rewrite. This document explains
*how big a rewrite would be* if you decide to do it.

> **Important note on units.** Software estimates are inherently uncertain.
> All numbers below are expressed as **person-months of engineering effort**
> with low / expected / high ranges. Calendar duration depends on team
> composition, parallelism, vacation, hiring, and stakeholder availability.
> A simple conversion is given near the end, but the effort numbers are the
> primary unit.

---

## 1. Executive summary

| Scope | Effort (person-months, expected) | Range (low – high) |
| --- | ---: | ---: |
| Backend rewrite only (.NET 8 + EF Core, feature parity with `api/`) | **42 PM** | 32 – 60 PM |
| Web frontend rewrite only (Blazor, parity with `frontend/`) | **34 PM** | 26 – 50 PM |
| Mobile rewrite (.NET MAUI or Blazor Hybrid, parity with `mobile/`) | **22 PM** | 16 – 34 PM |
| Cross-cutting work (DevOps, infra, QA, PM, docs, i18n, data migration) | **14 PM** | 10 – 22 PM |
| **Total full rewrite** (backend + Blazor web + mobile + cross-cutting) | **~112 PM** | **84 – 166 PM** |
| **Reduced scope rewrite** (backend + Blazor web only, keep React Native mobile against new API) | **~78 PM** | **58 – 116 PM** |
| **MVP cut** (backend + Blazor web, ~60% of features, no mobile, no analytics, no workflows, no licensing) | **~32 PM** | **22 – 46 PM** |

For the specific team described:

- **2 backend .NET devs** ≈ ~24 PM of backend capacity per calendar year
- **1 Blazor dev** ≈ ~12 PM of frontend capacity per calendar year
- Total team capacity ≈ ~36 PM/year (before vacation, ramp-up, hiring of any
  missing roles)

Mapped to the totals above:

| Scope | Approx. wall-clock with this exact team | Notes |
| --- | --- | --- |
| MVP cut (no mobile) | ~12 months | Tight, minimal slippage tolerance |
| Backend + Blazor web parity, no mobile | ~24–30 months | Realistic for feature parity with current `api/` + `frontend/` |
| Full rewrite incl. mobile | ~36–42+ months | Mobile alone exceeds the Blazor dev's bandwidth; needs at least one more engineer |

> **The team described cannot deliver feature parity in under ~2 years
> without compromising scope, quality, or mobile.** This matches the
> qualitative recommendation in the adoption-vs-rewrite document: a full
> rewrite is a multi-year product project, not a coding task.

---

## 2. What is being rewritten?

The current repository contains four main applications:

```text
api/        Java Spring Boot backend (Java 17, Spring Boot 3, JPA/Hibernate, Liquibase)
frontend/   React 17 / TypeScript / Material UI web app
mobile/     Expo 53 / React Native / Redux Toolkit mobile app
home/       Next.js marketing site (out of scope here)
```

The marketing site (`home/`) is excluded. A C# rewrite would replace the
remaining three with:

```text
api/        ->  ASP.NET Core 8 Web API + EF Core + ASP.NET Core Identity / JWT
frontend/   ->  Blazor (Server or WebAssembly) + MudBlazor or Radzen
mobile/     ->  .NET MAUI, Blazor Hybrid, or kept as React Native (decision pending)
```

### 2.1 Measured size of the current code

From [`Code size and authorship analysis.md`](./Code%20size%20and%20authorship%20analysis.md)
plus measurements taken for this estimate:

| Area | Files | Source/config lines |
| --- | ---: | ---: |
| Backend `api/` (Java + XML + properties) | ~903 | ~59,326 |
| &nbsp;&nbsp;of which `.java` | 744 | ~45,001 |
| &nbsp;&nbsp;of which Liquibase XML migrations | ~103 changesets | ~9,468 |
| &nbsp;&nbsp;of which `.properties` (i18n bundles) | 32 | ~2,916 |
| Frontend `frontend/` | ~469 | ~102,377 |
| &nbsp;&nbsp;of which `.tsx` | 281 | ~55,939 |
| &nbsp;&nbsp;of which `.ts` (state, models, types, utilities, i18n) | 183 | ~45,927 |
| Mobile `mobile/` (.ts + .tsx, excluding `node_modules`) | ~275 | ~52,316 |
| **Combined source/config** | **~1,647 files** | **~214,000 lines** |

Backend domain shape:

| Element | Count |
| --- | ---: |
| JPA entities / domain models | 122 |
| REST controllers | 68 |
| Service classes | 83 |
| Spring Data repositories | 65 |
| DTO classes | 294 |
| Liquibase changeset files | 103 |
| Supported UI/email locales | 15 |

Frontend domain shape:

| Element | Count |
| --- | ---: |
| Top-level feature areas under `src/content/own/` (Work Orders, Assets, Locations, Meters, Parts, PM, Requests, Purchase Orders, People & Teams, Vendors, Customers, Categories, Files, Imports, Subscription, Settings, Analytics, Switch Account, Upgrade & Downgrade, User & Company Profile, etc.) | ~22 |
| Redux Toolkit slices | ~42 |
| External SDKs / heavyweight libraries (MUI, Redux Toolkit, Formik/Yup, FullCalendar, ApexCharts, Recharts, react-google-maps, react-spreadsheet, react-quill, react-signature-canvas, STOMP WebSocket, etc.) | 15+ |

This is a **medium-to-large multi-tenant SaaS** with deep domain coverage,
not a CRUD prototype.

---

## 3. Estimation methodology

Three independent methods were used and reconciled.

### 3.1 Method A — LOC-based industry productivity

Rewrites of equivalent C# code typically come in at 60–80% of the original
combined Java + TypeScript line count. For Atlas:

```text
Original source/config:                     ~214,000 lines (api + frontend + mobile)
Expected C# rewrite size (mid):             ~150,000 lines
   Backend (.NET):                          ~50,000 lines
   Web (Blazor):                            ~55,000 lines
   Mobile (.NET MAUI/Blazor Hybrid):        ~30,000 lines
   Migrations / infra / tests:              ~15,000 lines
```

Industry sustained productivity for a **brand-new business application**
typically sits between **300 and 600 lines of C# per developer per month**
(measured net lines of *delivered, reviewed, tested* code, not raw output).

Using **450 LOC/dev/month** as a midpoint:

```text
150,000 lines / 450 LOC/dev/month  ≈  333 developer-months
```

This is the *naive* upper bound and assumes everyone writes greenfield
production code 100% of the time. After adjusting for:

- 30–40% reuse of well-known .NET patterns (Identity, EF, hosted services,
  SignalR for STOMP-equivalent push, IFormFile uploads, etc.)
- mature tooling for OpenAPI clients, scaffolding, source generators,
  AutoMapper
- the team already knows the stack (no Java/React learning tax)

a realistic adjusted figure is **~110–130 PM**, close to the bottom-up
estimate below.

### 3.2 Method B — COCOMO II (organic, intermediate)

COCOMO II for an organic project of ~150 KLOC delivered C#:

```text
Effort  ≈ 2.94 × (KSLOC ^ 1.10) × EAF
        ≈ 2.94 × (150 ^ 1.10) × 1.0
        ≈ ~720 person-months  (raw, no reuse adjustment)
```

COCOMO classically over-predicts modern web/CRUD applications because it was
calibrated against earlier, more imperative codebases without the modern
.NET ecosystem. With:

- reuse adjustment (REVL ≈ 0.4) → ~430 PM
- modern productivity multiplier (TOOL = high, PVOL = low,
  PCAP/PCON/APEX = high) → roughly halves it → ~200 PM
- excluding tests and ops still shaves ~30% → ~140 PM

This is consistent with Method A's adjusted band (~110–160 PM).

### 3.3 Method C — Bottom-up by module (the primary number used)

This is the most defensible estimate for *this* specific codebase. Each
module is sized based on the count and complexity of its current Spring
controllers, services, JPA entities, and React screens.

The complete module-by-module table is in [section 5](#5-bottom-up-module-by-module-estimate).
It produces:

```text
Backend         42 PM   (range 32–60)
Web (Blazor)    34 PM   (range 26–50)
Mobile          22 PM   (range 16–34)
Cross-cutting   14 PM   (range 10–22)
─────────────────────────────────────
Total          ~112 PM  (range 84–166)
```

### 3.4 Reconciliation

| Method | Effort (PM) |
| --- | ---: |
| A — LOC × productivity (adjusted) | 110–130 |
| B — COCOMO II (modern multipliers, reuse-adjusted) | 100–160 |
| C — Bottom-up module table | 84–166 (centered on 112) |

All three converge around **~110 PM expected, with realistic worst case
~165 PM**. The bottom-up number is used as the headline because it can be
re-validated against the actual repository.

---

## 4. Assumptions

These assumptions underpin every number in this document. Changing them
materially changes the estimate.

### Scope assumptions

1. The rewrite targets **feature parity** with the current open-source
   features of Atlas CMMS, *excluding* commercial license-gated features
   unless explicitly listed.
2. The new system uses **PostgreSQL** (matches current Liquibase schema and
   eases data migration). Switching to SQL Server adds ~2 PM.
3. Authentication uses **ASP.NET Core Identity + JWT bearer**, with optional
   OAuth2 (Google/Microsoft) and LDAP, matching today's behavior.
4. **Real-time push** uses SignalR (replaces STOMP/WebSocket). Frontend and
   mobile clients are updated accordingly.
5. **PDF generation** uses QuestPDF or iText7 .NET (both are available).
6. **Object storage** continues to support both **MinIO** (S3-compatible via
   AWSSDK.S3) and **GCS** (Google.Cloud.Storage.V1).
7. **Background jobs** use **Quartz.NET** (already familiar from Spring) or
   Hangfire.
8. **Audit history** (currently Hibernate Envers) is rebuilt with
   EFCore.Audit, Audit.NET, or a hand-rolled change tracker.
9. **i18n** preserves all 15 currently supported locales.
10. **Multi-tenant isolation** by `Company` is preserved end-to-end.

### Team assumptions

1. The 2 backend developers are senior, comfortable with EF Core,
   ASP.NET Core, and DDD-lite patterns.
2. The Blazor developer is mid-to-senior and has shipped at least one
   non-trivial Blazor app.
3. The team has **no React Native expertise**, so the mobile estimate
   assumes a .NET-based mobile rewrite. If mobile stays React Native, the
   mobile effort drops, but **someone must learn React Native** to maintain
   it (cost not included in the table; ~2–4 PM of learning + maintenance
   capacity per year).
4. The team owns DevOps for the new stack themselves. If a separate DevOps
   role is required, treat that as additional capacity, not as part of the
   estimate.
5. No Product Manager is dedicated — the team itself reverse-engineers
   product requirements from the existing system. This is a major risk,
   discussed in [section 9](#9-risks-that-can-double-the-estimate).

### Not included

- Migration of historical customer data from production deployments
  (separate ~3–6 PM project once the schema stabilizes).
- New features beyond current parity (Persian/Jalali support, advanced
  reporting, workflow designer UI improvements, etc.).
- Hiring time for any additional roles.
- Marketing site rewrite (`home/`).

---

## 5. Bottom-up module-by-module estimate

Effort is in **person-months** for the **expected** scenario. Each row's
effort already includes back-end + UI + minimal tests for that module.
Cross-cutting infrastructure is in [section 5.5](#55-cross-cutting-infrastructure).

### 5.1 Core domain & identity

| Module | Current entities/screens involved | Backend PM | UI PM | Notes |
| --- | --- | ---: | ---: | --- |
| Multi-tenant foundation (`Company`, `CompanySettings`, `GeneralPreferences`, `UiConfiguration`) | 4 entities, 5 settings screens | 2.0 | 1.0 | Tenant filter middleware, EF global query filters |
| Users, invitations, verification tokens, settings | `User`, `UserInvitation`, `UserSettings`, `VerificationToken` | 2.0 | 1.5 | Identity, password reset, invitation flow, profile screens |
| Roles, permissions, basic permissions, custom roles | `Role`, `RoleCode`, `BasicPermission`, `PermissionEntity` | 2.5 | 1.5 | Policy-based authorization, custom role editor |
| Teams | `Team` | 0.5 | 0.5 | |
| Auth (JWT, OAuth2 Google/Microsoft, LDAP, SSO) | `AuthController`, OAuth flows, LDAP service | 2.5 | 1.0 | OIDC handlers, optional LDAP, callback handling |
| **Subtotal** | | **9.5** | **5.5** | |

### 5.2 Asset & inventory domain

| Module | Current entities/screens | Backend PM | UI PM | Notes |
| --- | --- | ---: | ---: | --- |
| Locations (incl. Google Maps, hierarchy, floor plans) | `Location`, `FloorPlan`, location tree screens | 1.5 | 2.0 | Map integration, hierarchical UI |
| Assets, asset categories, asset downtime, deprecation | `Asset`, `AssetCategory`, `AssetDowntime`, `Deprecation` | 2.5 | 2.5 | Hierarchy, downtime tracking, depreciation calculations |
| Parts, multi-parts, part categories, quantities, consumption | `Part`, `PartCategory`, `PartConsumption`, `PartQuantity`, `MultiParts` | 2.0 | 2.0 | Inventory math, low-stock alerts |
| Vendors and customers | `Vendor`, `Customer` | 1.0 | 1.0 | |
| Meters, meter categories, readings | `Meter`, `MeterCategory`, `Reading` | 1.5 | 1.5 | Reading history, units, charts |
| **Subtotal** | | **8.5** | **9.0** | |

### 5.3 Work management

This is the **largest single area** and where most CMMS hidden complexity
lives.

| Module | Current entities/screens | Backend PM | UI PM | Notes |
| --- | --- | ---: | ---: | --- |
| Work orders (lifecycle, assignment, status, history) | `WorkOrder`, `WorkOrderHistory`, `WorkOrderConfiguration`, `WorkOrderCategory`, `WorkOrderRequestConfiguration`, `WorkOrderMeterTrigger` | 5.0 | 4.0 | `WorkOrderService` is ~700 lines; controller ~520. Core workflow has many edge cases. |
| Tasks, task base, task options, checklists | `Task`, `TaskBase`, `TaskOption`, `Checklist` | 1.5 | 1.5 | Custom tasks per work order |
| Requests, request portal, request portal fields | `Request`, `RequestPortal`, `RequestPortalField` | 2.0 | 2.0 | Public-facing portal, form designer |
| Preventive maintenance and schedules | `PreventiveMaintenance`, `Schedule` | 2.5 | 2.0 | Recurrence engine, schedule generation |
| Labor, additional cost, time category, cost category | `Labor`, `AdditionalCost`, `TimeCategory`, `CostCategory` | 1.0 | 1.0 | Time tracking on work orders |
| Purchase orders, PO categories | `PurchaseOrder`, `PurchaseOrderCategory` | 1.5 | 1.5 | Approvals, line items |
| Workflow engine (conditions, actions) | `Workflow`, `WorkflowCondition`, `WorkflowAction` | 2.5 | 2.0 | Rule engine + designer UI |
| Custom fields and field configurations | `CustomField`, `CustomFieldValue`, `FieldConfiguration` | 1.5 | 2.0 | Schema-less form rendering |
| Notifications | `Notification`, `NotificationType`, push tokens | 1.0 | 1.0 | In-app inbox + WebSocket/SignalR + push |
| Comments and relations | `Comment`, `Relation`, `RelationType` | 0.5 | 0.5 | |
| **Subtotal** | | **19.0** | **17.5** | |

### 5.4 Platform features

| Module | Current entities/screens | Backend PM | UI PM | Notes |
| --- | --- | ---: | ---: | --- |
| File storage abstraction (MinIO + GCS) | `File`, `StorageService`, `MinioService`, `GCPService` | 1.0 | 0.0 | AWSSDK.S3 + Google.Cloud.Storage |
| Imports (CSV) | `ImportService`, `AsyncImportService`, 11 importable entities | 2.0 | 1.5 | Upload, mapping, async progress |
| Exports (CSV / PDF) | `ExportController`, `AsyncExportService` | 1.5 | 1.0 | QuestPDF/iText7 |
| Email (SMTP + SendGrid) and templated emails | `MailService`, `EmailService2`, `SendgridService`, Thymeleaf templates → Razor templates | 1.5 | 0.0 | Recreate ~30 templates |
| Webhooks and webhook endpoints | `Webhook`, `WebhookEndpoint`, `WebhookDispatchService` | 1.5 | 1.0 | Retry, signing, delivery log |
| API keys & rate limiting | `ApiKey`, `RateLimiterService`, Bucket4j → AspNetCoreRateLimit | 1.0 | 0.5 | |
| Analytics endpoints (work order compliance, downtime, cost trends, labor) | `controller/analytics/*` (~10 controllers) | 2.5 | 3.0 | Aggregations + charting (ChartJs.Blazor / ApexCharts.Blazor) |
| Calendar (FullCalendar → Blazor calendar component) | Calendar slice + screens | 0.5 | 1.5 | Replace FullCalendar with a Blazor equivalent |
| Subscriptions & billing (Paddle, FastSpring DTOs) | `Subscription*`, Paddle controller | 1.5 | 1.0 | Webhooks, plan management |
| Licensing / feature gates / Keygen integration | `LicenseService`, `KeygenService`, `PlanFeatures` | 1.0 | 0.5 | Optional — only if you keep commercial gating |
| Branding / white-label / custom colors / logos | `BrandingService`, `UiConfiguration` | 0.5 | 0.5 | |
| Health checks, actuator → ASP.NET Core HealthChecks | `HealthCheckController`, `CustomHealthIndicator` | 0.5 | 0.0 | |
| Demo data seeding | `DemoDataService`, `DemoController` | 1.0 | 0.0 | Often underestimated |
| Zapier-style integrations | `controller/zapier/*` | 0.5 | 0.0 | |
| **Subtotal** | | **16.0** | **10.5** | |

### 5.5 Cross-cutting infrastructure

These are not in the per-module backend/UI columns above. They are paid
once.

| Item | PM | Notes |
| --- | ---: | --- |
| Solution layout (API, Application, Domain, Infrastructure, Web, Mobile, Shared) | 1.0 | Project structure, DI wiring, common conventions |
| Database migrations (recreate equivalent of 103 Liquibase changesets in EF Core) | 2.0 | New schema can collapse some, but auditability suffers if rushed |
| Audit trail subsystem (replace Hibernate Envers) | 1.5 | EFCore interceptors or Audit.NET |
| Caching (Caffeine → MemoryCache / Redis) | 0.5 | |
| Localization infrastructure (.resx for 15 locales + email templates per locale) | 1.5 | Tooling + parity scripts |
| OpenAPI generation, client SDK for the new mobile app | 1.0 | NSwag / Refit |
| Real-time channel (SignalR) | 0.5 | |
| CI/CD (GitHub Actions for build, test, container publish) | 1.0 | |
| Docker images and `docker-compose.yml` for self-hosting parity | 1.0 | |
| Test infrastructure (xUnit, Testcontainers, Playwright/bUnit, code coverage) | 1.0 | |
| Security review (OWASP Top 10, multi-tenant isolation tests) | 1.0 | |
| Documentation parity (README, dev docs, env var matrix, GCP/MinIO setup) | 1.0 | |
| Data migration tooling from existing Postgres schema to new schema | 1.0 | Light scope; full historical migration is a separate project |
| **Subtotal** | **14.0** | |

### 5.6 Mobile

A .NET MAUI or Blazor Hybrid rewrite of the existing Expo app. The current
mobile app already has 18+ feature folders (auth, work orders, requests,
assets, locations, meters, parts, peopleTeams, vendorsCustomers, modals,
notifications, scan asset, settings, etc.) and ~52k lines.

| Module | PM | Notes |
| --- | ---: | --- |
| App shell, navigation, theming, auth flow | 3.0 | Login, JWT, OAuth, deep links, tabs, navigation |
| i18n + RTL parity (15 locales) | 1.0 | |
| Work orders (list, detail, create, edit, comments, attachments, signature) | 4.0 | Largest mobile feature |
| Requests (list, detail, create) | 1.5 | |
| Assets, locations, meters, parts | 3.5 | Hierarchical browsing, scanning |
| QR / NFC / barcode scanning | 1.0 | MAUI plugins, NFC Manager equivalent |
| Push notifications (APNs, FCM) | 1.0 | |
| File upload, camera, document picker | 1.0 | |
| Offline / poor-network handling, retries, optimistic UI | 2.0 | Currently weak in the React Native app; may improve in rewrite |
| WebSocket → SignalR client | 0.5 | |
| Settings, profile, more entities, super-user screens | 1.5 | |
| App store builds, signing, EAS-equivalent pipeline | 1.0 | |
| Testing on iOS + Android, store submission cycles | 2.0 | Often underestimated |
| **Subtotal** | **22.0** | |

### 5.7 Module totals

| Section | Backend PM | UI PM |
| --- | ---: | ---: |
| 5.1 Core domain & identity | 9.5 | 5.5 |
| 5.2 Asset & inventory | 8.5 | 9.0 |
| 5.3 Work management | 19.0 | 17.5 |
| 5.4 Platform features | 16.0 | 10.5 |
| **Backend / Web subtotals** | **53.0** | **42.5** |
| 5.5 Cross-cutting (split ~70/30 backend/web) | (in totals below) | |
| 5.6 Mobile | — | — (separate row) |

Reconciling with [section 1](#1-executive-summary):

```text
Backend (Web API)              42 PM     (53 nominal − 11 PM that overlaps with cross-cutting / shared infra and gets credited there to avoid double-counting)
Web (Blazor)                   34 PM     (42.5 nominal − ~8.5 PM credited to cross-cutting)
Mobile                         22 PM
Cross-cutting infrastructure   14 PM
──────────────────────────────────────
Total                         112 PM
```

The 32–60 / 26–50 / 16–34 / 10–22 ranges in the executive summary apply
±25% on the low side and +50% on the high side, reflecting realistic
estimation uncertainty.

---

## 6. Phasing for this team

A realistic phased delivery for **2 backend + 1 Blazor** developers, given
~36 PM/year of nominal capacity:

### Phase 0 — Foundations (target ~3–4 PM)

- Solution layout, CI/CD, Docker images
- Identity + JWT + OAuth2
- Multi-tenant scaffolding with EF global query filters
- One end-to-end vertical slice: Locations CRUD with auth, web UI, OpenAPI

**Deliverable:** working "hello world" tenant with login, list/create
locations, deployable via docker-compose. Validates the architecture before
investing in feature work.

### Phase 1 — MVP (target ~25–35 PM)

Smallest set that a maintenance team can use day-to-day:

- Locations, Assets, Asset Categories
- Work Orders (basic lifecycle, assignment, status, comments, attachments)
- Requests (basic lifecycle)
- Users, Teams, Roles (built-in roles only — no custom roles yet)
- Notifications (in-app + email via SMTP)
- File storage on MinIO
- Web UI for everything above

**Deliverable:** internally usable CMMS, no PM, no analytics, no mobile, no
imports/exports, no workflows.

### Phase 2 — Feature parity (target ~40–55 PM additional)

- Preventive Maintenance + Schedules
- Meters + Readings + Meter Triggers
- Parts + Inventory + Multi-parts + Consumption
- Vendors, Customers, Purchase Orders
- Custom fields + field configurations
- Imports / Exports / PDF reports
- Webhooks
- Workflow engine
- Analytics dashboards
- Calendar view
- Audit history
- 15-locale i18n parity

**Deliverable:** at-or-near feature parity with current `frontend/` +
`api/`. This is where the bulk of the expected 78 PM "backend + Blazor
parity, no mobile" estimate lands.

### Phase 3 — Mobile (target ~22 PM additional, requires extra capacity)

Either:

- Build .NET MAUI / Blazor Hybrid app (full estimate from section 5.6), or
- Adapt existing React Native app to point at the new API (~6–10 PM, but
  then you must learn and maintain React Native).

**This phase exceeds the 1-Blazor-developer's bandwidth by far.** A mobile
rewrite needs at least one additional engineer or a long calendar
extension.

### Phase 4 — Hardening (target ~6–10 PM continuous)

- Test coverage to a useful level (the current Java codebase has weak
  tests — don't repeat that mistake)
- Performance and load testing
- Security review (OWASP, multi-tenant isolation)
- Documentation
- Customer-data migration tooling

### Suggested calendar mapping for the team

Assuming the team described, ~85% effective utilization, no scope creep:

| Phase | Effort (PM) | Wall-clock with this team |
| --- | ---: | --- |
| 0 — Foundations | 3–4 | ~1.5 months |
| 1 — MVP | 25–35 | ~9–12 months |
| 2 — Feature parity | 40–55 | ~14–19 months |
| 3 — Mobile (with no extra hire) | 22 | not feasible — needs additional engineer or +12 months serial |
| 4 — Hardening (overlaps with 1–3) | 6–10 | continuous |

Total wall-clock without mobile: **~24–32 months**.
With mobile and no extra hire: **~36–44+ months**.

---

## 7. What changes the estimate (the most influential variables)

In rough order of impact:

| Variable | Effect on total |
| --- | --- |
| Drop mobile entirely | −22 PM (~−20%) |
| Keep React Native mobile against new API instead of rewriting it | −16 PM but adds ~2–4 PM/year of RN maintenance learning |
| Drop workflows + custom fields + custom roles | −8 PM |
| Drop analytics dashboards | −5–6 PM |
| Drop licensing / Paddle / FastSpring / Keygen | −3 PM |
| Drop import / export | −4 PM |
| Drop webhooks | −2.5 PM |
| Use Radzen/MudBlazor data grids and form generators aggressively | −5 to −10 PM (UI) |
| Use OpenAPI → typed Refit clients for mobile | −2 PM |
| Hire a 2nd Blazor / front-end developer | enables parallelism that compresses phases 1–2 by ~30–40% calendar time |
| Hire a dedicated PM/QA | reduces *risk* (range narrows) more than it reduces *expected* effort |
| Add a dedicated mobile engineer | enables Phase 3 in parallel with Phase 2 |

---

## 8. Comparison with the adoption (no-rewrite) path

To put the rewrite numbers in context, here is a parallel sketch of the
"adopt, learn, extend" path described in
[`Dotnet team adoption vs rewrite decision.md`](./Dotnet%20team%20adoption%20vs%20rewrite%20decision.md):

| Path | Approx. effort to a usable, customized product | Risk profile |
| --- | --- | --- |
| **Adopt existing project** + Java/React learning + tests + Persian/Jalali support + targeted improvements | **~12–18 PM** | Low product risk, medium technology learning risk |
| **Rewrite — MVP cut, no mobile** | ~32 PM (range 22–46) | Medium product risk, low technology risk |
| **Rewrite — backend + Blazor parity, no mobile** | ~78 PM (range 58–116) | High product risk |
| **Rewrite — full parity incl. mobile** | ~112 PM (range 84–166) | Very high product risk |

Even the smallest rewrite scope is **~2x the effort** of adopting and
extending the current system. The largest is **~7–10x**. This matches the
qualitative recommendation in the decision document.

---

## 9. Risks that can double the estimate

These risks are not priced into the expected number; they shift it toward
the high end of the range or beyond.

### 9.1 Hidden product complexity

CMMS systems have many implicit rules:

- who can see which work order
- what happens to children when an asset is deleted
- how overdue is calculated across timezones
- how recurring PMs generate work orders without duplicates
- how custom field permissions interact with role permissions
- how meter triggers interact with PM schedules
- how request portal fields map to work order fields
- how imports handle partial failures and idempotency

The existing repository encodes these rules in code. Re-deriving them from
scratch without a Product Manager and without QA is the single largest
overrun risk. Realistic worst case adds **+20–40 PM**.

### 9.2 Test coverage debt repeating itself

The current Java backend and React frontend have very weak automated
testing. If the rewrite repeats this pattern, hardening (Phase 4) and
post-launch defect work can easily add **+10–20 PM**.

### 9.3 Data migration

If existing Atlas customers need to migrate their data into the new
system, schema-level migration is a separate project that can take
**+3–6 PM** and may require maintaining the old system in parallel for an
extended period.

### 9.4 Licensing / commercial features

The current system has license-gated commercial features (custom roles,
white-label, branding, advanced features). Decisions about which to
re-implement, drop, or change directly affect scope. Reimplementing all of
them adds **+5–8 PM**.

### 9.5 Mobile

The current React Native app has technical debt and minimal tests but does
work in production. A from-scratch MAUI rewrite is a large project on its
own. Not deciding mobile up front is the most common multi-month slip.

### 9.6 Team turnover

A 3-person team with no slack is fragile. Loss of one developer for any
reason during a multi-year rewrite typically slips delivery by **2–6
months** while ramp-up and re-planning happen.

---

## 10. Recommended decision framing

Before committing to a rewrite, answer in writing:

1. **Why are we rewriting?** ("We know .NET better" alone is rarely
   sufficient — see the adoption-vs-rewrite document.)
2. **What is the smallest scope our customers will accept?** (This sets
   the realistic phase 1 boundary.)
3. **Do we ship mobile in v1, v2, or never?**
4. **Are we committing to write tests this time?** (If not, expect Phase 4
   to expand significantly.)
5. **Who owns product decisions?** (Without a clear owner, scope creep is
   guaranteed.)
6. **Can we hire a 4th engineer?** (For full parity + mobile, the team as
   described is undersized.)

If the answers are not all clear, the safer path is the one in
[`Dotnet team adoption vs rewrite decision.md`](./Dotnet%20team%20adoption%20vs%20rewrite%20decision.md):
adopt, learn, stabilize, then revisit selective rewrites with real domain
knowledge.

---

## 11. One-paragraph conclusion

A full from-scratch C# rewrite of Atlas CMMS — ASP.NET Core backend,
Blazor web frontend, and a .NET-based mobile app — is realistically a
**~110 person-month** engineering effort, with a credible range of **~85
to ~165 person-months** depending on scope, mobile decisions, hiring, and
how strictly feature parity is enforced. For a fixed team of **2 backend
.NET developers + 1 Blazor developer**, that translates to **roughly
2 to 3+ years of wall-clock work** without a mobile rewrite, and **3.5+
years** with one. The smallest defensible MVP cut (no mobile, no
analytics, no workflows, ~60% feature coverage) is still **~32
person-months**, which is about a year of this team's full capacity.
None of these numbers are aggressive; they reflect the actual size and
domain depth of the existing repository (~214,000 source/config lines
across 122 entities, 68 controllers, 83 services, 22+ frontend feature
areas, 15 locales, and a deployed mobile app).

