# Licensing Implementation Assessment

This document evaluates the licensing implementation as a reusable engineering
pattern. It answers:

1. Is this implementation reliable?
2. Is it good practice?
3. Can you reuse this pattern in another open-source project?

This is a technical architecture review, not legal advice. For legal decisions
about AGPL, dual licensing, commercial licensing, or removing license checks,
consult a qualified lawyer.

## Short answer

The pattern is good and common, but this implementation should be hardened before
you copy it into another project.

The project uses a recognizable model:

```text
dual license text
  -> runtime license validation
  -> server-side entitlement checks
  -> frontend/mobile UI hints
```

That is a sound pattern for an open-source product with paid self-hosted or
enterprise features.

However, this implementation has gaps:

- no automated licensing tests
- two different feature gate concepts: `PlanFeatures` and `LicenseEntitlement`
- frontend/mobile entitlement lists can drift from backend entitlements
- license cache policy is fixed in code
- `/license/state` is public
- machine fingerprint is logged
- offline license parsing needs more defensive validation
- entitlement-fetch failure behavior is not explicit enough

Use the pattern, but not the exact implementation as-is.

## What pattern is implemented here?

The repository combines:

- legal licensing:
  - `LICENSE`: AGPLv3
  - `COMMERCIAL_LICENSE.MD`: paid commercial self-hosted license
- runtime licensing:
  - `LicenseService`
  - Keygen online validation
  - optional encrypted offline license file
  - entitlement checks in backend services/controllers
- client UX:
  - web/mobile fetch `GET /license/state`
  - UI optionally hides or disables gated features

The most important design choice is correct:

> The backend is the enforcement boundary. The frontend only improves UX.

That means users cannot unlock server-side features just by changing frontend
code. They would need to change the backend or provide a valid license.

## Is this good practice for open-source projects?

Yes, with important qualifications.

For open-source commercial products, this is a common approach:

1. Release the source under a copyleft or source-available license.
2. Offer a commercial license for companies that do not want open-source
   obligations or need commercial terms.
3. Keep some enterprise/self-hosted features behind runtime entitlements.
4. Enforce those entitlements on the server.
5. Use frontend checks only for display and product messaging.

That model works especially well for:

- self-hosted B2B products
- admin tools
- infrastructure dashboards
- workflow/business apps
- products where enterprise buyers need support, white-labeling, SSO, audit,
  or high usage limits

It works less well for:

- libraries where runtime feature gates feel hostile
- purely community-driven projects
- apps where most users expect every feature to be free
- products where users can trivially remove backend checks and still receive all
  commercial value without support or updates

## What is good in this implementation?

### 1. Server-side enforcement

The backend calls:

```java
licenseService.hasEntitlement(...)
```

from controllers, services, and filters.

Examples:

- custom roles: `RoleService`
- API access: `ApiKeyService`, `ApiKeyAuthFilter`
- files: `FileController`
- field configuration: `FieldConfigurationController`
- request portal: `RequestPortalService`
- workflows: `WorkflowController`
- SSO/LDAP: `LdapSecurityConfig`

This is the right trust boundary. A frontend-only license check is not enough.

### 2. Clear runtime license state

The backend exposes:

```text
GET /license/state
```

with:

- `hasLicense`
- `valid`
- `planName`
- `entitlements`
- `expirationDate`
- `usersCount`

This is useful for admin UI, diagnostics, and support.

### 3. Online and offline validation paths

The implementation supports:

1. Online validation through Keygen.
2. Offline license file validation through encrypted/signed license files.

That is a strong design for self-hosted software because some customers are
air-gapped or have restricted outbound network access.

### 4. Entitlements rather than one global paid flag

The backend checks specific entitlements such as:

```text
CUSTOM_ROLES
BRANDING
API_ACCESS
REQUEST_PORTAL
UNLIMITED_USERS
```

This is better than a single `isPaid` boolean because plans can evolve.

### 5. License provisioning is separated from runtime checks

Runtime checks happen in:

```text
api/src/main/java/com/grash/service/LicenseService.java
```

Provisioning/admin calls to Keygen happen in:

```text
api/src/main/java/com/grash/service/KeygenService.java
```

That separation is healthy. The product token used for creating/updating
licenses should stay server-side only.

## Reliability assessment

### Cache behavior

`LicenseService` caches license state for 12 hours:

```java
private static final long CACHE_DURATION_MILLIS = 12 * 60 * 60 * 1000;
```

This improves reliability when Keygen is slow or unavailable, but it creates a
tradeoff:

- a revoked license may keep working until cache expires
- an upgraded license may not unlock features until cache refreshes
- operational behavior is hardcoded, not configurable

This is acceptable for many self-hosted products, but it should be a deliberate
product decision.

Recommended improvement:

```text
LICENSE_CACHE_TTL_MINUTES=720
LICENSE_STALE_GRACE_MINUTES=...
```

Document whether the product prefers:

- fail closed: no validation means no features
- fail open with grace: keep last known entitlements temporarily
- fail mixed: valid license state but no new entitlement changes

### Entitlement fetch failure

Online validation does two calls:

1. validate license key
2. fetch entitlements

If the license validates but entitlement fetching fails, the current code clears
entitlements. That is safe from an over-permission perspective, but confusing:

```text
license valid = true
entitlements = []
```

Recommended improvement:

- define an explicit state such as `ENTITLEMENTS_UNAVAILABLE`
- return `lastCheckedAt` and `lastError`
- optionally keep last known entitlements for a short grace period

### Offline license validation

Offline files are verified and decrypted. This is good.

But reliability gaps exist:

- no maximum license file size
- plaintext conversion uses platform default charset in `new String(output)`
- malformed decrypted payloads can cause runtime exceptions later

Recommended improvements:

- enforce a small max license file size
- use `StandardCharsets.UTF_8`
- validate required payload fields before caching state

## Security assessment

### Public `/license/state`

`/license/state` is public in `WebSecurityConfig`.

That means unauthenticated callers can see license state details, depending on
deployment:

- valid or invalid license
- plan name
- expiration date
- entitlements
- user count

This may be acceptable for a simple support/debug endpoint, but it is an
information leak.

Recommended options:

1. require authentication for full license state
2. expose only minimal anonymous state
3. separate endpoints:

```text
GET /license/public-state
GET /license/admin-state
```

### Fingerprint logging

When fingerprint binding is enabled, `LicenseService` logs:

```java
log.info("X-Machine-Fingerprint: {}", fingerprint);
```

This fingerprint is derived from MAC addresses. Even though it is hashed, it is
still deployment-identifying information.

Recommended improvement:

- log it only at debug level
- or log only a short prefix
- or avoid logging it entirely

### Public key configuration

The offline license public key is hardcoded:

```java
private static final String KEYGEN_PUBLIC_KEY = "...";
```

Hardcoding is simple, but rotation requires code changes.

Recommended improvement:

```text
KEYGEN_PUBLIC_KEY=...
```

Keep a safe default if this is a single-vendor product, but allow override for
forks, staging, or key rotation.

### Source visibility, runtime images, and bypass resistance

The licensing implementation is visible in this repository.

Examples:

- `api/src/main/java/com/grash/service/LicenseService.java`
- `api/src/main/java/com/grash/utils/LicenseFileValidator.java`
- `api/src/main/java/com/grash/service/KeygenService.java`
- `api/src/main/java/com/grash/service/RoleService.java`
- `api/src/main/java/com/grash/dto/license/LicenseEntitlement.java`

For example, custom role creation checks `LicenseEntitlement.CUSTOM_ROLES` in
`RoleService`.

So, if someone builds their own fork from this source, they can technically
modify or remove those checks. That statement is about the public source code in
this repository.

However, that is not exactly the same as saying the official Docker runtime has
no closed or vendor-controlled parts. The root `docker-compose.yml` runs prebuilt
images:

```yaml
api:
  image: intelloop/atlas-cmms-backend

frontend:
  image: intelloop/atlas-cmms-frontend
```

Those images are artifacts produced by the vendor. This repository strongly
suggests the licensing code is part of the public backend source, and the
commercial license says the source code is publicly available under AGPLv3. But
from the repository alone, you should not blindly assume every official image is
bit-for-bit reproducible from the public source unless you verify the image
build process.

There are three different cases:

| Case | What it means |
| --- | --- |
| Build from this repository source | Licensing checks are visible and can be changed in a fork, subject to legal obligations. |
| Run official Docker images | Runtime behavior depends on what is inside those images; likely aligned with source, but should be verified if this matters. |
| Vendor provides extra non-public activation material | License keys, activation mechanisms, private docs, or unreleased features may be non-public even if main source is public. |

The commercial license itself acknowledges this split:

- source code is publicly available under AGPLv3
- non-public information may include license keys, activation mechanisms,
  private documentation, and unreleased features

If you need high confidence that the runtime exactly matches the public source,
verify one of these:

1. The project publishes Dockerfiles and CI build instructions for the images.
2. The image labels or SBOM point to a commit SHA.
3. You can build the image locally from this repository and reproduce behavior.
4. The vendor documents which parts, if any, are not in the public repository.

For an open-source commercial product, protection should not depend only on
source secrecy. In practice, protection comes from:

- legal license terms
- commercial support
- update access
- hosted services
- customer trust
- compliance requirements
- official license issuance and activation infrastructure

Do not rely on frontend checks or hidden code alone. Rely on backend enforcement,
clear contracts, and a valuable commercial relationship.

## Maintainability assessment

### Two gate systems

The project has:

```text
PlanFeatures
LicenseEntitlement
```

Some capabilities require a plan feature. Some require a license entitlement.
Some require both.

Custom roles are the clearest example:

```text
PlanFeatures.ROLE + LicenseEntitlement.CUSTOM_ROLES
```

This works, but it is easy to misunderstand.

Recommended improvement:

Create one policy layer:

```java
public enum ProductCapability {
    CUSTOM_ROLES,
    API_ACCESS,
    REQUEST_PORTAL,
    WORKFLOW
}
```

Then centralize rules:

```java
FeaturePolicy.canUse(company, ProductCapability.CUSTOM_ROLES)
```

That policy can check:

- user permission
- subscription plan
- license entitlement
- usage limits
- deployment mode

Avoid scattering these rules across controllers and services.

### Entitlement type drift

The backend enum includes:

```text
UNLIMITED_CHECKLISTS
REQUEST_PORTAL
UNLIMITED_USERS
```

Frontend/mobile entitlement lists include:

```text
UNLIMITED_CHECKLIST
```

and omit some backend entitlements.

This is a real maintainability problem. UI gating can become wrong even when the
backend is correct.

Recommended improvement:

- generate TypeScript entitlement types from the backend enum
- or expose an OpenAPI schema and generate clients
- or keep a single shared contract package

### Missing tests

There are no meaningful tests around licensing.

For a licensing system, that is a major weakness.

You should test:

- valid online license
- invalid online license
- Keygen unavailable
- entitlement fetch unavailable
- valid offline file
- expired offline file
- wrong signature
- wrong algorithm
- malformed payload
- required entitlement present/missing
- usage limits
- frontend parsing of `/license/state`

## Is Keygen a good choice?

Using Keygen is reasonable.

Benefits:

- avoids building license infrastructure from scratch
- supports license validation
- supports entitlements
- supports offline license files
- integrates with provisioning workflows
- reduces legal/commercial operational work

Risks:

- vendor dependency
- network dependency for online validation
- you must understand Keygen's data model
- webhook/provisioning errors can affect customers
- account IDs, policy IDs, and entitlement codes become important operational
  configuration

For most small or medium open-source commercial products, using a service like
Keygen is better than inventing your own licensing cryptography and dashboard.

## Should you copy this implementation?

Copy the architecture, not the exact code.

### Good parts to copy

- dual-license legal model, if it fits your product
- server-side entitlement enforcement
- explicit entitlement enum
- online validation plus offline license file support
- license state endpoint for diagnostics
- frontend/mobile hooks as UX helpers only
- plan/entitlement distinction if you have both cloud plans and self-hosted
  licenses

### Parts to improve before copying

- centralize feature policy
- avoid duplicated entitlement lists
- add tests from the beginning
- make cache TTL configurable
- make public key configurable
- define stale/offline behavior clearly
- make `/license/state` authenticated or split public/admin views
- avoid logging machine fingerprints
- make offline license parsing defensive
- use UTF-8 explicitly
- document all gated features in product docs

## Recommended design for your own project

If you want to reuse this pattern, start with this architecture:

```text
Legal layer
  LICENSE
  COMMERCIAL_LICENSE.md

Runtime contract
  Entitlement enum
  LicenseState DTO
  LicenseValidator interface

Validation providers
  OnlineLicenseValidator
  OfflineLicenseFileValidator

Policy layer
  FeaturePolicy
  ProductCapability enum

Enforcement
  backend services/controllers call FeaturePolicy

Client UX
  /license/state
  useLicenseEntitlement()
  disabled buttons and upgrade messaging
```

### Example backend shape

```java
public interface LicenseValidator {
    LicenseState validate();
}
```

```java
public enum Entitlement {
    CUSTOM_ROLES,
    SSO,
    API_ACCESS
}
```

```java
public enum ProductCapability {
    CUSTOM_ROLES,
    SSO,
    API_ACCESS
}
```

```java
public final class FeaturePolicy {
    public boolean canUse(Company company, ProductCapability capability) {
        // Check plan, license, usage limits, deployment mode, etc.
    }
}
```

Then feature code should say:

```java
featurePolicy.require(company, ProductCapability.CUSTOM_ROLES);
```

instead of manually checking plan and entitlement in many places.

## Open-source community considerations

Dual licensing can work, but it should be communicated honestly.

Users may feel misled if the README says "free open source" but common features
are locked without a clear feature matrix.

Good practice:

- state clearly that the code is AGPL/open source
- state clearly that some runtime features require a commercial license
- list free features and paid features
- list usage limits
- explain what Docker Compose includes by default
- explain what a license key unlocks
- avoid hiding gates until users hit an error

For this project, the existing README mentions commercial advanced features, but
the custom-role experience shows the product docs could be clearer.

## Reliability checklist for a production-grade licensing system

Use this checklist for your own project:

### Legal and product

- [ ] License model is clear: AGPL, commercial, source-available, etc.
- [ ] Paid features are documented.
- [ ] Terms explain SaaS, redistribution, internal use, and modifications.
- [ ] README does not overpromise "free" if features are gated.

### Backend enforcement

- [ ] All paid features are enforced server-side.
- [ ] Feature policy is centralized.
- [ ] UI checks are treated as hints only.
- [ ] Usage limits are enforced in backend writes, not only UI.

### Validation

- [ ] Online validation handles provider downtime.
- [ ] Offline validation supports air-gapped customers if needed.
- [ ] Cache TTL is configurable.
- [ ] Revocation/expiry behavior is documented.
- [ ] License public keys can rotate.

### Security

- [ ] Product tokens never reach clients.
- [ ] Machine fingerprints are not logged at info level.
- [ ] License state endpoint exposes only necessary information.
- [ ] License files have size limits.
- [ ] Parsing is defensive.

### Contracts

- [ ] Backend entitlements and frontend types are generated or synced.
- [ ] License state DTO is versioned or backward-compatible.
- [ ] Error responses include structured codes.

### Tests

- [ ] Valid and invalid online license tests.
- [ ] Provider outage tests.
- [ ] Offline license file tests.
- [ ] Entitlement present/missing tests.
- [ ] Usage limit tests.
- [ ] Frontend/mobile UI gate tests.

## Overall grade

Pragmatic grade for this implementation:

```text
Pattern:           Good
Backend boundary:  Good
Keygen usage:      Good
Offline support:   Good idea, needs hardening
Security:          Acceptable, not fully hardened
Maintainability:   Mixed
Frontend sync:     Needs improvement
Testing:           Weak
```

## Final recommendation

Yes, you can use this pattern in another open-source project.

But do it like this:

1. Keep the dual-license legal model clear.
2. Enforce commercial features in the backend.
3. Use entitlements rather than one global paid flag.
4. Use a provider like Keygen unless licensing is your core competency.
5. Add offline files if self-hosted customers need them.
6. Centralize feature policy early.
7. Generate client entitlement types from backend contracts.
8. Add tests before you ship paid gates.
9. Document free vs paid behavior clearly.

Do not copy the current implementation verbatim. It is a solid first version,
but a production-grade licensing system needs stronger tests, clearer policy
boundaries, safer parsing, and better frontend/backend contract synchronization.
