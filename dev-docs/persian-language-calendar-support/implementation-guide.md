# Persian Language and Calendar Support Implementation Guide

This guide explains how to add Persian language support and Persian/Jalali
calendar support across this repository.

The repository has four relevant surfaces:

- `api/`: Spring Boot API, company preferences, emails, reports, import templates.
- `frontend/`: main React web application.
- `mobile/`: React Native/Expo mobile application.
- `home/`: Next.js marketing/home application.

## Current state

### Shared product model

The product stores language and date preferences at the company level through
`GeneralPreferences`.

- API entity: `api/src/main/java/com/grash/model/GeneralPreferences.java`
- API language enum: `api/src/main/java/com/grash/model/enums/Language.java`
- API date format enum: `api/src/main/java/com/grash/model/enums/DateFormat.java`
- Frontend model: `frontend/src/models/owns/generalPreferences.ts`
- Mobile model: `mobile/models/generalPreferences.ts`
- Home model copy: `home/src/models/owns/generalPreferences.ts`

Current limits:

- There is no `FA`/`fa` language option.
- `DateFormat` only supports `MMDDYY` and `DDMMYY`.
- There is no separate calendar-system preference.
- Dates are stored and exchanged as normal Gregorian/ISO dates.

### API localization

The API uses the `Language` enum and Java `Locale` to localize server-side text.

Important files:

- `api/src/main/java/com/grash/model/enums/Language.java`
- `api/src/main/java/com/grash/utils/Helper.java`
- `api/src/main/resources/messages.properties`
- `api/src/main/resources/messages_*.properties`
- `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`
- `api/src/main/java/com/grash/dto/UserSignupRequest.java`
- `api/src/main/java/com/grash/dto/requestPortal/RequestPortalPublicDTO.java`

`Helper.getLocale(...)` maps each `Language` enum value to a Java `Locale`.
Emails, notifications, exports, import templates, and PDF reports use this
locale through `MessageSource`.

### Main web frontend localization

The main web app uses `i18next`, `react-i18next`, and bundled TypeScript
translation files.

Important files:

- `frontend/src/i18n/i18n.ts`
- `frontend/src/i18n/translations/en.ts`
- `frontend/src/i18n/translations/*.ts`
- `frontend/src/contexts/JWTAuthContext.tsx`
- `frontend/src/content/own/Settings/General/index.tsx`
- `frontend/src/content/own/Settings/Features/RequestPortal/PublicPage/RequestPortalPublicPage.tsx`
- `frontend/src/App.tsx`
- `frontend/src/theme/ThemeProvider.tsx`
- `frontend/src/theme/schemes/PureLightTheme.ts`

Language is detected from query string, local storage, and browser settings, but
logged-in users are switched to the company language from
`generalPreferences.language`.

The app already has RTL plumbing through `i18n.dir()`, `stylis-plugin-rtl`, and
`document.documentElement.dir`.

### Main web frontend dates and calendars

Important files:

- `frontend/src/contexts/CompanySettingsContext.tsx`
- `frontend/src/i18n/i18n.ts`
- `frontend/src/hooks/useDateLocale.tsx`
- `frontend/src/content/own/WorkOrders/Calendar/index.tsx`
- `frontend/src/content/own/WorkOrders/Calendar/Actions.tsx`
- `frontend/src/content/own/components/form/DateRangePicker.tsx`
- `frontend/src/content/own/Analytics/CustomDateRangePicker.tsx`

Current behavior:

- `CompanySettingsContext.getFormattedDate` uses `date-fns` and
  `date-fns-tz`.
- The format is selected from `MMDDYY` or `DDMMYY`.
- `getDateLocale` loads `date-fns` locales.
- `getCalendarLocale` loads FullCalendar locales.
- Some range pickers use hardcoded Gregorian English date formats.
- MUI date pickers use `AdapterDateFns`; no Jalali adapter is configured.

### Mobile localization

The mobile app uses `i18next`, `react-i18next`, and bundled TypeScript
translation files.

Important files:

- `mobile/i18n/i18n.ts`
- `mobile/i18n/translations/en.ts`
- `mobile/i18n/translations/*.ts`
- `mobile/contexts/AuthContext.tsx`
- `mobile/contexts/CompanySettingsContext.tsx`

Current behavior:

- Language is set from `companySettings.generalPreferences.language`.
- There is no device language detection.
- There is no `I18nManager` RTL setup.
- Arabic translations exist, but mobile RTL layout mirroring is not wired.

### Mobile dates and calendars

Current behavior:

- `mobile/contexts/CompanySettingsContext.tsx` formats dates with
  `moment-timezone`.
- The format is selected from `MMDDYY` or `DDMMYY`.
- `mobile/components/CustomDateTimePicker.tsx` uses
  `react-native-modal-datetime-picker`.
- There is no Jalali calendar picker.

### Home app localization

The home app uses `next-intl`.

Important files:

- `home/src/i18n/request.ts`
- `home/src/i18n/i18n.ts`
- `home/src/i18n/translations/en.ts`
- `home/src/i18n/translations/*.ts`
- `home/middleware.ts`
- `home/app/[locale]/layout.tsx`
- `home/src/components/LanguageSwitcher/index.tsx`
- `home/src/theme/ThemeProvider.tsx`

Current behavior:

- Locale routing supports 15 locales, but not Persian.
- `ThemeProvider.tsx` already treats `fa` as RTL.
- The app does not currently have meaningful date/calendar formatting.
- The layout uses Inter with Latin subsets, which is not ideal for Persian text.

## Recommended scope split

Implement Persian support in two coordinated tracks.

1. Persian language and RTL support:
   - Add `FA`/`fa` everywhere languages are registered.
   - Add translation resources.
   - Enable Persian in language selectors and public portal language handling.
   - Verify RTL behavior on web, home, and mobile.

2. Persian/Jalali calendar support:
   - Add a separate calendar-system preference instead of overloading
     `dateFormat`.
   - Keep persisted dates and API transport as Gregorian/ISO UTC dates.
   - Convert to/from Jalali only at presentation and date-input boundaries.
   - Update reports/exports only where user-facing formatted dates are expected.

Keeping language and calendar as separate preferences is important. Some Persian
users may still want Gregorian dates, and some non-Persian users may need Jalali
dates.

## Language code convention

Use these codes consistently:

- API enum: `FA`
- i18next and Next locale: `fa`
- Java locale: `new Locale("fa", "IR")`
- Message bundle: `messages_fa_IR.properties`
- Translation files:
  - `frontend/src/i18n/translations/fa.ts`
  - `mobile/i18n/translations/fa.ts`
  - `home/src/i18n/translations/fa.ts`

Avoid introducing both `fa` and `fa_ir` unless there is a real need to support
multiple Persian regional variants.

## Backend implementation steps

### 1. Add Persian to the API language enum

Update `api/src/main/java/com/grash/model/enums/Language.java`:

```java
public enum Language {
    EN,
    FR,
    ...
    BA,
    FA;
    // always add new languages at the end
}
```

The comment says to add new languages at the end. Follow that rule because enum
values may be persisted as small integers in the database.

### 2. Map `FA` to a Persian locale

Update `api/src/main/java/com/grash/utils/Helper.java`:

```java
case FA:
    return new Locale("fa", "IR");
```

This allows `MessageSource`, Thymeleaf templates, reports, and exports to select
Persian messages.

### 3. Add server-side message bundle

Create:

```text
api/src/main/resources/messages_fa_IR.properties
```

Use `messages.properties` as the source of required keys and translate every
key. Keep placeholders such as `{0}`, `{1}`, and HTML-sensitive values exactly
compatible with the English bundle.

Validation checklist:

- Every key in `messages.properties` exists in `messages_fa_IR.properties`.
- Placeholder counts match the source language.
- Files are saved as UTF-8.
- Emails render correctly in RTL.

### 4. Add import templates if required

`ImportController` selects CSV import templates by language. Locate the existing
template files and add Persian variants for each import entity if localized
templates are part of the supported language contract.

Expected naming should follow the existing template naming pattern. If there is
no Persian file, the controller may need a fallback to English.

### 5. Decide whether signup can choose Persian

Signup sends `UserSignupRequest.language`; the web registration flow currently
uses the active i18n language uppercased. After adding `fa` to frontend i18n,
registration can send `FA` automatically.

Confirm:

- API accepts `FA`.
- New companies can be seeded with Persian language.
- Existing companies can patch `generalPreferences.language` to `FA`.

## Backend calendar-system design

### Recommended API model

Do not add Jalali to `DateFormat`. `DateFormat` describes pattern order, not
calendar system. Add a new enum and field:

```java
public enum CalendarSystem {
    GREGORIAN,
    JALALI
}
```

Add it to:

- `api/src/main/java/com/grash/model/GeneralPreferences.java`
- `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`
- frontend/mobile/home TypeScript `GeneralPreferences` models
- database migrations

Suggested default:

```java
private CalendarSystem calendarSystem = CalendarSystem.GREGORIAN;
```

When language is set to `FA`, the UI may offer to switch calendar system to
`JALALI`, but it should not be forced silently unless product requirements say
so.

### Database migration

Add a Liquibase migration for the new preference column. Follow the existing
enum storage style in the repository.

Requirements:

- Existing companies default to `GREGORIAN`.
- New companies default to `GREGORIAN`.
- The column is included in any generated schema expectations.

### Backend date storage rule

Keep all stored dates as Gregorian instants/dates. Jalali is a presentation and
input-calendar concern.

Recommended invariant:

- API input/output dates remain ISO/Gregorian.
- Date-only range boundaries are converted to Gregorian before API submission.
- Backend analytics, recurrence, PM scheduling, and due-date comparisons keep
  using Java date/time types.
- User-facing export/report rendering can format to Jalali when
  `calendarSystem == JALALI`.

This avoids breaking scheduling logic and integrations that already expect ISO
dates.

### Server-side reports and exports

Review these areas for user-facing date strings:

- `api/src/main/java/com/grash/controller/WorkOrderController.java`
  PDF generation passes `dateFormat`, `timeZone`, `messageSource`, and `locale`
  to `work-order-report.html`.
- `api/src/main/java/com/grash/service/AsyncExportService.java`
  passes locale and CSV separator into exports.
- `api/src/main/java/com/grash/utils/CsvFileGenerator.java`
  currently writes some date objects directly.

If Jalali dates are required in PDFs/CSVs, create a dedicated date formatting
helper that accepts:

- instant/date value
- company timezone
- date format preference
- calendar system
- locale

Then use that helper for exported user-facing cells. Do not change JSON API date
serialization globally unless a separate API versioning plan exists.

## Main frontend language implementation

### 1. Create Persian translation file

Create:

```text
frontend/src/i18n/translations/fa.ts
```

Use `frontend/src/i18n/translations/en.ts` as the canonical key list.

Requirements:

- Export the same object shape as other translation files.
- Preserve interpolation placeholders and HTML-sensitive strings.
- Translate DataGrid strings, settings labels, request portal strings, work
  order terms, PM terms, and validation messages.

### 2. Register Persian in i18n

Update `frontend/src/i18n/i18n.ts`:

- Import the Iran flag from `country-flag-icons/react/3x2`:

```ts
import { IR } from 'country-flag-icons/react/3x2';
```

- Add translation loader:

```ts
fa: () => import('./translations/fa')
```

- Add `date-fns` locale loader:

```ts
fa: () => import('date-fns/locale').then((m) => m.faIR)
```

- Add FullCalendar locale loader if available in the installed package:

```ts
fa: () => import('@fullcalendar/core/locales/fa').then((m) => m.default)
```

- Add `FA` to `SupportedLanguage`.
- Add `{ code: 'fa', label: 'Persian', Icon: IR }` to
  `supportedLanguages`.

`i18n.dir()` should return `rtl` for `fa`, so the existing web RTL plumbing
should activate after the language is changed.

### 3. Update language selectors and settings

Most selectors read from `supportedLanguages`, so they should pick up Persian
automatically after `i18n.ts` is updated.

Verify:

- General settings language dropdown.
- Public request portal language dropdown.
- Registration language submission.
- Initial authenticated language selection from company preferences.

### 4. Update document language

`frontend/public/index.html` has a static `lang="en"`. Consider adding a small
effect near the existing direction handling to set:

```ts
document.documentElement.lang = i18n.language;
```

This improves accessibility and browser text handling for Persian.

## Main frontend Jalali calendar implementation

### Recommended approach

Create a small date formatting and conversion layer instead of scattering
Jalali-specific logic through components.

Suggested files:

```text
frontend/src/utils/calendarSystem.ts
frontend/src/hooks/useCompanyDateFormatter.ts
```

Responsibilities:

- Read `generalPreferences.calendarSystem`.
- Convert Gregorian dates to Jalali strings when needed.
- Convert Jalali date-picker selections back to Gregorian dates before API
  calls.
- Keep timezone handling centralized.
- Preserve existing `MMDDYY`/`DDMMYY` behavior for Gregorian dates.

### Dependency decision

The current web stack has `date-fns`, `date-fns-tz`, FullCalendar, MUI lab date
pickers, and some `dayjs` usage. It does not include a Jalali conversion or
Jalali picker dependency.

Evaluate one web strategy before implementation:

1. Lightweight conversion library plus custom formatting:
   - Use a Jalali conversion package for display/input conversion.
   - Keep existing date pickers where Gregorian input is acceptable.

2. Jalali-aware date picker/adapter:
   - Use a Jalali-compatible picker for user-facing Persian date selection.
   - Keep API values as Gregorian.

3. Day.js Jalali plugin path:
   - Useful if date pickers and formatting can standardize on Day.js.
   - This is more invasive because the app currently uses `date-fns` broadly.

Prefer the smallest dependency surface that supports required picker behavior.
When implementing, add dependencies through the package manager so the latest
compatible version is selected.

### Places to update first

Start with central formatters:

- `frontend/src/contexts/CompanySettingsContext.tsx`
- `frontend/src/hooks/useDateLocale.tsx`
- `frontend/src/i18n/i18n.ts`

Then update date inputs and hardcoded formats:

- `frontend/src/content/own/components/form/DateRangePicker.tsx`
- `frontend/src/content/own/Analytics/CustomDateRangePicker.tsx`
- MUI date picker usages under `frontend/src/content/own/**`
- Work order calendar actions and headings

Finally update FullCalendar:

- Set locale to Persian when language is `fa`.
- Decide whether FullCalendar remains Gregorian with Persian labels or must
  display a true Jalali month grid. FullCalendar locale alone translates labels;
  it does not make the calendar Jalali.

If a true Jalali month grid is required, treat that as a larger calendar-widget
replacement or customization task.

## Mobile language implementation

### 1. Create Persian translation file

Create:

```text
mobile/i18n/translations/fa.ts
```

Use `mobile/i18n/translations/en.ts` as the canonical key list.

### 2. Register Persian

Update `mobile/i18n/i18n.ts`:

```ts
import faJSON from './translations/fa';

const resources = {
  ...
  fa: { translation: faJSON },
};
```

### 3. Add mobile RTL support

React Native RTL requires explicit setup. Add `I18nManager` handling near app
startup or language setup.

Important considerations:

- `I18nManager.forceRTL(true)` generally requires an app reload to fully mirror
  layout.
- Switching between LTR and RTL at runtime should show a restart/reload prompt
  if needed.
- Test both iOS and Android.

Likely files:

- `mobile/App.tsx`
- `mobile/contexts/AuthContext.tsx`
- navigation/theme files

Implementation sketch:

```ts
import { I18nManager } from 'react-native';

const isRtlLanguage = lng === 'fa' || lng === 'ar';
if (I18nManager.isRTL !== isRtlLanguage) {
  I18nManager.allowRTL(isRtlLanguage);
  I18nManager.forceRTL(isRtlLanguage);
  // Trigger an app reload or ask the user to restart.
}
```

### 4. Refresh language after preference updates

`patchGeneralPreferences` updates state but does not currently call
`changeLanguage`. If mobile gets a language selector, update i18n immediately
after a successful preferences patch.

## Mobile Jalali calendar implementation

The existing native date/time picker is Gregorian. For true Jalali date input,
select a React Native compatible Persian/Jalali date picker or implement a
custom picker.

Recommended behavior:

- Display Jalali dates when `calendarSystem == JALALI`.
- Store selected values as Gregorian `Date`/ISO strings before API submission.
- Keep `moment-timezone` timezone conversion until the date layer is refactored.
- Avoid calling `moment.locale('fa')` as the only solution; locale changes text,
  but does not guarantee Jalali calendar behavior.

Update:

- `mobile/contexts/CompanySettingsContext.tsx`
- `mobile/components/CustomDateTimePicker.tsx`
- any screens that display date ranges or due dates

## Home app language implementation

### 1. Create Persian messages

Create:

```text
home/src/i18n/translations/fa.ts
```

Use `home/src/i18n/translations/en.ts` as the canonical key list.

### 2. Register Persian locale

Update `home/src/i18n/request.ts`:

```ts
export const locales = [
  "en",
  ...
  "fa"
];
```

Update `home/middleware.ts` matcher:

```ts
"/(en|es|fr|de|tr|pt-br|pl|ar|it|sv|ru|hu|nl|zh-cn|ba|fa)/:path*"
```

### 3. Update supported languages

Update `home/src/i18n/i18n.ts`:

- Add `FA` to the `SupportedLanguage` type.
- Import `IR` from `country-flag-icons/react/3x2`.
- Add `{ code: "fa", label: "Persian", Icon: IR }`.

Note: this file currently uses `pt_br`, while Next routing uses `pt-br`. Verify
the language switcher behavior when adding `fa` and consider normalizing the
existing underscore/hyphen mismatch separately if it causes bugs.

### 4. Fonts and HTML direction

`home/app/[locale]/layout.tsx` sets `<html lang={locale}>`, which will work for
`fa` after the locale is registered.

`home/src/theme/ThemeProvider.tsx` already includes `fa` in its RTL locale list.

Review fonts because the current Inter Latin subset is not ideal for Persian.
Options:

- Use a font with Arabic/Persian glyph support.
- Add a Persian-specific font stack for `fa`.
- Test headings, body text, buttons, and dense tables.

## Translation workflow

For every new Persian translation file:

1. Copy the English key structure.
2. Translate values only.
3. Preserve keys exactly.
4. Preserve interpolation tokens such as `{{name}}`, `{0}`, and `{1}`.
5. Preserve markdown/HTML placeholders and URLs.
6. Keep punctuation and spacing appropriate for RTL text.
7. Run a key parity check against English.

Recommended scripts/checks to add:

- Compare `frontend/src/i18n/translations/en.ts` keys against `fa.ts`.
- Compare `mobile/i18n/translations/en.ts` keys against `fa.ts`.
- Compare `home/src/i18n/translations/en.ts` keys against `fa.ts`.
- Compare `api/src/main/resources/messages.properties` keys against
  `messages_fa_IR.properties`.

## Calendar UX decisions to make before coding

Answer these before implementing Jalali support:

1. Is Persian language support enough for the first release, or is true Jalali
   date input/display required immediately?
2. Should `calendarSystem` be independent from language?
3. Should new Persian companies default to Jalali or Gregorian?
4. Should reports and CSV exports display Jalali dates?
5. Should FullCalendar show a true Jalali month grid or only Persian labels and
   RTL layout?
6. Should mobile support true Jalali date picking in the first release?
7. Should numbers use Latin digits or Persian digits?

Recommended defaults:

- Keep `calendarSystem` independent.
- Default existing companies to Gregorian.
- Default new companies to Gregorian unless explicitly selected.
- Use Persian labels and RTL first, then add true Jalali widgets where required.
- Keep API dates ISO/Gregorian.

## Testing checklist

### API

- `Language.fromString("FA")` returns `Language.FA`.
- Patching general preferences to `FA` succeeds.
- Signup with `language: "FA"` succeeds.
- `Helper.getLocale` returns `fa_IR` for Persian companies.
- Persian emails render with translated subject/body.
- Work order PDF renders Persian text and direction correctly.
- CSV exports are UTF-8 and keep Persian text intact.
- Import templates fall back or resolve for Persian.

### Main frontend

- App loads with `?lang=fa`.
- `localStorage.lang = "fa"` is honored.
- Company language `FA` switches the authenticated app to Persian.
- Settings language dropdown includes Persian.
- Public request portal includes Persian.
- `<html dir="rtl">` is set for Persian.
- `<html lang="fa">` is set if the document language effect is added.
- Sidebar, tables, dialogs, forms, and menus are usable in RTL.
- Date display still works for Gregorian mode.
- Jalali mode displays and submits correct dates if implemented.
- Work order calendar uses Persian locale.

### Mobile

- Company language `FA` switches translations to Persian.
- RTL layout is applied after restart/reload.
- Navigation, forms, action sheets, and tables are usable in RTL.
- Date display works in Gregorian mode.
- Jalali display/input works if implemented.
- Push notification permission strings and non-component `i18n.t(...)` calls are
  translated.

### Home

- `/fa` route renders.
- Language switcher includes Persian.
- `<html lang="fa">` is rendered.
- `<html dir="rtl">` is applied by theme provider.
- Persian font rendering is acceptable.
- Marketing pages, pricing, FAQs, and forms are translated.

### Regression

- Existing languages still load.
- Arabic RTL still works.
- `pt_br`/`pt-br` and `zh_cn`/`zh-cn` behavior is not broken.
- Existing companies without `calendarSystem` get Gregorian defaults.
- API clients receiving dates are not broken by Jalali presentation logic.

## Suggested implementation order

1. Add backend `FA` language support and `messages_fa_IR.properties`.
2. Add `fa` translation files and language registration in `frontend`,
   `mobile`, and `home`.
3. Verify Persian language and RTL without Jalali calendar changes.
4. Add `calendarSystem` to API and TypeScript models.
5. Add central date formatting/conversion helpers.
6. Update web date displays and date inputs.
7. Update mobile date displays and date inputs.
8. Update reports/exports if Jalali output is required.
9. Add parity tests and manual RTL/Jalali test coverage.

## Key risks

- Java enum persistence: add enum values at the end and verify Liquibase enum
  migrations before deploying.
- RTL layout regressions, especially mobile where RTL is currently absent.
- Assuming locale equals calendar system.
- FullCalendar Persian locale translating labels but not creating a Jalali month
  grid.
- Hardcoded date formats outside central helpers.
- Font support for Persian glyphs.
- Placeholder mismatches in translated strings.
