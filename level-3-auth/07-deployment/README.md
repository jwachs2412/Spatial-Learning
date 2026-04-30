# Step 7 — Deployment: Shipping with Secrets

## Spatial Orientation

```
PRODUCTION
┌──────────────┐   ┌────────────────┐   ┌──────────────┐
│   Vercel     │   │ Render/Railway │   │  Supabase/   │
│   (Frontend) │──▶│   (Backend)    │──▶│  Neon/Render  │
│              │   │                │   │  (Database)  │
│              │   │  JWT_SECRET    │   │              │
│              │   │  (env var)     │   │  users table │
│              │   │                │   │  notes table │
└──────────────┘   └────────────────┘   └──────────────┘
```

Level 3 deployment adds a critical dimension: **secret management**. Your `JWT_SECRET` must be unique per environment and NEVER committed to code.

---

## 1. Verify Locally

Test the full flow locally:
1. Register two users
2. Create notes for each user
3. Verify cross-user isolation (user A can't see user B's notes)
4. Log out and log back in — token should persist
5. Verify `npm run build` succeeds in the server directory

---

## 2. Push to GitHub

```bash
git add .
git commit -m "chore: prepare for deployment"
gh repo create vault-note --public --source=. --remote=origin --push
```

---

## 3. Deploy the Database

### Provider Strategy Across the Curriculum

Each level uses a **different free Postgres provider** so your portfolio shows variety and you don't exhaust any single provider's free tier:

| Level | Provider | Why this provider for this level |
|-------|----------|----------------------------------|
| Level 2 — TaskForge | Supabase | Beginner-friendly dashboard |
| **Level 3 — VaultNote** | **Neon** ◀ this lesson | Serverless Postgres with branching — good fit for auth |
| Level 4 — DataDash | Render | Standard managed Postgres — your one allowed Render free DB |
| Level 5 — CollabBoard | CockroachDB Serverless | Distributed, Postgres-compatible |

> ⚠️ **Render only gives you ONE free PostgreSQL database per account, and it expires after 90 days.** We save that single slot for Level 4. For Level 3 (this lesson), use **Neon**.

### Deploy on Neon

1. Go to [neon.tech](https://neon.tech) and sign up
2. Click **Create Project**
3. Name: `vaultnote`. Pick the latest Postgres version. Choose a region close to you.
4. After provisioning, the dashboard shows a **Connection string** field. Copy the full URI — it looks like: `postgresql://your-user:your-password@ep-cool-name-12345.us-east-2.aws.neon.tech/neondb?sslmode=require`
5. Run the schema:

```bash
psql "YOUR_NEON_CONNECTION_STRING" < server/src/db/schema.sql
```

**About Neon's connection strings:** Neon requires SSL, so you'll see `?sslmode=require` at the end of the URI. The `pg` driver in your backend respects this automatically. Quote the URL when passing it to `psql` — the `?` is shell-special.

Verify:
```bash
psql "YOUR_NEON_CONNECTION_STRING" -c "\dt"
```

You should see `users` and `notes` tables.

> [!NOTE]
> **Why Neon for Level 3?** Neon offers **database branching** — you can spin up an isolated copy of your production database for testing auth flows or running migrations safely. We don't use branching in this lesson, but it's a feature worth seeing on your portfolio résumé.
>
> **If Neon is unavailable**, the same schema deploys identically to **Supabase** or **CockroachDB Serverless** — they all speak Postgres. Just don't use Render here unless you've decided to skip Level 4's database deployment.

---

## 4. Deploy the Backend

On Render (or Railway):

| Setting | Value |
|---------|-------|
| **Root Directory** | `server` |
| **Build Command** | `npm install && npm run build` |
| **Start Command** | `npm start` |

Environment variables:

| Variable | Value |
|----------|-------|
| `DATABASE_URL` | Your database connection string |
| `CORS_ORIGIN` | `https://vault-note.vercel.app` (fill after frontend deploy) |
| `JWT_SECRET` | Generate: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `JWT_EXPIRES_IN` | `7d` |
| `NODE_ENV` | `production` |

> ⚠️ **JWT_SECRET must be unique per environment.** Never reuse your development secret in production. Generate a new random string for production.

Verify: `https://your-api.onrender.com/api/health` should show `database: "connected"`.

---

## 5. Deploy the Frontend

On Vercel:

| Setting | Value |
|---------|-------|
| **Framework** | Vite |
| **Root Directory** | `client` |
| **Build Command** | `npm run build` |
| **Output Directory** | `dist` |

Environment variable:
- `VITE_API_URL` = `https://vault-note-api.onrender.com/api`

### React Router Fix

Create `client/vercel.json`:

```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

### Reading This Config

`vercel.json` is a JSON file Vercel reads when deploying. Just one section here.

- `"rewrites"` is an array of rules. Each rule maps an incoming URL pattern to an internal destination.
- `"source": "/(.*)"` — a pattern that matches any URL. The `(.*)` is a regex group that captures everything after the leading `/`.
- `"destination": "/index.html"` — for every match, internally serve `index.html` (the built React app). The browser's URL bar still shows the original URL — this is **server-side rewriting**, not redirecting.

> **Why this is needed:** React Router handles navigation client-side. If a user goes directly to `https://yourapp.vercel.app/notes` (or refreshes on that page), Vercel tries to find a `/notes` file — which doesn't exist. The rewrite sends all paths to `index.html`, letting React Router handle routing.

---

## 6. Update CORS

Set `CORS_ORIGIN` on your backend to your Vercel URL:

```
CORS_ORIGIN=https://vault-note.vercel.app
```

> ⚠️ **No trailing slash!** This is the #1 production deployment bug.

---

## 7. Verify Production

1. Register a new account
2. Log in
3. Create notes
4. Log out and log back in — notes should persist
5. Register a second account — verify cross-user isolation
6. Open DevTools → Application → Local Storage — verify token is stored

### Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| CORS error | Trailing slash in CORS_ORIGIN | Remove trailing slash |
| 401 on all requests | JWT_SECRET mismatch between local/prod | Ensure production JWT_SECRET is set |
| React Router 404 on refresh | Missing vercel.json | Add the rewrites configuration |
| `TS7016: Could not find declaration file` | @types in devDependencies | Move to dependencies |
| `Cannot find name 'process'` | Missing `"types": ["node"]` in tsconfig | Add it |
| Login works but notes fail | VITE_API_URL wrong or missing | Check the environment variable in Vercel |

---

## 8. Commit

```bash
git add .
git commit -m "feat: deploy VaultNote with secret management"
```

---

> **Session Break** — Deployed with authentication!
> When you return, you'll review your skills in [Step 8 — Growth Review](../08-growth/).

---

| | |
|:---|---:|
| [← Step 6: Frontend](../06-frontend/) | [Step 8 — Growth Review →](../08-growth/) |
