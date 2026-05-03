# Rebranding from Atlas CMMS to Polfilm CMMS

This is a concrete, file-level playbook for rebranding this project from
**Atlas CMMS** (the upstream brand) to **Polfilm CMMS** (your target brand).

It is the practical companion to the more general
[Rebranding guide for industry deployment](./Rebranding%20guide%20for%20industry%20deployment.md).
That doc explains the *concepts*; this one tells you exactly which files,
strings, environment variables, and assets need to change to ship a
Polfilm CMMS deployment.

It also gives a realistic complexity assessment, an effort-banded playbook,
and a checklist you can follow end-to-end.

> ⚠️ **Licensing reminder.** The runtime brand-config + custom-logo + custom-color
> path is gated by the `BRANDING` license entitlement. If you do **not** hold a
> license that includes `BRANDING`, the runtime will fall back to "Atlas CMMS"
> defaults regardless of the env vars you set. See §3 for what to do about it.
> Modifying the gate is a separate decision, see the
> [Meter trigger license removal walkthrough](./Meter%20trigger%20license%20removal%20walkthrough.md)
> for the analogous pattern on a different feature.

---

## 1. TL;DR

There are three sensible target levels. Pick one *before* you start changing
anything.

| Level | What users see | Touches | Complexity |
| --- | --- | --- | --- |
| **L1 — Runtime white-label** | "Polfilm CMMS" name + Polfilm logo + Polfilm colors inside the logged-in web app, emails, and PDF reports. | 3 env vars + 2 PNGs. License-gated. | Low |
| **L2 — Full web rebrand** | L1 + browser tab title, favicon, PWA name/icons, Open Graph, no-JS fallback, Swagger title, landing pages. No "Atlas" anywhere on the web surface. | ~25 files in `frontend/`, `api/`, `home/`. Some i18n strings. Not license-gated. | Medium |
| **L3 — Full ecosystem rebrand** | L2 + a separately published "Polfilm CMMS" mobile app (new bundle id, new icons, new app store listing), rebranded Docker images, rebranded backups, rebranded URLs/domains, rebranded user docs, rebranded license-plan strings. | ~100+ files including iOS/Android native projects and store listings. | High |

The minimum useful rebrand for an internal Polfilm rollout is **L1 + the
"static metadata" subset of L2** — that gets you a clean web surface that
says "Polfilm CMMS" everywhere a user looks, without rebuilding the mobile
app.

---

## 2. What "Polfilm CMMS" needs to look like

Before you touch code, lock down the brand inputs. Everything else
references these. Treat them as configuration; do not let them drift across
files.

| Brand input | Suggested value | Used in |
| --- | --- | --- |
| `name` | `Polfilm CMMS` | Web titles, emails, mobile app name |
| `shortName` | `Polfilm` | Browser short name, sidebar, mobile splash |
| `website` | e.g. `https://cmms.polfilm.example` | Email footers, SEO meta |
| `mail` | e.g. `cmms-support@polfilm.example` | Email "from" / contact |
| `phone` | Polfilm support phone | Email/footer/contact |
| `addressStreet` / `addressCity` | Polfilm office address | Emails, invoices |
| Primary color | a Polfilm brand hex, e.g. `#0E5AA7` | Web theme, email header |
| Secondary color | second brand hex, e.g. `#F2A900` | Web accents |
| Logo (dark on light) | `polfilm-logo.png` (square or wide PNG) | App header on light theme, emails, PDF |
| Logo (light on dark) | `polfilm-logo-white.png` | App header on dark theme |
| Favicon | `polfilm-favicon.ico` + 16/32 PNGs | Browser tab |
| PWA icon | 192/256/384/512 px PNG | "Install app" |
| Mobile icon | 1024×1024 PNG (Expo will resize) | iOS/Android home screen |
| Mobile splash | 2048×2048 PNG | App launch |
| Deep link scheme | `polfilmcmms` | Mobile push, OAuth callbacks |
| iOS bundle id | `com.polfilm.cmms` | App Store / TestFlight / MDM |
| Android package | `com.polfilm.cmms` | Play Store / MDM |

Hold a single shared sheet (or `branding/README.md`) with these values and
**do not** inline them in code anywhere except where this guide says to.

---

## 3. The licensing checkpoint (read this before L1)

The brand-customization runtime is wrapped behind the `BRANDING` license
entitlement in two places:

```37:48:api/src/main/java/com/grash/service/BrandingService.java
    public BrandConfig getBrandConfig() {
        BrandConfig defaultConfig = BrandConfig.builder()
                .name("Atlas CMMS")
                .shortName("Atlas")
                .website("https://www.atlas-cmms.com")
                .mail("contact@atlas-cmms.com")
                .phone("+212 6 30 69 00 50")
                .addressStreet("410, Boulevard Zerktouni, Hamad, №1")
                .addressCity("Casablanca-Morocco 20040")
                .build();
        if (!licenseService.hasEntitlement(LicenseEntitlement.BRANDING)) return defaultConfig;
        if (brandRawConfig == null || brandRawConfig.isEmpty()) {
            return defaultConfig;
        } else {
```

```18:48:frontend/src/hooks/useBrand.ts
export function useBrand(): BrandConfig {
  const defaultBrand: Omit<BrandConfig, 'logo'> = {
    name: 'Atlas CMMS',
    shortName: 'Atlas',
    website: 'https://www.atlas-cmms.com',
    mail: 'contact@atlas-cmms.com',
    phone: '+212 6 30 69 00 50',
    addressStreet: '410, Boulevard Zerktouni, Hamad, №1',
    addressCity: 'Casablanca-Morocco 20040'
  };
  const isLicenseValid = useLicenseEntitlement('BRANDING');
  return {
    logo: {
      white: customLogoPaths
        ? isLicenseValid == null
          ? null
          : isLicenseValid
          ? CUSTOM_WHITE_LOGO
          : DEFAULT_WHITE_LOGO
        : DEFAULT_WHITE_LOGO,
```

Practical implications for Polfilm:

- **If you have a commercial license including `BRANDING`** → set the env
  vars and you're done with L1; nothing else needs to be done in code at
  this level.
- **If you do not have it but the deployment is internal/self-hosted** →
  you have two clean options:
  1. **Hardcode the Polfilm defaults** (recommended for an internal-only
     deployment): replace the seven `defaultBrand` strings in
     `BrandingService.java`, `useBrand.ts`, and `home/src/utils/serverBrand.ts`
     with Polfilm values. The license check then becomes a no-op because
     "Polfilm CMMS" is the default. Minimal blast radius.
  2. **Remove the license gate** for branding only, the same way the meter
     trigger gate was removed in
     [Meter trigger license removal walkthrough](./Meter%20trigger%20license%20removal%20walkthrough.md).
     One-line removal in each of the two functions above. Bigger
     conceptual change; same mechanical pattern.

The rest of this guide assumes you have picked a path. The file-level
changes for L2/L3 are the same either way — only L1 differs.

---

## 4. Where "Atlas" actually lives in this repo

This is the inventory. Numbers are exact at the time of writing; verify with
`grep -rni "atlas"` on your branch. Java package names and the Postgres
schema name `grash` are *not* user-visible and are deliberately **not**
rebranded — see §10.

### 4.1 Backend (`api/`)

| File | What it controls | Action |
| --- | --- | --- |
| `api/src/main/java/com/grash/service/BrandingService.java` | Default brand returned when no `BRAND_CONFIG` or no `BRANDING` license. | Replace 7 defaults with Polfilm values. |
| `api/src/main/java/com/grash/configuration/OpenApiConfig.java` | Swagger / OpenAPI title, description, server URLs, contact, security key copy. | Replace ~15 occurrences of "Atlas" / `atlas-cmms.com` / `atlas-cmms`. |
| `api/src/main/java/com/grash/configuration/SwaggerConfig.java` | OpenAPI group name `atlas-cmms`. | Rename to `polfilm-cmms` (also update Swagger UI URL refs). |
| `api/src/main/java/com/grash/controller/WebhookController.java` | Webhook documentation strings ("Atlas CMMS license key" / "license key renewal"). | Replace 2 strings. |
| `api/src/main/java/com/grash/controller/WebhookEndpointController.java` | Webhook endpoint operation description. | Replace 4 occurrences. |
| `api/src/main/java/com/grash/controller/FileController.java` | Comments mentioning "Atlas CMMS Terms of service". | Cosmetic; safe to leave or rename. |
| `api/src/main/java/com/grash/utils/Consts.java` | Self-hosted plan display names ("Professional Atlas CMMS license", "Business Atlas CMMS license"). | Replace 5 strings if you ship plans, or leave if Polfilm doesn't expose paid plans. |
| `api/src/main/resources/static/api-docs.html` | Swagger UI title + `data-url="/v3/api-docs/atlas-cmms"`. | Replace 2 occurrences (and keep in sync with the rename in `SwaggerConfig`). |
| `api/src/main/resources/templates/checkout-complete.html` | Email after self-hosted plan purchase. | Replace "Atlas Team" + footer copyright. |
| `api/pom.xml` | Project description "Atlas application programming interface". | Replace 1 line. Cosmetic. |
| `api/README.md` | Document title. | Replace 1 line. Cosmetic. |

### 4.2 Frontend web app (`frontend/`)

| File | What it controls | Action |
| --- | --- | --- |
| `frontend/src/hooks/useBrand.ts` | Default brand block when `BRANDING` license absent. | Replace 7 defaults. |
| `frontend/public/index.html` | Browser `<meta name="author">`, OG title, OG site name, no-JavaScript fallback. | Replace 5 occurrences. |
| `frontend/public/manifest.json` | PWA `name` + `short_name` + icon paths. | Replace 2 strings + swap icon paths. |
| `frontend/public/.well-known/assetlinks.json` | Android App Links to mobile app (uses `com.atlas.cmms`). | Replace package name → `com.polfilm.cmms`. |
| `frontend/package.json` | `title` field. | Replace 1 string. |
| `frontend/index-now.js` | Hard-coded sitemap host. | Replace if you generate sitemaps. |
| `frontend/src/utils/urlPaths.ts` | URLs to atlas docs/support/FAQ. | Replace each external URL with Polfilm-equivalent or remove. |
| `frontend/src/layouts/ExtendedSidebarLayout/Sidebar/SidebarFooter/index.tsx` | "Atlas CMMS" string in the sidebar footer. | Replace; better — switch it to use `useBrand().shortName`. |
| `frontend/src/components/MobileAppDownloadDialog/index.tsx` | "Atlas" mention. | Replace or use `useBrand()`. |
| `frontend/src/content/pages/Auth/Register/Cover/index.tsx` | "Atlas" mentioned 6 times in registration cover. | Replace; preferably read from `useBrand()`. |
| `frontend/src/content/own/CompanyProfile/CompanyPlan.tsx` | Plan-related text. | Verify; replace if it mentions "Atlas". |
| `frontend/README.md` | Document title. | Replace 1 line. Cosmetic. |
| `frontend/src/i18n/translations/*.ts` | Marketing/SEO translations referencing "Atlas CMMS" inside the app's i18n bundle (mostly the home-style text used by free CMMS / pricing sub-pages). | ~5 occurrences per locale × ~14 locales ≈ ~70 total strings. Mechanical replacement. The Dutch (`nl.ts`) bundle uses "Atlas" as a brand verb a lot (~22 hits), worth dedicated attention. |

### 4.3 Marketing site (`home/`)

The marketing site is mostly *Atlas marketing copy*. For an internal Polfilm
deployment you have two reasonable choices:

- **Don't deploy `home/` at all.** Save ~80 source files of work. Users
  reach the app directly through your internal URL.
- **Deploy a stripped-down Polfilm landing page** that doesn't try to be a
  marketing site for "open-source CMMS for X industry" — those pages
  (`home/app/[locale]/industries/...`, `features/...`, `pricing`, etc.) are
  upstream marketing material and most are not relevant to Polfilm.

If you do keep it, the canonical inventory:

| File group | Action |
| --- | --- |
| `home/src/utils/serverBrand.ts` | Same defaults as `useBrand.ts`; replace 7 strings. |
| `home/src/utils/metadata.ts` | Page-title and OG generators referencing "Atlas". Replace. |
| `home/src/components/Footer/index.tsx` | Footer copy. Replace. |
| `home/app/[locale]/layout.tsx` | Root metadata title / OG. Replace 2 strings. |
| `home/app/[locale]/page.tsx` | Homepage hero copy. Replace 5 strings. |
| `home/app/[locale]/pricing/page.tsx` | Pricing page. Replace 3 strings. |
| `home/app/[locale]/industries/*/page.tsx` | 7 industry landing pages. ~5 occurrences each. Consider deleting these instead. |
| `home/app/[locale]/features/*/page.tsx` | 5 feature pages. ~4 occurrences each. |
| `home/app/[locale]/{privacy,terms-of-service,deletion-policy}/page.tsx` | Legal pages. **Replace carefully** — they're legal copy, not marketing. Polfilm's legal team should review or replace. |
| `home/app/[locale]/mb-app/page.tsx` | Mobile app landing page; uses "Atlas". Replace. |
| `home/app/sitemap.ts` | Hard-coded URLs. Replace. |
| `home/public/.well-known/assetlinks.json` | App-link to mobile app. Replace package. |
| `home/public/robots.txt` | `Sitemap` URL. Replace. |
| `home/wrangler.jsonc` | Cloudflare worker name. Replace if you publish to CF. |
| `home/src/i18n/translations/*.ts` | Same pattern as the frontend i18n files; ~3 strings per locale × 14 locales ≈ ~40 total. |

### 4.4 Mobile app (`mobile/`)

This is the deepest workstream. The mobile app has identity baked into
*native* config files plus app-store metadata.

#### Native iOS

```text
mobile/ios/AtlasCMMS/Info.plist
mobile/ios/AtlasCMMS.xcodeproj/                            # folder name + many internal refs
mobile/ios/AtlasCMMS.xcodeproj/xcshareddata/xcschemes/AtlasCMMS.xcscheme
mobile/ios/AtlasCMMS.xcworkspace/contents.xcworkspacedata
mobile/ios/Podfile                                         # `target 'AtlasCMMS' do`
```

For a clean rename of the iOS target (`AtlasCMMS` → `PolfilmCMMS`) the most
robust route is:

```bash
cd mobile
# 1. rename Expo-controlled identity in app.config.ts (see §5.4)
# 2. let Expo prebuild regenerate the iOS project from scratch:
npx expo prebuild --clean -p ios
```

`prebuild --clean` regenerates the entire `ios/` folder from
`app.config.ts`, so you don't have to hand-edit `project.pbxproj`. This is
the recommended path. Manual `.xcodeproj` editing works but is brittle.

#### Native Android

```text
mobile/android/app/src/main/res/values/strings.xml          # <string name="app_name">
mobile/android/app/src/main/AndroidManifest.xml             # com.atlas.cmms references
mobile/android/app/src/main/java/com/atlas/cmms/MainActivity.kt
mobile/android/app/src/main/java/com/atlas/cmms/MainApplication.kt
mobile/android/app/build.gradle                             # applicationId
mobile/android/settings.gradle                              # rootProject.name
```

Same recommendation — change `app.config.ts`, then run
`npx expo prebuild --clean -p android` to regenerate. Manual editing is
possible but you must:

1. Rename folder `mobile/android/app/src/main/java/com/atlas/cmms` →
   `com/polfilm/cmms`.
2. Update `package` declaration on top of the two `.kt` files.
3. Update `applicationId` and `namespace` in `build.gradle`.
4. Update `package` attribute in `AndroidManifest.xml`.
5. Update `rootProject.name` in `settings.gradle`.

#### Expo / cross-platform

| File | Change |
| --- | --- |
| `mobile/app.config.ts` | `name`, `slug`, `scheme`, `ios.bundleIdentifier`, `android.package`, `extra.eas.projectId`, camera permission copy. |
| `mobile/package.json` + `package-lock.json` | `"name": "atlas"` → `"name": "polfilm-cmms"`. |
| `mobile/utils/fields.ts` | Placeholder "Atlas" — replace or remove. |
| `mobile/screens/WelcomeScreen.tsx` | `<Text>Atlas</Text>` welcome line. Replace with `Polfilm` or `useBrand`. |
| `mobile/screens/auth/RegisterScreen.tsx` | Hard-coded `https://atlas-cmms.com/terms-of-service`. Replace with Polfilm legal URL or hide. |
| `mobile/navigation/index.tsx` | "Atlas" mention. Replace. |
| `mobile/i18n/translations/*.ts` | ~14 locales × ~17 strings each ≈ ~230 occurrences. Mechanical. |
| `mobile/README.md` | Cosmetic. |

#### App-store / distribution

This is *not* a code change but it's the most expensive part of L3:

- A **new** Apple App Store record under `com.polfilm.cmms`
  (you cannot rename an existing app's bundle id — old users would be
  orphaned; you must publish a new app).
- A **new** Google Play listing under `com.polfilm.cmms` for the same
  reason.
- A **new** Expo/EAS project (or update settings); the existing Expo
  project id `803b5007-0c60-4030-ac3a-c7630b223b92` is hardcoded in
  `mobile/app.config.ts`. Generate a new one for Polfilm.
- New code-signing identities (Apple Developer team, Google Play signing
  key).
- Push notification credentials (APNs / FCM `google-services.json`) per
  bundle.
- New deep-link host (`assetlinks.json` for Android, Apple App Site
  Association for iOS).

If Polfilm distributes mobile via MDM only (Intune / Jamf / similar) and
not via the public stores, you skip the store listings but still need new
signing keys.

### 4.5 Operations / Docker / scripts

| File | What it does | Action |
| --- | --- | --- |
| `docker-compose.yml` | `name: atlas-cmms`, container names `atlas-cmms-backend`, `atlas-cmms-frontend`, `atlas_db`, `atlas_minio`; bucket `atlas-bucket`; image `intelloop/atlas-cmms-backend`. | Rename project, containers, bucket. Image rename only if you publish your own images (see §6). |
| `scripts/backup/atlas-backup.{sh,ps1}` | Backup script names + variables. | Optional rename for hygiene. |
| `dev-docs/Atlas Cmms Nginx proxy setup.md`, `dev-docs/*.md` | Internal ops docs that mention Atlas. | Optional rewrite for Polfilm-internal use. |

---

## 5. Step-by-step playbook

This is the order to do things in. Doing them out of order causes painful
rework (especially mobile).

### 5.1 Decide L1 vs L2 vs L3

If unsure, start at L1 and validate it. You can always add L2 / L3 later.

### 5.2 Lock the brand inputs sheet (§2)

Get sign-off from whoever owns "Polfilm" branding before you put hex codes
into source files. A single change to a primary color later means re-rolling
mobile screenshots and email previews.

### 5.3 L1 — runtime white-label

This is purely a `.env` + asset operation, so any operator can do it
without touching source code (license permitting):

```env
# .env
BRAND_CONFIG={"name":"Polfilm CMMS","shortName":"Polfilm","website":"https://cmms.polfilm.example","mail":"cmms-support@polfilm.example","phone":"+48 00 000 00 00","addressStreet":"Polfilm HQ","addressCity":"<city>"}
LOGO_PATHS={"dark":"polfilm-logo.png","white":"polfilm-logo-white.png"}
CUSTOM_COLORS={"emailColors":"#0E5AA7","primary":"#0E5AA7","secondary":"#F2A900","success":"#57CA22","warning":"#FFA319","error":"#FF1943","info":"#33C2FF","black":"#223354","white":"#ffffff","primaryAlt":"#08407A"}
LICENSE_KEY=<your_keygen_license_with_BRANDING_entitlement>
```

Place logos in the `./logo` folder (mounted into the API container at
`/app/static/images`):

```text
logo/polfilm-logo.png
logo/polfilm-logo-white.png
```

Restart:

```bash
docker compose up -d
```

Verify:

```bash
curl http://localhost:8080/license/state | jq '.entitlements'
# expect "BRANDING" in the array
curl http://localhost:8080/branding/config | jq
# expect {"name":"Polfilm CMMS", ...}
```

If you don't have the `BRANDING` license, see §3 — change the defaults in
`BrandingService.java`, `useBrand.ts`, and `home/src/utils/serverBrand.ts`
to the Polfilm values directly.

### 5.4 L2 — full web rebrand

Do these in order; each step is independently shippable.

1. **Backend defaults + Swagger metadata** — replace strings in:
   - `api/src/main/java/com/grash/service/BrandingService.java`
   - `api/src/main/java/com/grash/configuration/OpenApiConfig.java`
   - `api/src/main/java/com/grash/configuration/SwaggerConfig.java`
   - `api/src/main/resources/static/api-docs.html`
   - `api/src/main/resources/templates/checkout-complete.html`
   - `api/src/main/java/com/grash/controller/WebhookController.java`
   - `api/src/main/java/com/grash/controller/WebhookEndpointController.java`
   - `api/src/main/java/com/grash/utils/Consts.java`
   - `api/pom.xml`, `api/README.md`

2. **Frontend defaults + static metadata**:
   - `frontend/src/hooks/useBrand.ts`
   - `frontend/public/index.html` (5 occurrences)
   - `frontend/public/manifest.json` (name + short_name + icons)
   - Replace `frontend/public/favicon.ico`, `favicon-16x16.png`,
     `favicon-32x32.png`, plus the four PWA icons
     (`/static/images/logo/logo.png` and the white variant).
   - `frontend/public/.well-known/assetlinks.json` (Android package).
   - `frontend/package.json` `title` field.
   - `frontend/src/utils/urlPaths.ts` external URLs.
   - `frontend/src/layouts/.../SidebarFooter/index.tsx`,
     `frontend/src/components/MobileAppDownloadDialog/index.tsx`,
     `frontend/src/content/pages/Auth/Register/Cover/index.tsx` —
     replace inline "Atlas" with `useBrand()` calls (best practice) or
     direct strings (quicker).

3. **i18n strings** in `frontend/src/i18n/translations/*.ts`. Use a
   one-shot script to keep things consistent across locales:

   ```bash
   # dry run
   rg -l "Atlas CMMS|Atlas " frontend/src/i18n/translations
   # apply (review before running!)
   sed -i.bak 's/Atlas CMMS/Polfilm CMMS/g; s/\bAtlas\b/Polfilm/g' \
       frontend/src/i18n/translations/*.ts
   ```

   The Dutch bundle uses "Atlas" as a verb-like marker much more often
   (~22 strings) — verify the result reads naturally in that locale.

4. **Marketing site (`home/`) — only if deploying it.** Same shape:
   `serverBrand.ts`, `metadata.ts`, layout, page copy, footer, sitemap.
   Or, simpler: do not deploy `home/` and skip this entirely.

5. **Build, smoke-test, and ship.**

   ```bash
   docker compose build api frontend
   docker compose up -d api frontend
   ```

   Smoke-test surfaces (§9 has the full checklist).

### 5.5 L3 — mobile rebrand

Order matters here.

1. Make sure you have:
   - A new Apple Developer team / app id ready.
   - A new Google Play signing key / `applicationId` ready.
   - A new `google-services.json` for Firebase (if you use FCM push).
   - A new Expo / EAS project id (run `eas init` after you change
     `app.config.ts`).
2. Edit `mobile/app.config.ts`:

   ```diff
   -  name: 'Atlas CMMS',
   -  slug: 'atlas-cmms',
   -  scheme: 'atlascmms',
   +  name: 'Polfilm CMMS',
   +  slug: 'polfilm-cmms',
   +  scheme: 'polfilmcmms',
      ...
      ios: {
   -    bundleIdentifier: 'com.cmms.atlas',
   +    bundleIdentifier: 'com.polfilm.cmms',
        ...
      },
      android: {
   -    package: 'com.atlas.cmms',
   +    package: 'com.polfilm.cmms',
        ...
      },
      extra: {
        eas: {
   -      projectId: '803b5007-0c60-4030-ac3a-c7630b223b92'
   +      projectId: '<your-new-eas-project-id>'
        }
      },
      plugins: [
        ...
        ['expo-camera', {
   -      cameraPermission: 'Allow Atlas to access camera.'
   +      cameraPermission: 'Allow Polfilm CMMS to access camera.'
        }],
   ```

3. Update `mobile/package.json` `name`.
4. Replace assets:
   - `mobile/assets/images/icon.png`
   - `mobile/assets/images/splash.png`
   - `mobile/assets/images/adaptive-icon.png`
   - `mobile/assets/images/favicon.png`
   - `mobile/assets/images/notification.png`
5. Replace remaining hard-coded "Atlas" strings:
   - `mobile/screens/WelcomeScreen.tsx`
   - `mobile/screens/auth/RegisterScreen.tsx` (terms-of-service URL)
   - `mobile/navigation/index.tsx`
   - `mobile/utils/fields.ts`
   - `mobile/i18n/translations/*.ts` (use the same `sed` pattern as
     §5.4.3).
6. Regenerate native:

   ```bash
   cd mobile
   rm -rf ios android
   npx expo prebuild --clean
   ```

   This rebuilds `ios/` and `android/` from `app.config.ts` with the new
   bundle id, package, scheme, and target name. You no longer need to
   manually edit `project.pbxproj`, `Info.plist`, or any of the
   `AndroidManifest`/Gradle/Kotlin files — they are regenerated.

7. Build and smoke-test:

   ```bash
   eas build -p ios --profile preview
   eas build -p android --profile preview
   ```

8. Submit to TestFlight / internal Play track first; then production.

### 5.6 L3 — operations and Docker

Do these once you are confident the apps look right; they do not affect
end-user experience but matter for your ops team.

```diff
- name: atlas-cmms
+ name: polfilm-cmms
- container_name: atlas_db
+ container_name: polfilm_db
- container_name: atlas-cmms-backend
+ container_name: polfilm-cmms-backend
- container_name: atlas-cmms-frontend
+ container_name: polfilm-cmms-frontend
- container_name: atlas_minio
+ container_name: polfilm_minio
- MINIO_BUCKET: atlas-bucket
+ MINIO_BUCKET: polfilm-bucket
```

If you publish your own images instead of pulling `intelloop/atlas-cmms-*`:

```diff
-  image: intelloop/atlas-cmms-backend
+  image: <your-registry>/polfilm-cmms-backend
-  image: intelloop/atlas-cmms-frontend
+  image: <your-registry>/polfilm-cmms-frontend
```

Then update `.github/workflows/push-docker.yml` to push under your own
image names.

### 5.7 Documentation

Don't forget end-user docs, screenshots, and internal runbooks. Internal
documentation under `dev-docs/` mentions "Atlas" in many places. Cosmetic
but important for new operators joining Polfilm.

---

## 6. Complexity assessment

The accurate way to characterize this work is by *surface count* and
*coupling*, not days. For each level:

### 6.1 L1 (runtime white-label)

- **Surfaces touched.** 1 service (`BrandingService`), 1 hook
  (`useBrand`), 1 server util (`serverBrand`), runtime env, two PNGs.
- **Coupling.** Self-contained; no breaking changes to data, schema, or
  API.
- **Risks.** License gate — see §3. Otherwise minimal.
- **Difficulty.** Low. Operator-grade work, not engineering.

### 6.2 L2 (full web rebrand)

- **Surfaces touched.** ~25 source files plus i18n bundles plus
  PWA/favicon assets.
- **Coupling.** Almost none. Each file is independent.
- **Risks.**
  - Forgetting one surface → mixed branding ("Polfilm" in app, "Atlas" in
    PWA install prompt). The checklist in §9 exists exactly to catch
    this.
  - `home/` is the largest workstream by file count if you keep it; the
    safest move is to *not deploy it* for an internal Polfilm rollout.
  - Legal pages (`privacy`, `terms-of-service`, `deletion-policy`) need
    real legal review, not just s/Atlas/Polfilm/.
- **Difficulty.** Medium. Mechanical edits + a careful checklist pass.

### 6.3 L3 (full ecosystem)

- **Surfaces touched.** Everything in L2 plus:
  - mobile app identity at the *native* level (iOS + Android),
  - app store / EAS / signing-key plumbing,
  - Docker compose, image names, CI workflow,
  - operational docs.
- **Coupling.** High between mobile config and store/signing
  infrastructure. Coupling inside the codebase remains low (Expo
  prebuild does the heavy lifting).
- **Risks.**
  - **Mobile is a separate app, not a renamed app.** A rebrand of bundle
    id / package id means you publish a *new* application that existing
    users must install. Plan migration (in-app banner, FAQ, support
    notice).
  - **Push notification credentials** must be re-issued under the new
    bundle id. Old `google-services.json` will not work.
  - **Deep links break.** Anything outside the codebase that links to
    `atlascmms://` will not open Polfilm CMMS. Audit OAuth redirect URLs,
    QR codes, and email templates.
  - **App-store review** is real elapsed time you do not control; Apple
    in particular tends to scrutinize CMMS-style apps for their data and
    push permissions.
- **Difficulty.** High; not because each file is hard but because the
  store / signing / push surface is unforgiving and outside your repo.

### 6.4 Compared to the work already done on this branch

For calibration:

| Workstream | Approximate file count | Cross-cutting? | Practical difficulty |
| --- | --- | --- | --- |
| Removing the meter trigger license check (this branch) | 1 | No | Trivial |
| Persian language localization | ~3–5 source files + 1 large translation file | Some | Medium |
| Persian/Jalali calendar support | many — date utilities, pickers, DTOs, mobile | High | High |
| **L1 Polfilm rebrand** | 0 source / 2 PNGs + env | No | Low |
| **L2 Polfilm rebrand** | ~25 source + many i18n strings | Low | Medium |
| **L3 Polfilm rebrand** | ~100+ source + native mobile + store ops | Medium-high | High |

So: a configuration-only Polfilm rebrand is *easier* than persian
localization. A complete L3 rebrand (especially the mobile/native portion)
is *more* work than persian localization but probably *less* than full
Jalali calendar support — *in pure code*. The store/signing operational
overhead is what tips L3 into "high" effort.

---

## 7. Search-and-replace recipes

These are deliberately conservative — review each match before committing.
None of them are blanket-safe across the entire repo.

### 7.1 Find all current occurrences

```bash
# all files mentioning the word in any case
rg -ni "\batlas\b" --stats

# only user-visible strings (rough heuristic)
rg -n "Atlas CMMS|Atlas " --type ts --type tsx --type java --type html --type json
```

### 7.2 Mechanical replacement (review!)

```bash
# DO NOT run this without reading the diff first.
# Skip Java packages and the Postgres schema — see §10.

rg -l "Atlas CMMS|Atlas " \
    --glob '!**/com/grash/**' \
    --glob '!**/grash/**' \
    --glob '!**/.git/**' \
    --glob '!**/node_modules/**' \
    --glob '!**/build/**' \
  | xargs sed -i.bak 's/Atlas CMMS/Polfilm CMMS/g; s/\bAtlas\b/Polfilm/g'
```

After it runs:

```bash
# review every change
git diff
# remove backup files once happy
find . -name '*.bak' -delete
```

### 7.3 URLs

```bash
sed -i.bak 's@https://www.atlas-cmms.com@https://cmms.polfilm.example@g; \
            s@https://atlas-cmms.com@https://cmms.polfilm.example@g; \
            s@https://api.atlas-cmms.com@https://api-cmms.polfilm.example@g' \
    $(rg -l 'atlas-cmms.com')
```

### 7.4 Mobile bundle ids (only after `expo prebuild --clean`)

You should *not* need this if you regenerate native. Listed for completeness:

```bash
# DO NOT use unless you are not running expo prebuild --clean
sed -i.bak 's/com\.atlas\.cmms/com.polfilm.cmms/g; s/com\.cmms\.atlas/com.polfilm.cmms/g' \
    $(rg -l 'com\.atlas\.cmms\|com\.cmms\.atlas' mobile)
```

---

## 8. Things to **not** rebrand

The instinct is to rename "everything that says Atlas". Resist for these:

- **Java package names `com.grash.*`** — internal-only; renaming requires
  refactoring the entire backend; offers no user-visible benefit. Spring
  Boot scans, JPA repositories, mappers, and many configurations all key
  off this name. Leave it alone.
- **Postgres database name `atlas` / schema-style references**
  (`POSTGRES_DB: atlas`) — visible only to operators with DB access. You
  *can* rename it, but it requires a fresh database or a migration. For a
  fresh deploy, change it once. For an existing deploy, leave it.
- **Existing on-disk MinIO bucket name `atlas-bucket`** — same story.
  Easy to rename for a new deploy; painful (and offline-time-consuming)
  to rename in place.
- **Existing iOS/Android apps that users have already installed** — you
  cannot rebrand them in place; you publish a new app under the new
  bundle id and run a migration period.
- **Keygen license keys, Paddle price ids, Expo project id** — these are
  vendor-side identifiers. Issue new ones; do not rewrite the strings in
  source without re-issuing on the vendor.

---

## 9. Verification checklist

Run these surfaces after each level. If any one of them still says
"Atlas", you have an incomplete rebrand.

### 9.1 L1 verification

- [ ] `curl /branding/config` returns Polfilm values.
- [ ] App header (logged-in dashboard) shows Polfilm logo.
- [ ] Email after sign-up shows Polfilm logo + Polfilm primary color in
      header.
- [ ] Work-order PDF report shows Polfilm logo and contact details.

### 9.2 L2 verification

L1 plus:

- [ ] Browser tab title (logged out) says "Polfilm CMMS".
- [ ] Login / register pages: no "Atlas" anywhere.
- [ ] PWA install prompt shows "Polfilm" / "Polfilm CMMS".
- [ ] Favicon is Polfilm in all browsers.
- [ ] No-JavaScript fallback says "Polfilm CMMS".
- [ ] Open Graph preview (paste your URL into Slack/Teams) shows Polfilm.
- [ ] Swagger / OpenAPI page (`/swagger-ui`) shows "Polfilm CMMS API".
- [ ] Webhook docs (in Swagger) say Polfilm.
- [ ] Self-hosted plan strings (if shown anywhere in the UI) say
      Polfilm.
- [ ] All 14 i18n locales — at least spot-check English, Polish (this is
      the locale Polfilm users are most likely to use given the brand
      name), and one RTL (`ar.ts`) for layout regressions.

### 9.3 L3 verification

L2 plus:

- [ ] Mobile app icon, splash, name on home screen.
- [ ] Push notifications arrive under "Polfilm CMMS".
- [ ] Deep link (e.g. password reset email) opens the new mobile app.
- [ ] App store / Play Store / TestFlight / internal track listing
      reviewed and approved.
- [ ] Docker compose project + container names use `polfilm-*`.
- [ ] CI publishes images under your registry, not under
      `intelloop/atlas-cmms-*`.
- [ ] Backups land in a Polfilm-named folder/bucket.

---

## 10. Risks and mitigations specific to Polfilm

### 10.1 The license gate (most common foot-gun)

You set `BRAND_CONFIG`, restart, log in, and still see "Atlas CMMS". This is
almost always the `BRANDING` license entitlement being missing. Verify:

```bash
curl http://localhost:8080/license/state
```

If `valid` is false or `entitlements` does not include `BRANDING`, see §3.

### 10.2 12-hour cache staleness

`LicenseService` caches license state for 12 hours. After you set / change a
license key, the API may still report the old state for up to 12 hours. To
force a refresh, restart the API container:

```bash
docker compose restart api
```

This is documented in detail in
[Meter trigger licensing mechanism](./Meter%20trigger%20licensing%20mechanism.md).

### 10.3 Mixed branding

The single biggest UX failure mode. The checklist in §9 is the cure.
Specifically watch for:

- "Atlas CMMS" in the browser tab while the dashboard says "Polfilm
  CMMS" (you forgot `frontend/public/index.html`).
- Polfilm logo in the app but Atlas logo in the password-reset email
  (you forgot `BrandingService.getBrandConfig()` defaults, and you don't
  have `BRANDING` license).
- Polfilm web app but Atlas-branded mobile app on the user's phone (you
  did L2 but not L3 — perfectly valid choice, just be explicit about it
  in user comms).

### 10.4 Polfilm-specific terminology

If Polfilm is in film/foil/extrusion manufacturing, the upstream CMMS
terminology may not match how Polfilm operators actually talk:

| Upstream term | Possible Polfilm term |
| --- | --- |
| Asset | Line / extruder / winder |
| Location | Hall / bay |
| Work order | Maintenance job / nakaz pracy |
| Preventive maintenance | Planned maintenance / przegląd |
| Request | Service request / zgłoszenie |

Terminology changes are *localization* work, not branding. They live in
`frontend/src/i18n/translations/pl.ts` and `mobile/i18n/translations/pl.ts`,
not in `BrandingService`. Treat them as a follow-up phase after the visual
rebrand is done; mixing them increases the chance of regressions.

### 10.5 Domain and email

`SMTP_FROM` is what Polfilm users see as the email "From" address. After
the rebrand, set it to a Polfilm-owned mailbox; otherwise emails will say
"Polfilm CMMS" but come from a non-Polfilm domain, which is a deliverability
and trust issue.

```env
SMTP_FROM="Polfilm CMMS <cmms-no-reply@polfilm.example>"
```

### 10.6 Legal pages

The upstream `home/app/[locale]/{privacy,terms-of-service,deletion-policy}`
pages are written for Atlas's legal entity. Do **not** ship them under the
Polfilm name without legal review. Either replace with Polfilm's own legal
copy or do not deploy `home/` at all.

---

## 11. Recommended path for Polfilm

For a typical internal Polfilm rollout, the recommended sequencing is:

1. **L1 first.** Get Polfilm name + logo + colors live with env vars and
   confirm `BRANDING` license behavior. This is hours of work, not days.
2. **L2 web only.** Skip `home/` entirely (don't deploy a marketing site
   at all). Replace static metadata, favicons, PWA assets, OpenAPI titles,
   and the i18n bundles. After this, every web surface a Polfilm employee
   sees says Polfilm.
3. **Decide on mobile.** If Polfilm employees rely on the mobile app,
   commit to L3 with Expo prebuild and a new EAS project. If mobile is
   minor, defer L3 indefinitely and tell users that the mobile app is
   "Atlas-branded but connects to Polfilm CMMS".
4. **Do operations rebrand last.** Renaming Docker projects and
   containers is almost free, but it's the most disruptive to anyone
   running `docker compose ps` from muscle memory. Do it once, document
   it, move on.
5. **Treat Polish industry terminology as a separate project.** It is
   not a rebrand; it is a localization / domain customization phase.

If you do exactly this, the rebrand is well within reach: **L1+L2 is
mechanical engineering on ~25 source files plus configuration**; **L3 is
mostly app-store / signing operational work** that the codebase itself
makes easy via `expo prebuild --clean`. The licensing layer is the only
non-obvious dependency, and §3 gives you two clean options for handling
it.

---

## 12. Related reading

- [Rebranding guide for industry deployment](./Rebranding%20guide%20for%20industry%20deployment.md)
  — the generic, concept-level guide. Use this Polfilm doc for the
  file-level details, that one for the *why*.
- [Licensing and feature gates](./Licensing%20and%20feature%20gates.md) —
  the broader picture of how `BRANDING` and other entitlements work.
- [Licensing implementation assessment](./Licensing%20implementation%20assessment.md)
  — known issues with the licensing layer.
- [Meter trigger licensing mechanism](./Meter%20trigger%20licensing%20mechanism.md)
  / [Meter trigger license removal walkthrough](./Meter%20trigger%20license%20removal%20walkthrough.md)
  — exact same pattern of "default value behind a license gate" that
  applies to `BrandingService.getBrandConfig()` if you choose to remove
  the gate.
- [Docker image build guide](./Docker%20image%20build%20guide.md) —
  needed if you publish your own Polfilm-named images instead of pulling
  upstream `intelloop/atlas-cmms-*`.
