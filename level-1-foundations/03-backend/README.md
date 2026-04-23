# Step 3 — Building the Backend

## Spatial Orientation

```
dev-pulse/
├── client/                ← Built in Step 2 (untouched this step)
└── server/                ← YOU ARE HERE
    └── src/
        ├── types/
        │   └── index.ts     ← Define data shapes
        ├── routes/
        │   └── entries.ts   ← Handle API requests
        └── index.ts         ← Server entry point
```

You're working in `server/src/`. Every file you create lives here. The frontend doesn't exist to the backend — they only communicate through HTTP.

---

## 1. Define Your Types

> **Key Concept: TypeScript Interfaces**
> An interface defines the *shape* of your data — what fields exist, what types they are. Think of it like a blueprint: it doesn't create anything, but it ensures everything built from it has the right structure. If you try to create an object missing a required field, TypeScript catches it at compile time.

### 🏗️ Your Turn

Before looking at the code, think about what a mood entry needs:
- A unique identifier
- The mood itself (e.g., "happy", "frustrated")
- An energy level (1-5)
- A note about the day
- When it was created

Write the interface yourself, then check the solution.

<details>
<summary>See the solution</summary>

Create `server/src/types/index.ts`:

```typescript
export interface MoodEntry {
  id: number;
  mood: 'happy' | 'neutral' | 'sad' | 'frustrated' | 'energized';
  energy: number;      // 1-5
  note: string;
  date: string;        // ISO format: "2025-01-15"
  createdAt: string;   // ISO timestamp
}

export type CreateEntryRequest = Omit<MoodEntry, 'id' | 'createdAt'>;
```

</details>

**Why these choices matter:**

| Decision | Why |
|----------|-----|
| `mood: 'happy' \| 'neutral' \| ...` | Union type limits values to valid options. TypeScript will catch `mood: "ecstatic"` as an error. |
| `energy: number` | We'll validate 1-5 in the route handler, but the type stays general. |
| `date: string` (not `Date`) | JSON doesn't have a Date type. Dates travel as strings over HTTP. |
| `Omit<MoodEntry, 'id' \| 'createdAt'>` | When creating an entry, the client doesn't provide `id` or `createdAt` — the server generates those. `Omit` creates a new type with those fields removed. |

> **Key Concept: `Omit<Type, Keys>`**
> `Omit` is a TypeScript utility type that creates a new type by removing specified keys from an existing type. Instead of duplicating the interface with fewer fields, you derive one type from another. This means if you add a field to `MoodEntry`, the `CreateEntryRequest` type automatically stays in sync.

---

## 2. Build the Server Entry Point

> **Key Concept: Express**
> Express is a minimal web framework for Node.js. It handles incoming HTTP requests and routes them to the right handler function. Think of it like a receptionist: it receives every request that comes in and directs it to the right department.

### 🏗️ Your Turn

Write the server entry point. It needs to:
1. Import and create an Express app
2. Add middleware to parse JSON request bodies
3. Add CORS middleware to allow frontend requests
4. Add a health check endpoint at `GET /api/health`
5. Start listening on port 3001

Hints:
- `express.json()` is middleware that parses JSON request bodies
- `cors({ origin: '...' })` configures which origins can make requests
- `app.listen(port, callback)` starts the server

<details>
<summary>See the solution</summary>

Create `server/src/index.ts`:

```typescript
import express, { Request, Response } from 'express';
import cors from 'cors';
import { entriesRouter } from './routes/entries';

const app = express();
const PORT = process.env.PORT || 3001;

// Middleware
app.use(express.json());
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
}));

// Routes
app.use('/api/entries', entriesRouter);

// Health check
app.get('/api/health', (_req: Request, res: Response) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
  });
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

</details>

**Line-by-line breakdown:**

| Line | What It Does | Why It Matters |
|------|-------------|----------------|
| `import express, { Request, Response } from 'express'` | Import Express and its TypeScript types | `Request` and `Response` give you autocomplete and type checking on handler parameters |
| `const PORT = process.env.PORT \|\| 3001` | Read port from environment variable, default to 3001 | Deployment platforms set `PORT` automatically. Fallback to 3001 for local development |
| `app.use(express.json())` | Parse JSON request bodies | Without this, `req.body` is `undefined` for POST requests |
| `app.use(cors({ origin: '...' }))` | Allow cross-origin requests from your frontend | Without this, the browser blocks every request from your React app |
| `(_req: Request, res: Response)` | Type the parameters explicitly | Prevents `Parameter implicitly has 'any' type` errors. The `_` prefix means "I know I'm not using this parameter" |
| `app.use('/api/entries', entriesRouter)` | Mount the entries router at `/api/entries` | All routes in `entriesRouter` are prefixed with `/api/entries` |

> ⚠️ **Common Mistake: Implicit `any` Types**
> If you write `(req, res)` instead of `(req: Request, res: Response)`, TypeScript will error with `Parameter 'req' implicitly has an 'any' type`. Always import and use the types from Express. This also gives you autocomplete on `req.body`, `req.params`, `res.json()`, etc.

> ⚠️ **Common Mistake: Forgetting `express.json()`**
> Without `app.use(express.json())`, your POST route will receive `undefined` for `req.body`. This is the #1 "why is my data undefined?" question. The middleware must be added **before** your routes.

---

## 3. Build the Entries Route

> **Key Concept: Router**
> An Express Router is a mini-application that handles a subset of routes. Instead of putting every endpoint in `index.ts`, you group related routes in their own file. Think of it like departments in a company: the main receptionist (index.ts) directs requests to the right department (router), which handles the specifics.

### 🏗️ Your Turn

Build the entries router with two endpoints:
- `GET /` — Return all entries (remember, the router is mounted at `/api/entries`, so `/` here means `/api/entries`)
- `POST /` — Create a new entry with validation

Validation rules:
- `mood` must be one of: 'happy', 'neutral', 'sad', 'frustrated', 'energized'
- `energy` must be a number between 1 and 5
- `note` must be a string (can be empty)
- `date` must be provided

Think about: Where does the data live? (In memory — an array.) How do you generate unique IDs? How do you validate input?

<details>
<summary>See the solution</summary>

Create `server/src/routes/entries.ts`:

```typescript
import { Router, Request, Response } from 'express';
import { MoodEntry, CreateEntryRequest } from '../types';

const router = Router();

// In-memory data store
const entries: MoodEntry[] = [];
let nextId = 1;

// Valid mood values
const VALID_MOODS = ['happy', 'neutral', 'sad', 'frustrated', 'energized'] as const;

// GET /api/entries — Return all entries
router.get('/', (_req: Request, res: Response) => {
  res.json(entries);
});

// POST /api/entries — Create a new entry
router.post('/', (req: Request, res: Response) => {
  const { mood, energy, note, date } = req.body as CreateEntryRequest;

  // Validation
  const errors: string[] = [];

  if (!mood || !VALID_MOODS.includes(mood)) {
    errors.push(`mood must be one of: ${VALID_MOODS.join(', ')}`);
  }

  if (energy === undefined || typeof energy !== 'number' || energy < 1 || energy > 5) {
    errors.push('energy must be a number between 1 and 5');
  }

  if (typeof note !== 'string') {
    errors.push('note must be a string');
  }

  if (!date) {
    errors.push('date is required');
  }

  if (errors.length > 0) {
    res.status(400).json({ errors });
    return;
  }

  // Create entry
  const newEntry: MoodEntry = {
    id: nextId++,
    mood,
    energy,
    note: note || '',
    date,
    createdAt: new Date().toISOString(),
  };

  entries.push(newEntry);
  res.status(201).json(newEntry);
});

export { router as entriesRouter };
```

</details>

**Key concepts in this code:**

| Pattern | What It Does | Why |
|---------|-------------|-----|
| `const entries: MoodEntry[] = []` | In-memory data store | Simple for Level 1. Data disappears when the server restarts. Level 2 adds a real database. |
| `let nextId = 1` | Simple auto-incrementing ID | In production, databases generate IDs. This is a temporary solution. |
| `VALID_MOODS.includes(mood)` | Validates against allowed values | Never trust data from the client. Always validate on the server. |
| `errors: string[]` | Collect all errors, then return them | Better UX: the user sees all problems at once, not one at a time. |
| `res.status(400).json({ errors })` | Return 400 with error details | 400 = "your request was bad" + details about what was wrong. |
| `res.status(201).json(newEntry)` | Return 201 with the created entry | 201 = "created successfully." Return the full object so the frontend doesn't need a second request. |
| `return` after `res.status(400)` | Stop execution after sending error | Without `return`, the code continues and tries to create the entry anyway. |

> ⚠️ **Common Mistake: Not returning after `res.status()`**
> Express doesn't stop execution when you call `res.json()`. If you don't `return`, the code continues running and may try to send a second response, causing: `Error: Cannot set headers after they are sent to the client`.

### 🧠 Debugging Exercise

What would happen if you removed `app.use(express.json())` from `index.ts` and then sent a POST request with a JSON body?

<details>
<summary>Answer</summary>

`req.body` would be `undefined`. The destructuring `const { mood, energy, note, date } = req.body` would throw: `TypeError: Cannot destructure property 'mood' of 'undefined'`. The server would crash with a 500 error. The `express.json()` middleware is what parses the raw request body string into a JavaScript object.

</details>

---

## 4. Test Your API

Start the server:

```bash
cd server
npm run dev
```

You should see: `Server running on http://localhost:3001`

### Test with curl

Open a **new terminal** (keep the server running):

```bash
# Health check
curl http://localhost:3001/api/health
```

Expected: `{"status":"ok","timestamp":"..."}`

```bash
# Create an entry
curl -X POST http://localhost:3001/api/entries \
  -H "Content-Type: application/json" \
  -d '{"mood": "happy", "energy": 4, "note": "Great day!", "date": "2025-01-15"}'
```

Expected: `{"id":1,"mood":"happy","energy":4,"note":"Great day!","date":"2025-01-15","createdAt":"..."}`

```bash
# Get all entries
curl http://localhost:3001/api/entries
```

Expected: Array containing the entry you just created.

```bash
# Test validation (missing mood)
curl -X POST http://localhost:3001/api/entries \
  -H "Content-Type: application/json" \
  -d '{"energy": 3, "note": "test", "date": "2025-01-15"}'
```

Expected: `{"errors":["mood must be one of: happy, neutral, sad, frustrated, energized"]}` with status 400.

### ✅ Checkpoint

- [ ] `GET /api/health` returns `{"status":"ok",...}`
- [ ] `POST /api/entries` with valid data returns the created entry with status 201
- [ ] `POST /api/entries` with invalid data returns errors with status 400
- [ ] `GET /api/entries` returns all created entries

If any of these fail, check:
1. Is `express.json()` middleware added before routes?
2. Are you sending `Content-Type: application/json` header?
3. Is the JSON body valid (double quotes, no trailing commas)?

---

## 5. Commit

```bash
git add .
git commit -m "feat: add Express backend with typed entries API"
```

---

## 🧠 Spatial Check-In

1. Why do we validate data on the server even though we'll also validate on the frontend later?

2. What happens to the entries array when you stop and restart the server? Why is this a problem, and what will we do about it in Level 2?

3. Look at this route handler — what's the bug?
```typescript
router.post('/', (req: Request, res: Response) => {
  const { mood } = req.body;
  if (!mood) {
    res.status(400).json({ error: 'mood required' });
  }
  // Create entry...
  res.status(201).json(newEntry);
});
```

<details>
<summary>Check Your Answers</summary>

1. **Never trust the client.** The frontend runs in the user's browser — they can modify it, bypass validation, or send requests directly (with curl, Postman, or malicious scripts). Server validation is your last line of defense.

2. **The array resets to empty.** In-memory storage is temporary — it only exists while the process runs. This is fine for Level 1, but in Level 2 we'll add PostgreSQL so data survives restarts.

3. **Missing `return` after the error response.** After `res.status(400).json(...)`, execution continues to `res.status(201).json(newEntry)`, causing `Cannot set headers after they are sent`. Fix: add `return;` after the 400 response.

</details>

---

> **Session Break** — You've built the backend API.
> When you return, you'll build the React frontend in [Step 4 — Frontend](../04-frontend/).

---

| | |
|:---|---:|
| [← Step 2: Project Setup](../02-project-setup/) | [Step 4 — Frontend →](../04-frontend/) |
