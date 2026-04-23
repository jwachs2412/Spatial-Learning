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

> [!WARNING]
> **Render's free PostgreSQL has a 90-day expiration and only 1 free database per account.** If you already used it for Level 2, 3, or 4, choose Supabase or Neon instead.

### Option A: Neon (Recommended for Level 5)

1. Go to [neon.tech](https://neon.tech) → **Create Project**: `collabboard`
2. Copy the connection string
3. Run schema and seed:

```bash
psql "YOUR_NEON_CONNECTION_STRING" < server/src/db/schema.sql
DATABASE_URL="YOUR_NEON_CONNECTION_STRING" npx tsx server/src/db/seed.ts
```

### Option B: Supabase

1. Go to [supabase.com](https://supabase.com) → **New Project**
2. Name: `collabboard`, set a database password, choose a region
3. Go to **Project Settings** → **Database** → **Connection string** → **URI**
4. Run schema and seed:

```bash
psql "YOUR_SUPABASE_CONNECTION_STRING" < server/src/db/schema.sql
DATABASE_URL="YOUR_SUPABASE_CONNECTION_STRING" npx tsx server/src/db/seed.ts
```

### Option C: Render PostgreSQL

> Only use if you haven't used your free Render database for a previous level.

1. Go to [render.com](https://render.com) → **"New"** → **"PostgreSQL"**
2. Configure: Name: `collabboard-db`, Database: `collabboard`, Plan: Free
3. Copy both the **Internal** and **External** Database URLs
4. Run schema and seed using the **External** URL:

```bash
psql "YOUR_EXTERNAL_DATABASE_URL" < server/src/db/schema.sql
DATABASE_URL="YOUR_EXTERNAL_DATABASE_URL" npx tsx server/src/db/seed.ts
```

> [!WARNING]
> **Use the External URL** for the seed command — you're connecting from your local machine. The Internal URL only works between Render services.

### Portfolio Strategy

| Level | Database Provider | Reason |
|-------|------------------|--------|
| Level 2 — TaskForge | Supabase | 2 free projects |
| Level 3 — VaultNote | Neon | Free serverless PostgreSQL |
| Level 4 — DataDash | Supabase (2nd project) | Or Render if unused |
| Level 5 — CollabBoard | Neon (2nd project) | Or Render if unused |

Expected: 3 users, 2 boards, 7 lists, 12 cards, 5 comments created.

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
| `DATABASE_URL` | (Internal Database URL from Render) |
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
