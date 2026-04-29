`Level 5` **Step 7 of 9** — CI/CD & Monitoring

# 07 — CI/CD & Monitoring: GitHub Actions and Sentry

## What's New You'll Meet in This Lesson

This lesson introduces **YAML** (a config-file format) and a few patterns specific to GitHub Actions and Sentry:

- **YAML** — a markup language designed for human-friendly config files. **Indentation is significant** (like Python). Lists use `- item`. Key-value pairs use `key: value`. There are no curly braces or commas — structure comes from indentation alone. The `.yml` and `.yaml` extensions are interchangeable.
- **GitHub Actions workflow file** — `.github/workflows/*.yml`. GitHub watches this directory and runs the file's instructions when triggered.
- **`uses: action-name@version`** — an Actions-specific keyword for "use this prebuilt action." `actions/checkout@v4` clones the repo; `actions/setup-node@v4` installs Node.
- **Service containers** — `services:` block that spins up Docker containers (like Postgres) alongside the test environment. Lets your tests hit a real database in CI.
- **`${{ env.DATABASE_URL }}`** — Actions' template syntax for substituting variables into the YAML at runtime.
- **`Sentry.init(config)`** — the React SDK's setup function. Configures DSN, environment, and integrations. Call once in `main.tsx`.
- **`Sentry.ErrorBoundary`** — a drop-in error boundary that ALSO reports to Sentry. Replaces the hand-rolled class component from Level 4 when you want telemetry.
- **`Sentry.setUser({...})`** — attaches user identity to subsequent error reports. Lets you filter Sentry by user.
- **`useRef<HTMLButtonElement>(null)`** — React's hook for getting a stable reference to a DOM element. Used in focus management.

Every code block below has a "Reading This Line-by-Line" walkthrough.

## Spatial Orientation

Until now, deployment was trust-based: you pushed code and hoped it worked. Level 5 adds two safety nets: **CI/CD** catches bugs before they reach production, and **error monitoring** catches bugs that slip through.

```
DEVELOPMENT → PRODUCTION PIPELINE:

  Developer pushes code
       │
       ▼
  GitHub Actions triggers
       │
       ├── Install dependencies
       ├── Run frontend tests (Vitest + RTL)
       ├── Run backend tests (Vitest + Supertest + PostgreSQL)
       ├── Build frontend
       └── Build backend
            │
            ├── All pass → ✅ Green check → Vercel + Render auto-deploy
            │
            └── Any fail → ❌ Red X → Deployment blocked
                                      Developer fixes and pushes again

  PRODUCTION (after deploy):

  User encounters error
       │
       ▼
  Sentry captures:
   - Error message + stack trace
   - Browser, OS, URL
   - User context
       │
       ▼
  Sentry Dashboard → Developer notified → Fix deployed
```

---

## Part 1: GitHub Actions CI Pipeline

### What is CI/CD?

> [!NOTE]
> **Technical:** CI (Continuous Integration) automatically runs tests and builds on every push or pull request. CD (Continuous Deployment) automatically deploys code that passes CI. Together, they create an automated pipeline from code change to production. GitHub Actions is GitHub's built-in CI/CD service.
>
> **Plain English:** CI/CD is a robot assistant that checks your homework (tests) before submitting it (deploying). If you made a mistake, the robot catches it and sends it back. Only perfect homework gets submitted.

### Create the Workflow

> [!IMPORTANT]
> **You should be in:** `collab-board/`

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test-frontend:
    name: Frontend Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: client/package-lock.json

      - name: Install dependencies
        run: npm ci
        working-directory: client

      - name: Run tests
        run: npm test
        working-directory: client

      - name: Build
        run: npm run build
        working-directory: client

  test-backend:
    name: Backend Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: collabboard_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/collabboard_test
      JWT_SECRET: test-secret-for-ci
      JWT_EXPIRES_IN: 7d
      NODE_ENV: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: server/package-lock.json

      - name: Install dependencies
        run: npm ci
        working-directory: server

      - name: Run database schema
        run: psql "$DATABASE_URL" < src/db/schema.sql
        working-directory: server

      - name: Seed test data
        run: npx tsx src/db/seed.ts
        working-directory: server
        env:
          DATABASE_URL: ${{ env.DATABASE_URL }}

      - name: Run tests
        run: npm test
        working-directory: server

      - name: Build
        run: npm run build
        working-directory: server
```

### Reading This YAML Line-by-Line

YAML reads top-down with **indentation defining structure**. Each indent level means "this belongs to the parent above." Two-space indentation is the convention.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

- `name: CI` — top-level field. The string `'CI'` shows up in the GitHub Actions UI.
- `on:` — declares triggers. The two indented children (`push` and `pull_request`) are different events.
  - `push: branches: [main]` — runs the workflow when commits are pushed to main.
  - `pull_request: branches: [main]` — also runs on PRs targeting main.
- `[main]` is YAML shorthand for an array `["main"]`. Could be expanded as `branches:\n  - main` if you have multiple.

```yaml
jobs:
  test-frontend:
    name: Frontend Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```

- `jobs:` — declares the parallel jobs. Each indented key (`test-frontend`, `test-backend`) is a job ID.
- `runs-on: ubuntu-latest` — GitHub spins up a fresh Ubuntu virtual machine to execute this job.
- `steps:` — an array of steps. Each `- name: ...` is one step.
- `uses: actions/checkout@v4` — instead of writing shell commands, this step "uses" a prebuilt **Action**. `actions/checkout@v4` is GitHub's official action for cloning the repo. The `@v4` pins to a specific major version.

```yaml
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: client/package-lock.json
```

- `with:` — pass parameters to the action. The action's docs define what keys are valid.
- `node-version: 20` — install Node.js 20.
- `cache: 'npm'` — turn on npm dependency caching to speed up subsequent runs.
- `cache-dependency-path: client/package-lock.json` — fingerprint the cache by this lockfile so it's invalidated when dependencies change.

```yaml
      - name: Install dependencies
        run: npm ci
        working-directory: client
```

- `run: npm ci` — execute a shell command. `npm ci` is "clean install": deletes `node_modules` and installs exactly what `package-lock.json` specifies. Faster and more deterministic than `npm install`.
- `working-directory: client` — run the command from inside `client/`.

```yaml
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: collabboard_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
```

The `services:` block on the backend job. Each service starts a Docker container.

- `image: postgres:15` — pull the official PostgreSQL 15 Docker image.
- `env:` — environment variables passed into the container (Postgres reads `POSTGRES_USER`, etc., on startup).
- `options: >- ...` — `>-` is YAML's **folded scalar** syntax: the multiline string folds into a single line at runtime. Becomes one long `--health-cmd ...` argument.
- `--health-cmd pg_isready --health-interval 10s ...` — Docker health-check options. GitHub Actions waits until the container reports healthy before running steps. Without this, the test could try to connect before Postgres is ready.
- `ports: - 5432:5432` — expose the container's port 5432 (Postgres's default) on the VM. Tests connect to `localhost:5432` and reach the container.

```yaml
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/collabboard_test
      JWT_SECRET: test-secret-for-ci
      JWT_EXPIRES_IN: 7d
      NODE_ENV: test
```

Job-level environment variables. Available to every step. The `DATABASE_URL` here uses the credentials from the service container.

```yaml
      - name: Seed test data
        run: npx tsx src/db/seed.ts
        working-directory: server
        env:
          DATABASE_URL: ${{ env.DATABASE_URL }}
```

`${{ env.DATABASE_URL }}` is **Actions template syntax** — read the env var and inject its value here. Looks redundant (we already declared `env: DATABASE_URL` at the job level) but the explicit pass-through is sometimes needed for sub-shells. The double curly braces `{{ }}` distinguish Actions templates from raw text.

> [!NOTE]
> **Technical:** `npm ci` is like `npm install` but stricter — it deletes `node_modules` first and installs exactly what's in `package-lock.json`. This ensures CI uses the same dependency versions as development. `npm install` might update the lockfile.
>
> **Plain English:** `npm ci` is a clean install — it starts fresh every time. This ensures the CI environment matches what you tested locally, with no leftover files from previous runs.

### Two Jobs in Parallel

```
GitHub Actions
  │
  ├── test-frontend (ubuntu VM #1)
  │   ├── Install → Test → Build
  │   └── No database needed
  │
  └── test-backend (ubuntu VM #2)
      ├── Start PostgreSQL container
      ├── Install → Schema → Seed → Test → Build
      └── Needs database for Supertest
```

Frontend and backend jobs run **simultaneously** on separate VMs. This halves the total CI time compared to running them sequentially.

### Reading CI Results

After pushing, go to GitHub → your repository → **Actions** tab.

- **Green checkmark** ✅ — all tests passed, builds succeeded
- **Red X** ❌ — something failed. Click the job to see which step failed and read the error output
- **Yellow circle** 🟡 — still running

On pull requests, the CI status appears directly on the PR page. Many teams configure branch protection rules to **require** CI to pass before merging.

---

> [!TIP]
> **Session Break** — You've built the complete CI pipeline with parallel frontend and backend test jobs. Save your work and take a break.
> When you return, you'll set up Sentry error monitoring and add accessibility improvements.

---

## Part 2: Sentry Error Monitoring

### What Sentry Does

Sentry is a service that captures errors from your deployed application. When a user hits a bug in production, Sentry records:

- The exact error message and stack trace
- Which browser, OS, and device the user was on
- The URL and user action that triggered the error
- How many users are affected
- Whether this is a new error or a repeat

> [!NOTE]
> **Technical:** Sentry's React SDK wraps your app with an error boundary that reports uncaught exceptions. It also hooks into `window.onerror` and `unhandledrejection` to capture errors that React doesn't catch. Each error is sent to Sentry's API with context metadata, then grouped by stack trace signature.
>
> **Plain English:** Sentry is like a security camera for your app. When something breaks for a user, Sentry records what happened, where, and who was affected. Without Sentry, you only know about bugs when users email you — and most users just leave.

### Set Up Sentry

1. Go to [sentry.io](https://sentry.io) → Sign up (free tier is generous)
2. Create a new project → Select **React**
3. Copy the **DSN** (Data Source Name) — it looks like: `https://abc123@o456.ingest.sentry.io/789`

### Add Sentry DSN to Environment

Add to `collab-board/.env`:

```env
VITE_SENTRY_DSN=https://your-dsn-here@o123.ingest.sentry.io/456
```

> [!WARNING]
> **The Sentry DSN is safe to expose in frontend code** — it only allows sending error reports, not reading them. However, keeping it in an environment variable lets you use different projects for dev vs production.

### Initialize Sentry

Update `client/src/main.tsx`:

```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import * as Sentry from '@sentry/react';
import { Provider } from 'react-redux';
import { store } from './app/store';
import App from './App';
import './App.css';

// Initialize Sentry (only in production)
Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,     // "development" or "production"
  enabled: import.meta.env.PROD,          // Only send errors in production builds
  integrations: [
    Sentry.browserTracingIntegration(),
  ],
  tracesSampleRate: 0.1,                  // Sample 10% of transactions for performance
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Provider store={store}>
      <App />
    </Provider>
  </React.StrictMode>
);
```

### Reading the Sentry init Line-by-Line

```typescript
Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  enabled: import.meta.env.PROD,
  integrations: [
    Sentry.browserTracingIntegration(),
  ],
  tracesSampleRate: 0.1,
});
```

- `Sentry.init({...})` — the SDK setup call. Run it once, before any other code that might throw.
- `dsn: import.meta.env.VITE_SENTRY_DSN` — read the DSN from your env vars. `VITE_*` env vars are exposed to client code by Vite at build time.
- `environment: import.meta.env.MODE` — Vite sets this to `'development'` or `'production'` automatically. Sentry tags every error with this value so you can filter dev errors out of your production dashboard.
- `enabled: import.meta.env.PROD` — `import.meta.env.PROD` is `true` only in production builds. If `enabled` is false, Sentry's init runs but no errors are sent. Means you don't need separate dev DSNs.
- `integrations: [Sentry.browserTracingIntegration()]` — opt into performance monitoring. The browser-tracing integration measures page-load times, route changes, and slow API calls.
- `tracesSampleRate: 0.1` — sample 10% of page loads for performance traces. Sending 100% would generate too much data and exceed Sentry's free-tier quota; 10% gives a representative picture.

| Option | Value | Purpose |
|--------|-------|---------|
| `dsn` | Your Sentry DSN | Where to send error reports |
| `environment` | "development" / "production" | Filters errors by environment |
| `enabled` | `import.meta.env.PROD` | Only captures in production builds |
| `tracesSampleRate` | 0.1 | Samples 10% of page loads for performance monitoring |

### Use Sentry Error Boundary

Update `App.tsx` to wrap routes with Sentry's error boundary:

```typescript
import * as Sentry from '@sentry/react';

// In the JSX, wrap your Routes:
<Sentry.ErrorBoundary fallback={<div className="error-page">Something went wrong. Please refresh.</div>}>
  <Routes>
    {/* ... your routes ... */}
  </Routes>
</Sentry.ErrorBoundary>
```

Sentry's error boundary works like the custom one from Level 4, but it also **reports the error to Sentry** automatically.

### Set User Context

When a user logs in, tell Sentry who they are so errors include user information:

```typescript
// After successful login:
Sentry.setUser({ id: String(user.id), email: user.email });

// On logout:
Sentry.setUser(null);
```

This means when you see an error in Sentry, you can tell which user experienced it.

---

## Part 3: Accessibility Quick Wins

Accessibility isn't a separate feature — it's a quality of good engineering. These quick wins make CollabBoard usable by more people and demonstrate professionalism to employers.

### Semantic HTML

Use the right elements for the right purpose:

```typescript
// BAD — div with click handler
<div onClick={handleClick} className="button">Save</div>

// GOOD — actual button (focusable, keyboard-accessible, screen-reader-friendly)
<button onClick={handleClick}>Save</button>
```

| Element | Use for |
|---------|---------|
| `<button>` | Clickable actions |
| `<a>` | Navigation links |
| `<main>` | Primary page content |
| `<nav>` | Navigation sections |
| `<h1>`-`<h6>` | Headings in order |
| `<label>` | Form input labels (with `htmlFor`) |

### Aria Labels

For icon-only buttons, add `aria-label` so screen readers can announce the purpose:

```typescript
<button onClick={onClose} aria-label="Close modal">✕</button>
<button onClick={onDelete} aria-label="Delete card">🗑</button>
```

### Focus Management

When a modal opens, focus should move to the modal. When it closes, focus should return to the element that opened it:

```typescript
import { useEffect, useRef } from 'react';

function Modal({ isOpen, onClose, children }) {
  const closeButtonRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (isOpen) {
      closeButtonRef.current?.focus();
    }
  }, [isOpen]);

  // ... modal JSX with ref={closeButtonRef} on close button
}
```

### Reading the focus-management snippet

- **`useRef<HTMLButtonElement>(null)`** — React's hook for getting a stable reference to a DOM element. The generic `<HTMLButtonElement>` types it (button-specific element). Initial value `null` because the element doesn't exist until React renders.
- **`closeButtonRef.current?.focus()`** — `.current` is the actual DOM node (or `null` if not yet rendered). `?.focus()` uses optional chaining so we don't crash before render. `.focus()` is a built-in DOM method that moves keyboard focus to the element.
- **The effect's dependency `[isOpen]`** — runs on mount AND whenever `isOpen` flips. So opening the modal moves focus into it; closing has no effect (focus naturally returns to the previously-focused element).
- The JSX would attach this ref via `ref={closeButtonRef}`. Once mounted, `closeButtonRef.current` becomes the actual button.

### Keyboard Navigation

All interactive elements should be reachable via Tab. dnd-kit already provides keyboard support for drag-and-drop (added in Step 6). Ensure:

- Modal can be closed with Escape key
- Forms can be submitted with Enter
- All buttons are focusable (use `<button>`, not `<div>`)

---

## Step 5: Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
git add .
git commit -m "feat: add CI/CD pipeline and Sentry error monitoring"
git push
```

After pushing, check the **Actions** tab on GitHub. Your CI workflow should trigger and run both jobs.

---

> [!TIP]
> ## Spatial Check-In

1. **What happens if CI fails on a pull request?**

<details><summary>Answer</summary>

The PR gets a red X next to the CI check. If the repository has branch protection rules, the PR cannot be merged until CI passes. The developer clicks the failing check to see which job and step failed, reads the error output, fixes the issue locally, and pushes again. The CI re-runs automatically on the new push.

</details>

2. **What information does Sentry capture that `console.error` doesn't?**

<details><summary>Answer</summary>

`console.error` only prints to the browser console — if the user closes the tab, the error is gone forever. Sentry captures: the full stack trace with source maps (exact file and line), the user's browser/OS/device, the URL and navigation history, user context (who experienced it), error frequency (how many users are affected), and whether it's a new or recurring issue. All of this persists in a searchable dashboard.

</details>

3. **Why does the CI backend job need a PostgreSQL service container?**

<details><summary>Answer</summary>

Supertest API tests send real HTTP requests to the Express server, which executes real SQL queries. Without a database, the tests fail with connection errors. The `services: postgres` block starts a PostgreSQL container that the test environment connects to via `DATABASE_URL`. The schema and seed steps populate it with test data before tests run.

</details>

---

| | | |
|:---|:---:|---:|
| [← 06 — Drag & Drop](../06-drag-and-drop/) | [Level 5 Overview](../) | [08 — Deployment →](../08-deployment/) |
