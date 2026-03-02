`Level 4` **Step 8 of 9** — Deployment

# 08 — Deployment: Shipping with Tests and Middleware

## Spatial Orientation

Same three-tier deployment as Levels 2-3. The new element: **run your test suite before deploying** to verify nothing is broken.

```
PRODUCTION ENVIRONMENT:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌──────────────────┐   ┌────────────────────────┐   ┌───────────┐ │
│  │  Vercel CDN       │   │   Render Web Service    │   │  Render   │ │
│  │  (Frontend)       │   │   (Backend)              │   │  Postgres │ │
│  │                  │   │                          │   │           │ │
│  │  VITE_API_URL    │   │  DATABASE_URL            │   │ analytics │ │
│  │                  │   │  CORS_ORIGIN             │   │ _events   │ │
│  │  React + Redux   │   │  NODE_ENV=production     │   │           │ │
│  │  Recharts        │   │  PORT                    │   │  ~5000    │ │
│  │                  │   │                          │   │  rows     │ │
│  │                  │   │  Middleware stack:        │   │           │ │
│  │                  │   │   - pino (JSON logs)     │   │  Indexed  │ │
│  │                  │   │   - rate limiter          │   │           │ │
│  │                  │   │   - CORS                  │   │           │ │
│  │                  │   │   - error handler         │   │           │ │
│  └──────────────────┘   └────────────────────────┘   └───────────┘ │
│                                                                      │
│  No auth — this is a public analytics dashboard.                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Run Tests Locally

Before deploying, verify everything works.

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
# Run frontend tests
cd client && npm test

# Run backend tests (requires database with seed data)
cd ../server && npm test
```

Both should pass. If any test fails, fix it before deploying — shipping broken code defeats the purpose of having tests.

```
DEPLOYMENT FLOW:

  Tests pass locally
       │
       ▼
  Push to GitHub
       │
       ├──▶ Vercel auto-deploys frontend
       │
       └──▶ Render auto-deploys backend
              │
              ▼
         Production database (already seeded)
```

---

## Step 2: Verify Locally

Start both servers and test the full flow:

**Terminal 1 — Backend:**

```bash
cd server && npm run dev
```

**Terminal 2 — Frontend:**

```bash
cd client && npm run dev
```

Open `http://localhost:5173` and verify:

1. Overview cards show data (page views, signups, revenue, avg session)
2. Line chart renders page views over time
3. Bar chart shows category breakdown
4. Pie chart shows device distribution
5. Click **30d** → all charts update to 30-day range
6. Select **Category: marketing** → charts filter to marketing events
7. Click **Clear Filters** → back to all data
8. Click **Events** in sidebar → paginated events table loads
9. Click **Next** → page 2 loads
10. Click **Overview** → back to charts

Stop both servers.

---

## Step 3: Push to GitHub

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git status
```

Commit any uncommitted changes:

```bash
git add .
git commit -m "chore: prepare for deployment"
```

Create the GitHub repo and push:

```bash
# Option A: GitHub CLI
gh repo create data-dash --public --source=. --remote=origin --push

# Option B: Manual
# 1. Create "data-dash" repo on github.com (no README)
# 2. Then:
git remote add origin git@github.com:yourusername/data-dash.git
git push -u origin main
```

---

## Step 4: Deploy the Database (Render PostgreSQL)

1. Go to [render.com](https://render.com) → **"New"** → **"PostgreSQL"**
2. Configure:
   - **Name**: `datadash-db`
   - **Database**: `datadash`
   - **Region**: Same as previous projects
   - **Plan**: Free

3. Click **"Create Database"**
4. Copy both the **Internal** and **External** Database URLs

### Run Schema on Production

```bash
psql "YOUR_EXTERNAL_DATABASE_URL" < server/src/db/schema.sql
```

### Seed Production Data

```bash
DATABASE_URL="YOUR_EXTERNAL_DATABASE_URL" npx tsx server/src/db/seed.ts
```

> [!WARNING]
> **Use the External URL** for the seed command — you're connecting from your local machine. The Internal URL only works between Render services. After seeding, the backend will use the Internal URL for fast, private connections.

Expected output: `Seeded ~5400 events across 90 days.`

---

## Step 5: Deploy the Backend (Render Web Service)

1. Render → **"New"** → **"Web Service"**
2. Connect your `data-dash` repository
3. Configure:
   - **Name**: `data-dash-api`
   - **Root Directory**: `server`
   - **Runtime**: Node
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `npm start`

4. **Add environment variables:**

| Key | Value |
|-----|-------|
| `DATABASE_URL` | (Internal Database URL from Render) |
| `CORS_ORIGIN` | `https://data-dash-xxxxx.vercel.app` (from Vercel, after step 6) |
| `NODE_ENV` | `production` |
| `PORT` | `3001` |

5. Click **"Create Web Service"**

### Verify the Backend

```
https://data-dash-api.onrender.com/api/health
```

Expected: `{"status":"ok","database":"connected",...}`

```
https://data-dash-api.onrender.com/api/analytics/overview
```

Expected: JSON with page views, signups, revenue, and avg session duration.

---

## Step 6: Deploy the Frontend (Vercel)

1. Go to [vercel.com](https://vercel.com) → **"Add New"** → **"Project"**
2. Import `data-dash` repository
3. Configure:
   - **Framework Preset**: Vite
   - **Root Directory**: `client`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`

4. **Add environment variable:**

| Key | Value |
|-----|-------|
| `VITE_API_URL` | `https://data-dash-api.onrender.com/api` |

5. Click **"Deploy"**

### Update Render CORS

Go back to Render → your web service → Environment:
- Set `CORS_ORIGIN` to your Vercel URL: `https://data-dash-xxxxx.vercel.app`
- Save → Render redeploys automatically

---

## Step 7: Verify Production

Visit your Vercel URL:

1. Overview cards load with data
2. Charts render (line, bar, pie)
3. Date range buttons filter data
4. Category and device dropdowns filter data
5. Events table loads with pagination
6. Switching between Overview and Events works
7. Test from a different device/browser

### Production Logging

Check your backend logs on Render → your web service → **Logs**. You should see structured JSON logs from pino:

```json
{"level":30,"time":1709312456789,"req":{"method":"GET","url":"/api/analytics/overview"},"res":{"statusCode":200},"responseTime":45}
```

### Rate Limiting

The rate limiter is active in production. If you hit the API too fast:

```json
{"error":"Too many requests. Please try again later."}
```

With a `429 Too Many Requests` status code.

---

## Production Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Charts don't render | VITE_API_URL wrong or CORS error | Check Vercel env var, verify CORS_ORIGIN on Render |
| "Cannot connect to database" | DATABASE_URL wrong | Verify Internal URL is set on Render |
| No data in charts | Production DB not seeded | Run the seed command with External URL |
| 429 errors on normal use | Rate limit too strict | Increase `max` in rateLimiter.ts, redeploy |
| 502 on backend | Server crash | Check Render logs — likely missing env var |
| Charts show "NaN" | Number parsing issue | Check that `Number()` casts are in service layer |
| Blank page | Build error | Check Vercel build logs for TypeScript errors |

---

## Step 8: Commit and Push

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
git add .
git commit -m "docs: add deployment configuration and live URLs"
git push
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why run tests before deploying?**

<details><summary>Answer</summary>

Tests verify that your code works correctly. Deploying without running tests means you might ship broken features to production. In professional environments, CI/CD pipelines run tests automatically and block deployment if any test fails. Running tests locally simulates this safeguard.

</details>

2. **Why does the seed command use the External URL but the backend uses the Internal URL?**

<details><summary>Answer</summary>

The seed command runs from your local machine — it needs to connect over the public internet, so it uses the External URL. The backend runs on Render's network — it connects to the database privately using the Internal URL, which is faster and doesn't count against bandwidth limits.

</details>

3. **What does `NODE_ENV=production` change in the backend?**

<details><summary>Answer</summary>

Three things: (1) pino logs as JSON instead of pretty-printed text, making logs machine-parseable by log aggregators. (2) The error handler sends generic "Internal server error" messages instead of detailed error messages, preventing information leakage. (3) The server doesn't start if imported by tests (the `if (NODE_ENV !== 'test')` guard).

</details>

---

| | | |
|:---|:---:|---:|
| [← 07 — Performance](../07-performance/) | [Level 4 Overview](../) | [09 — Growth Review →](../09-growth/) |
