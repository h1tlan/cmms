# Meter Trigger Licensing Mechanism

This document explains, in depth, what happens when a user opens a meter and
clicks **Add trigger** to create a work-order meter trigger, and ends up seeing:

```text
You need a license to create a meter trigger
```

It traces every step of the call from the click in the browser, through the
HTTP request, into the Spring Boot backend, into the license service, and back
out to the snackbar that displays the error. It also explains the licensing
model behind the message: what `CONDITION_BASED_PM` is, where entitlements come
from, how Keygen.sh is involved, and how Docker Compose configuration affects
all of it.

This is a technical analysis of the repository at the current commit. It is
not legal advice and it is not an instruction to bypass any license check.

---

## 1. TL;DR

Meter triggers are a **condition-based maintenance (CBM)** feature.

Creating one is gated at the backend by a single check:

```text
licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM)
```

If the running backend does **not** have a valid commercial/self-hosted
license that includes the `CONDITION_BASED_PM` entitlement, the create call
fails with HTTP 403 and message:

```text
You need a license to create a meter trigger
```

The frontend currently does not check this entitlement before opening the
"Add trigger" dialog, so users see the create form, fill it in, submit it,
and only then receive the error in a red snackbar.

The fix is either:

1. Provide a valid license that grants `CONDITION_BASED_PM` (production fix), or
2. Hide / disable the `Add trigger` button when the entitlement is missing,
   for a better UX (frontend fix), or
3. Both, which is the right long-term answer.

---

## 2. Where the message comes from

The exact string lives in the backend service layer:

```36:51:api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java
    @Transactional
    public WorkOrderMeterTrigger create(WorkOrderMeterTrigger workOrderMeterTrigger, Company company) {
        if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
            throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);

        if (workOrderMeterTrigger instanceof WorkOrderMeterTriggerPostDTO workOrderMeterTriggerPostDTO) {
            workOrderMeterTrigger = workOrderMeterTriggerMapper.fromPostDto(workOrderMeterTriggerPostDTO);
            if (!workOrderMeterTriggerPostDTO.getCustomFields().isEmpty()) {
                setMeterTriggerCustomFields(workOrderMeterTrigger, workOrderMeterTriggerPostDTO.getCustomFields(),
                        company);
            }
        }
```

That line is the source of the popup you saw. Everything else in this doc
explains what makes that condition true or false at runtime.

---

## 3. End-to-end trace: from clicking "Add" to seeing the error

This is the complete flow, in order, with the exact files involved.

### Step 1. The user opens a meter and the trigger UI loads

File:

```text
frontend/src/content/own/Meters/MeterDetails.tsx
```

When the meter detail page mounts it asks for the meter triggers:

```74:78:frontend/src/content/own/Meters/MeterDetails.tsx
  useEffect(() => {
    dispatch(getWorkOrderMeterTriggers(meter.id));
    dispatch(getReadings(meter.id));
  }, [meter.id]);
```

Below the list of existing triggers, the page renders an **Add trigger**
button. The only check guarding the button is the meters edit permission,
not a license entitlement:

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

That is important: there is **no** frontend license gate on this button. The
frontend never asks "is `CONDITION_BASED_PM` enabled?" before showing it.
That is why a user with a free / unlicensed install can still see and click
the button.

### Step 2. The Add Trigger dialog opens

File:

```text
frontend/src/content/own/Meters/AddTriggerModal.tsx
```

The modal renders a form with:

- trigger name
- condition (`MORE_THAN` / `LESS_THAN`)
- threshold value
- the standard work-order base fields (title, priority, asset, etc.)
- any company custom fields for work orders

The user fills in the form and clicks **Add**.

### Step 3. The form submits

The submit handler in `AddTriggerModal.tsx` first uploads any files, then
dispatches the create thunk:

```130:154:frontend/src/content/own/Meters/AddTriggerModal.tsx
          onSubmit={async (values) => {
            let formattedValues = formatValues(values);
            try {
              const uploadedFiles = await uploadFiles(
                formattedValues.files,
                formattedValues.image
              );

              const imageAndFiles = getImageAndFiles(uploadedFiles);
              formattedValues = {
                ...formattedValues,
                image: imageAndFiles.image,
                files: imageAndFiles.files
              };

              await dispatch(
                createWorkOrderMeterTrigger(meter.id, formattedValues)
              );
              onCreationSuccess();
            } catch (err) {
              onCreationFailure(err);
              throw err;
            }
          }}
```

`onCreationFailure` is what shows the snackbar with the backend error message:

```101:102:frontend/src/content/own/Meters/AddTriggerModal.tsx
  const onCreationFailure = (err) =>
    showSnackBar(getErrorMessage(err, t('wo_trigger_create_failure')), 'error');
```

`getErrorMessage` extracts the backend `message` field from the JSON error
body:

```58:68:frontend/src/utils/api.ts
export const getErrorMessage = (
  error: any,
  defaultMessage?: string
): string => {
  try {
    const parsed = JSON.parse(error.message);
    return parsed?.message ?? error.message ?? defaultMessage;
  } catch {
    return error.message ?? defaultMessage;
  }
};
```

So the exact string the user sees in the red snackbar is the `message` field
from the backend JSON response.

### Step 4. The Redux thunk POSTs to the API

File:

```text
frontend/src/slices/workOrderMeterTrigger.ts
```

```89:105:frontend/src/slices/workOrderMeterTrigger.ts
export const createWorkOrderMeterTrigger =
  (
    id: number,
    workOrderMeterTrigger: Partial<WorkOrderMeterTrigger>
  ): AppThunk =>
  async (dispatch) => {
    const workOrderMeterTriggerResponse = await api.post<WorkOrderMeterTrigger>(
      `${basePath}`,
      { ...workOrderMeterTrigger, meter: { id } }
    );
    dispatch(
      slice.actions.createWorkOrderMeterTrigger({
        meterId: id,
        workOrderMeterTrigger: workOrderMeterTriggerResponse
      })
    );
  };
```

The request that is sent is:

```text
POST {API_URL}/work-order-meter-triggers
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "name": "...",
  "triggerCondition": "MORE_THAN" | "LESS_THAN",
  "value": 80,
  "title": "...",
  ...standard work order fields...,
  "meter": { "id": <meterId> }
}
```

When the request is non-ok, the generic API helper throws an `Error` whose
message is the **stringified** backend JSON:

```4:14:frontend/src/utils/api.ts
function api<T>(url: string, options: Options): Promise<T> {
  return fetch(url, { headers: authHeader(false), ...options }).then(
    async (response) => {
      if (!response.ok) {
        throw new Error(JSON.stringify(await response.json()));
      }
      if (options?.raw) return response as unknown as Promise<T>;
      return response.json() as Promise<T>;
    }
  );
}
```

That is why `getErrorMessage` later does `JSON.parse(error.message)` and pulls
out the `message` property.

### Step 5. Spring routes the request to the controller

File:

```text
api/src/main/java/com/grash/controller/WorkOrderMeterTriggerController.java
```

The endpoint:

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

Things to notice:

- `@PreAuthorize("hasRole('ROLE_CLIENT')")` only gates the call by user role,
  not by license.
- The controller does **not** check `PlanFeatures` here. The plan-feature gate
  for meters lives on the Reading / Meter side.
- The controller passes the request straight to
  `WorkOrderMeterTriggerService.create(...)`, which is where the license check
  happens.

### Step 6. The license entitlement check fires

This is the core of the issue. Inside the service:

```36:39:api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java
    @Transactional
    public WorkOrderMeterTrigger create(WorkOrderMeterTrigger workOrderMeterTrigger, Company company) {
        if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
            throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);
```

`licenseService.hasEntitlement(...)` is the function that decides whether
this throws.

### Step 7. `LicenseService.hasEntitlement` checks cached license state

File:

```text
api/src/main/java/com/grash/service/LicenseService.java
```

```73:76:api/src/main/java/com/grash/service/LicenseService.java
    public boolean hasEntitlement(LicenseEntitlement entitlement) {
        LicensingState state = getLicensingState();
        return state.isValid() && state.getEntitlements().contains(entitlement.toString());
    }
```

A feature is allowed only when **both**:

1. `state.isValid()` is `true`, and
2. the entitlement code (the string `"CONDITION_BASED_PM"`) is present in the
   entitlements set.

If you have a license but it does not include `CONDITION_BASED_PM`, this
returns `false`. If you have no license at all, the state is invalid and this
also returns `false`.

### Step 8. `getLicensingState()` resolves the current state

```51:67:api/src/main/java/com/grash/service/LicenseService.java
    public synchronized LicensingState getLicensingState() {
        if (isCacheValid()) {
            return buildLicensingStateFromCache();
        }

        if (!hasLicenseKey() && !hasLicenseFile()) {
            return clearCacheAndReturnInvalid();
        }

        // Try license file validation first if available
        if (hasLicenseFile()) {
            return validateAndCacheLicenseFile();
        }

        // Fall back to Keygen API validation
        return validateAndCacheLicenseKey();
    }
```

The method has a 12 hour cache:

```23:23:api/src/main/java/com/grash/service/LicenseService.java
    private static final long CACHE_DURATION_MILLIS = 12 * 60 * 60 * 1000; // 12 hours
```

Resolution order:

1. **Cache hit.** If the cache is fresh, return whatever was last computed.
2. **No license.** If neither `LICENSE_KEY` nor `LICENSE_FILE_PATH` is set,
   return an invalid state. This is the default Docker Compose case, and the
   reason a fresh local stack always shows the meter trigger error.
3. **License file path set.** Validate and decrypt the offline license file.
4. **License key set, no file.** Call Keygen.sh to validate the key, then
   fetch the entitlements list.

### Step 9. Online path: Keygen.sh validation

If only `LICENSE_KEY` is configured, the service hits Keygen:

```24:26:api/src/main/java/com/grash/service/LicenseService.java
    private static final String API_URL_TEMPLATE = "https://api.keygen.sh/v1/accounts/%s/licenses/actions/validate-key";
    private static final String ENTITLEMENTS_URL_TEMPLATE = "https://api.keygen.sh/v1/accounts/%s/licenses/%s" +
            "/entitlements?limit=100";
```

Validation flow:

```147:184:api/src/main/java/com/grash/service/LicenseService.java
    private LicensingState validateAndCacheLicenseKey() {
        long now = System.currentTimeMillis();

        try {
            Optional<LicenseValidationResponse> response = performLicenseValidation();

            if (response.isPresent()) {
                cachedLicenseResponse = response.get();
                lastCheckedTime = now;

                if (cachedLicenseResponse.getMeta().isValid()) {
                    fetchAndCacheEntitlements(cachedLicenseResponse.getData().getId());
                } else {
                    cachedEntitlements.clear();
                }
                ...
```

Two pieces are fetched:

1. `validate-key` returns a `meta.valid` boolean, plan attributes, expiry,
   etc.
2. If valid, `/entitlements` returns the list of entitlement codes for that
   license.

Optionally, if `LICENSE_FINGERPRINT_REQUIRED` is true (the default), a machine
fingerprint is sent so the license is bound to one host:

```251:271:api/src/main/java/com/grash/service/LicenseService.java
    private LicenseValidationRequest buildValidationRequest() {
        LicenseValidationRequest request = new LicenseValidationRequest();
        LicenseValidationMeta meta = new LicenseValidationMeta();
        meta.setKey(licenseKey);

        if (licenseFingerprintRequired) {
            addFingerprintToMeta(meta);
        }

        request.setMeta(meta);
        return request;
    }

    private void addFingerprintToMeta(LicenseValidationMeta meta) {
        String fingerprint = FingerprintGenerator.generateFingerprint();
        log.info("X-Machine-Fingerprint: {}", fingerprint);

        LicenseValidationScope scope = new LicenseValidationScope();
        scope.setFingerprint(fingerprint);
        meta.setScope(scope);
    }
```

The entitlement codes are stored in the local cache:

```301:308:api/src/main/java/com/grash/service/LicenseService.java
    private void cacheEntitlements(EntitlementsResponse response) {
        cachedEntitlements = response.getData().stream()
                .map(EntitlementData::getAttributes)
                .map(EntitlementAttributes::getCode)
                .collect(Collectors.toSet());

        log.info("Cached {} entitlements: {}", cachedEntitlements.size(), cachedEntitlements);
    }
```

Each Keygen entitlement has a `code` such as `CONDITION_BASED_PM`. That is the
exact string compared in `hasEntitlement`.

### Step 10. Offline path: signed license file validation

If `LICENSE_FILE_PATH` is set, an offline `.lic` file is decrypted using the
license key. The file is verified using **Ed25519** signatures and decrypted
using **AES-256-GCM**:

```24:34:api/src/main/java/com/grash/utils/LicenseFileValidator.java
@Slf4j
public class LicenseFileValidator {

    private static final String LICENSE_FILE_HEADER = "-----BEGIN LICENSE FILE-----";
    private static final String LICENSE_FILE_FOOTER = "-----END LICENSE FILE-----";
    private static final String SUPPORTED_ALGORITHM = "aes-256-gcm+ed25519";
```

The Keygen public key used for signature verification is hard-coded:

```28:30:api/src/main/java/com/grash/service/LicenseService.java
    private static final String KEYGEN_PUBLIC_KEY =
            "cf9ce7f95c29c0cb0666a61d89c931bd4170a5fbaa0a391ff6649c213f4d13fc";
```

Once decrypted, the JSON payload is parsed into a `DecryptedLicenseData`, and
the entitlements are extracted from its `included` resources:

```97:105:api/src/main/java/com/grash/dto/license/DecryptedLicenseData.java
    public Set<String> getEntitlements() {
        Map<String, Object> relationships = this.getData().getRelationships();
        Map<String, Object> entitlements = (Map<String, Object>) relationships.get("entitlements");
        List<Map<String, Object>> entitlementObjects = (List<Map<String, Object>>) entitlements.get("data");
        List<String> ids =
                entitlementObjects.stream().map(entitlementObject -> (String) entitlementObject.get("id")).toList();
        return ids.stream().map(id -> this.getIncluded().stream().filter(includedResource -> includedResource.getType().equals(
                "entitlements") && includedResource.getId().equals(id)).findFirst().get().getAttributes().getCode()).collect(Collectors.toSet());
    }
```

`isTimeValid()` also enforces the `issued`/`expiry` window so a stolen file
cannot outlive its expiry:

```46:66:api/src/main/java/com/grash/dto/license/DecryptedLicenseData.java
    public boolean isTimeValid() {
        if (meta == null) {
            return false;
        }

        Date issued = meta.getIssued();
        Date expiry = meta.getExpiry();
        Date now = new Date();

        // Check that issued is not in the future (clock tampering)
        if (issued != null && issued.after(now)) {
            return false;
        }

        // Check that expiry is not in the past
        if (expiry != null && expiry.before(now)) {
            return false;
        }

        return true;
    }
```

Note: even in offline mode, a `LICENSE_KEY` is still required because it is
used as the AES key derivation input.

### Step 11. The exception is thrown

If the entitlement is missing, the service throws:

```7:7:api/src/main/java/com/grash/exception/CustomException.java
public class CustomException extends RuntimeException {
```

```36:38:api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java
        if (!licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM))
            throw new CustomException("You need a license to create a meter trigger", HttpStatus.FORBIDDEN);
```

### Step 12. The global exception handler shapes the HTTP response

File:

```text
api/src/main/java/com/grash/exception/GlobalExceptionHandlerController.java
```

```39:43:api/src/main/java/com/grash/exception/GlobalExceptionHandlerController.java
    @ExceptionHandler(CustomException.class)
    public ResponseEntity<SuccessResponse> handleCustomException(HttpServletResponse res, CustomException ex) {
        ex.printStackTrace();
        return new ResponseEntity<>(new SuccessResponse(false, ex.getMessage()), ex.getHttpStatus());
    }
```

This is what the frontend actually receives:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "success": false,
  "message": "You need a license to create a meter trigger"
}
```

### Step 13. The frontend extracts the message and displays it

Path back through the stack:

1. `fetch` sees `response.ok === false` and throws
   `new Error(JSON.stringify(await response.json()))`.
2. The thunk re-throws, so the modal's submit handler catches the error.
3. `onCreationFailure(err)` calls
   `showSnackBar(getErrorMessage(err, t('wo_trigger_create_failure')), 'error')`.
4. `getErrorMessage` parses the stringified JSON and returns
   `parsed.message`.
5. `CustomSnackBarContext` displays a red snackbar with the text.

That is the full path from click to error.

---

## 4. The licensing model behind the message

The error is part of a broader licensing model that mixes two distinct
concepts.

### 4.1 `PlanFeatures` vs `LicenseEntitlement`

| Concept | Source of truth | Granularity | Example |
| --- | --- | --- | --- |
| `PlanFeatures` | Database subscription plan attached to the Company | Per company | `METER`, `ROLE`, `SIGNATURE` |
| `LicenseEntitlement` | Self-hosted/commercial license via Keygen | Per running backend instance | `CONDITION_BASED_PM`, `CUSTOM_ROLES`, `SSO` |

For meter triggers, the gate that fires is the **license entitlement**, not
the plan feature. The plan-feature `METER` only gates *creating readings* in
`ReadingController` and *managing meters* in `MeterController`. The trigger
service itself only checks `CONDITION_BASED_PM`.

### 4.2 Where `CONDITION_BASED_PM` is defined

Backend enum:

```39:41:api/src/main/java/com/grash/dto/license/LicenseEntitlement.java
    @Schema(description = "Condition-based preventive maintenance triggers")
    CONDITION_BASED_PM,
```

Frontend mirror list:

```1:33:frontend/src/models/owns/license.ts
const licenseEntitlements = [
  'SSO',
  'WORK_ORDER_HISTORY',
  ...
  'CONDITION_BASED_PM',
  ...
] as const;
```

Note: `CONDITION_BASED_PM` is in both lists, but the frontend never actually
checks it for the meter trigger button.

### 4.3 The `LicensingState` returned to the frontend

The frontend can fetch the current state from `GET /license/state`:

```12:24:api/src/main/java/com/grash/controller/LicenseController.java
@RestController
@RequestMapping("/license")
@RequiredArgsConstructor
@Hidden
public class LicenseController {

    private final LicenseService licenseService;

    @GetMapping("/state")
    public LicensingState getValidity(HttpServletRequest req) {
        return licenseService.getLicensingState();
    }
}
```

The DTO it returns:

```12:29:api/src/main/java/com/grash/dto/license/LicensingState.java
@Data
@Builder
@Schema(description = "Current licensing state for an organization, including plan, entitlements, and validity")
public class LicensingState {
    @Schema(description = "Whether the organization has an active license")
    private boolean hasLicense;
    @Schema(description = "Whether the current license is valid")
    private boolean valid;
    @Schema(description = "Name of the current subscription plan")
    private String planName;
    @Schema(description = "Set of entitlement codes granted by the current license")
    @Builder.Default
    private Set<String> entitlements = new HashSet<>();
    @Schema(description = "Expiration date of the current license")
    private Date expirationDate;
    @Schema(description = "Number of licensed users")
    private int usersCount;
}
```

The frontend stores it in Redux and exposes the `useLicenseEntitlement` hook:

```1:14:frontend/src/hooks/useLicenseEntitlement.ts
import { useSelector } from '../store';
import { LicenseEntitlement, LicensingState } from '../models/owns/license';

export const useLicenseEntitlement = (entitlement: LicenseEntitlement) => {
  const licensingState = useSelector((state) => state.license.state);

  return hasLicenseEntitlement(licensingState, entitlement);
};
const hasLicenseEntitlement = (
  license: LicensingState,
  entitlement: LicenseEntitlement
) => {
  return license.valid && license.entitlements.some((e) => e === entitlement);
};
```

The hook exists, but the meter trigger UI does not call it. That is the
primary UX bug behind your experience.

---

## 5. Why a default Docker run always shows the error

The root `docker-compose.yml` injects empty defaults for the license envs:

```yaml
LICENSE_KEY: ${LICENSE_KEY:-}
LICENSE_FINGERPRINT_REQUIRED: ${LICENSE_FINGERPRINT_REQUIRED:-true}
LICENSE_FILE_PATH: ${LICENSE_FILE_PATH:-}
```

The Spring properties:

```yaml
license-key: ${LICENSE_KEY:}
license-fingerprint-required: ${LICENSE_FINGERPRINT_REQUIRED:true}
license-file-path: ${LICENSE_FILE_PATH:}
```

So the very first thing `getLicensingState()` checks is:

```56:58:api/src/main/java/com/grash/service/LicenseService.java
        if (!hasLicenseKey() && !hasLicenseFile()) {
            return clearCacheAndReturnInvalid();
        }
```

That returns:

```json
{
  "hasLicense": false,
  "valid": false,
  "entitlements": []
}
```

Therefore `hasEntitlement(CONDITION_BASED_PM)` is `false`, and the create
endpoint always 403s. This is by design in the open-source / no-license path.

You can confirm this in your local stack:

```bash
curl http://localhost:8080/license/state
```

If you see `"valid": false`, that is your root cause.

---

## 6. State diagram of the license check

The decision tree the backend follows for **every** trigger create:

```
            ┌─────────────────────────────────┐
            │ POST /work-order-meter-triggers │
            └──────────────┬──────────────────┘
                           │
                @PreAuthorize ROLE_CLIENT
                           │
           ┌───────────────▼─────────────────┐
           │ WorkOrderMeterTriggerService    │
           │  .create(...)                   │
           └───────────────┬─────────────────┘
                           │
                hasEntitlement(CONDITION_BASED_PM) ?
                           │
        ┌──────────────────┴──────────────────┐
        │ false                                true │
        │                                       │
        ▼                                       ▼
 throw 403 "You need a              persist trigger,
 license to create a                 return ShowDTO
 meter trigger"
```

`hasEntitlement` itself is:

```
        ┌──────────────────────┐
        │ hasEntitlement(E)    │
        └──────────┬───────────┘
                   │
              getLicensingState()
                   │
       ┌───────────┴────────────┐
       │ cache fresh?           │
       │   yes → use cache      │
       │   no  → continue       │
       └───────────┬────────────┘
                   │
        no key && no file ?
        ┌──────────┴──────────┐
        │ yes                  no │
        ▼                        ▼
  return invalid           file present?
                       ┌─────────┴─────────┐
                       │ yes                no │
                       ▼                       ▼
              decrypt+verify file         POST validate-key
              parse entitlements          GET entitlements
                       │                       │
                       └────────┬──────────────┘
                                │
                       cache for 12h
                                │
                                ▼
                state.valid && entitlements.contains(E.toString())
```

---

## 7. Why the UX is confusing

The flow has three distinct UX problems:

1. **No frontend gate.** `MeterDetails.tsx` only gates the *Add trigger* button
   by `hasEditPermission(METERS)`. It does not call
   `useLicenseEntitlement('CONDITION_BASED_PM')`. The button is always shown
   to users who can edit meters.

2. **No pre-flight check.** The form does not call `/license/state` and adapt
   the dialog. It lets the user fill in the form, upload files, and submit
   before discovering the gate.

3. **Generic error message.** The backend just says "You need a license".
   It does not say:
   - whether the system has any license,
   - whether the license is invalid or expired,
   - which entitlement is missing,
   - or what plan the user should upgrade to.

The information is all available in `GET /license/state` (`hasLicense`,
`valid`, `entitlements`, `expirationDate`, `planName`), but the frontend
does not surface it on the meter trigger screen.

---

## 8. How to fix it

There are three meaningful fixes, depending on what you want.

### 8.1 Provide a valid license

If you are an Atlas CMMS commercial customer, set the env variable in your
`.env`:

```env
LICENSE_KEY=your_license_key_here
```

Restart the API:

```bash
docker compose up -d
# or
docker compose restart api
```

For offline / air-gapped installs, also set:

```env
LICENSE_FILE_PATH=/app/static/config/license.lic
LICENSE_KEY=your_license_key_here
```

Then verify:

```bash
curl http://localhost:8080/license/state
```

You want to see:

```json
{
  "hasLicense": true,
  "valid": true,
  "entitlements": ["...","CONDITION_BASED_PM","..."]
}
```

If `CONDITION_BASED_PM` is in `entitlements`, the trigger create call will
succeed.

### 8.2 Improve the frontend (recommended)

Update `MeterDetails.tsx` so the *Add trigger* button is disabled (or hidden)
when the entitlement is missing, and shows a clear tooltip explaining why.

Sketch:

```tsx
import { useLicenseEntitlement } from '../../../hooks/useLicenseEntitlement';
...
const canCreateMeterTrigger = useLicenseEntitlement('CONDITION_BASED_PM');
...
<Tooltip
  title={
    canCreateMeterTrigger
      ? ''
      : t('upgrade_meter_trigger') /* needs i18n key */
  }
>
  <span>
    <Button
      startIcon={<AddTwoToneIcon />}
      sx={{ my: 1 }}
      variant="outlined"
      disabled={!canCreateMeterTrigger}
      onClick={() => setOpenAddTriggerModal(true)}
    >
      {t('add_trigger')}
    </Button>
  </span>
</Tooltip>
```

This mirrors the existing pattern used by the roles page
(`PageHeader.tsx` with `hasFeature(PlanFeature.ROLE)`), but uses license
entitlements which is the correct gate for this feature.

### 8.3 Improve the backend error

Make the backend response carry a structured error code so the frontend can
render a useful message:

```json
{
  "success": false,
  "code": "LICENSE_ENTITLEMENT_REQUIRED",
  "requiredEntitlement": "CONDITION_BASED_PM",
  "message": "Meter triggers require a valid license with CONDITION_BASED_PM entitlement."
}
```

This requires a small change to the exception type and to
`GlobalExceptionHandlerController`. Long term it would let the frontend show
specific copy and even an "Upgrade" link.

---

## 9. Quick reference table

| Layer | File | What it does |
| --- | --- | --- |
| UI | `frontend/src/content/own/Meters/MeterDetails.tsx` | Renders the *Add trigger* button (no license gate) |
| UI | `frontend/src/content/own/Meters/AddTriggerModal.tsx` | Dialog with the trigger form, dispatches the create thunk |
| Redux | `frontend/src/slices/workOrderMeterTrigger.ts` | `createWorkOrderMeterTrigger` thunk |
| HTTP | `frontend/src/utils/api.ts` | `fetch` wrapper, throws `Error(JSON.stringify(body))` on non-2xx |
| Snackbar | `frontend/src/contexts/CustomSnackBarContext.tsx` | Shows the error message in a red snackbar |
| Hook | `frontend/src/hooks/useLicenseEntitlement.ts` | `valid && entitlements.includes(E)` |
| Slice | `frontend/src/slices/license.ts` | Loads `/license/state` |
| Controller | `api/src/main/java/com/grash/controller/WorkOrderMeterTriggerController.java` | Maps `POST /work-order-meter-triggers` to the service |
| Service | `api/src/main/java/com/grash/service/WorkOrderMeterTriggerService.java` | Throws `"You need a license to create a meter trigger"` if entitlement is missing |
| Service | `api/src/main/java/com/grash/service/LicenseService.java` | Validates license, fetches entitlements, caches for 12h |
| Validator | `api/src/main/java/com/grash/utils/LicenseFileValidator.java` | Verifies Ed25519 signature, decrypts AES-256-GCM payload |
| DTO | `api/src/main/java/com/grash/dto/license/DecryptedLicenseData.java` | Offline license payload + `isTimeValid()` + `getEntitlements()` |
| DTO | `api/src/main/java/com/grash/dto/license/LicenseEntitlement.java` | Enum of all entitlements, including `CONDITION_BASED_PM` |
| DTO | `api/src/main/java/com/grash/dto/license/LicensingState.java` | The `/license/state` response shape |
| Controller | `api/src/main/java/com/grash/controller/LicenseController.java` | Exposes `GET /license/state` |
| Exceptions | `api/src/main/java/com/grash/exception/GlobalExceptionHandlerController.java` | Wraps `CustomException` into a JSON response with `{success, message}` |
| Config | `api/src/main/resources/application.yml` | `license-key`, `license-fingerprint-required`, `license-file-path` |
| Config | `docker-compose.yml` | Injects `LICENSE_KEY`, `LICENSE_FINGERPRINT_REQUIRED`, `LICENSE_FILE_PATH` |
| Trigger usage | `api/src/main/java/com/grash/controller/ReadingController.java` | Where existing triggers actually fire (creates work orders, sends webhooks) — runs even on free tier; only **creation** of triggers is gated |

---

## 10. Related reading

- [Licensing and feature gates](./Licensing%20and%20feature%20gates.md) — broader
  view of how licensing works across the product.
- [Licensing implementation assessment](./Licensing%20implementation%20assessment.md)
  — review of the current licensing implementation and known issues.
- [Condition-based maintenance support](./Condition-based%20maintenance%20support.md)
  — what CBM means in this product and which parts of it are licensed.

---

## 11. Bottom line

The error message you saw is a single backend guard:

```text
WorkOrderMeterTriggerService.create
  -> licenseService.hasEntitlement(LicenseEntitlement.CONDITION_BASED_PM)
  -> false
  -> throw 403 "You need a license to create a meter trigger"
  -> GlobalExceptionHandlerController returns JSON {success:false, message:...}
  -> frontend api wrapper throws
  -> AddTriggerModal onCreationFailure shows it in a snackbar
```

The reason it triggers in a normal `docker compose up -d` run is that no
`LICENSE_KEY` and no `LICENSE_FILE_PATH` are configured, so the license state
is invalid, so no entitlements are granted, so `CONDITION_BASED_PM` is
missing, so meter trigger creation is forbidden.

To make meter triggers work, the backend needs a valid commercial / self-hosted
license that includes `CONDITION_BASED_PM`. To make the user experience
acceptable, the frontend should additionally check
`useLicenseEntitlement('CONDITION_BASED_PM')` before showing the *Add
trigger* button, and ideally return a structured error code from the backend
so the UI can show a meaningful, upgrade-aware message.
