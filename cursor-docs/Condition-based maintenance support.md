# Condition-Based Maintenance Support

This document explains whether this CMMS supports condition-based maintenance
(CBM), what level of CBM is currently available, and what would still be needed
for a mature CBM or predictive maintenance program.

It is written for product, maintenance, and engineering stakeholders.

## Short answer

Yes, this project supports a basic form of condition-based maintenance.

The implemented CBM model is:

```text
Meter reading crosses a configured threshold
  -> notify assigned users
  -> create a work order from a predefined work order template
  -> dispatch a webhook event
```

This is best described as:

```text
threshold-based CBM using meter readings
```

It is not yet a full predictive maintenance platform.

The project does not appear to include mature features such as:

- trend-based failure prediction
- anomaly detection
- remaining useful life calculations
- IoT sensor ingestion pipeline
- multi-sensor condition rules
- hysteresis / deadband rules
- one-open-work-order-per-alarm suppression
- advanced condition dashboards

## What CBM means in this project

In maintenance language, CBM means work is triggered by equipment condition
instead of only by calendar time.

In this project, the condition signal is a meter reading.

Examples:

- compressor temperature is greater than 80 C
- tank level is less than 20%
- vibration reading is greater than a threshold
- pressure is lower than expected
- operating hours pass a configured level

When a new reading is entered and the threshold condition is met, the system can
create a work order automatically.

## CBM capability level

| Capability | Current support |
| --- | --- |
| Manual meter readings | Supported |
| Meter update frequency | Supported |
| Threshold trigger: greater than | Supported |
| Threshold trigger: less than | Supported |
| Automatic work order creation | Supported |
| Notification on trigger | Supported |
| Webhook on trigger | Supported |
| Mobile reading entry | Supported |
| Mobile trigger setup | Not fully supported |
| Multi-condition rules | Not supported |
| Predictive analytics | Not supported |
| IoT/sensor ingestion | Not clearly implemented |
| Alarm suppression / one open WO per trigger | Not clearly implemented |
| Hysteresis / deadband | Not supported |

## Functional CBM workflow

The current condition-based maintenance workflow is:

1. A maintenance administrator creates a meter for an asset.
2. The meter has a unit and expected reading frequency.
3. A meter trigger is configured.
4. The trigger defines:
   - a trigger name
   - greater-than or less-than condition
   - threshold value
   - work order details to create if triggered
5. A technician or user records a new meter reading.
6. The system checks the reading against all triggers for that meter.
7. If the reading crosses a trigger threshold:
   - assigned users are notified
   - a work order is created
   - a webhook event is sent

## Example scenario

### Scenario: motor temperature too high

1. Asset: Main production motor.
2. Meter: Temperature.
3. Unit: C.
4. Trigger condition: greater than 80.
5. Triggered work order template:
   - title: Inspect motor overheating
   - priority: High
   - assigned team: Maintenance
   - checklist: inspect cooling fan, inspect bearing, check lubrication
6. Technician records reading: 85.
7. The system creates a work order.

This is a valid CBM workflow.

## Implemented CBM building blocks

### 1. Meters

Meters represent measurable asset conditions or counters.

Functional examples:

- runtime hours
- temperature
- pressure
- vibration
- level
- cycle count
- energy consumption

The project supports meters connected to assets and readings recorded over time.

### 2. Readings

Readings are numeric values recorded against meters.

The system also supports an update frequency. This means the system can restrict
how often readings are entered based on the meter's configured frequency.

Functional value:

- encourages regular inspections
- prevents duplicate reading entry too soon
- provides historical condition values

### 3. Meter triggers

Meter triggers are the main CBM feature.

Supported trigger conditions:

```text
LESS_THAN
MORE_THAN
```

That means the system can trigger a work order when a reading is:

- below a threshold
- above a threshold

### 4. Triggered work order template

A meter trigger includes work order configuration.

That allows the system to create a useful work order automatically, not just an
alert.

The generated work order can carry normal work order details such as:

- title
- description
- priority
- asset
- location
- assigned users
- team
- category
- tasks or custom fields, depending on configuration

### 5. Notifications

When a meter trigger fires, the system notifies assigned users.

Functional value:

- technicians know action is required
- managers can react to abnormal equipment conditions
- condition alarms become actionable work

### 6. Webhook event

The system also dispatches a webhook event when a meter trigger fires.

Functional value:

- external systems can receive condition events
- integrations can notify other tools
- automation platforms can react to CBM events

## What is supported on web

The web application appears to support:

- creating meters
- viewing meter details
- adding meter readings
- viewing reading history
- creating work order triggers for meters
- editing and deleting meter triggers
- seeing trigger thresholds on meter details

The web app is the main place for configuring CBM.

## What is supported on mobile

The mobile app appears to support:

- viewing meters
- viewing meter details
- adding readings
- seeing trigger information
- viewing reading history

Mobile appears more focused on field execution than administration.

Mobile limitations:

- creating/editing/deleting meter triggers does not appear fully supported
- mobile CBM planning/admin is weaker than web

This is acceptable if technicians only need to record readings and respond to
work orders. It is not enough if supervisors need to manage CBM rules from the
field.

## Licensing and availability

CBM is a gated feature.

Important gates:

- meter usage depends on meter-related plan features
- creating meter triggers requires the `CONDITION_BASED_PM` license entitlement

Practical meaning:

```text
The code supports CBM, but a default Docker run without the right license may
not allow creating condition-based meter triggers.
```

This matters because a user may see meters and readings but still be blocked
from creating CBM triggers.

## Difference between PM and CBM in this project

The project supports both preventive maintenance and condition-based maintenance,
but they are implemented as separate concepts.

### Preventive maintenance

Preventive maintenance is schedule-based.

Examples:

- inspect pump every 30 days
- service HVAC every quarter
- generate work order every Monday
- generate next PM after previous work order is completed

### Condition-based maintenance

Condition-based maintenance is reading-based.

Examples:

- generate a work order when temperature is greater than 80
- generate a work order when oil level is less than 30
- generate a work order when meter reading crosses a threshold

Current CBM is not the same as the normal recurring PM scheduler. It is a
separate meter-trigger workflow.

## What CBM does not yet cover

### 1. Predictive maintenance

Predictive maintenance tries to forecast failures before a threshold is crossed.

Examples:

- bearing failure probability
- remaining useful life
- vibration trend analysis
- anomaly detection
- failure prediction from historical data

This project does not appear to implement predictive maintenance today.

### 2. Multi-signal condition rules

Mature CBM often needs rules like:

```text
temperature > 80 AND vibration > 5 for more than 10 minutes
```

or:

```text
pressure below threshold OR flow rate decreasing rapidly
```

Current support appears limited to one meter reading and one threshold condition.

### 3. Hysteresis and deadband

Without deadband, systems can repeatedly trigger when readings hover around a
threshold.

Example:

- threshold: temperature > 80
- readings: 81, 79, 82, 78, 83

A mature CBM system may need:

- trigger at 80
- clear only below 75
- do not create duplicate work orders while alarm remains active

This behavior is not clearly implemented.

### 4. Alarm lifecycle

A mature CBM system usually tracks condition alarm states:

- normal
- warning
- alarm
- acknowledged
- work order created
- resolved
- closed

This project creates work orders from threshold crossings, but it does not appear
to provide a full alarm-state lifecycle.

### 5. Duplicate work order suppression

If every high reading creates a new work order, teams can get duplicate work.

A mature CBM system should decide:

- should one open work order suppress new work orders?
- should repeated readings update the existing work order?
- should repeated alarms increase severity?
- should there be a cooldown period?

The model contains concepts named `recurrent` and `waitBefore`, but they do not
appear fully surfaced or enforced in the current trigger firing behavior.

### 6. Sensor ingestion

The current model works well for manual or API-entered readings.

It does not appear to include a complete IoT ingestion layer for:

- MQTT
- OPC-UA
- SCADA integration
- sensor gateways
- high-frequency telemetry
- streaming anomaly detection

This would be future work if the organization wants industrial CBM.

## Current CBM maturity level

Functional maturity estimate:

```text
Level 1: Manual meter tracking                     Supported
Level 2: Threshold-based work order triggers       Supported
Level 3: Alarm lifecycle and duplicate suppression Partial / unclear
Level 4: Multi-condition CBM rules                 Not supported
Level 5: IoT condition monitoring                  Not supported
Level 6: Predictive maintenance                    Not supported
```

So the product supports basic CBM, not advanced CBM or predictive maintenance.

## CBM requirements that are already covered

For many maintenance teams, the current system may be enough if requirements are:

- record meter readings manually
- define high/low limits
- create work orders when limits are crossed
- notify responsible users
- keep reading history
- connect readings to assets
- use mobile for reading entry
- send external webhook events

This is a practical starting point.

## CBM requirements that should be added in the future

### Priority 1: make current CBM safer

Recommended improvements:

- prevent duplicate work orders for the same trigger while an existing one is open
- expose and enforce cooldown/wait time
- clarify whether triggers are recurrent or one-time
- add trigger history
- add alarm acknowledgement
- show why a work order was created
- improve license messaging when CBM is not available

### Priority 2: improve mobile CBM

Recommended improvements:

- let supervisors create/edit triggers on mobile if needed
- improve mobile reading routes
- support offline reading capture
- sync readings later when online
- show triggered work orders clearly on mobile

### Priority 3: expand condition logic

Recommended improvements:

- equals / between / outside range conditions
- rate-of-change conditions
- rolling average conditions
- duration-in-alarm conditions
- multiple meter conditions
- warning and critical thresholds

### Priority 4: improve analytics

Recommended improvements:

- meter trend charts
- readings over time
- alarm frequency by asset
- top assets by condition alarms
- CBM-generated work order reports
- cost of CBM-triggered work
- downtime prevented by CBM

### Priority 5: support sensor integration

Recommended improvements:

- API endpoint or ingestion service for external sensors
- MQTT or gateway integration
- bulk reading ingestion
- validation for abnormal readings
- sensor health monitoring
- integration with SCADA or industrial systems

### Priority 6: move toward predictive maintenance

Recommended improvements:

- trend analysis
- anomaly detection
- asset failure patterns
- remaining useful life estimation
- maintenance recommendation engine
- model validation dashboards

This should come after good data quality and consistent meter readings exist.

## Functional recommendation

If the question is:

```text
Does this CMMS support CBM?
```

The answer is:

```text
Yes, basic threshold-based CBM is supported.
```

If the question is:

```text
Does this CMMS support mature condition monitoring or predictive maintenance?
```

The answer is:

```text
Not yet.
```

## Practical decision for maintenance teams

Use the current CBM feature when:

- equipment has simple high/low thresholds
- readings are manual or low-frequency
- a threshold crossing should create a work order
- mobile users need to enter readings
- webhook integration is useful

Do not treat the current CBM feature as enough when:

- you need real-time sensor monitoring
- alarms need lifecycle management
- repeated readings must not create duplicate work
- you need trend-based predictions
- you need multi-condition industrial rules
- you need predictive maintenance algorithms

## Final conclusion

The project has a real CBM foundation:

- meters
- readings
- high/low threshold triggers
- automatic work order creation
- notifications
- webhooks
- mobile reading entry

This is valuable and practical for many maintenance teams.

The current CBM should be considered an early-to-mid maturity feature. The next
step is to harden trigger behavior, prevent duplicate work orders, improve mobile
support, and add richer condition logic before attempting predictive maintenance.
