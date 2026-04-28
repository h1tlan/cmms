# CMMS Feature Coverage and Future Roadmap

This document explains how much of a CMMS is already implemented in this
project, what appears partial, and what should be considered for future work.

It is based on repository inspection across:

- `api/`: Spring Boot backend
- `frontend/`: main React web app
- `mobile/`: Expo / React Native app
- `home/`: marketing site, only where relevant

It is a product/engineering inventory, not a runtime QA certification. Some
features may exist in code but still need testing, UX polish, licensing, or
deployment validation.

## Short answer

This project implements a large part of a real CMMS already.

It is not just a work-order demo. It includes:

- work orders
- tasks and checklists
- labor/time and additional costs
- preventive maintenance
- assets and asset downtime
- locations
- parts and inventory
- meters and readings
- purchase orders
- service requests and public request portal
- vendors and customers
- users, teams, roles, permissions
- files and attachments
- categories
- custom fields and configurable forms
- workflows
- analytics and reports
- imports and exports
- notifications
- API keys and webhooks
- licensing/subscription gates
- mobile app for field workflows

The project is strongest as a web-first CMMS with a supporting mobile technician
app. It is weaker in mobile admin parity, formal offline-first support, advanced
enterprise/EAM features, and product documentation around what is free,
licensed, complete, partial, or roadmap.

## Overall coverage estimate

A rough engineering estimate:

```text
Core CMMS backend:              high coverage
Main web CMMS app:              high coverage
Mobile technician workflows:    medium coverage
Mobile admin/planner features:  low-to-medium coverage
Advanced EAM/enterprise:        partial coverage
Offline-first operation:        limited / not first-class
Testing and QA confidence:      low
```

The most important takeaway:

```text
The product surface is broad, but not every surface has equal depth or equal
platform parity.
```

## Repository feature surfaces

| Surface | Role |
| --- | --- |
| `api/` | Backend domain model, APIs, persistence, jobs, licensing, integrations. |
| `frontend/` | Main authenticated CMMS web app. This is the richest UI. |
| `mobile/` | Mobile companion app for technicians and field workflows. |
| `home/` | Marketing/home site. Not the operational CMMS app. |

## Existing public feature claims

The root `README.MD` lists these feature groups:

- work orders and maintenance
- analytics and reporting
- equipment and inventory
- user and workflow management
- locations and requests

The repository also includes `api/Current features.pdf`, which lists many
features such as:

- work order archiving, filtering, time logs, checklists, PDF reports,
  signatures, configurable work order fields, relationships, calendar view,
  imports/exports, inventory consumption, and additional costs
- work order analytics
- reliability dashboard
- asset downtime and availability
- NFC support
- request approval/rejection and request-to-work-order conversion
- meters and meter triggers
- parts, sets, minimum quantity notifications
- purchase orders
- locations and map view
- people, teams, roles, checklists
- custom workflows

The codebase contains controllers, routes, screens, and slices corresponding to
most of these areas.

## Implemented feature map

### 1. Work orders

Status: strongly implemented.

Evidence:

- API:
  - `api/src/main/java/com/grash/controller/WorkOrderController.java`
  - `WorkOrderHistoryController.java`
  - `WorkOrderConfigurationController.java`
  - `WorkOrderCategoryController.java`
  - `WorkOrderMeterTriggerController.java`
- Frontend:
  - `frontend/src/content/own/WorkOrders/index.tsx`
  - `frontend/src/content/own/WorkOrders/Calendar/index.tsx`
- Mobile:
  - `mobile/screens/workOrders/WorkOrdersScreen.tsx`
  - `mobile/screens/workOrders/WODetailsScreen.tsx`
  - `mobile/screens/workOrders/CreateWorkOrderScreen.tsx`
  - `mobile/screens/workOrders/EditWorkOrderScreen.tsx`

Implemented capabilities appear to include:

- create, view, edit, and filter work orders
- list and calendar views
- assignment to users/teams/service providers
- priorities, statuses, categories
- related asset/location/parts/tasks
- configurable work order fields
- work order history
- completion flow
- mobile work order screens

Future improvements:

- stronger automated tests around status transitions
- clearer documented work order lifecycle
- better offline mobile handling for technicians
- stronger calendar/date behavior for Jalali support
- richer SLA/escalation behavior if needed

### 2. Tasks and checklists

Status: implemented.

Evidence:

- API:
  - `TaskController.java`
  - `TaskBaseController.java`
  - `ChecklistController.java`
- Frontend:
  - `frontend/src/content/own/Settings/Checklists/index.tsx`
  - shared task selection components
- Mobile:
  - `mobile/screens/workOrders/TasksScreen.tsx`
  - `mobile/screens/modals/SelectTasksModal.tsx`
  - `mobile/screens/modals/SelectChecklistsModal.tsx`
  - `mobile/screens/modals/SelectTasksOrChecklistModal.tsx`

Implemented capabilities appear to include:

- reusable checklists
- configurable task lists
- tasks attached to work orders
- mobile task/checklist selection

Future improvements:

- richer validation by task type
- conditional checklist logic
- audit history for checklist changes
- checklist scoring or compliance reporting if required

### 3. Labor, time, and additional costs

Status: implemented and license-gated in places.

Evidence:

- API:
  - `LaborController.java`
  - `AdditionalCostController.java`
  - `TimeCategoryController.java`
  - `CostCategoryController.java`
- Frontend:
  - work order detail and category areas
- Mobile:
  - `mobile/screens/workOrders/CreateAdditionalTime.tsx`
  - `mobile/screens/workOrders/CreateAdditionalCost.tsx`

Implemented capabilities appear to include:

- time logs
- labor cost
- additional costs
- time/cost categories
- mobile add-time/add-cost flows

Future improvements:

- timesheet approval
- labor rate history
- cost forecasting
- cost center/accounting integration

### 4. Preventive maintenance

Status: implemented on backend and web; limited on mobile.

Evidence:

- API:
  - `PreventiveMaintenanceController.java`
  - `ScheduleController.java`
  - `WorkOrderMeterTriggerController.java`
- Frontend:
  - `frontend/src/content/own/PreventiveMaintenance/index.tsx`
- Mobile:
  - preventive maintenance slice exists
  - work order filters reference parent PM
  - no dedicated PM schedule/calendar editor screens were found

Implemented capabilities appear to include:

- preventive maintenance schedules
- recurrence/schedule model
- PM-generated work orders
- meter-based triggers
- web PM management

Partial or missing:

- mobile PM list/editor
- mobile PM calendar
- advanced PM planning UX on mobile

Future improvements:

- mobile PM module
- PM forecast calendar
- PM compliance reporting improvements
- more explicit scheduling rules documentation
- better Jalali-aware scheduling display if Persian calendar support is added

### 5. Assets and equipment

Status: strongly implemented.

Evidence:

- API:
  - `AssetController.java`
  - `AssetCategoryController.java`
  - `AssetDowntimeController.java`
  - `DeprecationController.java`
- Frontend:
  - `frontend/src/content/own/Assets/index.tsx`
  - `frontend/src/content/own/Assets/Show/index.tsx`
  - asset analytics pages
- Mobile:
  - `mobile/screens/assets/AssetsScreen.tsx`
  - `mobile/screens/assets/CreateAssetScreen.tsx`
  - `mobile/screens/assets/EditAssetScreen.tsx`
  - `mobile/screens/assets/details/*`
  - `mobile/screens/ScanAssetScreen.tsx`

Implemented capabilities appear to include:

- asset records
- asset categories
- asset detail tabs
- work orders per asset
- asset parts/files
- asset downtime
- NFC/barcode scanning support
- asset analytics/reliability/cost views on web

Future improvements:

- deeper hierarchy/tree visualization
- calibration module
- warranty management
- lifecycle management
- asset condition scoring
- mobile floor-plan/asset location support
- advanced depreciation/accounting workflows if needed

### 6. Locations

Status: strongly implemented.

Evidence:

- API:
  - `LocationController.java`
  - `FloorPlanController.java`
- Frontend:
  - `frontend/src/content/own/Locations/index.tsx`
  - map component under `frontend/src/content/own/components/Map/index.tsx`
- Mobile:
  - `mobile/screens/locations/LocationsScreen.tsx`
  - `mobile/screens/locations/CreateLocationScreen.tsx`
  - `mobile/screens/locations/EditLocationScreen.tsx`
  - `mobile/screens/locations/details/*`

Implemented capabilities appear to include:

- location records
- location assets
- location work orders
- location files
- map support on web
- floor plan API and web-related usage

Partial or missing:

- mobile floor plan screens were not found
- advanced multi-level location hierarchy should be verified

Future improvements:

- mobile floor plan viewer
- QR/barcode wayfinding
- richer hierarchy visualization
- GIS/map enhancements

### 7. Parts and inventory

Status: strongly implemented.

Evidence:

- API:
  - `PartController.java`
  - `PartCategoryController.java`
  - `PartQuantityController.java`
  - `MultiPartsController.java`
- Frontend:
  - `frontend/src/content/own/Inventory/index.tsx`
- Mobile:
  - `mobile/screens/parts/PartsScreen.tsx`
  - `mobile/screens/parts/CreatePartScreen.tsx`
  - `mobile/screens/parts/EditPartScreen.tsx`
  - `mobile/screens/parts/details/*`

Implemented capabilities appear to include:

- part records
- categories
- inventory quantities
- parts used on work orders
- part assets/work orders/files on mobile
- multipart/sets
- low stock behavior through license entitlement

Future improvements:

- storerooms/bin locations
- stock transfers
- reorder points and purchase suggestions
- vendor pricing
- inventory valuation
- barcode label printing

### 8. Meters and readings

Status: implemented.

Evidence:

- API:
  - `MeterController.java`
  - `MeterCategoryController.java`
  - `ReadingController.java`
  - `WorkOrderMeterTriggerController.java`
- Frontend:
  - `frontend/src/content/own/Meters/index.tsx`
  - `frontend/src/content/own/Settings/Features/Meters/index.tsx`
- Mobile:
  - `mobile/screens/meters/MetersScreen.tsx`
  - `mobile/screens/meters/MeterDetails.tsx`
  - `mobile/screens/meters/CreateMeterScreen.tsx`
  - `mobile/screens/meters/EditMeterScreen.tsx`

Implemented capabilities appear to include:

- meters/counters
- readings
- meter categories
- meter-triggered work orders
- mobile meter management

Future improvements:

- IoT/sensor ingestion
- meter anomaly detection
- bulk readings import
- richer charts on mobile

### 9. Requests and request portal

Status: implemented.

Evidence:

- API:
  - `RequestController.java`
  - `RequestPortalController.java`
- Frontend:
  - `frontend/src/content/own/Requests/index.tsx`
  - public route `request-portal/:uuid`
  - request portal settings under `Settings/Features/RequestPortal`
- Mobile:
  - `mobile/screens/requests/RequestsScreen.tsx`
  - `mobile/screens/requests/CreateRequestScreen.tsx`
  - `mobile/screens/requests/EditRequestScreen.tsx`
  - `mobile/screens/requests/RequestDetails.tsx`

Implemented capabilities appear to include:

- create/manage requests
- approve/reject request workflow
- public request portal
- configurable request fields
- automatic work order creation after approval
- mobile request screens

Future improvements:

- requester-facing mobile app or PWA
- request SLA/escalation
- request communication thread
- portal branding and advanced access controls

### 10. Purchase orders

Status: implemented on backend and web; not implemented as a dedicated mobile UI.

Evidence:

- API:
  - `PurchaseOrderController.java`
  - `PurchaseOrderCategoryController.java`
- Frontend:
  - `frontend/src/content/own/PurchaseOrders/index.tsx`
- Mobile:
  - `mobile/slices/purchaseOrder.ts`
  - no dedicated `PurchaseOrdersScreen.tsx` found under `mobile/screens`

Implemented capabilities appear to include:

- purchase orders
- purchase order categories
- web management
- approval/rejection behavior according to feature document

Partial or missing:

- mobile purchase order workflow
- receiving/inventory reconciliation should be verified

Future improvements:

- mobile PO approvals
- receiving goods into inventory
- vendor price history
- purchasing requests/requisitions
- budget/cost center integration

### 11. Vendors, customers, and service providers

Status: implemented.

Evidence:

- API:
  - `VendorController.java`
  - `CustomerController.java`
- Frontend:
  - `frontend/src/content/own/VendorsAndCustomers/index.tsx`
- Mobile:
  - `mobile/screens/vendorsCustomers/*`

Implemented capabilities appear to include:

- vendor records
- customer records
- vendor/customer details
- mobile screens

Future improvements:

- vendor contracts
- vendor compliance documents
- preferred vendor rules
- contractor portal

### 12. People, teams, roles, and permissions

Status: implemented, with licensed custom-role gate.

Evidence:

- API:
  - `UserController.java`
  - `TeamController.java`
  - `RoleController.java`
  - `UserSettingsController.java`
- Frontend:
  - `frontend/src/content/own/PeopleAndTeams/index.tsx`
  - `frontend/src/content/own/Settings/Roles/index.tsx`
- Mobile:
  - `mobile/screens/peopleTeams/*`

Implemented capabilities appear to include:

- users
- teams
- invitations
- roles
- permissions
- profile/user settings
- mobile people/team screens

Important limitation:

- custom roles require both a plan feature and license entitlement.

Future improvements:

- clearer role/license UX
- role templates
- permission audit
- approval workflow permissions
- SCIM provisioning for enterprise users

### 13. Files and attachments

Status: implemented.

Evidence:

- API:
  - `FileController.java`
- Frontend:
  - `frontend/src/content/own/Files/index.tsx`
  - shared upload components
- Mobile:
  - entity-specific file tabs such as `AssetFiles.tsx`, `LocationFiles.tsx`,
    `PartFiles.tsx`

Implemented capabilities appear to include:

- file upload/download
- attachments to core entities
- company-wide file hub on web
- mobile entity file views

Partial or missing:

- standalone mobile files hub
- document scanning/OCR
- versioning

Future improvements:

- file permissions
- antivirus scanning
- OCR/search
- document version history
- storage quota reporting

### 14. Categories and configuration

Status: implemented.

Evidence:

- API:
  - `WorkOrderCategoryController.java`
  - `AssetCategoryController.java`
  - `PartCategoryController.java`
  - `PurchaseOrderCategoryController.java`
  - `MeterCategoryController.java`
  - `TimeCategoryController.java`
  - `CostCategoryController.java`
  - `GeneralPreferencesController.java`
  - `CompanySettingsController.java`
  - `UiConfigurationController.java`
- Frontend:
  - `frontend/src/content/own/Categories/*`
  - `frontend/src/content/own/Settings/General/index.tsx`
  - `frontend/src/content/own/Settings/Features/*`

Implemented capabilities appear to include:

- category administration
- general preferences
- feature configuration
- UI configuration
- work order/request form configuration

Future improvements:

- configuration audit history
- environment-level templates
- setup wizard
- import/export configuration

### 15. Custom fields and configurable forms

Status: implemented on backend and web; mobile consumes configured fields.

Evidence:

- API:
  - `CustomFieldController.java`
  - `FieldConfigurationController.java`
- Frontend:
  - settings feature pages
  - shared dynamic form component
- Mobile:
  - `mobile/utils/fields.ts`
  - form models/utilities

Implemented capabilities appear to include:

- custom fields
- required/optional/hidden field configuration
- configurable work order/request/entity forms

Future improvements:

- richer custom field types
- custom field validation rules
- conditional fields
- reporting on custom fields
- mobile admin screens for configuration

### 16. Workflows

Status: implemented on backend and web; not mobile admin.

Evidence:

- API:
  - `WorkflowController.java`
- Frontend:
  - `frontend/src/content/own/Settings/Features/Workflows/index.tsx`
- Mobile:
  - `mobile/slices/workflow.ts`
  - no workflow editor screens found

Implemented capabilities appear to include:

- custom workflow rules
- if/and/then style workflow configuration
- web settings UI
- license/plan gating

Future improvements:

- workflow execution logs
- workflow simulator/test mode
- workflow templates
- mobile read-only workflow status
- approval workflow builder

### 17. Analytics, dashboards, and reports

Status: implemented strongly on web; partial on mobile.

Evidence:

- API:
  - `analytics/WOAnalyticsController.java`
  - `analytics/AssetAnalyticsController.java`
  - `analytics/PartAnalyticsController.java`
  - `analytics/RequestAnalyticsController.java`
  - `analytics/UserAnalyticsController.java`
- Frontend:
  - `frontend/src/content/own/Analytics/*`
  - work order, request, part, asset analytics pages
- Mobile:
  - `mobile/screens/HomeScreen.tsx`
  - `mobile/screens/WorkOrderStatsScreen.tsx`

Implemented capabilities appear to include:

- work order analytics
- request analytics
- part consumption analytics
- asset reliability/cost analytics
- dashboard stats
- PDF reports for some workflows

Partial or missing:

- mobile full analytics parity
- advanced custom dashboard builder
- scheduled reports

Future improvements:

- custom dashboards
- scheduled email reports
- KPI targets
- SLA compliance dashboards
- reliability-centered maintenance analytics
- exportable analytics definitions

### 18. Imports and exports

Status: implemented on backend and web.

Evidence:

- API:
  - `ImportController.java`
  - `ExportController.java`
- Frontend:
  - `frontend/src/content/own/Imports/index.tsx`
  - export slices

Implemented capabilities appear to include:

- importing core entities from spreadsheet/CSV-style templates
- exporting work orders, assets, locations, parts, meters, and PMs
- localized import templates

Partial or missing:

- mobile import/export
- user-friendly import validation preview should be verified

Future improvements:

- import preview and mapping UI
- background import progress
- import rollback
- richer export scheduling
- Persian/Jalali export formatting

### 19. Notifications and messaging

Status: implemented.

Evidence:

- API:
  - `NotificationController.java`
  - push notification token model
  - email services/templates
- Frontend:
  - notification UI in header
- Mobile:
  - `mobile/screens/NotificationsScreen.tsx`
  - Expo Notifications setup

Implemented capabilities appear to include:

- in-app notifications
- email notifications
- push notifications on mobile
- PM notification jobs

Future improvements:

- notification preferences by event type
- digest emails
- notification delivery audit
- escalation notifications
- Slack/Teams integration

### 20. Integrations, API access, and webhooks

Status: implemented.

Evidence:

- API:
  - `ApiKeyController.java`
  - `WebhookController.java`
  - `WebhookEndpointController.java`
  - `zapier/ZapierSampleController.java`
- Frontend:
  - `frontend/src/content/own/Settings/Integrations/index.tsx`

Implemented capabilities appear to include:

- API keys
- webhook endpoints
- webhook dispatch
- Zapier sample payloads
- integration settings UI

Future improvements:

- more connector templates
- webhook retry dashboard
- webhook signing secrets
- outbound event audit
- inbound integration framework

### 21. Authentication, SSO, LDAP, subscription, and licensing

Status: implemented.

Evidence:

- API:
  - `AuthController.java`
  - `LicenseController.java`
  - `SubscriptionController.java`
  - `SubscriptionPlanController.java`
  - `PaddleController.java`
  - LDAP/OAuth configuration
- Frontend:
  - auth pages/routes
  - company plan screens
  - upgrade/downgrade screens
- Mobile:
  - auth screens
  - custom server screen

Implemented capabilities appear to include:

- login/register
- JWT auth
- OAuth/SSO hooks
- LDAP support
- subscription plans
- license state
- billing integration hooks

Future improvements:

- clearer free-vs-paid feature matrix
- SCIM
- SAML enterprise SSO if not already covered by generic OAuth/LDAP
- audit logs for admin/security events

## Gated or licensed features

Some features are implemented but gated.

Backend `PlanFeatures` includes:

- preventive maintenance
- checklists
- files
- purchase orders
- meters
- request configuration
- additional time
- additional cost
- analytics
- request portal
- signatures
- roles
- workflows
- API access
- webhooks
- import CSV

Backend `LicenseEntitlement` includes:

- SSO
- work order history
- workflows
- webhooks
- branding
- NFC/barcode
- custom roles
- file attachments
- time tracking
- cost tracking
- work order linking
- signature capture
- PM calendar
- condition-based PM
- asset hierarchy
- asset downtime
- low stock alerts
- parts cost tracking
- customer/vendor
- field configuration
- voice notes
- advanced analytics
- API access
- unlimited usage entitlements
- request portal

This means "implemented" does not always mean "available in a default Docker
Compose run." Some features need plan features, license entitlements, user
permissions, or UI configuration.

## Platform parity summary

| Module | API | Web app | Mobile app | Notes |
| --- | --- | --- | --- | --- |
| Work orders | Yes | Yes | Yes | Strong coverage. |
| Tasks/checklists | Yes | Yes | Yes | Mobile supports task/checklist selection. |
| Labor/time/costs | Yes | Yes | Yes | Mobile add-time/add-cost screens exist. |
| Preventive maintenance | Yes | Yes | Partial | No dedicated mobile PM module found. |
| Assets | Yes | Yes | Yes | Strong mobile support. |
| Locations | Yes | Yes | Yes | Mobile floor plans not found. |
| Parts/inventory | Yes | Yes | Yes | Strong core coverage. |
| Meters/readings | Yes | Yes | Yes | Mobile meter screens exist. |
| Requests | Yes | Yes | Yes | Public portal is web. |
| Purchase orders | Yes | Yes | Partial/No | Mobile slice exists, no dedicated mobile PO screens found. |
| Vendors/customers | Yes | Yes | Yes | Mobile screens exist. |
| People/teams | Yes | Yes | Yes | Mobile screens exist. |
| Roles/permissions | Yes | Yes | Partial | Mobile has people/team flows, not full role admin. |
| Files | Yes | Yes | Partial | Mobile entity file tabs, no global file hub found. |
| Categories | Yes | Yes | No/Partial | Admin-heavy web feature. |
| Custom fields | Yes | Yes | Consumes | Mobile consumes forms; no admin settings found. |
| Workflows | Yes | Yes | No/Partial | Mobile slice exists; no editor screens found. |
| Analytics | Yes | Yes | Partial | Mobile has limited stats. |
| Imports/exports | Yes | Yes | No | Web/admin feature. |
| Notifications | Yes | Yes | Yes | Push/in-app/email. |
| API keys/webhooks | Yes | Yes | No | Admin/integration web feature. |
| Licensing/subscriptions | Yes | Yes | Partial | Mobile has license slice/hook but limited global use. |

## What is partially implemented or needs clarification?

### Mobile preventive maintenance

Backend and web support PM, but mobile does not appear to provide a full PM
planner/editor.

Future decision:

- add PM to mobile if technicians need PM schedule visibility
- otherwise document mobile as work-order/request execution only

### Mobile purchase orders

Backend and web have purchase orders, but no dedicated mobile PO screens were
found.

Future decision:

- add mobile PO approval/receiving if field purchasing matters
- keep PO web-only if procurement is office/admin-only

### Mobile workflows

Workflow slice exists, but no mobile workflow builder UI was found.

Future decision:

- web-only workflow admin is reasonable
- mobile could show workflow status or approval actions later

### Floor plans

Floor plan API and web-related usage exist, but mobile floor-plan UI was not
found.

Future decision:

- add mobile floor plan viewer for technicians
- add asset/location overlays

### Full analytics on mobile

Mobile has stats, but not web analytics parity.

Future decision:

- keep mobile lightweight
- or build mobile dashboard summaries for managers

### Licensing and feature availability

Some implemented features are locked in default Docker usage.

Future decision:

- document free-vs-paid clearly
- align frontend feature gates with backend license gates
- improve admin diagnostics for license state

## What should be implemented in the future?

### Priority 1: product clarity and quality foundation

These should come before large new feature expansion:

1. Feature matrix
   - list each module
   - show API/web/mobile support
   - show plan/license requirements
   - show free-tier limits
2. Tests around core flows
   - work order lifecycle
   - PM generation
   - role/permission checks
   - imports/exports
   - license gates
3. Documentation
   - setup guide
   - admin guide
   - technician/mobile guide
   - integration guide
4. Security and audit hardening
   - public endpoint review
   - better error handling
   - admin audit logs

### Priority 2: Persian/Jalali support

Based on the user's current goal:

1. Persian language support
   - backend `FA`
   - message bundles
   - frontend `fa.ts`
   - mobile `fa.ts`
   - RTL testing
2. Calendar system preference
   - `GREGORIAN`
   - `JALALI`
3. Central date formatting
   - backend reports/exports
   - frontend displays and pickers
   - mobile displays and pickers
4. Persian digits decision
   - Latin digits or Persian digits

### Priority 3: mobile parity where it matters

Add only mobile features that field users actually need:

- mobile PM schedule visibility
- mobile PM completion flow improvements
- mobile purchase order approval/receiving if needed
- mobile floor plan viewer
- better offline behavior for work orders, assets, parts, and readings
- mobile RTL support

### Priority 4: advanced CMMS/EAM modules

Potential future modules:

- calibration management
- warranty tracking
- safety permits / lockout-tagout
- inspection rounds
- SLA/escalation rules
- contractor portal
- storerooms and stock transfers
- barcode label printing
- predictive maintenance / sensor ingestion
- IoT meter readings
- budgeting and cost centers
- compliance documents
- audit trails for configuration changes

### Priority 5: integrations and automation

Future integration improvements:

- webhook retry dashboard
- webhook signing secrets
- Zapier/Make connector docs
- Microsoft Teams / Slack notifications
- SCIM user provisioning
- SAML SSO if required
- accounting/ERP integration
- email-to-request ingestion

## What should not be implemented immediately?

Avoid adding these before the foundation is stronger:

- a full rewrite
- too many mobile admin screens
- advanced AI/predictive maintenance before data quality is good
- complex offline sync before deciding offline product requirements
- deep Jalali calendar changes without a central date design
- new licensed features without a clearer feature matrix

## Suggested roadmap phases

### Phase 1: stabilize and document

- create feature matrix
- document plan/license gates
- add smoke tests
- add work order lifecycle tests
- add PM generation tests
- document Docker/source image build
- clarify free vs paid behavior

### Phase 2: Persian language

- add `FA` backend language
- add Persian message bundles
- add frontend/mobile Persian translations
- add RTL support where missing
- update docs and tests

### Phase 3: Jalali calendar

- add `calendarSystem`
- centralize date formatting
- update reports/exports
- update frontend date displays/pickers
- update mobile date displays/pickers

### Phase 4: mobile field experience

- improve offline strategy
- add PM visibility
- add floor plan viewer
- improve scanning flows
- add mobile manager dashboard only if needed

### Phase 5: enterprise hardening

- audit logs
- SCIM/SAML if required
- webhook observability
- advanced security review
- backup/restore docs
- performance testing

## Strategic conclusion

This project already implements a substantial CMMS. It is much closer to a real
product than a starter template.

What is implemented:

- most core CMMS modules
- rich web administration
- operational mobile app
- integrations and licensing
- reports, analytics, imports, exports

What is not fully implemented:

- mobile parity for admin/planner modules
- first-class offline sync
- Persian/Jalali support
- some advanced EAM modules
- comprehensive tests and product documentation

The best future direction is not to rebuild the CMMS from scratch. The best
direction is to:

1. document exactly what exists
2. add tests around core behavior
3. improve licensing/feature-gate clarity
4. implement Persian language support
5. design Jalali calendar support carefully
6. add mobile parity only where field users need it

