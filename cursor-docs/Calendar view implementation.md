# Calendar view implementation

This document explains how the calendar view in this project is implemented end
to end. It focuses on the western (Gregorian) calendar that the application
ships today, where months are rendered as `January`, `February`, …, `April`,
`May`, etc.

The calendar view is the **Work Orders → Calendar** tab in the main React
frontend. It renders work orders and preventive maintenance occurrences as
events on a calendar, with month / week / day / agenda views, navigation
controls, and click-through to a work order or preventive maintenance.

It does **not** cover:

- The mobile app (`mobile/`) — it does not contain a FullCalendar-based view.
- The marketing site (`home/`).
- The Material UI `DateTimePicker` used elsewhere in forms.

## TL;DR

- The calendar UI is a single React component built on top of
  [FullCalendar](https://fullcalendar.io) v5.
- It lives at `frontend/src/content/own/WorkOrders/Calendar/`.
- It is mounted as one of the tabs of the Work Orders page
  (`frontend/src/content/own/WorkOrders/index.tsx`), selected via
  `currentTab === 'calendar'` or the `?view=calendar` URL parameter.
- Events are loaded from the backend endpoint `POST /work-orders/events` with a
  `{ start, end }` date range, returning an array of `CalendarEvent` objects
  that wrap either a `WorkOrder` or a `PreventiveMaintenance`.
- The Redux slice `workOrders.calendar.events` holds the events; the calendar
  view selects them via `useSelector` and converts each one into a
  FullCalendar `Event` via `getEventFromWO`.
- Month / day labels (`April`, `May`, …) come from two independent locale
  systems:
  - The header label `April 2026` is produced by `date-fns`'s `format(date,
    'MMMM yyyy', { locale })` using the `useDateLocale` hook.
  - The labels rendered inside the FullCalendar grid (column headers, list
    view headings, etc.) come from FullCalendar's own per-language locale
    bundles, loaded by `getCalendarLocale(lang)` from `frontend/src/i18n/i18n.ts`.

## Where the code lives

```text
frontend/src/
├── content/own/WorkOrders/
│   ├── index.tsx                       # Work Orders page, hosts the Calendar tab
│   └── Calendar/
│       ├── index.tsx                   # ApplicationsCalendar — the FullCalendar view
│       ├── Actions.tsx                 # Toolbar (prev/today/next + month label + view buttons)
│       ├── PageHeader.tsx              # Optional "Events" page header (not currently mounted)
│       └── EventDrawer.tsx             # Add/edit event drawer (legacy, used by the meetings slice)
├── slices/
│   ├── workOrder.ts                    # Calendar events thunk + state for work orders/PMs
│   └── calendar.ts                     # Generic meetings calendar slice (legacy/unused by WO calendar)
├── models/
│   └── calendar.ts                     # `Event` type and `View` union
├── hooks/
│   ├── useDateLocale.tsx               # Returns the active date-fns Locale
│   └── usePrevious.ts                  # Tiny helper used to detect view transitions
└── i18n/
    └── i18n.ts                         # Lazy translation, date-fns and FullCalendar locale loaders
```

Backend endpoint and DTOs:

```text
api/src/main/java/com/grash/
├── controller/WorkOrderController.java       # POST /work-orders/events
├── dto/CalendarEvent.java                    # { type, event, date }
├── dto/DateRange.java                        # { start, end }
├── dto/WorkOrderBaseMiniDTO.java             # Mini DTO returned to the frontend
└── service/PreventiveMaintenanceService.java # Computes recurring PM events from Quartz triggers
```

## High-level architecture

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ WorkOrders page (index.tsx)                                              │
│  Tabs: list | calendar | column                                          │
│                                                                          │
│   currentTab === 'calendar'                                              │
│       ↓                                                                  │
│   <WorkOrderCalendar                                                     │
│       handleAddWorkOrder={(date) => open create-WO modal at this date}   │
│       handleOpenDetails={(id, type) => open WO drawer or PM page}        │
│   />                                                                     │
└──────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ ApplicationsCalendar (Calendar/index.tsx)                                │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ Actions toolbar                                                  │    │
│  │   ◀  today  ▶      April 2026      [month][week][day][agenda]    │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ <FullCalendar>                                                   │    │
│  │   plugins: dayGrid, timeGrid, interaction, list                  │    │
│  │   locale: from getCalendarLocale(i18n.language)                  │    │
│  │   initialView: 'timeGridWeek'                                    │    │
│  │   events: from Redux state.workOrders.calendar.events            │    │
│  │   eventClick: handleOpenDetails(id, type)                        │    │
│  │   dateClick: handleAddWorkOrder(date)                            │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
              │                ▲
              │ dispatch       │ events
              ▼                │
┌──────────────────────────────────────────────────────────────────────────┐
│ Redux slice: workOrder.ts                                                │
│   getWorkOrderEvents(start, end) thunk                                   │
│     → POST /work-orders/events { start, end }                            │
│     → state.workOrders.calendar.events = response                        │
└──────────────────────────────────────────────────────────────────────────┘
              │                ▲
              ▼                │
┌──────────────────────────────────────────────────────────────────────────┐
│ Backend: WorkOrderController#getEvents                                   │
│   - Resolves the calling user via JWT                                    │
│   - Checks WORK_ORDERS view permission                                   │
│   - Aggregates two streams of events:                                    │
│       1. PM occurrences for [now, end] computed from Quartz triggers     │
│       2. Work orders with dueDate in [start, end]                        │
│   - Returns Collection<CalendarEvent<WorkOrderBaseMiniDTO>>              │
└──────────────────────────────────────────────────────────────────────────┘
```

## Frontend in depth

### Tab wiring on the Work Orders page

The Work Orders page exposes the calendar as a tab. The relevant pieces in
`frontend/src/content/own/WorkOrders/index.tsx`:

```70:70:frontend/src/content/own/WorkOrders/index.tsx
import WorkOrderCalendar from './Calendar';
```

```109:109:frontend/src/content/own/WorkOrders/index.tsx
  const [currentTab, setCurrentTab] = useState<string>('list');
```

```133:144:frontend/src/content/own/WorkOrders/index.tsx
  const tabs = [
    { value: 'list', label: t('list_view'), disabled: false },
    {
      value: 'calendar',
      label: t('calendar_view'),
      disabled: !hasViewPermission(PermissionEntity.WORK_ORDERS)
    },
    { value: 'column', label: t('column_view'), disabled: true }
  ];
  const handleTabsChange = (_event: ChangeEvent<{}>, value: string): void => {
    setCurrentTab(value);
  };
```

The calendar tab is hidden when the user lacks `WORK_ORDERS` view permission
(`hasViewPermission(PermissionEntity.WORK_ORDERS)`).

The page can also be deep-linked to the calendar via the `?view=calendar`
query param:

```327:329:frontend/src/content/own/WorkOrders/index.tsx
    if (viewParam === 'calendar') {
      setCurrentTab('calendar');
    }
```

When the calendar tab is active, the list-view filter bar is suppressed and
`<WorkOrderCalendar>` is rendered with two callbacks:

```1003:1012:frontend/src/content/own/WorkOrders/index.tsx
              <WorkOrderCalendar
                handleAddWorkOrder={(date: Date) => {
                  setInitialDueDate(date);
                  setOpenAddModal(true);
                }}
                handleOpenDetails={(id, type) => {
                  if (type === 'WORK_ORDER') handleOpenDetails(id);
                  else navigate(getPreventiveMaintenanceUrl(id));
                }}
              />
```

- `handleAddWorkOrder(date)` opens the existing **create work order** modal
  pre-filled with the clicked date as the due date.
- `handleOpenDetails(id, type)` either opens the work order side drawer (for
  `WORK_ORDER` events) or navigates to the corresponding preventive
  maintenance page (for `PREVENTIVE_MAINTENANCE` events).

### `ApplicationsCalendar` — the FullCalendar view

`frontend/src/content/own/WorkOrders/Calendar/index.tsx` is the main calendar
component. The interesting parts:

#### Imports and plugins

FullCalendar is split into several packages, each contributing a *plugin* that
enables a different view or behavior:

```1:8:frontend/src/content/own/WorkOrders/Calendar/index.tsx
import { useEffect, useRef, useState } from 'react';
import frLocale from '@fullcalendar/core/locales/fr';
import enLocale from '@fullcalendar/core/locales/en-gb';
import FullCalendar, { LocaleSingularArg } from '@fullcalendar/react';
import dayGridPlugin from '@fullcalendar/daygrid';
import timeGridPlugin from '@fullcalendar/timegrid';
import interactionPlugin from '@fullcalendar/interaction';
import listPlugin from '@fullcalendar/list';
```

Versions (from `frontend/package.json`):

```text
"@fullcalendar/core": "5.11.0",
"@fullcalendar/daygrid": "5.11.0",
"@fullcalendar/interaction": "5.11.0",
"@fullcalendar/list": "5.11.0",
"@fullcalendar/react": "5.11.1",
"@fullcalendar/timegrid": "5.11.0"
```

Plugin → view mapping:

| Plugin                    | Provides                                                                       |
|---------------------------|--------------------------------------------------------------------------------|
| `dayGridPlugin`           | The classic month grid view (`dayGridMonth`, the one with `April`, `May`, …). |
| `timeGridPlugin`          | The week / day time grid views (`timeGridWeek`, `timeGridDay`).                |
| `listPlugin`              | The vertical agenda list view (`listWeek`).                                    |
| `interactionPlugin`       | Date click, event click, drag & drop, etc.                                     |

Note: `frLocale` and `enLocale` are imported at the top but the runtime locale
is resolved through the i18n loader (`getCalendarLocale`) — these imports are
effectively dead but kept for ergonomics.

#### Component state

```141:158:frontend/src/content/own/WorkOrders/Calendar/index.tsx
function ApplicationsCalendar({
  handleAddWorkOrder,
  handleOpenDetails
}: OwnProps) {
  const theme = useTheme();
  const { i18n } = useTranslation();
  const calendarRef = useRef<FullCalendar | null>(null);
  const mobile = useMediaQuery(theme.breakpoints.down('md'));
  const dispatch = useDispatch();
  const { calendar, loadingGet } = useSelector((state) => state.workOrders);
  const [date, setDate] = useState<Date>(new Date());
  const [view, setView] = useState<View>('timeGridWeek');
  const getLanguage = i18n.language;
  const [calendarLocale, setCalendarLocale] = useState<LocaleSingularArg>(enGb);

  useEffect(() => {
    getCalendarLocale(i18n.language).then(setCalendarLocale);
  }, [i18n.language]);
```

Key local state:

- `calendarRef` — imperative handle to FullCalendar, used to call
  `calApi.next()`, `prev()`, `today()`, `changeView()`, and to read the
  currently displayed view's `activeStart` / `activeEnd`.
- `date` — the date currently centered in the calendar, used to drive the
  toolbar label and to re-fetch events when navigation happens.
- `view` — the currently selected `View`. The default is `'timeGridWeek'`
  (week view), not month view.
- `calendarLocale` — the FullCalendar locale object used to localize day
  headers and other in-grid labels. Initialized to British English (`enGb`)
  and asynchronously swapped via `getCalendarLocale(lang)` whenever the i18n
  language changes.
- `calendar.events` is selected from `state.workOrders` (see
  `slices/workOrder.ts`). `loadingGet` toggles a centered `CircularProgress`
  spinner while events are fetched.

The `View` type comes from `models/calendar.ts`:

```1:11:frontend/src/models/calendar.ts
export interface Event {
  id: string;
  allDay: boolean;
  color?: string;
  description: string;
  end: Date;
  start: Date;
  title: string;
}

export type View = 'dayGridMonth' | 'timeGridWeek' | 'timeGridDay' | 'listWeek';
```

These four view IDs are the four FullCalendar views the toolbar exposes:

- `dayGridMonth` → month grid (the "western" `April`, `May`, … grid)
- `timeGridWeek` → week with hour rows
- `timeGridDay` → single day with hour rows
- `listWeek` → list/agenda for the week

#### Mapping backend events to FullCalendar events

The backend sends an array of `CalendarEvent<WorkOrder | PreventiveMaintenance>`.
The component maps each one into a FullCalendar `Event`:

```181:198:frontend/src/content/own/WorkOrders/Calendar/index.tsx
  const getEventFromWO = (
    eventPayload: CalendarEvent<WorkOrder | PreventiveMaintenance>
  ): Event => {
    return {
      id: eventPayload.event.id.toString(),
      allDay: false,
      color:
        'status' in eventPayload.event &&
        eventPayload.event.status === 'COMPLETE'
          ? theme.colors.alpha.black[30]
          : getColor(eventPayload.event.priority),
      description: eventPayload.event?.description,
      end: new Date(eventPayload.date),
      start: new Date(eventPayload.date),
      title: eventPayload.event.title,
      extendedProps: { type: eventPayload.type }
    };
  };
```

Notable behaviors:

- Both `start` and `end` are set to the same date (`eventPayload.date`). The
  events are essentially point-in-time markers on the calendar, not durations.
- `allDay` is hard-coded to `false`, which means events appear in the time
  grid (week/day views) at their exact hour rather than as all-day stripes.
- The color is derived from the work order's `priority` via
  `getColor(priority)`, except completed work orders (`status === 'COMPLETE'`)
  which are rendered in a muted grey:

```167:180:frontend/src/content/own/WorkOrders/Calendar/index.tsx
  const getColor = (priority: Priority) => {
    switch (priority) {
      case 'HIGH':
        return theme.colors.error.main;
      case 'MEDIUM':
        return theme.colors.warning.main;
      case 'LOW':
        return theme.colors.info.main;
      case 'NONE':
        return theme.colors.primary.main;
      default:
        break;
    }
  };
```

  | Priority | Theme color           | Visual meaning |
  |----------|-----------------------|----------------|
  | `HIGH`   | `error.main`          | Red            |
  | `MEDIUM` | `warning.main`        | Amber/orange   |
  | `LOW`    | `info.main`           | Blue           |
  | `NONE`   | `primary.main`        | Brand primary  |
  | `COMPLETE` (any priority) | `alpha.black[30]` | Muted grey |

- `extendedProps.type` carries the original `type` string (`"WORK_ORDER"` or
  `"PREVENTIVE_MAINTENANCE"`) so the click handler can dispatch to the right
  detail view.

#### Toolbar navigation (prev / today / next / change view)

The toolbar in `Actions.tsx` is rendered above the calendar and controls
FullCalendar imperatively via `calendarRef.current.getApi()`:

```199:255:frontend/src/content/own/WorkOrders/Calendar/index.tsx
  const handleDateToday = (): void => {
    const calItem = calendarRef.current;

    if (calItem) {
      const calApi = calItem.getApi();

      calApi.today();
      setDate(calApi.getDate());
    }
  };
  useEffect(() => {
    const calItem = calendarRef.current;
    const newView = calItem.getApi().view;
    if (
      previousView &&
      previousView !== view &&
      viewsOrder.findIndex((v) => v === previousView) <
        viewsOrder.findIndex((v) => v === view)
    ) {
      return;
    }
    const start = newView.activeStart;
    const end = newView.activeEnd;
    dispatch(getWorkOrderEvents(start, end));
  }, [date, view]);
  const changeView = (changedView: View): void => {
    const calItem = calendarRef.current;

    if (calItem) {
      const calApi = calItem.getApi();

      calApi.changeView(changedView);
      setView(changedView);
    }
  };

  const handleDatePrev = (): void => {
    const calItem = calendarRef.current;

    if (calItem) {
      const calApi = calItem.getApi();

      calApi.prev();
      setDate(calApi.getDate());
    }
  };

  const handleDateNext = (): void => {
    const calItem = calendarRef.current;

    if (calItem) {
      const calApi = calItem.getApi();

      calApi.next();
      setDate(calApi.getDate());
    }
  };
```

A subtle but important piece is the events-fetching effect. Every time `date`
or `view` changes:

1. It reads the current view's visible range via `view.activeStart` /
   `view.activeEnd` (FullCalendar always keeps these in sync with what is
   actually rendered, including padding cells in month view).
2. If the user just *zoomed out* (e.g. day → week → month, in
   `viewsOrder = ['dayGridMonth', 'timeGridWeek', 'listWeek', 'timeGridDay']`,
   the index of the previous view is *less than* the new one in the array),
   the effect early-returns to **avoid an extra refetch** because the
   already-loaded wider range is still a superset. Only narrowing or moving
   the date triggers a new request.
3. Otherwise it dispatches `getWorkOrderEvents(start, end)`, which calls the
   backend with that range and refreshes the Redux store.

The `usePrevious` hook from `frontend/src/hooks/usePrevious.ts` is what
enables that previous-vs-current-view comparison.

#### Toolbar UI: `Actions.tsx`

`Actions.tsx` renders three groups of controls:

```72:121:frontend/src/content/own/WorkOrders/Calendar/Actions.tsx
    <Grid
      container
      spacing={3}
      alignItems="center"
      justifyContent="space-between"
    >
      <Grid item>
        <Tooltip arrow placement="top" title={t('previous')}>
          <IconButton color="primary" onClick={onPrevious}>
            <ArrowBackTwoToneIcon />
          </IconButton>
        </Tooltip>
        <Tooltip arrow placement="top" title={t('today')}>
          <IconButton color="primary" sx={{ mx: 1 }} onClick={onToday}>
            <TodayTwoToneIcon />
          </IconButton>
        </Tooltip>
        <Tooltip arrow placement="top" title={t('next')}>
          <IconButton color="primary" onClick={onNext}>
            <ArrowForwardTwoToneIcon />
          </IconButton>
        </Tooltip>
      </Grid>
      <Grid item sx={{ display: { xs: 'none', sm: 'inline-block' } }}>
        <Typography variant="h3" color="text.primary">
          {format(date, 'MMMM yyyy', { locale: dateLocale })}
        </Typography>
      </Grid>
      <Grid item sx={{ display: { xs: 'none', sm: 'inline-block' } }}>
        {viewOptions.map((viewOption) => {
          const Icon = viewOption.icon;
          return (
            <Tooltip
              key={viewOption.value}
              arrow
              placement="top"
              title={t(viewOption.label)}
            >
              <IconButton
                color={viewOption.value === view ? 'primary' : 'secondary'}
                onClick={() => changeView(viewOption.value)}
              >
                <Icon />
              </IconButton>
            </Tooltip>
          );
        })}
      </Grid>
    </Grid>
```

- Left: previous / today / next icon buttons. Tooltips are translated via
  `useTranslation()`'s `t('previous' | 'today' | 'next')`.
- Center: the main label, e.g. `April 2026`. The label is produced by
  `date-fns`'s `format(date, 'MMMM yyyy', { locale: dateLocale })`. The locale
  comes from the `useDateLocale()` hook, which lazy-loads the date-fns locale
  matching the active i18next language. On `xs` breakpoints the label is
  hidden.
- Right: four view-switch icon buttons. Each one is described by an entry in
  `viewOptions`:

```38:59:frontend/src/content/own/WorkOrders/Calendar/Actions.tsx
const viewOptions: ViewOption[] = [
  {
    label: 'month',
    value: 'dayGridMonth',
    icon: CalendarViewMonthTwoToneIcon
  },
  {
    label: 'week',
    value: 'timeGridWeek',
    icon: ViewWeekTwoToneIcon
  },
  {
    label: 'day',
    value: 'timeGridDay',
    icon: ViewDayTwoToneIcon
  },
  {
    label: 'agenda',
    value: 'listWeek',
    icon: ViewAgendaTwoToneIcon
  }
];
```

  The current view's button is colored `primary`, the others `secondary`.

#### `<FullCalendar>` configuration

```280:309:frontend/src/content/own/WorkOrders/Calendar/index.tsx
        <FullCalendar
          allDayMaintainDuration
          initialDate={date}
          initialView={view}
          locale={calendarLocale}
          droppable
          eventDisplay="block"
          eventClick={(arg) =>
            handleOpenDetails(
              Number(arg.event.id),
              arg.event.extendedProps.type
            )
          }
          dateClick={(event) => handleAddWorkOrder(event.date)}
          dayMaxEventRows={4}
          events={calendar.events.map((eventPayload) =>
            getEventFromWO(eventPayload)
          )}
          headerToolbar={false}
          height={660}
          ref={calendarRef}
          rerenderDelay={10}
          weekends
          plugins={[
            dayGridPlugin,
            timeGridPlugin,
            interactionPlugin,
            listPlugin
          ]}
        />
```

Key options:

- `headerToolbar={false}` — the built-in FullCalendar header is disabled
  because the project renders its own MUI-based toolbar (`Actions`).
- `height={660}` — fixed pixel height. The wrapping `Card` body therefore has
  a known size for the spinner overlay.
- `weekends` — Saturday and Sunday columns are visible.
- `eventDisplay="block"` — events are rendered as solid colored blocks rather
  than dot indicators, even in month view.
- `dayMaxEventRows={4}` — at most four events are listed per cell in month
  view; additional ones collapse into a "+N more" link.
- `eventClick` → `handleOpenDetails(Number(event.id), event.extendedProps.type)`.
  This is the bridge to the work order drawer or PM page.
- `dateClick` → `handleAddWorkOrder(event.date)`. Clicking an empty cell opens
  the create-work-order modal pre-filled with that date.
- `locale={calendarLocale}` — the FullCalendar locale bundle (different from
  the date-fns locale used by `Actions`).
- `droppable` and `allDayMaintainDuration` are present but no drop handler is
  wired in this component, so they currently have no functional effect.
- `rerenderDelay={10}` — small debounce to avoid rerender thrash when many
  events change at once.

The whole calendar is wrapped in a `FullCalendarWrapper` styled `Box` which
overrides FullCalendar's default theme to match the MUI theme — it handles
borders, hover highlights, the "today" cell color, list-view headings, and
hides the FullCalendar v5 license footer message:

```40:123:frontend/src/content/own/WorkOrders/Calendar/index.tsx
const FullCalendarWrapper = styled(Box)(
  ({ theme }) => `
    padding: ${theme.spacing(3)};
    position: relative;
   
    & .fc-license-message {
      display: none;
    }
    .fc {
      .fc-daygrid-day,.fc-timegrid-slot{
        cursor: pointer;
      }
      .fc-col-header-cell {
        padding: ${theme.spacing(1)};
        background: ${theme.colors.alpha.black[5]};
      }

      .fc-scrollgrid {
        border: 2px solid ${theme.colors.alpha.black[10]};
        ...
      }
      ...
      td.fc-daygrid-day.fc-day-today {
        background-color: ${theme.colors.primary.lighter};
      }
      ...
    }
`
);
```

#### Loading spinner

The component overlays a centered `CircularProgress` while events are being
fetched:

```274:279:frontend/src/content/own/WorkOrders/Calendar/index.tsx
      <FullCalendarWrapper>
        {loadingGet && (
          <Stack position="absolute" top={'45%'} left={'45%'} zIndex={10}>
            <CircularProgress size={64} />
          </Stack>
        )}
```

`loadingGet` is the same flag toggled by `getWorkOrderEvents` and other
`workOrders` thunks.

### Redux: where events live

#### State shape

```24:48:frontend/src/slices/workOrder.ts
interface WorkOrderState {
  workOrders: Page<WorkOrder>;
  workOrdersMini: Page<WorkOrderBaseMiniDTO>;
  workOrdersByLocation: { [key: number]: WorkOrder[] };
  workOrdersByPart: { [key: number]: WorkOrder[] };
  singleWorkOrder: WorkOrder;
  urgentCount: number;
  loadingGet: boolean;
  calendar: {
    events: CalendarEvent<WorkOrder | PreventiveMaintenance>[];
  };
}

const initialState: WorkOrderState = {
  workOrders: getInitialPage<WorkOrder>(),
  workOrdersByLocation: {},
  workOrdersByPart: {},
  singleWorkOrder: null,
  urgentCount: 0,
  loadingGet: false,
  workOrdersMini: getInitialPage<WorkOrderBaseMiniDTO>(),
  calendar: {
    events: []
  }
};
```

The `CalendarEvent` shape is defined in the same file:

```18:22:frontend/src/slices/workOrder.ts
export interface CalendarEvent<T extends WorkOrderBase> {
  type: string;
  date: Date;
  event: T;
}
```

#### Thunk: `getWorkOrderEvents`

```341:357:frontend/src/slices/workOrder.ts
export const getWorkOrderEvents =
  (start: Date, end: Date): AppThunk =>
  async (dispatch) => {
    dispatch(slice.actions.setLoadingGet({ loading: true }));
    const response = await api.post<
      CalendarEvent<WorkOrder | PreventiveMaintenance>[]
    >(`${basePath}/events`, {
      start,
      end
    });
    dispatch(
      slice.actions.getEvents({
        events: response
      })
    );
    dispatch(slice.actions.setLoadingGet({ loading: false }));
  };
```

The thunk sends `{ start, end }` to `POST /work-orders/events` and stores the
result in `state.workOrders.calendar.events` via the `getEvents` reducer:

```173:181:frontend/src/slices/workOrder.ts
    getEvents(
      state: WorkOrderState,
      action: PayloadAction<{
        events: CalendarEvent<WorkOrder | PreventiveMaintenance>[];
      }>
    ) {
      const { events } = action.payload;
      state.calendar.events = events;
    },
```

### A note on `slices/calendar.ts` and `EventDrawer.tsx`

The repository also contains a generic *meetings* calendar slice in
`frontend/src/slices/calendar.ts`. It defines its own `Event` model (in
`frontend/src/models/calendar.ts`) and CRUD thunks against
`/api/calendar/meetings*`:

```95:155:frontend/src/slices/calendar.ts
export const getEvents =
  (): AppThunk =>
  async (dispatch): Promise<void> => {
    const response = await axios.get<{ events: Event[] }>(
      '/api/calendar/meetings'
    );
    dispatch(slice.actions.getEvents(response.data));
  };

export const createEvent = ...
export const selectEvent = ...
export const updateEvent = ...
export const deleteEvent = ...
export const selectRange = ...
```

`EventDrawer.tsx` and `PageHeader.tsx` are wired to that slice and show an
"Add meeting" / "Create new calendar event" form. The Work Orders calendar in
`Calendar/index.tsx` only imports `selectEvent` from this slice and never
opens the drawer. In other words:

- `slices/calendar.ts`, `EventDrawer.tsx` and `PageHeader.tsx` are **legacy**
  scaffolding originally intended for a generic meetings calendar.
- The active production calendar UI is the one driven by
  `slices/workOrder.ts` → `getWorkOrderEvents` → `state.workOrders.calendar`.

When reading the code, treat `EventDrawer.tsx` / `PageHeader.tsx` as inert
unless the Work Orders calendar is later extended to dispatch
`openDrawerPanel` / `selectEvent` against the meetings slice.

## Localization: how `April`, `May`, … are produced

There are **two independent locale systems** in play and you need both to get
fully localized output. They are wired up through `frontend/src/i18n/i18n.ts`.

### 1. `i18next` (UI strings)

The toolbar tooltips (`previous`, `today`, `next`, `month`, `week`, `day`,
`agenda`), tab labels (`list_view`, `calendar_view`, `column_view`) and the
"Coming Soon" / "Events" / "Add meeting" copy all come from i18next bundles:

```26:42:frontend/src/i18n/i18n.ts
const translationLoaders: Record<string, () => Promise<{ default: object }>> = {
  de: () => import('./translations/de'),
  en: () => import('./translations/en'),
  es: () => import('./translations/es'),
  fr: () => import('./translations/fr'),
  pl: () => import('./translations/pl'),
  tr: () => import('./translations/tr'),
  pt_br: () => import('./translations/pt_BR'),
  ar: () => import('./translations/ar'),
  it: () => import('./translations/it'),
  sv: () => import('./translations/sv'),
  ru: () => import('./translations/ru'),
  hu: () => import('./translations/hu'),
  nl: () => import('./translations/nl'),
  zh_cn: () => import('./translations/zh_cn'),
  ba: () => import('./translations/ba')
};
```

`i18next-browser-languagedetector` picks the current language from the
`?lang=` query, then `localStorage`, then the navigator. The detector also
normalizes `pt`/`pt-br`/`pt_br` to `pt_br` and `zh`/`zh-cn`/`zh_cn` to
`zh_cn`:

```86:115:frontend/src/i18n/i18n.ts
i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: {},
    partialBundledLanguages: true,
    supportedLngs: Object.keys(translationLoaders),
    keySeparator: false,
    fallbackLng: 'en',
    react: {
      useSuspense: true
    },
    interpolation: {
      escapeValue: false
    },
    detection: {
      order: ['querystring', 'localStorage', 'navigator'],
      lookupQuerystring: 'lang',
      lookupLocalStorage: 'lang',
      caches: ['localStorage'],
      convertDetectedLanguage: (lng) => {
        const lower = lng.toLowerCase();
        if (lower === 'pt' || lower === 'pt-br' || lower === 'pt_br')
          return 'pt_br';
        if (lower === 'zh' || lower === 'zh-cn' || lower === 'zh_cn')
          return 'zh_cn';
        return lower.split('-')[0].split('_')[0];
      }
    }
  });
```

### 2. `date-fns` (the `April 2026` toolbar label)

The toolbar's month/year text is produced by `date-fns`:

```98:99:frontend/src/content/own/WorkOrders/Calendar/Actions.tsx
        <Typography variant="h3" color="text.primary">
          {format(date, 'MMMM yyyy', { locale: dateLocale })}
        </Typography>
```

`dateLocale` comes from `useDateLocale()`, which looks up the date-fns
locale that matches the current i18next language and falls back to
`enUS`:

```1:19:frontend/src/hooks/useDateLocale.tsx
import { useEffect, useState } from 'react';
import { Locale as DateLocale } from 'date-fns';
import { enUS } from 'date-fns/locale';
import { getDateLocale } from '../i18n/i18n';
import { useTranslation } from 'react-i18next';

const useDateLocale = (): DateLocale => {
  const { i18n } = useTranslation();
  const [dateLocale, setDateLocale] = useState<DateLocale>(enUS);

  useEffect(() => {
    getDateLocale(i18n.language).then(setDateLocale);
  }, [i18n.language]);

  return dateLocale;
};

export default useDateLocale;
```

The mapping table is in `i18n.ts`:

```45:61:frontend/src/i18n/i18n.ts
const dateLocaleLoaders: Record<string, () => Promise<DateLocale>> = {
  en: () => import('date-fns/locale').then((m) => m.enUS),
  fr: () => import('date-fns/locale').then((m) => m.fr),
  es: () => import('date-fns/locale').then((m) => m.es),
  de: () => import('date-fns/locale').then((m) => m.de),
  tr: () => import('date-fns/locale').then((m) => m.tr),
  pt_br: () => import('date-fns/locale').then((m) => m.ptBR),
  pl: () => import('date-fns/locale').then((m) => m.pl),
  ar: () => import('date-fns/locale').then((m) => m.ar),
  it: () => import('date-fns/locale').then((m) => m.it),
  sv: () => import('date-fns/locale').then((m) => m.sv),
  ru: () => import('date-fns/locale').then((m) => m.ru),
  hu: () => import('date-fns/locale').then((m) => m.hu),
  nl: () => import('date-fns/locale').then((m) => m.nl),
  zh_cn: () => import('date-fns/locale').then((m) => m.zhCN),
  ba: () => import('date-fns/locale').then((m) => m.bs)
};
```

So when the active language is, for example, `fr`, the toolbar header reads
`avril 2026` instead of `April 2026`. All of these locales are Gregorian —
`date-fns@2.28.0` does not provide a Persian / Jalali locale natively.

### 3. FullCalendar locale bundles (column headers, time formats, list view)

FullCalendar manages day-of-week labels, time formats and list-view headings
using its own locale bundles, loaded asynchronously:

```63:83:frontend/src/i18n/i18n.ts
const calendarLocaleLoaders: Record<string, () => Promise<LocaleSingularArg>> =
  {
    en: () => import('@fullcalendar/core/locales/en-gb').then((m) => m.default),
    fr: () => import('@fullcalendar/core/locales/fr').then((m) => m.default),
    es: () => import('@fullcalendar/core/locales/es').then((m) => m.default),
    de: () => import('@fullcalendar/core/locales/de').then((m) => m.default),
    tr: () => import('@fullcalendar/core/locales/tr').then((m) => m.default),
    pt_br: () =>
      import('@fullcalendar/core/locales/pt-br').then((m) => m.default),
    pl: () => import('@fullcalendar/core/locales/pl').then((m) => m.default),
    ar: () => import('@fullcalendar/core/locales/ar').then((m) => m.default),
    it: () => import('@fullcalendar/core/locales/it').then((m) => m.default),
    sv: () => import('@fullcalendar/core/locales/sv').then((m) => m.default),
    ru: () => import('@fullcalendar/core/locales/ru').then((m) => m.default),
    hu: () => import('@fullcalendar/core/locales/hu').then((m) => m.default),
    nl: () => import('@fullcalendar/core/locales/nl').then((m) => m.default),
    zh_cn: () =>
      import('@fullcalendar/core/locales/zh-cn').then((m) => m.default),
    ba: () => import('@fullcalendar/core/locales/bs').then((m) => m.default)
  };
```

The component side:

```154:158:frontend/src/content/own/WorkOrders/Calendar/index.tsx
  const [calendarLocale, setCalendarLocale] = useState<LocaleSingularArg>(enGb);

  useEffect(() => {
    getCalendarLocale(i18n.language).then(setCalendarLocale);
  }, [i18n.language]);
```

`getCalendarLocale` falls back to `en` (which loads `en-gb`) if no mapping
exists:

```160:165:frontend/src/i18n/i18n.ts
export const getCalendarLocale = async (
  lang: string
): Promise<LocaleSingularArg> => {
  const loader = calendarLocaleLoaders[lang] ?? calendarLocaleLoaders['en'];
  return loader();
};
```

### Putting the three together

For the user-visible string `April 2026` in the toolbar:

1. The user's language is detected by `i18next-browser-languagedetector`
   (`querystring → localStorage → navigator`, normalized).
2. `useDateLocale()` resolves that language to a `date-fns` `Locale`.
3. `format(date, 'MMMM yyyy', { locale })` produces the localized string,
   e.g. `April 2026` (English) or `avril 2026` (French).

For the day-of-week column headers and the `Mon`, `Tue`, … labels rendered
*inside* the FullCalendar grid:

1. The active language is read from `i18n.language`.
2. `getCalendarLocale(lang)` lazy-imports the matching FullCalendar bundle.
3. `<FullCalendar locale={calendarLocale} ... />` reapplies it on language
   change.

For tooltips and view labels (`previous`, `today`, `month`, `agenda`, …)
the standard `useTranslation()` / `t(key)` flow drives the strings.

### Right-to-left

There is no calendar-specific RTL toggle. RTL behavior is configured globally
elsewhere in the app (Arabic — `ar`) by setting the document direction via
`i18n.dir()`. FullCalendar inherits that through its own locale bundle (e.g.
the `ar` locale ships with `direction: 'rtl'`) and the surrounding MUI theme.

## Backend in depth

### Endpoint

```119:139:api/src/main/java/com/grash/controller/WorkOrderController.java
    @PostMapping("/events")
    @PreAuthorize("hasRole('ROLE_CLIENT')")
    public Collection<CalendarEvent<WorkOrderBaseMiniDTO>> getEvents(@Parameter(description = "Date range for " +
            "calendar events") @Valid @RequestBody DateRange
                                                                             dateRange, HttpServletRequest req) {
        User user = userService.whoami(req);
        if (user.getRole().getViewPermissions().contains(PermissionEntity.WORK_ORDERS)) {
            List<CalendarEvent<WorkOrderBaseMiniDTO>> result = new ArrayList<>();
            result.addAll(preventiveMaintenanceService.getEvents(dateRange.getEnd(), user.getCompany().getId()).stream()
                    .filter(calendarEvent -> calendarEvent.getDate().after(new Date()))
                    .filter(calendarEvent -> canViewWorkOrderBase(user, calendarEvent.getEvent()))
                    .map(calendarEvent -> new CalendarEvent<>(calendarEvent.getType(),
                            preventiveMaintenanceMapper.toBaseMiniDto(calendarEvent.getEvent()),
                            calendarEvent.getDate()))
                    .collect(Collectors.toList()));
            result.addAll(workOrderService.findByDueDateBetweenAndCompany(dateRange.getStart(), dateRange.getEnd(),
                    user.getCompany().getId()).stream().filter(workOrder -> canViewWorkOrderBase(user, workOrder)).map(workOrderMapper::toBaseMiniDto).map(workOrderMiniDTO -> new CalendarEvent<>("WORK_ORDER",
                    workOrderMiniDTO, workOrderMiniDTO.getDueDate())).collect(Collectors.toList()));
            return result;
        } else throw new CustomException("Access Denied", HttpStatus.FORBIDDEN);
    }
```

What it does:

- Authenticated via Spring Security (`@PreAuthorize("hasRole('ROLE_CLIENT')")`)
  using the JWT-resolved user.
- The user must hold the `WORK_ORDERS` view permission, otherwise the
  endpoint returns `403 Access Denied`.
- The response is the union of two lists:
  - **Preventive maintenance occurrences** — computed via
    `preventiveMaintenanceService.getEvents(end, companyId)`, then filtered to
    only future occurrences (`getDate().after(new Date())`) and to those the
    caller can view. Each one is mapped to `CalendarEvent<WorkOrderBaseMiniDTO>`
    using `PreventiveMaintenanceMapper#toBaseMiniDto`.
  - **Work orders** with `dueDate` between `start` and `end` for the caller's
    company, filtered by `canViewWorkOrderBase`, mapped to
    `WorkOrderBaseMiniDTO` and tagged with `type = "WORK_ORDER"`.
- Per-row visibility is enforced by `canViewWorkOrderBase`:

```141:147:api/src/main/java/com/grash/controller/WorkOrderController.java
    private boolean canViewWorkOrderBase(User user, WorkOrderBase workOrderBase) {
        boolean canViewOthers =
                user.getRole().getViewOtherPermissions().contains(workOrderBase instanceof PreventiveMaintenance ?
                        PermissionEntity.PREVENTIVE_MAINTENANCES : PermissionEntity.WORK_ORDERS);
        return canViewOthers || (workOrderBase.getCreatedBy() != null && workOrderBase.getCreatedBy().equals(user.getId())) || workOrderBase.isAssignedTo(user);

    }
```

  i.e. the user sees an event when they have the `view-others` permission for
  its category, **or** they created it, **or** they are assigned to it.

### DTOs

`DateRange` is the request body:

```10:19:api/src/main/java/com/grash/dto/DateRange.java
@SuperBuilder
@Data
@NoArgsConstructor
@Schema(description = "Date range for filtering events and scheduled work orders")
public class DateRange {
    @Schema(description = "Start date of the range")
    private Date start;
    @Schema(description = "End date of the range")
    private Date end;
}
```

`CalendarEvent<T>` is the response wrapper:

```14:23:api/src/main/java/com/grash/dto/CalendarEvent.java
public class CalendarEvent<T> {
    @Schema(description = "Event type")
    private String type;
    
    @Schema(description = "Event data")
    private T event;
    
    @Schema(description = "Event date")
    private Date date;
}
```

`WorkOrderBaseMiniDTO` is the slim payload returned for each event:

```12:30:api/src/main/java/com/grash/dto/WorkOrderBaseMiniDTO.java
@Data
@NoArgsConstructor
@Schema(description = "Work order base mini DTO with essential work order details")
public class WorkOrderBaseMiniDTO {
    @Schema(description = "Work order unique identifier")
    private Long id;
    @Schema(description = "Work order title")
    private String title;
    @Schema(description = "Work order due date")
    private Date dueDate;
    @Schema(description = "Work order creation timestamp")
    private Instant createdAt;
    @Schema(description = "Work order priority level")
    private Priority priority;
    @Schema(description = "Work order current status")
    private Status status;
    @Schema(description = "Custom work order identifier")
    private String customId;
}
```

Type hint: even though the frontend types calendar events as
`CalendarEvent<WorkOrder | PreventiveMaintenance>`, the actual JSON payload is
`CalendarEvent<WorkOrderBaseMiniDTO>`. The frontend just needs `id`, `title`,
`priority` and `status`, all of which are present on the mini DTO, so this
works in practice. Anyone touching this code should be aware of the mismatch
between the TypeScript type and the runtime shape.

### Preventive maintenance events: Quartz triggers

`PreventiveMaintenanceService#getEvents(end, companyId)` materializes upcoming
PM occurrences from the Quartz scheduler:

```167:208:api/src/main/java/com/grash/service/PreventiveMaintenanceService.java
    public List<CalendarEvent<PreventiveMaintenance>> getEvents(Date end, Long companyId) {
        if (!licenseService.hasEntitlement(LicenseEntitlement.PM_CALENDAR))
            return Collections.emptyList();
        List<PreventiveMaintenance> preventiveMaintenances =
                preventiveMaintenanceRepository.findByCreatedAtBeforeAndCompany_Id(end, companyId);
        List<CalendarEvent<PreventiveMaintenance>> result = new ArrayList<>();

        for (PreventiveMaintenance preventiveMaintenance : preventiveMaintenances) {
            Schedule schedule = preventiveMaintenance.getSchedule();
            if (schedule == null || schedule.isDisabled()) continue;

            if (schedule.getRecurrenceBasedOn() != RecurrenceBasedOn.SCHEDULED_DATE) continue;

            try {
                TriggerKey triggerKey = new TriggerKey("wo-trigger-" + schedule.getId(), "wo-group");
                Trigger trigger = scheduler.getTrigger(triggerKey);

                if (trigger == null) {
                    log.warn("No trigger found for schedule {}", schedule.getId());
                    continue;
                }

                // Get all fire times up to the end date
                List<Date> fireTimes = new ArrayList<>();

                // Use TriggerUtils to get computed fire times
                if (trigger instanceof OperableTrigger) {
                    OperableTrigger operableTrigger = (OperableTrigger) trigger;
                    Date currentTime = new Date();

                    // Start from now or startsOn, whichever is earlier
                    Date startTime = schedule.getStartsOn().before(currentTime) ?
                            schedule.getStartsOn() : currentTime;

                    // Compute fire times
                    Date fireTime = operableTrigger.getFireTimeAfter(startTime);
                    while (fireTime != null && (fireTime.before(end) || fireTime.equals(end))) {
                        if (shouldFireOnDate(schedule, fireTime)) {
                            fireTimes.add(fireTime);
                        }
                        fireTime = operableTrigger.getFireTimeAfter(fireTime);
```

Important properties of this implementation:

- It is **gated by the `PM_CALENDAR` license entitlement**. Without it, the
  endpoint returns no PM occurrences (work orders still show up).
- It only considers schedules whose `recurrenceBasedOn` is `SCHEDULED_DATE`.
  Schedules driven by completion date are not enumerated on the calendar.
- Each PM corresponds to a Quartz trigger keyed
  `wo-trigger-{scheduleId}` in group `wo-group`. The service loads the
  trigger and walks `getFireTimeAfter(...)` until it crosses the `end`
  boundary, building a virtual list of future occurrences.
- The controller layer additionally filters those occurrences to keep only
  the future ones (`getDate().after(new Date())`).

### End-to-end request

Putting the whole flow together for a user opening the calendar tab:

1. The user clicks the **Calendar** tab on the Work Orders page.
2. `<WorkOrderCalendar>` mounts. Its initial `view` is `'timeGridWeek'` and
   `date` is `new Date()`. The `useEffect` on `[date, view]` runs.
3. The effect reads the rendered range from FullCalendar (`activeStart`,
   `activeEnd`) and dispatches `getWorkOrderEvents(start, end)`.
4. The thunk POSTs `/work-orders/events` with `{ start, end }`,
   authenticated by the user's JWT.
5. The Spring controller checks the `WORK_ORDERS` view permission and
   collects:
   - all work orders for the user's company whose `dueDate` falls in
     `[start, end]`, filtered by `canViewWorkOrderBase`,
   - plus PM occurrences computed by the Quartz trigger walk, future-only
     and filtered by `canViewWorkOrderBase`.
6. Each entry is wrapped in `CalendarEvent<WorkOrderBaseMiniDTO>` with
   `type = "WORK_ORDER"` or `type = "PREVENTIVE_MAINTENANCE"`.
7. The thunk stores the array in `state.workOrders.calendar.events`.
8. `useSelector` re-renders `<ApplicationsCalendar>`, which maps each event
   through `getEventFromWO` into FullCalendar events with priority-based
   colors and `extendedProps.type`.
9. FullCalendar paints the events onto whichever view is active. The
   user's clicks dispatch back to the page-level handlers:
   - clicking an event → `handleOpenDetails(id, type)` → opens the WO drawer
     or navigates to the PM page;
   - clicking an empty cell → `handleAddWorkOrder(date)` → opens the
     create-WO modal pre-filled with that due date.

## Customization points

If you want to extend or change calendar behavior, these are the natural
hooks:

- **Default view and views list**
  - Change the `useState<View>('timeGridWeek')` initial value or the
    `viewsOrder` array to alter the default view, the toolbar order or
    add/remove views.
  - Each new view requires its FullCalendar plugin to be registered in the
    `plugins` array.
- **Event styling**
  - `getColor(priority)` controls priority colors.
  - `getEventFromWO` returns the `Event` object, including the `color`
    override for completed work orders. Add fields to `extendedProps` here if
    you need them in `eventClick` or `eventContent` rendering.
- **What gets fetched**
  - The fetch range is whatever FullCalendar exposes via `view.activeStart` /
    `view.activeEnd`. To change the prefetch window, prefetch around the
    range or coalesce calls, modify the `useEffect` on `[date, view]`.
  - The previous-vs-current view comparison guards against extra fetches when
    zooming out across `viewsOrder`. Reorder that array carefully — the order
    is "wider → narrower".
- **Server-side data**
  - Add new event sources by extending `WorkOrderController#getEvents` with
    additional `result.addAll(...)` calls and a new `type` string.
  - Tighten visibility by changing `canViewWorkOrderBase`.
  - Adjust PM occurrence computation in
    `PreventiveMaintenanceService#getEvents`. The current implementation
    requires a license entitlement; remove or change that gate as needed.
- **Localization**
  - To add a new language to the calendar, add an entry to
    `translationLoaders`, `dateLocaleLoaders` and `calendarLocaleLoaders` in
    `frontend/src/i18n/i18n.ts`, and add the language to `supportedLanguages`.
  - Replacing the current Gregorian month label `format(date, 'MMMM yyyy', …)`
    with a different calendar system (e.g. Persian / Jalali) requires
    swapping `date-fns` for a calendar-aware library at that call site, plus
    similar handling for the FullCalendar grid (FullCalendar v5 has no
    built-in Jalali support — see
    [`cursor-docs/persian-language-calendar-support`](./persian-language-calendar-support/README.md)
    for the in-flight assessment of that work).

## Common pitfalls

- **Two locale systems**. Changing only `i18next` does not change the
  in-grid day headers; you must add a corresponding entry to
  `calendarLocaleLoaders` in `i18n.ts`. Likewise, the toolbar `MMMM yyyy`
  label is driven by `date-fns`, not by FullCalendar.
- **Type vs. runtime mismatch**. The frontend types
  `state.workOrders.calendar.events` as
  `CalendarEvent<WorkOrder | PreventiveMaintenance>[]`, but the wire format
  is actually `CalendarEvent<WorkOrderBaseMiniDTO>[]`. Code that reaches into
  the `event` payload should only rely on the mini DTO fields (`id`,
  `title`, `priority`, `status`, `dueDate`).
- **Permission gates**. The calendar tab is hidden, the events endpoint
  returns 403, and PM occurrences are silently empty if the company is
  missing the `PM_CALENDAR` license entitlement. If the calendar appears
  empty for users that "should" see events, check the permissions and the
  entitlement before debugging the UI.
- **Legacy `EventDrawer`/`PageHeader`/`slices/calendar.ts`**. They look like
  they belong to the Work Orders calendar but actually target a separate,
  unused `/api/calendar/meetings` API. Don't accidentally wire them into the
  Work Orders calendar without an explicit product decision.
