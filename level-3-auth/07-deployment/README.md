`Level 3` **Step 7 of 8** — Deployment

# 07 — Deployment: Shipping with Secrets

## Spatial Orientation

Same three-tier deployment as Level 2, with one critical addition: **secrets management**. The `JWT_SECRET` must exist in production without ever touching version control.

```
PRODUCTION ENVIRONMENT:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌──────────────────┐   ┌────────────────────────┐   ┌───────────┐ │
│  │  Vercel CDN       │   │   Render Web Service    │   │  Render   │ │
│  │  (Frontend)       │   │   (Backend)              │   │  Postgres │ │
│  │                  │   │                          │   │           │ │
│  │  VITE_API_URL    │   │  DATABASE_URL            │   │  users    │ │
│  │                  │   │  CORS_ORIGIN             │   │  notes    │ │
│  │  No secrets here │   │  JWT_SECRET ← NEW        │   │           │ │
│  │  (public code)   │   │  JWT_EXPIRES_IN          │   │  Hashed   │ │
│  └──────────────────┘   │  NODE_ENV                │   │  passwords│ │
│                          └────────────────────────┘   └───────────┘ │
│                                                                      │
│  Secrets live ONLY in Render's environment variables.                │
│  Never in code. Never in Git. Never in the frontend.                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Verify Everything Works Locally

Run both servers and test the full authentication flow:

**Terminal 1 — Backend:**

```bash
cd server && npm run dev
```

**Terminal 2 — Frontend:**

```bash
cd client && npm run dev
```

Open `http://localhost:5173` and test:

1. Visit `/register` → create an account → redirected to `/notes`
2. Create a note → it appears in the sidebar
3. Click a note → edit it → save
4. Delete a note → it disappears
5. Click "Log Out" → redirected to `/login`
6. Log in with your credentials → see your notes again
7. **Refresh the page** → still logged in (token persists in localStorage)
8. **Restart the backend** → refresh → notes still there (database persistence)

### Cross-User Isolation Test

1. Register a second account (different email)
2. Create notes with the second account
3. Log out, log in as the first account
4. **First account should NOT see second account's notes**

Stop both servers.

---

## Step 2: Push to GitHub

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
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
gh repo create vault-note --public --source=. --remote=origin --push

# Option B: Manual
# 1. Create "vault-note" repo on github.com (no README)
# 2. Then:
git remote add origin git@github.com:yourusername/vault-note.git
git push -u origin main
```

---

## Step 3: Deploy the Database (Render PostgreSQL)

1. Go to [render.com](https://render.com) → **"New"** → **"PostgreSQL"**
2. Configure:
   - **Name**: `vaultnote-db`
   - **Database**: `vaultnote`
   - **Region**: Same as your Level 2 deployment
   - **Plan**: Free

3. Click **"Create Database"**
4. Copy both the **Internal** and **External** Database URLs

### Run Schema on Production

```bash
psql "YOUR_EXTERNAL_DATABASE_URL" < server/src/db/schema.sql
```

Expected: `DROP TABLE` / `CREATE TABLE` output.

> [!WARNING]
> **Only run this once in production** (or when you intentionally want to reset all data). The schema drops existing tables first.

---

## Step 4: Deploy the Backend (Render Web Service)

1. Render → **"New"** → **"Web Service"**
2. Connect your `vault-note` repository
3. Configure:
   - **Name**: `vault-note-api`
   - **Root Directory**: `server`
   - **Runtime**: Node
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `npm start`

4. **Add environment variables:**

| Key | Value |
|-----|-------|
| `DATABASE_URL` | (Internal Database URL from Render) |
| `CORS_ORIGIN` | `https://vault-note-xxxxx.vercel.app` (from Vercel, after step 5) |
| `JWT_SECRET` | (generate with `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`) |
| `JWT_EXPIRES_IN` | `7d` |
| `NODE_ENV` | `production` |

> [!WARNING]
> **Generate a unique JWT_SECRET for production.** Do NOT use the same value as your development `.env`. Run the crypto command to generate a random 64-character hex string. This secret is the foundation of your authentication system — if it leaks, anyone can forge tokens.

5. Click **"Create Web Service"**

### Verify the Backend

```
https://vault-note-api.onrender.com/api/health
```

Expected: `{"status":"ok","database":"connected",...}`

---

## Step 5: Deploy the Frontend (Vercel)

1. Go to [vercel.com](https://vercel.com) → **"Add New"** → **"Project"**
2. Import `vault-note` repository
3. Configure:
   - **Framework Preset**: Vite
   - **Root Directory**: `client`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`

4. **Add environment variable:**

| Key | Value |
|-----|-------|
| `VITE_API_URL` | `https://vault-note-api.onrender.com/api` |

5. Click **"Deploy"**

### Update Render CORS

Go back to Render → your web service → Environment:
- Set `CORS_ORIGIN` to your Vercel URL: `https://vault-note-xxxxx.vercel.app`
- Save → Render redeploys automatically

---

## Step 6: Verify Production

Visit your Vercel URL:

1. Register a new account
2. Create notes
3. Log out → log back in → notes are there
4. Refresh → still logged in
5. Register a second account → confirm isolation (can't see first account's notes)
6. Try from a different device → same experience

### Security Verification

Open browser DevTools → Application tab → Local Storage. You should see the JWT token stored there. This is expected — it's how the frontend persists the login state.

Open DevTools → Network tab → look at a request to `/api/notes`. In the request headers, you should see `Authorization: Bearer eyJ...`. This is how the token travels to the backend.

---

## Production Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| "No token provided" on protected routes | Frontend not sending Authorization header | Check `authHeaders()` in api.ts, verify VITE_API_URL |
| "Invalid or expired token" | JWT_SECRET mismatch or token expired | Verify JWT_SECRET is set on Render, try logging in again |
| CORS error | Origin mismatch | Verify CORS_ORIGIN on Render matches Vercel URL exactly |
| "Email already registered" | User exists in prod DB | Use a different email, or reset the production DB |
| Login works but notes don't load | VITE_API_URL wrong | Check Vercel env var includes `/api` |
| 502 on backend | Server crashed | Check Render logs — likely a missing env var |
| Blank page after login | React Router issue | Check Vercel redirect rules — add a `vercel.json` rewrite (see below) |

### React Router + Vercel Fix

If refreshing `/notes` gives a 404, create `client/vercel.json`:

```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

This tells Vercel to serve `index.html` for all routes, letting React Router handle the routing client-side.

```bash
git add client/vercel.json
git commit -m "fix: add Vercel rewrites for client-side routing"
git push
```

---

## Step 7: Commit and Push

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add .
git commit -m "docs: add deployment configuration and live URLs"
git push
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why must JWT_SECRET be different in production vs development?**

<details><summary>Answer</summary>

If the production secret is the same as the dev secret (which is in your .env file on your laptop), anyone with access to your computer can forge production tokens. A unique, random production secret limits the blast radius.

</details>

2. **Why do we need a Vercel rewrite rule for React Router?**

<details><summary>Answer</summary>

When a user visits `/notes` directly (or refreshes), Vercel looks for a file at that path. It doesn't exist — it's a client-side route. The rewrite tells Vercel to serve `index.html` for all paths, letting React Router determine what to render.

</details>

3. **What environment variables does the backend need in production?**

<details><summary>Answer</summary>

DATABASE_URL (Internal), CORS_ORIGIN (Vercel URL), JWT_SECRET (random hex string), JWT_EXPIRES_IN, NODE_ENV=production.

</details>

---

| | | |
|:---|:---:|---:|
| [← 06 — Building the Frontend](../06-frontend/) | [Level 3 Overview](../) | [08 — Growth Review →](../08-growth/) |
