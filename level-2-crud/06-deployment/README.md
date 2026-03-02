`Level 2` **Step 6 of 7** — Deployment

# 06 — Deployment: Shipping a Three-Tier App

## Spatial Orientation

You're about to deploy three things instead of two:

```
BEFORE (localhost)                      AFTER (internet)
──────────────────                      ────────────────────

Your Computer                           The Internet
┌────────────────────┐                 ┌──────────────────────────────────┐
│ localhost:5173      │                 │  task-forge.vercel.app           │
│ localhost:3001      │                 │  task-forge-api.onrender.com     │
│ localhost:5432      │                 │  Render PostgreSQL (managed)     │
│                    │                 │                                  │
│ Only YOU can access │                 │ ANYONE can access                │
│ 3 things running   │                 │ 3 things running 24/7            │
└────────────────────┘                 └──────────────────────────────────┘
```

**What's new vs Level 1 deployment**: You now need a managed PostgreSQL database in the cloud. Render provides this for free (limited tier). The database gets its own connection URL that your backend uses instead of `localhost:5432`.

---

## Step 1: Verify Everything Works Locally

Before deploying, make sure the full stack works on your machine. You need two terminals.

**Terminal 1 — Backend:**

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
cd server
npm run dev
```

Expected: `Connected to PostgreSQL` and `Server running on http://localhost:3001`

**Terminal 2 — Frontend:**

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
cd client
npm run dev
```

Open `http://localhost:5173` in your browser. Test the full cycle:

1. Create a project → it appears in the list
2. Click the project → see the detail view
3. Add tasks → they appear in the task list
4. Toggle a task's checkbox → it marks as completed
5. Double-click a task title → edit it → press Enter
6. Delete a task → it disappears
7. Click "Back to Projects" → see all projects
8. Delete a project → it and its tasks disappear

### The Key Test: Data Persistence

Stop the backend (`Ctrl+C` in Terminal 1). Start it again (`npm run dev`).

Refresh the browser. **All your data should still be there.**

This is the "aha" moment. In Level 1, restarting the server lost everything. Now data survives because it's in PostgreSQL, not memory.

Stop both servers when you're done testing.

---

## Step 2: Push to GitHub

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

First, make sure all work is committed:

```bash
git status
```

If you see uncommitted changes, add and commit them:

```bash
git add .
git commit -m "chore: prepare for deployment"
```

Create the GitHub repository and push:

```bash
# Option A: Using GitHub CLI
gh repo create task-forge --public --source=. --remote=origin --push

# Option B: Manual
# 1. Go to github.com and create a new repository called "task-forge"
# 2. Do NOT initialize with a README (you already have one)
# 3. Then run:
git remote add origin git@github.com:yourusername/task-forge.git
git push -u origin main
```

---

## Step 3: Deploy the Database (Render PostgreSQL)

This is the new step that Level 1 didn't have. You need a PostgreSQL database running in the cloud.

### Create a Render PostgreSQL Instance

1. Go to [render.com](https://render.com) and log in (same account as Level 1)
2. Click **"New"** → **"PostgreSQL"**
3. Configure:
   - **Name**: `taskforge-db`
   - **Database**: `taskforge`
   - **User**: Leave as default
   - **Region**: Choose the same region you used for Level 1's backend
   - **Plan**: Free

4. Click **"Create Database"**

Render will provision the database. This takes a minute or two. When it's ready, you'll see two important URLs:

- **Internal Database URL** — Used by your Render web service (faster, same network)
- **External Database URL** — Used from your local machine or other services

**Copy both URLs and save them somewhere.** You'll need them.

### Run the Schema on the Production Database

You need to create the tables in the production database. Use the **External** URL:

```bash
psql "your-external-database-url" < server/src/db/schema.sql
```

Replace `your-external-database-url` with the External Database URL from Render. It looks something like:

```
postgresql://user:password@host.oregon-postgres.render.com/taskforge
```

> [!WARNING]
> **The External Database URL contains your password.** Do not commit it to Git, share it in Slack, or paste it in browser URL bars. Treat it like a secret.

You should see:

```
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
```

Your production database now has the `projects` and `tasks` tables.

---

## Step 4: Deploy the Backend (Render Web Service)

1. In Render, click **"New"** → **"Web Service"**
2. Connect your `task-forge` GitHub repository
3. Configure:
   - **Name**: `task-forge-api`
   - **Root Directory**: `server`
   - **Runtime**: Node
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `npm start`

4. **Add environment variables:**

| Key | Value |
|-----|-------|
| `DATABASE_URL` | (paste the **Internal** Database URL from Render) |
| `CORS_ORIGIN` | `https://task-forge-xxxxx.vercel.app` (you'll get this from Vercel next) |
| `NODE_ENV` | `production` |

> [!NOTE]
> Use the **Internal** Database URL here (not External). Internal URLs are faster because the web service and database are on the same network inside Render.

5. Click **"Create Web Service"**

Render will build and deploy your backend. Watch the logs for:

```
Connected to PostgreSQL
Server running on http://localhost:10000
```

(Render assigns its own PORT automatically.)

### Verify the Backend

Once deployed, visit your backend health check:

```
https://task-forge-api.onrender.com/api/health
```

Expected: `{"status":"ok","database":"connected",...}`

If `database` shows `"disconnected"`, check your `DATABASE_URL` environment variable on Render.

---

## Step 5: Deploy the Frontend (Vercel)

Same process as Level 1:

1. Go to [vercel.com](https://vercel.com) and log in
2. Click **"Add New"** → **"Project"**
3. Import your `task-forge` repository
4. Configure:
   - **Framework Preset**: Vite
   - **Root Directory**: `client`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`

5. **Add environment variables:**

| Key | Value |
|-----|-------|
| `VITE_API_URL` | `https://task-forge-api.onrender.com/api` |

6. Click **"Deploy"**

### Update Render CORS

Now that you have the Vercel URL, go back to Render:

1. Navigate to your `task-forge-api` web service
2. Go to **Environment** → edit `CORS_ORIGIN`
3. Set it to your Vercel URL: `https://task-forge-xxxxx.vercel.app`
4. Click **"Save Changes"**

Render will automatically redeploy with the new environment variable.

---

## Step 6: Verify Production

Visit your Vercel URL in a browser. Test the full cycle:

1. Create a project → appears in the list
2. Click into the project → add tasks
3. Toggle tasks, edit titles, delete tasks
4. **Refresh the page** → data is still there (database persistence)
5. **Close the browser, open a new one, go to the URL** → data is still there
6. **Try from a different device (phone)** → same data (it's in the cloud)

This is the full three-tier application running in production.

---

## Deployed Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                  DEPLOYED ARCHITECTURE                                 │
│                                                                      │
│  User's Browser                                                       │
│  ┌──────────────────────────┐                                        │
│  │  URL: task-forge.vercel.app                                       │
│  └──────────────┬───────────┘                                        │
│                 │                                                     │
│                 │ HTTPS                                                │
│                 ▼                                                     │
│  ┌──────────────────────────┐    Serves static files (HTML, JS, CSS) │
│  │      Vercel CDN          │    from client/dist/                     │
│  │   (Frontend Host)         │    No server code runs here             │
│  └──────────────────────────┘                                        │
│                 │                                                     │
│                 │ fetch() calls to task-forge-api.onrender.com/api    │
│                 │                                                     │
│                 │ HTTPS                                                │
│                 ▼                                                     │
│  ┌──────────────────────────┐    Runs Node.js + Express              │
│  │   Render Web Service     │    Processes API requests               │
│  │   (Backend Host)          │    Sends SQL queries                    │
│  └──────────────┬───────────┘                                        │
│                 │                                                     │
│                 │ Internal Database URL (same network)                │
│                 │                                                     │
│                 │ SQL                                                  │
│                 ▼                                                     │
│  ┌──────────────────────────┐    Stores data permanently on disk     │
│  │  Render PostgreSQL       │    Runs 24/7                            │
│  │   (Database Host)         │    Managed by Render (backups, etc.)   │
│  └──────────────────────────┘                                        │
│                                                                      │
│  All three auto-deploy when you push to GitHub.                      │
│  (Database persists independently — pushing code doesn't erase data) │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Production Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Frontend blank page | Build failed | Check Vercel build logs |
| Backend 502 error | Backend crashed | Check Render logs for errors |
| CORS error in browser | Origin mismatch | Verify `CORS_ORIGIN` on Render matches Vercel URL exactly |
| `database: "disconnected"` in health check | Wrong DATABASE_URL | Verify DATABASE_URL on Render (use Internal URL) |
| Data missing after deploy | You re-ran schema.sql on production | Schema drops tables first — only run it once (or when you want to reset) |
| API returns 404 | Wrong VITE_API_URL | Check Vercel env var includes `/api` at the end |
| "relation does not exist" | Forgot to run schema on production DB | Run `psql "external-url" < server/src/db/schema.sql` |
| Tasks not showing | Frontend fetching from wrong URL | Check Network tab in DevTools for the request URL |
| Render free tier slow first load | Cold start (free tier spins down) | Wait ~30 seconds, refresh — subsequent loads are fast |

---

## Step 7: Update README and Push

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

Update `task-forge/README.md` with your live URLs (replace the placeholder URLs with your actual deployment URLs).

```bash
git add .
git commit -m "docs: add deployment configuration and live URLs"
git push
```

Both Vercel and Render will automatically redeploy from this push.

---

## What's Different From Level 1 Deployment

| | Level 1 | Level 2 |
|---|---------|---------|
| **Tiers deployed** | 2 (frontend + backend) | 3 (frontend + backend + database) |
| **Data on restart** | Lost | Persists |
| **Database** | None | Render PostgreSQL |
| **Env vars** | PORT, CORS_ORIGIN | PORT, CORS_ORIGIN, DATABASE_URL |
| **Schema setup** | None | Must run schema.sql on production DB |
| **Backend health** | `{ status: 'ok' }` | `{ status: 'ok', database: 'connected' }` |

---

> [!TIP]
> ## Spatial Check-In

1. **Why do we use the Internal Database URL on Render instead of the External one?**

<details><summary>Answer</summary>

The Internal URL routes traffic through Render's private network, which is faster and more secure. The External URL goes over the public internet and is only needed when connecting from outside Render (like your local machine).

</details>

2. **What happens to data when you push new code to GitHub?**

<details><summary>Answer</summary>

Vercel rebuilds the frontend and Render rebuilds the backend, but the database is unaffected. Data persists across deployments because the database is a separate, independent service.

</details>

3. **Why is it dangerous to re-run schema.sql on the production database?**

<details><summary>Answer</summary>

The schema file starts with `DROP TABLE IF EXISTS` — it deletes all existing data before recreating the tables. In production, this would destroy all user data.

</details>

---

| | | |
|:---|:---:|---:|
| [← 05 — Building the Frontend](../05-frontend/) | [Level 2 Overview](../) | [07 — Growth Review →](../07-growth/) |
