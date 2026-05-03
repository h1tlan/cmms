# Frontend Persian Language and Calendar Support Guide

This document focuses only on the main React web application in `frontend/`.
It explains the current localization and date behavior, then describes how to
add Persian language support and optional Persian/Jalali calendar support.

It does not cover:

- `mobile/`
- `home/`
- backend implementation details beyond the frontend contract it consumes

## Short answer: is it okay?

Yes, adding Persian language support to the main frontend is reasonable. The
application already has most of the required infrastructure:

- `i18next`
- `react-i18next`
- language detection
- lazy translation bundle loading
- a central supported-language list
- existing RTL support for Arabic through `i18n.dir()`
- date-fns locale loading
- FullCalendar locale loading

Persian language and RTL support should be a normal incremental change.

Persian/Jalali calendar support is a larger change. The current frontend
formats Gregorian dates and uses Gregorian date pickers/calendars. True Jalali
support requires a date conversion/picker strategy and a new product preference
such as `calendarSystem`.

## Current frontend architecture

### Application location

The main app lives in:

```text
frontend/
```

It is a React application using Material UI, i18next, date-fns, FullCalendar,
and Redux-style slices.

Relevant dependencies in `frontend/package.json` include:

- `i18next`
- `react-i18next`
- `i18next-browser-languagedetector`
- `date-fns`
- `date-fns-tz`
- `dayjs`
- `@fullcalendar/*`
- `@mui/lab`
- `country-flag-icons`
- `stylis`
- `stylis-plugin-rtl`

### Translation files

Translations are TypeScript object files:

```text
frontend/src/i18n/translations/
```

Current supported translation files include:

- `en.ts`
- `fr.ts`
- `es.ts`
- `de.ts`
- `tr.ts`
- `pt_BR.ts`
- `pl.ts`
- `ar.ts`
- `it.ts`
- `sv.ts`
- `ru.ts`
- `hu.ts`
- `nl.ts`
- `zh_cn.ts`
- `ba.ts`

There is no `fa.ts` today.

### i18n setup

Primary file:

```text
frontend/src/i18n/i18n.ts
```

This file defines:

- `translationLoaders`
- `dateLocaleLoaders`
- `calendarLocaleLoaders`
- i18next initialization
- language detection
- `loadLanguage`
- `getDateLocale`
- `getCalendarLocale`
- `SupportedLanguage`
- `supportedLanguages`

The current i18next setup:

- starts with empty resources
- lazy-loads translation files
- supports only keys from `translationLoaders`
- falls back to English
- detects language from:
  - query string `?lang=`
  - local storage key `lang`
  - browser navigator language

Detected languages are normalized by `convertDetectedLanguage`.

### Authenticated language behavior

When users log in, the app switches to the company language preference.

Relevant file:

```text
frontend/src/contexts/JWTAuthContext.tsx
```

Important behavior:

```ts
const switchLanguage = async ({ lng }: { lng: any }) => {
  await loadLanguage(lng);
  internationalization.changeLanguage(lng);
};
```

The company setting is applied with:

```ts
companySettings.generalPreferences.language.toLowerCase()
```

So backend `FA` should become frontend `fa`.

### General settings language dropdown

Relevant file:

```text
frontend/src/content/own/Settings/General/index.tsx
```

The settings screen reads language options from `supportedLanguages`. Once
Persian is added there, the language dropdown should include Persian.

### Public request portal language behavior

Relevant file:

```text
frontend/src/content/own/Settings/Features/RequestPortal/PublicPage/RequestPortalPublicPage.tsx
```

The public portal reads `portal.companyLanguage`, lowercases it, and switches
i18n. It also renders a language selector from `supportedLanguages`.

### Registration language behavior

Relevant file:

```text
frontend/src/content/pages/Auth/Register/RegisterJWT.tsx
```

Registration uses the active i18n language and sends an uppercased value to the
API. After `fa` is registered, registration can send `FA`.

### RTL support

Relevant files:

```text
frontend/src/theme/ThemeProvider.tsx
frontend/src/theme/schemes/PureLightTheme.ts
```

`ThemeProvider.tsx` uses:

- `i18n.dir()`
- `stylis-plugin-rtl`
- Emotion RTL cache
- `document.documentElement.setAttribute('dir', 'rtl')`

`PureLightTheme.ts` sets theme direction from `i18n.dir()`.

This means Persian should activate RTL automatically if i18next recognizes
`fa` as an RTL language.

## Current frontend date behavior

### Central display formatter

Relevant file:

```text
frontend/src/contexts/CompanySettingsContext.tsx
```

The central display helper is:

```ts
const getFormattedDate = (dateString: string, hideTime?: boolean) => {
  if (!dateString) return '';

  const tz = generalPreferences.timeZone;
  const date = utcToZonedTime(new Date(dateString), tz);

  const timeStr = hideTime ? '' : format(date, ' HH:mm');

  if (generalPreferences.dateFormat === 'MMDDYY') {
    return format(date, 'MM/dd/yy') + timeStr;
  } else {
    return format(date, 'dd/MM/yy') + timeStr;
  }
};
```

This currently:

- converts dates to the company timezone
- formats dates with `date-fns`
- supports only `MMDDYY` and `DDMMYY`
- does not know about Jalali/Persian calendars

### date-fns locales

`frontend/src/i18n/i18n.ts` defines `dateLocaleLoaders`, used through:

```text
frontend/src/hooks/useDateLocale.tsx
```

This supports localized Gregorian month/day names where components use the hook.

### FullCalendar locales

`frontend/src/i18n/i18n.ts` defines `calendarLocaleLoaders`.

The work order calendar uses:

```text
frontend/src/content/own/WorkOrders/Calendar/index.tsx
```

FullCalendar locale files translate UI labels, but they do not make the
calendar grid Jalali.

### Hardcoded date formats

Some components bypass the central formatter and use fixed English/Gregorian
formats.

Known examples:

```text
frontend/src/content/own/components/form/DateRangePicker.tsx
frontend/src/content/own/Analytics/CustomDateRangePicker.tsx
```

These should be reviewed before claiming full Persian/Jalali date support.

### MUI date pickers

`frontend/src/App.tsx` uses `LocalizationProvider` with `AdapterDateFns`.

The current setup does not configure a Jalali adapter.

## Frontend Persian language implementation

### 1. Create Persian translation file

Create:

```text
frontend/src/i18n/translations/fa.ts
```

Use:

```text
frontend/src/i18n/translations/en.ts
```

as the source of truth for keys.

Requirements:

- Export the same object shape as `en.ts`.
- Preserve all keys exactly.
- Preserve interpolation tokens.
- Preserve URLs and HTML/markdown fragments.
- Keep all strings UTF-8.
- Translate application terms consistently.

Important translation domains include:

- authentication
- navigation
- settings
- work orders
- preventive maintenance
- assets
- parts
- locations
- meters
- request portal
- analytics
- data grid labels
- validation and error messages

### 2. Register Persian in `i18n.ts`

Update:

```text
frontend/src/i18n/i18n.ts
```

Import Iran flag:

```ts
import { IR } from 'country-flag-icons/react/3x2';
```

Add translation loader:

```ts
fa: () => import('./translations/fa')
```

Add date-fns locale loader:

```ts
fa: () => import('date-fns/locale').then((m) => m.faIR)
```

Add FullCalendar locale loader if available in the installed package:

```ts
fa: () => import('@fullcalendar/core/locales/fa').then((m) => m.default)
```

Add `FA` to `SupportedLanguage`:

```ts
export type SupportedLanguage =
  | 'DE'
  | 'EN'
  ...
  | 'BA'
  | 'FA';
```

Add Persian to `supportedLanguages`:

```ts
{ code: 'fa', label: 'Persian', Icon: IR }
```

Consider using the Persian label:

```ts
{ code: 'fa', label: 'فارسی', Icon: IR }
```

Either is acceptable; the Persian label is usually better in a language
selector because users can recognize their own language.

### 3. Verify language detection

The existing `convertDetectedLanguage` should handle:

- `fa`
- `fa-IR`
- `fa_IR`

because it lowercases and returns the first language segment unless a language
has a special mapping.

Expected results:

```text
?lang=fa     -> fa
fa-IR        -> fa
fa_IR        -> fa
```

No special Persian mapping should be needed.

### 4. Verify settings and public portal

After updating `supportedLanguages`, verify:

- General settings language dropdown includes Persian.
- Selecting Persian calls `loadLanguage('fa')`.
- The company preference is patched as `FA`.
- Reloading as an authenticated user loads Persian from company settings.
- Public request portal uses Persian when `companyLanguage` is `FA`.
- Public request portal language dropdown includes Persian.

### 5. Update document language

`frontend/public/index.html` currently has a static HTML language value.

Add a runtime update near direction handling:

```ts
document.documentElement.lang = i18n.language;
```

Good location:

```text
frontend/src/theme/ThemeProvider.tsx
```

This improves accessibility, browser spellchecking, text segmentation, and
assistive technology behavior.

### 6. Test RTL layout

Persian should set:

```html
<html dir="rtl">
```

Test:

- login and registration
- dashboard
- sidebar
- settings
- work order list
- work order details
- create/edit dialogs
- data grids
- date range pickers
- public request portal
- notification menus
- profile and account menus

Pay special attention to:

- icons that imply direction
- left/right paddings
- menu anchors
- drag/drop behavior
- table column alignment
- charts and legends
- form helper text

## Frontend Jalali calendar implementation

### Important distinction

Persian language support and Jalali calendar support are different features.

Adding `fa` translations and RTL does not make dates Jalali.

For true Jalali support, the frontend needs:

- a backend preference such as `calendarSystem`
- TypeScript model updates
- date conversion helpers
- display formatting updates
- date picker updates
- regression tests

### Recommended frontend model contract

After backend adds `calendarSystem`, update:

```text
frontend/src/models/owns/generalPreferences.ts
```

Example:

```ts
export type CalendarSystem = 'GREGORIAN' | 'JALALI';

export interface GeneralPreferences {
  language: SupportedLanguage;
  dateFormat: 'MMDDYY' | 'DDMMYY';
  calendarSystem: CalendarSystem;
  ...
}
```

Preserve `dateFormat` for order/pattern. Do not overload it to mean calendar
system.

### Centralize calendar logic

Create a small frontend calendar/date abstraction.

Suggested files:

```text
frontend/src/utils/calendarSystem.ts
frontend/src/hooks/useCompanyDateFormatter.ts
```

Responsibilities:

- read company timezone
- read `dateFormat`
- read `calendarSystem`
- format Gregorian dates as today
- format Jalali dates when selected
- convert Jalali picker values to Gregorian API values
- keep API transport values ISO/Gregorian

Avoid placing Jalali conversion logic directly in screens.

### Dependency options

The project currently uses `date-fns`, `date-fns-tz`, `dayjs`, and MUI
date-fns adapters. There is no Jalali dependency.

Evaluate these options:

1. Conversion-only library
   - Smallest footprint.
   - Good for display formatting.
   - Does not solve Jalali date picker UX.

2. Jalali-aware picker component
   - Best if users must select Jalali dates.
   - Requires UI integration and form compatibility work.

3. Day.js Jalali plugin strategy
   - Can work if the app standardizes more date logic around Day.js.
   - More invasive because current central formatting uses date-fns.

Use the package manager to add any new dependency so the latest compatible
version is selected.

### Update central display formatter

Change `getFormattedDate` to delegate to a central helper.

Today:

```ts
return format(date, 'dd/MM/yy') + timeStr;
```

Target:

```ts
return formatCompanyDate(dateString, {
  timeZone: generalPreferences.timeZone,
  dateFormat: generalPreferences.dateFormat,
  calendarSystem: generalPreferences.calendarSystem,
  hideTime
});
```

The helper should keep current behavior for Gregorian mode.

### Update date input components

Review and update:

```text
frontend/src/content/own/components/form/DateRangePicker.tsx
frontend/src/content/own/Analytics/CustomDateRangePicker.tsx
frontend/src/content/own/**/*
```

Search for:

```text
format(
toLocaleDateString
DateRangePicker
LocalizationProvider
AdapterDateFns
```

For every date input:

- decide whether Jalali input is required
- convert Jalali selection to Gregorian before API submission
- keep timezone behavior consistent
- preserve form validation

### FullCalendar behavior

Adding the FullCalendar Persian locale should translate labels.

It does not guarantee:

- Jalali month names in all locations
- Jalali date math
- a true Jalali month grid

If product requirements require a true Jalali calendar grid, treat that as a
larger calendar-widget customization or replacement task.

### Numbers and digits

Decide whether Persian UI should use:

- Latin digits: `2026/04/28`
- Persian digits: `۱۴۰۵/۰۲/۰۸`

This affects:

- date formatting
- numeric inputs
- exports
- reports
- charts
- data grids

Do not convert digits inside machine-readable fields, hidden inputs, API
payloads, IDs, or URLs.

## Testing checklist

### Language loading

- `?lang=fa` loads Persian.
- `localStorage.lang = "fa"` loads Persian.
- Browser locale `fa-IR` maps to `fa`.
- English fallback still works for missing keys.
- Existing languages still lazy-load.

### Authenticated app

- Company preference `FA` switches the app to Persian.
- Settings dropdown displays Persian.
- Updating language to Persian persists.
- Reload keeps Persian selected.
- Registration can submit `FA`.

### RTL

- `<html dir="rtl">` is set.
- `<html lang="fa">` is set if implemented.
- Sidebar alignment is correct.
- Dialogs and menus open in expected positions.
- Forms and labels align correctly.
- Data grid controls are usable.
- Arabic RTL still works.

### Dates

- Gregorian mode output remains unchanged.
- Company timezone is still respected.
- `MMDDYY` and `DDMMYY` still work.
- Persian `date-fns` locale works where `useDateLocale` is used.
- FullCalendar labels are Persian if locale is registered.
- Jalali mode works only if the calendar-system feature is implemented.

### Date inputs

- Date range pickers display expected values.
- Date selections submit Gregorian API values.
- Validation still works.
- Editing existing dates does not shift the day because of timezone conversion.

### Public request portal

- Persian loads for a portal whose company language is `FA`.
- Language selector includes Persian.
- RTL layout is usable for anonymous users.

## Suggested frontend implementation order

1. Add `frontend/src/i18n/translations/fa.ts`.
2. Register `fa` in `frontend/src/i18n/i18n.ts`.
3. Verify language selector and authenticated language switching.
4. Verify RTL with Persian.
5. Set `document.documentElement.lang` dynamically.
6. Add frontend `calendarSystem` model support after backend exposes it.
7. Add central date formatting/conversion helper.
8. Update hardcoded date displays and date inputs.
9. Add or update tests/checks for translation key parity and date formatting.

## Risks

- Missing translation keys causing English fallback in Persian UI.
- Placeholder mismatches in translated strings.
- Assuming Persian language automatically means Jalali calendar.
- FullCalendar locale being mistaken for a true Jalali calendar.
- Hardcoded date formats bypassing central helpers.
- Timezone conversions causing off-by-one-day bugs.
- RTL layout regressions in menus, tables, and dialogs.
- Font rendering issues for Persian text.

## Acceptance criteria

Persian language support is complete when:

- `fa.ts` exists and matches English keys.
- `fa` is in `translationLoaders`.
- `fa` is in `supportedLanguages`.
- `FA` is in `SupportedLanguage`.
- `?lang=fa` works.
- company language `FA` works.
- settings and public portal selectors include Persian.
- RTL direction is applied.
- major authenticated workflows are usable in Persian.

Jalali support is complete only when:

- frontend models include `calendarSystem`
- display helpers support Jalali
- required date inputs support Jalali selection
- API submissions remain Gregorian/ISO
- timezone behavior is tested
- FullCalendar behavior matches product requirements
