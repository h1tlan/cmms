# Licensing and Feature Gates

This document explains how licensing works in this project, why some features
are unavailable after a normal `docker compose up -d`, and why creating a custom
role can show this backend error:

```text
You need a license to create custom roles
```

This is a technical analysis of the repository. It is not legal advice.

## Short answer

Atlas CMMS is free/open source in the legal sense because the repository is
available under AGPLv3. However, the running application also contains runtime
feature gates for commercial/self-hosted entitlements.

That means:

- You can run the project from Docker without a license.
- You can use the free/open-source parts of the application.
- Some advanced features are disabled unless a valid commercial/self-hosted
  license grants the required entitlement.
- Custom roles are one of the gated features.

For your specific custom-role issue, the backend requires:

1. The company's subscription plan must include `PlanFeatures.ROLE`.
2. The self-hosted license must be valid and include
   `LicenseEntitlement.CUSTOM_ROLES`.

If you run with Docker Compose and leave `LICENSE_KEY` empty, license state is
invalid and `CUSTOM_ROLES` is not granted, so custom role creation fails.

## Legal licensing model

The root license is:

```text
LICENSE
```

It contains the GNU Affero General Public License v3. AGPLv3 is an open-source
license. In simple terms, it allows running, studying, modifying, and sharing
the software, subject to the AGPL terms. For network software, AGPL has source
code disclosure obligations when modified versions are offered over a network.

The repository also has:

```text
COMMERCIAL_LICENSE.MD
```

That file describes a dual-license model:

- AGPLv3 option.
- Paid commercial license option.

The commercial license explicitly references enterprise/commercial features.
The root `README.MD` also says:

```text
Atlas CMMS is dual-licensed:
- AGPLv3 License — Free and open source.
- Commercial License — Required for white labeling, custom branding, and advanced features.
```

So "free" and "licensed features" both exist in this project:

- Free/open source describes the legal availability of the source code.
- Runtime licensing controls access to selected product features.

## Docker Compose behavior

The root `docker-compose.yml` runs prebuilt images:

```yaml
api:
  image: intelloop/atlas-cmms-backend

frontend:
  image: intelloop/atlas-cmms-frontend
```

The API container receives these license-related environment variables:

```yaml
LICENSE_KEY: ${LICENSE_KEY:-}
LICENSE_FINGERPRINT_REQUIRED: ${LICENSE_FINGERPRINT_REQUIRED:-true}
LICENSE_FILE_PATH: ${LICENSE_FILE_PATH:-}
KEYGEN_PRODUCT_TOKEN: ${KEYGEN_PRODUCT_TOKEN:-}
```

By default, `LICENSE_KEY` and `LICENSE_FILE_PATH` are empty.

That is why a normal local start:

```bash
docker compose up -d
```

starts the application, but does not grant commercial entitlements.

## Backend license configuration

Backend configuration is in:

```text
api/src/main/resources/application.yml
```

Important properties:

```yaml
license-key: ${LICENSE_KEY:}
license-fingerprint-required: ${LICENSE_FINGERPRINT_REQUIRED:true}
license-file-path: ${LICENSE_FILE_PATH:}

keygen:
  product-token: ${KEYGEN_PRODUCT_TOKEN:}
  account-id: 1ca3e517-f3d8-473f-a45c-81069900acb7
```

The backend license service is:

```text
api/src/main/java/com/grash/service/LicenseService.java
```

The public license-state endpoint is:

```text
GET /license/state
```

Controller:

```text
api/src/main/java/com/grash/controller/LicenseController.java
```

Response DTO:

```text
api/src/main/java/com/grash/dto/license/LicensingState.java
```

The endpoint returns fields like:

- `hasLicense`
- `valid`
- `planName`
- `entitlements`
- `expirationDate`
- `usersCount`

You can inspect the local state after Docker starts:

```bash
curl http://localhost:8080/license/state
```

Without a license, expect a response conceptually like:

```json
{
  "hasLicense": false,
  "valid": false,
  "entitlements": []
}
```

Exact fields may vary depending on serialization and version.

## How license validation works

`LicenseService.getLicensingState()` uses this order:

1. If cached license state is still fresh, return it.
2. If there is no license key and no license file, return invalid state.
3. If a license file exists, validate and decrypt the license file.
4. Otherwise, validate `LICENSE_KEY` with Keygen.
5. If the license is valid, fetch entitlements.

Important behavior:

```java
if (!hasLicenseKey() && !hasLicenseFile()) {
    return clearCacheAndReturnInvalid();
}
```

Entitlement checks use:

```java
public boolean hasEntitlement(LicenseEntitlement entitlement) {
    LicensingState state = getLicensingState();
    return state.isValid() && state.getEntitlements().contains(entitlement.toString());
}
```

So a feature entitlement only passes when:

- the license is valid
- and the entitlement string is present

## License entitlements

The backend entitlement enum is:

```text
api/src/main/java/com/grash/dto/license/LicenseEntitlement.java
```

Current entitlements include:

- `SSO`
- `WORK_ORDER_HISTORY`
- `WORKFLOW`
- `MULTI_INSTANCE`
- `WEBHOOK`
- `BRANDING`
- `NFC_BARCODE`
- `CUSTOM_ROLES`
- `FILE_ATTACHMENTS`
- `TIME_TRACKING`
- `COST_TRACKING`
- `WORK_ORDER_LINKING`
- `SIGNATURE_CAPTURE`
- `PM_CALENDAR`
- `CONDITION_BASED_PM`
- `ASSET_HIERARCHY`
- `ASSET_DOWNTIME`
- `LOW_STOCK_ALERTS`
- `PARTS_COST_TRACKING`
- `CUSTOMER_VENDOR`
- `FIELD_CONFIGURATION`
- `VOICE_NOTES`
- `ADVANCED_ANALYTICS`
- `API_ACCESS`
- `UNLIMITED_ASSETS`
- `UNLIMITED_LOCATIONS`
- `UNLIMITED_PARTS`
- `UNLIMITED_PM_SCHEDULES`
- `UNLIMITED_ACTIVE_WORK_ORDERS`
- `UNLIMITED_CHECKLISTS`
- `UNLIMITED_METERS`
- `UNLIMITED_USERS`
- `REQUEST_PORTAL`

These are self-hosted/commercial license entitlements. They are different from
the in-app subscription feature enum described below.

## Subscription plan features

The backend also has subscription-plan features:

```text
api/src/main/java/com/grash/model/enums/PlanFeatures.java
```

Current values include:

- `PREVENTIVE_MAINTENANCE`
- `CHECKLIST`
- `FILE`
- `PURCHASE_ORDER`
- `METER`
- `REQUEST_CONFIGURATION`
- `ADDITIONAL_TIME`
- `ADDITIONAL_COST`
- `ANALYTICS`
- `REQUEST_PORTAL`
- `SIGNATURE`
- `ROLE`
- `WORKFLOW`
- `API_ACCESS`
- `WEBHOOK`
- `IMPORT_CSV`

These are attached to database subscription plans such as `FREE`, `STARTER`,
`PROFESSIONAL`, and `BUSINESS`.

The default plans are seeded in:

```text
api/src/main/java/com/grash/ApiApplication.java
```

That startup code creates plans if they do not exist.

Important distinction:

- `PlanFeatures` controls what the company subscription plan allows.
- `LicenseEntitlement` controls what the self-hosted license allows.

Some features check only one of these. Some features check both.

## Why custom roles need a license

Custom role creation is gated in two backend places.

### 1. RoleController checks subscription plan feature

File:

```text
api/src/main/java/com/grash/controller/RoleController.java
```

Create endpoint:

```java
@PostMapping("")
@PreAuthorize("hasRole('ROLE_CLIENT')")
Role create(@Valid @RequestBody Role roleReq, HttpServletRequest req) {
    User user = userService.whoami(req);
    roleReq.setPaid(true);
    if (user.getRole().getViewPermissions().contains(PermissionEntity.SETTINGS)
            && user.getCompany().getSubscription().getSubscriptionPlan().getFeatures().contains(PlanFeatures.ROLE)) {
        return roleService.create(roleReq);
    } else throw new CustomException("Access denied", HttpStatus.FORBIDDEN);
}
```

This means the user's company subscription plan must include:

```text
PlanFeatures.ROLE
```

If not, the API returns:

```text
Access denied
```

### 2. RoleService checks commercial license entitlement

File:

```text
api/src/main/java/com/grash/service/RoleService.java
```

Create logic:

```java
public Role create(Role role) {
    if (role.getCode().equals(RoleCode.USER_CREATED)
            && !licenseService.hasEntitlement(LicenseEntitlement.CUSTOM_ROLES))
        throw new CustomException("You need a license to create custom roles", HttpStatus.FORBIDDEN);
    return roleRepository.save(role);
}
```

This is the exact message you saw.

It means:

- The role being created is a user-created/custom role.
- The license state is either invalid or missing `CUSTOM_ROLES`.

So even if the UI lets you open the custom-role form, the backend can still
reject the request.

## Frontend behavior for custom roles

The frontend role-management page checks subscription features, not the
self-hosted license entitlement.

Relevant file:

```text
frontend/src/content/own/Settings/Roles/PageHeader.tsx
```

It uses:

```tsx
disabled={!hasFeature(PlanFeature.ROLE)}
```

The tooltip uses:

```tsx
hasFeature(PlanFeature.ROLE) ? t('create_role') : t('upgrade_role')
```

That means the frontend button is mainly gated by:

```text
PlanFeature.ROLE
```

The frontend does have license-state support:

```text
frontend/src/slices/license.ts
frontend/src/hooks/useLicenseEntitlement.ts
```

But the role creation UI does not currently appear to check:

```text
CUSTOM_ROLES
```

This creates a confusing UX:

1. The subscription plan may allow roles.
2. The UI may enable the create button.
3. The backend may reject custom role creation because the self-hosted license
   lacks `CUSTOM_ROLES`.

## How to verify your current licensing state

After starting Docker:

```bash
docker compose up -d
```

Check the backend state:

```bash
curl http://localhost:8080/license/state
```

Look for:

```json
"valid": true
```

and:

```json
"entitlements": ["CUSTOM_ROLES"]
```

For custom roles, the important entitlement is:

```text
CUSTOM_ROLES
```

If `valid` is false or `CUSTOM_ROLES` is absent, the backend will reject
user-created role creation.

## How to provide a license in Docker

The documented Docker path is to set `LICENSE_KEY` in `.env`:

```env
LICENSE_KEY=your_license_key_here
```

Then restart the API container:

```bash
docker compose up -d
```

or:

```bash
docker compose restart api
```

If using a license file, the backend also supports:

```env
LICENSE_FILE_PATH=/path/inside/container/to/license-file
LICENSE_KEY=your_license_key_here
```

Important: license file validation still requires a license key for decryption
according to `LicenseService`.

The root `docker-compose.yml` mounts:

```yaml
./config:/app/static/config
```

So a practical license-file path, if the product supports placing your file
there, would need to point to a path inside the container such as:

```env
LICENSE_FILE_PATH=/app/static/config/license-file-name
```

Use the vendor's actual license file instructions if provided.

## Why the project can be open source but still have gated features

This repository combines two concepts:

1. Source-code license:
   - AGPLv3 makes the code free/open source under AGPL terms.
2. Product licensing:
   - The application enforces commercial entitlements for selected features.

That is why the README can say the project is open source while the app says a
license is needed for advanced features.

This is not unusual in dual-license/self-hosted products, but it can be
confusing because "free" may mean:

- free as in open-source code availability
- free as in no-cost product tier
- free as in every feature unlocked

In this project, those are not the same.

## Feature-gate examples

The backend uses `licenseService.hasEntitlement(...)` in many places.

Examples:

| Area | Backend file | Entitlement |
| --- | --- | --- |
| Custom roles | `RoleService.java` | `CUSTOM_ROLES` |
| File uploads/attachments | `FileController.java` | `FILE_ATTACHMENTS` |
| Custom fields | `FieldConfigurationController.java` | `FIELD_CONFIGURATION` |
| NFC/barcode | `AssetController.java` | `NFC_BARCODE` |
| Work order signature | `WorkOrderController.java` | `SIGNATURE_CAPTURE` |
| API keys/API access | `ApiKeyService.java`, `ApiKeyAuthFilter.java` | `API_ACCESS` |
| SSO/LDAP | `LdapSecurityConfig.java`, `LicenseService.java` | `SSO` |
| Request portal | `RequestPortalService.java` | `REQUEST_PORTAL` |
| Webhooks | `WebhookEndpointService.java`, `WebhookDispatchService.java` | `WEBHOOK` |
| Work order history | `WorkOrderHistoryService.java` | `WORK_ORDER_HISTORY` |
| Workflows | `WorkflowController.java` | `WORKFLOW` |

There are also usage-limit gates. Defaults are defined in:

```text
api/src/main/java/com/grash/utils/Consts.java
```

Example default limits when unlimited entitlements are absent:

- users: 5
- assets: 50
- parts: 100
- locations: 10
- preventive maintenance schedules: 10
- active work orders: 30
- meters: 10
- checklist templates: 10

These are tied to entitlements such as:

- `UNLIMITED_USERS`
- `UNLIMITED_ASSETS`
- `UNLIMITED_ACTIVE_WORK_ORDERS`

## Frontend license-state model

The frontend fetches license state from:

```text
GET /license/state
```

Redux slice:

```text
frontend/src/slices/license.ts
```

Hook:

```text
frontend/src/hooks/useLicenseEntitlement.ts
```

The hook returns true only when:

```ts
license.valid && license.entitlements.some((e) => e === entitlement)
```

This is the correct general model, but not every feature screen uses it. Some
screens only check `PlanFeature`.

## Inconsistencies and maintainability concerns

The licensing system is functional, but it is not as easy to understand as it
could be.

### 1. Two kinds of gates

There are two separate enums:

- `PlanFeatures`
- `LicenseEntitlement`

Some features need both. Some need only one. This should be documented clearly
in code or centralized in a policy service.

### 2. Frontend and backend can disagree

Custom roles are the clearest example:

- frontend checks `PlanFeature.ROLE`
- backend checks `PlanFeatures.ROLE` and `CUSTOM_ROLES`

This can produce a button that looks available but then fails at submit time.

### 3. Entitlement names drift between backend and frontend

Backend has:

```text
UNLIMITED_CHECKLISTS
REQUEST_PORTAL
UNLIMITED_USERS
```

The frontend `licenseEntitlements` list currently includes:

```text
UNLIMITED_CHECKLIST
```

and does not list every backend entitlement. TypeScript typing does not prevent
the API from returning additional strings, but mismatches make UI gating harder
to trust.

### 4. Error messages are not product-friendly

The backend error:

```text
You need a license to create custom roles
```

is accurate, but it does not explain:

- whether the current license is missing
- whether it is invalid
- whether it lacks `CUSTOM_ROLES`
- whether the company plan lacks `ROLE`

A better UI could show both plan and license status.

## Recommended improvements

### Product/documentation improvements

1. Add a clear "Free vs licensed features" section to the main README.
2. Document which features require a commercial/self-hosted license.
3. Document which features are limited in the free tier and their limits.
4. Explain that Docker Compose starts without a license but advanced features
   remain locked.
5. Document `GET /license/state` as the first debugging step.

### Backend improvements

1. Centralize feature access policy in one service.
2. Make each feature declare whether it requires:
   - plan feature
   - license entitlement
   - both
3. Return structured errors, for example:

```json
{
  "code": "LICENSE_ENTITLEMENT_REQUIRED",
  "requiredEntitlement": "CUSTOM_ROLES",
  "message": "Custom roles require a valid license with CUSTOM_ROLES entitlement."
}
```

4. Add tests for custom-role gating:
   - no plan feature
   - plan feature but no license
   - valid license but no entitlement
   - valid license with entitlement
5. Add tests for `/license/state`.

### Frontend improvements

1. Update the roles UI to check both:
   - `hasFeature(PlanFeature.ROLE)`
   - `useLicenseEntitlement('CUSTOM_ROLES')`
2. Show a specific message when license entitlement is missing.
3. Add a license status panel or admin diagnostic page.
4. Keep frontend entitlement names in sync with backend enum.
5. Add tests for feature-gated buttons.

## Practical answer for your local Docker run

If you are running the default Docker Compose stack without a license key:

```env
LICENSE_KEY=
```

then custom roles are expected to fail.

To create custom roles without changing code, you need a valid license that
returns:

```text
CUSTOM_ROLES
```

from:

```text
GET /license/state
```

If you are evaluating or developing the open-source project and do not want to
use commercial features, avoid custom roles and use the built-in/default roles.

If your goal is to modify the product behavior for your own AGPL-compliant fork,
that becomes a legal/product decision outside this technical guide. Review AGPL
obligations and the commercial license restrictions before removing or changing
license checks.

## Bottom line

The project is free/open source under AGPLv3, but the shipped application has
commercial feature gates. The custom-role message you saw is intentional
runtime enforcement:

```text
RoleService.create -> LicenseEntitlement.CUSTOM_ROLES
```

The most important mental model is:

```text
AGPL source availability != every product feature unlocked
```

For custom roles specifically:

```text
PlanFeatures.ROLE + valid license with CUSTOM_ROLES = allowed
```

Without a valid license entitlement, Docker Compose will run, but custom role
creation will be blocked.
