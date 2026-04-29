`Level 4` **Step 6 of 9** — Testing

# 06 — Testing: Vitest, React Testing Library, and Supertest

## What's New: The Testing Vocabulary

This is the first lesson where you write **tests** — code whose only job is to verify other code. There's a small vocabulary that recurs in every test file. Read this once and the 800+ lines that follow are variations on the same shapes.

### The Core Functions

- **`describe(label, fn)`** — groups related tests. Purely organizational; the label appears in test output. Tests can be nested in multiple `describe` blocks.
- **`it(label, fn)`** — declares a single test case. The label should describe a behavior in plain English ("formats integers with commas"). `test(label, fn)` is an alias.
- **`expect(value)`** — wraps a value in an assertion object. Chains a matcher onto it.
- **Matchers** — methods on the expect object that check the value:
  - `.toBe(x)` — strict equality (`===`). Use for primitives.
  - `.toEqual(x)` — deep equality. Use for objects and arrays (compares contents, not references).
  - `.toBeNull()`, `.toBeUndefined()` — value-specific checks.
  - `.toBeGreaterThan(x)`, `.toBeLessThanOrEqual(x)` — numeric comparisons.
  - `.toBeCloseTo(x, n)` — numeric equality with `n` decimal places of tolerance. Use for floats.
  - `.toHaveLength(n)` — array or string has exactly `n` elements/characters.
  - `.toHaveProperty(key)` / `.toHaveProperty(key, value)` — object has the field (with optional value check).
  - `.toContain(substring)` — string includes substring, or array includes value.
  - `.toBeInTheDocument()` — DOM element exists in the rendered output (from `@testing-library/jest-dom`).

### React Testing Library

- **`render(<Component />)`** — mounts a React component into a fake DOM (jsdom). Returns helpers like `container`, `unmount`.
- **`screen`** — a global object with query methods to find elements in the rendered output.
- **`getByText('foo')`** — find an element by its visible text. Throws if not found.
- **`getByRole('button')`** — find by accessibility role (`button`, `combobox`, `textbox`, `heading`, etc.). Preferred for production-quality tests.
- **`queryByText('foo')`** — like `getByText` but returns `null` if not found. Use when asserting absence.
- **`getAllByRole(role)`** — returns an array (use when multiple matches exist).

### User Interaction

- **`userEvent.setup()`** — creates a virtual user. Returns an object with methods.
- **`await user.click(element)`** — simulate a real click (focus, mousedown, mouseup, click — all the events a real browser fires).
- **`await user.selectOptions(select, value)`** — pick an option from a dropdown.
- **`await user.type(input, 'text')`** — type into an input one character at a time.
- All `user.*` methods are async — always `await` them.

### Supertest (API Testing)

- **`request(app)`** — wraps an Express app for in-process testing. No real server, no port.
- **`.get('/path')`, `.post('/path')`, `.put('/path')`, `.delete('/path')`** — chain HTTP methods.
- **`.send(body)`** — for POST/PUT, send a JSON body.
- **`await`** the chained call to get the response object with `.status`, `.body`, `.headers`.

### Lifecycle Hooks

- **`beforeAll(fn)`** — runs once before any test in the file (for setup).
- **`afterAll(fn)`** — runs once after all tests finish (for cleanup).
- **`beforeEach(fn)` / `afterEach(fn)`** — run before/after each individual test.

Every code block below has a "Reading This File Line-by-Line" walkthrough using these.

## Spatial Orientation

Testing happens at three levels. Each level catches different categories of bugs:

```
                    ┌─────────────────────┐
                    │    API TESTS         │   Supertest
                    │    (Integration)     │   "Does the server respond correctly?"
                    │                     │
                ┌───┴─────────────────────┴───┐
                │    COMPONENT TESTS           │   React Testing Library
                │    (Behavior)                │   "Does the UI render and respond?"
                │                             │
            ┌───┴─────────────────────────────┴───┐
            │    UNIT TESTS                        │   Vitest
            │    (Logic)                           │   "Does this function return
            │                                     │    the correct value?"
            └─────────────────────────────────────┘

            More tests ←──────────────────→ Fewer tests
            Fast       ←──────────────────→ Slower
            Isolated   ←──────────────────→ Integrated
```

> [!NOTE]
> **Technical:** This is the "testing pyramid." Unit tests form the base — they're fast, isolated, and numerous. Component tests sit in the middle — they render UI and verify behavior. Integration tests (API tests) sit at the top — they test the full stack but are slower. A healthy test suite has many unit tests, some component tests, and a few integration tests.
>
> **Plain English:** Unit tests check individual Lego pieces. Component tests check that assembled sections look right. Integration tests check that the whole model stands up. You need all three, but you need more piece checks than whole-model checks.

---

## Part 1: Unit Tests (Vitest)

Unit tests verify pure logic — Redux reducers, selectors, and utility functions. They don't render UI or make network requests.

### Testing Utility Functions

> [!IMPORTANT]
> **You should be in:** `data-dash/client/`

Create `src/utils/formatters.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import {
  formatNumber,
  formatCurrency,
  formatDuration,
  formatDate,
  formatPercentage,
} from './formatters';

describe('formatNumber', () => {
  it('formats integers with commas', () => {
    expect(formatNumber(12847)).toBe('12,847');
  });

  it('handles zero', () => {
    expect(formatNumber(0)).toBe('0');
  });

  it('handles small numbers without commas', () => {
    expect(formatNumber(999)).toBe('999');
  });

  it('handles large numbers', () => {
    expect(formatNumber(1000000)).toBe('1,000,000');
  });
});

describe('formatCurrency', () => {
  it('formats as USD with dollar sign', () => {
    expect(formatCurrency(8291.5)).toBe('$8,291.50');
  });

  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });

  it('handles cents', () => {
    expect(formatCurrency(9.99)).toBe('$9.99');
  });
});

describe('formatDuration', () => {
  it('formats seconds as minutes:seconds', () => {
    expect(formatDuration(272)).toBe('4:32');
  });

  it('pads seconds with leading zero', () => {
    expect(formatDuration(65)).toBe('1:05');
  });

  it('handles zero', () => {
    expect(formatDuration(0)).toBe('0:00');
  });

  it('handles exact minutes', () => {
    expect(formatDuration(120)).toBe('2:00');
  });
});

describe('formatDate', () => {
  it('formats YYYY-MM-DD as Mon DD', () => {
    expect(formatDate('2025-03-15')).toBe('Mar 15');
  });

  it('formats January correctly', () => {
    expect(formatDate('2025-01-01')).toBe('Jan 1');
  });
});

describe('formatPercentage', () => {
  it('formats decimal as percentage', () => {
    expect(formatPercentage(0.623)).toBe('62.3%');
  });

  it('handles 100%', () => {
    expect(formatPercentage(1)).toBe('100.0%');
  });

  it('handles 0%', () => {
    expect(formatPercentage(0)).toBe('0.0%');
  });
});
```

### Reading This File Line-by-Line

```typescript
import { describe, it, expect } from 'vitest';
import {
  formatNumber,
  formatCurrency,
  formatDuration,
  formatDate,
  formatPercentage,
} from './formatters';
```

- Named imports of Vitest's three core test functions. (We could rely on `globals: true` from the Vitest config and skip these imports — but importing makes the dependencies explicit.)
- Import each function we plan to test from the file under test. Tests live next to their source: `formatters.ts` and `formatters.test.ts` are siblings.

```typescript
describe('formatNumber', () => {
  it('formats integers with commas', () => {
    expect(formatNumber(12847)).toBe('12,847');
  });
```

The classic three-step test:

- `describe('formatNumber', () => { ... })` opens a group of tests for the `formatNumber` function. The string is the label that prints when tests run.
- Inside, `it('formats integers with commas', () => { ... })` declares one test case. The string is a sentence describing the behavior — read it back as "it formats integers with commas." (That's why `it` is the function name.)
- The arrow function body contains the assertion: call the function with an input, then assert what it should return.
- `expect(value).toBe(expected)` — `expect` wraps the actual value; `.toBe(...)` compares it with strict equality. If they don't match, the test fails with a clear diff.

Each `it(...)` block is independent. If one fails, the others still run.

```typescript
  it('handles zero', () => {
    expect(formatNumber(0)).toBe('0');
  });

  it('handles small numbers without commas', () => {
    expect(formatNumber(999)).toBe('999');
  });
```

Add **edge-case tests** — zero and small numbers that don't need commas. Good test suites cover the happy path AND the boundaries. Asking "what if input is 0? What if it's negative? What if it's huge?" reveals bugs.

```typescript
describe('formatPercentage', () => {
  it('formats decimal as percentage', () => {
    expect(formatPercentage(0.623)).toBe('62.3%');
  });

  it('handles 100%', () => {
    expect(formatPercentage(1)).toBe('100.0%');
  });
```

Same pattern for every helper. Notice `'100.0%'` — the matcher is exact. If our function returned `'100%'`, the test would fail. Tests pin down the precise output format.

### Anatomy of a Test

```typescript
describe('formatNumber', () => {        // Group related tests
  it('formats integers with commas', () => {  // One specific behavior
    expect(formatNumber(12847)).toBe('12,847');  // Assert expected output
  });
});
```

| Part | Purpose |
|------|---------|
| `describe` | Groups related tests under a label |
| `it` | Defines a single test case (use `test` as an alias) |
| `expect` | Creates an assertion — "I expect this value..." |
| `.toBe` | Matcher — "...to be exactly this" |

Run the tests:

```bash
npx vitest run src/utils/formatters.test.ts
```

Expected: All tests pass with green checkmarks.

### Testing Redux Slices

Create `src/features/filters/filtersSlice.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import filtersReducer, {
  setDateRange,
  setCategory,
  setDevice,
  clearFilters,
  selectActiveFilters,
  selectHasActiveFilters,
} from './filtersSlice';
import type { RootState } from '../../app/store';

// Helper: create a mock RootState with the given filters
function stateWith(filters: ReturnType<typeof filtersReducer>): RootState {
  return { filters } as RootState;
}

describe('filtersSlice reducer', () => {
  const initialState = {
    startDate: null,
    endDate: null,
    category: null,
    device: null,
  };

  it('returns initial state', () => {
    expect(filtersReducer(undefined, { type: 'unknown' })).toEqual(initialState);
  });

  it('sets date range', () => {
    const state = filtersReducer(
      initialState,
      setDateRange({ startDate: '2025-01-01', endDate: '2025-01-31' })
    );
    expect(state.startDate).toBe('2025-01-01');
    expect(state.endDate).toBe('2025-01-31');
  });

  it('sets category', () => {
    const state = filtersReducer(initialState, setCategory('marketing'));
    expect(state.category).toBe('marketing');
  });

  it('clears category with null', () => {
    const state = filtersReducer(
      { ...initialState, category: 'marketing' },
      setCategory(null)
    );
    expect(state.category).toBeNull();
  });

  it('sets device', () => {
    const state = filtersReducer(initialState, setDevice('mobile'));
    expect(state.device).toBe('mobile');
  });

  it('clears all filters', () => {
    const dirtyState = {
      startDate: '2025-01-01',
      endDate: '2025-01-31',
      category: 'marketing',
      device: 'mobile',
    };
    const state = filtersReducer(dirtyState, clearFilters());
    expect(state).toEqual(initialState);
  });
});

describe('filtersSlice selectors', () => {
  it('selectActiveFilters returns only non-null values', () => {
    const state = stateWith({
      startDate: '2025-01-01',
      endDate: null,
      category: 'marketing',
      device: null,
    });
    const active = selectActiveFilters(state);
    expect(active).toEqual({
      startDate: '2025-01-01',
      category: 'marketing',
    });
    expect(active.endDate).toBeUndefined();
    expect(active.device).toBeUndefined();
  });

  it('selectHasActiveFilters returns true when filters are set', () => {
    const state = stateWith({
      startDate: null,
      endDate: null,
      category: 'product',
      device: null,
    });
    expect(selectHasActiveFilters(state)).toBe(true);
  });

  it('selectHasActiveFilters returns false when no filters are set', () => {
    const state = stateWith({
      startDate: null,
      endDate: null,
      category: null,
      device: null,
    });
    expect(selectHasActiveFilters(state)).toBe(false);
  });
});
```

### Reading This File Line-by-Line

```typescript
function stateWith(filters: ReturnType<typeof filtersReducer>): RootState {
  return { filters } as RootState;
}
```

A **test helper function**. We need a `RootState` (the full app state shape) for selectors to consume, but we only care about the `filters` slice. So we build a minimal mock by wrapping just the filters slice and casting. The `as RootState` is a type assertion telling TypeScript "trust me, this is enough." Selectors only ever touch `state.filters`, so the cast is safe in this context.

`ReturnType<typeof filtersReducer>` — the return type of calling the reducer. That's the `FiltersState` shape, automatically derived from the reducer.

```typescript
it('returns initial state', () => {
  expect(filtersReducer(undefined, { type: 'unknown' })).toEqual(initialState);
});
```

A reducer's first responsibility is "given undefined state and an unknown action, return the initial state." We pass `undefined` for state and a plain object with a fake type. Then assert the result matches the initial state.

`.toEqual(...)` — **deep equality**. We're comparing two objects, so we need to compare contents, not references. `.toBe(initialState)` would fail because they're different object instances even if their fields match.

```typescript
it('sets date range', () => {
  const state = filtersReducer(
    initialState,
    setDateRange({ startDate: '2025-01-01', endDate: '2025-01-31' })
  );
  expect(state.startDate).toBe('2025-01-01');
  expect(state.endDate).toBe('2025-01-31');
});
```

The classic reducer test:

1. Start from a known state (`initialState`).
2. Apply an action by calling the action creator (`setDateRange({...})` returns a real action object).
3. Pass both into the reducer to get the new state.
4. Assert the new state has the expected shape.

This is **black-box testing** — we don't peek inside the reducer. We give it inputs, check outputs.

```typescript
it('clears category with null', () => {
  const state = filtersReducer(
    { ...initialState, category: 'marketing' },
    setCategory(null)
  );
  expect(state.category).toBeNull();
});
```

To test "clearing" behavior, start from a state where the field has a value. `{ ...initialState, category: 'marketing' }` spreads initialState into a new object, then overrides `category`. Now we can verify that `setCategory(null)` resets it.

`.toBeNull()` — tighter than `.toBe(null)`. Reads more naturally.

```typescript
describe('filtersSlice selectors', () => {
  it('selectActiveFilters returns only non-null values', () => {
    const state = stateWith({
      startDate: '2025-01-01',
      endDate: null,
      category: 'marketing',
      device: null,
    });
    const active = selectActiveFilters(state);
    expect(active).toEqual({
      startDate: '2025-01-01',
      category: 'marketing',
    });
    expect(active.endDate).toBeUndefined();
    expect(active.device).toBeUndefined();
  });
```

Selector tests give a fake state, call the selector, and assert the derived value:

- `stateWith(...)` builds the mock state.
- `selectActiveFilters(state)` returns the derived object.
- `.toEqual(...)` checks the full shape — both fields present AND no others.
- The two `.toBeUndefined()` checks are redundant if `.toEqual` passed (it's an exact match), but they make the intent extra clear: "these fields should NOT be in the output."

### Why Test Reducers?

Reducers are pure functions — given the same state and action, they always return the same result. This makes them ideal for unit testing. Testing reducers verifies:

1. Initial state is correct
2. Each action modifies state as expected
3. Selectors derive the right values

Run:

```bash
npx vitest run src/features/filters
```

---

> [!TIP]
> **Session Break** — You've written unit tests for utility functions and Redux slices. Save your work and take a break.
> When you return, you'll write component tests with React Testing Library and API tests with Supertest.

---

## Part 2: Component Tests (React Testing Library)

Component tests render React components and verify they display correct content and respond to user interactions.

### Testing OverviewCards

Create `src/features/analytics/OverviewCards.test.tsx`:

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import analyticsReducer from './analyticsSlice';
import OverviewCards from './OverviewCards';

// Create a test store with pre-loaded state
function renderWithStore(preloadedAnalytics = {}) {
  const store = configureStore({
    reducer: { analytics: analyticsReducer },
    preloadedState: {
      analytics: {
        overview: null,
        timeSeries: [],
        categories: [],
        devices: [],
        topPages: [],
        loading: false,
        error: null,
        ...preloadedAnalytics,
      },
    },
  });

  return render(
    <Provider store={store}>
      <OverviewCards />
    </Provider>
  );
}

describe('OverviewCards', () => {
  it('renders nothing when overview is null and not loading', () => {
    const { container } = renderWithStore({ overview: null, loading: false });
    expect(container.firstChild).toBeNull();
  });

  it('renders skeleton cards when loading with no data', () => {
    renderWithStore({ overview: null, loading: true });
    const skeletons = document.querySelectorAll('.stat-card--skeleton');
    expect(skeletons).toHaveLength(4);
  });

  it('renders overview data when available', () => {
    renderWithStore({
      overview: {
        total_page_views: 12847,
        total_signups: 342,
        total_revenue: 8291.5,
        avg_session_duration: 272,
        period_start: '2025-01-01',
        period_end: '2025-03-01',
      },
    });

    expect(screen.getByText('12,847')).toBeInTheDocument();
    expect(screen.getByText('342')).toBeInTheDocument();
    expect(screen.getByText('$8,291.50')).toBeInTheDocument();
    expect(screen.getByText('4:32')).toBeInTheDocument();
  });

  it('renders card titles', () => {
    renderWithStore({
      overview: {
        total_page_views: 0,
        total_signups: 0,
        total_revenue: 0,
        avg_session_duration: 0,
        period_start: '',
        period_end: '',
      },
    });

    expect(screen.getByText('Page Views')).toBeInTheDocument();
    expect(screen.getByText('Signups')).toBeInTheDocument();
    expect(screen.getByText('Revenue')).toBeInTheDocument();
    expect(screen.getByText('Avg. Session')).toBeInTheDocument();
  });
});
```

### Reading This File Line-by-Line

```typescript
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
```

- `render` mounts the component into jsdom; `screen` is the global object for querying the rendered DOM.
- `Provider` and `configureStore` — same pieces from `main.tsx`, used here to build a **test store** for each test.

```typescript
function renderWithStore(preloadedAnalytics = {}) {
  const store = configureStore({
    reducer: { analytics: analyticsReducer },
    preloadedState: {
      analytics: {
        overview: null,
        timeSeries: [],
        categories: [],
        devices: [],
        topPages: [],
        loading: false,
        error: null,
        ...preloadedAnalytics,
      },
    },
  });

  return render(
    <Provider store={store}>
      <OverviewCards />
    </Provider>
  );
}
```

A **test helper** that builds a fresh store with a custom initial state and renders the component inside it.

- `preloadedAnalytics = {}` — default parameter. If the caller passes nothing, we use an empty object.
- `preloadedState: { analytics: { ... } }` — the second key feature of `configureStore`. Lets you override the initial state for tests.
- `{ ...defaults, ...preloadedAnalytics }` — spread the defaults first, then the test-specific overrides. Last spread wins, so callers can change just the fields they care about.
- `<Provider store={store}>` — the same wrapper as production. The component-under-test sees this isolated store, not the real one.
- `render(...)` returns helpers; we forward them to the test.

Why a fresh store per test? **Test isolation.** If two tests shared a store, the second one would see leftover state from the first. With a new store every time, each test starts from a known clean slate.

```typescript
it('renders nothing when overview is null and not loading', () => {
  const { container } = renderWithStore({ overview: null, loading: false });
  expect(container.firstChild).toBeNull();
});
```

- Destructure `container` from the render result. It's the wrapping div RTL inserts the component into.
- `container.firstChild` — the first DOM node the component rendered. If the component returned `null`, it rendered nothing, so `firstChild` is `null`.
- This is the only test that drops to direct DOM inspection; usually we use `screen` queries.

```typescript
it('renders skeleton cards when loading with no data', () => {
  renderWithStore({ overview: null, loading: true });
  const skeletons = document.querySelectorAll('.stat-card--skeleton');
  expect(skeletons).toHaveLength(4);
});
```

- `document.querySelectorAll(...)` is a built-in DOM API: returns a list of all elements matching a CSS selector.
- `.stat-card--skeleton` matches the skeleton cards we render during loading.
- `.toHaveLength(4)` — assert exactly four skeletons, matching our `[1, 2, 3, 4].map(...)` in the component.

```typescript
it('renders overview data when available', () => {
  renderWithStore({
    overview: {
      total_page_views: 12847,
      total_signups: 342,
      total_revenue: 8291.5,
      avg_session_duration: 272,
      period_start: '2025-01-01',
      period_end: '2025-03-01',
    },
  });

  expect(screen.getByText('12,847')).toBeInTheDocument();
  expect(screen.getByText('342')).toBeInTheDocument();
  expect(screen.getByText('$8,291.50')).toBeInTheDocument();
  expect(screen.getByText('4:32')).toBeInTheDocument();
});
```

- We give the test store an `overview` object with realistic values.
- `screen.getByText('12,847')` queries the rendered output for an element containing that exact text. **Notice the formatting** — we're testing not just that data appears, but that it's formatted correctly. `12847` raw would fail; `formatNumber` produces `'12,847'`.
- `.toBeInTheDocument()` — from `@testing-library/jest-dom`. Asserts the element exists in the rendered tree. Strictly speaking redundant after `getByText` (which throws if not found), but reads beautifully.

This single test exercises the entire chain: **Redux state → selector → component renders → formatter applied → DOM has formatted text.**

### RTL Testing Philosophy

React Testing Library follows one rule: **test what users see, not implementation details.**

```
GOOD: screen.getByText('12,847')
  → Tests what the user sees on screen

BAD: expect(component.state.overview.total_page_views).toBe(12847)
  → Tests internal implementation — if you rename the state property, the test breaks
```

| RTL Query | When to use |
|-----------|-------------|
| `getByText('...')` | Find element by visible text |
| `getByRole('button')` | Find by accessibility role |
| `getByLabelText('...')` | Find form inputs by label |
| `getByPlaceholderText('...')` | Find inputs by placeholder |
| `queryByText('...')` | Like getByText but returns null instead of throwing |

### Testing FilterBar with User Interaction

Create `src/features/filters/FilterBar.test.tsx`:

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import filtersReducer from './filtersSlice';
import FilterBar from './FilterBar';

function renderFilterBar() {
  const store = configureStore({
    reducer: { filters: filtersReducer },
  });

  render(
    <Provider store={store}>
      <FilterBar />
    </Provider>
  );

  return store;
}

describe('FilterBar', () => {
  it('renders date range buttons', () => {
    renderFilterBar();

    expect(screen.getByText('30d')).toBeInTheDocument();
    expect(screen.getByText('60d')).toBeInTheDocument();
    expect(screen.getByText('90d')).toBeInTheDocument();
  });

  it('renders category and device dropdowns', () => {
    renderFilterBar();

    expect(screen.getByText('All Categories')).toBeInTheDocument();
    expect(screen.getByText('All Devices')).toBeInTheDocument();
  });

  it('does not show clear button when no filters are active', () => {
    renderFilterBar();

    expect(screen.queryByText('Clear Filters')).not.toBeInTheDocument();
  });

  it('updates category in store when selected', async () => {
    const user = userEvent.setup();
    const store = renderFilterBar();

    const categorySelect = screen.getAllByRole('combobox')[0];
    await user.selectOptions(categorySelect, 'marketing');

    expect(store.getState().filters.category).toBe('marketing');
  });

  it('shows clear button when filters are active', async () => {
    const user = userEvent.setup();
    renderFilterBar();

    const categorySelect = screen.getAllByRole('combobox')[0];
    await user.selectOptions(categorySelect, 'marketing');

    expect(screen.getByText('Clear Filters')).toBeInTheDocument();
  });

  it('clears all filters when Clear Filters is clicked', async () => {
    const user = userEvent.setup();
    const store = renderFilterBar();

    // Set a filter
    const categorySelect = screen.getAllByRole('combobox')[0];
    await user.selectOptions(categorySelect, 'product');

    // Clear it
    await user.click(screen.getByText('Clear Filters'));

    expect(store.getState().filters.category).toBeNull();
    expect(screen.queryByText('Clear Filters')).not.toBeInTheDocument();
  });
});
```

### Reading This File Line-by-Line

```typescript
import userEvent from '@testing-library/user-event';
```

The user-interaction library. `userEvent` is the realistic input simulator — clicks, typing, selecting all behave like a real user would.

```typescript
function renderFilterBar() {
  const store = configureStore({
    reducer: { filters: filtersReducer },
  });

  render(
    <Provider store={store}>
      <FilterBar />
    </Provider>
  );

  return store;
}
```

Same test-store pattern as before, with a key difference: this helper returns the store. Some tests need to inspect the store's state after dispatching, so we hand it back.

```typescript
it('updates category in store when selected', async () => {
  const user = userEvent.setup();
  const store = renderFilterBar();

  const categorySelect = screen.getAllByRole('combobox')[0];
  await user.selectOptions(categorySelect, 'marketing');

  expect(store.getState().filters.category).toBe('marketing');
});
```

- `async () => { ... }` — interaction tests are always async.
- `userEvent.setup()` — creates a virtual user. Returns an object with methods.
- `screen.getAllByRole('combobox')` — `<select>` elements have ARIA role `'combobox'`. We use `getAllByRole` (not `getByRole`) because there are two selects: category and device. `[0]` gets the first one.
- `await user.selectOptions(categorySelect, 'marketing')` — pick the option whose value is `'marketing'`. The `await` is important because `userEvent` simulates browser timing.
- `store.getState().filters.category` — direct read from the store. We're verifying that the dispatched action made it through to the reducer.

This is testing **end-to-end behavior** without the network: user clicks → component dispatches → reducer runs → state updates. Single test, full chain.

```typescript
it('shows clear button when filters are active', async () => {
  const user = userEvent.setup();
  renderFilterBar();

  const categorySelect = screen.getAllByRole('combobox')[0];
  await user.selectOptions(categorySelect, 'marketing');

  expect(screen.getByText('Clear Filters')).toBeInTheDocument();
});
```

After dispatching, the component re-renders with the new state. The "Clear Filters" button (which was hidden) now appears. `screen.getByText('Clear Filters')` finds it; `.toBeInTheDocument()` confirms.

```typescript
it('clears all filters when Clear Filters is clicked', async () => {
  const user = userEvent.setup();
  const store = renderFilterBar();

  const categorySelect = screen.getAllByRole('combobox')[0];
  await user.selectOptions(categorySelect, 'product');

  await user.click(screen.getByText('Clear Filters'));

  expect(store.getState().filters.category).toBeNull();
  expect(screen.queryByText('Clear Filters')).not.toBeInTheDocument();
});
```

A **multi-step interaction test**:

1. Set a filter (so the Clear button appears).
2. Click Clear.
3. Verify the filter is null.
4. Verify the Clear button has disappeared.

`screen.queryByText('Clear Filters')` (not `getByText`) — we're asserting **absence**. `getByText` throws when nothing matches; `queryByText` returns `null`. Combined with `.not.toBeInTheDocument()`, we get a clean "this should NOT be there" assertion.

### userEvent vs fireEvent

```
userEvent.click(button)    ← Simulates a REAL user click (focus, mousedown, mouseup, click)
fireEvent.click(button)    ← Dispatches a single click event (may miss side effects)
```

Always prefer `userEvent` — it simulates real browser behavior more accurately.

Run all frontend tests:

```bash
npx vitest run
```

---

> [!TIP]
> **Session Break** — You've written component tests for OverviewCards and FilterBar with user interactions. Save your work and take a break.
> When you return, you'll write API integration tests with Supertest.

---

## Part 3: API Tests (Supertest)

API tests send HTTP requests to your Express server and verify the responses. Supertest creates a test server without needing a port — it works in-process.

> [!IMPORTANT]
> **You should be in:** `data-dash/server/`

Create `tests/analytics.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import app from '../src/index';
import pool from '../src/db/index';

// Ensure test data exists
beforeAll(async () => {
  const result = await pool.query('SELECT COUNT(*) FROM analytics_events');
  if (Number(result.rows[0].count) === 0) {
    throw new Error(
      'No test data found. Run `npm run seed` before running tests.'
    );
  }
});

afterAll(async () => {
  await pool.end();
});

describe('GET /api/health', () => {
  it('returns status ok with database connected', async () => {
    const res = await request(app).get('/api/health');

    expect(res.status).toBe(200);
    expect(res.body.status).toBe('ok');
    expect(res.body.database).toBe('connected');
  });
});

describe('GET /api/analytics/overview', () => {
  it('returns overview stats', async () => {
    const res = await request(app).get('/api/analytics/overview');

    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('total_page_views');
    expect(res.body).toHaveProperty('total_signups');
    expect(res.body).toHaveProperty('total_revenue');
    expect(res.body).toHaveProperty('avg_session_duration');
    expect(typeof res.body.total_page_views).toBe('number');
    expect(typeof res.body.total_revenue).toBe('number');
  });

  it('returns filtered stats with category', async () => {
    const res = await request(app)
      .get('/api/analytics/overview?category=marketing');

    expect(res.status).toBe(200);
    expect(res.body.total_page_views).toBeGreaterThanOrEqual(0);
  });

  it('returns filtered stats with date range', async () => {
    const res = await request(app)
      .get('/api/analytics/overview?startDate=2025-02-01&endDate=2025-02-28');

    expect(res.status).toBe(200);
    expect(res.body.total_page_views).toBeGreaterThanOrEqual(0);
  });

  it('rejects invalid date format', async () => {
    const res = await request(app)
      .get('/api/analytics/overview?startDate=not-a-date');

    expect(res.status).toBe(400);
    expect(res.body.error).toContain('Invalid startDate');
  });

  it('rejects invalid category', async () => {
    const res = await request(app)
      .get('/api/analytics/overview?category=invalid');

    expect(res.status).toBe(400);
    expect(res.body.error).toContain('Invalid category');
  });
});

describe('GET /api/analytics/time-series', () => {
  it('returns array of time series points', async () => {
    const res = await request(app).get('/api/analytics/time-series');

    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
    expect(res.body.length).toBeGreaterThan(0);

    const point = res.body[0];
    expect(point).toHaveProperty('date');
    expect(point).toHaveProperty('page_views');
    expect(point).toHaveProperty('signups');
    expect(point).toHaveProperty('revenue');
  });
});

describe('GET /api/analytics/categories', () => {
  it('returns category breakdown', async () => {
    const res = await request(app).get('/api/analytics/categories');

    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);

    const cat = res.body[0];
    expect(cat).toHaveProperty('category');
    expect(cat).toHaveProperty('count');
    expect(cat).toHaveProperty('revenue');
  });
});

describe('GET /api/analytics/devices', () => {
  it('returns device breakdown with percentages', async () => {
    const res = await request(app).get('/api/analytics/devices');

    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);

    const device = res.body[0];
    expect(device).toHaveProperty('device');
    expect(device).toHaveProperty('count');
    expect(device).toHaveProperty('percentage');
    expect(device.percentage).toBeGreaterThan(0);
    expect(device.percentage).toBeLessThanOrEqual(1);
  });

  it('percentages sum to approximately 1', async () => {
    const res = await request(app).get('/api/analytics/devices');
    const totalPercentage = res.body.reduce(
      (sum: number, d: { percentage: number }) => sum + d.percentage,
      0
    );
    expect(totalPercentage).toBeCloseTo(1, 1);
  });
});

describe('GET /api/analytics/events', () => {
  it('returns paginated events', async () => {
    const res = await request(app)
      .get('/api/analytics/events?page=1&limit=10');

    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('events');
    expect(res.body).toHaveProperty('total');
    expect(res.body).toHaveProperty('page', 1);
    expect(res.body).toHaveProperty('limit', 10);
    expect(res.body).toHaveProperty('total_pages');
    expect(res.body.events).toHaveLength(10);
  });

  it('respects page parameter', async () => {
    const page1 = await request(app)
      .get('/api/analytics/events?page=1&limit=5');
    const page2 = await request(app)
      .get('/api/analytics/events?page=2&limit=5');

    expect(page1.body.events[0].id).not.toBe(page2.body.events[0].id);
  });

  it('rejects invalid page number', async () => {
    const res = await request(app)
      .get('/api/analytics/events?page=-1');

    expect(res.status).toBe(400);
    expect(res.body.error).toContain('page must be');
  });

  it('rejects limit over 100', async () => {
    const res = await request(app)
      .get('/api/analytics/events?limit=200');

    expect(res.status).toBe(400);
    expect(res.body.error).toContain('limit must be');
  });
});

describe('GET /api/analytics/top-pages', () => {
  it('returns top pages sorted by views', async () => {
    const res = await request(app).get('/api/analytics/top-pages');

    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);

    if (res.body.length > 1) {
      expect(res.body[0].views).toBeGreaterThanOrEqual(res.body[1].views);
    }

    const page = res.body[0];
    expect(page).toHaveProperty('page');
    expect(page).toHaveProperty('views');
    expect(page).toHaveProperty('unique_sessions');
  });
});
```

### Reading This File Line-by-Line

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import app from '../src/index';
import pool from '../src/db/index';
```

- Two more Vitest hooks: `beforeAll` and `afterAll` for setup/teardown.
- `request` (default import) is Supertest's wrapper.
- We import `app` (the Express instance) and `pool` (the database connection) directly. Tests run in-process, so we work with these objects directly.

```typescript
beforeAll(async () => {
  const result = await pool.query('SELECT COUNT(*) FROM analytics_events');
  if (Number(result.rows[0].count) === 0) {
    throw new Error(
      'No test data found. Run `npm run seed` before running tests.'
    );
  }
});
```

- `beforeAll` runs once before any test in this file.
- We query the database to make sure it has seeded data. If not, we throw a helpful error.
- This is a guard rail: tests that hit an empty database would have weird failures ("expected 0 to be greater than 0"). A clear error message saves debugging time.

```typescript
afterAll(async () => {
  await pool.end();
});
```

- `afterAll` runs once after all tests complete.
- `pool.end()` closes all connections in the pool. Without this, Vitest hangs because Node's event loop has open handles.

```typescript
describe('GET /api/health', () => {
  it('returns status ok with database connected', async () => {
    const res = await request(app).get('/api/health');

    expect(res.status).toBe(200);
    expect(res.body.status).toBe('ok');
    expect(res.body.database).toBe('connected');
  });
});
```

The classic Supertest pattern:

- `request(app)` creates a test request wrapping our Express app. **No port**, no real HTTP — it calls Express directly in-memory.
- `.get('/api/health')` builds and sends a GET request. Returns a promise.
- `await` the promise to get the response object.
- `res.status` is the HTTP status code; `res.body` is the parsed JSON body.

Each assertion checks one fact about the response. If any fails, the test fails with a clear message.

```typescript
it('returns filtered stats with category', async () => {
  const res = await request(app)
    .get('/api/analytics/overview?category=marketing');

  expect(res.status).toBe(200);
  expect(res.body.total_page_views).toBeGreaterThanOrEqual(0);
});
```

- Query strings are part of the URL — pass them in the path.
- `.toBeGreaterThanOrEqual(0)` — we don't know the exact count (seed data is random), but we know it should be a non-negative number.

```typescript
it('rejects invalid date format', async () => {
  const res = await request(app)
    .get('/api/analytics/overview?startDate=not-a-date');

  expect(res.status).toBe(400);
  expect(res.body.error).toContain('Invalid startDate');
});
```

**Negative tests** — verify the server correctly rejects bad input. These catch regressions in your validation logic.

`.toContain('Invalid startDate')` — partial-string matcher. Useful when the error message includes details (like a list of valid options) but you only want to assert the key phrase.

```typescript
it('returns paginated events', async () => {
  const res = await request(app)
    .get('/api/analytics/events?page=1&limit=10');

  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('events');
  expect(res.body).toHaveProperty('total');
  expect(res.body).toHaveProperty('page', 1);
  expect(res.body).toHaveProperty('limit', 10);
  expect(res.body.events).toHaveLength(10);
});
```

- `.toHaveProperty('events')` — check that the field exists.
- `.toHaveProperty('page', 1)` — two-arg form: field exists AND has this value.
- `.toHaveLength(10)` — array length check.

```typescript
it('respects page parameter', async () => {
  const page1 = await request(app)
    .get('/api/analytics/events?page=1&limit=5');
  const page2 = await request(app)
    .get('/api/analytics/events?page=2&limit=5');

  expect(page1.body.events[0].id).not.toBe(page2.body.events[0].id);
});
```

A test that requires **two requests**. We fetch page 1 and page 2 separately, then assert their first events are different. `.not.toBe(...)` is the negation — passes when the two values are NOT equal.

This is testing pagination behavior: page 2 must show different rows than page 1.

```typescript
it('percentages sum to approximately 1', async () => {
  const res = await request(app).get('/api/analytics/devices');
  const totalPercentage = res.body.reduce(
    (sum: number, d: { percentage: number }) => sum + d.percentage,
    0
  );
  expect(totalPercentage).toBeCloseTo(1, 1);
});
```

- `.reduce(callback, initialValue)` — array method that boils an array into a single value. Here it sums up all the percentages.
- `(sum: number, d: { percentage: number }) => sum + d.percentage` — for each device, add its percentage to the running sum.
- `.toBeCloseTo(1, 1)` — assert the sum is close to `1` with `1` decimal place of tolerance. **Float arithmetic isn't exact.** `0.62 + 0.31 + 0.07` might give you `0.9999999...` or `1.00000001...`. `.toBeCloseTo` accepts both as "close enough."

### How Supertest Works

```typescript
const res = await request(app)     // Create a test request against the Express app
  .get('/api/analytics/overview')  // Send a GET request
  // No real server is started — Supertest handles the HTTP in-process

expect(res.status).toBe(200);      // Assert status code
expect(res.body.total_page_views)  // Assert response body
  .toBeGreaterThan(0);
```

Supertest avoids port conflicts and network latency. It calls Express directly.

### Server Export Fix

For Supertest to work, the server must export the Express app without starting it. Update `server/src/index.ts` — change the listen call:

```typescript
// Only start the server if this file is run directly (not imported by tests)
if (process.env.NODE_ENV !== 'test') {
  app.listen(PORT, () => {
    logger.info(`Server running on http://localhost:${PORT}`);
  });
}

export default app;
```

### Reading This Snippet

The problem this solves: when our test file does `import app from '../src/index'`, Node executes the entire file — including `app.listen()`. That binds a port, and the test process never exits. Multiple test runs can't share the same port.

The fix:

- `if (process.env.NODE_ENV !== 'test') { ... }` — only call `app.listen()` if we're NOT in a test environment.
- Vitest sets `NODE_ENV='test'` automatically when it runs. So tests import `app` without spinning up a real server.
- In production / development, `NODE_ENV` is `'production'` or undefined, the condition passes, and `listen()` runs.
- `export default app;` — always export the app object so Supertest can wrap it.

Add a Vitest config for the server. Create `server/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    testTimeout: 10000,
  },
});
```

### Reading This Config

Three options, each with a specific reason:

- `globals: true` — same as the client: makes `describe`, `it`, `expect` available without imports.
- `environment: 'node'` — the backend doesn't render UI, so we use Node's environment instead of jsdom. Faster (no DOM simulation) and matches the production runtime.
- `testTimeout: 10000` — by default, Vitest fails a test that takes longer than 5 seconds. Database queries can occasionally take that long under load. We bump to 10 seconds to avoid false failures in slow CI environments.

Run the API tests:

```bash
cd server && npm test
```

> [!WARNING]
> **API tests require a running database with seed data.** Run `npm run seed` before running tests. In a real CI pipeline, you'd use a test database that's seeded automatically.

---

## Running All Tests

From the project root:

```bash
# Frontend tests
cd client && npm test

# Backend tests
cd server && npm test
```

Or run them together:

```bash
cd client && npm test && cd ../server && npm test
```

Expected output:

```
 ✓ src/utils/formatters.test.ts (12 tests)
 ✓ src/features/filters/filtersSlice.test.ts (9 tests)
 ✓ src/features/analytics/OverviewCards.test.tsx (4 tests)
 ✓ src/features/filters/FilterBar.test.tsx (6 tests)

 Test Files  4 passed
 Tests       31 passed
```

```
 ✓ tests/analytics.test.ts (15 tests)

 Test Files  1 passed
 Tests       15 passed
```

---

## Step 10: Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: add test suite — unit tests, component tests, and API tests"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why do we test the reducer directly instead of through the component?**

<details><summary>Answer</summary>

Reducers are pure functions — given the same input, they always return the same output. Testing them directly is fast (no DOM rendering needed), reliable (no async timing issues), and thorough (you can test edge cases easily). Component tests verify the UI; reducer tests verify the logic. Together they cover both.

</details>

2. **Why does the component test create its own Redux store instead of using the real one?**

<details><summary>Answer</summary>

Test isolation. Each test should start with a known state and not be affected by other tests. Creating a fresh store with `preloadedState` gives you complete control over what data the component receives. Using the real store would mean tests depend on each other's side effects.

</details>

3. **Why does the Supertest test check `toHaveProperty` instead of exact values?**

<details><summary>Answer</summary>

The seed data is randomly generated — exact values change every time you reseed. Tests verify the *shape* of the response (correct properties, correct types, correct structure) rather than exact values. This makes tests reliable regardless of the seed data.

</details>

4. **What's the difference between `getByText` and `queryByText`?**

<details><summary>Answer</summary>

`getByText` throws an error if the element isn't found — use it when you expect the element to exist. `queryByText` returns `null` if the element isn't found — use it when testing that something does NOT appear (e.g., `expect(queryByText('Clear Filters')).not.toBeInTheDocument()`).

</details>

---

| | | |
|:---|:---:|---:|
| [← 05 — Frontend](../05-frontend/) | [Level 4 Overview](../) | [07 — Performance →](../07-performance/) |
