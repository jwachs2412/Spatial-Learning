`Level 4` **Step 5 of 9** — Frontend

# 05 — Frontend: Dashboard Components with Recharts

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

The FilterBar only dispatches actions — it doesn't make API calls. The App component listens for filter changes and triggers data fetching. This separation keeps the FilterBar simple and testable.

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

Key Recharts concepts:

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

The EventsTable uses `useEffect` to fetch data whenever `page` or `filters` change. The dependency array `[dispatch, filters, page]` ensures refetching when navigation or filtering occurs.

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

The App component is thin — it only orchestrates:

1. **Fetches data** when filters change (via `useEffect`)
2. **Renders the active view** based on Redux UI state
3. **Composes components** inside the Layout

---

## Step 8: Styles

Replace `src/App.css` with the complete dashboard styles:

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
