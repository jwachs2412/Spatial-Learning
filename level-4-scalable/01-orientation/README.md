`Level 4` **Step 1 of 9** — Orientation

# 01 — Orientation: From Features to Engineering

## Prerequisites Gate

> [!WARNING]
> **You must have ALL THREE previous levels deployed before starting Level 4.**
>
> Verify you have these working URLs:
>
> | Check | What to verify |
> |-------|---------------|
> | **Level 1 — DevPulse GitHub** | Repository is public, code is visible |
> | **Level 1 — DevPulse Frontend** | Vercel URL loads the app |
> | **Level 1 — DevPulse Backend** | Render URL `/api/health` returns JSON |
> | **Level 2 — TaskForge GitHub** | Repository is public, code is visible |
> | **Level 2 — TaskForge Frontend** | Vercel URL loads the app |
> | **Level 2 — TaskForge Backend** | Render URL `/api/health` returns `{"status":"ok","database":"connected"}` |
> | **Level 3 — VaultNote GitHub** | Repository is public, code is visible |
> | **Level 3 — VaultNote Frontend** | Vercel URL loads login/register page |
> | **Level 3 — VaultNote Backend** | Render URL `/api/health` returns JSON with database connected |
>
> **Skills check** — you should be comfortable with:
> - [ ] Building React components with TypeScript
> - [ ] Creating Express routes with parameterized SQL queries
> - [ ] useState and useEffect for state management
> - [ ] Full CRUD (Create, Read, Update, Delete)
> - [ ] JWT authentication and middleware
> - [ ] Git workflow (add, commit, push)
> - [ ] Deploying frontend (Vercel) and backend + database (Render)
>
> If anything above is incomplete, go back and finish it. Level 4 builds directly on all three levels.

---

## What Changes in Level 4

Levels 1-3 taught you how to **build features**. Level 4 teaches you how to **build them well**.

```
LEVELS 1-3: "Does it work?"
  ✓ Frontend renders data
  ✓ Backend serves API
  ✓ Database persists data
  ✓ Auth protects routes

LEVEL 4: "Does it scale, is it tested, is it maintainable?"
  → What happens when 10 components need the same data?
  → How do you know a code change didn't break something?
  → How do you find bugs in production?
  → What happens when someone sends 1000 requests per second?
```

---

## Why State Management Matters Now

In Level 1 (DevPulse), you had one component that needed data. `useState` was perfect.

In Level 2 (TaskForge), you had a few components sharing state through a parent. Props worked.

In Level 3 (VaultNote), you introduced Context to share auth state. This avoided prop drilling.

In Level 4, you'll have a dashboard where:

```
┌──────────────────────────────────────────────────────┐
│                                                        │
│  FilterBar  ──sets filters──▶  Redux Store             │
│                                    │                   │
│  OverviewCards  ◀──reads stats─────┤                   │
│  TimeSeriesChart ◀──reads data─────┤                   │
│  CategoryChart  ◀──reads data──────┤                   │
│  DeviceChart  ◀──reads data────────┤                   │
│  EventsTable  ◀──reads events──────┘                   │
│                                                        │
│  Changing a filter must update ALL of these.           │
│  useState can't do this without prop drilling.         │
│  Context re-renders EVERY consumer on ANY change.      │
│  Redux provides surgical updates via selectors.        │
│                                                        │
└──────────────────────────────────────────────────────┘
```

> [!NOTE]
> **Technical:** React Context re-renders every component that calls `useContext` whenever any value in the context changes. Redux uses selectors that only trigger re-renders when the specific slice of state a component reads has changed.
>
> **Plain English:** Context is like a megaphone — when anything changes, everyone hears it. Redux is like individual mailboxes — each component only gets notified about the mail it subscribed to.

---

## Why Testing Matters Now

Until now, you tested manually: click buttons, check the screen, try different inputs.

Manual testing has problems:

```
MANUAL TESTING:
  - You change the filter logic
  - You test the filter → it works
  - You forget to test the overview cards → they're broken
  - You deploy → users see broken cards
  - You spend 2 hours debugging something tests would have caught in 2 seconds

AUTOMATED TESTING:
  - You change the filter logic
  - You run `npm test` → 47 tests pass, 2 fail
  - The failures tell you EXACTLY what broke and WHERE
  - You fix it before deploying
  - Total time: 3 minutes
```

> [!NOTE]
> **Technical:** Automated tests are functions that assert expected behavior. Unit tests verify individual functions. Component tests verify UI rendering and interaction. Integration tests verify API endpoints. A test suite runs all tests and reports pass/fail.
>
> **Plain English:** Tests are robot QA testers that check your app in seconds. They never forget to check something, never get tired, and run every time you make a change. They're the safety net that lets you change code with confidence.

---

## What is Feature-Based Architecture

In Levels 1-3, you organized files by type:

```
BY TYPE (Levels 1-3):              BY FEATURE (Level 4):
src/                                src/
├── components/                     ├── features/
│   ├── Header.tsx                  │   ├── analytics/
│   ├── NoteList.tsx                │   │   ├── analyticsSlice.ts
│   ├── NoteEditor.tsx              │   │   ├── OverviewCards.tsx
│   └── LoginForm.tsx               │   │   ├── TimeSeriesChart.tsx
├── pages/                          │   │   └── CategoryChart.tsx
│   ├── LoginPage.tsx               │   ├── events/
│   └── NotesPage.tsx               │   │   ├── eventsSlice.ts
├── context/                        │   │   └── EventsTable.tsx
│   └── AuthContext.tsx             │   ├── filters/
└── services/                       │   │   ├── filtersSlice.ts
    └── api.ts                      │   │   └── FilterBar.tsx
                                    │   └── ui/
                                    │       ├── uiSlice.ts
                                    │       ├── Layout.tsx
                                    │       └── Sidebar.tsx
                                    ├── app/
                                    │   ├── store.ts
                                    │   └── hooks.ts
                                    └── services/
                                        └── api.ts
```

> [!NOTE]
> **Technical:** Feature-based architecture co-locates related code — a feature's state, components, and tests live together. This reduces cognitive load when working on a feature because everything you need is in one folder.
>
> **Plain English:** Instead of putting all shirts in one drawer and all pants in another, you organize by outfit — everything for "work" is together, everything for "gym" is together. When you need to change the gym outfit, you open one drawer instead of three.

---

## What DataDash Does

DataDash displays analytics for a fictional web product. Users see:

1. **Overview Cards** — four KPI summary cards (page views, signups, revenue, avg session duration)
2. **Time Series Chart** — line chart showing page views over the last 30/60/90 days
3. **Category Chart** — bar chart breaking down events by category
4. **Device Chart** — pie chart showing desktop vs mobile vs tablet
5. **Events Table** — paginated, filterable table of all analytics events
6. **Filter Bar** — date range, category, and device filters that update all views

### Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│  DataDash                          [30d] [60d] [90d]        │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                   │
│  Overview│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐│
│          │  │  Views   │ │ Signups  │ │ Revenue  │ │ Avg  ││
│  Events  │  │  12,847  │ │   342    │ │ $8,291   │ │ 4:32 ││
│          │  └──────────┘ └──────────┘ └──────────┘ └──────┘│
│          │                                                   │
│          │  ┌────────────────────────────────────────────┐  │
│          │  │         Page Views Over Time               │  │
│          │  │    /\      /\                              │  │
│          │  │   /  \    /  \    /\                       │  │
│          │  │  /    \  /    \  /  \_                     │  │
│          │  │ /      \/      \/                          │  │
│          │  └────────────────────────────────────────────┘  │
│          │                                                   │
│          │  ┌─────────────────┐  ┌──────────────────────┐  │
│          │  │  By Category    │  │  By Device           │  │
│          │  │  ██ ██ ██ ██   │  │     ████             │  │
│          │  │  ██ ██ ██ ██   │  │   ██    ██           │  │
│          │  │  ██ ██ ██ ██   │  │  █ 62%   █           │  │
│          │  └─────────────────┘  └──────────────────────┘  │
└──────────┴──────────────────────────────────────────────────┘
```

### API Overview

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/analytics/overview` | Summary KPIs (counts, sums) |
| GET | `/api/analytics/time-series` | Daily data for line charts |
| GET | `/api/analytics/categories` | Category breakdown for bar charts |
| GET | `/api/analytics/devices` | Device breakdown for pie charts |
| GET | `/api/analytics/events` | Paginated, filterable events list |
| GET | `/api/analytics/top-pages` | Most viewed pages |
| GET | `/api/health` | Health check with DB ping |

All endpoints accept query parameters: `startDate`, `endDate`, `category`, `device`.

---

## The Middleware Stack

Every request passes through middleware before reaching a route handler:

```
Request
  │
  ▼
┌──────────────────┐
│  Logger (pino)   │  ← Logs method, URL, status, response time
├──────────────────┤
│  Rate Limiter    │  ← Blocks IPs that exceed request limits
├──────────────────┤
│  CORS            │  ← Allows frontend origin
├──────────────────┤
│  JSON Parser     │  ← Parses request body
├──────────────────┤
│  Route Handler   │  ← Your analytics routes
├──────────────────┤
│  Error Handler   │  ← Catches any thrown errors, sends clean response
└──────────────────┘
  │
  ▼
Response
```

> [!NOTE]
> **Technical:** Express middleware functions execute in order. Each calls `next()` to pass control to the next middleware. Error-handling middleware has four parameters `(err, req, res, next)` and catches errors from any preceding middleware or route handler.
>
> **Plain English:** Middleware is like airport security checkpoints. Every passenger (request) goes through them in order: check ID, scan bags, metal detector. If any checkpoint fails, the passenger is stopped. The route handler is the gate — you only reach it if you pass all checkpoints.

---

## What You Won't Build (and Why)

| Omitted Feature | Reason |
|-----------------|--------|
| **Authentication** | You built it in Level 3. DataDash focuses on state management and testing. |
| **React Router** | Dashboard is a single page with sidebar views. Keeps focus on Redux. |
| **Real-time WebSockets** | Level 5 concept. DataDash uses polling on demand. |
| **ORM (Prisma, Drizzle)** | Raw SQL teaches what ORMs abstract. Same reasoning as Level 2. |
| **CI/CD pipeline** | Level 5 introduces GitHub Actions. Level 4 runs tests locally. |

---

> [!TIP]
> ## Spatial Check-In

1. **Why can't React Context replace Redux for DataDash?**

<details><summary>Answer</summary>

Context re-renders every consumer when any value changes. In a dashboard with 6+ components reading different pieces of state, this means changing a filter re-renders the sidebar, the charts, the table, and everything else — even components that don't care about filters. Redux selectors only trigger re-renders when the specific data a component reads has changed.

</details>

2. **What is the difference between unit tests, component tests, and API tests?**

<details><summary>Answer</summary>

Unit tests verify individual functions (like a Redux reducer or utility function) in isolation. Component tests render React components and verify they display the right content and respond to user interaction. API tests send HTTP requests to your Express server and verify the response status and body. Together they form a testing pyramid that catches bugs at different levels.

</details>

3. **Why does every request go through the logger middleware?**

<details><summary>Answer</summary>

Structured logging records every request's method, URL, status code, and response time. In production, this is how you debug issues — you can search logs for slow requests, error responses, or unusual patterns. Without logging, debugging production problems is guesswork.

</details>

---

| | | |
|:---|:---:|---:|
| [← Level 3 — VaultNote](../../level-3-auth/) | [Level 4 Overview](../) | [02 — Project Setup →](../02-project-setup/) |
