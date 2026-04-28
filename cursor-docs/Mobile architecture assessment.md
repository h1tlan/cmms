# Mobile Architecture Assessment

This document answers three questions about the React Native mobile app in
`mobile/`:

1. Is the mobile app architecture fine?
2. Is it maintainable and aligned with common Expo/React Native best practices?
3. Is it worth working on this project?

The short answer is:

- The mobile app is worth working on if you want a real Expo/React Native client
  for an existing CMMS product.
- It has a recognizable architecture and a useful feature set.
- It also has technical debt around a large auth context, full-state
  persistence, deep-link drift, weak typing, missing tests, and missing RTL
  layout support.
- Persian language support is possible, but true RTL and Jalali calendar support
  need more careful mobile-specific work than the web frontend.

## Executive verdict

### Is the mobile architecture fine?

Mostly yes, with caveats.

The app follows a familiar Expo/React Native shape:

- Expo 53
- React Native 0.79
- React 19
- React Navigation 6
- Redux Toolkit
- redux-persist
- React Context for auth and company settings
- React Native Paper
- Formik and Yup
- i18next and react-i18next
- AsyncStorage for tokens/config
- Expo Notifications, Camera, FileSystem, DocumentPicker, ImagePicker, and NFC

The folder structure is understandable:

```text
mobile/App.tsx
mobile/navigation/
mobile/screens/
mobile/components/
mobile/contexts/
mobile/slices/
mobile/store/
mobile/models/
mobile/i18n/
mobile/utils/
```

This is a reasonable mobile architecture for a business app with many screens.
The caveat is that several central files have become too large and too
important, especially `AuthContext` and the root navigator.

### Is it maintainable?

It is maintainable for normal feature work, but cross-cutting work carries risk.

Small screens, translations, and entity-specific fixes should be manageable.
Larger changes touching authentication, company settings, permissions,
notifications, deep links, offline behavior, or dates should be done with tests
and incremental refactoring.

The biggest maintainability risks are:

- very large `mobile/contexts/AuthContext.tsx`
- very large `mobile/navigation/index.tsx`
- full Redux tree persisted to AsyncStorage
- `serializableCheck: false`
- no visible authored tests
- `strict: false` TypeScript
- no explicit RTL layout support

### Does it follow best practices?

It follows several common Expo/RN practices:

- Expo config with native plugins
- React Navigation stack/tab structure
- Redux Toolkit slices
- typed Redux hooks
- PersistGate for rehydration
- React Native Paper theme provider
- i18next translation resources
- Formik/Yup form patterns
- AsyncStorage for persisted app settings and tokens
- notification and deep-link handling

But it misses some mature-production practices:

- no automated tests in the repo
- no ESLint config visible under `mobile/`
- TypeScript strict mode disabled
- full-store persistence without whitelist/blacklist
- serializable state checks disabled globally
- deep-link handling is split between `App.tsx` and `LinkingConfiguration.ts`
- no `I18nManager` usage for RTL languages
- API errors are encoded into `Error.message`
- 403/session invalidation behavior is incomplete

### Is it worth working on?

Yes.

The app is not a toy. It already covers real mobile workflows:

- work orders
- requests
- assets
- locations
- parts
- meters
- people and teams
- vendors/customers
- notifications
- scanning/barcode/NFC flows
- file/image/document handling
- account switching
- custom server configuration

That makes it worth improving. Treat it as a mature mobile app with technical
debt, not as something that needs a rewrite before feature work.

## Mobile app shape

The app lives in:

```text
mobile/
```

Key files and folders:

```text
mobile/App.tsx
mobile/app.config.ts
mobile/package.json
mobile/navigation/index.tsx
mobile/navigation/LinkingConfiguration.ts
mobile/contexts/AuthContext.tsx
mobile/contexts/CompanySettingsContext.tsx
mobile/store/index.ts
mobile/store/rootReducer.ts
mobile/slices/
mobile/screens/
mobile/components/
mobile/models/
mobile/i18n/i18n.ts
mobile/utils/api.ts
mobile/utils/dates.ts
mobile/custom-theme.ts
```

## Strengths

### 1. Familiar Expo/React Native stack

The app is on a modern Expo SDK:

```text
expo ~53.0.27
react-native 0.79.6
react 19.0.0
```

That is a good foundation for active mobile development, provided dependencies
stay aligned with Expo SDK requirements.

### 2. Clear broad package structure

The codebase has recognizable areas:

- `screens/` for screen UI
- `components/` for shared UI
- `navigation/` for route definitions
- `slices/` for Redux state
- `models/` for domain/API types
- `contexts/` for auth and company settings
- `utils/` for API, dates, permissions, and helpers
- `i18n/` for translations

This makes onboarding possible without learning a custom architecture.

### 3. Provider stack is explicit

`mobile/App.tsx` wires the app in a predictable order:

```text
SafeAreaProvider
Provider store
PersistGate
AuthProvider
CompanySettingsProvider
PaperProvider
CustomSnackbarProvider
SheetProvider
RootLayout
Navigation
```

That is easy to reason about at a high level.

### 4. Redux Toolkit is used

The mobile app uses Redux Toolkit slices under:

```text
mobile/slices/
```

The store has typed hooks in:

```text
mobile/store/index.ts
```

This is a good baseline for predictable domain state.

### 5. Mobile-specific integrations exist

The app already integrates with useful native capabilities:

- Expo Notifications
- Expo Camera
- Expo DocumentPicker
- Expo FileSystem
- Expo ImagePicker
- Expo Updates
- React Native NFC Manager
- Firebase Analytics
- React Native WebView
- React Native Signature Canvas

That makes it a real mobile companion app rather than a thin shell.

### 6. Company-aware date formatting exists

`mobile/contexts/CompanySettingsContext.tsx` formats dates using company
preferences:

```ts
const tz = generalPreferences.timeZone;
const date = moment.tz(dateString, tz);
```

It respects:

- company timezone
- `MMDDYY` / `DDMMYY`
- optional time display

This is a good starting point for date consistency.

## Maintainability risks

### 1. `AuthContext` is too large

Relevant file:

```text
mobile/contexts/AuthContext.tsx
```

It handles many responsibilities:

- login/logout/register
- JWT verification
- user and company loading
- company settings
- user settings
- permissions
- feature checks
- subscription actions
- account switching
- WebSocket/STOMP notifications
- push notification registration
- analytics side effects
- language switching
- UI configuration

That makes it hard to test and risky to change.

Recommended direction:

- keep session/auth in `AuthContext`
- move WebSocket behavior into a dedicated hook/service
- move push notification setup into a dedicated hook/service
- move permission checks into a permission hook
- move subscription actions into a billing/service module
- keep language switching as a small side effect of loaded company preferences

### 2. Root navigator is too large

Relevant file:

```text
mobile/navigation/index.tsx
```

It imports and registers many screens in one file. This works, but it increases:

- merge conflicts
- review cost
- risk of copy/paste mistakes
- difficulty finding route ownership

One suspicious example found:

```ts
import SelectCustomersModal from '../screens/modals/SelectCustomersModal';
import SelectVendorsModal from '../screens/modals/SelectCustomersModal';
```

`SelectVendorsModal` appears to import from the customer modal path. There is a
real file at:

```text
mobile/screens/modals/SelectVendorsModal.tsx
```

That import should be verified and likely fixed.

### 3. Deep linking is split and can drift

`mobile/navigation/LinkingConfiguration.ts` contains placeholder-style paths
such as:

```text
TabOneScreen: 'one'
TabTwoScreen: 'two'
```

But `mobile/App.tsx` manually parses URLs for real paths such as:

```text
/app/work-orders
/app/requests
```

This means deep-link behavior is split between two places.

Recommended direction:

- make `LinkingConfiguration.ts` the single source of truth
- or move manual parsing into a named deep-link service
- add tests for notification/deep-link route mapping

### 4. Entire Redux state is persisted

Relevant file:

```text
mobile/store/index.ts
```

Current config:

```ts
const persistConfig = {
  key: 'root',
  storage: AsyncStorage
};
```

This persists the whole root reducer.

That can persist:

- stale API lists
- loading flags
- old filters
- large entity collections
- state that should be refetched after app restart

Recommended direction:

- add a whitelist for safe slices
- or add a blacklist for volatile slices
- use transforms if needed
- document which slices are safe to rehydrate

### 5. Serializable checks are disabled globally

`mobile/store/index.ts` has:

```ts
serializableCheck: false
```

This avoids warnings, but it also hides real Redux state mistakes.

Recommended direction:

- identify why it was disabled
- configure allowed non-serializable paths/actions only where needed
- keep checks enabled for the rest of the store

### 6. API error handling is fragile

Relevant file:

```text
mobile/utils/api.ts
```

Current behavior:

```ts
throw new Error(JSON.stringify(await response.json()));
```

Then callers parse `error.message`.

This can fail or become confusing when:

- response body is not JSON
- response body has a different shape
- network fails before a response exists
- session expires

There is also a TODO on 403:

```ts
if (response.status === 403) {
  //TODO
  // AsyncStorage.clear();
}
```

Recommended direction:

- introduce a typed `ApiError`
- handle JSON and non-JSON errors safely
- define 401/403 behavior clearly
- centralize logout/session-expiry handling

### 7. TypeScript strict mode is disabled

Relevant file:

```text
mobile/tsconfig.json
```

Current setting:

```json
"strict": false
```

This lowers friction but makes refactoring riskier in a large app.

Recommended direction:

- keep new files strongly typed
- gradually enable stricter checks
- start with critical utilities and models
- consider `strictNullChecks` as a first step

### 8. No authored tests were found

`mobile/package.json` has:

```json
"test": "jest"
```

and:

```json
"preset": "jest-expo"
```

But no `*.test.*` or `*.spec.*` files were found under `mobile/`.

That means there is no visible in-repo test safety net for:

- auth bootstrap
- navigation branches
- permissions
- API error handling
- Redux slices
- date formatting
- notification routing
- deep links
- form validation

This is one of the biggest risks for larger mobile changes.

## i18n and Persian readiness

### Current i18n model

Translations are bundled TypeScript modules under:

```text
mobile/i18n/translations/
```

Main setup:

```text
mobile/i18n/i18n.ts
```

The app uses:

- `i18next`
- `react-i18next`
- bundled translation resources
- default `lng: 'en'`
- fallback `en`

Language is changed from company preferences in `AuthContext`.

### Persian language readiness

Persian language support is feasible.

Minimum work:

1. Create `mobile/i18n/translations/fa.ts`.
2. Import it in `mobile/i18n/i18n.ts`.
3. Add `fa: { translation: faJSON }` to resources.
4. Ensure backend `FA` is lowercased to `fa` when applied.
5. Verify all mobile strings are translated.

### RTL readiness

The mobile app is not ready for strong RTL support yet.

No `I18nManager`, `forceRTL`, `allowRTL`, or `isRTL` usage was found in the
mobile TypeScript/TSX code.

For Persian and Arabic, React Native usually needs explicit RTL handling:

```ts
import { I18nManager } from 'react-native';

I18nManager.allowRTL(true);
I18nManager.forceRTL(true);
```

Important caveat: changing RTL direction often requires an app reload/restart to
fully apply layout mirroring.

Recommended direction:

- add a small RTL utility
- define RTL languages: `fa`, `ar`
- apply `I18nManager` near startup or language setup
- show a restart prompt when direction changes
- test both iOS and Android
- audit icons, row direction, tabs, modals, action sheets, and forms

## Date and Jalali calendar readiness

### Current date behavior

Central formatter:

```text
mobile/contexts/CompanySettingsContext.tsx
```

Uses:

```text
moment-timezone
```

and company settings:

- `timeZone`
- `dateFormat`

There are also date utilities in:

```text
mobile/utils/dates.ts
```

These use `Date.toDateString()` and string splitting:

```ts
const date = new Date(str).toDateString();
const arr = date.split(' ');
return `${arr[1]} ${arr[2]}`;
```

That is not ideal for localization or Jalali calendar support.

### Jalali readiness

The app is not ready for true Jalali calendar support without a date-layer
design.

Current limitations:

- native date picker is Gregorian
- `moment-timezone` formatting is Gregorian unless extended/replaced
- no Jalali picker dependency exists
- `utils/dates.ts` bypasses company locale/format
- no tests protect date behavior

Recommended approach:

1. Keep API payloads Gregorian/ISO.
2. Add backend/frontend/mobile `calendarSystem`.
3. Add a mobile date utility that formats through company preferences.
4. Replace `utils/dates.ts` string-splitting helpers.
5. Choose a Jalali-compatible picker if users must select Jalali dates.
6. Convert Jalali input back to Gregorian before API submission.
7. Add tests for timezone and off-by-one-day behavior.

## Theming and appearance

Theme file:

```text
mobile/custom-theme.ts
```

The app uses React Native Paper with a custom light MD3 theme.

Concern:

```text
mobile/navigation/index.tsx
```

uses React Navigation `DefaultTheme` directly:

```tsx
<NavigationContainer theme={DefaultTheme}>
```

This can make navigation chrome drift from the Paper theme or OS dark mode.

Recommended direction:

- derive a React Navigation theme from the Paper theme
- handle `useColorScheme()` consistently
- verify headers, tabs, sheets, and modals in dark/light mode

## Offline and persistence assessment

The app uses:

- AsyncStorage
- redux-persist
- NetInfo in some screens

This is useful, but it is not a full offline-first architecture.

Current risk:

- persisted Redux state may show old data
- mutations do not appear to queue/replay offline
- stale persisted lists may be confused with fresh server data

Recommended direction:

- document what works offline
- persist only safe local state
- refetch server lists after login/app resume
- add clear loading/stale indicators if offline mode is expanded

## Native integration assessment

The native integration surface is useful and relatively broad:

- notifications
- camera/barcode
- NFC
- document picker
- file system
- image picker
- web view
- signature canvas
- Firebase analytics
- EAS updates

Risks:

- native permissions need consistent translated messaging
- EAS/runtime versions must be updated deliberately
- `newArchEnabled: false` should be tracked for future migration
- `postinstall` runs `patch-package`, but no `patches/` directory was found in
  this workspace snapshot

If `patch-package` is intentional, commit the patch files. If it is not needed,
remove the postinstall hook to avoid CI confusion.

## Best-practice review

### What follows best practices

The mobile app does well in these areas:

- uses Expo and its config/plugin system
- uses React Navigation
- uses Redux Toolkit
- uses typed Redux hooks
- uses PersistGate intentionally
- uses React Native Paper
- uses i18next for translations
- has central company date formatting
- has a custom server/API URL flow
- integrates mobile-specific features through maintained packages

### What needs improvement

Main gaps:

- no visible test suite
- TypeScript strict mode disabled
- no mobile ESLint config found
- full Redux state persistence
- serializable checks disabled globally
- oversized `AuthContext`
- oversized root navigator
- duplicate/dead deep-link logic
- no RTL layout manager
- fragile API error handling
- inconsistent date utilities
- possible wrong vendor modal import

## Security and reliability assessment

Positive:

- JWT tokens are stored and attached centrally.
- API base URL can be configured for self-hosted deployments.
- Authenticated and unauthenticated navigation branches are explicit.
- Permission helper methods exist.

Risks:

- tokens in AsyncStorage are common but increase impact of device compromise or
  app-level leakage
- 403 handling is incomplete
- persisted full state can leave sensitive or stale data in storage
- no tests protect auth/session flows
- API errors are not structured

Recommended direction:

- define token/session expiry behavior
- clear relevant persisted state on logout/session failure
- reduce persisted state to safe slices
- add tests for login/logout/session-expiry behavior

## Should you work on this mobile app?

Yes, if you approach it as a mature companion app that needs disciplined
maintenance.

This is a good mobile project to work on if:

- you want to extend a real Expo/RN application
- you are comfortable with React Navigation and Redux Toolkit
- you can add tests around areas you touch
- you are willing to refactor large modules gradually
- you value product coverage over pristine architecture

Be careful if:

- you need strong offline-first guarantees immediately
- you expect RTL to work automatically
- you need high confidence without tests
- you want to make broad date/calendar changes quickly
- you want strict TypeScript guarantees from day one

The best mindset is:

> Treat the mobile app as a valuable existing client with technical debt, not as
> a broken app that needs a rewrite.

## Recommended improvement roadmap

### Priority 1: correctness and safety

1. Fix or verify the `SelectVendorsModal` import.
2. Add basic tests for `utils/api.ts`, `getErrorMessage`, and one slice thunk.
3. Add tests for company date formatting.
4. Define 401/403 session handling.

### Priority 2: navigation and persistence

1. Consolidate deep-link handling.
2. Add a persistence whitelist/blacklist.
3. Re-enable serializable checks with targeted exceptions.
4. Split the root navigator by feature area.

### Priority 3: i18n and RTL

1. Add Persian translation resources.
2. Add RTL direction management with `I18nManager`.
3. Add restart/reload behavior when direction changes.
4. Test Arabic and Persian layouts on iOS and Android.

### Priority 4: date/calendar layer

1. Replace date helpers that use `toDateString()` splitting.
2. Centralize date formatting through company settings.
3. Add `calendarSystem` support after backend/model support exists.
4. Choose and integrate a Jalali date picker if required.

### Priority 5: modularity and type safety

1. Split `AuthContext` responsibilities.
2. Move push/WebSocket logic into focused hooks/services.
3. Enable stricter TypeScript gradually.
4. Add or restore mobile ESLint configuration.

## Practical recommendation for Persian support

For Persian language support, do this first:

1. Add `fa.ts` translations.
2. Register `fa` in `mobile/i18n/i18n.ts`.
3. Apply company language `FA -> fa`.
4. Add RTL direction handling with restart prompt.
5. Test main screens manually in RTL.
6. Only then start Jalali date input/display work.

For Jalali support, do not patch individual screens first. Start with a central
mobile date utility and tests.

## Final conclusion

The mobile architecture is acceptable but not excellent.

It has a good product foundation:

- modern Expo/RN stack
- many real screens
- Redux Toolkit slices
- typed models
- company settings context
- i18n resources
- native integrations

Its biggest weaknesses are maintainability and confidence risks:

- no visible tests
- weak TypeScript strictness
- large auth context
- large navigator
- persistence is too broad
- deep links are split
- no RTL wiring
- fragile API errors
- date utilities are not localization-ready

It is worth working on, especially if the goal is to make the mobile app a
better companion to the web product. Persian language support is feasible, but
mobile RTL and Jalali calendar support require deliberate mobile-specific design.
