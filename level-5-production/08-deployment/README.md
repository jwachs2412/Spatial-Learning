`Level 5` **Step 8 of 9** — Deployment

# 08 — Deployment: Shipping the Capstone

## Spatial Orientation

Same three-tier deployment as Levels 2-4, now with two additions: **CI/CD pipeline** verifying every push and **Sentry** monitoring production errors.

```
PRODUCTION ENVIRONMENT:

┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ┌──────────────┐                                                       │
│  │ GitHub Actions│                                                       │
│  │              │  Push → Tests pass?                                   │
│  │  - test-fe   │     ├── Yes → Vercel + Render auto-deploy            │
│  │  - test-be   │     └── No  → Blocked, fix and push again            │
│  └──────────────┘                                                       │
│                                                                          │
│  ┌──────────────────┐   ┌────────────────────────┐   ┌───────────┐     │
│  │  Vercel CDN       │   │  Render Web Service    │   │  Render   │     │
│  │  (Frontend)       │   │  (Backend)              │   │  Postgres │     │
│  │                  │   │                          │   │           │     │
│  │  VITE_API_URL    │   │  DATABASE_URL            │   │  6 tables │     │
│  │  VITE_SENTRY_DSN │   │  CORS_ORIGIN             │   │  users    │     │
│  │                  │   │  JWT_SECRET              │   │  boards   │     │
│  │                  │   │  JWT_EXPIRES_IN           │   │  members  │     │
│  │                  │   │  NODE_ENV=production     │   │  lists    │     │
│  └──────────────────┘   └────────────────────────┘   │  cards    │     │
│                                                        │  comments │     │
│  ┌──────────────┐                                    └───────────┘     │
│  │   Sentry     │  ◀── Frontend errors reported automatically          │
│  │  Dashboard   │                                                       │
│  └──────────────┘                                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Run Tests Locally

Before deploying, verify everything passes.

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
# Frontend tests
cd client && npm test

# Backend tests (requires database with seed data)
cd ../server && npm test
```

Both must pass. If any test fails, fix it before proceeding.

---

## Step 2: Verify Locally

Start both servers:

**Terminal 1:**

```bash
cd server && npm run dev
```

**Terminal 2:**

```bash
cd client && npm run dev
```

Open `http://localhost:5173` and test the full flow:

1. **Register** a new account → redirected to /boards
2. **Create a board** → see it in the board list
3. **Click the board** → see lists and cards (seed data)
4. **Drag a card** from one list to another → card moves instantly
5. **Click a card** → modal opens with description and comments
6. **Add a comment** → appears immediately
7. **Log out** → redirected to /login
8. **Log in** as a different seed user (sam@example.com / password123) → see different boards
9. **Refresh the page** → still logged in, data persists

Stop both servers.

---

## Step 3: Push to GitHub

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
git status
git add .
git commit -m "chore: prepare for deployment"
```

Create the GitHub repo and push:

```bash
# Option A: GitHub CLI
gh repo create collab-board --public --source=. --remote=origin --push

# Option B: Manual
# 1. Create "collab-board" repo on github.com (no README)
# 2. Then:
git remote add origin git@github.com:yourusername/collab-board.git
git push -u origin main
```

### Verify CI Pipeline

After pushing, go to your repository on GitHub → **Actions** tab. You should see the CI workflow running. It will:

1. Install frontend dependencies → run tests → build
2. Install backend dependencies → start PostgreSQL → run schema → seed → run tests → build

If both jobs pass, you'll see green checkmarks. If either fails, check the logs and fix the issue before deploying.

> [!WARNING]
> **The CI pipeline must pass before deploying.** This is the whole point of CI — catching bugs before they reach production. If tests fail on GitHub but pass locally, the most common cause is environment differences (database URL, missing env vars).

---

## Step 4: Deploy the Database

### Provider Strategy Across the Curriculum

Each level uses a **different free Postgres provider** so your portfolio shows variety and you don't exhaust any single provider's free tier:

| Level | Provider | Why this provider for this level |
|-------|----------|----------------------------------|
| Level 2 — TaskForge | Supabase | Beginner-friendly dashboard |
| Level 3 — VaultNote | Neon | Serverless Postgres with branching |
| Level 4 — DataDash | Render | Standard managed Postgres — your one allowed Render free DB |
| **Level 5 — CollabBoard** | **CockroachDB Serverless** ◀ this lesson | Distributed, Postgres-compatible — production-grade capstone feel |

> [!IMPORTANT]
> **Render's one free Postgres slot is spent on Level 4** (its 90-day expiration is fine for purely synthetic analytics data). For the capstone, we use **CockroachDB Serverless** — a distributed Postgres-compatible database with a free tier that doesn't expire, making it a better long-term portfolio anchor.

### Deploy on CockroachDB Serverless

1. Go to [cockroachlabs.cloud](https://cockroachlabs.cloud) and sign up
2. Click **Create Cluster** → choose **Serverless** plan (Free tier)
3. Name the cluster `collabboard-cluster`. Pick a region near you.
4. After provisioning (1–2 minutes), click **Connect** in the cluster dashboard
5. Choose **General connection string**. Create a SQL user when prompted (save the password!).
6. Copy the connection string. It looks like:
   ```
   postgresql://your-user:your-password@cluster-xyz-123.j77.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full
   ```
7. Run the schema and seed:

```bash
psql "YOUR_COCKROACH_CONNECTION_STRING" < server/src/db/schema.sql
DATABASE_URL="YOUR_COCKROACH_CONNECTION_STRING" npx tsx server/src/db/seed.ts
```

**Reading these two commands:**

- The first line pipes `schema.sql` into psql against the remote CockroachDB cluster. CockroachDB speaks the Postgres wire protocol, so your existing schema (`SERIAL PRIMARY KEY`, `REFERENCES ... ON DELETE CASCADE`, `UNIQUE(...)` composite constraints, etc.) runs unchanged. Same with `json_agg` and `json_build_object` — both supported.
- The second line uses the same `DATABASE_URL=... npx tsx ...` pattern from Level 4: set the env var for one command, run the seed script. The `pg` Node driver works against CockroachDB without changes.

**About CockroachDB's connection string:**

- **Port 26257** instead of Postgres's usual 5432. CockroachDB picks a different default to avoid clashing with vanilla Postgres on the same host.
- **`sslmode=verify-full`** is mandatory. CockroachDB Serverless rejects unencrypted connections.
- The `pg` driver in your backend handles SSL automatically when the URL includes `sslmode=...`.

Verify:
```bash
psql "YOUR_COCKROACH_CONNECTION_STRING" -c "\dt"
```

You should see all six tables (`users`, `boards`, `board_members`, `lists`, `cards`, `comments`).

Expected seed output: `3 users, 2 boards, 7 lists, 12 cards, 5 comments created.`

> [!NOTE]
> **If CockroachDB Serverless is unavailable for any reason**, the same schema and seed deploy identically to **Supabase** (your second free project) or **Neon** (your second free project). Just substitute the connection string. **Do not** use Render here — that slot is reserved for Level 4.

> [!WARNING]
> **CockroachDB note on `SERIAL`:** The schema uses `SERIAL PRIMARY KEY`. CockroachDB supports `SERIAL` but generates 64-bit random IDs (not the sequential 1, 2, 3 you see locally on Postgres). The seed script uses `RETURNING *` to capture each new ID, so it doesn't care about the values — but the IDs you see in the production DB will be large numbers. This is expected and doesn't affect the app.

---

## Step 5: Deploy the Backend (Render Web Service)

1. Render → **"New"** → **"Web Service"**
2. Connect your `collab-board` repository
3. Configure:
   - **Name**: `collab-board-api`
   - **Root Directory**: `server`
   - **Runtime**: Node
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `npm start`

4. **Add environment variables:**

| Key | Value |
|-----|-------|
| `DATABASE_URL` | Your CockroachDB Serverless connection string (the same one you used to seed) |
| `CORS_ORIGIN` | `https://collab-board-xxxxx.vercel.app` (after step 6) |
| `JWT_SECRET` | (generate: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`) |
| `JWT_EXPIRES_IN` | `7d` |
| `NODE_ENV` | `production` |

> [!WARNING]
> **Generate a unique production JWT_SECRET.** Do NOT reuse your development secret. This is the foundation of your auth system.

5. Click **"Create Web Service"**

### Verify the Backend

```
https://collab-board-api.onrender.com/api/health
```

Expected: `{"status":"ok","database":"connected",...}`

---

## Step 6: Deploy the Frontend (Vercel)

1. Go to [vercel.com](https://vercel.com) → **"Add New"** → **"Project"**
2. Import `collab-board` repository
3. Configure:
   - **Framework Preset**: Vite
   - **Root Directory**: `client`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`

4. **Add environment variables:**

| Key | Value |
|-----|-------|
| `VITE_API_URL` | `https://collab-board-api.onrender.com/api` |
| `VITE_SENTRY_DSN` | (your Sentry DSN from Step 7) |

5. Click **"Deploy"**

### React Router Fix

Create `client/vercel.json` (if not already created):

```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

```bash
git add client/vercel.json
git commit -m "fix: add Vercel rewrites for client-side routing"
git push
```

### Update Render CORS

Go back to Render → your web service → Environment:
- Set `CORS_ORIGIN` to your Vercel URL: `https://collab-board-xxxxx.vercel.app`
- Save → Render redeploys automatically

> [!WARNING]
> **No trailing slash in CORS_ORIGIN!** This is the #1 production deployment bug.
>
> ```
> ✗ CORS_ORIGIN=https://collab-board.vercel.app/    ← WRONG (trailing slash)
> ✓ CORS_ORIGIN=https://collab-board.vercel.app      ← CORRECT (no trailing slash)
> ```
>
> The browser sends `Origin: https://collab-board.vercel.app` (no slash). If your CORS_ORIGIN has a trailing slash, they don't match, and every request fails with `Access-Control-Allow-Origin` errors.

---

## Step 7: Verify Production

Visit your Vercel URL:

1. Register a new account
2. Log in as a seed user (alex@example.com / password123)
3. See boards, click one, see lists and cards
4. Drag a card between lists
5. Click a card, add a comment
6. Log out, log in as a different user
7. Verify board membership — users only see boards they belong to
8. Open a different browser — test that two users can access the same board
9. Check Sentry dashboard — any errors should appear there

### Production Checklist

| Verify | Expected |
|--------|----------|
| Health check | `{"status":"ok","database":"connected"}` |
| Register | Creates account, redirects to /boards |
| Login | Returns token, shows boards |
| Board view | Lists and cards render with seed data |
| Drag-and-drop | Cards move between lists, persist on refresh |
| Card detail | Modal opens, comments load |
| Cross-user | Different users see different boards |
| CI pipeline | Green checkmarks on GitHub Actions |
| Sentry | Dashboard accessible (may be empty if no errors) |

---

## Production Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| "No token provided" | Frontend not sending Authorization header | Check api.ts authHeaders() |
| CORS error | Trailing slash in CORS_ORIGIN | Remove the trailing slash from CORS_ORIGIN |
| `TS2580: Cannot find name 'process'` | Missing `"types": ["node"]` in tsconfig | Add `"types": ["node"]` to `compilerOptions` in `server/tsconfig.json` |
| `TS7016: Could not find declaration file` | @types in devDependencies | Move `@types/node`, `@types/express`, `@types/cors`, `@types/pg`, `@types/bcryptjs`, `@types/jsonwebtoken` to regular `dependencies` |
| Cards don't load on board | Backend JOINs failing | Check Render logs for SQL errors |
| Drag doesn't persist | moveCard API failing silently | Check network tab for 4xx/5xx responses |
| CI fails but local passes | Env difference | Check GitHub Actions logs, verify DATABASE_URL in workflow |
| Sentry not receiving errors | DSN wrong or not enabled | Verify VITE_SENTRY_DSN, check `enabled: import.meta.env.PROD` |
| 502 on backend | Server crash | Check Render logs — likely missing env var |
| Blank page after login | React Router issue | Verify vercel.json rewrite exists |

---

## Step 8: Final Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
git add .
git commit -m "docs: finalize deployment configuration"
git push
```

Watch the GitHub Actions tab — your CI pipeline should run and pass.

---

> [!TIP]
> ## Spatial Check-In

1. **What's different about this deployment compared to Level 2?**

<details><summary>Answer</summary>

Level 2 was push-and-pray: you pushed code and hoped it worked. Level 5 has a CI pipeline that runs tests automatically before deployment. It has auth with JWT secrets that must be generated uniquely for production. It has Sentry monitoring errors in real time. It has a 6-table database instead of 2. The deployment is fundamentally more professional — you have confidence that the code works (tests), awareness when it breaks (Sentry), and security (unique secrets).

</details>

2. **Why do we seed the production database?**

<details><summary>Answer</summary>

Without seed data, the app is empty — there's nothing to see or interact with. For a portfolio project, seed data demonstrates the app's features immediately: boards with lists, cards with assignees, comments with discussion. A reviewer can log in with a seed account and explore the full functionality without creating data from scratch.

</details>

3. **What happens when someone pushes a broken commit now?**

<details><summary>Answer</summary>

GitHub Actions runs the CI pipeline. If tests fail, the commit gets a red X. If this is a pull request, the PR is flagged as failing. The broken code is NOT deployed because the CI check blocks it. The developer sees which tests failed, fixes the issue, and pushes again. This is the CI safety net — broken code never reaches production.

</details>

---

| | | |
|:---|:---:|---:|
| [← 07 — CI/CD & Monitoring](../07-cicd-monitoring/) | [Level 5 Overview](../) | [09 — Growth Review →](../09-growth/) |
