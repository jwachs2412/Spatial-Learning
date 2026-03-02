`Level 1` **Step 6 of 7** — Deployment

# 06 — Deployment: Shipping to the Internet

## Spatial Orientation

Until now, your app runs on `localhost` — your own computer. Nobody else can see it. Deployment means putting it on computers that are connected to the internet 24/7, so anyone with the URL can access it.

```
BEFORE DEPLOYMENT (localhost)               AFTER DEPLOYMENT (internet)
──────────────────────────────              ───────────────────────────────

Your Computer                               The Internet
┌─────────────────────┐                    ┌──────────────────────────────┐
│ localhost:5173       │                    │  your-app.vercel.app         │
│ localhost:3001       │                    │  your-api.onrender.com       │
│                     │                    │                              │
│ Only YOU can access  │                    │ ANYONE can access            │
│ Dies when you close  │                    │ Runs 24/7                    │
│ the terminal         │                    │ Auto-deploys from GitHub     │
└─────────────────────┘                    └──────────────────────────────┘
```

---

## What "Build" Means

Before deploying, we need to **build** the app. Here's what that means for each part:

### Frontend Build

```
SOURCE CODE (what you write)              BUILD OUTPUT (what the browser gets)
────────────────────────────              ─────────────────────────────────────
src/                                      dist/
├── App.tsx        (TypeScript)           ├── index.html
├── App.css        (CSS)                  ├── assets/
├── components/    (multiple files)       │   ├── index-a1b2c3d4.js  (1 file)
│   ├── EntryForm.tsx                     │   └── index-e5f6g7h8.css (1 file)
│   ├── EntryList.tsx
│   └── MoodChart.tsx
├── services/api.ts
└── types/index.ts
```

> [!NOTE]
> **Technical**: The build process compiles TypeScript to JavaScript, bundles all modules into minimal files, minifies the code (removes whitespace, shortens variable names), tree-shakes unused code, and generates content-hashed filenames for cache optimization.

> [!NOTE]
> **Plain English**: "Building" takes your many source files and squishes them into a few tiny, optimized files that load fast in a browser. The browser doesn't understand TypeScript or JSX — the build converts them to plain JavaScript. The result is a `dist/` folder containing static files (HTML, JS, CSS) that any web server can serve.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/client/`

```bash
npm run build
```

This creates the `dist/` folder inside `client/`. That folder is what gets deployed.

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (back to the project root)

### Backend Build

```
SOURCE CODE                               BUILD OUTPUT
──────────                                ────────────
src/
├── index.ts       (TypeScript)           dist/
├── routes/                               ├── index.js      (JavaScript)
│   └── entries.ts (TypeScript)           ├── routes/
└── types/                                │   └── entries.js (JavaScript)
    └── index.ts   (TypeScript)           └── types/
                                              └── index.js   (JavaScript)
```

> [!NOTE]
> **Plain English**: Node.js can't run TypeScript directly in production (it can in development via `ts-node`). The build compiles TypeScript to JavaScript. The output is plain `.js` files that Node.js runs natively.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/server/`

```bash
npm run build
```

This creates the `dist/` folder inside `server/`.

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (back to the project root)

---

## Dev vs Production Environments

**Development** (your computer):
- Code changes reload instantly (hot module replacement)
- Errors show detailed stack traces
- Debug tools are available
- Uses `localhost` URLs
- TypeScript runs via `ts-node` (no build step needed)

**Production** (cloud server):
- Code is built and optimized
- Errors should be logged, not shown to users
- Performance is prioritized
- Uses public URLs (HTTPS)
- JavaScript runs directly (compiled from TypeScript)

### Environment Variables Differ

```
DEVELOPMENT (.env)                        PRODUCTION (set in hosting dashboard)
──────────────────                        ─────────────────────────────────────
PORT=3001                                 PORT=10000 (set by host automatically)
CORS_ORIGIN=http://localhost:5173         CORS_ORIGIN=https://your-app.vercel.app
NODE_ENV=development                      NODE_ENV=production
```

The same code runs in both environments, but environment variables change its behavior.

---

## Deploying the Frontend (Vercel)

### Why Vercel

- **Vercel over Netlify**: Both are excellent for static frontends. Vercel has slightly better integration with modern React tooling and simpler configuration. Netlify is a strong alternative.
- **Vercel over AWS S3**: S3 requires configuring CloudFront, SSL certificates, and DNS manually. Vercel does all of this automatically in seconds.
- **Vercel over GitHub Pages**: GitHub Pages is for truly static sites. Vercel handles environment variables, redirects, and client-side routing out of the box.

### Steps

1. **Check if you already have a Vercel account**: Go to https://vercel.com and try to log in. If you can log in, skip to step 2. If not, sign up with your GitHub account.

2. **Import your repository**:
   - Click "Add New" → "Project"
   - Select your `dev-pulse` repository from GitHub
   - Configure:
     - **Framework Preset**: Vite
     - **Root Directory**: `client`
     - **Build Command**: `npm run build`
     - **Output Directory**: `dist`

3. **Add environment variables**:
   - Click "Environment Variables"
   - Add: `VITE_API_URL` = `https://your-backend-url.onrender.com/api`
   - (You'll get this URL after deploying the backend)

4. **Click Deploy**

Vercel will:
- Pull your code from GitHub
- Run `npm install` in the `client/` folder
- Run `npm run build`
- Upload the `dist/` folder to their CDN (Content Delivery Network)
- Give you a URL like `https://dev-pulse-abc123.vercel.app`

### What Is a CDN?

```
                          ┌──── CDN Node (Tokyo) ──────┐
                          │  Copy of your dist/ files   │
┌────────────┐           │  Serves users in Asia fast  │
│ Your build │──deploy──▶ ├──── CDN Node (London) ─────┤
│ (dist/)    │           │  Copy of your dist/ files   │
└────────────┘           │  Serves users in Europe     │
                          ├──── CDN Node (New York) ────┤
                          │  Copy of your dist/ files   │
                          │  Serves users in US East    │
                          └────────────────────────────┘
```

> [!NOTE]
> **Technical**: A CDN distributes your static files across servers worldwide. Users download from the nearest server, reducing latency.

> [!NOTE]
> **Plain English**: Instead of one server in one location, your files are copied to servers around the world. Someone in Tokyo gets the files from a nearby server instead of waiting for them to travel from the US.

---

## Deploying the Backend (Render)

### Why Render

- **Render over Heroku**: Heroku's free tier was removed. Render offers a free tier that's sufficient for learning.
- **Render over Railway**: Both are good. Render has more straightforward documentation and a simpler onboarding experience.
- **Render over AWS EC2**: EC2 requires managing a virtual machine, installing Node, configuring security groups, setting up a reverse proxy... Render does all of this automatically.

### Steps

1. **Check if you already have a Render account**: Go to https://render.com and try to log in. If you can log in, skip to step 2. If not, sign up with your GitHub account.

2. **Create a new Web Service**:
   - Click "New" → "Web Service"
   - Connect your GitHub repository
   - Configure:
     - **Name**: `dev-pulse-api`
     - **Root Directory**: `server`
     - **Runtime**: Node
     - **Build Command**: `npm install && npm run build`
     - **Start Command**: `npm start`

3. **Add environment variables**:
   - `PORT`: (Render sets this automatically)
   - `CORS_ORIGIN`: `https://dev-pulse-abc123.vercel.app` (your Vercel URL)
   - `NODE_ENV`: `production`

4. **Click Create Web Service**

Render will:
- Pull your code from GitHub
- Run `npm install` in the `server/` folder
- Run `npm run build` (compiles TypeScript)
- Run `npm start` (starts the Express server)
- Give you a URL like `https://dev-pulse-api.onrender.com`

### Update Vercel Environment Variable

Now that you have the backend URL, go back to Vercel:
- Settings → Environment Variables
- Set `VITE_API_URL` to `https://dev-pulse-api.onrender.com/api`
- Redeploy (Vercel dashboard → Deployments → Redeploy)

---

## CORS in Production

In development, CORS was:
```typescript
cors({ origin: 'http://localhost:5173' })
```

In production, it must be:
```typescript
cors({ origin: process.env.CORS_ORIGIN || 'http://localhost:5173' })
```

We already wrote it this way in Step 2 of the backend. The `CORS_ORIGIN` environment variable on Render overrides the default.

**Why CORS matters in production**: Your frontend is at `your-app.vercel.app` and your backend is at `your-api.onrender.com`. These are different origins. Without CORS, the browser blocks every request from the frontend to the backend.

---

## Production Debugging

### What If Something Doesn't Work?

**Frontend not loading:**
- Check Vercel deployment logs (Dashboard → Deployments → click latest → Logs)
- Did the build succeed? Look for errors in the build log
- Is the environment variable set correctly?

**Backend not responding:**
- Check Render logs (Dashboard → your service → Logs)
- Is the server starting? Look for `Server running on...`
- Is the PORT being read from the environment variable?

**Frontend loads but can't reach backend:**
- Open browser DevTools → Network tab
- Look for failed requests (red)
- Check: Is the VITE_API_URL pointing to the correct Render URL?
- Check: Is CORS_ORIGIN on Render set to your Vercel URL?
- Check: Did you include `/api` in VITE_API_URL?

### Common Production Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| 502 Bad Gateway | Backend crashed | Check Render logs for errors |
| CORS error | Origin mismatch | Verify CORS_ORIGIN matches frontend URL exactly |
| API returns 404 | Wrong URL | Check that VITE_API_URL includes `/api` |
| Frontend blank | Build failed | Check Vercel build logs |
| Data missing after deploy | In-memory data resets on deploy | Expected — Level 2 adds a database |

---

## Making It Publicly Accessible

After deployment:

1. **Your frontend URL**: `https://dev-pulse-abc123.vercel.app`
   - This is publicly accessible. Share it with anyone.

2. **Your backend URL**: `https://dev-pulse-api.onrender.com`
   - This is publicly accessible. The `/api/health` endpoint shows it's running.

3. **Your GitHub repository**: `https://github.com/yourusername/dev-pulse`
   - This is where employers look at your code.

Update your README with the live URLs.

---

## Commit and Push

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
git add .
git commit -m "docs: add deployment configuration and update README with live URLs"
git push
```

---

## Spatial Check-In — The Complete Picture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DEPLOYED ARCHITECTURE                              │
│                                                                      │
│   User's Browser                                                     │
│   ┌──────────────────────┐                                          │
│   │  Types URL:           │                                          │
│   │  your-app.vercel.app  │                                          │
│   └──────────┬───────────┘                                          │
│              │                                                       │
│              │ HTTPS                                                  │
│              ▼                                                       │
│   ┌──────────────────────┐     Serves static files (HTML, JS, CSS)  │
│   │     Vercel CDN       │     from dist/ folder                     │
│   │  (Frontend Host)     │     No server code runs here              │
│   └──────────────────────┘                                          │
│              │                                                       │
│              │ The JS in the browser makes fetch() calls to:         │
│              │ your-api.onrender.com/api/entries                     │
│              │                                                       │
│              │ HTTPS                                                  │
│              ▼                                                       │
│   ┌──────────────────────┐     Runs Node.js + Express               │
│   │    Render Server     │     Processes API requests                │
│   │  (Backend Host)      │     Stores data in memory                 │
│   └──────────────────────┘                                          │
│                                                                      │
│   Both auto-deploy when you push to GitHub.                          │
└──────────────────────────────────────────────────────────────────────┘
```

---

| | | |
|:---|:---:|---:|
| [← 05 — Connecting Frontend to Backend](../05-connecting/) | [Level 1 Overview](../) | [07 — Growth Review →](../07-growth/) |
