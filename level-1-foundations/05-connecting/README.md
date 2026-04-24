# Step 5 — Connecting Frontend to Backend

## Spatial Orientation

```
┌───────────────────────┐         ┌───────────────────────┐
│   FRONTEND (client/)  │  HTTP   │   BACKEND (server/)   │
│                       │────────▶│                       │
│   React components    │◀────────│   Express API         │
│   call api.ts         │         │   returns JSON        │
│                       │         │                       │
│   localhost:5173      │         │   localhost:3001       │
└───────────────────────┘         └───────────────────────┘

YOU ARE HERE: Making these two talk to each other
```

Until now, you've built the frontend and backend separately. In this step, you'll run them simultaneously and verify data flows correctly.

---

## 1. Run Both Servers

You need **two terminal windows** — one for each server.

**Terminal 1 — Backend:**
```bash
cd server
npm run dev
# Should see: Server running on http://localhost:3001
```

**Terminal 2 — Frontend:**
```bash
cd client
npm run dev
# Should see: Local: http://localhost:5173/
```

### ✅ Checkpoint

1. Open `http://localhost:3001/api/health` in your browser — should see `{"status":"ok",...}`
2. Open `http://localhost:5173` in your browser — should see the DevPulse UI

---

## 2. Test the Connection

Open DevPulse at `http://localhost:5173` and:

1. **Submit a mood entry** using the form
2. **Check the entry appears** in the list below
3. **Open browser DevTools** (F12) → Network tab
4. **Submit another entry** and watch the Network tab

### What to Look For in DevTools

When you submit, you should see:

```
Name              Method    Status    Type
entries           POST      201       fetch
```

Click on the request to see:
- **Headers** tab: Request URL, method, content-type
- **Payload** tab: The data you sent
- **Response** tab: The entry the server created (with id and createdAt)

When the page loads, you should also see:

```
Name              Method    Status    Type
entries           GET       200       fetch
```

### 🧠 Think About It

What does the full lifecycle of creating an entry look like? Trace the data from the user's click to the screen update.

<details>
<summary>Answer</summary>

```
1. User clicks "Log Entry"
2. React calls handleSubmit() in App.tsx
3. handleSubmit() calls createEntry() in api.ts
4. api.ts sends POST /api/entries to localhost:3001
   - With headers: Content-Type: application/json
   - With body: { mood, energy, note, date }
5. Express receives the request
6. express.json() middleware parses the JSON body
7. CORS middleware checks if localhost:5173 is allowed → yes
8. entries.ts route handler validates the data
9. Handler creates a MoodEntry object with id and createdAt
10. Handler pushes it to the in-memory array
11. Handler responds with 201 and the new entry as JSON
12. api.ts receives the response, parses JSON
13. handleSubmit() receives the MoodEntry
14. setEntries() adds it to the front of the array
15. React re-renders EntryList with the new entry
16. React re-renders MoodChart with updated counts
17. User sees the new entry on screen
```

All of this happens in under 100 milliseconds locally.

</details>

---

## 3. Troubleshooting Common Issues

### CORS Error

If you see in the browser console:
```
Access to fetch at 'http://localhost:3001/api/entries' from origin 'http://localhost:5173' has been blocked by CORS policy
```

**Fix:** Check your `server/src/index.ts`:
```typescript
app.use(cors({
  origin: 'http://localhost:5173',  // No trailing slash!
}));
```

> ⚠️ **Common Mistake: Trailing Slash**
> `'http://localhost:5173/'` ≠ `'http://localhost:5173'`. The browser sends the origin WITHOUT a trailing slash. If your CORS config has one, every request gets blocked. Always omit the trailing slash.

### "Failed to fetch" Error

If you see `TypeError: Failed to fetch` in the console:

1. **Is the backend running?** Check Terminal 1 for the server output
2. **Is the port correct?** Your API URL should point to `localhost:3001`
3. **Did the server crash?** Check Terminal 1 for errors

### Data Disappears on Backend Restart

This is expected! Data lives in memory. When you restart the backend, the array resets to empty. Level 2 fixes this with a database.

---

## 4. Optional: Vite Proxy Configuration

> **Key Concept: Development Proxy**
> A proxy makes your frontend server forward certain requests to the backend. Instead of calling `http://localhost:3001/api/entries` directly (triggering CORS), your frontend calls `/api/entries` and Vite forwards it to the backend behind the scenes. This eliminates CORS issues in development.

This is optional but useful to understand. In `client/vite.config.ts`:

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

### Reading This Config Line-by-Line

- `import { defineConfig } from 'vite';` — named import of Vite's `defineConfig` helper. It's a no-op at runtime; its job is to give you TypeScript autocomplete on the config object so you don't have to remember every option.
- `import react from '@vitejs/plugin-react';` — default import of Vite's React plugin. Packages that start with `@something/name` are called **scoped packages** (`@vitejs` is the scope, `plugin-react` is the name).
- `export default defineConfig({ ... });` — default export of the whole config. Vite reads this file when you run `npm run dev` and honors the settings.
- `plugins: [react()]` — the `plugins` option takes an array of plugin instances. We call `react()` to activate it. Without this, Vite wouldn't know how to handle JSX.
- `server: { proxy: { ... } }` — configuration for the dev server. `proxy` is an object mapping URL prefixes to forwarding rules.
- `'/api': { target: 'http://localhost:3001', changeOrigin: true }` — "when the browser asks the dev server for any URL starting with `/api`, quietly forward that request to `http://localhost:3001` (the backend) and send the response back."
  - `changeOrigin: true` — rewrites the request's `Host` header so the backend sees itself as the target. Avoids subtle bugs when the backend cares about the `Host` header.

With the proxy active, your frontend code can say `fetch('/api/entries')` and Vite transparently routes it to the backend. The browser thinks everything is same-origin, so CORS never enters the picture — in development.

If you use the proxy, update `client/src/services/api.ts`:

```typescript
// With proxy, no need for full URL
const API_URL = import.meta.env.VITE_API_URL || '/api';
```

**Reading this small change:** the fallback is now just `'/api'` (a relative URL) instead of `'http://localhost:3001/api'` (an absolute URL). A relative URL is resolved against the current page's origin — `http://localhost:5173` during development. Vite's proxy catches it and forwards. In production, the same relative path hits your real backend because by then you'll have deployed frontend and backend together or set `VITE_API_URL` to the production backend URL.

**Tradeoff:** The proxy only works in development (Vite dev server). In production, you'll still need CORS configured because the frontend and backend run on different domains. We keep CORS as the primary approach so development matches production behavior.

---

## 5. Verify Everything Works

### Full Test Sequence

1. **Create 3-4 mood entries** with different moods
2. **Check the entry list** — newest should appear first
3. **Check the mood chart** — bars should show distribution
4. **Restart the backend** (`Ctrl+C` in Terminal 1, then `npm run dev`)
5. **Refresh the frontend** — entries should be gone (in-memory, as expected)
6. **Create another entry** — should work without errors

### ✅ Checkpoint

- [ ] Can create entries and see them in the list
- [ ] Mood chart shows the correct distribution
- [ ] No errors in the browser console (F12 → Console)
- [ ] Network tab shows successful POST and GET requests
- [ ] Backend terminal shows no errors

---

## 6. Commit

```bash
git add .
git commit -m "feat: connect frontend to backend, verify full data flow"
```

---

## 🧠 Spatial Check-In

1. In your browser's Network tab, you see a POST request to `/api/entries` that returns status 201. Then you see a GET request to `/api/entries` that returns status 200. Which component triggered each request?

2. If your backend is running but the frontend shows "Failed to fetch," what are the three most likely causes?

3. Why do we need CORS at all? If both the frontend and backend are YOUR code, why does the browser block the request?

<details>
<summary>Check Your Answers</summary>

1. **POST** was triggered by the `handleSubmit` function in `App.tsx` (via `createEntry()` in `api.ts`). **GET** was triggered by the `useEffect` in `App.tsx` (via `getEntries()` in `api.ts`), which runs when the component first mounts.

2. **(a)** Wrong port in the API URL. **(b)** Backend crashed and isn't actually running (check the terminal). **(c)** CORS is misconfigured (wrong origin or trailing slash).

3. **Security.** The browser can't know that the backend is "yours." From the browser's perspective, `localhost:5173` is trying to talk to `localhost:3001` — a different origin. Without CORS, any malicious website could make requests to your API using the user's browser. CORS forces the server to explicitly whitelist trusted origins.

</details>

---

> **Session Break** — Frontend and backend are connected!
> When you return, you'll deploy to the internet in [Step 6 — Deployment](../06-deployment/).

---

| | |
|:---|---:|
| [← Step 4: Frontend](../04-frontend/) | [Step 6 — Deployment →](../06-deployment/) |
