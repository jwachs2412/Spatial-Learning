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

### Option A: Supabase (Recommended)

> ⚠️ **If you used Supabase for Level 2**, you have 1 free project remaining. If you used Render for Level 2, you can use Supabase here.

1. Go to [supabase.com](https://supabase.com) → New Project
2. Name: `vaultnote`, set a database password, choose a region
3. Go to **Project Settings** → **Database** → **Connection string** → **URI**
4. Run schema: `psql "YOUR_CONNECTION_STRING" < server/src/db/schema.sql`

### Option B: Neon

1. Go to [neon.tech](https://neon.tech) → Create Project: `vaultnote`
2. Copy the connection string
3. Run schema: `psql "YOUR_CONNECTION_STRING" < server/src/db/schema.sql`

### Option C: Render PostgreSQL

> ⚠️ **Only 1 free database per account, 90-day expiration.** If you already used it for Level 2, choose Supabase or Neon.

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
