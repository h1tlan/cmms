# Frontend Architecture Assessment

This document answers three questions about the main web frontend in
`frontend/`:

1. Is the frontend architecture fine?
2. Is it maintainable and aligned with common React/TypeScript best practices?
3. Is it worth working on this project?

The short answer is:

- The frontend is worth working on if you are comfortable improving a mature
  React application with real product surface area.
- It has a usable architecture, strong i18n foundations, and enough structure to
  support Persian language work.
- It also has technical debt around tests, typing, HTTP/API boundaries, large
  contexts, aging dependencies, and date handling.
- Persian language and RTL support are reasonable incremental changes.
- True Jalali calendar support is a bigger cross-cutting change and should be
  done after tightening the date layer and adding tests.

## Executive verdict

### Is the frontend architecture fine?

Mostly yes, with caveats.

The app is a conventional Create React App style React application:

- React 17
- TypeScript
- Material UI v5
- React Router v6
- Redux Toolkit slices
- Formik and Yup forms
- i18next and react-i18next
- date-fns and FullCalendar
- feature pages under `src/content/`
- shared models under `src/models/`
- shared slices under `src/slices/`
- routing under `src/router/`
- theming under `src/theme/`

This is a recognizable architecture. A developer can usually find routes,
state, models, feature pages, translations, and theme behavior without learning
a custom framework.

The caveat is that several areas have grown organically. Some app-level
contexts and effects do too much, HTTP access is split between two clients,
TypeScript guardrails are weak, and automated tests are effectively absent.

### Is it maintainable?

It is maintainable for normal feature work, but risky for large cross-cutting
changes.

Persian translations and RTL verification are manageable because the i18n and
theme foundations already exist.

Jalali calendar support is more complex because date display, date pickers,
FullCalendar, hardcoded date formats, and backend API contracts all intersect.
That work should be centralized behind a date/calendar abstraction rather than
patched into individual screens.

### Does it follow best practices?

It follows several important practices:

- lazy-loaded routes
- centralized routing
- typed Redux hooks
- DTO/model types
- centralized theme provider
- translation bundles
- language detection
- RTL styling support through Emotion and Stylis
- Formik/Yup form patterns

But it misses or weakens other important practices:

- no authored frontend test suite
- `strict: false` in TypeScript
- many ESLint rules disabled
- `any` in shared store return typing
- two HTTP clients with different error behavior
- large `JWTAuthContext` with too many responsibilities
- state updates during render in the auth guard
- MUI date pickers are not wired to the active locale
- dependency drift and overlapping libraries

### Is it worth working on?

Yes.

The app is not a clean greenfield frontend, but it represents a real product
with significant implemented workflows:

- work orders
- preventive maintenance
- assets
- parts
- meters
- locations
- requests and request portal
- settings
- analytics
- imports/exports
- notifications
- subscriptions and licensing

That makes it worth improving. The right approach is incremental feature work
plus targeted cleanup, not a rewrite.

## Frontend shape

The main frontend lives in:

```text
frontend/
```

Important areas:

```text
frontend/src/App.tsx
frontend/src/index.tsx
frontend/src/router/
frontend/src/content/
frontend/src/layouts/
frontend/src/slices/
frontend/src/store/
frontend/src/models/
frontend/src/contexts/
frontend/src/theme/
frontend/src/i18n/
frontend/src/utils/
```

This organization is generally understandable:

- `router/` defines route trees.
- `content/` contains feature screens.
- `slices/` contains Redux Toolkit state.
- `models/` contains TypeScript domain/API shapes.
- `contexts/` contains cross-cutting providers such as auth and company
  settings.
- `theme/` contains MUI theme and RTL handling.
- `i18n/` contains language setup and translations.
- `utils/` contains API wrappers, date helpers, and general helpers.

## Strengths

### 1. Clear enough package structure

The frontend has a predictable folder layout. Feature pages, models, slices,
routes, theme, translations, and utilities are separated into recognizable
locations.

This is good for onboarding and for adding Persian support because translation
files and i18n registration are easy to find:

```text
frontend/src/i18n/i18n.ts
frontend/src/i18n/translations/
```

### 2. Route code-splitting exists

Routes use lazy loading and `Suspense` wrappers. That is a good pattern for a
large app because it avoids loading every page upfront.

Relevant files:

```text
frontend/src/router/index.tsx
frontend/src/router/app.tsx
frontend/src/router/account.tsx
frontend/src/router/analytics.tsx
```

### 3. Redux Toolkit is used

The store is configured with Redux Toolkit:

```text
frontend/src/store/index.ts
frontend/src/store/rootReducer.ts
frontend/src/slices/
```

Typed `useSelector` and `useDispatch` wrappers exist, which is a good baseline
for TypeScript React state management.

### 4. i18n foundation is strong

The frontend already has:

- `i18next`
- `react-i18next`
- browser language detection
- lazy translation loaders
- supported language metadata
- date-fns locale loaders
- FullCalendar locale loaders
- English fallback

Relevant file:

```text
frontend/src/i18n/i18n.ts
```

This makes Persian language support a reasonable incremental change.

### 5. RTL foundation exists

The theme provider already switches styling direction based on `i18n.dir()`:

```text
frontend/src/theme/ThemeProvider.tsx
```

It uses:

- `stylis-plugin-rtl`
- separate Emotion caches
- `document.documentElement.setAttribute('dir', 'rtl')`
- MUI theme direction through theme schemes

This is an important strength for Persian. The app already supports Arabic
translation files, so Persian can reuse the same RTL mechanism.

### 6. Product surface is valuable

The frontend is large because it supports a real application, not because it is
just accidental complexity. Existing workflows make the project worth improving.

## Maintainability risks

### 1. There is no visible authored test suite

The frontend has no `test` script in `frontend/package.json`, and no
`*.test.*` or `*.spec.*` files were found under `frontend/src`.

That means regressions in these areas are hard to catch automatically:

- auth redirects
- permissions
- route guards
- settings updates
- i18n loading
- RTL layout behavior
- work order flows
- date formatting
- date pickers
- analytics filters
- request portal

This is the biggest long-term maintainability issue.

Before deep Jalali calendar work, add tests around date formatting and critical
date input flows.

### 2. TypeScript guardrails are weak

`frontend/tsconfig.json` has:

```json
"strict": false
```

It also allows JavaScript:

```json
"allowJs": true
```

This makes the app easier to keep compiling, but harder to refactor safely.

Recommended direction:

- enable stricter checks gradually
- start with new files using strong types
- consider enabling `strictNullChecks` first
- remove shared `any` types over time

### 3. ESLint guardrails are weakened

`frontend/.eslintrc.json` extends useful presets, but many rules are disabled,
including unused variables and several TypeScript safety rules.

Examples:

```json
"@typescript-eslint/no-unused-vars": ["off"],
"@typescript-eslint/no-shadow": ["off"],
"@typescript-eslint/naming-convention": ["off"]
```

This is not fatal, but it reduces the value of linting as a quality gate.

### 4. Two HTTP clients exist

The app has a fetch-based API wrapper:

```text
frontend/src/utils/api.ts
```

It also has an axios instance:

```text
frontend/src/utils/axios.ts
```

This creates inconsistency in:

- auth headers
- base URL handling
- error handling
- mocking strategy
- future token refresh behavior
- global request/response instrumentation

Recommended direction:

- choose one HTTP client
- define one structured error shape
- centralize auth headers
- centralize public/private route behavior
- remove or isolate mock-only clients

### 5. Error handling from API calls is awkward

`frontend/src/utils/api.ts` throws:

```ts
throw new Error(JSON.stringify(await response.json()));
```

Then `getErrorMessage` parses `error.message`.

This works, but it is not ideal. A typed API error object would be easier to
handle and test.

Recommended direction:

```ts
type ApiError = {
  status: number;
  message: string;
  details?: unknown;
};
```

Then throw or return a consistent structure.

### 6. `JWTAuthContext` is too broad

Relevant file:

```text
frontend/src/contexts/JWTAuthContext.tsx
```

It handles much more than authentication:

- login/logout/register
- user and company loading
- user settings
- company settings
- general preferences
- subscription actions
- password reset/update
- account switching
- permission checks
- feature entitlement checks
- UI configuration
- analytics/tracking side effects
- language switching

This makes the context hard to test and hard to change safely.

Recommended direction:

- keep session/auth concerns in auth context
- move company settings to a separate provider or query layer
- move permission helpers into a permission hook/service
- move subscription actions into a billing hook/service
- keep language application as a small side effect of loaded preferences

### 7. Auth guard updates state during render

Relevant file:

```text
frontend/src/components/Authenticated/index.tsx
```

Current pattern:

```ts
if (!auth.isAuthenticated) {
  if (location.pathname !== requestedLocation) {
    setRequestedLocation(location.pathname);
  }
  return <Login />;
}
```

Calling `setState` during render is a React anti-pattern. It can cause extra
renders and unexpected behavior.

Recommended direction:

- use `useEffect` to capture requested location
- use router state for redirects where possible
- keep render paths pure

### 8. App-level redirects are spread through effects

`frontend/src/App.tsx` contains app-wide redirect effects for subscription
upgrade/downgrade and account switching.

That can work, but it makes route behavior harder to reason about because route
access is controlled both by router configuration and by app-level effects.

Recommended direction:

- centralize route guards
- document global redirects
- test redirect precedence
- avoid redirect loops by using precise dependencies and guard conditions

### 9. Date handling is not centralized enough for Jalali

Current central display formatting exists in:

```text
frontend/src/contexts/CompanySettingsContext.tsx
```

But some components bypass it with hardcoded formats, and date pickers use MUI
`LocalizationProvider` without passing the active locale.

Known related files:

```text
frontend/src/contexts/CompanySettingsContext.tsx
frontend/src/i18n/i18n.ts
frontend/src/hooks/useDateLocale.tsx
frontend/src/content/own/components/form/DateRangePicker.tsx
frontend/src/content/own/Analytics/CustomDateRangePicker.tsx
frontend/src/App.tsx
```

For Persian language only, this is manageable.

For true Jalali calendar support, centralize date display and date input
conversion before changing many screens.

### 10. Dependency drift exists

The stack has several older or overlapping dependencies:

- React 17
- Create React App / `react-scripts` 5
- `react-app-rewired`
- `axios` 0.27.2
- MUI v5 plus `@mui/core` alpha
- `@mui/styles`, which is legacy in MUI v5+
- `i18next` 25 with `react-i18next` 11
- `date-fns`, `date-fns-tz`, and `dayjs`
- `stompjs`, `@stomp/stompjs`, and `websocket`

This does not mean the app is bad, but it does mean dependency upgrades should
be planned carefully rather than mixed casually into feature work.

## Best-practice review

### What follows best practices

The frontend does well in these areas:

- uses TypeScript
- has feature/domain model types
- uses React Router v6 route objects
- lazy-loads route components
- uses Redux Toolkit
- uses typed Redux hooks
- centralizes theme setup
- uses i18next for translations
- lazy-loads translation bundles
- handles RTL at styling-cache level
- uses Formik/Yup for form validation
- keeps language list and metadata in one i18n module

### What does not follow best practices yet

Main gaps:

- no authored test suite
- TypeScript strict mode disabled
- important lint rules disabled
- shared `any` in store thunk return typing
- duplicate API clients
- API errors encoded into `Error.message`
- large context doing too many jobs
- render-time state updates in auth guard
- date picker locale not wired to active language
- dependency overlap and version skew

## Security and reliability assessment

Frontend security depends heavily on backend enforcement, but the frontend still
matters.

Positive:

- JWT token is attached through a central helper for the fetch API wrapper.
- Authenticated routes are guarded.
- Permission helpers exist.
- Public request portal has its own path and language behavior.

Risks:

- Tokens live in `localStorage`, which is common but vulnerable to XSS impact.
- Dual HTTP clients can cause inconsistent auth behavior.
- Public route behavior in `authHeader(publicRoute)` is not really expressed by
  the main API wrapper because it calls `authHeader(false)`.
- Weak lint/typing guardrails can allow unsafe patterns to remain unnoticed.

Recommended direction:

- maintain strict backend authorization as the real security boundary
- reduce XSS risk through dependency hygiene and code review
- unify HTTP clients
- document public API calls
- add tests for auth guard and permission-driven rendering

## Persian language readiness

The frontend is ready enough for Persian language support.

Why:

- translation files are already per-language TypeScript modules
- `i18n.ts` centralizes language registration
- language detection already maps `fa-IR` to `fa`
- settings and request portal selectors read from `supportedLanguages`
- authenticated users are switched to company language
- RTL is already wired through `i18n.dir()`

Minimum work:

1. Add `frontend/src/i18n/translations/fa.ts`.
2. Add `fa` to `translationLoaders`.
3. Add `fa` to `dateLocaleLoaders`.
4. Add `fa` to `calendarLocaleLoaders` if available in FullCalendar package.
5. Add `FA` to `SupportedLanguage`.
6. Add Persian to `supportedLanguages`.
7. Verify settings, registration, authenticated app loading, and request portal.
8. Set `<html lang="fa">` dynamically along with `dir`.
9. Test RTL screens manually.

This is worth doing.

## Jalali calendar readiness

The frontend is not ready for true Jalali support without additional design.

Why:

- current date display uses Gregorian `date-fns`
- date pickers are Gregorian
- FullCalendar locale translates labels but does not create a Jalali calendar
  grid
- some components use hardcoded date formats
- MUI `LocalizationProvider` does not receive the active locale
- no tests exist around date behavior

Recommended model:

- backend exposes `calendarSystem: 'GREGORIAN' | 'JALALI'`
- frontend models include `calendarSystem`
- display helpers convert Gregorian API dates to Jalali strings when selected
- date pickers convert Jalali selections back to Gregorian API values
- API payloads remain Gregorian/ISO

Recommended implementation approach:

1. Add tests for current Gregorian formatting.
2. Add a central date/calendar utility.
3. Update `CompanySettingsContext.getFormattedDate` to delegate to it.
4. Audit hardcoded `format(...)` and `toLocaleDateString(...)` calls.
5. Decide on a Jalali picker dependency or custom picker.
6. Keep FullCalendar label localization separate from true Jalali grid behavior.

## Should you work on this frontend?

Yes, if you approach it as an existing product frontend that needs disciplined
maintenance.

This is a good project to work on if:

- you want to improve a real React app
- you are okay with incremental cleanup
- you can add tests around the areas you touch
- you are comfortable with MUI, Redux Toolkit, i18next, and Formik
- you value feature depth over pristine architecture

Be careful if:

- you need modern React 18/19 patterns immediately
- you expect strict TypeScript coverage from day one
- you need high confidence without writing tests
- you want to make broad date/calendar changes without refactoring

The best mindset is:

> Treat the frontend as a valuable mature app with technical debt, not as a
> broken app that needs a rewrite.

## Recommended improvement roadmap

### Priority 1: Safety net

Add tests before large refactors:

- auth guard test
- route rendering smoke test
- i18n loading test
- translation key parity check
- `getFormattedDate` tests
- one or two critical form tests
- one request portal smoke test

### Priority 2: Persian language support

- add Persian translations
- register `fa`
- verify RTL
- set document `lang`
- check settings and public portal selectors

### Priority 3: Date/i18n consistency

- wire `LocalizationProvider` to active `date-fns` locale
- centralize date formatting
- remove hardcoded English date displays where practical
- document Gregorian vs Jalali behavior

### Priority 4: API and state cleanup

- consolidate fetch/axios usage
- standardize API errors
- split `JWTAuthContext` responsibilities
- improve thunk typing and reduce shared `any`

### Priority 5: Type and dependency hygiene

- enable stricter TypeScript options gradually
- re-enable useful ESLint rules
- plan dependency upgrades separately from feature work
- remove unused/dead imports
- audit duplicate libraries

### Priority 6: Jalali calendar support

- add `calendarSystem` to frontend models
- select a Jalali conversion/picker approach
- update central date display
- update required date inputs
- decide whether FullCalendar must be replaced/customized for true Jalali grid
  behavior

## Practical recommendation for your next work

If your goal is Persian support, do this first:

1. Add Persian language support and RTL verification.
2. Add simple translation key parity checks.
3. Fix document `lang` handling.
4. Wire date-fns/MUI locale for Persian Gregorian labels.
5. Add a small test harness for date formatting.
6. Only then start Jalali-specific work.

This keeps low-risk translation work separate from high-risk calendar semantics.

## Final conclusion

The frontend architecture is acceptable but not excellent.

It has a solid foundation for Persian language support:

- central i18n setup
- lazy translation bundles
- supported language metadata
- company-language switching
- public portal language switching
- RTL styling support
- date-fns and FullCalendar locale loaders

Its biggest weaknesses are maintainability risks:

- no authored tests
- weak TypeScript strictness
- disabled lint guardrails
- duplicate HTTP clients
- large auth context
- render-time state update in the auth guard
- inconsistent date handling
- dependency drift

It is worth working on if you improve the engineering foundation as you add
features. For Persian language support, the frontend is ready enough. For
Jalali calendar support, add a central date design and tests before touching
many screens.
