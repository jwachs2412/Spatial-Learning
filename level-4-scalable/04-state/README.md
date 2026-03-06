`Level 4` **Step 4 of 9** — State Management

# 04 — State Management: Redux Toolkit

## Spatial Orientation

Redux manages all application state in a single **store**. Components read from the store with **selectors** and update it by dispatching **actions**. Async operations (API calls) use **thunks**.

```
┌────────────────────────────────────────────────────────────────┐
│                         REDUX STORE                            │
│                                                                │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  analytics   │  │   events     │  │      filters         │ │
│  │  Slice       │  │   Slice      │  │      Slice           │ │
│  │             │  │             │  │                      │ │
│  │  overview   │  │  events[]   │  │  startDate           │ │
│  │  timeSeries │  │  total      │  │  endDate             │ │
│  │  categories │  │  page       │  │  category            │ │
│  │  devices    │  │  totalPages │  │  device              │ │
│  │  topPages   │  │  loading    │  │                      │ │
│  │  loading    │  │  error      │  │                      │ │
│  │  error      │  │             │  │                      │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬───────────┘ │
│         │                │                     │              │
│  ┌──────▼────────────────▼─────────────────────▼───────────┐ │
│  │                    SELECTORS                             │ │
│  │   selectOverview    selectEvents    selectFilters        │ │
│  │   selectTimeSeries  selectTotal     selectActiveFilters  │ │
│  │   selectLoading     selectPage                          │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           │                                   │
└───────────────────────────┼───────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         OverviewCards  EventsTable  FilterBar
         TimeSeriesChart
         CategoryChart
         DeviceChart
```

---

## Redux Toolkit Concepts

Before writing code, understand four concepts:

### 1. Store

The single JavaScript object that holds all application state.

```
store = {
  analytics: { overview: {...}, timeSeries: [...], loading: false },
  events:    { events: [...], total: 100, page: 1, loading: false },
  filters:   { startDate: null, endDate: null, category: null },
}
```

### 2. Slice

A slice defines a piece of state, the actions that modify it, and the reducer that handles those actions. Redux Toolkit's `createSlice` generates all three from one definition.

### 3. Thunk

An async function that can dispatch actions. Used for API calls: dispatch "loading" → make API call → dispatch "success" or "error."

### 4. Selector

A function that extracts a specific piece of state from the store. Components use selectors to read only the data they need.

> [!NOTE]
> **Technical:** Redux follows a unidirectional data flow: Component → dispatches Action → Reducer updates Store → Component re-reads via Selector. Thunks add async operations between dispatch and reducer.
>
> **Plain English:** Think of Redux like a bank. The store is the vault. Actions are deposit/withdrawal slips. The reducer is the teller who processes slips and updates the vault. Selectors are account statements — each customer only sees their own balance. Thunks are when you need to call another bank before completing the transaction.

---

## Step 1: Configure the Store

> [!IMPORTANT]
> **You should be in:** `data-dash/client/`

Create `src/app/store.ts`:

```typescript
import { configureStore } from '@reduxjs/toolkit';
import analyticsReducer from '../features/analytics/analyticsSlice';
import eventsReducer from '../features/events/eventsSlice';
import filtersReducer from '../features/filters/filtersSlice';

export const store = configureStore({
  reducer: {
    analytics: analyticsReducer,
    events: eventsReducer,
    filters: filtersReducer,
  },
});

// TypeScript types for the store
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

Line-by-line:

| Line | Purpose |
|------|---------|
| `configureStore` | Creates the Redux store with good defaults (DevTools, middleware) |
| `reducer: { ... }` | Combines slices — each key becomes a top-level state property |
| `RootState` | TypeScript type representing the entire store shape |
| `AppDispatch` | TypeScript type for the dispatch function (includes thunk support) |

### Typed Hooks

Create `src/app/hooks.ts`:

```typescript
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

// Use these throughout the app instead of plain useDispatch/useSelector
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

> [!NOTE]
> **Technical:** `useDispatch.withTypes<AppDispatch>()` creates a typed hook that knows about all your thunks. `useSelector.withTypes<RootState>()` creates a hook where the state parameter is automatically typed. Without these, you'd need to cast types on every usage.
>
> **Plain English:** These are convenience wrappers that tell TypeScript "I know exactly what shape my store is." Without them, TypeScript would complain everywhere you use Redux.

---

## Step 2: Filters Slice

Start with the simplest slice. Filters are synchronous — no API calls needed.

Create `src/features/filters/filtersSlice.ts`:

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { AnalyticsFilters } from '../../types';

interface FiltersState {
  startDate: string | null;
  endDate: string | null;
  category: string | null;
  device: string | null;
}

const initialState: FiltersState = {
  startDate: null,
  endDate: null,
  category: null,
  device: null,
};

const filtersSlice = createSlice({
  name: 'filters',
  initialState,
  reducers: {
    setDateRange(
      state,
      action: PayloadAction<{ startDate: string; endDate: string }>
    ) {
      state.startDate = action.payload.startDate;
      state.endDate = action.payload.endDate;
    },
    setCategory(state, action: PayloadAction<string | null>) {
      state.category = action.payload;
    },
    setDevice(state, action: PayloadAction<string | null>) {
      state.device = action.payload;
    },
    clearFilters(state) {
      state.startDate = null;
      state.endDate = null;
      state.category = null;
      state.device = null;
    },
  },
});

// --- Actions ---
export const { setDateRange, setCategory, setDevice, clearFilters } =
  filtersSlice.actions;

// --- Selectors ---
export const selectFilters = (state: RootState): FiltersState => state.filters;

export const selectActiveFilters = (state: RootState): AnalyticsFilters => {
  const { startDate, endDate, category, device } = state.filters;
  const filters: AnalyticsFilters = {};
  if (startDate) filters.startDate = startDate;
  if (endDate) filters.endDate = endDate;
  if (category) filters.category = category;
  if (device) filters.device = device;
  return filters;
};

export const selectHasActiveFilters = (state: RootState): boolean => {
  const { startDate, endDate, category, device } = state.filters;
  return !!(startDate || endDate || category || device);
};

export default filtersSlice.reducer;
```

### Anatomy of a Slice

```
createSlice({
  name: 'filters',           ← Prefix for action types (filters/setCategory)
  initialState,               ← Default state when app loads
  reducers: {                 ← Functions that modify state
    setCategory(state, action) {
      state.category = action.payload;   ← Looks like mutation, but isn't
    },
  },
})
```

> [!WARNING]
> **`state.category = action.payload` looks like mutation, but it isn't.** Redux Toolkit uses Immer under the hood. Immer tracks your "mutations" and produces a new immutable state object. You write mutating code; Immer makes it immutable. This is why Redux Toolkit exists — plain Redux required you to spread objects manually (`return { ...state, category: action.payload }`).

### What This Slice Exports

| Export | Type | Purpose |
|--------|------|---------|
| `setDateRange` | Action creator | Dispatched when user picks date range |
| `setCategory` | Action creator | Dispatched when user selects category |
| `setDevice` | Action creator | Dispatched when user selects device |
| `clearFilters` | Action creator | Dispatched when user clicks "Clear" |
| `selectFilters` | Selector | Returns full filter state |
| `selectActiveFilters` | Selector | Returns only non-null filters (for API calls) |
| `selectHasActiveFilters` | Selector | Returns boolean (for UI — show "Clear" button) |
| `default` | Reducer | Passed to configureStore |

---

> [!TIP]
> **Session Break** — You've configured the Redux store and built the filters slice with actions and selectors. Save your work and take a break.
> When you return, you'll build the analytics and events slices with async thunks.

---

## Step 3: Analytics Slice

This slice handles async API calls with `createAsyncThunk`.

Create `src/features/analytics/analyticsSlice.ts`:

```typescript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type {
  OverviewStats,
  TimeSeriesPoint,
  CategoryBreakdown,
  DeviceBreakdown,
  TopPage,
  AnalyticsFilters,
} from '../../types';
import * as api from '../../services/api';

// --- State shape ---

interface AnalyticsState {
  overview: OverviewStats | null;
  timeSeries: TimeSeriesPoint[];
  categories: CategoryBreakdown[];
  devices: DeviceBreakdown[];
  topPages: TopPage[];
  loading: boolean;
  error: string | null;
}

const initialState: AnalyticsState = {
  overview: null,
  timeSeries: [],
  categories: [],
  devices: [],
  topPages: [],
  loading: false,
  error: null,
};

// --- Async thunks ---

export const fetchOverview = createAsyncThunk(
  'analytics/fetchOverview',
  async (filters: AnalyticsFilters) => {
    return api.getOverview(filters);
  }
);

export const fetchTimeSeries = createAsyncThunk(
  'analytics/fetchTimeSeries',
  async (filters: AnalyticsFilters) => {
    return api.getTimeSeries(filters);
  }
);

export const fetchCategories = createAsyncThunk(
  'analytics/fetchCategories',
  async (filters: AnalyticsFilters) => {
    return api.getCategories(filters);
  }
);

export const fetchDevices = createAsyncThunk(
  'analytics/fetchDevices',
  async (filters: AnalyticsFilters) => {
    return api.getDevices(filters);
  }
);

export const fetchTopPages = createAsyncThunk(
  'analytics/fetchTopPages',
  async (filters: AnalyticsFilters) => {
    return api.getTopPages(filters);
  }
);

// Fetch all analytics data at once
export const fetchAllAnalytics = createAsyncThunk(
  'analytics/fetchAll',
  async (filters: AnalyticsFilters, { dispatch }) => {
    await Promise.all([
      dispatch(fetchOverview(filters)),
      dispatch(fetchTimeSeries(filters)),
      dispatch(fetchCategories(filters)),
      dispatch(fetchDevices(filters)),
      dispatch(fetchTopPages(filters)),
    ]);
  }
);

// --- Slice ---

const analyticsSlice = createSlice({
  name: 'analytics',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    // --- fetchOverview ---
    builder
      .addCase(fetchOverview.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchOverview.fulfilled, (state, action) => {
        state.overview = action.payload;
        state.loading = false;
      })
      .addCase(fetchOverview.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch overview';
      });

    // --- fetchTimeSeries ---
    builder
      .addCase(fetchTimeSeries.fulfilled, (state, action) => {
        state.timeSeries = action.payload;
      });

    // --- fetchCategories ---
    builder
      .addCase(fetchCategories.fulfilled, (state, action) => {
        state.categories = action.payload;
      });

    // --- fetchDevices ---
    builder
      .addCase(fetchDevices.fulfilled, (state, action) => {
        state.devices = action.payload;
      });

    // --- fetchTopPages ---
    builder
      .addCase(fetchTopPages.fulfilled, (state, action) => {
        state.topPages = action.payload;
      });
  },
});

// --- Selectors ---

export const selectOverview = (state: RootState) => state.analytics.overview;
export const selectTimeSeries = (state: RootState) => state.analytics.timeSeries;
export const selectCategories = (state: RootState) => state.analytics.categories;
export const selectDevices = (state: RootState) => state.analytics.devices;
export const selectTopPages = (state: RootState) => state.analytics.topPages;
export const selectAnalyticsLoading = (state: RootState) => state.analytics.loading;
export const selectAnalyticsError = (state: RootState) => state.analytics.error;

export default analyticsSlice.reducer;
```

### How createAsyncThunk Works

```
createAsyncThunk('analytics/fetchOverview', async (filters) => {
  return api.getOverview(filters);   // Returns a Promise
})
```

This generates THREE action types automatically:

```
analytics/fetchOverview/pending    → dispatched before the API call
analytics/fetchOverview/fulfilled  → dispatched when the API call succeeds
analytics/fetchOverview/rejected   → dispatched when the API call fails
```

You handle these in `extraReducers` to update loading/success/error state.

```
THUNK LIFECYCLE:

  dispatch(fetchOverview(filters))
       │
       ▼
  pending action → state.loading = true
       │
       ▼
  API call: GET /api/analytics/overview?category=...
       │
       ├── Success → fulfilled action → state.overview = data, loading = false
       │
       └── Failure → rejected action → state.error = message, loading = false
```

> [!NOTE]
> **Technical:** `createAsyncThunk` wraps your async function with automatic dispatching of pending/fulfilled/rejected actions. The `extraReducers` builder pattern handles these generated actions. This replaces the manual "dispatch loading, try/catch, dispatch success/error" pattern you'd write by hand.
>
> **Plain English:** Instead of manually saying "tell everyone I'm loading," then making the call, then "tell everyone I'm done" — the thunk does all three steps automatically. You just write the API call.

---

> [!TIP]
> **Session Break** — You've built the analytics slice with async thunks for all five API endpoints. Save your work and take a break.
> When you return, you'll build the events slice, API service layer, and wire up the store.

---

## Step 4: Events Slice

Create `src/features/events/eventsSlice.ts`:

```typescript
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { AnalyticsEvent, AnalyticsFilters } from '../../types';
import * as api from '../../services/api';

// --- State shape ---

interface EventsState {
  events: AnalyticsEvent[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
  loading: boolean;
  error: string | null;
}

const initialState: EventsState = {
  events: [],
  total: 0,
  page: 1,
  limit: 20,
  totalPages: 0,
  loading: false,
  error: null,
};

// --- Async thunks ---

export const fetchEvents = createAsyncThunk(
  'events/fetchEvents',
  async ({
    filters,
    page,
    limit,
  }: {
    filters: AnalyticsFilters;
    page: number;
    limit: number;
  }) => {
    return api.getEvents(filters, page, limit);
  }
);

// --- Slice ---

const eventsSlice = createSlice({
  name: 'events',
  initialState,
  reducers: {
    setPage(state, action: PayloadAction<number>) {
      state.page = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEvents.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchEvents.fulfilled, (state, action) => {
        state.events = action.payload.events;
        state.total = action.payload.total;
        state.page = action.payload.page;
        state.limit = action.payload.limit;
        state.totalPages = action.payload.total_pages;
        state.loading = false;
      })
      .addCase(fetchEvents.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch events';
      });
  },
});

// --- Actions ---
export const { setPage } = eventsSlice.actions;

// --- Selectors ---
export const selectEvents = (state: RootState) => state.events.events;
export const selectEventsTotal = (state: RootState) => state.events.total;
export const selectEventsPage = (state: RootState) => state.events.page;
export const selectEventsTotalPages = (state: RootState) => state.events.totalPages;
export const selectEventsLoading = (state: RootState) => state.events.loading;
export const selectEventsError = (state: RootState) => state.events.error;

export default eventsSlice.reducer;
```

---

## Step 5: API Service Layer

Create `src/services/api.ts`:

```typescript
import type {
  OverviewStats,
  TimeSeriesPoint,
  CategoryBreakdown,
  DeviceBreakdown,
  TopPage,
  PaginatedEvents,
  AnalyticsFilters,
} from '../types';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

// --- Helper: Build query string from filters ---

function buildQueryString(filters: AnalyticsFilters, extra?: Record<string, string | number>): string {
  const params = new URLSearchParams();

  if (filters.startDate) params.append('startDate', filters.startDate);
  if (filters.endDate) params.append('endDate', filters.endDate);
  if (filters.category) params.append('category', filters.category);
  if (filters.device) params.append('device', filters.device);

  if (extra) {
    for (const [key, value] of Object.entries(extra)) {
      params.append(key, String(value));
    }
  }

  const qs = params.toString();
  return qs ? `?${qs}` : '';
}

// --- Helper: Fetch with error handling ---

async function fetchJSON<T>(url: string): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    const error = await response.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(error.error || `HTTP ${response.status}`);
  }

  return response.json();
}

// --- API functions ---

export async function getOverview(filters: AnalyticsFilters): Promise<OverviewStats> {
  return fetchJSON<OverviewStats>(
    `${API_URL}/analytics/overview${buildQueryString(filters)}`
  );
}

export async function getTimeSeries(filters: AnalyticsFilters): Promise<TimeSeriesPoint[]> {
  return fetchJSON<TimeSeriesPoint[]>(
    `${API_URL}/analytics/time-series${buildQueryString(filters)}`
  );
}

export async function getCategories(filters: AnalyticsFilters): Promise<CategoryBreakdown[]> {
  return fetchJSON<CategoryBreakdown[]>(
    `${API_URL}/analytics/categories${buildQueryString(filters)}`
  );
}

export async function getDevices(filters: AnalyticsFilters): Promise<DeviceBreakdown[]> {
  return fetchJSON<DeviceBreakdown[]>(
    `${API_URL}/analytics/devices${buildQueryString(filters)}`
  );
}

export async function getTopPages(filters: AnalyticsFilters): Promise<TopPage[]> {
  return fetchJSON<TopPage[]>(
    `${API_URL}/analytics/top-pages${buildQueryString(filters)}`
  );
}

export async function getEvents(
  filters: AnalyticsFilters,
  page: number = 1,
  limit: number = 20
): Promise<PaginatedEvents> {
  return fetchJSON<PaginatedEvents>(
    `${API_URL}/analytics/events${buildQueryString(filters, { page, limit })}`
  );
}
```

### Pattern: API → Thunk → Slice → Component

```
api.getOverview(filters)           ← API service (fetch call)
  ↑
fetchOverview(filters)             ← Thunk (dispatches pending/fulfilled/rejected)
  ↑
dispatch(fetchOverview(filters))   ← Component (triggers the chain)
  │
  ▼
selectOverview(state)              ← Selector (reads the result)
  │
  ▼
<OverviewCards overview={...} />   ← Component (renders the data)
```

---

## Step 6: Wire Up the Store

Update `src/main.tsx` to provide the Redux store:

```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import { Provider } from 'react-redux';
import { store } from './app/store';
import App from './App';
import './App.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Provider store={store}>
      <App />
    </Provider>
  </React.StrictMode>
);
```

The `<Provider>` wraps the entire app, making the Redux store available to any component that uses `useAppSelector` or `useAppDispatch`.

---

## Step 7: Verify with Redux DevTools

Install the [Redux DevTools browser extension](https://github.com/reduxjs/redux-devtools) for Chrome or Firefox.

Open your app and the DevTools extension. You should see:

```
State:
  analytics:
    overview: null
    timeSeries: []
    categories: []
    devices: []
    topPages: []
    loading: false
    error: null
  events:
    events: []
    total: 0
    page: 1
    limit: 20
    totalPages: 0
    loading: false
    error: null
  filters:
    startDate: null
    endDate: null
    category: null
    device: null
```

This is the initial state — empty because no thunks have been dispatched yet. Once we build the frontend components, you'll see the state populate and actions flow through in real time.

> [!NOTE]
> **Technical:** Redux DevTools lets you inspect every action dispatched, see state before and after each action, and even "time travel" — replay actions to reproduce bugs. `configureStore` enables DevTools automatically in development.
>
> **Plain English:** Redux DevTools is like a security camera for your app's state. You can see every change that happened, rewind to any point, and understand exactly how your app reached its current state.

---

## Step 8: Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: add Redux store with analytics, events, and filters slices"
```

---

> [!TIP]
> ## Spatial Check-In

1. **What happens when you dispatch `setCategory("marketing")`?**

<details><summary>Answer</summary>

The dispatch sends the action `{ type: "filters/setCategory", payload: "marketing" }` to the store. The filtersSlice reducer runs and sets `state.category = "marketing"`. Any component using `selectFilters` or `selectActiveFilters` re-renders with the new value. Components using `selectOverview` or other analytics selectors do NOT re-render — they didn't read from `state.filters`.

</details>

2. **Why does `fetchOverview` generate three action types?**

<details><summary>Answer</summary>

API calls have three possible states: loading (pending), success (fulfilled), and failure (rejected). `createAsyncThunk` generates an action type for each so you can update the UI accordingly — show a spinner during pending, display data on fulfilled, show an error message on rejected.

</details>

3. **Why do we use `extraReducers` instead of `reducers` for thunk actions?**

<details><summary>Answer</summary>

`reducers` defines synchronous actions that belong to this slice. `extraReducers` handles actions defined elsewhere — like the pending/fulfilled/rejected actions generated by `createAsyncThunk`. The thunk actions aren't "owned" by the slice, so they go in `extraReducers`.

</details>

4. **Why does `fetchAllAnalytics` dispatch other thunks instead of making API calls directly?**

<details><summary>Answer</summary>

Each individual thunk (fetchOverview, fetchTimeSeries, etc.) handles its own pending/fulfilled/rejected states. If `fetchAllAnalytics` made the API calls directly, you'd need to handle the loading state for all five calls in a single pending/fulfilled cycle. By dispatching individual thunks, each one manages its own state independently, and they all run in parallel via `Promise.all`.

</details>

---

| | | |
|:---|:---:|---:|
| [← 03 — Backend](../03-backend/) | [Level 4 Overview](../) | [05 — Frontend →](../05-frontend/) |
