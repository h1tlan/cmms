# Removing the Meter Trigger Licensing Mechanism — Practical Walkthrough

This document is a hands-on companion to
[Meter trigger licensing mechanism](./Meter%20trigger%20licensing%20mechanism.md).

That doc explains, step by step, how the license check fires and produces the
red snackbar message:

```text
You need a license to create a meter trigger
```

This doc explains the **opposite direction**: how to *remove* that licensing
mechanism so that meter triggers can be created freely on a self-hosted /
unlicensed install. It records exactly what was changed in this branch, why
each change is needed, what was deliberately *not* changed, and the concepts
you should learn from doing this practically.

This is purely for learning / development purposes on a self-hosted instance.
It is not legal advice, and on a real Atlas CMMS commercial deployment you
should buy a license that includes `CONDITION_BASED_PM` instead.

---

## 1. TL;DR — what was removed

Exactly **one** guard in **one** backend file:

```java
// api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java
if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
    throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);
```

Removing those two lines (plus the now-unused `LicenseService` field and
import) makes the `POST /work-order-meter-triggers` endpoint accept create
requests without any license, on any deployment, regardless of `LICENSE_KEY`.

Nothing else needs to change:

- The frontend already does **not** gate the *Add trigger* button by license,
  so no frontend change is required.
- Trigger evaluation when readings come in already runs on every deployment —
  it was never license-gated, only the *creation* step was.
- The `LicenseService`, the Keygen integration, and the rest of the licensing
  model still work exactly as before; only this single feature stops asking
  for entitlements.

---

## 2. Diff that was applied

`api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java`:

```diff
 import com.grash.dto.WorkOrderMeterTriggerPatchDTO;
 import com.grash.dto.WorkOrderMeterTriggerPostDTO;
 import com.grash.dto.cutomField.CustomFieldValuePostDTO;
-import com.grash.dto.license.LicenseEntitlement;
 import com.grash.exception.CustomException;
 import com.grash.mapper.WorkOrderMeterTriggerMapper;
 import com.grash.model.*;
 ...

 public class WorkOrderMeterTriggerService {
     private final WorkOrderMeterTriggerRepository workOrderMeterTriggerRepository;
     private final WorkOrderService workOrderService;
     private final WorkOrderMeterTriggerMapper workOrderMeterTriggerMapper;
     private final MeterService meterService;
     private final EntityManager em;
-    private final LicenseService licenseService;
     private final CustomFieldValueService customFieldValueService;

     @Transactional
     public WorkOrderMeterTrigger create(WorkOrderMeterTrigger workOrderMeterTrigger, Company company) {
-        if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
-            throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);
-
         if (workOrderMeterTrigger instanceof WorkOrderMeterTriggerPostDTO workOrderMeterTriggerPostDTO) {
             workOrderMeterTrigger = workOrderMeterTriggerMapper.fromPostDto(workOrderMeterTriggerPostDTO);
             ...
         }
```

The result, after the change:

```23:46:api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java
@Service
@RequiredArgsConstructor
public class WorkOrderMeterTriggerService {
    private final WorkOrderMeterTriggerRepository workOrderMeterTriggerRepository;
    private final WorkOrderService workOrderService;
    private final WorkOrderMeterTriggerMapper workOrderMeterTriggerMapper;
    private final MeterService meterService;
    private final EntityManager em;
    private final CustomFieldValueService customFieldValueService;

    @Transactional
    public WorkOrderMeterTrigger create(WorkOrderMeterTrigger workOrderMeterTrigger, Company company) {
        if (workOrderMeterTrigger instanceof WorkOrderMeterTriggerPostDTO workOrderMeterTriggerPostDTO) {
            workOrderMeterTrigger = workOrderMeterTriggerMapper.fromPostDto(workOrderMeterTriggerPostDTO);
            if (!workOrderMeterTriggerPostDTO.getCustomFields().isEmpty()) {
                setMeterTriggerCustomFields(workOrderMeterTrigger, workOrderMeterTriggerPostDTO.getCustomFields(),
                        company);
            }
        }
        WorkOrderMeterTrigger savedWorkOrderMeterTrigger =
                workOrderMeterTriggerRepository.saveAndFlush(workOrderMeterTrigger);
        em.refresh(savedWorkOrderMeterTrigger);
        return savedWorkOrderMeterTrigger;
    }
```

Three things were removed:

1. The `if (!licenseService.hasEntitlement(...)) throw ...` block inside
   `create(...)` — the actual gate.
2. The `private final LicenseService licenseService;` field on the service —
   it had no other callers in this class, so leaving it in would be dead
   weight that Lombok's `@RequiredArgsConstructor` would still inject.
3. The `import com.grash.dto.license.LicenseEntitlement;` — no longer
   referenced after the gate is gone, so the file would otherwise fail an
   unused-import lint or look misleading.

---

## 3. Why this is sufficient (and nothing else has to change)

This is the most important conceptual point in this exercise: licensing is
*orthogonal* to the rest of the meter-trigger feature. To convince yourself,
let's walk the same end-to-end flow from
[Meter trigger licensing mechanism](./Meter%20trigger%20licensing%20mechanism.md)
and see at each layer whether anything else was license-gated.

### 3.1 Frontend — already free

The button in `MeterDetails.tsx` is gated only by *permissions*, not by
license:

```266:275:frontend/src/content/own/Meters/MeterDetails.tsx
                {hasEditPermission(PermissionEntity.METERS, meter) && (
                  <Button
                    startIcon={<AddTwoToneIcon />}
                    sx={{ my: 1 }}
                    variant="outlined"
                    onClick={() => setOpenAddTriggerModal(true)}
                  >
                    {t('add_trigger')}
                  </Button>
                )}
```

The dialog (`AddTriggerModal.tsx`) and the Redux thunk
(`createWorkOrderMeterTrigger`) likewise never call
`useLicenseEntitlement('CONDITION_BASED_PM')`. Nothing on the frontend
short-circuits before the HTTP request. So once the backend stops returning
403, the user-facing flow already works — no frontend edits needed.

### 3.2 Controller — only role-gated

```54:61:api/src/main/java/com/grash/controller/WorkOrderMeterTriggerController.java
    @PostMapping("")
    @PreAuthorize("hasRole('ROLE_CLIENT')")
    WorkOrderMeterTriggerShowDTO create(@Parameter(description = "Work order meter trigger to create") @Valid @RequestBody WorkOrderMeterTriggerPostDTO workOrderMeterTriggerReq,
                                        HttpServletRequest req) {
        User user = userService.whoami(req);
        return workOrderMeterTriggerMapper.toShowDto(workOrderMeterTriggerService.create(workOrderMeterTriggerReq,
                user.getCompany()));
    }
```

`@PreAuthorize("hasRole('ROLE_CLIENT')")` gates by user role only. There is
no `PlanFeature` annotation, no entitlement check at the controller layer. So
the controller passes straight through.

### 3.3 Service — was the only license gate, now gone

That is exactly the gate we removed. Once `create(...)` no longer calls
`hasEntitlement`, the request is persisted by JPA and returned as a normal
`WorkOrderMeterTriggerShowDTO`.

### 3.4 Trigger evaluation — never gated

When readings arrive, `ReadingController` (and the underlying meter logic)
inspects the existing `WorkOrderMeterTrigger` rows for the meter, evaluates
the `MORE_THAN` / `LESS_THAN` condition against the new value, and if it
matches it generates a work order. None of that path checks
`CONDITION_BASED_PM`. So the moment a user can *create* the trigger, the
trigger will *fire* on subsequent readings, completely independent of
licensing.

### 3.5 Other entrypoints — unchanged on purpose

The same service has `update`, `delete`, `findById`, `findByMeter`, and
`getAll`. None of those were license-gated to begin with — only `create`
was — so leaving them untouched preserves existing behavior. If you wanted
to be aggressive you could remove the import / dependency more broadly, but
nothing else in this file ever called `licenseService`, so the change here is
already minimal.

### 3.6 Other licensed features — untouched

The licensing system itself still works: `LicenseService`, `LicenseController`
(`GET /license/state`), the Keygen.sh validation path, the Ed25519 + AES-GCM
offline file path, and every other `hasEntitlement(...)` call in the
codebase are all left alone. So `SSO`, `WORKFLOW`, `BRANDING`,
`CUSTOM_ROLES`, etc. are still gated by their respective entitlements. We
only un-gated `CONDITION_BASED_PM` *as it pertains to creating meter
triggers*; the entitlement enum value itself still exists and is still
checked in any other place that uses it (currently, this was the only
place — see §6.4).

---

## 4. The concept this exercises

The change is small, but the underlying ideas are exactly what a real
licensing layer in a multi-tenant SaaS looks like. Mentally tagging each
layer makes future feature gating and de-gating much easier.

### 4.1 Two parallel gating systems

This codebase blends *two* separate gating mechanisms:

| System | Purpose | Lives in DB? | Per | Example check |
| --- | --- | --- | --- | --- |
| `PlanFeatures` (subscription plan) | Tier-based pricing, per-tenant limits | Yes — on `SubscriptionPlan` rows | Per company | `companyService.hasFeature(company, PlanFeature.METER)` |
| `LicenseEntitlement` (Keygen license) | Self-hosted feature unlocks | No — derived from license | Per running backend instance | `licenseService.hasEntitlement(CONDITION_BASED_PM)` |

For meter triggers, only the **license entitlement** path was used. So the
fix lives entirely in the entitlement layer. If meter triggers had also been
subscription-plan gated, you'd also need to flip a `PlanFeature` flag on the
plan, but that is *not* the case here — confirm it yourself by searching the
codebase for `WorkOrderMeterTrigger` and `PlanFeature` together; you'll find
no overlap.

### 4.2 Where to put / remove a feature gate

A clean feature gate has exactly **one** place that says yes/no, and every
caller funnels through that. In this codebase, the meter-trigger gate had
exactly this shape:

```
controller (role check)
    └──> service.create(...)
              └──> licenseService.hasEntitlement(CONDITION_BASED_PM)   <-- single gate
                     └──> 403 or pass
```

Because the gate was a single line in a single method, removing it is a
single edit. That is a *good* design: imagine if every controller or every
mapper repeated the same `hasEntitlement` call. Then you'd have to delete it
in many files, and one missed call would still 403. The lesson: **keep
gates centralized in the service layer, not sprinkled across controllers.**

### 4.3 Defense in depth: backend > frontend

The frontend never gated the *Add trigger* button. The backend always did.
That asymmetry is a feature, not a bug:

- Frontend gates exist for **UX** (don't show buttons that won't work).
- Backend gates exist for **enforcement** (you can't bypass a UX gate by
  calling the API directly).

If you only had a frontend check, anyone with curl could create triggers.
If you only have a backend check, the user has a poor UX and gets a snackbar
after filling out a form. A real product wants both.

In this learning exercise we removed the backend gate, which means *any*
client (the UI, mobile app, curl, Postman, internal scripts) can now create
triggers on this build. That is the desired effect.

### 4.4 Why removing the import + field matters

Java with Lombok's `@RequiredArgsConstructor` injects every `private final`
field as a constructor argument. If we leave `private final LicenseService
licenseService;` after deleting the only call site, three things happen:

1. The Spring container still wires `LicenseService` into the bean (harmless
   but pointless).
2. Future readers see the field and assume the service still does license
   checks somewhere — misleading.
3. Some compile/lint configurations flag unused fields and unused imports.

Removing the field and the import keeps the code honest. This is a tiny but
important habit when removing features.

---

## 5. How to learn from this practically

This is the workflow recommended for learning the licensing layer
hands-on. You will need a working `docker compose` stack of this repo
(`api`, `frontend`, db, etc.).

### 5.1 Confirm the original behavior (before the change)

On the `cursor/meter-trigger-license-doc-0a6e` branch, *without* a
`LICENSE_KEY`:

```bash
# 1. Bring up the stack
docker compose up -d

# 2. Confirm the license is invalid
curl http://localhost:8080/license/state
# Expect: {"hasLicense":false,"valid":false,"entitlements":[]}

# 3. Log in as a CLIENT user, find a meter id, then try to create a trigger
TOKEN=...   # JWT from /auth/signin
curl -X POST http://localhost:8080/work-order-meter-triggers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name":"demo",
        "triggerCondition":"MORE_THAN",
        "value":1,
        "title":"trigger-demo",
        "meter":{"id": 1}
      }'
# Expect: HTTP 403, body
# {"success":false,"message":"You need a license to create a meter trigger"}
```

That is the symptom described in the original doc.

### 5.2 Apply this branch's change

Switch to the branch that contains this walkthrough's change, then rebuild
just the API container:

```bash
docker compose build api
docker compose up -d api
```

(Or `mvn -f api/pom.xml package -DskipTests` and `docker compose up -d api`.)

### 5.3 Confirm the new behavior

The license state hasn't changed — there is still no key:

```bash
curl http://localhost:8080/license/state
# Still: {"hasLicense":false,"valid":false,"entitlements":[]}
```

But the create call now succeeds:

```bash
curl -X POST http://localhost:8080/work-order-meter-triggers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name":"demo",
        "triggerCondition":"MORE_THAN",
        "value":1,
        "title":"trigger-demo",
        "meter":{"id": 1}
      }'
# Expect: HTTP 200, with a JSON body describing the persisted trigger
```

This concretely shows that **the only thing standing between you and a
working meter trigger was that single `if`**.

### 5.4 Verify the trigger fires

Push a reading above the threshold:

```bash
curl -X POST http://localhost:8080/readings/meter/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": 999, "createdAt": "2026-01-01T00:00:00Z"}'
```

You should see a new work order created from the trigger's template and, if
you look at the API logs, the corresponding webhook events firing. The
trigger evaluation path was never license-gated, so it just works.

### 5.5 Verify the rest of licensing still works

Pick another entitlement-gated endpoint and confirm it still 403s without a
license. Good candidates (search for `hasEntitlement(LicenseEntitlement` in
the API source):

```bash
grep -rn "licenseService.hasEntitlement(LicenseEntitlement\." api/src/main/java
```

Hit one of those endpoints and observe that you still get the licensing
error. That confirms our change is *targeted*: it only un-gated meter
triggers, not the whole licensing system.

---

## 6. Things you might think you need to change — but don't

### 6.1 Don't remove `LicenseEntitlement.CONDITION_BASED_PM`

The enum value still exists in:

- `api/src/main/java/com/grash/dto/license/LicenseEntitlement.java`
- `frontend/src/models/owns/license.ts`
- `mobile/models/license.ts`
- `home/src/models/owns/license.ts`

We left them all in place because:

- Other parts of the codebase / docs still reference the constant by name.
- A real Keygen license that the customer holds may still list the code; if
  we delete the enum, validation might log an "unknown entitlement" warning
  for a code it can't map. Better to keep the enum for compatibility.
- The original docs (e.g. `Licensing and feature gates.md`) describe the
  CBM entitlement as part of the product taxonomy — this is purely
  documentary now.

So: the **entitlement still exists in the model**, but **no code path
checks it for meter trigger creation**. This is the cleanest possible
removal.

### 6.2 Don't touch `LicenseService`

We only removed the *call*. `LicenseService` itself still:

- Reads `LICENSE_KEY` and `LICENSE_FILE_PATH`.
- Calls Keygen for online validation.
- Decrypts offline `.lic` files with Ed25519+AES-GCM.
- Caches results for 12 hours.
- Exposes `GET /license/state`.

That whole subsystem is unrelated to meter-trigger creation now. Other
features still depend on it.

### 6.3 Don't change the controller

The controller's `@PreAuthorize("hasRole('ROLE_CLIENT')")` is *not* a
license check, it is an authorization check (is the caller logged in as a
client user?). Leaving it in is correct. Removing it would expose the API
to anonymous calls, which is a security regression, not a "license fix".

### 6.4 Search for other call sites first

Before declaring a feature "ungated", always search:

```bash
grep -rn "CONDITION_BASED_PM" api/src/main/java
```

In this repo, there is currently only **one** consumer of
`CONDITION_BASED_PM` — the meter-trigger create — plus the enum definition
itself. So a single edit is enough. If a future developer adds new call
sites and you want to keep meter triggers free, you'll have to revisit and
remove those new gates too.

### 6.5 Don't lift the gate everywhere blindly

The `LicenseEntitlement` enum has many values (`SSO`, `WORKFLOW`,
`BRANDING`, `CUSTOM_ROLES`, etc.). Each one corresponds to a separately
licensed feature. Removing the meter-trigger gate to learn how it works is
fine. Removing *all* gates wholesale would constitute disabling commercial
licensing on the entire product, which is a legal issue if you redistribute
the result. Keep the change *scoped to one feature you are studying*.

---

## 7. Test plan

Manual verification you should run after applying the change:

1. **Backend compiles.** `./mvnw -f api/pom.xml -q -DskipTests package` (or
   `mvn package`). Lombok will regenerate the `@RequiredArgsConstructor`
   without `LicenseService`.
2. **Spring context starts.** `docker compose up -d api` and confirm the
   API logs show `Started GrashApplication`. No bean wiring errors should
   appear (we removed a field, not a bean).
3. **License state endpoint still works.**
   `curl http://localhost:8080/license/state` returns the expected JSON.
4. **Trigger create works without a license.** As shown in §5.3, returns
   HTTP 200.
5. **Trigger fires on a reading.** As shown in §5.4, a work order is
   generated.
6. **Other entitlement-gated endpoints still 403.** Pick one from the grep
   in §5.5 and confirm the licensing layer still enforces other features.

There are no unit tests in this repo for `WorkOrderMeterTriggerService`, so
no test files needed updating. If you add one, gear it toward verifying
that the create path no longer touches `LicenseService` (e.g., construct
the service without a `LicenseService` mock and assert that creation
succeeds).

---

## 8. How to revert this change cleanly

If you ever want to put the gate back, the inverse patch is small and self
contained:

1. Re-add the import:

   ```java
   import com.grash.dto.license.LicenseEntitlement;
   ```

2. Re-add the `LicenseService` field:

   ```java
   private final LicenseService licenseService;
   ```

3. Re-add the guard at the top of `create(...)`:

   ```java
   if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
       throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);
   ```

That returns the system to its original behavior. Lombok regenerates the
constructor automatically; no other code changes are needed.

---

## 9. Concept checklist

After running through this exercise, you should be able to answer these
without looking at the code:

- [ ] What is the difference between a `PlanFeature` and a
      `LicenseEntitlement`?
- [ ] Why is gating the meter-trigger feature in the *service layer* (not
      the controller) better than gating it everywhere?
- [ ] What does `licenseService.hasEntitlement(...)` return when no
      `LICENSE_KEY` and no `LICENSE_FILE_PATH` are set, and why?
- [ ] How long is the license result cached, and why does that matter for
      "I just set my key but the API still 403s"?
- [ ] Why does removing the entitlement check on *creation* automatically
      enable trigger *firing* without further changes?
- [ ] Why didn't we have to touch the frontend at all?
- [ ] Why is removing the unused field and import (not just the `if`) the
      right hygiene?
- [ ] If you wanted to keep the gate but show a nicer UI message, where
      would you add a frontend check?

If you can answer all of those, the licensing layer of this codebase is no
longer a black box — it is a small, well-defined, two-axis gate that you
can confidently extend or remove on a per-feature basis.

---

## 10. Related reading

- [Meter trigger licensing mechanism](./Meter%20trigger%20licensing%20mechanism.md)
  — the original deep dive on what produces the error.
- [Licensing and feature gates](./Licensing%20and%20feature%20gates.md) —
  big-picture view of how licensing is wired into the product.
- [Licensing implementation assessment](./Licensing%20implementation%20assessment.md)
  — review of the current licensing implementation and known issues.
- [Condition-based maintenance support](./Condition-based%20maintenance%20support.md)
  — what CBM means in this product and which parts of it are licensed.
