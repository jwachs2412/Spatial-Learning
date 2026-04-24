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

## Before We Start: A Plain-English Syntax Primer

This is the first step in Level 1 where you write real code. Every snippet in this lesson (and the next) is built from the same small set of JavaScript and TypeScript building blocks. Read through this once so every shape that follows makes sense.

### Variables: `const` and `let`

```typescript
const name = 'DevPulse';   // "name" points at this string forever
let counter = 0;            // "counter" can be reassigned later
```

- `const` = "I will never reassign this name to a different value." Use it by default.
- `let` = "I need to reassign this later (inside a loop, for example)."

### Strings and Template Literals

```typescript
const normal = 'Hello';
const template = `Hello, ${name}!`;   // backticks, with ${...} embedded
```

Backticks (`` ` ``) let you insert a variable directly into a string using `${...}`. Regular quotes (`'` or `"`) do not allow this. You'll see backticks any time we build a string from variables.

### Functions: Declaration and Arrow Form

```typescript
// Named function declaration
function greet(name) {
  return `Hello, ${name}`;
}

// Arrow function — very common when a function is passed as an argument
const greet = (name) => `Hello, ${name}`;
```

Both do the same thing: take inputs, return an output. Arrow functions (`=>`) are compact and show up constantly in Express handlers.

### Objects and Arrays

```typescript
const user = { name: 'Jo', age: 30 };    // an object — named fields
const list = ['a', 'b', 'c'];             // an array — ordered list
```

- Read an object's field: `user.name` → `'Jo'`.
- Read an array item by index: `list[0]` → `'a'` (indexes start at 0).

### Destructuring: Pulling Fields Out

```typescript
const { mood, energy, note } = req.body;
```

Instead of writing `const mood = req.body.mood; const energy = req.body.energy; ...`, destructuring pulls multiple fields into their own named variables in one line. You'll see this everywhere.

### `import` and `export`

```typescript
// In file `math.ts`:
export function add(a, b) { return a + b; }   // named export
export default function multiply(a, b) { return a * b; }  // default export

// In another file:
import multiply, { add } from './math';        // default + named
```

- **Named exports** (with curly braces when imported) — many per file.
- **Default export** (no curly braces when imported) — one per file.

### TypeScript in 30 Seconds

TypeScript is JavaScript with **types** — labels that say what kind of value a variable or parameter holds. TypeScript checks these at compile time and refuses to build if they don't match, catching bugs before they ship.

```typescript
function greet(name: string): string {
  return `Hello, ${name}`;
}

const count: number = 5;
const names: string[] = ['Jo', 'Sam'];
```

You'll also meet:
- `interface` — a blueprint describing the shape of an object.
- Union types like `'happy' | 'sad'` — "the value must be exactly one of these strings."
- `Omit<X, 'a' | 'b'>` — "the type X but with these fields removed."

### Express Request/Response/Next

Every Express route handler gets three arguments:

- `req` — the incoming request (URL, query params, body, headers).
- `res` — the response you send back (status code, JSON body).
- `next` — a function you call to pass control to the next middleware (rarely used in Level 1).

```typescript
app.get('/hello', (req, res) => {
  res.json({ greeting: 'hi' });
});
```

That's your foundation. From here, every code block has a "Reading This File Line-by-Line" walkthrough below it.

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

### Reading This File Line-by-Line

This file defines two TypeScript types. Let's unpack every piece.

- `export interface MoodEntry { ... }`
  - `export` means "make this available to other files via `import`." Without `export`, the interface only exists inside this file.
  - `interface` is TypeScript's keyword for defining the shape of an object. It doesn't create anything at runtime — it only describes what a `MoodEntry` must look like.
  - `MoodEntry` is the name we chose. By convention, type and interface names are **PascalCase** (each word capitalized).
  - Everything between `{` and `}` is a list of fields.

- `id: number;`
  - Field named `id`, of type `number`. The semicolon ends the field declaration.
- `mood: 'happy' | 'neutral' | 'sad' | 'frustrated' | 'energized';`
  - This is a **union type**. `|` means "or." Read it as "the mood must be exactly one of these five strings."
  - Using specific string values instead of just `string` turns TypeScript into your safety net: try to assign `mood: 'ecstatic'` and the compiler rejects it.
- `energy: number;` — a number. We'll enforce the 1–5 range later, in the route handler.
- `note: string;` — a string. `note: ''` (empty string) is valid.
- `date: string;` — a string, not a Date. JSON has no Date type, so dates travel over HTTP as strings. We agree by convention to use ISO format `"YYYY-MM-DD"`.
- `createdAt: string;` — same reasoning. An ISO timestamp like `"2025-01-15T14:32:01.000Z"`.
- `// comments` — anything after `//` on a line is ignored by TypeScript. We use them to document units and formats.

Now the second declaration:

- `export type CreateEntryRequest = Omit<MoodEntry, 'id' | 'createdAt'>;`
  - `export type` creates a named type alias — like `interface`, but more general (it can name unions, tuples, intersections, etc., not just object shapes).
  - `=` gives it the value on the right.
  - `Omit<MoodEntry, 'id' | 'createdAt'>` is a **built-in utility type**. The `< >` angle brackets are how TypeScript passes types to other types (like function arguments, but for types).
  - `Omit` takes two type arguments: the source type (`MoodEntry`) and a union of keys to remove (`'id' | 'createdAt'`).
  - The result is a type identical to `MoodEntry` but without those two fields — exactly what we need for incoming POST requests, where the client provides mood/energy/note/date and the server generates the `id` and `createdAt`.

Why bother with `Omit` instead of writing a second interface by hand? Because if you later add a field to `MoodEntry`, `CreateEntryRequest` updates automatically. One source of truth.

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

### Reading This File Line-by-Line

```typescript
import express, { Request, Response } from 'express';
import cors from 'cors';
import { entriesRouter } from './routes/entries';
```

- `import express, { Request, Response } from 'express';`
  - This line does two imports at once. `express` (no braces) is the **default export** of the express package — the main function you call to create an app. `{ Request, Response }` are **named exports** — TypeScript types we'll use to annotate handler parameters.
- `import cors from 'cors';` — default import of the cors package.
- `import { entriesRouter } from './routes/entries';` — named import from our own file at `./routes/entries.ts` (we'll build it in a moment). The `./` at the start means "this is a relative path starting in the current folder." Without `./`, Node would look for a package in `node_modules/`.

```typescript
const app = express();
const PORT = process.env.PORT || 3001;
```

- `const app = express();` — call the `express` function we imported. It returns a fresh application object. Store it as `app`.
- `const PORT = process.env.PORT || 3001;`
  - `process.env` is a built-in Node object containing every environment variable. `process.env.PORT` reads the one named `PORT`.
  - `||` is the **logical OR operator**. Read it as "use the left value if it's truthy; otherwise use the right value." So: "use `process.env.PORT` if it's set, otherwise fall back to `3001`." Deployment platforms (Render, Vercel, Heroku) set `PORT` automatically; locally it's undefined, so we use `3001`.

```typescript
app.use(express.json());
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
}));
```

- `app.use(middleware)` registers middleware that runs on **every** incoming request, in the order you call `app.use`.
- `express.json()` is a call that returns a middleware function. It parses incoming JSON request bodies and attaches the parsed object to `req.body`. Without it, `req.body` is `undefined`.
- `cors({ origin: ... })` returns a middleware that adds the CORS headers we discussed in Step 1. The `origin` field tells it which frontend URL to allow.

```typescript
app.use('/api/entries', entriesRouter);
```

A special form of `app.use` with two arguments: a **path prefix** and a **router**. This says "any request starting with `/api/entries` should be handed to `entriesRouter`." Inside the router, paths are written relative to this prefix.

```typescript
app.get('/api/health', (_req: Request, res: Response) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
  });
});
```

- `app.get(path, handler)` registers a GET handler for the given path.
- `(_req: Request, res: Response) => { ... }` is an arrow function — the handler. `_req` and `res` are typed with the `Request` and `Response` types we imported. The underscore prefix on `_req` is a convention that says "I'm declaring this parameter but not using it." Some linters will warn about unused parameters without the underscore.
- `res.json({...})` sends a JSON response with status 200 (the default). The object is serialized automatically.
- `new Date().toISOString()` — create a new Date object for "right now," then format it as an ISO string like `"2025-01-15T14:32:01.000Z"`.

```typescript
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

- `app.listen(port, callback)` — start the HTTP server on the given port. Express begins listening for requests.
- The callback runs **once**, as soon as the server is ready. We log a message so you know it started successfully.
- Backticks around the string let us embed `${PORT}` — template-literal syntax from the primer.

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

### Reading This File Line-by-Line

```typescript
import { Router, Request, Response } from 'express';
import { MoodEntry, CreateEntryRequest } from '../types';
```

- `import { Router, Request, Response } from 'express';` — three named imports. `Router` is the factory for creating routers; `Request` and `Response` are types.
- `import { MoodEntry, CreateEntryRequest } from '../types';` — imports from our own `types/index.ts`. The `../` means "go up one folder" (from `routes/` back up to `src/`, then down into `types/`). Node and TypeScript automatically read `types/index.ts` when you import from `../types`.

```typescript
const router = Router();
```

Call the `Router` factory — no `new` — and store the returned router. This is a mini-app that will hold our entry-related routes.

```typescript
const entries: MoodEntry[] = [];
let nextId = 1;
```

- `const entries: MoodEntry[] = []` — declare an empty array. The type annotation `MoodEntry[]` says "this is an array of `MoodEntry` objects." The `const` is about the variable, not the array: you can still `.push(...)` items into the array, you just can't reassign `entries` to a different array.
- `let nextId = 1` — a plain counter. We use `let` because we'll increment it every time we create an entry.

**Important caveat:** these two variables live **only in memory**. When the server restarts, they reset. That's fine for Level 1; Level 2 swaps this for a real PostgreSQL database.

```typescript
const VALID_MOODS = ['happy', 'neutral', 'sad', 'frustrated', 'energized'] as const;
```

- `['happy', ...]` — an array literal.
- `as const` — a TypeScript **const assertion**. Normally TypeScript would infer the type as `string[]` (any strings). `as const` locks the values in place, so the type becomes `readonly ['happy', 'neutral', 'sad', 'frustrated', 'energized']`. Each element is typed as its exact literal, not just "a string."
- Why? It means `VALID_MOODS.includes(mood)` can narrow `mood` to the union type we defined — strict type safety.

```typescript
router.get('/', (_req: Request, res: Response) => {
  res.json(entries);
});
```

Register a GET handler at `/`. Remember, the router is mounted at `/api/entries` in `index.ts`, so this handler responds to `GET /api/entries`. It simply sends the `entries` array as JSON.

```typescript
router.post('/', (req: Request, res: Response) => {
  const { mood, energy, note, date } = req.body as CreateEntryRequest;
```

- `router.post('/', handler)` registers a POST handler at the same path. (GET and POST on the same URL are different endpoints, distinguished by HTTP method.)
- `const { mood, energy, note, date } = req.body as CreateEntryRequest;`
  - **Destructuring**: pull four fields from `req.body` into individual variables.
  - `as CreateEntryRequest` is a **type assertion** — we're telling TypeScript "trust me, `req.body` has this shape." This is necessary because Express doesn't know the exact shape of incoming data. We immediately validate afterward; if validation fails, the request is rejected before we use the values.

```typescript
const errors: string[] = [];

if (!mood || !VALID_MOODS.includes(mood)) {
  errors.push(`mood must be one of: ${VALID_MOODS.join(', ')}`);
}
```

- `const errors: string[] = []` — an array that will accumulate every problem we find. We gather them all so the user sees every issue at once, not one fix-and-retry at a time.
- `if (!mood || !VALID_MOODS.includes(mood))` — two conditions joined with `||`. `!` means "not."
  - `!mood` is true when `mood` is missing (undefined, null, or empty string).
  - `!VALID_MOODS.includes(mood)` is true when the provided mood isn't in our allowlist.
- `errors.push(...)` — add a new item to the end of the array.
- `${VALID_MOODS.join(', ')}` — `.join(separator)` turns an array into a single string with the separator between items. Produces `"happy, neutral, sad, frustrated, energized"`.

```typescript
if (energy === undefined || typeof energy !== 'number' || energy < 1 || energy > 5) {
  errors.push('energy must be a number between 1 and 5');
}
```

- `energy === undefined` — strict equality check for "the field wasn't provided."
- `typeof energy !== 'number'` — `typeof x` returns a string describing the type (`'number'`, `'string'`, `'object'`, etc.). This catches someone sending `"5"` (a string) instead of `5`.
- `energy < 1 || energy > 5` — range check.

```typescript
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
```

- Each `if` pushes a message onto `errors` when a check fails.
- After all checks, if `errors` has anything in it, we send a 400 Bad Request with the full list, then `return` to stop the function. Without `return`, execution would continue to the "Create entry" section and we'd try to send a second response.
- `{ errors }` is shorthand for `{ errors: errors }` — when a key and value have the same name, you can write it once.

```typescript
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
```

- `const newEntry: MoodEntry = { ... }` — build a new object; TypeScript will verify it matches `MoodEntry` exactly (no missing fields, no extras).
- `id: nextId++` — `++` is the **post-increment operator**. It returns the current value of `nextId`, then increments it. So `nextId` starts at 1: first entry gets `id: 1`, after which `nextId` becomes 2, next entry gets 2, and so on.
- `mood, energy, date` — shorthand property names again. Equivalent to `mood: mood, energy: energy, date: date`.
- `note: note || ''` — fall back to an empty string if `note` is undefined or empty.
- `createdAt: new Date().toISOString()` — generate the current timestamp in ISO format.
- `entries.push(newEntry)` — add to the in-memory array.
- `res.status(201).json(newEntry)` — `201 Created` is the correct HTTP status for "you POSTed and I made something new." We return the full new entry so the frontend can display it without a follow-up fetch.

```typescript
export { router as entriesRouter };
```

A named export with a rename. Locally, the router is called `router`; to the outside world, we export it as `entriesRouter`. This matches the import in `index.ts`: `import { entriesRouter } from './routes/entries'`.

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
