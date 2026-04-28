# .NET Team Decision: Adopt This CMMS or Rewrite in .NET Core and Blazor

This document is written for a team with:

- 2 backend developers experienced in .NET Core
- 1 Blazor developer
- limited existing Java/Spring Boot and React expertise

It compares two paths:

1. Adopt this existing Atlas CMMS project and learn enough Java/Spring Boot,
   React, and React Native to maintain and improve it.
2. Build a new CMMS from scratch using .NET Core and Blazor.

## Short answer

For most small teams, it is better to start from this existing project and learn
enough Java and React to maintain it, rather than rewriting a full CMMS from
scratch.

The reason is product scope.

This repository already includes many hard-to-recreate CMMS workflows:

- work orders
- preventive maintenance
- assets
- locations
- meters
- parts and inventory
- requests and request portal
- users, teams, roles, permissions
- notifications
- emails
- PDF reports
- CSV import/export
- analytics
- subscriptions/licensing
- webhooks
- mobile app
- Docker deployment

A rewrite in .NET Core and Blazor may feel easier technically at first because
the team knows the stack. But it would require rediscovering product behavior,
data models, edge cases, permissions, reporting rules, and mobile/API contracts.

The recommended path is:

```text
Adopt first, stabilize, learn the stack, then decide whether selective rewrites
are justified.
```

Do not begin with a full rewrite unless the goal is to build a different CMMS,
not just customize and improve this one.

## Decision summary

| Question | Recommendation |
| --- | --- |
| Need a working CMMS soon? | Use this project. |
| Need Persian language support? | Use this project and extend existing i18n. |
| Need deep custom business logic? | Start from this project, then refactor specific modules. |
| Need a pure .NET/Blazor codebase long-term? | Consider gradual replacement only after learning the product domain. |
| Want lowest technical learning curve for the team? | .NET rewrite is easier at first, but harder overall because of product scope. |
| Want lowest delivery risk? | Existing project is lower risk if you add tests and improve incrementally. |
| Want full ownership and no licensing gates? | A rewrite gives control, but costs much more product work. |

## Current project stack

The repository contains several applications:

```text
api/       Java Spring Boot backend
frontend/  React TypeScript main web app
mobile/    Expo / React Native mobile app
home/      Next.js marketing/home app
```

For the team described in this document, the most important parts are:

- `api/`: Java 17, Spring Boot, JPA/Hibernate, Liquibase, Spring Security,
  MapStruct, Quartz, WebSocket, mail, reports, licensing.
- `frontend/`: React 17, TypeScript, Material UI, React Router, Redux Toolkit,
  i18next, Formik/Yup, FullCalendar.
- `mobile/`: Expo 53, React Native, React Navigation, Redux Toolkit,
  i18next, native integrations.

## What is good about adopting this project?

### 1. Product depth already exists

The biggest advantage is not the technology. It is the existing product.

CMMS systems look simple at first, but the details grow quickly:

- work order lifecycle
- assignees and teams
- asset hierarchy
- request intake
- due dates and recurrence
- preventive maintenance
- meter triggers
- file attachments
- permissions
- custom fields
- exports
- notifications
- reports
- role visibility
- mobile scanning and field usage

This project already has many of those pieces.

Rebuilding them from scratch would require product analysis, UI design, database
design, API design, testing, migrations, and user feedback loops.

### 2. The backend architecture is understandable

The backend is not perfect, but it follows a recognizable Spring Boot monolith
shape:

```text
Controller -> Service -> Repository -> Database
```

It uses:

- Spring Boot 3
- Java 17
- Spring Data JPA
- Liquibase
- DTOs
- MapStruct
- Spring Security
- resource bundles for localization

A .NET backend team can learn this pattern because many concepts map cleanly to
.NET:

| Spring Boot concept | .NET Core equivalent |
| --- | --- |
| Controller | ASP.NET Core Controller / Minimal API endpoint |
| Service | Application service / domain service |
| Repository | EF Core repository/query service |
| JPA entity | EF Core entity |
| DTO | DTO / contract record |
| MapStruct | AutoMapper / manual mapping |
| Liquibase | EF migrations / DbUp / FluentMigrator |
| Spring Security | ASP.NET Core Authentication/Authorization |
| Bean / component | DI service |
| application.yml | appsettings.json / environment variables |

The team will need learning time, but the mental model is not alien.

### 3. React frontend has strong i18n and RTL foundations

For Persian work, the main frontend already has useful infrastructure:

- `i18next`
- lazy translation files
- supported language metadata
- date-fns locale loaders
- FullCalendar locale loaders
- RTL styling through `i18n.dir()` and `stylis-plugin-rtl`

That is a strong starting point for Persian language support.

Blazor can also handle localization, but building all equivalent screens and
interaction patterns from scratch is much bigger than adding `fa` to the existing
React app.

### 4. Docker self-hosting already exists

The project already has Docker Compose for:

- backend
- frontend
- PostgreSQL
- MinIO

This matters for operations. A new .NET/Blazor app would need its own deployment
story, storage setup, environment variables, migrations, background jobs, and
upgrade process.

### 5. Mobile app already exists

If mobile matters, this is a major advantage.

A .NET/Blazor rewrite usually means one of these:

- build a separate MAUI app
- build a Blazor Hybrid app
- build a responsive web app only
- postpone mobile

This repository already has an Expo/React Native app with real workflows. It has
technical debt, but it exists.

## What is risky about adopting this project?

### 1. The team must learn Java/Spring Boot and React

This is the obvious cost.

Your team knows .NET Core and Blazor. This project uses Java/Spring Boot and
React/React Native.

That means the team must learn:

- Maven
- Spring Boot dependency injection
- Spring Security
- JPA/Hibernate behavior
- Liquibase migrations
- Java build/test tooling
- React hooks
- Redux Toolkit
- Material UI
- React i18n
- React Native basics if mobile is important

This is real learning work. The question is whether learning the stack is
smaller than rebuilding the product. For a CMMS of this scope, learning the stack
is probably smaller.

### 2. The project has technical debt

Existing assessments found important issues:

- backend has almost no automated tests
- frontend has no authored tests
- mobile has no visible tests
- backend has some fat controllers
- backend enables lazy loading outside transactions
- frontend has a large `JWTAuthContext`
- mobile has a large `AuthContext`
- frontend and mobile TypeScript use `strict: false`
- frontend has duplicate HTTP clients
- mobile persists the whole Redux state
- licensing gates can confuse users

These are not reasons to rewrite immediately. They are reasons to adopt
carefully and add tests around every area you touch.

### 3. Licensing may affect your goals

The project is AGPL/open source, but it has runtime commercial feature gates.
For example, custom roles require the `CUSTOM_ROLES` license entitlement.

If your goal is to use every advanced feature freely without commercial
constraints, you must review the license situation carefully.

For a company internal fork, this becomes a legal and business decision. This
document is not legal advice.

### 4. Persian/Jalali support is more than translation

Persian language support is manageable.

True Jalali calendar support is harder because it touches:

- backend preferences
- date display
- date input
- reports
- exports
- mobile date pickers
- FullCalendar behavior
- timezone correctness

This would also be hard in a .NET/Blazor rewrite. The difficulty is product/date
semantics, not only framework choice.

## What is good about rewriting in .NET Core and Blazor?

### 1. Team productivity in a familiar stack

The team already knows:

- C#
- ASP.NET Core
- EF Core
- LINQ
- appsettings
- NuGet
- Blazor components
- .NET tooling

This reduces learning friction and can improve code quality if the team is much
stronger in .NET than Java/React.

### 2. Full control over architecture

A rewrite lets you design:

- clean modular monolith architecture
- stronger domain model
- better tests from day one
- centralized feature policy
- no unwanted licensing gates
- proper Persian/Jalali support from the start
- Blazor-first UI components
- a simpler deployment if you already run .NET infrastructure

### 3. Better long-term ownership if .NET is strategic

If your organization is strongly standardized on .NET, a .NET CMMS may be easier
to own over many years.

This matters if:

- hiring Java/React developers is difficult for you
- all other systems are .NET
- your DevOps pipeline is .NET-oriented
- internal libraries are .NET
- you want deep integration with existing .NET services

## What is risky about rewriting?

### 1. Rewriting a CMMS is a product project, not a coding task

The hard part is not creating controllers and pages. The hard part is product
coverage.

You would need to define and build:

- users
- companies
- roles
- permissions
- assets
- locations
- meters
- parts
- work orders
- requests
- preventive maintenance
- recurrence
- assignments
- notifications
- file storage
- imports
- exports
- reports
- analytics
- settings
- mobile behavior
- deployment
- auditing
- licensing or feature gates if needed

Each feature has edge cases.

### 2. You may underestimate the domain

CMMS products have many hidden rules:

- Who can see a work order?
- What happens when an asset is deleted?
- How are overdue work orders calculated?
- How do recurring PM schedules generate new work orders?
- How do custom fields affect forms and exports?
- How do permissions interact with teams?
- What must mobile users do offline or in poor network conditions?
- How should reports match user timezone and date format?

This repository already encodes many answers. A rewrite would have to rediscover
them.

### 3. Blazor UI speed may not compensate for missing product behavior

Blazor can be productive for a .NET team. But a CMMS has many screens, forms,
tables, filters, dialogs, and workflows.

Even if each Blazor page is easy to build, the total surface is large.

### 4. Mobile remains unresolved

If you rewrite backend and web in .NET/Blazor, what happens to mobile?

Options:

- Keep the existing React Native app and point it to a new .NET API.
- Rewrite mobile in .NET MAUI.
- Use Blazor Hybrid.
- Drop mobile temporarily.

Each option has cost. Keeping the existing mobile app means you still need
React Native knowledge. Rewriting mobile adds another large project.

## Decision matrix

Scores are pragmatic: 1 = weak, 5 = strong.

| Criterion | Adopt existing Java/React project | Rewrite in .NET/Blazor |
| --- | ---: | ---: |
| Fastest path to working CMMS | 5 | 1 |
| Team stack familiarity | 2 | 5 |
| Existing product coverage | 5 | 1 |
| Long-term .NET alignment | 1 | 5 |
| Persian translation effort | 4 | 3 |
| Jalali calendar effort | 3 | 3 |
| Mobile availability | 4 | 1-3 |
| Control over architecture | 2 | 5 |
| Risk of hidden product scope | 2 | 5 |
| Need to learn new stack | 2 | 5 |
| Ability to start delivering improvements | 4 | 1 |
| Ability to design tests from day one | 2 | 5 |

Interpretation:

- Existing project wins on product coverage and speed.
- Rewrite wins on stack familiarity and architectural control.
- Existing project is better if you need a CMMS.
- Rewrite is better if you need a .NET product platform and can accept a large
  product-building effort.

## Recommended strategy for this team

### Recommended path: adopt and learn first

For 2 .NET backend developers and 1 Blazor developer, the best strategy is:

```text
Adopt the existing project for the first phase.
Learn the minimum Java/React needed.
Add tests and documentation.
Implement Persian language support.
Postpone any rewrite decision until the team understands the CMMS domain.
```

Why:

- the existing system already runs
- the domain is large
- Persian language support is a bounded extension
- learning Java/Spring Boot is less risky than rebuilding CMMS behavior
- a rewrite decision will be better after real product understanding

### Suggested team split

Backend developer 1:

- learn Spring Boot controller/service/repository patterns
- own `api/` setup, database migrations, and tests
- study work orders, roles, permissions, and licensing

Backend developer 2:

- learn JPA/Hibernate and Liquibase
- own reports, exports, notifications, and date/calendar behavior
- add tests around backend Persian/Jalali changes

Blazor developer:

- learn React fundamentals
- own `frontend/` i18n, RTL layout, and UI verification
- gradually learn Material UI, Redux Toolkit, and Formik

All developers:

- write docs as they learn
- add tests before risky changes
- avoid broad rewrites in unfamiliar areas

## Learning roadmap

### Backend developers: learn enough Java/Spring Boot

Focus only on what is needed for this codebase:

1. Java basics for C# developers
   - classes, records, enums
   - streams
   - optionals
   - annotations
   - exceptions
2. Maven
   - `pom.xml`
   - dependencies
   - build/test lifecycle
3. Spring Boot
   - `@RestController`
   - `@Service`
   - dependency injection
   - configuration properties
4. Spring Data JPA
   - entities
   - repositories
   - lazy loading
   - transactions
5. Liquibase
   - changelog files
   - migrations
   - schema validation
6. Spring Security
   - JWT filters
   - `@PreAuthorize`
   - role/permission checks
7. Testing
   - JUnit
   - Mockito
   - Spring Boot tests
   - Testcontainers later

Do not try to become Java experts first. Learn by modifying small features.

### Blazor developer: learn enough React

Focus on:

1. React components
2. hooks: `useState`, `useEffect`, `useContext`
3. React Router
4. i18next translation usage
5. Material UI components
6. Formik/Yup forms
7. Redux Toolkit basics
8. CSS/RTL behavior

The Blazor mental model helps with components and state, but React has different
patterns. Start with translation and layout changes before large form changes.

## First 10 practical tasks after adoption

1. Run the full Docker Compose setup locally.
2. Document how to create a development company/user.
3. Add a backend smoke test that starts the Spring context.
4. Add tests for `Language.fromString` and `Helper.getLocale`.
5. Add a frontend translation key parity script for `en.ts` vs `fa.ts`.
6. Add Persian backend language enum and message bundle.
7. Add Persian frontend translation registration.
8. Verify RTL in main frontend screens.
9. Decide whether Jalali is required in the first release or later.
10. Add a small architecture decision record for every major change.

## When should you consider a rewrite?

Consider a .NET/Blazor rewrite only if several of these are true:

- the team cannot realistically maintain Java/React
- the existing licensing model is unacceptable for your goals
- you need a much smaller or much different CMMS
- mobile is not important or will be rebuilt separately
- you have strong internal .NET platform requirements
- you are willing to rebuild product behavior feature by feature
- you can invest in product management, QA, and domain discovery
- you want this to become your own long-term product, not just a customized CMMS

Even then, consider a phased rewrite instead of a big-bang rewrite.

## Safer alternative to full rewrite: strangler approach

If long-term .NET ownership is important, use a strangler pattern:

1. Keep the existing system running.
2. Add missing docs and tests.
3. Identify one bounded backend module.
4. Build a .NET service for that module only if there is strong reason.
5. Keep API contracts stable.
6. Move one feature at a time.

Possible candidates later:

- reporting service
- notification service
- import/export service
- analytics service
- integration/webhook service

Do not start by rewriting core work orders. Work orders are central and full of
domain rules.

## Cost comparison by work type

### Persian language support

Existing project:

- add backend `FA`
- add message bundle
- add frontend `fa.ts`
- add mobile `fa.ts` if needed
- verify RTL

.NET rewrite:

- build user/company preferences
- build localization infrastructure
- build all screens
- translate all strings
- implement RTL in Blazor UI

Winner: existing project.

### Jalali calendar support

Existing project:

- add `calendarSystem`
- centralize date formatting
- integrate Jalali frontend/mobile pickers
- update reports/exports

.NET rewrite:

- design `calendarSystem`
- implement date conversion
- build date pickers
- build reports/exports
- build all related screens

Winner: roughly equal for calendar logic, but existing project still has the
surrounding product already built.

### Licensing changes

Existing project:

- understand and adjust current licensing gates
- possibly improve docs and UI
- legal review if changing behavior

.NET rewrite:

- no licensing gates unless you add them
- but if you need commercial licensing, you must build or integrate it

Winner: depends on your goals.

### Core CMMS workflows

Existing project:

- understand and improve existing behavior

.NET rewrite:

- define and build everything

Winner: existing project.

## Risks if you adopt existing project

- Java/Spring learning curve
- React learning curve
- technical debt
- tests are weak
- licensing gates may affect expectations
- some architecture cleanup is needed
- dependency upgrades may be needed over time

Mitigation:

- train through small tasks
- write tests as you touch code
- document every feature learned
- avoid large refactors until the team understands the domain
- keep a backlog of technical debt

## Risks if you rewrite

- underestimating product scope
- long time before feature parity
- no mobile app unless separately built
- unclear migration path for users/data
- many hidden CMMS rules rediscovered late
- risk of building a technically clean but incomplete product
- QA burden grows quickly

Mitigation:

- only rewrite after a discovery phase
- build a feature matrix
- prototype one vertical slice first
- keep existing app running as reference
- migrate gradually

## Final recommendation

For this specific team, the best path is:

```text
Use this project.
Learn enough Java/Spring Boot and React to maintain it.
Improve it incrementally.
Do not rewrite from scratch now.
```

The team size is small. A full CMMS rewrite is large. The existing project has
technical debt, but it also has significant product value.

Use the existing codebase as a working product and learning platform. Start with
bounded improvements:

- Persian language support
- documentation
- tests
- frontend RTL fixes
- backend cleanup around touched modules
- date/calendar design

After the team has maintained the project for a while and understands the CMMS
domain deeply, revisit the rewrite question. At that point, you can make a much
better decision about whether a selective .NET rewrite is worth it.

## One-sentence conclusion

For a 2-backend + 1-Blazor .NET team, learning enough Java/React to improve this
existing CMMS is likely a better investment than rebuilding a CMMS from scratch
in .NET Core and Blazor.
