`Level 1` **Step 3 of 7** — Building the Backend

# 03 — Building the Backend

## Spatial Orientation

```
dev-pulse/
├── client/          ← NOT working here right now
└── server/          ← ★ WE ARE HERE ★
    └── src/
        ├── index.ts       ← Server entry point (we'll build this)
        ├── routes/
        │   └── entries.ts ← API routes (we'll build this)
        └── types/
            └── index.ts   ← Type definitions (we'll build this)
```

**What layer are we in?** The SERVER layer. This code runs on a computer (your machine now, a cloud server later). Users never see this code. The browser cannot access these files.

**What does this layer do?** It listens for HTTP requests, processes them, and sends back responses. It's the kitchen — it receives orders and sends back food.

---

## Step 1: Define Types

**Where are we?** `server/src/types/index.ts`

**Why types first?** Types are the blueprint. Before building anything, define what the data looks like. This is a TypeScript superpower — you describe the shape of your data, and the compiler ensures you never violate that shape.

First, create the folder structure. Open your terminal:

> [!IMPORTANT]
> **You should be in:** `dev-pulse/`

```bash
mkdir -p server/src/types
mkdir -p server/src/routes
```

Now create the types file. In VS Code, create a new file at `server/src/types/index.ts` (right-click on the `server/src/types` folder → New File → name it `index.ts`).

Add this code to `server/src/types/index.ts`:

```typescript
/**
 * Represents a single mood entry submitted by a developer.
 *
 * Think of this as a row in a spreadsheet:
 * | id | mood    | energy | note              | createdAt           |
 * |----|---------|--------|-------------------|---------------------|
 * | 1  | "happy" | 4      | "Great coding day"| 2026-03-01T14:00:00 |
 */
export interface MoodEntry {
  id: number;
  mood: 'happy' | 'neutral' | 'frustrated' | 'tired' | 'energized';
  energy: number;  // 1-5 scale
  note: string;
  createdAt: string;  // ISO 8601 date string
}

/**
 * The shape of data the client sends when creating a new entry.
 * Notice: no `id` or `createdAt` — the SERVER assigns those.
 *
 * Why? The client doesn't get to choose its own ID (that would be
 * a security and consistency problem). The server is the authority.
 */
export interface CreateEntryRequest {
  mood: MoodEntry['mood'];
  energy: number;
  note: string;
}
```

### TypeScript Concepts Explained

**`interface`**
- > [!NOTE]
> **Technical**: An interface defines the shape of an object — what properties it has and what types those properties are.
- > [!NOTE]
> **Plain English**: It's a contract. If something claims to be a `MoodEntry`, it MUST have an `id` (number), a `mood` (one of five specific strings), an `energy` (number), a `note` (string), and a `createdAt` (string). No exceptions.

**Union Types (`'happy' | 'neutral' | 'frustrated' | 'tired' | 'energized'`)**
- > [!NOTE]
> **Technical**: A union type allows a value to be one of several specified types.
- > [!NOTE]
> **Plain English**: The mood can only be one of these five words. If you try to set mood to `"ecstatic"`, TypeScript will show an error before your code even runs. This catches bugs at development time, not in production.

**`MoodEntry['mood']`**
- > [!NOTE]
> **Technical**: An indexed access type that extracts the type of a specific property from another type.
- > [!NOTE]
> **Plain English**: Instead of repeating the list of mood strings, we say "the mood field in CreateEntryRequest is the same type as the mood field in MoodEntry." If we add a new mood option to MoodEntry, CreateEntryRequest automatically includes it.

---

## Step 2: Build the Express Server

**Where are we?** `server/src/index.ts`

**What is this file?** The entry point — the first file that runs when you start the server. It creates the Express application, configures middleware, connects routes, and starts listening for requests.

In VS Code, create a new file at `server/src/index.ts` (right-click on the `server/src` folder → New File → name it `index.ts`).

Add this code to `server/src/index.ts`:

```typescript
import express from 'express';
import cors from 'cors';
import { entriesRouter } from './routes/entries';

// Create the Express application
const app = express();

// Define the port (use environment variable or default to 3001)
const PORT = process.env.PORT || 3001;

// ─── MIDDLEWARE ───────────────────────────────────────────────
// Middleware runs on EVERY request before it reaches your routes.
// Think of it as a security checkpoint at the entrance of a building.
// Every visitor must pass through before going to their destination.

// Parse JSON request bodies
// Without this, req.body would be undefined when clients send JSON
app.use(express.json());

// Enable CORS — allow the frontend to make requests to this server
// Without this, the browser would block all requests from localhost:5173
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
}));

// ─── ROUTES ──────────────────────────────────────────────────
// Routes define what happens when a request arrives at a specific URL.
// We organize routes in separate files to keep this file clean.
app.use('/api/entries', entriesRouter);

// ─── HEALTH CHECK ────────────────────────────────────────────
// A simple endpoint that confirms the server is running.
// Deployment platforms use this to verify your server is alive.
app.get('/api/health', (_req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// ─── START SERVER ────────────────────────────────────────────
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

### Line-by-Line Breakdown

**`import express from 'express'`**
Imports the Express library. Express is a minimal web framework that handles HTTP requests. Without it, you'd have to use Node's raw `http` module, which requires much more code for the same result.

**`const app = express()`**
Creates an Express application instance. This `app` object is the core of your server — you attach middleware and routes to it.

**`const PORT = process.env.PORT || 3001`**
Reads the PORT from environment variables. If none is set (local development), defaults to 3001. Why 3001? It doesn't conflict with Vite's 5173. Any unused port would work.

**`app.use(express.json())`**
Middleware. When a client sends JSON in a request body, this parses it and puts the result in `req.body`. Without it, you'd receive raw text and have to parse it yourself.

**`app.use(cors(...))`**
Middleware. Tells the browser "requests from this origin are allowed." Without it, the browser blocks cross-origin requests (see Orientation section on CORS).

**`app.use('/api/entries', entriesRouter)`**
Mounts the entries route handler at `/api/entries`. Any request to `/api/entries` or `/api/entries/anything` gets handled by `entriesRouter`.

**`app.listen(PORT, callback)`**
Tells the server to start listening for HTTP requests on the specified port. The callback runs once the server is ready.

### What Is Middleware?

```
                    REQUEST ARRIVES
                         │
                         ▼
              ┌─────────────────────┐
              │   express.json()    │  ← Parses JSON body
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │       cors()        │  ← Checks origin, adds headers
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │    YOUR ROUTE       │  ← Your handler code runs
              │    HANDLER          │
              └──────────┬──────────┘
                         │
                         ▼
                  RESPONSE SENT
```

> [!NOTE]
> **Technical**: Middleware functions are functions that have access to the request object, response object, and the next middleware function. They can execute code, modify the request/response, end the cycle, or pass control to the next middleware.

> [!NOTE]
> **Plain English**: Middleware is like a chain of checkpoints. Every request passes through each checkpoint in order. Each checkpoint can inspect the request, modify it, reject it, or pass it along. `express.json()` is a checkpoint that reads the request body and turns JSON text into a JavaScript object. `cors()` is a checkpoint that adds the right headers to allow cross-origin requests.

---

## Step 3: Build the API Routes

**Where are we?** `server/src/routes/entries.ts`

**What is this file?** It defines what happens when someone sends a request to `/api/entries`. It contains the route handlers — the functions that receive requests and send responses.

In VS Code, create a new file at `server/src/routes/entries.ts` (right-click on the `server/src/routes` folder → New File → name it `entries.ts`).

Add this code to `server/src/routes/entries.ts`:

```typescript
import { Router } from 'express';
import { MoodEntry, CreateEntryRequest } from '../types';

const router = Router();

// ─── IN-MEMORY DATA STORE ────────────────────────────────────
// This array acts as our "database" for now.
// WARNING: Data is lost when the server restarts.
// In Level 2, we replace this with a real database.
let entries: MoodEntry[] = [];
let nextId = 1;

// ─── GET /api/entries ────────────────────────────────────────
// Returns all mood entries, newest first.
//
// Request:  GET /api/entries
// Response: 200 OK, [{ id, mood, energy, note, createdAt }, ...]
router.get('/', (_req, res) => {
  // Return entries sorted newest first
  const sorted = [...entries].sort(
    (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
  );
  res.json(sorted);
});

// ─── POST /api/entries ───────────────────────────────────────
// Creates a new mood entry.
//
// Request:  POST /api/entries
//           Body: { mood: "happy", energy: 4, note: "Great day" }
// Response: 201 Created, { id, mood, energy, note, createdAt }
router.post('/', (req, res) => {
  const { mood, energy, note } = req.body as CreateEntryRequest;

  // ─── VALIDATION ──────────────────────────────────────────
  // Never trust data from the client. Always validate.
  // The browser is untrusted territory.

  const validMoods = ['happy', 'neutral', 'frustrated', 'tired', 'energized'];

  if (!mood || !validMoods.includes(mood)) {
    res.status(400).json({
      error: 'Invalid mood. Must be one of: ' + validMoods.join(', '),
    });
    return;
  }

  if (!energy || energy < 1 || energy > 5 || !Number.isInteger(energy)) {
    res.status(400).json({
      error: 'Energy must be an integer between 1 and 5',
    });
    return;
  }

  if (!note || typeof note !== 'string' || note.trim().length === 0) {
    res.status(400).json({
      error: 'Note is required and must be a non-empty string',
    });
    return;
  }

  if (note.length > 500) {
    res.status(400).json({
      error: 'Note must be 500 characters or fewer',
    });
    return;
  }

  // ─── CREATE ENTRY ────────────────────────────────────────
  const newEntry: MoodEntry = {
    id: nextId++,
    mood,
    energy,
    note: note.trim(),
    createdAt: new Date().toISOString(),
  };

  entries.push(newEntry);

  // 201 = "Created" — the standard status code when a new resource is made
  res.status(201).json(newEntry);
});

export { router as entriesRouter };
```

### Route Handler Anatomy

```typescript
router.get('/', (_req, res) => {
  res.json(sorted);
});
```

| Part | What It Is |
|------|-----------|
| `router.get` | Handle GET requests |
| `'/'` | At this path (relative to where the router is mounted — `/api/entries/`) |
| `_req` | The request object (underscore means "we don't use this") |
| `res` | The response object (what we send back) |
| `res.json(...)` | Send a JSON response |

### Why We Validate

```
CLIENT (untrusted)                    SERVER (trusted)
─────────────────                     ────────────────
User could send:                      We check:
{ mood: "HACKED", energy: 9999 }  →  Is mood valid? NO → 400 error
{ mood: "happy", energy: 4 }      →  Is mood valid? YES → continue
```

> [!NOTE]
> **Technical**: Input validation ensures that data conforms to expected formats and constraints before processing, preventing data corruption, injection attacks, and unexpected behavior.

> [!NOTE]
> **Plain English**: The frontend might send anything — even data that looks nothing like a mood entry. Maybe there's a bug in the frontend. Maybe someone is using curl or Postman to send garbage. The backend must check everything and reject bad data with a clear error message.

**Key principle**: The frontend validates for user experience (quick feedback). The backend validates for security and data integrity (final authority).

---

## Step 4: Test the Backend

### Start the Server

Open your terminal.

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

You should see: `Server running on http://localhost:3001`

### Test with curl or Thunder Client

Open a **second terminal** (keep the server running in the first one). The curl commands below can be run from any directory — they're sending HTTP requests to your running server, not accessing files.

**Test health check:**
```bash
curl http://localhost:3001/api/health
```
Expected: `{"status":"ok","timestamp":"..."}`

**Test GET entries (empty):**
```bash
curl http://localhost:3001/api/entries
```
Expected: `[]`

**Test POST a new entry:**
```bash
curl -X POST http://localhost:3001/api/entries \
  -H "Content-Type: application/json" \
  -d '{"mood":"happy","energy":4,"note":"Great coding day"}'
```
Expected: `{"id":1,"mood":"happy","energy":4,"note":"Great coding day","createdAt":"..."}`

**Test GET entries (should have one now):**
```bash
curl http://localhost:3001/api/entries
```
Expected: `[{"id":1,...}]`

**Test validation (bad mood):**
```bash
curl -X POST http://localhost:3001/api/entries \
  -H "Content-Type: application/json" \
  -d '{"mood":"invalid","energy":4,"note":"test"}'
```
Expected: `{"error":"Invalid mood. Must be one of: happy, neutral, frustrated, tired, energized"}`

### Using Thunder Client (VS Code)

If you installed the Thunder Client extension:
1. Click the thunder icon in VS Code's sidebar
2. Create a new request
3. Set method to GET, URL to `http://localhost:3001/api/entries`
4. Click Send
5. See the response in the panel below

This is faster than writing curl commands and gives you a visual interface.

---

## Step 5: Commit

Stop the server by pressing `Ctrl+C` in the terminal where it's running.

> [!IMPORTANT]
> **You should be in:** `dev-pulse/server/`

Go back to the project root first:

```bash
cd ..
```

> [!IMPORTANT]
> **You are now in:** `dev-pulse/` (the project root)

```bash
git add server/
git commit -m "feat: add Express server with mood entries API"
```

---

## Spatial Check-In

You now have a working backend. Let's make sure you understand its place in the system:

```
┌─────────────────────────────────────────────────────┐
│                  WHAT EXISTS NOW                     │
│                                                     │
│   ┌───────────────────┐    ┌───────────────────┐   │
│   │    FRONTEND        │    │    BACKEND ✓      │   │
│   │    (not built yet) │    │                   │   │
│   │                    │    │  GET /api/entries  │   │
│   │                    │    │  POST /api/entries │   │
│   │                    │    │  GET /api/health   │   │
│   │                    │    │                   │   │
│   │    NEXT STEP       │    │  In-memory data   │   │
│   └───────────────────┘    └───────────────────┘   │
│                                                     │
│   Currently, only curl/Thunder Client can           │
│   talk to the backend. Next we build the            │
│   frontend to provide a real user interface.        │
└─────────────────────────────────────────────────────┘
```

> [!TIP]
> **Spatial Check-In** — Questions to answer before moving on.

1. **What happens if you restart the server?**

<details><summary>Answer</summary>

All entries are lost (in-memory storage)

</details>

2. **What happens if you send a POST without `Content-Type: application/json`?**

<details><summary>Answer</summary>

`req.body` is undefined, validation fails

</details>

3. **Why do we validate on the server even though we'll also validate on the frontend?**

<details><summary>Answer</summary>

The server is the authority. The frontend is untrusted.

</details>

4. **What does `res.status(201).json(newEntry)` do?**

<details><summary>Answer</summary>

Sends HTTP status 201 (Created) and the new entry as JSON

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 1 Overview](../) | [04 — Building the Frontend →](../04-frontend/) |
