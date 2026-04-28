# CMMS Functional Requirements for Maintenance Teams

This document is written for maintenance, facilities, operations, and asset
management stakeholders. It avoids technical implementation details and focuses
on what the CMMS does functionally, what appears covered, what appears partial,
and what should be considered for the future.

## Short answer

This product already covers a large portion of normal CMMS needs.

It appears strongest for:

- managing work orders
- planning preventive maintenance
- tracking assets and locations
- managing parts and inventory
- recording labor, time, and costs
- handling service requests
- managing users, teams, and permissions
- reporting on maintenance performance
- supporting technicians through a mobile app

It appears weaker or incomplete for:

- advanced mobile planning features
- robust offline field work
- advanced asset management / EAM functions
- calibration management
- warranty management
- safety permit / lockout-tagout workflows
- contractor portals
- advanced inventory and purchasing controls
- Persian language and Jalali calendar support
- clear product packaging of free vs licensed features

Overall, this is a serious CMMS foundation. It is not just a work order tracker.
For many organizations, it can be used as a starting point and improved over
time.

## Who this CMMS serves

The system appears designed for these groups:

### Maintenance managers

They need to:

- plan and assign work
- monitor workload
- review overdue and completed work
- track compliance
- understand maintenance cost
- manage technicians and teams
- configure categories, forms, checklists, and workflows

### Technicians

They need to:

- see assigned work
- update work status
- complete checklists
- log labor/time
- add costs and parts
- attach photos/files
- scan assets by barcode or NFC
- close work orders from the field

### Requesters

They need to:

- submit maintenance requests
- provide location and asset information
- attach files or photos
- follow request status

### Asset and facilities teams

They need to:

- maintain an asset register
- organize locations
- track downtime
- understand maintenance history
- manage parts and inventory
- track meters and readings

### Administrators

They need to:

- configure company preferences
- manage users, teams, roles, and permissions
- set up request portals
- configure forms and fields
- manage integrations
- control feature access and licenses

## Functional coverage summary

| CMMS area | Current functional coverage | Notes |
| --- | --- | --- |
| Work orders | Strong | Core work management is well represented. |
| Preventive maintenance | Strong on web, partial on mobile | Planning exists, mobile PM planning appears limited. |
| Asset management | Strong | Includes equipment, downtime, files, history, and scanning support. |
| Locations | Strong | Includes location management and mapping/floor-plan direction. |
| Inventory and parts | Strong core coverage | Future enhancements can deepen storeroom and purchasing behavior. |
| Meters and readings | Good | Meter-triggered maintenance appears supported. |
| Service requests | Good | Includes request intake and public portal direction. |
| Purchase orders | Good on web, limited on mobile | Mobile PO workflows appear incomplete. |
| Vendors/customers | Good | Useful for service providers and supplier tracking. |
| People and teams | Good | Users, teams, roles, permissions are present. |
| Files/documents | Good | Attachments and file hub exist; document control can improve. |
| Checklists | Good | Reusable checklists and work order tasks are present. |
| Workflows | Good on web/admin | Future work can improve workflow visibility and audit. |
| Analytics/reports | Good on web, partial on mobile | Web reporting is much richer than mobile. |
| Mobile field work | Medium to good | Good for technicians; less complete for planners/admins. |
| Offline work | Limited/unclear | Needs clearer product definition. |
| Enterprise/EAM | Partial | Some foundations exist, but advanced modules are not complete. |

## Implemented functional areas

### 1. Work order management

Work orders are the heart of a CMMS, and this project appears to implement them
well.

Covered capabilities:

- create work orders
- view work order lists
- filter and search work orders
- assign work to people, teams, or service providers
- set priority and status
- connect work orders to assets and locations
- attach tasks and checklists
- attach files and photos
- log labor or time
- record additional costs
- consume inventory parts
- view work order history
- view work orders in a calendar
- generate work order reports
- close work orders
- capture signatures before closing where enabled
- use mobile screens for field work

Functional value:

- dispatchers can organize maintenance work
- technicians can execute and update tasks
- managers can review workload and completion
- history is preserved for assets and audits

Future functional needs:

- clearly documented work order lifecycle
- configurable approval steps
- SLA rules and escalation
- better mobile offline completion
- stronger distinction between planned, corrective, emergency, and inspection
  work
- richer close-out requirements by work type

### 2. Preventive maintenance

Preventive maintenance is present and appears meaningful.

Covered capabilities:

- create preventive maintenance schedules
- define recurring work
- configure the work order that will be created
- use due dates and calendar planning
- connect preventive work to assets, locations, meters, and tasks
- use meter-based triggers for condition-based maintenance
- report on preventive maintenance outcomes

Functional value:

- maintenance teams can move from reactive work to planned work
- planners can define repeated activities
- meter readings can trigger maintenance when thresholds are crossed

Partial or missing:

- mobile preventive maintenance planning appears limited
- long-range PM forecasting could be improved
- shutdown/outage planning is not clearly represented
- PM optimization based on history is not clearly represented

Future functional needs:

- PM forecast calendar by week/month/quarter
- mobile PM schedule visibility
- bulk PM generation preview
- route-based PM rounds
- PM compliance dashboards
- optimization of intervals based on failures or meter trends

### 3. Asset and equipment management

Asset management appears strong.

Covered capabilities:

- create and manage assets/equipment
- categorize assets
- connect assets to locations
- assign assets to users or teams
- view asset work history
- track asset downtime
- attach files and images
- connect parts to assets
- scan assets through barcode or NFC where enabled
- analyze asset reliability and cost

Functional value:

- teams can build an equipment register
- maintenance history can be traced by asset
- downtime and cost can be analyzed
- field technicians can identify assets quickly

Future functional needs:

- asset hierarchy tree visualization
- asset criticality ranking
- warranty tracking
- calibration management
- asset condition scoring
- replacement planning
- lifecycle costing
- asset risk matrix
- commissioning/decommissioning process

### 4. Location and facility management

Location management is present and useful.

Covered capabilities:

- create and manage locations
- connect assets to locations
- connect work orders to locations
- attach files to locations
- view location-related work
- map/location support
- floor plan direction exists in the product

Functional value:

- facilities teams can structure maintenance by building, floor, room, or site
- technicians can understand where work must happen
- managers can analyze maintenance by location

Partial or missing:

- mobile floor-plan usage appears limited
- advanced multi-site hierarchy should be verified in real use

Future functional needs:

- mobile floor plan viewer
- interactive asset placement on floor plans
- QR code wayfinding
- building/area/room hierarchy reports
- location condition inspections

### 5. Parts and inventory

Parts and inventory are meaningfully represented.

Covered capabilities:

- create and manage parts
- categorize parts
- track quantities
- connect parts to assets
- consume parts on work orders
- create sets of parts
- attach files to parts
- receive low-stock notifications where enabled
- import and export inventory data

Functional value:

- technicians can identify required spare parts
- maintenance cost can include consumed parts
- inventory teams can track basic stock
- managers can see part consumption

Future functional needs:

- storerooms and bin locations
- stock transfers between stores
- reorder points and reorder quantities
- automatic purchase suggestions
- vendor pricing and lead times
- cycle counts
- inventory valuation
- barcode label printing
- reserved stock for planned work
- part substitutions and equivalents

### 6. Meters and readings

Meter management appears implemented.

Covered capabilities:

- create meters/counters
- record readings
- define reading units
- connect meters to assets
- trigger work orders based on readings
- manage meters on mobile

Functional value:

- teams can move from calendar-based PM to usage-based PM
- equipment can be serviced based on actual utilization
- readings can support reliability analysis

Future functional needs:

- bulk meter reading entry
- mobile reading routes
- IoT/sensor integrations
- reading anomaly detection
- charts and trends per meter
- forecast next service date based on usage rate

### 7. Service requests and request portal

Request management appears implemented.

Covered capabilities:

- create service/intervention requests
- configure request fields
- approve or reject requests
- automatically create work orders after approval
- public request portal for request submission
- request analytics
- mobile request screens

Functional value:

- non-maintenance users can report issues
- administrators can review and approve work before dispatch
- request history can be tracked
- request intake can be standardized

Future functional needs:

- requester communication thread
- request SLA tracking
- request escalation rules
- requester satisfaction survey
- portal branding and access control
- requester status notifications
- QR-code request submission per location/asset

### 8. Purchase orders and procurement

Purchase orders appear implemented for the web product.

Covered capabilities:

- create purchase orders
- categorize purchase orders
- approve or reject purchase orders
- connect purchasing to inventory and vendors conceptually

Functional value:

- maintenance teams can manage procurement for parts and services
- approvals can control spending

Partial or missing:

- mobile purchase order workflows appear limited
- receiving goods into inventory should be verified
- deeper procurement controls are not clearly complete

Future functional needs:

- purchase requisitions
- goods receiving
- partial receiving
- vendor quotes
- vendor price history
- budget controls
- cost center approval
- purchase order matching against invoices
- mobile PO approvals

### 9. Vendors, customers, and service providers

Vendor and customer records are present.

Covered capabilities:

- create vendors/service providers
- create customers
- view and edit vendor/customer details
- connect service providers to work
- mobile vendor/customer screens

Functional value:

- external contractors and suppliers can be tracked
- maintenance history can reference service providers
- customer-facing service teams can manage customer context

Future functional needs:

- vendor contracts
- insurance/compliance documents
- preferred vendor lists
- contractor performance scoring
- contractor portal
- vendor service-level agreements

### 10. People, teams, roles, and permissions

People and permissions are implemented.

Covered capabilities:

- create users
- invite users
- group users into teams
- assign work to teams
- define roles and permissions
- manage profiles and settings
- use mobile people/team screens

Functional value:

- managers can organize the maintenance workforce
- technicians can receive assigned work
- permissions can separate admin, planner, technician, and requester behavior

Important product note:

- custom role creation is licensed/gated, so availability depends on the plan
  and license.

Future functional needs:

- clearer role templates
- permission audit
- approval authority levels
- skill/certification tracking
- labor availability calendar
- SCIM or enterprise user provisioning

### 11. Checklists and inspections

Checklists are implemented as reusable work instructions.

Covered capabilities:

- reusable checklists
- configurable tasks
- adding checklists to work orders
- completing tasks during work execution
- mobile task/checklist usage

Functional value:

- standardizes technician work
- improves compliance
- reduces missed steps
- supports repeatable PM activities

Future functional needs:

- conditional checklist questions
- pass/fail inspection scoring
- numeric readings inside checklist tasks
- required photo evidence
- checklist revision control
- compliance reports
- inspection rounds

### 12. Configurable forms and custom fields

Configurable forms are a strong product feature.

Covered capabilities:

- configure work order fields
- configure request fields
- mark fields optional, mandatory, or hidden
- define custom fields
- use dynamic forms in operational flows

Functional value:

- different organizations can adapt the CMMS without custom development
- request and work order forms can match local processes
- unnecessary fields can be hidden

Future functional needs:

- conditional fields
- custom validation rules
- calculated fields
- field visibility by role or work type
- custom field reporting
- form templates by industry

### 13. Workflows and automation

Workflow automation appears implemented for administrators.

Covered capabilities:

- define simple if/and/then workflows
- customize behavior according to team process
- gate workflow features by plan/license

Functional value:

- teams can automate repeated decisions
- admins can adapt the system without code
- workflow rules can support escalation, assignment, or notifications

Future functional needs:

- workflow templates
- workflow execution history
- workflow simulation/test mode
- approval workflow builder
- visual workflow designer
- mobile visibility of workflow status

### 14. Analytics, reports, and dashboards

Analytics and reporting appear strong on web.

Covered capabilities:

- work order status analysis
- compliance rate
- average cycle time
- work order age
- cost analysis
- labor/time analysis
- asset reliability
- downtime and availability
- part consumption
- request analysis
- PDF reports
- exports

Functional value:

- managers can understand maintenance performance
- costs can be tracked by asset, work order, and part consumption
- reliability issues can be identified
- request trends can be monitored

Partial or missing:

- mobile analytics are lighter than web
- custom dashboard builder is not clearly present
- scheduled reports are not clearly present

Future functional needs:

- custom dashboards
- scheduled email reports
- KPI targets
- SLA dashboards
- maintenance backlog forecasting
- downtime Pareto analysis
- mean time between failure and mean time to repair
- exportable dashboard definitions

### 15. Imports and exports

Imports and exports are implemented for administrative use.

Covered capabilities:

- import core data from spreadsheet-style templates
- export work orders, assets, locations, parts, meters, and PMs
- localized import templates

Functional value:

- organizations can migrate existing data
- managers can analyze data externally
- bulk onboarding is easier

Future functional needs:

- import preview before commit
- field mapping UI
- validation report before import
- rollback failed imports
- scheduled exports
- Persian/Jalali formatting in exported reports

### 16. Notifications

Notifications are implemented across the product.

Covered capabilities:

- in-app notifications
- email notifications
- mobile push notifications
- preventive maintenance reminders
- notifications related to work changes

Functional value:

- users are informed about assignments and updates
- managers can reduce missed work
- mobile technicians can receive alerts in the field

Future functional needs:

- per-event notification preferences
- digest emails
- notification escalation
- delivery audit
- Slack or Microsoft Teams notifications
- quiet hours

### 17. Integrations

Integration foundations are present.

Covered capabilities:

- API access
- API keys
- webhooks
- sample payloads for automation tools
- integration settings

Functional value:

- the CMMS can connect to other systems
- external workflows can react to maintenance events
- advanced customers can automate around CMMS data

Future functional needs:

- webhook retry dashboard
- webhook signing secrets
- connector templates
- ERP/accounting integration
- email-to-request creation
- SCIM provisioning
- Make/Zapier documentation

### 18. Mobile field work

The mobile app appears useful for field execution.

Covered capabilities:

- work order list and details
- create/edit work orders
- complete work orders
- tasks/checklists
- add time and cost
- assets, locations, parts, meters
- requests
- people/teams
- vendors/customers
- notifications
- barcode/NFC scanning
- custom server configuration

Functional value:

- technicians can work away from a desk
- asset identification is easier
- field updates can be captured closer to real time

Partial or missing:

- full mobile preventive maintenance planning
- mobile purchase orders
- mobile workflow administration
- full mobile analytics
- first-class offline work
- RTL layout for Persian/Arabic

Future functional needs:

- offline work order execution
- offline asset and part lookup
- sync conflict handling
- mobile PM schedule view
- mobile floor plan viewer
- mobile PO approval/receiving
- mobile manager dashboard if needed

## Functional gaps by CMMS maturity level

### Basic CMMS

Mostly covered:

- work orders
- assets
- locations
- preventive maintenance
- parts
- requests
- users and teams
- reports

### Intermediate CMMS

Partly to strongly covered:

- configurable forms
- checklists
- mobile execution
- purchase orders
- meters
- analytics
- integrations
- request portal
- workflows

### Advanced CMMS / EAM

Partially covered or future work:

- asset criticality and risk scoring
- warranty management
- calibration management
- safety permits and lockout-tagout
- contractor portal
- advanced inventory planning
- full procurement cycle
- offline-first mobile
- advanced reliability engineering
- predictive maintenance
- IoT integrations
- compliance document management
- audit trails for configuration changes

## Recommended future roadmap for maintenance stakeholders

### Priority 1: clarify current product scope

Create a simple functional feature matrix:

- module name
- available or not
- web support
- mobile support
- licensed or free
- user roles that can use it
- known limitations

This is important because the product is broad and some features are gated.

### Priority 2: improve maintenance execution reliability

Focus on:

- work order lifecycle clarity
- required close-out fields
- task/checklist compliance
- mobile completion flow
- technician notifications
- better history and audit trails

### Priority 3: complete Persian language and calendar support

For Persian users:

- translate the application
- support right-to-left layout
- decide on Persian or Latin digits
- support Jalali dates in displays and inputs
- support Jalali dates in reports and exports
- verify mobile RTL behavior

### Priority 4: strengthen mobile field operations

Field teams need reliability more than admin complexity.

Recommended mobile improvements:

- offline work order access
- offline completion and later sync
- PM schedule visibility
- floor plan viewing
- barcode/NFC improvements
- mobile purchase order approvals if required

### Priority 5: improve planning and reliability management

Recommended planning improvements:

- PM forecast calendar
- maintenance backlog dashboard
- SLA and escalation rules
- downtime Pareto reports
- MTBF / MTTR reporting
- asset criticality ranking
- replacement planning

### Priority 6: deepen inventory and procurement

Recommended inventory/procurement improvements:

- storerooms
- bin locations
- stock transfers
- reorder suggestions
- purchase requisitions
- receiving
- vendor price history
- inventory valuation

### Priority 7: enterprise and compliance modules

Future enterprise features:

- calibration
- warranty
- safety permit / lockout-tagout
- contractor portal
- compliance document tracking
- audit logs
- advanced SSO/provisioning

## Functional conclusion

From a CMMS perspective, this project is already a broad maintenance management
system.

It covers most core maintenance functions:

- plan work
- receive requests
- manage assets
- assign technicians
- execute work orders
- consume parts
- track labor and costs
- schedule preventive work
- report on performance

The biggest functional gaps are not basic CMMS capabilities. The biggest gaps
are maturity gaps:

- clearer product packaging
- deeper mobile field reliability
- advanced planning and reliability engineering
- advanced inventory/procurement
- enterprise compliance modules
- Persian/Jalali localization

For a maintenance organization, the best next step is not to rebuild the product.
The best next step is to validate the current workflows with real maintenance
users, document what is available, and prioritize improvements based on actual
maintenance operations.

