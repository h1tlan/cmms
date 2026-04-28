# Rebranding Guide for an Industry Deployment

This document explains how to rebrand this project for your own organization or
industry. Example target brand:

```text
Atlas CMMS -> PETCO CMMS
```

It also compares the complexity of rebranding with:

- Persian localization
- Persian/Jalali calendar support
- removing or changing license-gated parts

This is a technical and product guide, not legal advice.

## Short answer

There are three different levels of rebranding.

| Level | Meaning | Relative complexity |
| --- | --- | --- |
| Configuration white-labeling | Change logo, colors, brand name, support contacts using existing environment variables. | Low to medium |
| Full web product rename | Also change browser metadata, PWA manifest, default images, marketing text, docs, and hard-coded Atlas names. | Medium |
| Full ecosystem rebrand | Also rebrand mobile app name, bundle IDs, app icons, app stores, marketing site, legal text, Docker image names, and support channels. | High |

If your goal is simply:

```text
Users log in and see PETCO CMMS branding
```

then the existing white-labeling system gives you a useful start.

If your goal is:

```text
No visible Atlas/Grash identity anywhere in web, mobile, email, reports, docs,
URLs, metadata, app stores, or Docker images
```

then this is a larger product rebrand and requires code, asset, deployment, and
possibly legal/commercial decisions.

## Important licensing note

This project already has white-labeling controls, but they are license-gated.

The current product documentation says:

- custom logos require a license
- custom branding requires a license
- `BRANDING` is a license entitlement

In the current implementation, custom brand text and custom logos depend on a
valid license with the `BRANDING` entitlement.

That means a default Docker Compose run without a license key may not fully show
your custom PETCO branding, even if you set the branding environment variables.

If you intend to remove or bypass those gates, review the project licenses and
get legal guidance first.

## Existing rebranding support

The project already supports some white-labeling through environment variables.

### Brand name and contacts

Environment variable:

```text
BRAND_CONFIG
```

Expected fields:

```json
{
  "name": "PETCO CMMS",
  "shortName": "PETCO",
  "website": "https://petco.example.com",
  "mail": "maintenance-support@petco.example.com",
  "phone": "+1 555 0100",
  "addressStreet": "Example Street 1",
  "addressCity": "Example City"
}
```

Functional effect:

- changes brand text in places that use the central brand configuration
- affects some backend-generated emails and user-facing messages
- affects some web app brand display when license entitlement allows branding

### Logos

Environment variable:

```text
LOGO_PATHS
```

Example:

```json
{
  "dark": "petco-logo.png",
  "white": "petco-logo-white.png"
}
```

Docker Compose mounts a local `logo` folder into the backend container. Put the
logo files in that folder:

```text
logo/petco-logo.png
logo/petco-logo-white.png
```

Functional effect:

- backend copies these into static custom logo paths
- web can use the backend-served custom logos when branding is licensed
- emails and work order report templates can use the custom logo

### Colors

Environment variable:

```text
CUSTOM_COLORS
```

Example:

```json
{
  "emailColors": "#004B8D",
  "primary": "#004B8D",
  "secondary": "#00A3E0",
  "success": "#57CA22",
  "warning": "#FFA319",
  "error": "#FF1943",
  "info": "#33C2FF",
  "black": "#223354",
  "white": "#ffffff",
  "primaryAlt": "#002B5C"
}
```

Functional effect:

- changes web theme colors where the custom color config is used
- changes backend email header/background color through `emailColors`
- may not update every static browser/PWA color unless code/assets are changed

## Configuration-only PETCO example

For a first pass, you can add these values to `.env`:

```env
BRAND_CONFIG={"name":"PETCO CMMS","shortName":"PETCO","website":"https://petco.example.com","mail":"maintenance-support@petco.example.com","phone":"+1 555 0100","addressStreet":"PETCO Maintenance Office","addressCity":"Example City"}
LOGO_PATHS={"dark":"petco-logo.png","white":"petco-logo-white.png"}
CUSTOM_COLORS={"emailColors":"#004B8D","primary":"#004B8D","secondary":"#00A3E0","success":"#57CA22","warning":"#FFA319","error":"#FF1943","info":"#33C2FF","black":"#223354","white":"#ffffff","primaryAlt":"#002B5C"}
```

Then provide:

```text
logo/petco-logo.png
logo/petco-logo-white.png
```

Restart containers:

```bash
docker compose up -d
```

Expected result if the branding license entitlement is valid:

- custom PETCO name in supported brand-aware areas
- custom PETCO logo in supported web/email/report areas
- PETCO colors in supported theme/email areas

Expected result without branding entitlement:

- some or all branding may fall back to Atlas defaults
- custom-role-style licensing behavior may also block other advanced features

## What configuration rebranding does not fully cover

Configuration white-labeling does not appear to fully cover every user-visible
brand surface.

Areas that still need code or asset changes:

- browser tab and PWA metadata
- `manifest.json`
- static favicon files
- Open Graph metadata
- no-JavaScript fallback text
- default logo image files
- marketing/home pages
- hard-coded `atlas-cmms.com` URLs
- mobile app name
- mobile app icon and splash screen
- iOS bundle identifier
- Android package name
- deep link scheme
- app store listings
- README and user documentation
- Docker image names if you publish your own branded images

## Web app rebranding

The main web application can be rebranded in two layers.

### Runtime white-labeling

Use:

- `BRAND_CONFIG`
- `LOGO_PATHS`
- `CUSTOM_COLORS`
- valid license with `BRANDING`

This is the fastest path to make the logged-in app look branded.

### Static web identity changes

For a complete PETCO CMMS web identity, also update:

- browser title/metadata
- favicon
- PWA manifest name and short name
- PWA icons
- Open Graph image and title
- no-JavaScript fallback text
- default Atlas logo assets

Typical target changes:

```text
Atlas CMMS -> PETCO CMMS
Atlas -> PETCO
atlas-cmms.com -> your PETCO CMMS URL
```

This is more than configuration because those values are part of static web
assets.

## Backend-generated email and report rebranding

The backend already has central branding support for emails and report templates.

Brand-aware backend output can use:

- brand name
- brand short name
- website
- support email
- phone
- address
- email colors
- custom logo

For PETCO CMMS, validate:

- invitation emails
- password reset emails
- work order notification emails
- request notification emails
- PM notification emails
- work order PDF/report output
- footer/contact details
- sender/from email configuration

Important: email content may still include translated strings that reference the
old product name if a string was hard-coded outside the brand configuration.
Those should be reviewed manually.

## Marketing site rebranding

The `home/` site is a marketing website, not the operational CMMS app.

For a full PETCO deployment, decide whether you need the marketing site at all.

If yes, rebrand:

- page titles
- descriptions
- canonical URLs
- Open Graph metadata
- pricing pages
- feature pages
- industry pages
- legal pages
- app store links
- demo links
- logo assets
- colors

This is usually a separate workstream from operational CMMS branding.

If the system is only for internal industry use, you may choose not to deploy the
marketing site.

## Mobile app rebranding

Mobile is the largest rebranding gap.

The mobile app currently has hard-coded identity values such as:

- app display name
- app slug
- deep link scheme
- iOS bundle identifier
- Android package name
- app icon
- splash image
- Expo project/update settings
- native configuration files

For PETCO CMMS mobile, you must decide between two strategies.

### Strategy A: internal PETCO mobile build

Keep the codebase but create a PETCO-branded build.

Change:

- app name: `PETCO CMMS`
- app icon
- splash screen
- primary colors
- deep link scheme, for example `petcocmms`
- iOS bundle identifier, for example `com.petco.cmms`
- Android package name, for example `com.petco.cmms`
- app update/project configuration

Functional impact:

- users install PETCO CMMS as a separate app
- app store or MDM deployment can use PETCO branding
- mobile must be rebuilt and redistributed

### Strategy B: keep Atlas mobile app

Use the existing mobile app and point it to your backend.

Functional impact:

- faster and less complex
- users still see Atlas CMMS branding on mobile
- not a complete rebrand

For a serious industry deployment, Strategy A is more consistent but more work.

## Documentation and training rebranding

Rebrand all user-facing documentation:

- admin guide
- technician guide
- requester guide
- maintenance manager guide
- mobile guide
- support guide
- onboarding materials
- screenshots
- training videos
- help desk articles

For example:

```text
"Open Atlas CMMS" -> "Open PETCO CMMS"
"Contact Atlas support" -> "Contact PETCO maintenance support"
```

This is often underestimated. A product can look rebranded in the app but still
feel unfinished if docs and screenshots show the old brand.

## Deployment and operations rebranding

Optional but recommended for full ownership:

- Docker Compose project name
- container names
- Docker image names
- domain names
- email sender domains
- MinIO bucket naming
- monitoring dashboard names
- backup folder names
- support mailbox

Example:

```text
atlas-cmms-backend -> petco-cmms-backend
atlas-cmms-frontend -> petco-cmms-frontend
atlas_db -> petco_cmms_db
```

These changes are not always visible to end users, but they matter for operations
and long-term ownership.

## Industry-specific rebranding vs industry-specific customization

Changing the name to PETCO CMMS is branding.

Changing the product to fit your industry is customization.

Industry customization may include:

- industry-specific asset categories
- industry-specific work order types
- specialized inspection checklists
- regulatory fields
- custom reports
- custom request forms
- approval workflows
- terminology changes
- safety procedures
- industry KPIs

For example, if PETCO is in a petrochemical or energy context, the system may
need terms and processes such as:

- plants
- units
- equipment tags
- isolation permits
- criticality
- maintenance windows
- safety work permits
- calibration
- inspections
- regulatory compliance

Those are functional changes, not just visual rebranding.

## Terminology rebranding

Consider whether CMMS terms should change.

Examples:

| Current term | Possible PETCO term |
| --- | --- |
| Work order | Maintenance job / work request |
| Asset | Equipment / unit / tag |
| Location | Site / plant / area |
| Request | Service request |
| Technician | Maintenance technician |
| Vendor | Contractor / supplier |
| Preventive maintenance | Planned maintenance |

If terminology changes are needed, they should be handled like localization:

- update translation keys/values
- avoid hard-coded label changes in many places
- document approved terms
- validate with real maintenance users

## Complexity comparison

### Rebranding vs Persian localization

Persian localization includes:

- backend language enum and message bundle
- frontend translation file
- mobile translation file if mobile is included
- RTL checks
- translated user-facing text

Rebranding includes:

- brand config
- logos
- colors
- static web metadata
- mobile app identity
- docs and screenshots
- marketing pages
- email/report branding

Comparison:

| Work item | Relative complexity |
| --- | --- |
| Basic web/backend white-labeling | Lower than full Persian localization |
| Full web rebrand | Similar to Persian localization |
| Full web + mobile + docs rebrand | Higher than Persian localization |

Reason:

- Persian localization touches many text strings.
- Full rebranding touches fewer strings but more surfaces: assets, metadata,
  mobile identity, deployment, docs, and legal/support materials.

### Rebranding vs Persian/Jalali calendar

Persian/Jalali calendar support includes:

- new calendar preference
- date conversion
- date display changes
- date picker changes
- report/export date formatting
- mobile date handling
- timezone correctness

Comparison:

| Work item | Relative complexity |
| --- | --- |
| Basic white-labeling | Much lower than Jalali support |
| Full web/mobile rebrand | Similar or lower than Jalali support, depending on mobile/app-store needs |
| Industry terminology + process customization | Can exceed Jalali support if it changes workflows |

Reason:

- Jalali support changes business/date behavior.
- Rebranding mostly changes presentation and identity, unless terminology and
  industry workflows are also changed.

### Rebranding vs removing license parts

Removing or changing licensing is a different category.

It is not just technical. It involves:

- legal review
- commercial license review
- product strategy
- feature-gate behavior
- UI messaging
- plan/subscription logic
- entitlement checks
- support expectations

Comparison:

| Work item | Relative complexity |
| --- | --- |
| Basic white-labeling with valid license | Much lower than removing license gates |
| Full rebrand while keeping licensing | Lower than removing license gates |
| Removing/changing license behavior | Higher risk than rebranding because of legal and product implications |

If you have a valid commercial license with branding entitlement, rebranding is
straightforward compared with removing license checks.

If you do not have a branding entitlement and want to remove checks, that becomes
a licensing/legal/product decision, not just a rebranding task.

## Effort bands without calendar estimates

Instead of guessing calendar time, use effort bands.

### Band 1: Basic PETCO white-label

Scope:

- set `BRAND_CONFIG`
- set `LOGO_PATHS`
- set `CUSTOM_COLORS`
- provide logo files
- verify web app, email, report logo

Complexity:

```text
Low to medium
```

Dependencies:

- valid branding license entitlement
- good PETCO logo assets
- final brand colors

### Band 2: Complete web rebrand

Scope:

- Band 1
- update PWA manifest
- update index metadata
- update favicons
- update static images
- update hard-coded Atlas text in web surfaces
- update docs/screenshots

Complexity:

```text
Medium
```

Dependencies:

- brand style guide
- web QA
- documentation update

### Band 3: Web + backend + mobile rebrand

Scope:

- Band 2
- rebrand mobile app
- update app identifiers
- update mobile icons/splash
- rebuild and redistribute mobile
- update app store or MDM deployment
- update deep links

Complexity:

```text
High
```

Dependencies:

- mobile build pipeline
- app store or enterprise distribution plan
- mobile QA on iOS and Android

### Band 4: Industry rebrand plus workflow customization

Scope:

- Band 3
- change terminology
- add industry-specific fields
- add industry-specific reports
- add approval/safety workflows
- add compliance requirements
- train users

Complexity:

```text
High to very high
```

Dependencies:

- maintenance domain workshops
- approved terminology
- workflow design
- reporting requirements

## Recommended path for PETCO CMMS

### Step 1: Decide the target rebrand level

Choose one:

1. Internal white-label only
2. Full web rebrand
3. Full web + mobile rebrand
4. Full industry-specific product customization

Do not start by changing every occurrence of "Atlas" blindly. First decide the
target level.

### Step 2: Prepare brand assets

Prepare:

- dark logo
- white logo
- favicon
- PWA icons
- mobile icon
- mobile splash
- primary color
- secondary color
- email header color
- brand name
- short name
- support email
- support website
- support phone/address

### Step 3: Configure white-label variables

Use:

- `BRAND_CONFIG`
- `LOGO_PATHS`
- `CUSTOM_COLORS`

Validate with a license that includes `BRANDING`.

### Step 4: Review web surfaces

Check:

- login
- registration
- dashboard
- sidebar/header
- settings
- emails
- work order report
- browser tab metadata
- PWA installation
- public request portal

### Step 5: Decide mobile approach

If mobile users are important, choose:

- keep Atlas-branded mobile temporarily
- or create PETCO-branded mobile app

For a complete rebrand, mobile should eventually be branded too.

### Step 6: Update documentation and training

Create PETCO-specific:

- admin manual
- technician manual
- requester manual
- quick start guide
- screenshots
- support process

## Rebranding checklist

### Configuration

- [ ] `BRAND_CONFIG` set
- [ ] `LOGO_PATHS` set
- [ ] `CUSTOM_COLORS` set
- [ ] logo files mounted
- [ ] branding license available

### Web app

- [ ] header/sidebar logo
- [ ] login/register pages
- [ ] company profile pages
- [ ] public request portal
- [ ] browser title/meta
- [ ] PWA manifest
- [ ] favicon/icons
- [ ] no-JavaScript fallback text

### Backend output

- [ ] invitation email
- [ ] password reset email
- [ ] work order email
- [ ] request email
- [ ] PM notification email
- [ ] work order report/PDF
- [ ] sender email/domain

### Marketing site

- [ ] page titles
- [ ] homepage copy
- [ ] pricing copy
- [ ] feature pages
- [ ] app links
- [ ] canonical URLs
- [ ] legal/support pages

### Mobile

- [ ] app name
- [ ] app icon
- [ ] splash screen
- [ ] theme colors
- [ ] deep link scheme
- [ ] iOS bundle identifier
- [ ] Android package name
- [ ] app store/MDM listing

### Documentation

- [ ] admin guide
- [ ] technician guide
- [ ] requester guide
- [ ] screenshots
- [ ] training materials
- [ ] support contact details

## Risks

### 1. Mixed branding

The most common risk is users seeing both brands:

```text
PETCO logo in app header
Atlas CMMS in browser title
Atlas icon in PWA install
Atlas mobile app on phone
Atlas support email in notifications
```

Avoid this with a complete brand surface checklist.

### 2. Branding without license

If branding variables are set but the `BRANDING` entitlement is missing, the
system can fall back to Atlas defaults.

Avoid this by checking license state before rollout.

### 3. Mobile distribution complexity

Changing mobile app identifiers can create a new app instead of updating the old
one.

Decide early whether PETCO CMMS mobile is:

- an internal enterprise app
- a new public app store listing
- a temporary Atlas-branded app connected to PETCO backend

### 4. Industry terminology mismatch

Visual branding may succeed while the words still do not match the industry.

For example:

- equipment tag vs asset
- plant area vs location
- maintenance job vs work order

Terminology should be reviewed by maintenance users.

### 5. Legal/licensing confusion

Removing license gates is not the same as rebranding.

Keep those decisions separate:

- rebranding with licensed white-labeling
- rebranding by code changes
- removing licensing behavior

Each has different legal and product implications.

## Final recommendation

For PETCO CMMS, use a phased approach:

1. Start with configuration white-labeling using `BRAND_CONFIG`, `LOGO_PATHS`,
   and `CUSTOM_COLORS`.
2. Verify whether the `BRANDING` entitlement is available.
3. Rebrand static web metadata and icons.
4. Decide whether the marketing site is needed.
5. Rebrand mobile only if mobile deployment is part of the real rollout.
6. Treat terminology and industry workflow changes as a separate product
   customization phase.

Relative complexity:

```text
Basic rebrand < Persian localization < full web/mobile rebrand ~= Jalali support
< license-removal/legal-product changes < industry-specific workflow redesign
```

The simplest useful rebrand is not very complex if you have the branding
entitlement and assets ready. A complete rebrand across web, backend output,
marketing, mobile, documentation, deployment, and industry terminology is a
larger cross-product effort.

