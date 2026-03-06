# Level 4 — DataDash: Analytics Dashboard

> Scalable state management, automated testing, and production middleware.

## What You'll Build

**DataDash** is an analytics dashboard that displays charts, summary cards, a filterable events table, and real-time-ish updates. Multiple components share the same data. Filters in one component affect charts in another. A full test suite covers the backend API, Redux state, and React components.

This is the level where you stop building features and start building **engineering practices**.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│   ┌────────────────────────────────────┐      ┌───────────────────┐   │
│   │           FRONTEND                 │      │     BACKEND       │   │
│   │                                    │      │                   │   │
│   │  ┌──────────┐  ┌──────────┐       │      │  ┌─────────────┐ │   │
│   │  │ Charts   │  │ Tables   │       │      │  │ Middleware  │ │   │
│   │  │          │  │          │       │      │  │ Stack:      │ │   │
│   │  └────┬─────┘  └────┬─────┘       │      │  │  - Logger   │ │   │
│   │       │              │             │      │  │  - RateLimit│ │   │
│   │  ┌────▼──────────────▼─────┐      │      │  │  - CORS     │ │   │
│   │  │    REDUX STORE          │      │ HTTP │  │  - Validate │ │   │
│   │  │    (Single source of    │──────┼─────▶│  └──────┬──────┘ │   │
│   │  │     truth)              │◀─────┼──────│         │        │   │
│   │  │                         │      │      │  ┌──────▼──────┐ │   │
│   │  │  - analyticsSlice       │      │      │  │  Services   │ │   │
│   │  │  - eventsSlice          │      │      │  │  (SQL)      │ │   │
│   │  │  - filtersSlice         │      │      │  └──────┬──────┘ │   │
│   │  │  - uiSlice              │      │      │         │        │   │
│   │  └─────────────────────────┘      │      │  ┌──────▼──────┐ │   │
│   └────────────────────────────────────┘      │  │ PostgreSQL  │ │   │
│                                                │  └─────────────┘ │   │
│   ┌────────────────────────────────────┐      └───────────────────┘   │
│   │          TEST LAYER                │                              │
│   │                                    │                              │
│   │  - Unit tests (Vitest)             │                              │
│   │  - Component tests (RTL)           │                              │
│   │  - API tests (Supertest)           │                              │
│   └────────────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

```
USER ACTION (clicks filter)
  │
  ▼
FilterBar dispatches setCategory("marketing")
  │
  ▼
filtersSlice reducer updates state
  │
  ▼
analyticsSlice thunk re-fetches with new filters
  │
  ▼
GET /api/analytics/overview?category=marketing
  │
  ▼
Request → Logger → Rate Limiter → CORS → Route Handler
  │
  ▼
analyticsService.getOverview({ category: "marketing" })
  │
  ▼
SELECT COUNT(*), SUM(value) FROM analytics_events WHERE category = $1
  │
  ▼
Response → OverviewCards re-render → Charts re-render
```

---

## Folder Structure

```
data-dash/
├── client/
│   ├── src/
│   │   ├── app/
│   │   │   ├── store.ts              ← Redux store configuration
│   │   │   └── hooks.ts             ← Typed useAppSelector, useAppDispatch
│   │   ├── features/
│   │   │   ├── analytics/           ← Overview stats, charts
│   │   │   ├── events/              ← Events table, pagination
│   │   │   ├── filters/             ← Date range, category, device
│   │   │   └── ui/                  ← Layout, sidebar
│   │   ├── services/
│   │   │   └── api.ts               ← All API calls
│   │   ├── types/
│   │   │   └── index.ts             ← Shared TypeScript types
│   │   ├── utils/
│   │   │   └── formatters.ts        ← Number/date formatting
│   │   ├── App.tsx
│   │   ├── App.css
│   │   └── main.tsx
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── vitest.config.ts
├── server/
│   ├── src/
│   │   ├── db/
│   │   │   ├── index.ts             ← Pool connection
│   │   │   ├── schema.sql           ← analytics_events table
│   │   │   └── seed.ts              ← Generate 90 days of data
│   │   ├── middleware/
│   │   │   ├── logger.ts            ← Structured logging (pino)
│   │   │   ├── rateLimiter.ts       ← Request rate limiting
│   │   │   ├── validator.ts         ← Query param validation
│   │   │   └── errorHandler.ts      ← Centralized error handling
│   │   ├── routes/
│   │   │   └── analytics.ts         ← All analytics endpoints
│   │   ├── services/
│   │   │   └── analyticsService.ts  ← SQL queries + business logic
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts                 ← Server entry point
│   ├── tests/
│   │   └── analytics.test.ts        ← Supertest API tests
│   ├── package.json
│   └── tsconfig.json
├── .env
├── .gitignore
└── README.md
```

---

## Session Guide

Level 4 takes approximately **13–16 sessions** (30–60 minutes each).

| Lesson | Sessions | Notes |
|--------|----------|-------|
| 01 — Orientation | 1 | Read and absorb in one sitting |
| 02 — Project Setup | 2 | Break after backend deps, before frontend deps |
| 03 — Backend | 3 | Breaks after middleware stack and after service layer |
| 04 — State Management | 3 | Breaks after filtersSlice and after analyticsSlice |
| 05 — Frontend | 4 | Breaks after FilterBar, after charts, and after EventsTable |
| 06 — Testing | 3 | Breaks after unit tests and after component tests |
| 07 — Performance | 2 | Break after memoization, before error boundaries |
| 08 — Deployment | 1 | Complete in one sitting |
| 09 — Growth Review | 1 | Complete in one sitting |

Look for **Session Break** markers inside each lesson — they tell you exactly when to pause.

## Learning Path

| Step | Topic | What You Build |
|------|-------|----------------|
| [01 — Orientation](./01-orientation/) | Prerequisites + architecture | Mental model for scalable apps |
| [02 — Project Setup](./02-project-setup/) | Scaffold + tooling | Feature-based folder structure |
| [03 — Backend](./03-backend/) | Middleware + API + seed data | Analytics API with middleware stack |
| [04 — State Management](./04-state/) | Redux Toolkit | Store, slices, thunks, selectors |
| [05 — Frontend](./05-frontend/) | Dashboard components | Charts, tables, filters with Recharts |
| [06 — Testing](./06-testing/) | Vitest + RTL + Supertest | Full test suite |
| [07 — Performance](./07-performance/) | Optimization + production | React.memo, error boundaries, logging |
| [08 — Deployment](./08-deployment/) | Ship with test suite | Production with seeded data |
| [09 — Growth Review](./09-growth/) | Portfolio + interviews | How to present this project |

---

## New Concepts in This Level

| Concept | Why Now |
|---------|---------|
| **Redux Toolkit** | useState breaks down when many components share state |
| **Feature-based folders** | Flat file structures don't scale past 15-20 files |
| **Service layer** | Direct SQL in routes mixes concerns — services separate them |
| **Middleware stack** | Logging, rate limiting, and validation should happen before routes |
| **Vitest** | Automated tests catch regressions and prove correctness |
| **React Testing Library** | Test what users see, not implementation details |
| **Supertest** | Test API endpoints without starting a real server |
| **React.memo / useMemo** | Dashboard components are expensive — avoid unnecessary re-renders |
| **Error boundaries** | Charts crashing shouldn't take down the whole page |

---

## Prerequisites

- Completed [Level 1 — DevPulse](../level-1-foundations/)
- Completed [Level 2 — TaskForge](../level-2-crud/)
- Completed [Level 3 — VaultNote](../level-3-auth/)

---

| | |
|:---|---:|
| [← Level 3 — VaultNote](../level-3-auth/) | [Level 5 — CollabBoard →](../level-5-production/) |
