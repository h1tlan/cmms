# Backend Architecture Assessment

This document answers three questions about the backend in `api/`:

1. Is the backend architecture fine?
2. Is it maintainable and aligned with common Spring Boot best practices?
3. Is it worth working on this project?

The short answer is:

- The backend is workable and worth working on if you are comfortable improving
  a real-world Spring Boot monolith.
- It has a recognizable architecture and many good building blocks.
- It also has maintainability and production-readiness risks that should be
  addressed before large feature work becomes too expensive.
- It does not need a rewrite before adding Persian language support, but it
  would benefit from focused cleanup before deeper calendar, scheduling, or
  reporting changes.

## Executive verdict

### Is the backend architecture fine?

Mostly yes, with caveats.

The backend follows a conventional Spring Boot monolith shape:

- controllers
- services
- repositories
- JPA entities
- DTOs
- MapStruct mappers
- Liquibase migrations
- Spring Security configuration
- background jobs
- utility classes

That is a reasonable structure for this type of CMMS/productivity product. A
new engineer can usually find where things live.

The main problem is not the high-level structure. The problem is that some
classes have grown too broad, some layers are blurred, and there is almost no
automated testing safety net.

### Is it maintainable?

It is maintainable today, but not comfortably maintainable at scale.

Small features are likely possible without major architectural work. Larger
features, especially ones touching work orders, permissions, exports, reports,
or date/calendar logic, carry more risk because:

- some controllers are large and transactional
- services contain mixed responsibilities
- domain entities leak API/web concerns
- lazy-loading behavior can hide query and transaction problems
- automated tests are effectively absent

Maintainability can improve significantly with incremental refactoring. The
project does not require a big-bang rewrite.

### Does it follow best practices?

It follows many Spring Boot best practices, but misses some important ones.

Good:

- Spring Boot 3 and Java 17
- Spring Data JPA repositories
- Liquibase with `ddl-auto: validate`
- MapStruct for mapping
- DTOs for many API surfaces
- BCrypt password hashing
- stateless JWT security
- validation and OpenAPI dependencies
- caching, Quartz, WebSocket, actuator, mail, LDAP/OAuth integration

Needs improvement:

- no meaningful automated backend tests
- `hibernate.enable_lazy_load_no_trans: true`
- startup code creates business data and a default super admin
- hardcoded default super admin credentials
- fat controllers with transaction boundaries in the web layer
- raw exception messages returned for generic 500 errors
- Swagger and public routes should be audited for production exposure
- dependency/version mismatches should be cleaned up

### Is it worth working on?

Yes, if your goal is to improve and extend an existing product.

This is not a toy backend. It has many product features already implemented:

- work orders
- preventive maintenance
- assets
- locations
- meters
- parts
- requests
- request portal
- users, teams, roles, permissions
- imports/exports
- notifications
- emails
- reports
- subscriptions/licensing
- webhooks

That makes it worth working on because there is real product surface area.

However, you should treat the backend as a mature monolith with technical debt,
not as a clean greenfield codebase. The best path is to add features while also
improving tests, boundaries, and risky infrastructure areas.

## Backend shape

The backend lives in:

```text
api/
```

It is a Java 17 Spring Boot application. The Maven parent is Spring Boot 3.2.3
in `api/pom.xml`.

Important top-level packages under `api/src/main/java/com/grash/` include:

- `controller`
- `service`
- `repository`
- `model`
- `dto`
- `mapper`
- `configuration`
- `security`
- `exception`
- `job`
- `utils`
- `advancedsearch`
- `event`
- `factory`

This package layout is familiar and generally good for onboarding.

## Strengths

### 1. Conventional Spring layering

The project broadly follows:

```text
Controller -> Service -> Repository -> Database
```

This is a good baseline. It is easier to understand than a highly custom
architecture.

### 2. Good persistence foundation

The backend uses:

- Spring Data JPA
- PostgreSQL
- Liquibase
- Hibernate Envers on some audited entities
- repository interfaces and specifications

`application.yml` uses:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  liquibase:
    enabled: true
```

This is a good production pattern: schema changes should come from migrations,
not Hibernate auto-DDL.

### 3. DTO and mapper usage

The codebase has a large DTO tree and MapStruct mappers. That is generally good
because API shapes can be separated from entity shapes.

The current implementation does not keep that separation perfectly everywhere,
but the presence of DTOs and mappers is a strong starting point.

### 4. Feature-rich backend

This project already contains many real application concerns:

- authentication and authorization
- role and permission models
- asset and work-order management
- preventive maintenance jobs
- import/export
- PDF generation
- mail templates
- file storage
- subscriptions
- API keys
- webhooks
- request portal
- real-time WebSocket features

That means the backend has real business value. It is worth improving rather
than discarding.

### 5. Security building blocks exist

The backend has:

- stateless sessions
- JWT filter
- API key filter
- BCrypt password encoder with strength 12
- method security enabled
- OAuth2 support
- LDAP support
- rate-limit filter

These are useful building blocks, even though the current security config
should still be audited.

### 6. Localization pattern already exists

For Persian language support specifically, the backend already has a usable
server-side localization pattern:

- `Language` enum
- `Helper.getLocale(...)`
- `messages.properties`
- `messages_*.properties`
- Thymeleaf email/PDF templates
- locale-aware `MessageSource` usage

Adding Persian as `FA` and `messages_fa_IR.properties` fits the current
architecture.

## Maintainability risks

### 1. Startup performs business initialization

`ApiApplication` implements `SmartInitializingSingleton` and performs business
setup after beans are created:

- creates super admin role/user
- creates subscription plans
- updates default roles
- checks usage limits
- updates temporary time zones

Relevant file:

```text
api/src/main/java/com/grash/ApiApplication.java
```

This is risky because application startup becomes tied to mutable business
operations. It can make deployments, tests, and one-off maintenance commands
harder to reason about.

Better options:

- move static seed data to Liquibase migrations
- move first-admin setup to a one-time setup command
- make bootstrap jobs profile-gated
- require first-admin credentials through environment variables

### 2. Hardcoded default super admin credentials

`ApiApplication` currently builds a default super admin with:

```java
signupRequest.setEmail("superadmin@test.com");
signupRequest.setPassword("pls_change_me");
```

Even if intended for initial setup, credentials in code are an operational and
security risk.

Recommended fix:

- remove hardcoded password
- require setup through environment variables or CLI/admin onboarding
- fail startup if production setup is incomplete
- document first-admin creation clearly

### 3. Fat controllers

Some controllers do too much. `WorkOrderController` is a strong example:

```text
api/src/main/java/com/grash/controller/WorkOrderController.java
```

It injects many services and handles orchestration for work order actions,
notifications, reports, PDF generation, and persistence concerns. It is also
annotated with `@Transactional` at class level.

This makes the web layer too powerful.

Better pattern:

```text
Controller -> Application service/use case -> Domain services/repositories
```

Controllers should mostly:

- validate/request-bind input
- call one service method
- map response DTOs
- return HTTP responses

Transaction boundaries usually belong in service/application-service methods,
not controllers.

### 4. Layer boundaries are blurred

Some model classes import DTOs, HTTP classes, or exceptions. This means domain
objects are not fully independent from web/API concerns.

This is not fatal, but it raises maintenance costs because changes in HTTP/API
behavior can ripple into persistence/domain models.

Preferred direction:

- JPA entities should focus on persistence/domain state
- DTOs should describe API input/output
- mappers should translate between them
- exceptions should be raised from services, not embedded into entity-style
  model behavior where avoidable

### 5. Lazy loading outside transactions is enabled

`application.yml` contains:

```yaml
hibernate:
  enable_lazy_load_no_trans: true
```

This setting is often a maintainability foot-gun. It can hide missing
transaction boundaries and produce unexpected extra queries.

Better approach:

- use service-level `@Transactional`
- fetch required associations explicitly
- use fetch joins, `@EntityGraph`, DTO projections, or query-specific methods
- remove `enable_lazy_load_no_trans` after the code is ready

### 6. Weak automated test coverage

There is only one test file:

```text
api/src/test/java/com/grash/ApiApplicationTests.java
```

Its contents are commented out, so the backend effectively has no automated
test suite.

This is the biggest risk for future development.

Without tests, changes to these areas are risky:

- authentication
- permissions
- work orders
- preventive maintenance scheduling
- reports
- exports
- Liquibase migrations
- date/calendar behavior
- localization

Before investing heavily in new features, add a minimal test safety net.

## Best-practice review

### What follows best practices

The backend does well in these areas:

- uses Spring Boot rather than custom infrastructure
- uses Java 17
- uses Spring Data repositories
- uses Liquibase and validates schema
- externalizes many secrets/config values through environment variables
- uses DTOs and MapStruct
- uses validation dependencies
- uses BCrypt for password hashing
- separates many configs into `configuration/`
- uses `MessageSource` resource bundles for localization
- uses UTF-8 message configuration

### What does not follow best practices yet

The main gaps are:

- no meaningful tests
- startup code mutates business data
- hardcoded initial credentials
- fat controllers
- transactions in controllers
- lazy loading outside transactions
- raw `printStackTrace()` in global exception handling
- raw exception messages returned for generic 500 errors
- broad `permitAll` security surface that needs regular review
- Swagger/OpenAPI paths are publicly permitted
- possible dependency mismatch: `thymeleaf-spring5` with Spring Boot 3
- old `aspectjrt` version compared with the rest of the stack
- Liquibase master includes generated output/diff changelogs
- one Liquibase include/file has a leading space in its filename

## Security assessment

The backend has good security ingredients, but the configuration should be
audited before production confidence is high.

Positive:

- stateless security
- JWT authentication
- API key filter
- rate limiting
- BCrypt strength 12
- method security enabled
- optional LDAP/OAuth support

Risks:

- broad `permitAll` list in `WebSecurityConfig`
- public Swagger paths
- public webhook paths need strict verification/authentication
- public request portal upload endpoints need abuse controls
- access denied is configured with `accessDeniedPage("/login")`, which is odd
  for a stateless JSON API
- `WebSecurityCustomizer` ignores `/com/grash/configuration/**`, which looks
  like a package path rather than a real public URL

Recommended security cleanup:

1. Document every public endpoint and why it is public.
2. Gate Swagger in production.
3. Return JSON 401/403 for API clients.
4. Verify webhook signature validation.
5. Review public upload limits and scanning strategy.
6. Add security tests for public/private endpoint behavior.

## Error handling assessment

`GlobalExceptionHandlerController` centralizes error handling, which is good.

However, it uses:

```java
ex.printStackTrace();
return new SuccessResponse(false, ex.getMessage());
```

For generic exceptions, returning `ex.getMessage()` to clients can leak
implementation details.

Recommended approach:

- use a logger instead of `printStackTrace()`
- return generic messages for 500 errors
- include request IDs/correlation IDs in logs
- keep detailed messages for expected validation/business errors only
- normalize API error response shape

## Database and migration assessment

Liquibase is a good choice and the project uses it. That part is positive.

Concerns:

- `master.xml` includes generated files like `liquibase-outputChangeLog.xml`
  and `liquibase-diffChangeLog.xml`
- generated changelogs can be noisy or environment-specific if not curated
- one included changelog filename starts with a leading space

Recommended approach:

- keep generated diffs out of the production migration chain unless reviewed
- write small, named, human-reviewed changesets
- avoid odd filenames
- add migration tests against PostgreSQL/Testcontainers

## Dependency assessment

The stack is mostly reasonable, but there are version hygiene issues.

Examples:

- Spring Boot parent is 3.2.3.
- Java is 17.
- MapStruct 1.5.5 is reasonable.
- Liquibase 4.22 is reasonable.
- `thymeleaf-spring5` is suspicious with Spring Boot 3/Spring 6.
- `aspectjrt` 1.8.7 is old compared with the rest of the stack.
- iText dependencies are older and should be checked for licensing and security
  implications.

Recommended approach:

- run dependency vulnerability checks
- align Thymeleaf with Spring 6
- update old libraries carefully
- avoid dependency churn while feature work is active unless the dependency is
  risky or blocking

## Is it okay for Persian language support?

Yes.

Persian language support is a reasonable feature to add to this backend.

Why:

- the backend already models company language
- server-side message bundles already exist
- Java locale mapping already exists
- emails/reports already receive locale information
- DTOs already expose language preferences

The backend does not need architectural refactoring before adding `FA`.

Minimum backend Persian language work:

1. Add `FA` to `Language` at the end of the enum.
2. Map `FA` to `new Locale("fa", "IR")`.
3. Add `messages_fa_IR.properties`.
4. Verify templates render Persian text safely.
5. Confirm signup/preferences/public portal DTO flows accept and return `FA`.

## Is it okay for Persian/Jalali calendar support?

Partially.

The backend can support Jalali calendar behavior, but it should be designed
carefully.

Do not put Jalali into `DateFormat`.

Current `DateFormat` values:

```java
MMDDYY, DDMMYY
```

These represent display order, not calendar system.

Recommended model:

```java
public enum CalendarSystem {
    GREGORIAN,
    JALALI
}
```

Add `calendarSystem` to `GeneralPreferences` and keep:

- API transport dates as Gregorian/ISO
- persisted dates as existing Java date/time values
- Jalali conversion at user-facing output/input boundaries

Backend calendar work should be more cautious than language work because it can
affect reports, exports, analytics, scheduling, recurrence, and date ranges.

## Should you work on this project?

Yes, if you approach it strategically.

This is a good project to work on if:

- you want to improve a real Spring Boot product
- you are willing to add tests as you touch risky areas
- you prefer incremental refactoring over rewriting
- you can tolerate existing technical debt
- you value a project with many implemented features

Be careful if:

- you need a pristine architecture
- you expect high test coverage from day one
- you need to make large cross-cutting changes quickly
- you cannot invest in security and migration review

The best mindset is:

> Treat this backend as a valuable existing system that needs disciplined
> maintenance, not as a broken system that needs a rewrite.

## Recommended improvement roadmap

### Priority 1: Safety net

Add minimal tests before big refactors:

- context load test
- controller/auth smoke tests
- repository tests with Testcontainers
- Liquibase migration test
- `Language` and localization tests
- date formatting tests before Jalali work

### Priority 2: Security and production cleanup

- remove hardcoded super admin password
- gate Swagger in production
- review public endpoint list
- fix stateless API 401/403 behavior
- replace `printStackTrace()`
- avoid leaking generic exception messages

### Priority 3: Persistence discipline

- reduce reliance on `enable_lazy_load_no_trans`
- move transactions into services
- use fetch joins/entity graphs/projections intentionally
- clean Liquibase generated/fragile changelogs

### Priority 4: Layering improvements

- slim large controllers
- introduce application services for large use cases
- keep `EntityManager` out of controllers where possible
- reduce entity coupling to DTO/web concerns

### Priority 5: Feature work

Then add:

- Persian language support
- frontend/backend language parity checks
- calendar-system preference
- Jalali report/export formatting
- Jalali-aware frontend date input/display behavior

## Practical recommendation for your next work

If your goal is Persian support, use this sequence:

1. Add Persian language support first.
2. Add tests for `Language`, `Helper.getLocale`, and message key parity.
3. Verify emails, reports, and exports with Persian text.
4. Add `CalendarSystem` as a separate preference.
5. Add a backend date formatting helper for user-facing strings.
6. Only then update reports/exports for Jalali output.

This order avoids mixing low-risk translation work with higher-risk calendar
semantics.

## Final conclusion

The backend architecture is acceptable but not excellent.

It is a conventional, feature-rich Spring Boot monolith with a good foundation:
JPA, Liquibase, MapStruct, DTOs, security integrations, jobs, localization, and
clear package names.

Its biggest weaknesses are maintainability risks, not architectural impossibility:

- large controllers
- blurred layers
- startup side effects
- lazy loading outside transactions
- weak error handling
- broad security exposure
- dependency hygiene issues
- no meaningful automated tests

It is worth working on if you also improve the engineering foundation as you add
features. For Persian language support, the backend is ready enough. For
Jalali calendar support, add a careful model and tests first.
