`Level 4` **Step 2 of 9** — Project Setup

# 02 — Project Setup: Feature-Based Scaffold

## Spatial Orientation

Same monorepo pattern as Levels 2-3 — a `client/` and `server/` inside a single project folder. The new elements: **feature-based folder structure** on the frontend, a **middleware directory** on the backend, and **test configuration** in both.

```
data-dash/                  ← Project root
├── client/                 ← React + Redux + Recharts + Vitest
│   └── src/
│       ├── app/            ← Redux store + typed hooks     ← NEW
│       ├── features/       ← Feature folders               ← NEW
│       ├── services/       ← API calls (same as before)
│       ├── types/          ← Shared types
│       └── utils/          ← Formatting helpers            ← NEW
├── server/                 ← Express + middleware + services
│   ├── src/
│   │   ├── middleware/     ← Logger, rate limiter, etc.    ← NEW
│   │   ├── services/       ← SQL queries + business logic  ← NEW
│   │   └── routes/         ← Same as before
│   └── tests/              ← Supertest API tests           ← NEW
├── .env
└── .gitignore
```

---

## Step 1: Create the Project

> [!IMPORTANT]
> **You should be in:** your projects directory (e.g., `~/Desktop/Sites/` or wherever you keep projects)

```bash
mkdir data-dash && cd data-dash
```

> [!IMPORTANT]
> **You are now in:** `data-dash/`

```bash
git init
```

Create `.gitignore`:

```gitignore
node_modules/
dist/
.env
*.log
coverage/
```

> [!WARNING]
> **`coverage/` is new** — test coverage reports generate large folders that should never be committed.

Create a starter `README.md`:

```markdown
# DataDash — Analytics Dashboard

React + Redux Toolkit + TypeScript + Express + PostgreSQL

## Features
- Analytics overview with KPI cards
- Interactive charts (line, bar, pie) via Recharts
- Filterable, paginated events table
- Middleware stack (logging, rate limiting)
- Full test suite (Vitest, RTL, Supertest)

## Tech Stack
- **Frontend**: React, TypeScript, Redux Toolkit, Recharts
- **Backend**: Express, TypeScript, PostgreSQL
- **Testing**: Vitest, React Testing Library, Supertest
- **Middleware**: pino (logging), express-rate-limit

## Getting Started
1. `cd server && npm install && npm run dev`
2. `cd client && npm install && npm run dev`
3. Open http://localhost:5173
```

---

## Step 2: Set Up the Backend

```bash
mkdir -p server/src/{db,middleware,routes,services,types}
mkdir -p server/tests
```

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `data-dash/server/`

```bash
npm init -y
```

### Install Dependencies

```bash
npm install express cors dotenv pg pino pino-http express-rate-limit
```

| Package | Purpose | New? |
|---------|---------|------|
| `express` | Web framework | Carried from Level 1 |
| `cors` | Cross-origin requests | Carried from Level 1 |
| `dotenv` | Environment variables | Carried from Level 2 |
| `pg` | PostgreSQL client | Carried from Level 2 |
| `pino` | Structured JSON logging | **NEW** |
| `pino-http` | HTTP request logging middleware | **NEW** |
| `express-rate-limit` | Request rate limiting | **NEW** |

```bash
npm install typescript @types/node @types/express @types/cors @types/pg
```

```bash
npm install -D tsx vitest supertest @types/supertest pino-pretty
```

| Package | Purpose | New? |
|---------|---------|------|
| `typescript` | Type checking | Carried |
| `@types/node` | Types for `process`, `console`, Node.js globals | Carried |
| `@types/express` | Types for Express | Carried |
| `@types/cors` | Types for cors | Carried |
| `@types/pg` | Types for pg | Carried |

| Dev Package | Purpose | New? |
|-------------|---------|------|
| `tsx` | Run TypeScript directly | Carried |
| `vitest` | Test runner | **NEW** |
| `supertest` | HTTP assertion library | **NEW** |
| `@types/supertest` | Supertest types | **NEW** |
| `pino-pretty` | Readable log formatting in development | **NEW** |

> [!WARNING]
> **`@types/node` and other `@types` packages MUST be regular dependencies (not `--save-dev` / `-D`).**
>
> **Why?** Deployment platforms like Render run `npm install --omit=dev` in production, which skips devDependencies. If `@types/node` is a devDependency, `tsc` can't find types for `process`, `console`, or `Buffer` — your build fails with `TS2580: Cannot find name 'process'` and `TS2584: Cannot find name 'console'`.
>
> **Rule:** Any `@types` package needed during `npm run build` must be in `dependencies`, not `devDependencies`.

### TypeScript Configuration

Create `server/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "types": ["node"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### Package Scripts

Update `server/package.json` scripts:

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "seed": "tsx src/db/seed.ts",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

New scripts explained:

| Script | What it does |
|--------|-------------|
| `seed` | Populates the database with sample analytics data |
| `test` | Runs all tests once (for CI/deployment) |
| `test:watch` | Runs tests and re-runs on file changes (for development) |

---

## Step 3: Set Up the Frontend

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
npm create vite@latest client -- --template react-ts
cd client
npm install
```

> [!IMPORTANT]
> **You are now in:** `data-dash/client/`

### Install Dependencies

```bash
npm install @reduxjs/toolkit react-redux recharts
```

| Package | Purpose |
|---------|---------|
| `@reduxjs/toolkit` | Modern Redux with reduced boilerplate |
| `react-redux` | React bindings for Redux |
| `recharts` | Charting library built on D3 |

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

| Dev Package | Purpose |
|-------------|---------|
| `vitest` | Test runner (same as backend) |
| `jsdom` | Browser environment simulator for tests |
| `@testing-library/react` | React component testing utilities |
| `@testing-library/jest-dom` | Custom DOM matchers (toBeInTheDocument, etc.) |
| `@testing-library/user-event` | Simulate real user interactions in tests |

### Create Feature-Based Folder Structure

```bash
mkdir -p src/app
mkdir -p src/features/{analytics,events,filters,ui}
mkdir -p src/services
mkdir -p src/types
mkdir -p src/utils
```

Your `client/src/` should now look like:

```
src/
├── app/                     ← Redux store configuration
├── features/
│   ├── analytics/           ← Charts + overview cards
│   ├── events/              ← Events table
│   ├── filters/             ← Filter controls
│   └── ui/                  ← Layout + sidebar
├── services/                ← API client
├── types/                   ← Shared TypeScript types
├── utils/                   ← Helper functions
├── App.tsx                  ← Root component
├── App.css                  ← Global styles
└── main.tsx                 ← Entry point
```

> [!NOTE]
> **Technical:** Feature-based architecture groups code by domain (analytics, events, filters) rather than by type (components, hooks, slices). Each feature folder contains its Redux slice, components, and tests — everything needed to understand and modify that feature.
>
> **Plain English:** If someone asks you to "fix the filters," you open one folder. You don't hunt through components/, slices/, hooks/, and tests/ to find all the filter-related pieces.

### Vitest Configuration

Create `client/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test-setup.ts',
    css: true,
  },
});
```

Line-by-line:

| Line | Purpose |
|------|---------|
| `plugins: [react()]` | Enables JSX transformation in tests |
| `environment: 'jsdom'` | Simulates a browser DOM for component tests |
| `globals: true` | Makes `describe`, `it`, `expect` available without imports |
| `setupFiles` | Runs before each test file (for custom matchers) |
| `css: true` | Processes CSS imports instead of ignoring them |

Create the test setup file `client/src/test-setup.ts`:

```typescript
import '@testing-library/jest-dom';
```

This single import adds custom matchers like `toBeInTheDocument()`, `toHaveTextContent()`, and `toBeVisible()` to all test files.

Update `client/package.json` scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

### Vite Configuration

Update `client/vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
  },
});
```

---

> [!TIP]
> **Session Break** — You've set up both backend and frontend with all dependencies and test configuration. Save your work and take a break.
> When you return, you'll create the database, define shared types, and write utility functions.

---

## Step 4: Create the Database

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
createdb datadash
```

Create the environment file. Create `data-dash/.env`:

```env
DATABASE_URL=postgresql://localhost:5432/datadash
CORS_ORIGIN=http://localhost:5173
PORT=3001
```

> [!WARNING]
> **No JWT_SECRET in this project.** DataDash doesn't include authentication — you already built that in Level 3. This keeps the focus on state management, testing, and middleware.

---

## Step 5: Create Shared Types

These types are used by both the frontend and backend. Define them now so both sides agree on the data shape.

### Backend Types

Create `server/src/types/index.ts`:

```typescript
// --- Database row types ---

export interface AnalyticsEvent {
  id: number;
  event_type: string;       // page_view, signup, purchase, click
  category: string;         // marketing, product, support, engineering
  value: number;            // revenue for purchases, 0 otherwise
  country: string;          // US, UK, DE, FR, JP
  device: string;           // desktop, mobile, tablet
  browser: string;          // Chrome, Firefox, Safari, Edge
  page: string;             // /home, /pricing, /dashboard, /docs
  session_id: string;       // UUID
  created_at: string;       // ISO timestamp
}

// --- API response types ---

export interface OverviewStats {
  total_page_views: number;
  total_signups: number;
  total_revenue: number;
  avg_session_duration: number;
  period_start: string;
  period_end: string;
}

export interface TimeSeriesPoint {
  date: string;
  page_views: number;
  signups: number;
  revenue: number;
}

export interface CategoryBreakdown {
  category: string;
  count: number;
  revenue: number;
}

export interface DeviceBreakdown {
  device: string;
  count: number;
  percentage: number;
}

export interface TopPage {
  page: string;
  views: number;
  unique_sessions: number;
}

export interface PaginatedEvents {
  events: AnalyticsEvent[];
  total: number;
  page: number;
  limit: number;
  total_pages: number;
}

// --- Query filter types ---

export interface AnalyticsFilters {
  startDate?: string;
  endDate?: string;
  category?: string;
  device?: string;
}
```

Every type maps to an API response shape. When the frontend fetches `/api/analytics/overview`, it gets an `OverviewStats`. When it fetches `/api/analytics/events`, it gets `PaginatedEvents`.

### Frontend Types

Create `client/src/types/index.ts`:

```typescript
// Mirror the backend types exactly.
// In a real project, you'd share these via a package.
// For learning, we duplicate them.

export interface AnalyticsEvent {
  id: number;
  event_type: string;
  category: string;
  value: number;
  country: string;
  device: string;
  browser: string;
  page: string;
  session_id: string;
  created_at: string;
}

export interface OverviewStats {
  total_page_views: number;
  total_signups: number;
  total_revenue: number;
  avg_session_duration: number;
  period_start: string;
  period_end: string;
}

export interface TimeSeriesPoint {
  date: string;
  page_views: number;
  signups: number;
  revenue: number;
}

export interface CategoryBreakdown {
  category: string;
  count: number;
  revenue: number;
}

export interface DeviceBreakdown {
  device: string;
  count: number;
  percentage: number;
}

export interface TopPage {
  page: string;
  views: number;
  unique_sessions: number;
}

export interface PaginatedEvents {
  events: AnalyticsEvent[];
  total: number;
  page: number;
  limit: number;
  total_pages: number;
}

export interface AnalyticsFilters {
  startDate?: string;
  endDate?: string;
  category?: string;
  device?: string;
}
```

> [!NOTE]
> **Technical:** Duplicating types between frontend and backend isn't ideal. In production, you'd use a shared package (monorepo workspace) or generate types from an API schema (OpenAPI/Swagger). For learning, duplication keeps the setup simple.
>
> **Plain English:** Both sides need to agree on the data shape. We're writing it twice instead of building a complex sharing system. The important thing is that they match.

---

## Step 6: Create Utility Functions

Create `client/src/utils/formatters.ts`:

```typescript
/**
 * Format a number with commas: 12847 → "12,847"
 */
export function formatNumber(num: number): string {
  return num.toLocaleString();
}

/**
 * Format currency: 8291.5 → "$8,291.50"
 */
export function formatCurrency(num: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(num);
}

/**
 * Format seconds as duration: 272 → "4:32"
 */
export function formatDuration(seconds: number): string {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins}:${secs.toString().padStart(2, '0')}`;
}

/**
 * Format a date string: "2025-03-15" → "Mar 15"
 */
export function formatDate(dateStr: string): string {
  const date = new Date(dateStr + 'T00:00:00');
  return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

/**
 * Format a full date: "2025-03-15T10:30:00Z" → "Mar 15, 2025 10:30 AM"
 */
export function formatDateTime(dateStr: string): string {
  const date = new Date(dateStr);
  return date.toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
    hour: 'numeric',
    minute: '2-digit',
  });
}

/**
 * Format a percentage: 0.623 → "62.3%"
 */
export function formatPercentage(decimal: number): string {
  return `${(decimal * 100).toFixed(1)}%`;
}
```

These are pure functions — they take input and return output with no side effects. Pure functions are the easiest code to test, which you'll do in Step 6 (Testing).

---

## Step 7: Initial Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: scaffold data-dash with feature-based structure and test config"
```

---

## What You've Built So Far

```
data-dash/
├── client/                          ← React + Redux + Recharts
│   ├── src/
│   │   ├── app/                     ← (empty — Redux store next)
│   │   ├── features/
│   │   │   ├── analytics/           ← (empty — charts next)
│   │   │   ├── events/              ← (empty — table next)
│   │   │   ├── filters/             ← (empty — filters next)
│   │   │   └── ui/                  ← (empty — layout next)
│   │   ├── services/                ← (empty — API calls next)
│   │   ├── types/index.ts           ✓ All shared types
│   │   ├── utils/formatters.ts      ✓ Number/date formatters
│   │   └── test-setup.ts            ✓ Testing Library matchers
│   ├── vitest.config.ts             ✓ Test configuration
│   ├── vite.config.ts               ✓ Dev server config
│   └── package.json                 ✓ All dependencies
├── server/
│   ├── src/
│   │   ├── db/                      ← (empty — schema next)
│   │   ├── middleware/               ← (empty — middleware next)
│   │   ├── routes/                  ← (empty — routes next)
│   │   ├── services/                ← (empty — services next)
│   │   └── types/index.ts           ✓ All shared types
│   ├── tests/                       ← (empty — API tests next)
│   ├── tsconfig.json                ✓ TypeScript config
│   └── package.json                 ✓ All dependencies
├── .env                             ✓ DATABASE_URL, CORS_ORIGIN, PORT
├── .gitignore                       ✓ Includes coverage/
└── README.md                        ✓ Project overview
```

The scaffold is ready. Next, we build the backend — the middleware stack, database, and analytics API.

---

> [!TIP]
> ## Spatial Check-In

1. **Why is `coverage/` in `.gitignore`?**

<details><summary>Answer</summary>

Test coverage reports are generated output — they contain HTML, JSON, and source maps that can be thousands of files. They're generated fresh every time you run `vitest --coverage`, so committing them wastes space and creates noisy diffs.

</details>

2. **Why does the Vitest config specify `environment: 'jsdom'`?**

<details><summary>Answer</summary>

React components render to a DOM. Node.js doesn't have a DOM. jsdom is a JavaScript implementation of the DOM that runs in Node.js, allowing tests to render React components, query for elements, and simulate user interactions without a real browser.

</details>

3. **What's the difference between `vitest run` and `vitest` (no arguments)?**

<details><summary>Answer</summary>

`vitest run` executes all tests once and exits — this is what you use in CI/deployment scripts. `vitest` (no arguments) starts in watch mode — it runs tests, then watches for file changes and re-runs affected tests. Watch mode is for development.

</details>

---

| | | |
|:---|:---:|---:|
| [← 01 — Orientation](../01-orientation/) | [Level 4 Overview](../) | [03 — Backend →](../03-backend/) |
