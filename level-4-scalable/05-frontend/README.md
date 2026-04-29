`Level 4` **Step 5 of 9** — Frontend

# 05 — Frontend: Dashboard Components with Recharts

> **What you already know carries over.** JSX, hooks (`useState`, `useEffect`), controlled inputs, the `.map` pattern for lists — all from Level 1 Step 4 and Level 2 Step 5. Redux Toolkit's `useAppSelector`/`useAppDispatch` and slice patterns — from Step 4 of this level. If anything feels unfamiliar, those walkthroughs cover it.
>
> **What's new in Level 4 on the frontend:**
> - **Recharts components** — `<LineChart>`, `<BarChart>`, `<PieChart>` and their building blocks (`<Line>`, `<Bar>`, `<Pie>`, `<Cell>`, `<XAxis>`, `<YAxis>`, `<CartesianGrid>`, `<Tooltip>`, `<Legend>`, `<ResponsiveContainer>`). Render-as-JSX charts.
> - **Skeleton loading states** — render placeholder cards while data is loading, instead of nothing or a single spinner.
> - **`useEffect` with multi-value dependencies** — refetching when several Redux selectors change.
> - **The `children: ReactNode` prop pattern** — components that accept arbitrary nested JSX.
> - **Controlled `<select>` dropdowns** that dispatch Redux actions instead of `setState`.
>
> Every code block below has a "Reading This File Line-by-Line" walkthrough.

## Spatial Orientation

The frontend is a single-page dashboard with a sidebar for navigation and a main content area. The sidebar switches between the **Overview** view (cards + charts) and the **Events** view (filterable table). All data comes from the Redux store.

```
┌──────────────────────────────────────────────────────────────────┐
│  App                                                              │
│  ├── Provider (Redux store)                                      │
│  │   └── Layout                                                  │
│  │       ├── Sidebar                                             │
│  │       │   ├── "Overview" link                                 │
│  │       │   └── "Events" link                                   │
│  │       └── Main Content                                        │
│  │           ├── FilterBar (always visible)                      │
│  │           │                                                    │
│  │           ├── [If view = "overview"]                           │
│  │           │   ├── OverviewCards                                │
│  │           │   ├── TimeSeriesChart                              │
│  │           │   ├── CategoryChart                                │
│  │           │   └── DeviceChart                                  │
│  │           │                                                    │
│  │           └── [If view = "events"]                             │
│  │               └── EventsTable                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1: UI Slice (View Switching)

The sidebar needs to track which view is active. This is UI state — Redux manages it.

> [!IMPORTANT]
> **You should be in:** `data-dash/client/`

Create `src/features/ui/uiSlice.ts`:

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';

type View = 'overview' | 'events';

interface UIState {
  activeView: View;
  sidebarCollapsed: boolean;
}

const initialState: UIState = {
  activeView: 'overview',
  sidebarCollapsed: false,
};

const uiSlice = createSlice({
  name: 'ui',
  initialState,
  reducers: {
    setActiveView(state, action: PayloadAction<View>) {
      state.activeView = action.payload;
    },
    toggleSidebar(state) {
      state.sidebarCollapsed = !state.sidebarCollapsed;
    },
  },
});

export const { setActiveView, toggleSidebar } = uiSlice.actions;

export const selectActiveView = (state: RootState) => state.ui.activeView;
export const selectSidebarCollapsed = (state: RootState) => state.ui.sidebarCollapsed;

export default uiSlice.reducer;
```

Now add the UI reducer to the store. Update `src/app/store.ts`:

```typescript
import { configureStore } from '@reduxjs/toolkit';
import analyticsReducer from '../features/analytics/analyticsSlice';
import eventsReducer from '../features/events/eventsSlice';
import filtersReducer from '../features/filters/filtersSlice';
import uiReducer from '../features/ui/uiSlice';

export const store = configureStore({
  reducer: {
    analytics: analyticsReducer,
    events: eventsReducer,
    filters: filtersReducer,
    ui: uiReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

## Step 2: Layout and Sidebar

Create `src/features/ui/Layout.tsx`:

```typescript
import { ReactNode } from 'react';
import Sidebar from './Sidebar';
import { useAppSelector } from '../../app/hooks';
import { selectSidebarCollapsed } from './uiSlice';

interface LayoutProps {
  children: ReactNode;
}

export default function Layout({ children }: LayoutProps) {
  const collapsed = useAppSelector(selectSidebarCollapsed);

  return (
    <div className={`layout ${collapsed ? 'layout--collapsed' : ''}`}>
      <Sidebar />
      <main className="main-content">
        {children}
      </main>
    </div>
  );
}
```

Create `src/features/ui/Sidebar.tsx`:

```typescript
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import { setActiveView, toggleSidebar, selectActiveView } from './uiSlice';

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const activeView = useAppSelector(selectActiveView);

  return (
    <aside className="sidebar">
      <div className="sidebar-header">
        <h1 className="sidebar-title">DataDash</h1>
        <button
          className="sidebar-toggle"
          onClick={() => dispatch(toggleSidebar())}
          aria-label="Toggle sidebar"
        >
          ☰
        </button>
      </div>

      <nav className="sidebar-nav">
        <button
          className={`nav-item ${activeView === 'overview' ? 'nav-item--active' : ''}`}
          onClick={() => dispatch(setActiveView('overview'))}
        >
          <span className="nav-icon">📊</span>
          <span className="nav-label">Overview</span>
        </button>

        <button
          className={`nav-item ${activeView === 'events' ? 'nav-item--active' : ''}`}
          onClick={() => dispatch(setActiveView('events'))}
        >
          <span className="nav-icon">📋</span>
          <span className="nav-label">Events</span>
        </button>
      </nav>
    </aside>
  );
}
```

### Reading Layout.tsx Line-by-Line

```tsx
interface LayoutProps {
  children: ReactNode;
}

export default function Layout({ children }: LayoutProps) {
  const collapsed = useAppSelector(selectSidebarCollapsed);

  return (
    <div className={`layout ${collapsed ? 'layout--collapsed' : ''}`}>
      <Sidebar />
      <main className="main-content">
        {children}
      </main>
    </div>
  );
}
```

- `children: ReactNode` — the **children prop pattern**. Any component using `<Layout>...stuff...</Layout>` automatically passes the nested JSX as the `children` prop. `ReactNode` is a TypeScript type meaning "anything React can render" — JSX elements, strings, numbers, fragments, arrays, even `null`.
- We pull `children` out of props by destructuring.
- `useAppSelector(selectSidebarCollapsed)` reads one boolean from Redux. The component re-renders only when this specific value changes (Redux's selective subscription).
- `\`layout ${collapsed ? 'layout--collapsed' : ''}\`` — template literal building a dynamic class string. When collapsed, an extra modifier class is added. CSS keys off it.
- `{children}` inside `<main>` — render whatever was passed in. The Layout doesn't know or care what the children are; it just provides a frame around them.

### Reading Sidebar.tsx Line-by-Line

```tsx
const dispatch = useAppDispatch();
const activeView = useAppSelector(selectActiveView);
```

Two hooks — the standard Redux duo. `useAppDispatch` to send actions; `useAppSelector` to read state.

```tsx
<button
  className={`nav-item ${activeView === 'overview' ? 'nav-item--active' : ''}`}
  onClick={() => dispatch(setActiveView('overview'))}
>
```

- The button's `className` adds `'nav-item--active'` when this view is the active one. Triple-equality compares strings; it's `true` when `activeView` matches.
- `onClick={() => dispatch(setActiveView('overview'))}` — arrow function that dispatches an action. This is the heart of "Redux instead of setState": instead of a local `setActiveView` from `useState`, we call the dispatch function with an action creator's result.
- `aria-label="Toggle sidebar"` (on the toggle button above) is an accessibility attribute — screen readers announce it. Always add `aria-label` to icon-only buttons.

Notice: the sidebar dispatches Redux actions instead of managing local state. This means any component in the app can read `selectActiveView` to know which view is active.

---

## Step 3: Filter Bar

Create `src/features/filters/FilterBar.tsx`:

```typescript
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  setDateRange,
  setCategory,
  setDevice,
  clearFilters,
  selectFilters,
  selectHasActiveFilters,
} from './filtersSlice';

// Pre-defined date ranges
function getDateRange(days: number): { startDate: string; endDate: string } {
  const end = new Date();
  const start = new Date();
  start.setDate(start.getDate() - days);
  return {
    startDate: start.toISOString().split('T')[0],
    endDate: end.toISOString().split('T')[0],
  };
}

export default function FilterBar() {
  const dispatch = useAppDispatch();
  const filters = useAppSelector(selectFilters);
  const hasFilters = useAppSelector(selectHasActiveFilters);

  return (
    <div className="filter-bar">
      <div className="filter-group">
        <label className="filter-label">Period</label>
        <div className="filter-buttons">
          <button
            className="filter-btn"
            onClick={() => dispatch(setDateRange(getDateRange(30)))}
          >
            30d
          </button>
          <button
            className="filter-btn"
            onClick={() => dispatch(setDateRange(getDateRange(60)))}
          >
            60d
          </button>
          <button
            className="filter-btn"
            onClick={() => dispatch(setDateRange(getDateRange(90)))}
          >
            90d
          </button>
        </div>
      </div>

      <div className="filter-group">
        <label className="filter-label">Category</label>
        <select
          className="filter-select"
          value={filters.category || ''}
          onChange={(e) =>
            dispatch(setCategory(e.target.value || null))
          }
        >
          <option value="">All Categories</option>
          <option value="marketing">Marketing</option>
          <option value="product">Product</option>
          <option value="support">Support</option>
          <option value="engineering">Engineering</option>
        </select>
      </div>

      <div className="filter-group">
        <label className="filter-label">Device</label>
        <select
          className="filter-select"
          value={filters.device || ''}
          onChange={(e) =>
            dispatch(setDevice(e.target.value || null))
          }
        >
          <option value="">All Devices</option>
          <option value="desktop">Desktop</option>
          <option value="mobile">Mobile</option>
          <option value="tablet">Tablet</option>
        </select>
      </div>

      {hasFilters && (
        <button
          className="filter-clear"
          onClick={() => dispatch(clearFilters())}
        >
          Clear Filters
        </button>
      )}
    </div>
  );
}
```

### Reading This File Line-by-Line

```tsx
function getDateRange(days: number): { startDate: string; endDate: string } {
  const end = new Date();
  const start = new Date();
  start.setDate(start.getDate() - days);
  return {
    startDate: start.toISOString().split('T')[0],
    endDate: end.toISOString().split('T')[0],
  };
}
```

A pure helper outside the component (no React state or hooks needed).

- `new Date()` — JavaScript's built-in date object, defaulting to "right now."
- `start.setDate(start.getDate() - days)` — `getDate()` returns the day-of-month (1–31); subtracting `days` and passing back to `setDate()` shifts the date that many days into the past. JavaScript automatically handles month/year rollover (Jan 5 minus 10 days = Dec 26).
- `start.toISOString()` returns a string like `"2025-01-15T14:32:00.000Z"`. We split on `'T'` and take `[0]` to get just `"2025-01-15"` — the format our backend expects.
- Returns a plain object with both dates. Action creators like `setDateRange` accept this shape directly.

```tsx
export default function FilterBar() {
  const dispatch = useAppDispatch();
  const filters = useAppSelector(selectFilters);
  const hasFilters = useAppSelector(selectHasActiveFilters);
```

Standard Redux hooks — read filters and a derived boolean.

```tsx
<button
  className="filter-btn"
  onClick={() => dispatch(setDateRange(getDateRange(30)))}
>
  30d
</button>
```

Click → call helper → wrap in action creator → dispatch. Nothing local; the change goes through Redux and any component reading `selectFilters` updates automatically.

```tsx
<select
  className="filter-select"
  value={filters.category || ''}
  onChange={(e) =>
    dispatch(setCategory(e.target.value || null))
  }
>
  <option value="">All Categories</option>
  <option value="marketing">Marketing</option>
  ...
</select>
```

A **controlled `<select>` dropdown** that dispatches Redux instead of using local state.

- `value={filters.category || ''}` — fall back to empty string when category is null. HTML `<select>` requires a string value, not null.
- `onChange={(e) => dispatch(setCategory(e.target.value || null))}` — when the user picks an option, take the new value. If it's an empty string ("All Categories"), convert back to `null` (our Redux convention for "no filter"). If it's a real category name, pass it through.
- Each `<option>` has `value={...}` (sent to onChange when selected) and inner text (shown to user). The empty-string option is the "no filter" choice.

```tsx
{hasFilters && (
  <button
    className="filter-clear"
    onClick={() => dispatch(clearFilters())}
  >
    Clear Filters
  </button>
)}
```

- `{hasFilters && <button>...</button>}` — conditional rendering. When `hasFilters` is `true`, render the button; when `false`, the expression evaluates to `false` and React renders nothing.
- `dispatch(clearFilters())` — fires the action with no payload. The reducer resets all four filter fields to null.

The FilterBar only dispatches actions — it doesn't make API calls. The App component listens for filter changes and triggers data fetching. This separation keeps the FilterBar simple and testable.

---

> [!TIP]
> **Session Break** — You've built the layout, sidebar, and filter bar components. Save your work and take a break.
> When you return, you'll build the overview cards, charts, and events table.

---

## Step 4: Overview Cards

Create `src/features/analytics/OverviewCards.tsx`:

```typescript
import { useAppSelector } from '../../app/hooks';
import { selectOverview, selectAnalyticsLoading } from './analyticsSlice';
import { formatNumber, formatCurrency, formatDuration } from '../../utils/formatters';

interface StatCardProps {
  title: string;
  value: string;
  subtitle: string;
}

function StatCard({ title, value, subtitle }: StatCardProps) {
  return (
    <div className="stat-card">
      <h3 className="stat-card-title">{title}</h3>
      <p className="stat-card-value">{value}</p>
      <p className="stat-card-subtitle">{subtitle}</p>
    </div>
  );
}

export default function OverviewCards() {
  const overview = useAppSelector(selectOverview);
  const loading = useAppSelector(selectAnalyticsLoading);

  if (loading && !overview) {
    return (
      <div className="overview-cards">
        {[1, 2, 3, 4].map((i) => (
          <div key={i} className="stat-card stat-card--skeleton" />
        ))}
      </div>
    );
  }

  if (!overview) return null;

  return (
    <div className="overview-cards">
      <StatCard
        title="Page Views"
        value={formatNumber(overview.total_page_views)}
        subtitle="Total page views"
      />
      <StatCard
        title="Signups"
        value={formatNumber(overview.total_signups)}
        subtitle="New registrations"
      />
      <StatCard
        title="Revenue"
        value={formatCurrency(overview.total_revenue)}
        subtitle="Total purchases"
      />
      <StatCard
        title="Avg. Session"
        value={formatDuration(overview.avg_session_duration)}
        subtitle="Minutes per session"
      />
    </div>
  );
}
```

### Reading This File Line-by-Line

```tsx
interface StatCardProps {
  title: string;
  value: string;
  subtitle: string;
}

function StatCard({ title, value, subtitle }: StatCardProps) {
  return (
    <div className="stat-card">
      <h3 className="stat-card-title">{title}</h3>
      <p className="stat-card-value">{value}</p>
      <p className="stat-card-subtitle">{subtitle}</p>
    </div>
  );
}
```

A **private subcomponent**. `StatCard` is defined in the same file but not exported — only `OverviewCards` (the file's default export) uses it. This is a common pattern: keep small presentational helpers next to the component that uses them.

```tsx
const overview = useAppSelector(selectOverview);
const loading = useAppSelector(selectAnalyticsLoading);

if (loading && !overview) {
  return (
    <div className="overview-cards">
      {[1, 2, 3, 4].map((i) => (
        <div key={i} className="stat-card stat-card--skeleton" />
      ))}
    </div>
  );
}

if (!overview) return null;
```

Three rendering states handled by two early returns:

1. **First load** (`loading && !overview`) — show four skeleton placeholders. `[1, 2, 3, 4].map(i => ...)` is a quick way to render N copies of an element. CSS animates these (typically a shimmer or pulse).
2. **No data** (`!overview`) — return `null`, render nothing. Defensive against a state where loading is false but the data is still missing.
3. **Has data** — fall through to the real markup below.

The pattern of "loading AND no previous data = skeleton; otherwise show stale data while refetching" is called **stale-while-revalidate**. It avoids flickering when filters change — the user sees the old numbers continuously, then they update once new data arrives.

```tsx
<StatCard
  title="Page Views"
  value={formatNumber(overview.total_page_views)}
  subtitle="Total page views"
/>
```

Pass formatted strings via props. The card itself doesn't know about formatting — the parent decides. Same pattern for revenue (formatCurrency) and avg session (formatDuration).

Notice the loading state: when data is loading AND there's no previous data, show skeleton cards. This prevents a flash of empty content. When data is loading but we have previous data, keep showing the old data (it will update seamlessly).

---

## Step 5: Charts with Recharts

### Time Series Chart (Line Chart)

Create `src/features/analytics/TimeSeriesChart.tsx`:

```typescript
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Legend,
} from 'recharts';
import { useAppSelector } from '../../app/hooks';
import { selectTimeSeries } from './analyticsSlice';
import { formatDate } from '../../utils/formatters';

export default function TimeSeriesChart() {
  const timeSeries = useAppSelector(selectTimeSeries);

  if (timeSeries.length === 0) {
    return <div className="chart-placeholder">No time series data available.</div>;
  }

  // Format dates for display
  const data = timeSeries.map((point) => ({
    ...point,
    label: formatDate(point.date),
  }));

  return (
    <div className="chart-container">
      <h2 className="chart-title">Page Views Over Time</h2>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#333" />
          <XAxis
            dataKey="label"
            stroke="#888"
            tick={{ fontSize: 12 }}
            interval="preserveStartEnd"
          />
          <YAxis stroke="#888" tick={{ fontSize: 12 }} />
          <Tooltip
            contentStyle={{
              backgroundColor: '#1e1e2e',
              border: '1px solid #333',
              borderRadius: '8px',
            }}
          />
          <Legend />
          <Line
            type="monotone"
            dataKey="page_views"
            name="Page Views"
            stroke="#6366f1"
            strokeWidth={2}
            dot={false}
            activeDot={{ r: 4 }}
          />
          <Line
            type="monotone"
            dataKey="signups"
            name="Signups"
            stroke="#22c55e"
            strokeWidth={2}
            dot={false}
            activeDot={{ r: 4 }}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### Reading TimeSeriesChart Line-by-Line

Recharts is **declarative** — you describe what you want by composing React components, and Recharts handles the SVG drawing.

```tsx
const data = timeSeries.map((point) => ({
  ...point,
  label: formatDate(point.date),
}));
```

- `.map(...)` transforms each point. We **spread** the original point's fields (`...point`) into a new object and add a `label` field. Now each row has both `date` (raw ISO) and `label` (`"Jan 15"`).
- Why? The chart's X-axis shows `label`; tooltips can use `date` for precision. Both live on the same row.

```tsx
<ResponsiveContainer width="100%" height={300}>
  <LineChart data={data}>
```

- `<ResponsiveContainer>` is Recharts' "fill the parent" wrapper. The chart resizes when the window does. Required for any responsive layout.
- `<LineChart data={data}>` — pass the entire array via `data`. Recharts' children automatically read from this prop.

```tsx
<CartesianGrid strokeDasharray="3 3" stroke="#333" />
```

- Background grid lines. `strokeDasharray="3 3"` = 3px dash, 3px gap (dotted look). `stroke` is the line color.

```tsx
<XAxis
  dataKey="label"
  stroke="#888"
  tick={{ fontSize: 12 }}
  interval="preserveStartEnd"
/>
<YAxis stroke="#888" tick={{ fontSize: 12 }} />
```

- `dataKey="label"` tells the X-axis which field of each row to display.
- `tick={{ fontSize: 12 }}` styles the axis labels — passed as a JS object containing SVG attributes.
- `interval="preserveStartEnd"` — when there are too many data points to fit, Recharts auto-skips ticks but always keeps the first and last labels visible.
- The Y-axis has no `dataKey` because Recharts derives its scale from the `<Line>` series below.

```tsx
<Tooltip
  contentStyle={{
    backgroundColor: '#1e1e2e',
    border: '1px solid #333',
    borderRadius: '8px',
  }}
/>
<Legend />
```

- `<Tooltip>` shows the data values when the user hovers. `contentStyle` is the inline style for the tooltip's container.
- `<Legend>` shows which color represents which line, automatically generated from each `<Line>`'s `name` prop.

```tsx
<Line
  type="monotone"
  dataKey="page_views"
  name="Page Views"
  stroke="#6366f1"
  strokeWidth={2}
  dot={false}
  activeDot={{ r: 4 }}
/>
```

- One `<Line>` per data series. We have two — one for page views, one for signups.
- `type="monotone"` — smooth curve interpolation. Other options: `"linear"` (sharp angles), `"step"` (staircase).
- `dataKey="page_views"` — the field on each row to plot.
- `name="Page Views"` — what shows in the legend and tooltip.
- `stroke` — line color. `strokeWidth={2}` — thickness in pixels.
- `dot={false}` — hide the small circle at each data point. Cleaner with many points.
- `activeDot={{ r: 4 }}` — when the user hovers, show a 4px-radius circle on the active point.

Recharts components quick reference:

| Component | Purpose |
|-----------|---------|
| `ResponsiveContainer` | Makes the chart resize with its parent |
| `LineChart` | Container for line-type charts |
| `Line` | A data series rendered as a line |
| `XAxis` / `YAxis` | Axis labels and ticks |
| `CartesianGrid` | Background grid lines |
| `Tooltip` | Shows data values on hover |
| `Legend` | Shows which color represents which data series |

### Category Chart (Bar Chart)

Create `src/features/analytics/CategoryChart.tsx`:

```typescript
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from 'recharts';
import { useAppSelector } from '../../app/hooks';
import { selectCategories } from './analyticsSlice';

export default function CategoryChart() {
  const categories = useAppSelector(selectCategories);

  if (categories.length === 0) {
    return <div className="chart-placeholder">No category data available.</div>;
  }

  return (
    <div className="chart-container chart-container--half">
      <h2 className="chart-title">Events by Category</h2>
      <ResponsiveContainer width="100%" height={250}>
        <BarChart data={categories}>
          <CartesianGrid strokeDasharray="3 3" stroke="#333" />
          <XAxis dataKey="category" stroke="#888" tick={{ fontSize: 12 }} />
          <YAxis stroke="#888" tick={{ fontSize: 12 }} />
          <Tooltip
            contentStyle={{
              backgroundColor: '#1e1e2e',
              border: '1px solid #333',
              borderRadius: '8px',
            }}
          />
          <Bar
            dataKey="count"
            name="Events"
            fill="#6366f1"
            radius={[4, 4, 0, 0]}
          />
        </BarChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### Reading CategoryChart — What's Different

Same shape as TimeSeriesChart with three changes:

- `<BarChart>` instead of `<LineChart>` and `<Bar>` instead of `<Line>`. Recharts swaps in the right SVG primitives.
- `dataKey="category"` on the X-axis — the bar labels come from the `category` field on each row.
- `<Bar>` uses `fill` (interior color) instead of `stroke` (line color). `radius={[4, 4, 0, 0]}` rounds the top corners only — `[topLeft, topRight, bottomRight, bottomLeft]`.

### Device Chart (Pie Chart)

Create `src/features/analytics/DeviceChart.tsx`:

```typescript
import {
  PieChart,
  Pie,
  Cell,
  Tooltip,
  ResponsiveContainer,
  Legend,
} from 'recharts';
import { useAppSelector } from '../../app/hooks';
import { selectDevices } from './analyticsSlice';
import { formatPercentage } from '../../utils/formatters';

const COLORS = ['#6366f1', '#22c55e', '#f59e0b'];

export default function DeviceChart() {
  const devices = useAppSelector(selectDevices);

  if (devices.length === 0) {
    return <div className="chart-placeholder">No device data available.</div>;
  }

  const data = devices.map((d) => ({
    name: d.device,
    value: d.count,
    percentage: formatPercentage(d.percentage),
  }));

  return (
    <div className="chart-container chart-container--half">
      <h2 className="chart-title">By Device</h2>
      <ResponsiveContainer width="100%" height={250}>
        <PieChart>
          <Pie
            data={data}
            cx="50%"
            cy="50%"
            innerRadius={50}
            outerRadius={80}
            paddingAngle={3}
            dataKey="value"
            label={({ name, percentage }) => `${name} ${percentage}`}
          >
            {data.map((_, index) => (
              <Cell key={index} fill={COLORS[index % COLORS.length]} />
            ))}
          </Pie>
          <Tooltip
            contentStyle={{
              backgroundColor: '#1e1e2e',
              border: '1px solid #333',
              borderRadius: '8px',
            }}
          />
          <Legend />
        </PieChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### Reading DeviceChart — What's New

```tsx
const COLORS = ['#6366f1', '#22c55e', '#f59e0b'];
```

Module-level constant array of three hex colors. One per slice (we have three devices: desktop, mobile, tablet).

```tsx
const data = devices.map((d) => ({
  name: d.device,
  value: d.count,
  percentage: formatPercentage(d.percentage),
}));
```

Reshape into the three fields a Recharts pie expects: `name`, `value`, plus our extra `percentage` for the label.

```tsx
<Pie
  data={data}
  cx="50%"
  cy="50%"
  innerRadius={50}
  outerRadius={80}
  paddingAngle={3}
  dataKey="value"
  label={({ name, percentage }) => `${name} ${percentage}`}
>
  {data.map((_, index) => (
    <Cell key={index} fill={COLORS[index % COLORS.length]} />
  ))}
</Pie>
```

- `cx="50%"`, `cy="50%"` — center of the pie. Percent values are relative to the chart container.
- `innerRadius={50}` and `outerRadius={80}` — having a non-zero inner radius makes a **donut chart** instead of a solid pie.
- `paddingAngle={3}` — gap (in degrees) between slices.
- `label={({ name, percentage }) => \`${name} ${percentage}\`}` — a function that returns the label string for each slice. Recharts passes the slice's data; we destructure `name` and `percentage`.
- `{data.map((_, index) => <Cell ... />)}` — `<Cell>` lets us assign a different color per slice. The underscore `_` means "I'm not using the first parameter (the data point)" — only `index` matters here.
- `COLORS[index % COLORS.length]` — modulo wraps around safely. If we had 5 slices and 3 colors, slice 4 would reuse color 1 (`4 % 3 = 1`).

---

> [!TIP]
> **Session Break** — You've built the overview cards and all three chart components with Recharts. Save your work and take a break.
> When you return, you'll build the events table with pagination and the App component.

---

## Step 6: Events Table

Create `src/features/events/EventsTable.tsx`:

```typescript
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  fetchEvents,
  selectEvents,
  selectEventsPage,
  selectEventsTotalPages,
  selectEventsTotal,
  selectEventsLoading,
  setPage,
} from './eventsSlice';
import { selectActiveFilters } from '../filters/filtersSlice';
import { formatDateTime } from '../../utils/formatters';

export default function EventsTable() {
  const dispatch = useAppDispatch();
  const events = useAppSelector(selectEvents);
  const page = useAppSelector(selectEventsPage);
  const totalPages = useAppSelector(selectEventsTotalPages);
  const total = useAppSelector(selectEventsTotal);
  const loading = useAppSelector(selectEventsLoading);
  const filters = useAppSelector(selectActiveFilters);

  // Fetch events when page or filters change
  useEffect(() => {
    dispatch(fetchEvents({ filters, page, limit: 20 }));
  }, [dispatch, filters, page]);

  return (
    <div className="events-table-container">
      <div className="events-header">
        <h2 className="chart-title">Events</h2>
        <span className="events-count">{total.toLocaleString()} total events</span>
      </div>

      {loading && events.length === 0 ? (
        <div className="events-loading">Loading events...</div>
      ) : (
        <>
          <table className="events-table">
            <thead>
              <tr>
                <th>Type</th>
                <th>Category</th>
                <th>Page</th>
                <th>Device</th>
                <th>Country</th>
                <th>Value</th>
                <th>Date</th>
              </tr>
            </thead>
            <tbody>
              {events.map((event) => (
                <tr key={event.id}>
                  <td>
                    <span className={`event-badge event-badge--${event.event_type}`}>
                      {event.event_type}
                    </span>
                  </td>
                  <td>{event.category}</td>
                  <td className="events-page-cell">{event.page}</td>
                  <td>{event.device}</td>
                  <td>{event.country}</td>
                  <td>
                    {event.value > 0 ? `$${event.value}` : '—'}
                  </td>
                  <td className="events-date-cell">
                    {formatDateTime(event.created_at)}
                  </td>
                </tr>
              ))}
            </tbody>
          </table>

          {/* Pagination */}
          <div className="pagination">
            <button
              className="pagination-btn"
              disabled={page <= 1}
              onClick={() => dispatch(setPage(page - 1))}
            >
              Previous
            </button>

            <span className="pagination-info">
              Page {page} of {totalPages}
            </span>

            <button
              className="pagination-btn"
              disabled={page >= totalPages}
              onClick={() => dispatch(setPage(page + 1))}
            >
              Next
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

### Reading This File Line-by-Line

```tsx
const dispatch = useAppDispatch();
const events = useAppSelector(selectEvents);
const page = useAppSelector(selectEventsPage);
const totalPages = useAppSelector(selectEventsTotalPages);
const total = useAppSelector(selectEventsTotal);
const loading = useAppSelector(selectEventsLoading);
const filters = useAppSelector(selectActiveFilters);
```

Six pieces pulled from Redux — six independent subscriptions. Each `useAppSelector` re-renders this component only when its specific value changes. So toggling a filter triggers a refetch but doesn't re-render if the events haven't arrived yet.

```tsx
useEffect(() => {
  dispatch(fetchEvents({ filters, page, limit: 20 }));
}, [dispatch, filters, page]);
```

A **multi-dependency `useEffect`**. The dependency array `[dispatch, filters, page]` tells React: "re-run this effect any time any of these values changes."

- On **mount** — the effect runs once with the initial filters and page.
- When the **page** changes (user clicks Next) — it runs again with the new page.
- When **filters** change (user picks a date range) — it runs again with the new filters.
- `dispatch` is included for correctness (lint rule). It's stable across renders, so it never actually causes a refetch.

This is how Redux state drives data fetching: the component watches the store and refetches whenever inputs change. No manual "after I dispatch X, also dispatch Y" logic needed.

```tsx
{loading && events.length === 0 ? (
  <div className="events-loading">Loading events...</div>
) : (
  <>
    <table className="events-table">...</table>
    <div className="pagination">...</div>
  </>
)}
```

Same stale-while-revalidate pattern. Show "Loading events..." only on first load (loading AND empty events). When events exist, keep showing them while a new request is in flight.

```tsx
{events.map((event) => (
  <tr key={event.id}>
    ...
    <td>
      {event.value > 0 ? `$${event.value}` : '—'}
    </td>
    ...
  </tr>
))}
```

Standard `.map` pattern. The ternary `event.value > 0 ? \`$${event.value}\` : '—'` shows a dollar amount for purchases and an em-dash placeholder for everything else.

```tsx
<button
  className="pagination-btn"
  disabled={page <= 1}
  onClick={() => dispatch(setPage(page - 1))}
>
  Previous
</button>
```

`disabled={page <= 1}` — the button greys out when there's no previous page. `onClick` dispatches `setPage(page - 1)`, the synchronous Redux action that updates the page number. The change in the page number triggers our `useEffect` to fetch the new page's data.

The EventsTable uses `useEffect` to fetch data whenever `page` or `filters` change. The dependency array `[dispatch, filters, page]` ensures refetching when navigation or filtering occurs.

---

> [!TIP]
> **Session Break** — You've built the events table with pagination and filter integration. Save your work and take a break.
> When you return, you'll wire everything together in the App component and add styles.

---

## Step 7: App Component

Update `src/App.tsx`:

```typescript
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from './app/hooks';
import { fetchAllAnalytics } from './features/analytics/analyticsSlice';
import { selectActiveFilters } from './features/filters/filtersSlice';
import { selectActiveView } from './features/ui/uiSlice';
import Layout from './features/ui/Layout';
import FilterBar from './features/filters/FilterBar';
import OverviewCards from './features/analytics/OverviewCards';
import TimeSeriesChart from './features/analytics/TimeSeriesChart';
import CategoryChart from './features/analytics/CategoryChart';
import DeviceChart from './features/analytics/DeviceChart';
import EventsTable from './features/events/EventsTable';

export default function App() {
  const dispatch = useAppDispatch();
  const filters = useAppSelector(selectActiveFilters);
  const activeView = useAppSelector(selectActiveView);

  // Fetch analytics data on mount and when filters change
  useEffect(() => {
    dispatch(fetchAllAnalytics(filters));
  }, [dispatch, filters]);

  return (
    <Layout>
      <FilterBar />

      {activeView === 'overview' && (
        <>
          <OverviewCards />
          <TimeSeriesChart />
          <div className="charts-row">
            <CategoryChart />
            <DeviceChart />
          </div>
        </>
      )}

      {activeView === 'events' && <EventsTable />}
    </Layout>
  );
}
```

### Reading This File Line-by-Line

```tsx
useEffect(() => {
  dispatch(fetchAllAnalytics(filters));
}, [dispatch, filters]);
```

The **single source of truth** for the dashboard's data flow. Whenever the active filters change in Redux, this effect refetches all analytics. Because `fetchAllAnalytics` is the parallel thunk you wrote in Step 4, all five charts update simultaneously.

```tsx
{activeView === 'overview' && (
  <>
    <OverviewCards />
    <TimeSeriesChart />
    <div className="charts-row">
      <CategoryChart />
      <DeviceChart />
    </div>
  </>
)}

{activeView === 'events' && <EventsTable />}
```

Two **conditional renders** based on `activeView`:

- `condition && <Component />` — when condition is true, render the component; otherwise nothing.
- `<>...</>` — a **React fragment**. Lets us return multiple sibling elements from the conditional without adding an extra `<div>` to the DOM.
- The two ternaries are mutually exclusive — only one block renders at any time, depending on which view the user clicked in the sidebar.

The App component is thin — it only orchestrates:

1. **Fetches data** when filters change (via `useEffect`)
2. **Renders the active view** based on Redux UI state
3. **Composes components** inside the Layout

---

## Step 8: Styles

Replace `src/App.css` with the complete dashboard styles:

> **Already comfortable with CSS?** Skim this for the new techniques: `position: fixed`, `display: grid`, `grid-template-columns`, `flex-direction`, transitions, BEM-style modifier classes (`.layout--collapsed`). If CSS basics still feel new, see Level 1 Step 4's "How to Read This CSS" primer — it covers selectors, properties, units, and colors.

```css
/* --- Reset & Base --- */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #0f0f1a;
  color: #e0e0e0;
  line-height: 1.6;
}

/* --- Layout --- */

.layout {
  display: flex;
  min-height: 100vh;
}

.main-content {
  flex: 1;
  padding: 24px;
  margin-left: 220px;
  transition: margin-left 0.3s ease;
}

.layout--collapsed .main-content {
  margin-left: 60px;
}

/* --- Sidebar --- */

.sidebar {
  position: fixed;
  left: 0;
  top: 0;
  width: 220px;
  height: 100vh;
  background-color: #1a1a2e;
  border-right: 1px solid #2a2a4a;
  display: flex;
  flex-direction: column;
  transition: width 0.3s ease;
  z-index: 10;
}

.layout--collapsed .sidebar {
  width: 60px;
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px;
  border-bottom: 1px solid #2a2a4a;
}

.sidebar-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: #6366f1;
}

.layout--collapsed .sidebar-title {
  display: none;
}

.sidebar-toggle {
  background: none;
  border: none;
  color: #888;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
}

.sidebar-toggle:hover {
  background-color: #2a2a4a;
  color: #e0e0e0;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  padding: 8px;
  gap: 4px;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  border: none;
  background: none;
  color: #888;
  font-size: 0.9rem;
  cursor: pointer;
  border-radius: 8px;
  transition: all 0.2s;
  text-align: left;
  width: 100%;
}

.nav-item:hover {
  background-color: #2a2a4a;
  color: #e0e0e0;
}

.nav-item--active {
  background-color: #6366f1;
  color: #fff;
}

.nav-icon {
  font-size: 1.1rem;
}

.layout--collapsed .nav-label {
  display: none;
}

/* --- Filter Bar --- */

.filter-bar {
  display: flex;
  align-items: flex-end;
  gap: 16px;
  padding: 16px;
  background-color: #1a1a2e;
  border-radius: 12px;
  margin-bottom: 24px;
  flex-wrap: wrap;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.filter-label {
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #888;
}

.filter-buttons {
  display: flex;
  gap: 4px;
}

.filter-btn {
  padding: 6px 12px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
}

.filter-btn:hover {
  border-color: #6366f1;
  color: #6366f1;
}

.filter-select {
  padding: 6px 12px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.85rem;
  cursor: pointer;
}

.filter-select:focus {
  outline: none;
  border-color: #6366f1;
}

.filter-clear {
  padding: 6px 12px;
  border: 1px solid #ef4444;
  background: none;
  color: #ef4444;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
  align-self: flex-end;
}

.filter-clear:hover {
  background-color: #ef4444;
  color: #fff;
}

/* --- Overview Cards --- */

.overview-cards {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.stat-card {
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 20px;
}

.stat-card--skeleton {
  height: 110px;
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 0.4; }
  50% { opacity: 0.8; }
}

.stat-card-title {
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #888;
  margin-bottom: 8px;
}

.stat-card-value {
  font-size: 1.8rem;
  font-weight: 700;
  color: #e0e0e0;
  margin-bottom: 4px;
}

.stat-card-subtitle {
  font-size: 0.8rem;
  color: #666;
}

/* --- Charts --- */

.chart-container {
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 24px;
}

.chart-container--half {
  flex: 1;
  min-width: 0;
}

.charts-row {
  display: flex;
  gap: 24px;
  margin-bottom: 24px;
}

.chart-title {
  font-size: 1rem;
  font-weight: 600;
  color: #e0e0e0;
  margin-bottom: 16px;
}

.chart-placeholder {
  text-align: center;
  padding: 60px 20px;
  color: #666;
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
}

/* Recharts tooltip/legend overrides */
.recharts-default-tooltip {
  background-color: #1e1e2e !important;
  border: 1px solid #333 !important;
  border-radius: 8px !important;
}

.recharts-tooltip-label {
  color: #e0e0e0 !important;
}

.recharts-legend-item-text {
  color: #888 !important;
}

/* --- Events Table --- */

.events-table-container {
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 20px;
}

.events-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.events-count {
  font-size: 0.85rem;
  color: #888;
}

.events-loading {
  text-align: center;
  padding: 40px;
  color: #888;
}

.events-table {
  width: 100%;
  border-collapse: collapse;
}

.events-table th {
  text-align: left;
  padding: 10px 12px;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #888;
  border-bottom: 1px solid #2a2a4a;
}

.events-table td {
  padding: 10px 12px;
  font-size: 0.85rem;
  border-bottom: 1px solid #1a1a2e;
  color: #ccc;
}

.events-table tr:hover td {
  background-color: #22223a;
}

.events-page-cell {
  font-family: monospace;
  font-size: 0.8rem;
}

.events-date-cell {
  white-space: nowrap;
  font-size: 0.8rem;
  color: #888;
}

/* Event type badges */
.event-badge {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
}

.event-badge--page_view {
  background-color: rgba(99, 102, 241, 0.2);
  color: #818cf8;
}

.event-badge--signup {
  background-color: rgba(34, 197, 94, 0.2);
  color: #4ade80;
}

.event-badge--purchase {
  background-color: rgba(245, 158, 11, 0.2);
  color: #fbbf24;
}

.event-badge--click {
  background-color: rgba(136, 136, 136, 0.2);
  color: #aaa;
}

/* --- Pagination --- */

.pagination {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  padding: 16px 0 0;
}

.pagination-btn {
  padding: 6px 16px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
}

.pagination-btn:hover:not(:disabled) {
  border-color: #6366f1;
  color: #6366f1;
}

.pagination-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.pagination-info {
  font-size: 0.85rem;
  color: #888;
}

/* --- Responsive --- */

@media (max-width: 1024px) {
  .overview-cards {
    grid-template-columns: repeat(2, 1fr);
  }

  .charts-row {
    flex-direction: column;
  }
}

@media (max-width: 768px) {
  .sidebar {
    width: 60px;
  }

  .sidebar-title,
  .nav-label {
    display: none;
  }

  .main-content {
    margin-left: 60px;
  }

  .overview-cards {
    grid-template-columns: 1fr;
  }

  .filter-bar {
    flex-direction: column;
    align-items: stretch;
  }
}
```

---

## Step 9: Remove Boilerplate

Delete the Vite default files you don't need:

```bash
rm -f src/assets/react.svg public/vite.svg src/index.css
```

Update `src/main.tsx` to remove the default `index.css` import (if present). The final `main.tsx` should match what we wrote in Step 4 of the State Management lesson.

---

## Step 10: Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: add dashboard components with Recharts, sidebar navigation, and events table"
```

---

## Step 11: Test It

Start both servers:

**Terminal 1:**

```bash
cd server && npm run dev
```

**Terminal 2:**

```bash
cd client && npm run dev
```

Open `http://localhost:5173`. You should see:

1. **Sidebar** with Overview and Events links
2. **Filter bar** with 30d/60d/90d buttons, category and device dropdowns
3. **Four KPI cards** with page views, signups, revenue, and avg session time
4. **Line chart** showing page views and signups over time
5. **Bar chart** showing events by category
6. **Pie chart** showing device distribution

Click **Events** in the sidebar → see the paginated events table.

Click **30d** → all charts and cards update to show only the last 30 days.

Select **Category: marketing** → everything filters to marketing events.

Click **Clear Filters** → back to all data.

If anything doesn't render, open the browser console and check for errors. Common issues:

| Problem | Fix |
|---------|-----|
| "Cannot read property 'map' of undefined" | Check that the API is returning the expected shape |
| Charts don't appear | Verify Recharts is installed (`npm list recharts`) |
| Filter changes don't update charts | Check that `selectActiveFilters` is in the `useEffect` dependency array |
| No data at all | Did you run `npm run seed` in the server directory? |

---

> [!TIP]
> ## Spatial Check-In

1. **Why does the App component dispatch `fetchAllAnalytics` instead of each component fetching its own data?**

<details><summary>Answer</summary>

Centralized fetching ensures all components update from the same API call with the same filters. If each component fetched independently, they could get out of sync — the bar chart might show marketing data while the line chart still shows all categories because its request was slower.

</details>

2. **Why does the EventsTable have its own `useEffect` for fetching?**

<details><summary>Answer</summary>

The events table has its own pagination state — when the user clicks "Next," only the events table needs to refetch. The overview cards and charts don't change when the page changes. Giving EventsTable its own fetch keeps the pagination isolated from the overview data.

</details>

3. **Why are the charts in a `charts-row` div with `display: flex`?**

<details><summary>Answer</summary>

The category bar chart and device pie chart should appear side by side on desktop screens. Flexbox with `flex: 1` on each chart makes them share the row equally. On smaller screens, the CSS media query switches to `flex-direction: column` so they stack vertically.

</details>

---

| | | |
|:---|:---:|---:|
| [← 04 — State Management](../04-state/) | [Level 4 Overview](../) | [06 — Testing →](../06-testing/) |
