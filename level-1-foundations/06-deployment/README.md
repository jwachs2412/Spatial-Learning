# Step 6 — Deployment: Shipping to the Internet

## Spatial Orientation

```
DEVELOPMENT (your computer)              PRODUCTION (the internet)
┌───────────┐  ┌───────────┐            ┌───────────┐  ┌───────────┐
│ Frontend  │  │ Backend   │            │ Frontend  │  │ Backend   │
│ :5173     │  │ :3001     │    ───▶    │ Vercel    │  │ Render    │
│ npm run   │  │ npm run   │            │ (CDN)     │  │ Railway   │
│ dev       │  │ dev       │            │           │  │ or Fly.io │
└───────────┘  └───────────┘            └───────────┘  └───────────┘
    localhost       localhost                yourapp.       yourapi.
                                          vercel.app     onrender.com
```

Moving from development to production changes three things:
1. **Where the code runs** — Your machine → cloud servers
2. **How users access it** — `localhost` → real URLs
3. **How frontend finds backend** — Hardcoded port → environment variable

---

## What Changes Between Development and Production

| Aspect | Development | Production |
|--------|------------|------------|
| Frontend URL | `http://localhost:5173` | `https://yourapp.vercel.app` |
| Backend URL | `http://localhost:3001` | `https://yourapi.onrender.com` |
| How frontend starts | `npm run dev` (Vite dev server) | Static files served by CDN |
| How backend starts | `npm run dev` (ts-node + nodemon) | `npm run build` then `npm start` |
| Backend code | TypeScript (interpreted) | JavaScript (compiled by `tsc`) |
| CORS origin | `http://localhost:5173` | `https://yourapp.vercel.app` |

> **Key Concept: Build Process**
> In development, tools like Vite and ts-node interpret your code on-the-fly. In production, you *build* the code first: TypeScript gets compiled to JavaScript, React gets bundled into optimized files. The build output is what actually runs in production.

---

## 1. Prepare the Backend for Production

Before deploying, verify the build works locally:

```bash
cd server
npm run build
```

This runs `tsc` and compiles your TypeScript to JavaScript in the `dist/` folder. If this fails, fix the TypeScript errors before deploying — the deployment platform will run the same command.

```bash
npm start
```

This runs `node dist/index.js` — the compiled version. Verify `http://localhost:3001/api/health` works.

### ✅ Checkpoint

- [ ] `npm run build` succeeds without errors
- [ ] `npm start` runs the server from compiled JavaScript
- [ ] Health check works at `http://localhost:3001/api/health`

Press `Ctrl+C` to stop the server.

---

## 2. Push to GitHub

```bash
cd ..  # Back to project root
git add .
git commit -m "chore: prepare for deployment"
```

Create a GitHub repository and push:

```bash
# Using GitHub CLI (if installed)
gh repo create dev-pulse --public --source=. --remote=origin --push

# Or manually: create the repo on github.com, then:
git remote add origin git@github.com:yourusername/dev-pulse.git
git branch -M main
git push -u origin main
```

---

## 3. Deploy the Backend

You have several options for backend hosting. Choose one:

### Option A: Render (Simple, Popular)

**Pros:** Easy setup, auto-deploys from GitHub, free tier available.
**Cons:** Free tier has cold starts (first request after inactivity takes 30-60 seconds). Only 1 free PostgreSQL database per account (relevant for Level 2+).

1. Go to [render.com](https://render.com) and sign up with GitHub
2. Click **New** → **Web Service**
3. Connect your GitHub repository
4. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `dev-pulse-api` |
| **Root Directory** | `server` |
| **Runtime** | Node |
| **Build Command** | `npm install && npm run build` |
| **Start Command** | `npm start` |

5. Add environment variable:
   - `CORS_ORIGIN` = (leave blank for now — you'll fill this after deploying the frontend)

6. Click **Create Web Service**
7. Wait for the build to complete (2-3 minutes)
8. Test: `https://your-service-name.onrender.com/api/health`

### Option B: Railway (No Cold Starts)

**Pros:** No cold starts on paid plan, easy environment variables, PostgreSQL included.
**Cons:** Free tier has limited hours per month ($5/month credit).

1. Go to [railway.app](https://railway.app) and sign up with GitHub
2. Click **New Project** → **Deploy from GitHub repo**
3. Select your repository
4. Railway auto-detects Node.js. Configure:

| Setting | Value |
|---------|-------|
| **Root Directory** | `server` |
| **Build Command** | `npm install && npm run build` |
| **Start Command** | `npm start` |

5. Add environment variable: `CORS_ORIGIN` = (fill after frontend deploy)
6. Deploy and note your URL

### Option C: Fly.io (Global Edge)

**Pros:** Global deployment, generous free tier, fast cold starts.
**Cons:** Requires CLI tool installation, slightly more complex setup.

1. Install the Fly CLI: `brew install flyctl` (macOS) or see [fly.io/docs](https://fly.io/docs/getting-started/installing-flyctl/)
2. Sign up: `fly auth signup`
3. From `server/`, run: `fly launch`
4. Follow the prompts to configure
5. Set environment variables: `fly secrets set CORS_ORIGIN=https://yourapp.vercel.app`
6. Deploy: `fly deploy`

---

## 4. Deploy the Frontend

### Vercel (Recommended)

1. Go to [vercel.com](https://vercel.com) and sign up with GitHub
2. Click **Add New** → **Project**
3. Import your repository
4. Configure:

| Setting | Value |
|---------|-------|
| **Framework Preset** | Vite |
| **Root Directory** | `client` |
| **Build Command** | `npm run build` |
| **Output Directory** | `dist` |

5. Add environment variable:
   - `VITE_API_URL` = `https://your-backend-url.onrender.com/api`

> ⚠️ **Important:** The URL must NOT have a trailing slash. Use `https://your-api.onrender.com/api` not `https://your-api.onrender.com/api/`.

6. Click **Deploy**

---

## 5. Update CORS on the Backend

Now that you have your frontend URL, go back to your backend hosting provider and set the `CORS_ORIGIN` environment variable:

```
CORS_ORIGIN=https://dev-pulse.vercel.app
```

> ⚠️ **Common Mistake: Trailing Slash in CORS Origin**
> This is the most common production deployment bug. The browser sends the origin **without** a trailing slash.
>
> Wrong: `https://dev-pulse.vercel.app/`
> Right: `https://dev-pulse.vercel.app`
>
> If you see a CORS error in production where the error message says the origin value "is not equal to the supplied origin," check for a trailing slash.

After updating the environment variable, your backend will automatically redeploy (on Render) or you may need to trigger a redeploy.

---

## 6. Verify Production

Open your Vercel URL and test:

1. **Submit a mood entry** — should succeed
2. **Entries should appear** in the list
3. **Open DevTools** → Network tab — check requests go to your backend URL
4. **No CORS errors** in the console
5. **Try on your phone** — open the Vercel URL on your phone's browser

### Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| CORS error | Trailing slash in origin, or CORS_ORIGIN not set | Remove trailing slash, verify env var on backend |
| "Failed to fetch" | Backend not running or wrong URL | Check backend logs, verify VITE_API_URL |
| Frontend shows blank page | Build error, or wrong root directory | Check Vercel build logs |
| Backend returns 503 | Server crashed during startup | Check backend logs for TypeScript compilation errors |
| `Cannot find module` on backend | Build failed, or start command wrong | Verify `npm run build` works locally, check start command |

### 🧠 Debugging Exercise

You deploy and see this error in the browser console:
```
Access to fetch at 'https://dev-pulse-api.onrender.com/api/entries' from origin 'https://dev-pulse.vercel.app' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header has a value 'https://dev-pulse.vercel.app/' that is not equal to the supplied origin.
```

What's the problem, and how do you fix it?

<details>
<summary>Answer</summary>

The error message tells you exactly what's wrong: the CORS header value (`https://dev-pulse.vercel.app/`) has a trailing slash, but the browser sent the origin without one (`https://dev-pulse.vercel.app`). They don't match, so CORS blocks the request.

Fix: Go to your backend hosting provider's environment variables and change `CORS_ORIGIN` from `https://dev-pulse.vercel.app/` to `https://dev-pulse.vercel.app` (remove the trailing slash). Redeploy.

</details>

---

## 7. Understanding What Just Happened

```
PRODUCTION ARCHITECTURE

User's Browser                    Cloud Services
┌────────────────┐               ┌──────────────────┐
│                │   HTTPS       │ Vercel CDN       │
│  React App     │◀──────────────│                  │
│  (JavaScript   │               │ Serves built     │
│   + HTML + CSS)│               │ static files     │
│                │               │ (index.html,     │
│                │               │  bundle.js, etc) │
│                │               └──────────────────┘
│                │
│                │   HTTPS       ┌──────────────────┐
│  fetch() ──────│──────────────▶│ Render / Railway │
│                │               │                  │
│                │◀──────────────│ Express server   │
│  JSON response │               │ (compiled JS)    │
│                │               │                  │
└────────────────┘               │ In-memory data   │
                                 └──────────────────┘
```

Key differences from development:
- **HTTPS** instead of HTTP (the hosting platforms handle SSL certificates)
- **CDN** serves the frontend (fast, globally distributed)
- **Compiled JavaScript** runs on the backend (not TypeScript directly)
- **Environment variables** configure the connection between services

---

## 8. Commit

```bash
git add .
git commit -m "feat: deploy DevPulse to Vercel and Render"
```

---

## 🧠 Spatial Check-In

1. Why does the backend need a `build` step (`tsc`) in production but not in development?

2. If you update your backend code and push to GitHub, what happens on Render? What about on Vercel for frontend changes?

3. Your app works locally but fails in production. The backend logs show no errors, but the frontend console shows "Failed to fetch." What's the most likely cause?

<details>
<summary>Check Your Answers</summary>

1. **In development**, `ts-node` interprets TypeScript on-the-fly — convenient but slow and not suitable for production. **In production**, `tsc` compiles TypeScript to JavaScript once, and Node.js runs the compiled JavaScript directly — faster and more reliable.

2. **Both auto-deploy.** Render watches your GitHub repo and rebuilds when you push. Vercel does the same. This is called continuous deployment.

3. **CORS misconfiguration or wrong API URL.** If the backend is running fine (no errors in its logs), the frontend is either sending requests to the wrong URL (`VITE_API_URL` not set or wrong) or the backend is rejecting the origin (`CORS_ORIGIN` not set, wrong, or has a trailing slash).

</details>

---

> **Session Break** — Your app is live on the internet!
> When you return, you'll review your skills in [Step 7 — Growth Review](../07-growth/).

---

| | |
|:---|---:|
| [← Step 5: Connecting](../05-connecting/) | [Step 7 — Growth Review →](../07-growth/) |
