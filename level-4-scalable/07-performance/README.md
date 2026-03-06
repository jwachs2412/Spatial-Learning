`Level 4` **Step 7 of 9** — Performance

# 07 — Performance: Optimization and Production Hardening

## Spatial Orientation

Performance optimization answers one question: **which renders are unnecessary, and how do we eliminate them?**

```
BEFORE OPTIMIZATION:

  Filter changes
       │
       ▼
  ALL components re-render    ← OverviewCards, all three charts,
                                 EventsTable, Sidebar — everything

AFTER OPTIMIZATION:

  Filter changes
       │
       ▼
  ONLY affected components    ← OverviewCards and charts re-render
  re-render                      Sidebar does NOT (it doesn't read filters)
```

Redux selectors already prevent most unnecessary re-renders. This lesson adds additional techniques for expensive components (charts) and introduces production safeguards (error boundaries, debouncing).

---

## React.memo — Skip Re-renders When Props Haven't Changed

`React.memo` wraps a component and skips re-rendering if its props are the same as the previous render.

> [!IMPORTANT]
> **You should be in:** `data-dash/client/`

### When to Use React.memo

```
USE React.memo when:
  ✓ The component is expensive to render (charts, large tables)
  ✓ The component receives the same props frequently
  ✓ The parent re-renders often but this component's data hasn't changed

DON'T USE React.memo when:
  ✗ The component is simple (a few divs and text)
  ✗ Props change on almost every render
  ✗ You haven't profiled and confirmed a performance issue
```

> [!WARNING]
> **Don't wrap everything in React.memo.** Memoization has a cost — React must compare all props on every render. For simple components, the comparison is more expensive than just re-rendering. Only memoize components where you've confirmed unnecessary re-renders.

### Memoize Chart Components

Charts are expensive because Recharts calculates layouts, scales, and SVG paths. If the chart data hasn't changed, there's no reason to recalculate.

Update `src/features/analytics/TimeSeriesChart.tsx` — wrap the export:

```typescript
import { memo } from 'react';
// ... rest of imports and component code unchanged ...

// Wrap the component with React.memo
export default memo(TimeSeriesChart);
```

Do the same for `CategoryChart.tsx` and `DeviceChart.tsx`:

```typescript
import { memo } from 'react';
// ... component code ...
export default memo(CategoryChart);
```

```typescript
import { memo } from 'react';
// ... component code ...
export default memo(DeviceChart);
```

Now, when a filter changes and the analytics data is refetched, charts only re-render when their specific data prop changes — not on every parent render.

---

## useMemo — Cache Expensive Calculations

`useMemo` caches the result of a calculation and only recomputes when its dependencies change.

### When to Use useMemo

```
USE useMemo when:
  ✓ The calculation is genuinely expensive (sorting, filtering, transforming large arrays)
  ✓ The result is used in rendering

DON'T USE useMemo when:
  ✗ The calculation is simple (string concatenation, basic math)
  ✗ You're caching a value just to pass it as a stable prop
```

### Example: Formatted Time Series Data

In `TimeSeriesChart.tsx`, the date formatting runs on every render even if the data hasn't changed:

```typescript
// Without useMemo — runs on every render
const data = timeSeries.map((point) => ({
  ...point,
  label: formatDate(point.date),
}));
```

With useMemo:

```typescript
import { useMemo, memo } from 'react';

// Inside the component:
const data = useMemo(
  () =>
    timeSeries.map((point) => ({
      ...point,
      label: formatDate(point.date),
    })),
  [timeSeries]   // Only recompute when timeSeries changes
);
```

The dependency array `[timeSeries]` tells React: "only recompute if `timeSeries` is a different array reference."

---

> [!TIP]
> **Session Break** — You've applied React.memo to chart components and useMemo for expensive calculations. Save your work and take a break.
> When you return, you'll add error boundaries, debouncing, and review production logging.

---

## useCallback — Stable Function References

`useCallback` returns a memoized version of a callback function. This prevents child components from re-rendering when the parent passes the same function.

### When to Use useCallback

```
USE useCallback when:
  ✓ Passing a function to a memoized child component (React.memo)
  ✓ The function is used as a dependency in useEffect/useMemo

DON'T USE useCallback when:
  ✗ The child component isn't memoized (the function reference doesn't matter)
  ✗ The function has no dependencies (define it outside the component instead)
```

### Example: Pagination Handler

In a parent component that passes a page handler to a memoized table:

```typescript
import { useCallback } from 'react';

// Without useCallback — new function reference on every render
const handlePageChange = (page: number) => {
  dispatch(setPage(page));
};

// With useCallback — same function reference unless dispatch changes
const handlePageChange = useCallback(
  (page: number) => {
    dispatch(setPage(page));
  },
  [dispatch]
);
```

> [!NOTE]
> **Technical:** JavaScript creates a new function object on every render. When a parent passes `onClick={handleClick}` to a `React.memo` child, the child sees a "new" prop and re-renders — even though the function does the same thing. `useCallback` returns the same function reference, so `React.memo` correctly skips the re-render.
>
> **Plain English:** Without `useCallback`, it's like giving someone a new copy of the same letter every day — they open it each time to check. With `useCallback`, you give them the same envelope — they see it's the same one and skip re-reading.

---

## Error Boundaries — Catch Component Crashes

If a chart component throws an error (bad data, missing property), it shouldn't crash the entire dashboard. Error boundaries catch errors in child components and display a fallback UI.

Create `src/features/ui/ErrorBoundary.tsx`:

```typescript
import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export default class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('ErrorBoundary caught:', error, errorInfo);
    // In production, send this to an error tracking service (Sentry, etc.)
  }

  render() {
    if (this.state.hasError) {
      return (
        this.props.fallback || (
          <div className="error-boundary">
            <h3>Something went wrong</h3>
            <p>{this.state.error?.message}</p>
            <button onClick={() => this.setState({ hasError: false, error: null })}>
              Try Again
            </button>
          </div>
        )
      );
    }

    return this.props.children;
  }
}
```

> [!NOTE]
> **Technical:** Error boundaries must be class components — there is no hook equivalent for `componentDidCatch`. This is one of the few cases where class components are still necessary in modern React. The error boundary catches errors during rendering, lifecycle methods, and constructors of child components.
>
> **Plain English:** An error boundary is like a circuit breaker. When a component short-circuits (throws an error), the boundary trips and shows a fallback message instead of letting the entire app go dark.

### Wrap Charts in Error Boundaries

Update `src/App.tsx` to wrap charts:

```typescript
import ErrorBoundary from './features/ui/ErrorBoundary';

// In the JSX:
{activeView === 'overview' && (
  <>
    <OverviewCards />
    <ErrorBoundary fallback={<div className="chart-placeholder">Chart failed to load.</div>}>
      <TimeSeriesChart />
    </ErrorBoundary>
    <div className="charts-row">
      <ErrorBoundary fallback={<div className="chart-placeholder">Chart failed to load.</div>}>
        <CategoryChart />
      </ErrorBoundary>
      <ErrorBoundary fallback={<div className="chart-placeholder">Chart failed to load.</div>}>
        <DeviceChart />
      </ErrorBoundary>
    </div>
  </>
)}
```

Now if one chart crashes, the others continue working. The user sees "Chart failed to load" instead of a blank screen.

Add the error boundary styles to `App.css`:

```css
/* --- Error Boundary --- */

.error-boundary {
  text-align: center;
  padding: 40px 20px;
  background-color: #1a1a2e;
  border: 1px solid #ef4444;
  border-radius: 12px;
  margin-bottom: 24px;
}

.error-boundary h3 {
  color: #ef4444;
  margin-bottom: 8px;
}

.error-boundary p {
  color: #888;
  margin-bottom: 16px;
  font-size: 0.85rem;
}

.error-boundary button {
  padding: 8px 16px;
  border: 1px solid #6366f1;
  background: none;
  color: #6366f1;
  border-radius: 6px;
  cursor: pointer;
}

.error-boundary button:hover {
  background-color: #6366f1;
  color: #fff;
}
```

---

## Debouncing — Avoid Excessive API Calls

When a user rapidly clicks filter buttons, each click dispatches an action and triggers a `useEffect` that fetches data. Five clicks in one second = five API calls. Debouncing waits for the user to stop clicking before making the call.

Create `src/utils/debounce.ts`:

```typescript
export function debounce<T extends (...args: unknown[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>;

  return (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}
```

### Apply Debouncing to Data Fetching

Update `src/App.tsx` to debounce the analytics fetch:

```typescript
import { useEffect, useMemo } from 'react';
import { debounce } from './utils/debounce';

// Inside the App component:
const debouncedFetch = useMemo(
  () =>
    debounce((filters: AnalyticsFilters) => {
      dispatch(fetchAllAnalytics(filters));
    }, 300),
  [dispatch]
);

useEffect(() => {
  debouncedFetch(filters);
}, [debouncedFetch, filters]);
```

Now filter changes wait 300ms before fetching. If the user changes three filters in quick succession, only one API call fires.

> [!NOTE]
> **Technical:** `debounce` wraps a function so it only executes after a specified delay since the last call. Each new call resets the timer. This is different from throttling, which executes at most once per time period regardless of calls.
>
> **Plain English:** Debouncing is like a elevator door — it waits for people to stop entering before closing. If someone keeps stepping in, the door keeps resetting its close timer. It only closes when there's a pause.

---

## Production Logging Review

Your pino logger already logs structured JSON in production. Here's what it captures:

```json
{"level":30,"time":1709312456789,"req":{"method":"GET","url":"/api/analytics/overview?category=marketing"},"res":{"statusCode":200},"responseTime":23,"msg":"request completed"}
```

Each field is searchable. In a log aggregator, you could:

- **Find slow requests**: `responseTime > 1000`
- **Find errors**: `res.statusCode >= 500`
- **Find specific endpoints**: `req.url CONTAINS "overview"`

This is the difference between structured logging and `console.log`. `console.log("Request completed")` gives you a string. Structured logging gives you a searchable database of request metadata.

---

## Step 8: Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
git add .
git commit -m "feat: add performance optimizations — memoization, error boundaries, debouncing"
```

---

> [!TIP]
> ## Spatial Check-In

1. **When should you NOT use React.memo?**

<details><summary>Answer</summary>

Don't use React.memo on simple components (a few divs with text), components whose props change on almost every render, or components you haven't profiled. Memoization has a cost — React must shallow-compare all props on every render. If the component is cheap to render, the comparison costs more than the render itself.

</details>

2. **What's the difference between useMemo and useCallback?**

<details><summary>Answer</summary>

`useMemo` caches the *result* of a function call: `useMemo(() => expensiveCalc(), [deps])` returns the calculated value. `useCallback` caches the *function itself*: `useCallback((x) => doSomething(x), [deps])` returns the function. Use `useMemo` for expensive calculations, `useCallback` for functions passed to memoized children.

</details>

3. **Why must error boundaries be class components?**

<details><summary>Answer</summary>

React doesn't provide a hook equivalent for `componentDidCatch` or `getDerivedStateFromError`. These are lifecycle methods that only exist on class components. This is a deliberate API gap — the React team hasn't shipped an `useErrorBoundary` hook. Error boundaries are the one remaining use case for class components in modern React.

</details>

4. **Why does the debounce use `useMemo` instead of just declaring the function?**

<details><summary>Answer</summary>

Without `useMemo`, a new debounced function would be created on every render — each with its own timer. The old timer from the previous render would still fire. `useMemo` ensures the same debounced function (with the same timer) persists across renders, so the debounce actually works.

</details>

---

| | | |
|:---|:---:|---:|
| [← 06 — Testing](../06-testing/) | [Level 4 Overview](../) | [08 — Deployment →](../08-deployment/) |
