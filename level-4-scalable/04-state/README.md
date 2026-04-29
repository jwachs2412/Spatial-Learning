`Level 4` **Step 4 of 9** — State Management

# 04 — State Management: Redux Toolkit

## New Syntax You'll Meet in This Lesson

Redux Toolkit (RTK) introduces a small vocabulary that recurs in every slice. Read this once and the 700+ lines that follow are variations on the same shapes.

- **`configureStore({ reducer: {...} })`** — RTK's store factory. Replaces classic Redux's `combineReducers` + `createStore` + middleware setup with sane defaults.
- **`createSlice({ name, initialState, reducers, extraReducers })`** — bundles state, actions, and reducers into one declaration. Generates action creators (`slice.actions`) and a reducer (`slice.reducer`) automatically.
- **`PayloadAction<T>`** — a TypeScript type for an action whose `payload` is of type `T`. Used to type reducer parameters: `(state, action: PayloadAction<string>) => { ... }`.
- **Immer "mutations"** — inside RTK reducers you can write `state.x = y` or `state.list.push(item)`. Looks like mutation, but Immer (built into RTK) tracks your changes and produces a new immutable state object behind the scenes. You get readable code without the spread-operator gymnastics classic Redux required.
- **`createAsyncThunk('slice/name', async (arg) => {...})`** — wraps an async function to auto-dispatch three actions: `slice/name/pending`, `slice/name/fulfilled`, `slice/name/rejected`. The state of every API call gets these three events for free.
- **`extraReducers: (builder) => builder.addCase(actionCreator, (state, action) => {...})`** — handles actions defined outside this slice (especially thunk actions). The builder pattern lets you respond to pending/fulfilled/rejected without manually dispatching.
- **`useSelector` and `useDispatch`** (from `react-redux`) — React hooks. `useSelector(state => state.thing)` reads from the store and re-renders when the slice it returns changes. `useDispatch()` returns the dispatch function so components can fire actions.
- **`<Provider store={store}>`** — the React component that makes the store available to descendants. Wraps your whole app once at the entry point.
- **`ReturnType<typeof store.getState>`** — a TypeScript type-level operation. `typeof store.getState` is the type of the function; `ReturnType<F>` extracts the return type. So this captures the exact shape of your state without you writing it out by hand.
- **`URLSearchParams`** — a built-in browser API for building query strings safely (`?startDate=...&category=...`). Handles URL-encoding for you.
- **`Promise.all([p1, p2, p3])`** — runs multiple promises concurrently and resolves when all of them resolve.

Every code block below has a "Reading This File Line-by-Line" walkthrough using these.

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

### Reading This File Line-by-Line

```typescript
import { configureStore } from '@reduxjs/toolkit';
import analyticsReducer from '../features/analytics/analyticsSlice';
import eventsReducer from '../features/events/eventsSlice';
import filtersReducer from '../features/filters/filtersSlice';
```

- Named import of `configureStore` from RTK.
- Three default imports of reducers from the three slices we'll build below. Each slice's `default export` is its reducer function.

```typescript
export const store = configureStore({
  reducer: {
    analytics: analyticsReducer,
    events: eventsReducer,
    filters: filtersReducer,
  },
});
```

- `configureStore({ reducer: {...} })` is the only call needed to build the store.
- The `reducer` field is an object. Each key (`analytics`, `events`, `filters`) becomes a top-level slice of state. The full state object will look like `{ analytics: {...}, events: {...}, filters: {...} }`.
- Each value is a reducer function — RTK invokes the matching one when an action is dispatched.
- RTK also wires up the Redux DevTools extension and the thunk middleware automatically. Classic Redux required several extra lines for these.

```typescript
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

These are **TypeScript type-level operations**:

- `typeof store.getState` is the type of the function `store.getState`. (At runtime `typeof` returns a string; at the type level, it captures a value's type.)
- `ReturnType<F>` is a built-in TypeScript utility that extracts the return type of a function type. So `ReturnType<typeof store.getState>` is the type returned by calling `store.getState()` — your full state shape.
- Similarly, `typeof store.dispatch` is the type of the dispatch function (which knows about all your thunks because RTK added them automatically).

Why bother? So we can type our hooks and selectors without writing the state shape by hand. The compiler stays in sync with the store automatically.

### Typed Hooks

Create `src/app/hooks.ts`:

```typescript
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

// Use these throughout the app instead of plain useDispatch/useSelector
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

### Reading This File Line-by-Line

- `import { useDispatch, useSelector } from 'react-redux';` — the two core hooks from `react-redux`. They work without TypeScript, but every call requires a manual type annotation.
- `import type { RootState, AppDispatch } from './store';` — pull in the types we just exported. The `import type` keyword tells TypeScript "these only exist for type-checking; don't include them in the compiled JavaScript."
- `useDispatch.withTypes<AppDispatch>()` — a method on the hook itself. Returns a new hook with `AppDispatch` baked in. Calling `useAppDispatch()` returns a dispatch function that already knows about every thunk in our app.
- `useSelector.withTypes<RootState>()` — same pattern. The selector you pass is typed: `useAppSelector((state) => state.filters)` — TypeScript autocompletes `state.filters`.

Pattern to remember: import these app-specific hooks (`useAppSelector`, `useAppDispatch`) in every component, never the raw `useSelector`/`useDispatch`. That way TypeScript catches mistakes everywhere.

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

### Reading This File Line-by-Line

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { AnalyticsFilters } from '../../types';
```

- `createSlice` is RTK's main factory. `PayloadAction<T>` is a type for actions with a payload field of type `T`.
- The `import type` keyword strips these from the runtime bundle (they're for the type checker only).

```typescript
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
```

- Define the shape of this slice's state. `string | null` for each field — null means "no filter set."
- Build an initial state matching the shape. Redux uses this when the store is created and whenever the reducer receives an unknown action.

```typescript
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
```

- `name: 'filters'` — namespaces the auto-generated action types. Dispatching `setCategory` actually fires `'filters/setCategory'`. Helpful in the Redux DevTools to identify which slice an action belongs to.
- `initialState` — shorthand for `initialState: initialState` (the field is the same name as the variable).
- `reducers: { ... }` — an object where each key is a reducer name and the value is a function `(state, action) => void`.
- `PayloadAction<{ startDate: string; endDate: string }>` types the action payload. RTK enforces that whoever dispatches `setDateRange(...)` must pass an object of this shape.
- Inside each reducer, `state.x = y` looks like mutation, but RTK uses **Immer** under the hood. Immer wraps `state` in a proxy that records what you "mutate," then produces a brand-new immutable state object. You write readable code; Immer handles immutability for you.
- `clearFilters` doesn't read the action payload — it ignores it. The function takes `(state)` only.

```typescript
export const { setDateRange, setCategory, setDevice, clearFilters } =
  filtersSlice.actions;
```

- `filtersSlice.actions` is an object of **action creators** — functions that return action objects. RTK generates one per reducer.
- Calling `setCategory('marketing')` returns `{ type: 'filters/setCategory', payload: 'marketing' }`.
- We destructure them and re-export so components can `import { setCategory } from '../filtersSlice'` and dispatch them.

```typescript
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
```

- **Selectors** are pure functions that take the whole state and return a piece of it.
- `selectFilters` returns the entire `filters` slice as-is.
- `selectActiveFilters` returns only the filters that have a value. Useful for API calls — we want to send `?category=marketing` but not `?category=null` if the user hasn't picked a category. Building the object conditionally keeps the URL clean.

```typescript
export const selectHasActiveFilters = (state: RootState): boolean => {
  const { startDate, endDate, category, device } = state.filters;
  return !!(startDate || endDate || category || device);
};
```

- `!!` is a **double negation** trick to coerce any value to a strict boolean. `!value` returns the opposite as a boolean; `!!value` flips it back. So if any of the four is truthy, this returns `true`; otherwise `false`. Used by the UI to decide whether to show the "Clear Filters" button.

```typescript
export default filtersSlice.reducer;
```

- The slice's reducer goes out as the default export. `store.ts` imports it and registers it under the `filters` key.

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

### Reading This File Line-by-Line

This is the most pattern-dense file in Level 4. We'll go in three chunks: **state shape**, **thunks**, and **slice with extraReducers**.

#### Chunk 1: State Shape

```typescript
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
```

- Each chart has its own field. Arrays start empty `[]`; the single overview object starts `null` (we don't have data until we fetch it).
- `loading` and `error` are global to this slice — used by the UI to show a spinner or error banner.

#### Chunk 2: Async Thunks

```typescript
export const fetchOverview = createAsyncThunk(
  'analytics/fetchOverview',
  async (filters: AnalyticsFilters) => {
    return api.getOverview(filters);
  }
);
```

- `createAsyncThunk(typePrefix, asyncFn)` — RTK helper that wraps an async function.
- `'analytics/fetchOverview'` is the action type prefix. RTK generates three real action types from it: `analytics/fetchOverview/pending`, `analytics/fetchOverview/fulfilled`, and `analytics/fetchOverview/rejected`.
- The async function takes one argument (`filters`) and returns whatever `api.getOverview(filters)` resolves to. RTK puts that value in the `fulfilled` action's `payload`.
- Dispatching: `dispatch(fetchOverview({ category: 'marketing' }))`. Components don't dispatch the pending/fulfilled/rejected actions directly — RTK does it automatically based on the promise's state.

```typescript
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
```

- A thunk's async function takes a **second argument**, an options object. Destructuring `{ dispatch }` gives us the dispatch function so this thunk can dispatch other thunks.
- `Promise.all([...])` runs all five thunks concurrently. Each one returns a promise; `Promise.all` resolves when **all** of them resolve. Five API calls happen in parallel, not one-after-another.
- We `await` the `Promise.all`, so this thunk's `fulfilled` action only fires after every child thunk has completed.

#### Chunk 3: The Slice and `extraReducers`

```typescript
const analyticsSlice = createSlice({
  name: 'analytics',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
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
    // ... more cases for the other thunks
  },
});
```

- `reducers: {}` — empty because this slice has no synchronous actions; everything is driven by thunks.
- `extraReducers: (builder) => {...}` — used to handle actions defined **outside** this slice. RTK calls our function with a `builder` object that has an `.addCase(actionCreator, reducer)` method.
- `fetchOverview.pending`, `fetchOverview.fulfilled`, `fetchOverview.rejected` — RTK attaches these three "child" action creators to every thunk. We pass them to `addCase` to wire up our handlers.
- The chained `.addCase(...)` calls return the builder, so they read like a fluent API.

Each handler:

- `pending` — flip on `loading`, clear any previous `error`. Components reading these fields show a spinner.
- `fulfilled` — store the payload, flip off `loading`. The payload is whatever the async function returned (so for `fetchOverview`, it's `OverviewStats`).
- `rejected` — flip off `loading`, store the error message. `action.error` has `.message` and `.name`; we fall back to a generic string if message is somehow missing.

Notice how only `fetchOverview` has all three cases here. The other thunks (`fetchTimeSeries`, etc.) only register a `fulfilled` case in this file — we just need to know when they finish so we can store the data. They share the global `loading` and `error` state via `fetchAllAnalytics` instead.

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

### Reading This File — What's Different from analyticsSlice

The events slice follows the same pattern as the analytics slice with three small differences worth calling out:

- **The thunk takes an object argument**:
  ```typescript
  async ({ filters, page, limit }: { filters: AnalyticsFilters; page: number; limit: number; }) => {...}
  ```
  Thunks accept exactly one argument. When you need to pass multiple values, wrap them in an object and destructure inside. Dispatch like `dispatch(fetchEvents({ filters, page: 1, limit: 20 }))`.
- **It has a synchronous reducer too**:
  ```typescript
  reducers: {
    setPage(state, action: PayloadAction<number>) {
      state.page = action.payload;
    },
  }
  ```
  When the user clicks the next-page button, we want an instant UI update before the API responds. Dispatching `setPage(2)` synchronously updates the UI; then a thunk fires to load page 2's data.
- **The fulfilled handler unpacks a richer payload**: the API's response includes `total`, `page`, `limit`, and `total_pages` alongside the events array. Each becomes its own state field.

Everything else (PayloadAction, addCase chain, selectors) follows the filters/analytics patterns.

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

### Reading This File Line-by-Line

Three building blocks: a query-string helper, a generic fetch helper, and one-line API functions per endpoint.

```typescript
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
```

- `URLSearchParams` is a **built-in browser API** for building query strings safely. No npm install needed.
- `new URLSearchParams()` creates an empty container. `.append(key, value)` adds a key/value pair.
- Each `if` block only appends a parameter if it has a value, so unset filters don't appear in the URL.
- `extra?: Record<string, string | number>` — an **optional second argument** for endpoint-specific params (like pagination's `page` and `limit`). `Record<K, V>` is a TypeScript utility type meaning "object with K keys and V values."
- `for (const [key, value] of Object.entries(extra))` walks the optional extras:
  - `Object.entries(obj)` — built-in JavaScript that returns `[[key1, val1], [key2, val2], ...]`.
  - `for (const [key, value] of array)` — destructure each pair as we iterate.
- `String(value)` — explicit conversion to string. `URLSearchParams.append` requires strings; `String(5)` is `'5'`.
- `params.toString()` — turns the params into the query-string text: `'startDate=2024-01-01&category=marketing'`.
- The ternary at the end handles the empty case: if no params, return `''` instead of `'?'`. Clean URLs.

```typescript
async function fetchJSON<T>(url: string): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    const error = await response.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(error.error || `HTTP ${response.status}`);
  }

  return response.json();
}
```

This is a generic JSON-fetch helper, like the `request<T>` from Level 2 frontend. Same shape:

- `<T>` makes it generic. The caller picks `T` and the return type follows.
- Check `response.ok`. If not, try to parse the error body as JSON, falling back to a default if parsing fails.
- `throw new Error(...)` with the server's error message (or a generic HTTP code if the server didn't provide one).
- Return the parsed JSON on success.

```typescript
export async function getOverview(filters: AnalyticsFilters): Promise<OverviewStats> {
  return fetchJSON<OverviewStats>(
    `${API_URL}/analytics/overview${buildQueryString(filters)}`
  );
}
```

One-line wrapper per endpoint. Composes:

- `${API_URL}` — base URL from env.
- `/analytics/overview` — endpoint path.
- `${buildQueryString(filters)}` — the query string we just built.

The result is a properly-typed promise that resolves to `OverviewStats`. The thunk we wrote earlier just calls this and is done.

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

### Reading This File Line-by-Line

```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import { Provider } from 'react-redux';
import { store } from './app/store';
import App from './App';
import './App.css';
```

Standard imports plus two new ones:

- `Provider` from `react-redux` — the React component that exposes the store to descendants.
- `store` from our `./app/store` — the actual store instance.

```typescript
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Provider store={store}>
      <App />
    </Provider>
  </React.StrictMode>
);
```

- `document.getElementById('root')!` — the `!` is TypeScript's **non-null assertion**. We're saying "I'm sure this returns an element, not null." Because Vite ensures `<div id="root"></div>` exists in `index.html`, the assertion is safe. Without `!`, TypeScript would complain that `getElementById` could return `null`.
- `<React.StrictMode>` — React's development helper that catches bugs by intentionally double-invoking effects. No production effect.
- `<Provider store={store}>` — passes the store down through React's component tree. Any descendant can call `useAppSelector` or `useAppDispatch` to access it.
- `<App />` — your real root component. Once Provider wraps it, every component inside has access to Redux.

The order matters: `Provider` must wrap the components that use Redux. If `Provider` is below `App`, components above it would crash with "could not find Redux store."

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
