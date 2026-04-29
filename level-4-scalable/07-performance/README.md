`Level 4` **Step 7 of 9** — Performance

# 07 — Performance: Optimization and Production Hardening

## What's New: Performance Patterns

Most of this lesson is small wraps and configurations on top of code you've already written. The handful of genuinely new shapes:

- **`memo(Component)`** (from `react`) — wraps a component so React skips re-rendering when props haven't changed.
- **`useMemo(fn, deps)`** — caches the **result** of an expensive calculation. Recomputes only when something in `deps` changes.
- **`useCallback(fn, deps)`** — caches the **function itself**. Recomputes only when something in `deps` changes. Used to pass stable function references to memoized children.
- **Class components** (`extends Component<Props, State>`) — the older React component style. Required for **error boundaries** because no hook exists for `componentDidCatch`. We'll see this once and never again.
- **`getDerivedStateFromError`** and **`componentDidCatch`** — class-component lifecycle methods that React calls when a child component throws.
- **TypeScript advanced generics** — `T extends (...args: unknown[]) => void` constrains a generic to "any function that takes any args and returns nothing." `Parameters<T>` extracts the parameter types of a function type. Used to type the `debounce` helper without losing type safety.
- **`setTimeout` / `clearTimeout`** — built-in browser timers. `setTimeout(fn, ms)` schedules `fn` to run after `ms` milliseconds and returns a timer ID. `clearTimeout(id)` cancels a pending timer.

Every code block below has a "Reading This File Line-by-Line" walkthrough.

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

### Reading This Pattern

- `import { memo } from 'react';` — pull `memo` (a function) from React. Despite the historical name "React.memo", we just import the function directly.
- `memo(TimeSeriesChart)` — wraps the component in a memoized version. Returns a new component that:
  - On its first render, renders normally and remembers its props.
  - On subsequent renders, **shallow-compares** the new props to the remembered ones. If every top-level prop is `===` to the previous one, React reuses the previous output and skips the function body entirely. If anything changed, it renders normally.
- `export default memo(TimeSeriesChart);` — replace the default export with the wrapped version. Imports elsewhere are unchanged; consumers use the memoized one automatically.

**Shallow comparison** means props are compared with `===`, not deeply inspected. So `{ a: 1 }` !== `{ a: 1 }` (different object instances). For memo to actually skip renders, parents must avoid creating new prop objects each render — which is why `useMemo` and `useCallback` exist.

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

### Reading `useMemo` Line-by-Line

- `useMemo(factoryFn, depsArray)` — React hook with two arguments.
- The first argument is a **factory function** (`() => ...`). React calls it once on first render and caches the result.
- The second argument is the **dependency array**. On every render, React shallow-compares each item to the previous render's array. If anything changed, React calls the factory again and updates the cache. If everything is the same, React returns the cached value without calling the factory.
- Here, the factory does `timeSeries.map(...)` — building a new array of formatted points.
- `[timeSeries]` is the deps. As long as `timeSeries` is the same array reference (which it is, until Redux updates the analytics slice), the factory doesn't re-run.

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

### Reading `useCallback`

`useCallback(fn, deps)` and `useMemo(() => fn, deps)` are nearly equivalent. `useCallback(x, y)` is just a shortcut for `useMemo(() => x, y)` when the cached value is itself a function.

- The arrow function `(page: number) => { dispatch(setPage(page)); }` is the function we want to remember.
- `[dispatch]` — re-create the function only if `dispatch` changes. In Redux, `useAppDispatch` returns a stable reference, so `dispatch` never changes — meaning the callback is created exactly once.
- The returned function is `===` to itself across renders. Memoized children that receive it as a prop see "same reference" and skip re-rendering.

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

### Reading This File Line-by-Line

This is the only **class component** in the entire curriculum. Functional components don't have access to `componentDidCatch`, so we have no choice. Read carefully — the syntax is different from anything before.

```typescript
import { Component, ReactNode } from 'react';
```

`Component` is the base class for class components.

```typescript
interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}
```

Class components have **two** type parameters: a `Props` shape (data passed in) and a `State` shape (internal data). Each is an interface.

```typescript
export default class ErrorBoundary extends Component<Props, State> {
```

- `class ErrorBoundary` — declaring a class.
- `extends Component<Props, State>` — inheriting from React's `Component` class. The `<Props, State>` is a **generic** specifying the type parameters: this component accepts `Props` and manages `State`.

```typescript
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
```

- `constructor(props)` — runs when the component is instantiated.
- `super(props)` — call the parent class's constructor (`Component`). Required — JavaScript throws if you don't call `super` before accessing `this`.
- `this.state = { ... }` — initialize the component's state. In class components, `state` is an object stored on the instance. (Functional components use `useState` instead.)

```typescript
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
```

- `static` — this method belongs to the class itself, not instances. React calls it directly on the class.
- React calls this when a child throws during rendering. Whatever you return becomes the new state.
- We return `{ hasError: true, error }` so the next render shows the fallback UI.

```typescript
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('ErrorBoundary caught:', error, errorInfo);
  }
```

- An instance method — runs on the component instance.
- React calls this after `getDerivedStateFromError`, with extra info (component stack trace).
- Used for **side effects** like logging or sending the error to an error-tracking service. Don't do state updates here — `getDerivedStateFromError` already did that.

```typescript
  render() {
    if (this.state.hasError) {
      return (
        this.props.fallback || (
          <div className="error-boundary">
            <h3>Something went wrong</h3>
            ...
          </div>
        )
      );
    }

    return this.props.children;
  }
}
```

- `render()` is the class-component equivalent of the function body in a functional component.
- `this.state` and `this.props` access the component's data.
- If we caught an error, render either the user-provided `fallback` (if any) or the default error UI. The `||` falls back to the default when `fallback` is undefined.
- Otherwise, render `this.props.children` — pass through whatever was wrapped.
- The `Try Again` button calls `this.setState({ hasError: false, error: null })` to reset, attempting to re-render the children.

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

### Reading This File Line-by-Line

The most TypeScript-heavy snippet in Level 4. Let's unpack the type signature first, then the runtime behavior.

```typescript
export function debounce<T extends (...args: unknown[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
```

- `<T extends (...args: unknown[]) => void>` — generic type parameter `T` constrained to "any function that takes any args and returns nothing." `T` is whatever function the caller passes in.
- `(...args: unknown[])` — **rest parameter syntax** in a type. Means "any number of arguments, of any types." `unknown` is TypeScript's "I don't know the type yet, force me to check before using."
- `=> void` — the function returns nothing meaningful.
- `fn: T` — the first runtime parameter, typed by the generic.
- `delay: number` — milliseconds to wait.
- `: (...args: Parameters<T>) => void` — the return type. `Parameters<T>` is a TypeScript utility that **extracts the parameter types of a function type**. So if `T` is `(s: string, n: number) => void`, then `Parameters<T>` is `[string, number]`. The returned function has the same parameter signature as the wrapped one.

The whole declaration says: "give me any function plus a delay; I return a new function with the same signature."

```typescript
  let timeoutId: ReturnType<typeof setTimeout>;
```

- `setTimeout` returns a timer ID (a number in browsers, a `Timer` object in Node — different types in different environments).
- `ReturnType<typeof setTimeout>` extracts whatever type `setTimeout` returns, automatically. Cross-platform safe.
- `let timeoutId` — declared without an initial value because we set it later. TypeScript allows this because we always assign before reading.

```typescript
  return (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}
```

The returned function is called every time the user triggers the debounced event:

- `(...args: Parameters<T>)` — capture all arguments using rest syntax.
- `clearTimeout(timeoutId)` — cancel any pending timer. If this is the first call, `timeoutId` is undefined and `clearTimeout(undefined)` is a no-op.
- `timeoutId = setTimeout(() => fn(...args), delay)` — schedule a new timer. After `delay` ms, the inner arrow runs and calls `fn(...args)` with the most recent arguments.

The clever part: every new call cancels the previous timer. So if the user clicks five times in 200ms with a 300ms delay, only the last click's timer ever fires.

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

### Reading the Debounced Fetch

```typescript
const debouncedFetch = useMemo(
  () =>
    debounce((filters: AnalyticsFilters) => {
      dispatch(fetchAllAnalytics(filters));
    }, 300),
  [dispatch]
);
```

- We're using `useMemo` to cache the debounced function across renders. **This matters.** If we wrote `const debouncedFetch = debounce(...)` directly, we'd build a new debounced function every render — each with its own internal `timeoutId`. The previous one's timer would still fire. Debouncing wouldn't actually work.
- `useMemo` keeps the same debounced function across renders, so the same internal `timeoutId` is reused. Calls genuinely cancel each other.
- `[dispatch]` — re-create only if `dispatch` changes (which it never does). One debounced function for the lifetime of the component.

```typescript
useEffect(() => {
  debouncedFetch(filters);
}, [debouncedFetch, filters]);
```

Same `useEffect` pattern as before, but calling the debounced version. When `filters` changes, this effect runs immediately — but the actual `dispatch(fetchAllAnalytics(filters))` is deferred 300ms.

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
