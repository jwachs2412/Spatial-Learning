# Step 6 — Deployment: Shipping a Three-Tier App

## Spatial Orientation

```
DEVELOPMENT                              PRODUCTION
┌────────┐ ┌────────┐ ┌────────┐        ┌────────┐ ┌────────┐ ┌──────────┐
│Frontend│ │Backend │ │Database│  ───▶  │Vercel  │ │Render/ │ │ Supabase/│
│ :5173  │ │ :3001  │ │ :5432  │        │(CDN)   │ │Railway │ │ Neon/    │
│        │ │        │ │        │        │        │ │        │ │ Render   │
└────────┘ └────────┘ └────────┘        └────────┘ └────────┘ └──────────┘
  localhost   localhost   localhost         .vercel    .onrender   managed
                                           .app       .com        PostgreSQL
```

This is your first three-tier deployment: frontend, backend, AND database — all on separate services.

---

## 1. Verify Locally

Before deploying, make sure everything works:

```bash
# Terminal 1
cd server && npm run dev

# Terminal 2
cd client && npm run dev
```

Create a project, add tasks, toggle them, delete one, refresh the page — data should persist.

Also verify the build works:

```bash
cd server && npm run build
```

If this fails, fix the TypeScript errors before deploying.

---

## 2. Push to GitHub

```bash
git add .
git commit -m "chore: prepare for deployment"

# Create repo and push
gh repo create task-forge --public --source=. --remote=origin --push

# Or manually: create on github.com, then:
git remote add origin git@github.com:yourusername/task-forge.git
git branch -M main
git push -u origin main
```

---

## 3. Deploy the Database

You have several options. Choose based on your needs:

### Option A: Supabase (Recommended)

> **Why Supabase?** You get 2 free PostgreSQL databases (500MB each). If you plan to build all 5 projects in this curriculum, Supabase gives you the most room. Plus, it has a beautiful dashboard for viewing your data.

1. Go to [supabase.com](https://supabase.com) and sign up
2. Click **New Project**
3. Name it `taskforge`, choose a strong database password, select a region close to you
4. Wait for the project to provision (1-2 minutes)
5. Go to **Project Settings** → **Database** → **Connection string** → **URI**
6. Copy the connection string — it looks like: `postgresql://postgres.[ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres`
7. Run the schema:

```bash
psql "YOUR_SUPABASE_CONNECTION_STRING" < server/src/db/schema.sql
```

Verify:
```bash
psql "YOUR_SUPABASE_CONNECTION_STRING" -c "\dt"
```

You should see `projects` and `tasks` tables.

### Option B: Neon

1. Go to [neon.tech](https://neon.tech) and sign up
2. Create a project named `taskforge`
3. Copy the connection string from the dashboard
4. Run the schema: `psql "YOUR_NEON_CONNECTION_STRING" < server/src/db/schema.sql`

### Option C: Render PostgreSQL

> ⚠️ **Limitation:** Render only allows **1 free PostgreSQL database** per account, and it **expires after 90 days**. If you plan to do all 5 levels, you'll run out. Consider Supabase or Neon instead.

1. Go to [render.com](https://render.com) → **New** → **PostgreSQL**
2. Name: `taskforge-db`
3. Database: `taskforge`
4. Select the free plan
5. Note the **Internal Database URL** (for backend) and **External Database URL** (for running schema)
6. Run the schema: `psql "EXTERNAL_URL" < server/src/db/schema.sql`

---

## 4. Deploy the Backend

### On Render

1. **New** → **Web Service** → connect your GitHub repo
2. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `task-forge-api` |
| **Root Directory** | `server` |
| **Runtime** | Node |
| **Build Command** | `npm install && npm run build` |
| **Start Command** | `npm start` |

3. Add environment variables:

| Variable | Value |
|----------|-------|
| `DATABASE_URL` | Your database connection string (Internal URL if Render DB, full URL if Supabase/Neon) |
| `CORS_ORIGIN` | `https://task-forge.vercel.app` (fill after frontend deploy) |
| `NODE_ENV` | `production` |

4. Deploy and verify: `https://task-forge-api.onrender.com/api/health`

Expected: `{"status":"ok","database":"connected",...}`

### On Railway (Alternative)

1. New Project → Deploy from GitHub
2. Root Directory: `server`
3. Build: `npm install && npm run build`
4. Start: `npm start`
5. Set the same environment variables

---

## 5. Deploy the Frontend

### On Vercel

1. Import your repo
2. Configure:

| Setting | Value |
|---------|-------|
| **Framework** | Vite |
| **Root Directory** | `client` |
| **Build Command** | `npm run build` |
| **Output Directory** | `dist` |

3. Add environment variable:
   - `VITE_API_URL` = `https://task-forge-api.onrender.com/api`

> ⚠️ **No trailing slash!** Use `.../api` not `.../api/`

4. Deploy

---

## 6. Update CORS

After getting your Vercel URL, go back to your backend hosting and set:

```
CORS_ORIGIN=https://task-forge.vercel.app
```

> ⚠️ **Common Mistake: Trailing Slash in CORS Origin**
> Wrong: `https://task-forge.vercel.app/`
> Right: `https://task-forge.vercel.app`
>
> The error message will say: `The 'Access-Control-Allow-Origin' header has a value '...' that is not equal to the supplied origin`. This means there's a trailing slash mismatch.

---

## 7. Verify Production

Open your Vercel URL and test:

1. Create a project
2. Add tasks
3. Toggle task completion
4. Edit a task (double-click)
5. Delete a task
6. Refresh the page — **data should persist!**
7. Delete the project — tasks should cascade delete

### Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| CORS error | Trailing slash in CORS_ORIGIN, or not set | Remove trailing slash, check env var |
| "Failed to fetch" | Wrong VITE_API_URL, or backend down | Check URL, check backend logs |
| Backend 503 | TypeScript build failed | Check build logs for TS errors |
| `TS7016: Could not find declaration file` | @types in devDependencies | Move to dependencies: `npm uninstall @types/... && npm install @types/...` |
| `Cannot find name 'process'` | Missing `"types": ["node"]` in tsconfig | Add it to `compilerOptions` |
| Database "disconnected" | Wrong DATABASE_URL | Check connection string, check DB is running |
| Data not persisting | Using wrong database or in-memory fallback | Verify DATABASE_URL points to production DB |

---

## 8. Deployment Architecture

```
User's Browser
      │
      │ HTTPS
      ▼
┌──────────────┐
│    Vercel     │  Serves static files (HTML, JS, CSS)
│    (CDN)      │  Built from client/ with Vite
└──────┬───────┘
       │
       │ HTTPS (fetch API calls)
       ▼
┌──────────────┐
│   Render /    │  Runs Express server (compiled JS)
│   Railway     │  Handles API requests, validation
└──────┬───────┘
       │
       │ PostgreSQL protocol (encrypted)
       ▼
┌──────────────┐
│  Supabase /   │  Stores data permanently
│  Neon /       │  Runs SQL queries
│  Render DB    │
└──────────────┘
```

---

## 9. Commit

```bash
git add .
git commit -m "feat: deploy TaskForge to production"
```

---

## 🧠 Spatial Check-In

1. Why are the database, backend, and frontend deployed as three separate services instead of one?

2. If you see `Cannot find name 'console'` in the Render build logs, what two things should you check?

3. Why do we set `CORS_ORIGIN` as an environment variable instead of hardcoding it?

<details>
<summary>Check Your Answers</summary>

1. **Separation of concerns and scalability.** Each tier has different needs: the frontend needs a CDN for fast global delivery, the backend needs a Node.js runtime, and the database needs persistent storage and indexing. Separate services also mean you can scale them independently (e.g., add more backend instances without changing the database).

2. **(a)** Check that `tsconfig.json` has `"types": ["node"]`. **(b)** Check that `@types/node` is in `dependencies`, not `devDependencies`.

3. **Different environments need different values.** In development, the origin is `http://localhost:5173`. In production, it's `https://task-forge.vercel.app`. Environment variables let you change this without changing code.

</details>

---

> **Session Break** — Three-tier deployment complete!
> When you return, you'll review your skills in [Step 7 — Growth Review](../07-growth/).

---

| | |
|:---|---:|
| [← Step 5: Frontend](../05-frontend/) | [Step 7 — Growth Review →](../07-growth/) |
