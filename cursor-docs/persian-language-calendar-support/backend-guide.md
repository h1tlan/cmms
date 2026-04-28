# Backend Persian Language and Calendar Support Guide

This document covers only the backend API in `api/`. It explains the current
localization/date model, the files that need to change for Persian language
support, and the safest way to add Persian/Jalali calendar support without
breaking stored dates, scheduling, analytics, exports, or API clients.

## Scope

Backend scope includes:

- Company language and date preferences.
- API DTOs that expose or patch preferences.
- Java locale mapping.
- Server-side translation bundles.
- Email, notification, PDF, CSV, and import-template localization.
- Persistence changes for a calendar-system preference.
- Backend formatting helpers for user-facing date output.

Backend scope does not include:

- React UI language switchers.
- Browser date pickers.
- Mobile layout or mobile date pickers.
- Marketing/home site locale routing.

## Current backend state

The backend is a Spring Boot API under `api/`.

Important files:

- `api/src/main/java/com/grash/model/GeneralPreferences.java`
- `api/src/main/java/com/grash/model/enums/Language.java`
- `api/src/main/java/com/grash/model/enums/DateFormat.java`
- `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`
- `api/src/main/java/com/grash/dto/UserSignupRequest.java`
- `api/src/main/java/com/grash/dto/requestPortal/RequestPortalPublicDTO.java`
- `api/src/main/java/com/grash/utils/Helper.java`
- `api/src/main/resources/messages.properties`
- `api/src/main/resources/messages_*.properties`
- `api/src/main/resources/templates/*.html`
- `api/src/main/resources/db/changelog/**`

`GeneralPreferences` currently stores:

- `language`: a `Language` enum, defaulting to `Language.EN`.
- `dateFormat`: a `DateFormat` enum, currently `MMDDYY` or `DDMMYY`.
- `timeZone`: a Java timezone identifier.
- `csvSeparator` and other company-wide settings.

Current limitations:

- `Language` does not include `FA`.
- `DateFormat` describes only day/month order and does not represent a calendar
  system.
- There is no `CalendarSystem` preference.
- Dates are stored and exchanged as regular Gregorian/ISO dates.

## Is the current design okay?

The current design is okay for adding Persian translations and RTL-aware
server-rendered content, but it is not enough for true Persian/Jalali calendar
support.

What is okay:

- Adding `FA` to `Language` is straightforward.
- Adding `messages_fa_IR.properties` follows the existing localization pattern.
- Mapping `FA` to `new Locale("fa", "IR")` is consistent with existing locale
  handling.
- Keeping API JSON dates as ISO/Gregorian dates is the right approach.

What needs improvement:

- Do not overload `DateFormat` with Jalali. A date format is not a calendar
  system.
- Add a separate calendar-system preference, for example
  `CalendarSystem.GREGORIAN` and `CalendarSystem.JALALI`.
- Add a centralized backend date formatting helper for PDF/CSV/user-facing
  server output instead of formatting dates ad hoc.

## Recommended backend model

Keep language, format, timezone, and calendar system separate.

Recommended model:

```java
public enum CalendarSystem {
    GREGORIAN,
    JALALI
}
```

Add this field to `GeneralPreferences`:

```java
@Schema(description = "Calendar system used for user-facing date display")
private CalendarSystem calendarSystem = CalendarSystem.GREGORIAN;
```

Recommended semantics:

- `language = FA` controls Persian text.
- `calendarSystem = JALALI` controls Persian/Jalali date display.
- `dateFormat = MMDDYY | DDMMYY` controls numeric order where applicable.
- `timeZone` continues to control local time conversion.
- API date transport remains Gregorian/ISO.

This keeps the product flexible. A Persian-speaking company may still want
Gregorian dates, and a non-Persian company may need Jalali dates.

## Language implementation steps

### 1. Add `FA` to the language enum

Update `api/src/main/java/com/grash/model/enums/Language.java`.

Add new enum values at the end because the file already warns that this matters:

```java
public enum Language {
    EN,
    FR,
    TR,
    ES,
    PT_BR,
    PL,
    DE,
    AR,
    IT,
    SV,
    RU,
    PT,
    HU,
    NL,
    ZH_CN,
    ZH,
    BA,
    FA;
    // always add new languages at the end
}
```

Reason: enum values may be persisted as numeric ordinals in existing schema
paths. Adding at the end minimizes migration risk.

### 2. Map `FA` to a Java locale

Update `api/src/main/java/com/grash/utils/Helper.java`.

Add:

```java
case FA:
    return new Locale("fa", "IR");
```

This makes `MessageSource`, Thymeleaf, emails, exports, notifications, and PDF
reports select Persian resources when the company language is `FA`.

### 3. Add the Persian message bundle

Create:

```text
api/src/main/resources/messages_fa_IR.properties
```

Use `messages.properties` as the canonical key list.

Rules:

- Keep every key from `messages.properties`.
- Translate values only.
- Preserve placeholders such as `{0}`, `{1}`, and `{2}`.
- Preserve quotes and escaping rules used by Java `.properties` files.
- Save as UTF-8.
- Review any HTML snippets for RTL rendering and injection safety.

### 4. Verify DTO behavior

Check these DTOs:

- `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`
- `api/src/main/java/com/grash/dto/UserSignupRequest.java`
- `api/src/main/java/com/grash/dto/requestPortal/RequestPortalPublicDTO.java`

Expected behavior:

- PATCHing `language: "FA"` works.
- Signup with `language: "FA"` works.
- Request portal public DTO can return `companyLanguage: "FA"`.
- Unknown language values still fall back according to `Language.fromString`.

### 5. Update server-side import template handling

`ImportController` accepts a `Language` query parameter for import templates.

Actions:

- Locate existing import template resources.
- Add Persian templates if the product expects localized CSV headers/examples.
- If Persian templates are not ready, make sure the controller safely falls
  back to English.

### 6. Review notification and email output

Server-side localized text is used in:

- Work order notifications.
- Request notifications.
- Preventive maintenance notification jobs.
- Purchase order notifications.
- Invite, signup, reset password, and other mail templates.

Actions:

- Confirm every `messageSource.getMessage(...)` key exists in the Persian
  bundle.
- Render all Thymeleaf templates with `fa_IR`.
- Verify `dir="rtl"` or RTL-friendly layout in templates that include large
  Persian text blocks.

## Calendar-system implementation steps

### 1. Add `CalendarSystem`

Create:

```text
api/src/main/java/com/grash/model/enums/CalendarSystem.java
```

Recommended content:

```java
package com.grash.model.enums;

public enum CalendarSystem {
    GREGORIAN,
    JALALI
}
```

### 2. Add the field to `GeneralPreferences`

Update `api/src/main/java/com/grash/model/GeneralPreferences.java`:

```java
import com.grash.model.enums.CalendarSystem;

@Schema(description = "Calendar system used for user-facing date display")
private CalendarSystem calendarSystem = CalendarSystem.GREGORIAN;
```

### 3. Add the field to patch DTOs

Update `api/src/main/java/com/grash/dto/GeneralPreferencesPatchDTO.java`:

```java
private CalendarSystem calendarSystem;
```

The frontend should be able to PATCH only this value without changing language.

### 4. Add a Liquibase migration

Add a changelog under:

```text
api/src/main/resources/db/changelog/
```

Follow the existing enum storage style in the repository.

Requirements:

- Existing companies default to Gregorian.
- New companies default to Gregorian.
- The migration is safe for production data.
- If enums are stored as smallints, use a stable value for
  `CalendarSystem.GREGORIAN`.

### 5. Keep API dates Gregorian/ISO

Do not globally change Jackson date serialization.

Backend invariant:

- Persist instants/dates as existing Java date/time types.
- Accept API date inputs as existing Gregorian/ISO values.
- Return API date outputs as existing Gregorian/ISO values.
- Convert Jalali only for user-facing strings in reports, exports, and possibly
  template-rendered content.

This avoids breaking:

- Preventive maintenance scheduling.
- Work order due dates.
- Analytics date ranges.
- Integrations and existing API clients.
- Database query logic.

## Backend date formatting helper

Create one helper for user-facing formatted dates. Suggested location:

```text
api/src/main/java/com/grash/utils/CompanyDateFormatter.java
```

Suggested API:

```java
public final class CompanyDateFormatter {
    public static String format(
        Date value,
        String timeZone,
        DateFormat dateFormat,
        CalendarSystem calendarSystem,
        Locale locale,
        boolean hideTime
    ) {
        // Convert instant to company timezone.
        // Format as Gregorian or Jalali based on calendarSystem.
        // Preserve dateFormat order where applicable.
    }
}
```

Responsibilities:

- Timezone conversion.
- Gregorian formatting.
- Jalali formatting.
- Optional time display.
- Locale-aware digits/text if product requirements call for Persian digits.

Do not scatter Jalali conversion logic through controllers.

## Reports and exports

Review and update these areas only when user-facing formatted dates are needed:

- `api/src/main/java/com/grash/controller/WorkOrderController.java`
- `api/src/main/java/com/grash/service/AsyncExportService.java`
- `api/src/main/java/com/grash/utils/CsvFileGenerator.java`
- Thymeleaf templates under `api/src/main/resources/templates/`

For PDFs:

- Pass `calendarSystem` to the Thymeleaf model with `dateFormat` and `timeZone`.
- Use the centralized formatter before rendering date strings, or expose a
  formatting helper safely to the template.

For CSV:

- Keep UTF-8 output.
- Translate headers with Persian messages.
- Format exported date cells through the centralized formatter if exported CSVs
  are intended for humans.
- Do not change machine-oriented API date serialization.

## Tests to add

Recommended backend tests:

- `Language.fromString("FA")` returns `Language.FA`.
- `Language.fromString("fa")` returns `Language.FA`.
- `Helper.getLocale(...)` returns a Persian locale for companies using `FA`.
- `messages_fa_IR.properties` has all keys from `messages.properties`.
- Placeholder counts match the English/default bundle.
- `GeneralPreferencesPatchDTO` accepts `calendarSystem`.
- Existing companies without explicit calendar system use `GREGORIAN`.
- PDF/export date formatter returns Gregorian output in Gregorian mode.
- PDF/export date formatter returns Jalali output in Jalali mode.

## Manual backend verification

Use these flows:

1. Create or update a company with `language = FA`.
2. Request company settings and confirm `FA` is returned.
3. Trigger each main notification/email path and verify Persian text.
4. Generate a work order PDF and verify text direction and date output.
5. Export CSV and verify Persian headers/text survive in UTF-8.
6. Patch `calendarSystem = JALALI` and verify user-facing report/export date
   strings if Jalali formatting is implemented.

## Backend risks

- Enum persistence risk if new enum values are inserted in the middle.
- Missing translation keys causing fallback or runtime message issues.
- Placeholder mismatches in `.properties` files.
- Confusing language with calendar system.
- Accidentally changing JSON date serialization globally.
- CSV consumers expecting raw Gregorian dates.
- Server-rendered RTL templates with broken alignment.

## Backend acceptance criteria

Backend Persian language support is ready when:

- `FA` is accepted and persisted in company preferences.
- `fa_IR` locale mapping works.
- Persian server message bundle has key parity.
- Server-generated emails/PDFs/exports can render Persian text.
- Existing languages still work.

Backend Jalali support is ready when:

- `calendarSystem` is modeled independently from `language` and `dateFormat`.
- Existing companies default safely to Gregorian.
- API JSON dates remain backward-compatible.
- User-facing backend date outputs can use Jalali where product requirements
  call for it.
