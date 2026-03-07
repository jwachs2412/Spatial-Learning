`Level 1` **Step 5 of 7** — Connecting Frontend to Backend

# 05 — Connecting Frontend to Backend

## Spatial Orientation

```
┌──────────────────────────────────────────────────────────────┐
│                    YOUR COMPUTER                             │
│                                                              │
│   Terminal 1                      Terminal 2                 │
│   ┌────────────────────┐         ┌────────────────────┐     │
│   │  cd server          │         │  cd client          │     │
│   │  npm run dev        │         │  npm run dev        │     │
│   │                    │         │                    │     │
│   │  Express listening │         │  Vite dev server   │     │
│   │  on port 3001      │ ◀─────▶ │  on port 5173      │     │
│   └────────────────────┘  HTTP   └────────────────────┘     │
│                                                              │
│   ★ THIS SECTION: Making these two talk to each other ★     │
└──────────────────────────────────────────────────────────────┘
```

**Where are we?** At the boundary between frontend and backend. This is where full-stack development actually happens — connecting two independent systems through HTTP.

---

## Step 1: Run Both Servers

You need **two terminal windows** (or two VS Code integrated terminals). To open a second terminal in VS Code, click Terminal → New Terminal, or press `` Ctrl+Shift+` ``.

**Terminal 1 — Backend:**

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd server
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/server/`

```bash
npm run dev
```

Output: `Server running on http://localhost:3001`

**Leave this terminal running.** Do not close it or press `Ctrl+C`.

**Terminal 2 — Frontend:**

Open a **new, separate terminal**. It should start in the project root.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
cd client
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/client/`

```bash
npm run dev
```

Output: Shows a URL like `http://localhost:5173`

**Leave this terminal running too.** You now have two terminals open — one running the backend, one running the frontend.

Open `http://localhost:5173` in your browser. You should see the DevPulse app.

---

## Step 2: Verify the Connection

If everything is wired correctly:

1. The page loads with "No entries yet"
2. Fill out the form and click "Log Entry"
3. The entry appears in the list below
4. Refresh the page — entries reload from the backend

### If It's Not Working

**"Failed to load entries. Is the backend running?"**
- The frontend can't reach the backend
- Check: Is Terminal 1 still running? Did it crash?
- Check: Is the backend on port 3001? Look at the terminal output

**CORS Error (in browser console):**
```
Access to fetch at 'http://localhost:3001/api/entries' from origin
'http://localhost:5173' has been blocked by CORS policy
```
This means the CORS middleware isn't configured correctly. Verify `server/src/index.ts` has:
```typescript
app.use(cors({
  origin: 'http://localhost:5173',
}));
```

**Network Error / Connection Refused:**
- The backend isn't running
- In a terminal, navigate to `dev-pulse/server/` and run `npm run dev`

---

## Step 3: Understand What Just Happened

Let's trace the entire request lifecycle in detail:

### Loading the Page (Initial GET)

```
1. Browser navigates to http://localhost:5173
   │
   ▼
2. Vite dev server sends back index.html + JavaScript bundle
   │  (This is your React app — all of it)
   │
   ▼
3. Browser executes JavaScript
   │  React renders App component
   │  useEffect fires → calls loadEntries()
   │
   ▼
4. loadEntries() calls getEntries() in api.ts
   │
   ▼
5. fetch('http://localhost:3001/api/entries')
   │
   │  ┌──── HTTP Request ────────────────────────┐
   │  │  GET /api/entries HTTP/1.1                │
   │  │  Host: localhost:3001                     │
   │  │  Origin: http://localhost:5173            │
   │  └──────────────────────────────────────────┘
   │
   ▼
6. Express receives the request
   │  → cors() middleware checks Origin header → allowed ✓
   │  → express.json() middleware → no body to parse
   │  → route handler: returns entries array as JSON
   │
   │  ┌──── HTTP Response ───────────────────────┐
   │  │  HTTP/1.1 200 OK                         │
   │  │  Access-Control-Allow-Origin: ...5173     │
   │  │  Content-Type: application/json           │
   │  │                                           │
   │  │  []                                       │
   │  └──────────────────────────────────────────┘
   │
   ▼
7. fetch() resolves with the response
   │  response.json() parses the empty array
   │
   ▼
8. setEntries([]) → React re-renders
   │  EntryList shows "No entries yet"
   │
   ▼
9. User sees the empty state
```

### Submitting an Entry (POST)

```
1. User fills form and clicks "Log Entry"
   │
   ▼
2. handleSubmit fires → e.preventDefault() → no page reload
   │
   ▼
3. Calls onSubmit({ mood: 'happy', energy: 4, note: '...' })
   │  (This is the handleCreateEntry function from App)
   │
   ▼
4. createEntry() sends a POST request
   │
   │  ┌──── HTTP Request ────────────────────────┐
   │  │  POST /api/entries HTTP/1.1               │
   │  │  Content-Type: application/json           │
   │  │  Origin: http://localhost:5173            │
   │  │                                           │
   │  │  {"mood":"happy","energy":4,"note":"..."}│
   │  └──────────────────────────────────────────┘
   │
   ▼
5. Express receives the request
   │  → cors() ✓
   │  → express.json() parses body into req.body
   │  → route handler:
   │     validates data ✓
   │     creates entry with id and timestamp
   │     pushes to array
   │     responds with 201 + new entry
   │
   │  ┌──── HTTP Response ───────────────────────┐
   │  │  HTTP/1.1 201 Created                    │
   │  │  Content-Type: application/json           │
   │  │                                           │
   │  │  {"id":1,"mood":"happy","energy":4,...}   │
   │  └──────────────────────────────────────────┘
   │
   ▼
6. createEntry() returns the new entry object
   │
   ▼
7. setEntries([newEntry, ...prev])
   │  State updates → React re-renders
   │
   ▼
8. EntryList shows the new card
   MoodChart updates the bars
   Form resets to default values
```

---

> [!TIP]
> **Session Break** — You've connected the frontend and backend and traced the full request lifecycle. Save your work and take a break.
> When you return, you'll explore DevTools, optionally configure a proxy, and push to GitHub.

---

## Step 4: Open the Browser DevTools

This is one of the most important development skills. The browser's DevTools let you see exactly what's happening.

### Open DevTools
- **Chrome/Edge**: `F12` or `Cmd+Option+I` (macOS) / `Ctrl+Shift+I` (Windows)
- **Firefox**: `F12` or `Cmd+Option+I` / `Ctrl+Shift+I`

### Network Tab

Click the **Network** tab. Now refresh the page or submit a form entry.

You'll see each HTTP request:

```
Name               Method   Status   Type
entries            GET      200      fetch
entries            POST     201      fetch
```

Click any request to see:
- **Headers**: The full request and response headers
- **Payload**: What was sent (for POST)
- **Response**: What came back (the JSON data)

**This is how you debug connection issues.** If a request fails, the Network tab tells you exactly what happened.

### Console Tab

Click the **Console** tab. Errors appear here in red. If you see CORS errors or network errors, this is where they'll show up.

---

## Step 5: Add a Proxy (Optional Convenience)

In development, the frontend runs on `:5173` and the backend on `:3001`. You can set up a Vite proxy so the frontend automatically forwards API requests to the backend. This avoids CORS entirely in development.

In VS Code, open the file `client/vite.config.ts` and replace its contents with:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
});
```

**What this does**: Any request from the frontend to `/api/...` gets forwarded to `http://localhost:3001/api/...` by Vite's dev server. The browser sees the request going to `localhost:5173/api/...`, so there's no cross-origin issue.

**If you add the proxy**, update `client/src/services/api.ts`:

```typescript
// With the proxy, we use a relative URL in development
// In production, we still need the full URL (set via VITE_API_URL)
const API_URL = import.meta.env.VITE_API_URL || '/api';
```

**Important**: The proxy only works in development (Vite dev server). In production, you still need CORS configured on the backend and the full API URL set in `VITE_API_URL`.

---

## Step 6: Commit

Stop both servers by pressing `Ctrl+C` in each terminal. Open a terminal at the project root.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

```bash
git add .
git commit -m "feat: connect frontend to backend, verify full data flow"
```

---

## Step 7: Push to GitHub

> [!IMPORTANT]
> **You should be in:** `dev-pulse/` (the project root)

First, check if a remote is already set up:

```bash
git remote -v
```

If you see `origin` with a URL listed, a remote is already configured — just run `git push`. If nothing is listed, create a GitHub repository and push:

```bash
# Option A: Using GitHub CLI — first check if it's installed:
gh --version
# If you see a version number, you can use:
gh repo create dev-pulse --public --source=. --remote=origin --push

# Option B: If gh is not installed, do it manually:
# 1. Go to github.com and create a new repository called "dev-pulse"
# 2. Do NOT initialize with a README (you already have one)
# 3. Then run these commands from dev-pulse/:
git remote add origin git@github.com:yourusername/dev-pulse.git
git push -u origin main
```

---

You now have a **complete, working full-stack application** running on your local machine.

```
┌───────────────────────────────────────────────────────────────┐
│                  WHAT EXISTS NOW                               │
│                                                               │
│   ┌──────────────────┐         ┌──────────────────┐          │
│   │   FRONTEND ✓     │  HTTP   │   BACKEND ✓      │          │
│   │                  │ ◀─────▶ │                  │          │
│   │  React app       │         │  Express API     │          │
│   │  Shows form      │         │  Stores entries  │          │
│   │  Shows list      │         │  Validates data  │          │
│   │  Shows chart     │         │  Returns JSON    │          │
│   │                  │         │                  │          │
│   │  localhost:5173  │         │  localhost:3001   │          │
│   └──────────────────┘         └──────────────────┘          │
│                                                               │
│   NEXT: Deploy both to the internet so anyone can use it.    │
└───────────────────────────────────────────────────────────────┘
```

**Remaining limitation**: Data lives in memory. When you restart the backend, all entries are lost. This is intentional for Level 1 — Level 2 introduces a real database.

---

> [!TIP]
> ## Spatial Check-In

1. **What does the proxy setting in vite.config.ts do?**

<details><summary>Answer</summary>

It forwards API requests from the frontend dev server (port 5173) to the backend (port 3001), avoiding CORS issues during development.

</details>

2. **What happens if the backend is not running when you load the frontend?**

<details><summary>Answer</summary>

The fetch calls fail. The user sees either an error message (if you handle the error) or a blank page (if you don't). The frontend loads fine because Vite serves it, but the data never arrives.

</details>

---

| | | |
|:---|:---:|---:|
| [← 04 — Building the Frontend](../04-frontend/) | [Level 1 Overview](../) | [06 — Deployment →](../06-deployment/) |
