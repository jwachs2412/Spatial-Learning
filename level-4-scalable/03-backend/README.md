`Level 4` **Step 3 of 9** — Backend

# 03 — Backend: Middleware Stack and Analytics API

## What This Lesson Teaches

In Level 2 you built a CRUD backend: receive a request, run a query, send a response. That works for small apps. It does not work for apps that:

- Serve many users (you need **rate limiting**)
- Need to be debugged in production (you need **structured logging**)
- Aggregate large data sets (you need a **service layer** so routes stay readable)
- Reject bad input early (you need **validation middleware**)

This lesson is where you stop writing "scripts that respond to HTTP" and start writing **layered backends**. You will think like a backend engineer about *where* a concern belongs, not just *how* to satisfy the immediate request.

---

## Spatial Orientation

You are building three new layers on top of the Level 2 pattern: a **middleware stack**, a **service layer**, and a **seed script**.

```
REQUEST FLOW:

  Client Request
       │
       ▼
  ┌──────────────┐
  │   Logger     │  ← Records every request
  ├──────────────┤
  │ Rate Limiter │  ← Blocks excessive requests
  ├──────────────┤
  │    CORS      │  ← Allows frontend origin
  ├──────────────┤
  │ JSON Parser  │  ← Parses request body
  ├──────────────┤
  │  Validator   │  ← Validates query parameters
  ├──────────────┤
  │   Route      │  ← Calls service layer
  │   Handler    │
  ├──────────────┤
  │   Service    │  ← Executes SQL, returns data
  ├──────────────┤
  │   Error      │  ← Catches and formats errors
  │   Handler    │
  └──────────────┘
       │
       ▼
  JSON Response
```

```
data-dash/
└── server/
    └── src/
        ├── db/
        │   ├── schema.sql      ← Database structure
        │   ├── index.ts        ← Connection pool
        │   └── seed.ts         ← Generate fake data
        ├── middleware/
        │   ├── logger.ts       ← Structured logging
        │   ├── rateLimiter.ts  ← Throttle requests
        │   ├── validator.ts    ← Reject bad input
        │   └── errorHandler.ts ← Catch all errors
        ├── services/
        │   └── analyticsService.ts  ← All SQL lives here
        ├── routes/
        │   └── analytics.ts    ← HTTP layer (no SQL)
        ├── types/
        │   └── index.ts        ← Shared types
        └── index.ts            ← Wires everything together
```

> **Mental Model: The Pipeline**
> Every request enters at the top and falls through a pipeline of middleware. Each layer can do one of three things: **inspect** (logger), **reject** (rate limiter, validator), or **enrich** (parser). If nothing rejects the request, it reaches the route handler. If anything throws, the error handler at the bottom catches it. Routes themselves stay short because everything around them is handled by the layers.

---

## Before We Start: A Plain-English Syntax Primer

Pretend you have never written code before. Every snippet in this lesson is built from the same small set of JavaScript and TypeScript building blocks. Read through this once. When you see these shapes later, you will know exactly what they mean instead of glossing over them.

### Variables: `const` and `let`

```typescript
const name = 'DataDash';   // "name" points at this string forever
let counter = 0;            // "counter" can be reassigned later
```

- `const` = "I will never reassign this name to a different value." Use it by default.
- `let` = "I need to reassign this later (inside a loop, for example)."
- There is also `var` — ignore it. Modern code uses `const` and `let`.

### Strings and Template Literals

```typescript
const normal = 'Hello';
const template = `Hello, ${name}!`;   // backticks, with ${...} embedded
```

Backticks (`` ` ``) let you insert a variable or expression directly into a string using `${...}`. The old way was `'Hello, ' + name + '!'`. Template literals are cleaner and used everywhere in this lesson.

### Functions: Three Forms You Will See

```typescript
// 1. Classic function declaration
function add(a, b) {
  return a + b;
}

// 2. Arrow function assigned to a variable
const add = (a, b) => a + b;

// 3. Anonymous arrow passed directly as an argument
array.map((item) => item * 2);
```

All three do the same kind of thing: take inputs, return an output. Arrow functions (`=>`) are shorter and very common when a function is passed as an argument.

### Arrays and Objects

```typescript
const list = ['a', 'b', 'c'];           // an array — ordered list
const user = { name: 'Jo', age: 30 };   // an object — named fields
```

- Access array items by index: `list[0]` → `'a'`.
- Access object fields by name: `user.name` → `'Jo'`.

### Destructuring: Pulling Fields Out

```typescript
const { name, age } = user;             // same as name = user.name; age = user.age
const [first, second] = list;           // same as first = list[0]; second = list[1]
```

When you see `const { startDate, endDate } = req.query`, that is just "pull `startDate` and `endDate` out of `req.query` and give them their own names."

### The Spread Operator: `...`

```typescript
const more = [...list, 'd'];            // copies list and appends 'd'
const extended = { ...user, role: 'admin' };  // copies user, adds role
```

`...` copies everything from one array/object into another. In this lesson you will see `[...params, limit, offset]` — that means "take the existing `params` array and append `limit` and `offset` at the end."

### `async` / `await` and Promises

A **promise** represents "a value that will arrive later" — usually because it depends on the database, the network, or the disk. Any function marked `async` automatically returns a promise. Inside that function, `await` pauses execution until a promise resolves.

```typescript
async function getUser(id) {
  const user = await pool.query(...);   // pause here until the DB answers
  return user;                           // this becomes the promise's value
}
```

Without `await`, you would get a promise object, not the actual result. Almost every database, HTTP, or file operation in Node.js returns a promise, so you will see `await` constantly.

### Modules: `import` and `export`

```typescript
// In file `math.ts`:
export function add(a, b) { return a + b; }   // named export
export default function multiply(a, b) { return a * b; }  // default export

// In another file:
import multiply, { add } from './math';        // default + named
import * as math from './math';                 // everything as a namespace
```

- **Named exports**: many per file, imported with curly braces: `import { add }`.
- **Default export**: one per file, imported without curly braces: `import multiply`.
- **Namespace import** (`* as`): grabs everything and lets you call `math.add(...)`.

### TypeScript in 30 Seconds

TypeScript is JavaScript with **types**. A type is a label that says "this value is a string" or "this function returns a number." TypeScript checks your types at build time and refuses to compile if they don't match — catching bugs before they ship.

```typescript
function greet(name: string): string {          // takes a string, returns a string
  return `Hello, ${name}`;
}

const count: number = 5;
const names: string[] = ['Jo', 'Sam'];           // array of strings
const user: { name: string; age: number } = { name: 'Jo', age: 30 };
```

You will also see:

- `: void` — "this function returns nothing."
- `Promise<X>` — "a promise that will resolve to a value of type X."
- `T | null` — "either a T or null."
- `<T>` on a function — **generic**, meaning "works with whatever type you pass in."

Don't panic if types feel noisy at first. They are comments that the compiler actually checks.

### Request, Response, Next (Express basics)

Every Express middleware and route handler gets three arguments:

- `req` — the incoming request (URL, query params, body, headers).
- `res` — the response you send back (status code, JSON body).
- `next` — a function you call to say "I'm done; hand the request to the next middleware."

```typescript
function myMiddleware(req, res, next) {
  // inspect or modify req/res
  next();   // pass along
}
```

That's enough foundation. From here on, every code block will be followed by a plain-English walkthrough that explains what each piece is actually doing.

---

## 1. Database Schema

> **Key Concept: Schema as Contract**
> A schema is a contract: it tells PostgreSQL exactly what shape your data must take, *and* it tells future-you (and your team) what fields exist. Once a column is `NOT NULL`, the database refuses any row missing that field. This pushes correctness down into the data layer — bugs that would be allowed in JavaScript become impossible at the database level.

### 🏗️ Your Turn

Before looking at the code, design the table. You're storing **analytics events** — every click, page view, signup, and purchase. Think:

- What fields do you need to answer "how many signups last week from US mobile users?"
- Which fields should be `NOT NULL`?
- What types make sense — should `value` (revenue) be `INTEGER` or `DECIMAL`?
- Which columns will you filter on most often? (Hint: those are candidates for indexes.)

Sketch the columns and types on paper, then check the solution.

<details>
<summary>See the solution</summary>

Create `server/src/db/schema.sql`:

```sql
-- DataDash analytics schema
-- One table stores all events. API aggregates with SQL.

DROP TABLE IF EXISTS analytics_events;

CREATE TABLE analytics_events (
  id              SERIAL PRIMARY KEY,
  event_type      VARCHAR(50)   NOT NULL,
  category        VARCHAR(50)   NOT NULL,
  value           DECIMAL(10,2) DEFAULT 0,
  country         VARCHAR(2)    NOT NULL,
  device          VARCHAR(20)   NOT NULL,
  browser         VARCHAR(30)   NOT NULL,
  page            VARCHAR(200)  NOT NULL,
  session_id      VARCHAR(36)   NOT NULL,
  created_at      TIMESTAMP     NOT NULL DEFAULT NOW()
);

-- Indexes for common query patterns
CREATE INDEX idx_events_type       ON analytics_events(event_type);
CREATE INDEX idx_events_category   ON analytics_events(category);
CREATE INDEX idx_events_device     ON analytics_events(device);
CREATE INDEX idx_events_created_at ON analytics_events(created_at);
```

</details>

### Reading This SQL Line-by-Line

SQL (Structured Query Language) is the language you speak to a relational database. It is almost plain English. Here is what every piece above is saying:

- `-- anything after two dashes is a comment.` PostgreSQL ignores comments; they are only for humans reading the file.
- `DROP TABLE IF EXISTS analytics_events;` — "If a table named `analytics_events` already exists, delete it." The `IF EXISTS` part prevents an error the first time you run this (when the table doesn't exist yet).
- `CREATE TABLE analytics_events ( ... );` — "Make a new table called `analytics_events` with these columns." Everything inside the parentheses is the list of columns and their rules.
- Each column is written as `column_name TYPE CONSTRAINTS`:
  - `id SERIAL PRIMARY KEY` — a column named `id` of type `SERIAL` (auto-incrementing integer). `PRIMARY KEY` means "this is the unique identifier for each row; PostgreSQL should enforce uniqueness and build an index on it automatically."
  - `event_type VARCHAR(50) NOT NULL` — a variable-length string up to 50 characters. `NOT NULL` means "every row must have a value; inserting a row without this field is an error."
  - `value DECIMAL(10,2) DEFAULT 0` — a decimal number with up to 10 digits total, 2 of which come after the decimal point. `DEFAULT 0` means "if the insert doesn't specify a value, use 0."
  - `created_at TIMESTAMP NOT NULL DEFAULT NOW()` — a timestamp column. `NOW()` is a built-in PostgreSQL function that returns the current time. `DEFAULT NOW()` means "if no value is given, stamp the current time automatically."
- `CREATE INDEX idx_events_type ON analytics_events(event_type);` — "Build an index on the `event_type` column so filtering by it is fast." The name `idx_events_type` is just a label; by convention it is `idx_<table>_<column>` so you can spot indexes at a glance.

Every semicolon (`;`) ends a statement. PostgreSQL runs them one by one.

**Why these choices matter:**

| Column | Type | Why this type and not another |
|--------|------|-------------------------------|
| `id` | `SERIAL PRIMARY KEY` | `SERIAL` auto-generates an integer that increases on every insert. `PRIMARY KEY` makes it unique and indexed automatically. You never set this column manually. |
| `event_type` | `VARCHAR(50)` | Short, bounded string. We could use an `ENUM`, but `VARCHAR` makes it easier to add new event types later without a migration. |
| `value` | `DECIMAL(10,2)` | Money. **Never use `FLOAT` for money** — floating-point arithmetic introduces rounding errors (`0.1 + 0.2 !== 0.3`). `DECIMAL(10,2)` means up to 10 total digits with 2 after the decimal point. |
| `country` | `VARCHAR(2)` | ISO country codes are exactly 2 characters. Bounding the type prevents garbage like `"United States"` from creeping in. |
| `created_at` | `TIMESTAMP NOT NULL DEFAULT NOW()` | Captured automatically when the row is inserted. The `DEFAULT NOW()` means you never forget to set it. |

> **Key Concept: Indexes**
> **Technical:** An index is a separate data structure (typically a B-tree) that lets the database find rows matching a column value in `O(log n)` time instead of `O(n)`. Without an index, a query like `WHERE created_at >= '2025-01-01'` scans every row in the table — a *full table scan*. With an index, the database walks the tree directly to matching rows.
>
> **Plain English:** An index is like the index in the back of a textbook. To find every page that mentions "middleware," you don't read the whole book — you flip to the index, find "middleware → 47, 89, 122," and jump straight there. Indexes make lookups fast but cost extra disk space and slow down writes (every insert has to update the index too).

> ⚠️ **Common Mistake: Indexing Every Column**
> Beginners often think "more indexes = faster." It is the opposite past a point. Each index slows down `INSERT`, `UPDATE`, and `DELETE` because the index has to be updated. Index only the columns you frequently filter or sort on. Here we index `event_type`, `category`, `device`, and `created_at` because those drive every analytics query.

### Why One Denormalized Table?

In a "real" production analytics system you would likely split this into multiple tables (events, sessions, users) or use a columnar database (BigQuery, ClickHouse) and pre-aggregate daily summaries. We are deliberately using one wide table because it teaches:

1. **SQL aggregation** — `GROUP BY`, `COUNT`, `SUM`, `AVG`, `FILTER`
2. **Filtering** — composing `WHERE` clauses dynamically
3. **Index design** — feeling the difference indexed columns make on real queries

Run the schema:

```bash
psql datadash < server/src/db/schema.sql
```

Expected output: lines saying `DROP TABLE`, `CREATE TABLE`, then four `CREATE INDEX` lines.

---

## 2. Database Connection

> **Key Concept: Connection Pools**
> **Technical:** Opening a TCP connection to PostgreSQL takes ~10–50ms (handshake + auth). If every request opens its own connection, your API tops out at maybe 20 requests per second per connection cycle. A **pool** keeps a small number of connections (default 10) open and reuses them. A request "checks out" a connection, runs its query, and returns it. The pool also prevents you from overwhelming PostgreSQL — Postgres has a hard connection limit (often 100) and crashes if you exceed it.
>
> **Plain English:** Imagine a pizza shop. Without a pool, every customer brings their own oven, uses it once, and throws it away. With a pool, the shop has 10 ovens shared by everyone — fast, cheap, and you never run out of ovens.

### 🏗️ Your Turn

Predict: what would happen if you used `new Client()` (single, throwaway connection) inside every route handler instead of a pool? Think about latency, concurrency, and what happens at 100 concurrent users.

<details>
<summary>See the answer</summary>

Three things break:

1. **Latency** — Every request pays the 10–50ms handshake cost before any SQL runs.
2. **Concurrency** — Connections build up faster than they close. PostgreSQL hits its `max_connections` limit (often 100) and starts rejecting new connections with `FATAL: too many clients`.
3. **Leaks** — If you forget `client.end()` in any code path (especially error paths), connections stay open until idle timeout. The pool fixes this — connections are returned automatically.

</details>

Create `server/src/db/index.ts`:

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config({ path: '../.env' });

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export default pool;
```

### Reading This File Line-by-Line

Even though it's only a few lines, every symbol here is meaningful. Let's unpack each piece:

- `import { Pool } from 'pg';`
  - `import` = "I want to use something from another file or package."
  - `{ Pool }` = "Give me the `Pool` export by name." (curly braces mean **named import**.)
  - `from 'pg'` = "Get it from the npm package called `pg`" — the PostgreSQL driver for Node.js.
  - Think of `Pool` as a **class** — a blueprint for making pool objects. You'll call `new Pool(...)` below to actually build one.

- `import dotenv from 'dotenv';`
  - No curly braces, so this is a **default import**. `dotenv` is the package's default export.
  - `dotenv` is a tool for loading environment variables from a `.env` file into `process.env`.

- `dotenv.config({ path: '../.env' });`
  - `dotenv.config(...)` is a function call. You're asking the `dotenv` library to read the file and populate `process.env`.
  - `{ path: '../.env' }` is an **object literal** passed as the argument — an object with one field, `path`, pointing two levels up (because the script runs from `server/`, but `.env` lives in the project root).
  - After this line runs, `process.env.DATABASE_URL` contains the value from the `.env` file.

- `const pool = new Pool({ connectionString: process.env.DATABASE_URL });`
  - `const pool` = "create a constant named `pool`" that we'll never reassign.
  - `new Pool(...)` = "build a new Pool object from the `Pool` class." `new` is how you create an instance of a class.
  - The argument is a configuration object. `connectionString` is one of many options `Pool` accepts; we're telling it, "here's the URL of the database — parse host, port, username, password, and database name out of it."
  - `process.env` is a built-in Node object containing all environment variables. `process.env.DATABASE_URL` reads the one named `DATABASE_URL`.

- `export default pool;`
  - "Make `pool` the default export of this file."
  - Any other file can now write `import pool from './db/index'` (no curly braces — remember, no braces = default import) and get this exact same pool object.
  - There's **one pool for the whole app**, shared by every service and route. This is crucial — if each file made its own pool, you'd have many disconnected pools competing for PostgreSQL's connection limit.

### Why This Matters

A single shared pool is a classic **singleton** pattern: one instance, reused everywhere. The moment you import `pool` in another file, Node's module system gives you back the exact same object — it doesn't run this file again. So `pool.query(...)` calls from `services/analyticsService.ts` and `routes/analytics.ts` all funnel through the same connection pool.

---

## 3. Seed Script

> **Key Concept: Seed Scripts**
> **Technical:** A seed script populates the database with realistic sample data. It is run once after a fresh schema, separately from migrations. Seeds let you develop and test against meaningful data without waiting for real users.
>
> **Plain English:** Building a dashboard with no data is like designing a stadium with no fans — you cannot see how it feels under load. Seeds give you a populated stadium to design against.

### Why Realistic Distributions?

If we generated 90 days of perfectly uniform data — exactly 50 events per day, evenly split across event types — the dashboard would look fake and would not stress-test our queries. Real analytics data is **skewed**:

- Way more page views than purchases
- Weekends quieter than weekdays
- Mobile and desktop dominate; tablet is rare
- Most countries are quiet; a few dominate

Our seed script encodes these realistic distributions using **weighted random selection**. This is a small example of a deeper engineering habit: **make your test data look like your production data, or you will be surprised in production**.

### 🏗️ Your Turn

Before writing the script, sketch the structure on paper:

1. What helper functions will you need? (Hint: random integer, random item from array, weighted random.)
2. What is the outer loop iterating over? (Days? Events?)
3. How will you insert thousands of rows efficiently? (Hint: not one `INSERT` per row.)

<details>
<summary>See the structure (just the shape, not the code)</summary>

```
1. Constants (event types, categories, etc.)
2. Helper functions (randomItem, randomInt, weighted random)
3. Outer loop: for each day in the last 90 days
   - Decide how many events today (fewer on weekends)
   - Inner loop: for each event today
     - Pick weighted-random event type, device, etc.
     - Add to a batch
   - Batch insert all events for this day in one query
4. Print summary (counts grouped by event type)
```

The key insight: **batch inserts are much faster than one-at-a-time inserts**. A single `INSERT ... VALUES (...), (...), (...)` is faster than 50 individual `INSERT` statements because each statement has network round-trip overhead.

</details>

Create `server/src/db/seed.ts`:

```typescript
import pool from './index';

// --- Configuration ---

const DAYS = 90;
const EVENTS_PER_DAY_MIN = 40;
const EVENTS_PER_DAY_MAX = 80;

const EVENT_TYPES = ['page_view', 'signup', 'purchase', 'click'];
const CATEGORIES = ['marketing', 'product', 'support', 'engineering'];
const COUNTRIES = ['US', 'UK', 'DE', 'FR', 'JP', 'AU', 'CA', 'BR'];
const DEVICES = ['desktop', 'mobile', 'tablet'];
const BROWSERS = ['Chrome', 'Firefox', 'Safari', 'Edge'];
const PAGES = ['/home', '/pricing', '/dashboard', '/docs', '/blog', '/about', '/contact', '/signup'];

// --- Helpers ---

function randomItem<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)];
}

function randomInt(min: number, max: number): number {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

function generateUUID(): string {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
    const r = Math.random() * 16 | 0;
    const v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}

// --- Weighted random for realistic distributions ---

function weightedEventType(): string {
  const rand = Math.random();
  if (rand < 0.55) return 'page_view';    // 55% page views
  if (rand < 0.75) return 'click';         // 20% clicks
  if (rand < 0.90) return 'signup';        // 15% signups
  return 'purchase';                        // 10% purchases
}

function weightedDevice(): string {
  const rand = Math.random();
  if (rand < 0.62) return 'desktop';       // 62% desktop
  if (rand < 0.93) return 'mobile';        // 31% mobile
  return 'tablet';                          // 7% tablet
}

function purchaseValue(): number {
  // Random purchase between $9.99 and $299.99
  const values = [9.99, 19.99, 29.99, 49.99, 99.99, 149.99, 199.99, 299.99];
  return randomItem(values);
}

// --- Main seed function ---

async function seed(): Promise<void> {
  console.log('Seeding database...');
  console.log(`Generating ${DAYS} days of analytics data.\n`);

  // Clear existing data
  await pool.query('DELETE FROM analytics_events');

  const now = new Date();
  let totalEvents = 0;

  for (let day = DAYS - 1; day >= 0; day--) {
    const date = new Date(now);
    date.setDate(date.getDate() - day);

    // Weekend days have fewer events
    const isWeekend = date.getDay() === 0 || date.getDay() === 6;
    const multiplier = isWeekend ? 0.6 : 1;
    const eventsToday = Math.round(
      randomInt(EVENTS_PER_DAY_MIN, EVENTS_PER_DAY_MAX) * multiplier
    );

    const values: string[] = [];
    const params: (string | number)[] = [];
    let paramIndex = 1;

    for (let i = 0; i < eventsToday; i++) {
      const eventType = weightedEventType();
      const value = eventType === 'purchase' ? purchaseValue() : 0;

      // Random time within the day (weighted toward business hours)
      const hour = randomInt(6, 23);
      const minute = randomInt(0, 59);
      const eventDate = new Date(date);
      eventDate.setHours(hour, minute, randomInt(0, 59));

      values.push(
        `($${paramIndex}, $${paramIndex + 1}, $${paramIndex + 2}, $${paramIndex + 3}, $${paramIndex + 4}, $${paramIndex + 5}, $${paramIndex + 6}, $${paramIndex + 7}, $${paramIndex + 8})`
      );

      params.push(
        eventType,
        randomItem(CATEGORIES),
        value,
        randomItem(COUNTRIES),
        weightedDevice(),
        randomItem(BROWSERS),
        randomItem(PAGES),
        generateUUID(),
        eventDate.toISOString()
      );

      paramIndex += 9;
    }

    // Batch insert all events for this day
    if (values.length > 0) {
      await pool.query(
        `INSERT INTO analytics_events
          (event_type, category, value, country, device, browser, page, session_id, created_at)
         VALUES ${values.join(', ')}`,
        params
      );
    }

    totalEvents += eventsToday;
  }

  console.log(`Seeded ${totalEvents} events across ${DAYS} days.`);

  // Print summary
  const summary = await pool.query(`
    SELECT
      event_type,
      COUNT(*) as count,
      ROUND(SUM(value)::numeric, 2) as total_value
    FROM analytics_events
    GROUP BY event_type
    ORDER BY count DESC
  `);

  console.log('\nEvent summary:');
  console.table(summary.rows);

  await pool.end();
  console.log('\nDone!');
}

seed().catch((err) => {
  console.error('Seed failed:', err);
  process.exit(1);
});
```

### Reading This Script Line-by-Line

This is a lot of code. We'll walk through it in five chunks — **Imports**, **Constants**, **Helpers**, **The Main Loop**, and **Cleanup**. Take your time on each one.

#### Chunk 1: Imports

```typescript
import pool from './index';
```

- `import pool` — no curly braces, so this is a **default import** of whatever `./index.ts` exported with `export default`. That was the pool we just built.
- `from './index'` — relative path to a sibling file in the same folder. The `.ts` extension is implied.

After this line, `pool` in this file is the exact same pool object shared with the rest of the app.

#### Chunk 2: Constants

```typescript
const DAYS = 90;
const EVENTS_PER_DAY_MIN = 40;
const EVENTS_PER_DAY_MAX = 80;

const EVENT_TYPES = ['page_view', 'signup', 'purchase', 'click'];
// ...
```

- `const` declares a constant you won't reassign.
- **UPPER_SNAKE_CASE** is a convention for "this is a configuration value that's the same forever" — it signals to readers, "don't change me mid-run."
- `['page_view', ...]` is an **array literal** — square brackets, comma-separated values. This one happens to be an array of strings.

Why pull these out as constants? Two reasons:
1. **One source of truth.** If you want to change the number of days later, you edit one line.
2. **Readability.** `EVENTS_PER_DAY_MAX` at a call site is self-documenting. `80` is not.

#### Chunk 3: Helpers

```typescript
function randomItem<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)];
}
```

Let's unpack the **signature** piece by piece:

- `function randomItem` — declaring a function named `randomItem`.
- `<T>` — this is a **generic type parameter**. Read it as "this function works for any type `T`." When you call `randomItem(['a', 'b'])`, TypeScript figures out `T = string`. When you call `randomItem([1, 2, 3])`, `T = number`. Generics let one function be type-safe for many types without rewriting it.
- `(arr: T[])` — one parameter named `arr`, typed as "an array of `T`." The `T[]` syntax means "array of T."
- `: T` — the function **returns** a single value of type `T` — one element from the array.

Now the body:

- `arr.length` — every array has a `.length` property; `['a', 'b', 'c'].length` is `3`.
- `Math.random()` — built-in function that returns a random decimal between `0` (inclusive) and `1` (exclusive), e.g., `0.5382...`.
- `Math.random() * arr.length` — multiplies that random decimal by the length. For a 3-item array, you get something between `0` and `2.999...`.
- `Math.floor(...)` — rounds **down** to the nearest integer. So `Math.floor(2.7)` is `2`; `Math.floor(0.1)` is `0`.
- `arr[...]` — array index access. With a random floor between `0` and `arr.length - 1`, we get a random item.

```typescript
function randomInt(min: number, max: number): number {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

Same idea, but scaled so you get a random integer between `min` and `max` (both inclusive). The `+1` inside makes `max` reachable; the `+ min` at the end shifts the range up.

```typescript
function generateUUID(): string {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
    const r = Math.random() * 16 | 0;
    const v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}
```

This one is tricky because it combines four new concepts. Let's decode it:

- The string `'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'` is a **template** for a UUID (a universally unique identifier — a standard 36-character ID format). Every `x` and `y` will be replaced with a random hex digit.
- `.replace(/[xy]/g, ...)` — a string method. It finds every character matching the **regular expression** `/[xy]/g` (meaning "any `x` or `y`, globally — all of them") and replaces each with whatever the second argument returns.
- `(c) => { ... }` — an **arrow function** that's called once per matched character. `c` is the character being replaced (`'x'` or `'y'`).
- `Math.random() * 16 | 0` — generate a random number 0–15.999..., then **bitwise OR with 0** (`| 0`) to quickly truncate to an integer. (It's a faster-but-sneakier version of `Math.floor`.)
- `c === 'x' ? r : (r & 0x3 | 0x8)` — the **ternary operator**: `condition ? valueIfTrue : valueIfFalse`. If the character is `'x'`, use the random number as-is. If it's `'y'`, mangle it with bit operations (this is what the UUID spec requires for that position).
- `v.toString(16)` — convert the number to base 16 (hex), giving you one character like `'a'`, `'5'`, or `'f'`.

You don't need to internalize the bitwise magic. The takeaway: this helper produces a random string like `"a1b2c3d4-e5f6-4789-abcd-0123456789ab"`.

```typescript
function weightedEventType(): string {
  const rand = Math.random();
  if (rand < 0.55) return 'page_view';
  if (rand < 0.75) return 'click';
  if (rand < 0.90) return 'signup';
  return 'purchase';
}
```

- Roll a random number `rand` between 0 and 1.
- Cascading `if` statements pick the event type based on where `rand` falls.
- The numbers are **cumulative probabilities**: 55% ≤ page_view, next 20% (0.55→0.75) = click, next 15% (0.75→0.90) = signup, last 10% (0.90→1.0) = purchase.

Notice each branch has `return`, which exits the function immediately. If `rand < 0.55`, we return right there and never check the other `if`s.

#### Chunk 4: The Main Seed Function

```typescript
async function seed(): Promise<void> {
```

- `async function` — this function uses `await` inside, so it must be declared `async`.
- `seed()` — no parameters.
- `: Promise<void>` — returns a promise that resolves to **nothing** (`void` means "no meaningful value"). The function does its work and finishes.

```typescript
await pool.query('DELETE FROM analytics_events');
```

- `pool.query(...)` sends a SQL statement to PostgreSQL and returns a promise.
- `await` pauses here until the query finishes. Without `await`, the next line would run before the delete even started.
- `DELETE FROM analytics_events` — SQL for "delete every row in the table." A clean slate before seeding.

```typescript
const now = new Date();
let totalEvents = 0;
```

- `new Date()` — creates a `Date` object set to the current date and time. This is a built-in JavaScript class.
- `let totalEvents = 0` — we use `let` (not `const`) because we'll reassign with `totalEvents += eventsToday` later.

```typescript
for (let day = DAYS - 1; day >= 0; day--) {
```

A **for loop** in three parts, separated by semicolons:

1. `let day = DAYS - 1` — initialization; we start `day` at `89`.
2. `day >= 0` — condition; keep looping as long as `day` is at least 0.
3. `day--` — step; decrease `day` by 1 after each iteration (shorthand for `day = day - 1`).

Loop counts down: 89, 88, 87, ..., 1, 0. So `day` represents "how many days ago." `day = 0` means today.

```typescript
const date = new Date(now);
date.setDate(date.getDate() - day);
```

- `new Date(now)` — make a **copy** of `now`. If we modified `now` directly, every iteration would drift.
- `date.setDate(date.getDate() - day)` — grab the day-of-month from `date`, subtract `day`, set it back. JavaScript's `Date` automatically handles month/year rollover (subtracting 5 from "March 3" gives you "February 26").

```typescript
const isWeekend = date.getDay() === 0 || date.getDay() === 6;
const multiplier = isWeekend ? 0.6 : 1;
const eventsToday = Math.round(
  randomInt(EVENTS_PER_DAY_MIN, EVENTS_PER_DAY_MAX) * multiplier
);
```

- `date.getDay()` returns `0` for Sunday, `6` for Saturday. `===` is strict equality.
- `||` is logical OR. `isWeekend` is `true` if either check passes.
- `isWeekend ? 0.6 : 1` — the ternary again: 0.6x traffic if weekend, else 1x.
- `Math.round(...)` — round to the nearest integer.

```typescript
const values: string[] = [];
const params: (string | number)[] = [];
let paramIndex = 1;
```

Three **accumulators** we'll fill inside the inner loop:

- `values: string[]` — an array of strings; each string will be a chunk of SQL like `($1, $2, $3, ...)`.
- `params: (string | number)[]` — an array of either strings or numbers. The `|` is a **union type**: "this array contains values that are either strings or numbers, in any mix."
- `paramIndex` — `let` because we'll increment it with each event.

```typescript
for (let i = 0; i < eventsToday; i++) {
```

Standard inner loop: `i` counts up from `0` to `eventsToday - 1`. One iteration per event we're generating.

```typescript
const eventType = weightedEventType();
const value = eventType === 'purchase' ? purchaseValue() : 0;
```

- Pick a weighted event type.
- If it's a purchase, generate a dollar amount; otherwise value is 0. (You can't earn revenue from a page view.)

```typescript
const hour = randomInt(6, 23);
const minute = randomInt(0, 59);
const eventDate = new Date(date);
eventDate.setHours(hour, minute, randomInt(0, 59));
```

- Pick a random hour (6am–11pm, weighted toward business hours by excluding 0–5).
- Copy `date` again (don't mutate the day loop's date).
- `setHours(hour, minute, seconds)` sets hours, minutes, and seconds in one call.

```typescript
values.push(
  `($${paramIndex}, $${paramIndex + 1}, ..., $${paramIndex + 8})`
);
```

- `values.push(x)` adds `x` to the end of the `values` array.
- The backticks make a template literal. `$${paramIndex}` expands to e.g. `$1`, `$2`, ... — these are PostgreSQL's **placeholder syntax** (like `?` in other databases).
- Each event needs 9 placeholders (one per column), so we emit `($1, $2, $3, $4, $5, $6, $7, $8, $9)` for the first event, `($10, ..., $18)` for the second, and so on.

```typescript
params.push(
  eventType,
  randomItem(CATEGORIES),
  // ...
);
```

- `params.push(a, b, c)` — you can push multiple values at once. Each one lands in the next slot.
- The order here **must** match the order of placeholders above. Position is the only thing linking them.

```typescript
paramIndex += 9;
```

Shorthand for `paramIndex = paramIndex + 9`. After each event, jump the placeholder counter by 9 so the next event starts at the right number.

```typescript
if (values.length > 0) {
  await pool.query(
    `INSERT INTO analytics_events
      (event_type, category, value, country, device, browser, page, session_id, created_at)
     VALUES ${values.join(', ')}`,
    params
  );
}
```

- Only run the insert if we actually built something (avoids inserting nothing on empty days).
- `values.join(', ')` — takes the array `['($1, ..., $9)', '($10, ..., $18)', ...]` and turns it into one big string separated by commas. This is how JavaScript turns an array into a single string.
- The final SQL looks like `INSERT INTO ... VALUES ($1, ...), ($10, ...), ($19, ...)` — **one insert, many rows**.
- `pool.query(sql, params)` — the second argument is the values array. The driver sends `sql` and `params` separately, substituting safely on the server side. This is the safe pattern that prevents SQL injection.

```typescript
totalEvents += eventsToday;
```

Shorthand for `totalEvents = totalEvents + eventsToday`. Running tally.

#### Chunk 5: Cleanup

```typescript
const summary = await pool.query(`
  SELECT event_type, COUNT(*) as count, ...
  FROM analytics_events
  GROUP BY event_type
  ORDER BY count DESC
`);

console.log('\nEvent summary:');
console.table(summary.rows);
```

- One last query to aggregate what we just inserted, for confirmation.
- `summary.rows` is the array of result rows — the query driver wraps the result in an object with `.rows`, `.rowCount`, and some metadata.
- `console.table(rows)` — pretty-prints rows as a table in the terminal. Much nicer than `console.log` for tabular data.
- `\n` is a **newline character** in a string, giving a blank line before the table.

```typescript
await pool.end();
```

Gracefully close the pool — otherwise the script hangs because connections stay open. Important for scripts; for a long-running server you'd never call this.

```typescript
seed().catch((err) => {
  console.error('Seed failed:', err);
  process.exit(1);
});
```

- `seed()` returns a promise (because it's `async`).
- `.catch((err) => {...})` — if the promise rejects (something threw), run this handler.
- `process.exit(1)` — exit the Node process with code `1`. Exit code `0` means success; anything else means failure. This tells CI/CD tools "the seed failed, don't continue."

> **Key Concept: Parameterized Queries (Defense Against SQL Injection)**
> **Technical:** Passing values as a second argument to `pool.query(sql, params)` tells the PostgreSQL driver to send them as separate binary data, not concatenated into the SQL string. Even if a value contains malicious SQL like `'; DROP TABLE users; --`, it will be treated as a literal string, never executed.
>
> **Plain English:** It is the difference between handing a librarian a card that says "look up this title" plus the title written separately, versus shouting "look up the book called [whatever the user shouted]." The first form keeps instructions and data in separate channels. The second lets a malicious user smuggle instructions into the data slot.

> ⚠️ **Common Mistake: String Concatenation Inside SQL**
> **Never** do `pool.query(\`INSERT INTO events VALUES ('${userInput}')\`)`. That is the canonical SQL injection vulnerability. Even if you "trust" the input, you are training a habit that will eventually bite you. Always use `$1, $2, ...` placeholders with a `params` array.

Run the seed:

```bash
cd server && npm run seed
```

Expected output (your numbers will vary since the data is random):

```
Seeding database...
Generating 90 days of analytics data.

Seeded 5427 events across 90 days.

Event summary:
┌─────────────┬───────┬─────────────┐
│ event_type  │ count │ total_value │
├─────────────┼───────┼─────────────┤
│ page_view   │ 2981  │ 0.00        │
│ click       │ 1094  │ 0.00        │
│ signup      │  812  │ 0.00        │
│ purchase    │  540  │ 42891.40    │
└─────────────┴───────┴─────────────┘

Done!
```

### ✅ Checkpoint

- [ ] Schema runs without errors
- [ ] Seed script generates ~5000+ events
- [ ] Event summary shows roughly 55% page_view, 20% click, 15% signup, 10% purchase
- [ ] Only purchases have non-zero value

If the proportions look off, re-run the seed — random variance can shift numbers a few percentage points. If they are wildly off, double-check the boundaries in `weightedEventType()`.

---

> **Session Break** — You have a database with realistic seed data. Save your work and take a break.
> When you return, you will build the middleware stack: logging, rate limiting, validation, and error handling.

---

## 4. The Middleware Stack

> **Key Concept: What Is Middleware?**
> **Technical:** In Express, a middleware is a function with the signature `(req, res, next) => void`. Express runs middleware in the order they are registered. Each middleware can read or modify `req` and `res`, and must either call `next()` to pass control to the next middleware, or call `res.send()` (or similar) to end the request. If neither happens, the request hangs forever.
>
> **Plain English:** Think of middleware as a pipeline of inspectors at an airport. Bag check, ID check, security scan, gate agent — each one inspects you, may stop you, or waves you through to the next. The route handler is your seat on the plane. If anything earlier rejects you, you never get there.

### Why a Middleware Stack?

In Level 2 you had a single `app.use(express.json())` and that was it. As your app grows, you need more cross-cutting concerns — and putting them all into route handlers would create massive duplication. Middleware solves this by **factoring shared concerns out of routes**.

You will build four pieces of middleware:

1. **Logger** — record every request, so you can debug production issues
2. **Rate Limiter** — block clients that send too many requests
3. **Validator** — reject malformed query parameters
4. **Error Handler** — catch unhandled errors and return clean responses

### 4a. Logger Middleware

> **Key Concept: Structured Logging**
> **Technical:** Structured logs are emitted as JSON objects with named fields: `{"level":"info","time":1709312345,"msg":"request completed","method":"GET","url":"/api/analytics/overview","responseTime":23}`. Log aggregators (Datadog, Papertrail, CloudWatch) can index, search, and alert on any field. Unstructured logs (`console.log("GET /api/foo took 23ms")`) require regex parsing and are fragile.
>
> **Plain English:** Structured logs are like a spreadsheet — you can filter by any column. Unstructured logs are like a notebook — you read top to bottom and can only `grep` for substrings.

#### 🏗️ Your Turn

Before writing the code, think:

1. Why would you NOT want to log the `/api/health` endpoint?
2. Why log differently in development vs production?

<details>
<summary>See the answers</summary>

1. **Health checks are noisy.** Render and most platforms ping `/api/health` every 30 seconds to know if your service is alive. If you log every health check, your logs are 99% noise. You skip these in the autoLogger config.

2. **Different audiences.** In dev, *you* read the logs as they stream by — colored, indented, human-readable text is best. In production, *machines* (log aggregators) read the logs — JSON is easier to parse and index. Same logger, different output formatting.

</details>

Create `server/src/middleware/logger.ts`:

```typescript
import pino from 'pino';
import pinoHttp from 'pino-http';

// Create the base logger
export const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
});

// Create HTTP request logger middleware
export const httpLogger = pinoHttp({
  logger,
  // Don't log health check requests (noisy)
  autoLogging: {
    ignore: (req) => req.url === '/api/health',
  },
});
```

### Reading This File Line-by-Line

```typescript
import pino from 'pino';
import pinoHttp from 'pino-http';
```

Two default imports from two different packages. `pino` is the core logger; `pino-http` is the Express middleware wrapper for it.

```typescript
export const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
});
```

- `export const logger = ...` — we declare a constant **and** export it in one line, so other files can `import { logger } from './logger'`.
- `pino({...})` — call the `pino` factory with a config object. It returns a logger instance.
- `process.env.NODE_ENV` is an environment variable set by your hosting platform. By convention, it's `'production'` in production and `'development'` locally.
- `process.env.NODE_ENV === 'production' ? 'info' : 'debug'` — ternary: "if production, set the log level to `'info'`; otherwise `'debug'`."
  - `'debug'` prints everything, including fine-grained trace logs.
  - `'info'` prints normal events but skips debug noise.
- `transport: ... ? {...} : undefined` — another ternary, but on a whole config object.
  - In development, use `pino-pretty` to format logs as colored, human-friendly text.
  - In production, `undefined` means "no transport" — pino emits raw JSON, which log aggregators love.

```typescript
export const httpLogger = pinoHttp({
  logger,
  autoLogging: {
    ignore: (req) => req.url === '/api/health',
  },
});
```

- `pinoHttp({...})` — wraps the logger into Express middleware.
- `logger` — shorthand for `logger: logger`. When an object key and value share a name, you can write the name once. This is called **shorthand property names**.
- `autoLogging.ignore` is a function; pino-http calls it with the incoming request. If it returns `true`, skip logging. Here we skip health checks.
- `(req) => req.url === '/api/health'` — an arrow function that takes the request and returns `true` if the URL is `/api/health`.

> ⚠️ **Common Mistake: `console.log` in Production**
> `console.log` writes synchronously, blocking the event loop. Under load, this becomes a measurable performance hit. pino batches writes asynchronously and is designed for production use. Once you use pino, never reach for `console.log` again — call `logger.info(...)` so logs are consistent and structured.

### 4b. Rate Limiter Middleware

> **Key Concept: Rate Limiting**
> **Technical:** Rate limiting tracks how many requests a client has made within a time window. If they exceed the limit, the server responds `429 Too Many Requests` instead of running the handler. The most common algorithm is a **fixed window**: count requests in 15-minute buckets per IP. More sophisticated algorithms (sliding window, token bucket) smooth out edge cases.
>
> **Plain English:** Rate limiting is the bouncer at a club with a capacity limit. After 100 people enter in a 15-minute window, new arrivals wait outside until the window resets. Without a bouncer, a single bot could send a million requests and crash your server.

#### 🏗️ Your Turn

You are designing rate limits for a public REST API. The API has two kinds of endpoints:

- Lightweight: aggregations that finish in <50ms (`/overview`, `/categories`)
- Heavy: paginated raw data dumps (`/events`)

Should both share the same limit? Why or why not?

<details>
<summary>See the answer</summary>

No. Heavier endpoints deserve stricter limits. A general limit of "100 requests per 15 minutes" is appropriate for cheap endpoints, but a `/events` endpoint that returns thousands of rows with a `COUNT` query is more expensive — a single user hammering it could degrade the database for everyone. So you stack a stricter per-endpoint limiter on top of the global one. Cheap endpoints inherit only the general limit; expensive ones get both.

</details>

Create `server/src/middleware/rateLimiter.ts`:

```typescript
import rateLimit from 'express-rate-limit';

// General rate limit: 100 requests per 15 minutes per IP
export const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 minutes
  max: 100,                     // limit each IP to 100 requests per window
  standardHeaders: true,        // Return rate limit info in RateLimit-* headers
  legacyHeaders: false,         // Disable X-RateLimit-* headers
  message: {
    error: 'Too many requests. Please try again later.',
  },
});

// Stricter rate limit for expensive endpoints
export const strictLimiter = rateLimit({
  windowMs: 1 * 60 * 1000,    // 1 minute
  max: 20,                      // 20 requests per minute
  message: {
    error: 'Rate limit exceeded for this endpoint.',
  },
});
```

### Reading This File Line-by-Line

```typescript
import rateLimit from 'express-rate-limit';
```

Default import from the `express-rate-limit` npm package. `rateLimit` is a **factory function** — you call it with a config and it hands back an Express middleware.

```typescript
export const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    error: 'Too many requests. Please try again later.',
  },
});
```

- `export const generalLimiter` — export this middleware so `index.ts` can `import { generalLimiter } from '...'`.
- `rateLimit({...})` — call the factory, passing one config object. What it returns is a function that looks like `(req, res, next) => ...` — exactly the middleware shape Express expects.

Each config option:

- `windowMs: 15 * 60 * 1000` — JavaScript does the math at read time. `15 * 60 * 1000` = `900,000` milliseconds = 15 minutes. Writing it as a multiplication is a **readability trick**: anyone reading "15 × 60 seconds × 1000 ms" instantly sees "fifteen minutes." Writing `900000` would be opaque.
- `max: 100` — allow at most 100 requests per IP per window.
- `standardHeaders: true` — send modern `RateLimit-*` response headers so clients can see how many requests they have left.
- `legacyHeaders: false` — skip the old `X-RateLimit-*` headers (many APIs still emit both; we don't need to).
- `message: { error: '...' }` — object sent as the JSON body when someone hits the limit. Along with the `429 Too Many Requests` status code, the client sees `{ "error": "Too many requests..." }`.

The `strictLimiter` below works the same way with tighter numbers — shorter window (1 minute), fewer requests (20). You'll later attach this one to only the expensive `/events` endpoint.

> **Spatial Note: Where Does the Counter Live?**
> By default, `express-rate-limit` stores counters in memory inside the Node.js process. That works for a single server. The moment you scale to multiple servers, each one has its own counter — a user could send 100 requests to each server and bypass the limit. In production at scale, you would use a Redis-backed store. For Level 4, in-memory is fine.

### 4c. Validation Middleware

> **Key Concept: Validate at the Edge**
> **Technical:** Validation middleware rejects malformed requests *before* they reach route handlers. This keeps route code clean (it can trust its inputs) and prevents bad data from ever touching the database. Validation belongs at the boundary of your system — the moment data crosses from "untrusted" (user input) to "trusted" (your code).
>
> **Plain English:** It is the difference between checking IDs at the front door versus letting everyone in and checking IDs at every individual room. Front-door validation is simpler, more consistent, and harder to bypass.

#### 🏗️ Your Turn

List the validation rules you would enforce on the analytics query parameters. Think about what a malicious or buggy client could send.

- `startDate` / `endDate`: ?
- `category`: ?
- `device`: ?
- `page` / `limit` (pagination): ?

<details>
<summary>See the rules</summary>

- `startDate` / `endDate` must match `YYYY-MM-DD` format. `startDate` must be before `endDate`. Otherwise the SQL `WHERE` clause silently produces 0 results or breaks.
- `category` must be one of the known categories. Otherwise we run a query that returns 0 rows and the user gets confusing empty data.
- `device` must be one of `desktop`, `mobile`, `tablet`.
- `page` must be a positive integer. (Negative `page` produces a negative `OFFSET` and crashes the query.)
- `limit` must be a positive integer between 1 and 100. (Without a cap, a client could request `limit=1000000` and exhaust memory.)

The deeper lesson: **assume every client is hostile or buggy.** Validation isn't paranoia, it's structure.

</details>

Create `server/src/middleware/validator.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';

/**
 * Validates analytics query parameters.
 * Rejects requests with invalid date formats, categories, etc.
 */
export function validateAnalyticsQuery(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const { startDate, endDate, category, device, page: pageNum, limit } = req.query;

  // Validate date format (YYYY-MM-DD)
  const dateRegex = /^\d{4}-\d{2}-\d{2}$/;

  if (startDate && !dateRegex.test(startDate as string)) {
    res.status(400).json({ error: 'Invalid startDate format. Use YYYY-MM-DD.' });
    return;
  }

  if (endDate && !dateRegex.test(endDate as string)) {
    res.status(400).json({ error: 'Invalid endDate format. Use YYYY-MM-DD.' });
    return;
  }

  // Validate date range
  if (startDate && endDate) {
    if (new Date(startDate as string) > new Date(endDate as string)) {
      res.status(400).json({ error: 'startDate must be before endDate.' });
      return;
    }
  }

  // Validate category
  const validCategories = ['marketing', 'product', 'support', 'engineering'];
  if (category && !validCategories.includes(category as string)) {
    res.status(400).json({
      error: `Invalid category. Must be one of: ${validCategories.join(', ')}`,
    });
    return;
  }

  // Validate device
  const validDevices = ['desktop', 'mobile', 'tablet'];
  if (device && !validDevices.includes(device as string)) {
    res.status(400).json({
      error: `Invalid device. Must be one of: ${validDevices.join(', ')}`,
    });
    return;
  }

  // Validate pagination
  if (pageNum && (isNaN(Number(pageNum)) || Number(pageNum) < 1)) {
    res.status(400).json({ error: 'page must be a positive integer.' });
    return;
  }

  if (limit && (isNaN(Number(limit)) || Number(limit) < 1 || Number(limit) > 100)) {
    res.status(400).json({ error: 'limit must be between 1 and 100.' });
    return;
  }

  next();
}
```

### Reading This File Line-by-Line

```typescript
import { Request, Response, NextFunction } from 'express';
```

Named imports of **TypeScript types** — not values, but shapes. `Request`, `Response`, and `NextFunction` describe what Express passes into middleware. We use them below to annotate the function's parameters so TypeScript can check our code.

```typescript
export function validateAnalyticsQuery(
  req: Request,
  res: Response,
  next: NextFunction
): void {
```

- `export function` — declare and export a named function in one line.
- Parameters are typed: `req: Request`, `res: Response`, `next: NextFunction`. This is standard Express middleware.
- `: void` — "this function returns nothing." It acts through side effects (sending a response or calling `next()`).

```typescript
const { startDate, endDate, category, device, page: pageNum, limit } = req.query;
```

This is **destructuring**. Read it as "pull these fields out of `req.query` and give me named variables":

- `startDate` → `req.query.startDate`
- `endDate` → `req.query.endDate`
- `category` → `req.query.category`
- `device` → `req.query.device`
- `page: pageNum` → pull `req.query.page` but **rename it** to `pageNum` locally. We rename to avoid shadowing any global or built-in `page` variable.
- `limit` → `req.query.limit`

`req.query` is an object Express builds for you by parsing the URL's query string. If the URL is `/overview?startDate=2025-01-01&category=marketing`, then `req.query` is `{ startDate: '2025-01-01', category: 'marketing' }`.

```typescript
const dateRegex = /^\d{4}-\d{2}-\d{2}$/;
```

A **regular expression** — a pattern for matching text. Breaking it down:

- `/ ... /` — the slashes delimit a regex literal.
- `^` — "match starts at the beginning."
- `\d{4}` — "exactly 4 digits."
- `-` — a literal hyphen.
- `\d{2}` — "exactly 2 digits."
- `$` — "match ends here."

So this pattern says "the whole string must be four digits, a hyphen, two digits, a hyphen, two digits" — the shape of `YYYY-MM-DD`.

```typescript
if (startDate && !dateRegex.test(startDate as string)) {
  res.status(400).json({ error: 'Invalid startDate format. Use YYYY-MM-DD.' });
  return;
}
```

- `if (startDate && ...)` — `&&` is logical AND. Since `startDate` might be `undefined` (the user didn't provide it), we check that first. If `startDate` is falsy, JavaScript short-circuits — the right side never runs.
- `dateRegex.test(value)` — returns `true` if `value` matches the regex, `false` otherwise. The `!` in front negates it — "if it does NOT match."
- `startDate as string` — a **type assertion**. `req.query` values can be strings, arrays, or even other objects (depending on the URL). We're telling TypeScript, "trust me, treat it as a plain string." Use this carefully — you're overriding the type checker.
- `res.status(400).json({ ... })` — **method chaining**. `res.status(400)` sets the HTTP status code and returns `res` itself, letting you immediately call `.json(...)` on the same object. `.json(obj)` serializes `obj` to JSON and sends it as the response body.
- `return;` — **critical**. Without this, execution continues past the `if` and might eventually call `next()`, sending the same request to the real route handler with bad data. The `return` stops the function dead.

```typescript
if (startDate && endDate) {
  if (new Date(startDate as string) > new Date(endDate as string)) {
    res.status(400).json({ error: 'startDate must be before endDate.' });
    return;
  }
}
```

- Only runs if both dates are provided.
- `new Date(string)` parses a date string into a `Date` object. `Date` objects can be compared with `>` — JavaScript converts them to timestamps (milliseconds since 1970) under the hood.

```typescript
const validCategories = ['marketing', 'product', 'support', 'engineering'];
if (category && !validCategories.includes(category as string)) {
  res.status(400).json({ ... });
  return;
}
```

- `validCategories.includes(x)` — built-in array method; returns `true` if `x` is somewhere in the array.
- Same shape as the date check: if value is present AND not in our allowlist, reject.

```typescript
if (pageNum && (isNaN(Number(pageNum)) || Number(pageNum) < 1)) {
  res.status(400).json({ error: 'page must be a positive integer.' });
  return;
}
```

- `Number(x)` — convert `x` to a number. `Number('5')` is `5`; `Number('abc')` is `NaN` (a special "not a number" value).
- `isNaN(x)` — returns `true` if `x` is `NaN`. You can't do `x === NaN` because `NaN` is famously not equal to itself.
- The condition reads: "page is provided AND (it's not a number OR it's less than 1)." Catches both `?page=abc` and `?page=-1`.

```typescript
next();
```

If we made it here, everything passed. Call `next()` with no arguments to hand the request to the next middleware (in our case, the route handler). Passing no argument (or `undefined`) means "no error."

> ⚠️ **Common Mistake: Forgetting `return` After `res.status(400)`**
> `res.status().json()` does NOT stop the function. If you forget `return`, execution falls through and may hit `next()`, which sends the request to the route handler with invalid input. Or worse, two responses are sent and Express crashes with `Cannot set headers after they are sent`.

### 4d. Error Handler Middleware

> **Key Concept: Centralized Error Handling**
> **Technical:** Express error-handling middleware has a special signature: four arguments instead of three. Express checks the arity of registered middleware to identify error handlers. They run only when something earlier called `next(err)` — passing a non-null value to `next` skips all normal middleware and goes straight to error handlers.
>
> **Plain English:** Think of the error handler as the lost-and-found at an airport. Anything that goes wrong elsewhere ends up here. There is one central place that decides how errors are logged and what the user sees — instead of duplicating that logic in every route.

#### 🏗️ Your Turn

In production, why would you NOT want to send the raw error message to the client?

<details>
<summary>See the answer</summary>

Error messages often leak sensitive information: database table names, file paths, SQL syntax, internal service URLs, library versions. An attacker probing for vulnerabilities reads error responses to map your stack and find weak points. A generic `"Internal server error"` reveals nothing — but the full error is still logged server-side so *you* can debug.

This is **defense in depth**: even if a code path you didn't anticipate throws an error, the user sees nothing exploitable.

</details>

Create `server/src/middleware/errorHandler.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import { logger } from './logger';

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
  // Log the full error server-side
  logger.error({ err }, 'Unhandled error');

  // Send clean error to client
  const statusCode = res.statusCode !== 200 ? res.statusCode : 500;

  res.status(statusCode).json({
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message,
  });
}
```

### Reading This File Line-by-Line

```typescript
import { Request, Response, NextFunction } from 'express';
import { logger } from './logger';
```

Named imports. The second line is a **relative import** — it reaches into the sibling `logger.ts` file and pulls out the `logger` constant we exported earlier.

```typescript
export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
```

The **four-parameter signature** is how Express knows this is an error handler, not a normal middleware. Regular middleware has three parameters (`req`, `res`, `next`); error handlers have four, with `err` first.

- `err: Error` — the error object. `Error` is a built-in JavaScript class with a `.message` and a `.stack`.
- `_req: Request` and `_next: NextFunction` — we must declare them because Express inspects the function's arity (parameter count), but we don't use them. The **underscore prefix** is a common convention meaning "I'm aware I'm not using this." It silences unused-variable warnings in many linters.

```typescript
logger.error({ err }, 'Unhandled error');
```

- `logger.error(...)` — log at the `error` level. pino methods typically accept `(objectWithData, messageString)`.
- `{ err }` — another **shorthand property name**. Equivalent to `{ err: err }` — an object with a single field `err` holding the error object. pino will serialize the error's stack trace, message, and name into the structured log.

```typescript
const statusCode = res.statusCode !== 200 ? res.statusCode : 500;
```

- `res.statusCode` — the HTTP status currently set on the response (defaults to `200` if nothing was set).
- Ternary: "if the status was already changed from 200, keep it; otherwise use 500."
- Why? Sometimes a route sets `res.status(403)` then throws — we want to preserve the 403. But if the status is still 200 (meaning we never set one), the only sensible default is 500 ("Internal Server Error").

```typescript
res.status(statusCode).json({
  error: process.env.NODE_ENV === 'production'
    ? 'Internal server error'
    : err.message,
});
```

- Method chaining again: set status, then send JSON.
- The body's `error` field is decided by a ternary based on environment:
  - **Production:** a generic `'Internal server error'`. Reveals nothing about your stack.
  - **Development:** `err.message` — the real message, so you can debug.

The **full error with stack trace** still goes to the logs in both environments. The user-facing response is simply sanitized in production.

---

> **Session Break** — You have built the complete middleware stack. Save your work and take a break.
> When you return, you will build the service layer with all the SQL.

---

## 5. The Service Layer

> **Key Concept: Separation of Concerns**
> **Technical:** "Separation of concerns" is the principle that different kinds of work belong in different parts of the codebase. In a backend, the two main concerns are **HTTP** (parsing requests, sending responses, status codes) and **data** (queries, calculations, business rules). The service layer holds the data concern; routes hold the HTTP concern. They never mix.
>
> **Plain English:** Imagine a restaurant. The waiter (route) takes your order and brings the food. The cook (service) prepares it. They have different skills, different tools, and they should not do each other's jobs. If you make the waiter cook, your service collapses under load.

### Why Separate Routes from Services?

Picture a route handler with SQL embedded in it. Now imagine you need to:

- **Test** the SQL without spinning up an HTTP server → you can't, it is glued to `req`/`res`
- **Reuse** the same query from a CLI script → you can't, you would have to copy it
- **Switch** from PostgreSQL to a different store → you have to rewrite every route
- **Read** the route to understand what it does → you have to mentally separate "is this HTTP code or data code?"

Each of those gets easier when SQL lives in its own layer with a clean function signature like `getOverview(filters): Promise<OverviewStats>`.

#### 🏗️ Your Turn

Look at a typical Level 2 route handler:

```typescript
router.get('/users/:id', async (req, res) => {
  const result = await pool.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
  if (result.rows.length === 0) {
    return res.status(404).json({ error: 'Not found' });
  }
  res.json(result.rows[0]);
});
```

Refactor this in your head into route + service. What does the service function look like? What does the route look like?

<details>
<summary>See the answer</summary>

```typescript
// service
export async function getUserById(id: string): Promise<User | null> {
  const result = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
  return result.rows[0] ?? null;
}

// route
router.get('/users/:id', async (req, res, next) => {
  try {
    const user = await getUserById(req.params.id);
    if (!user) return res.status(404).json({ error: 'Not found' });
    res.json(user);
  } catch (err) {
    next(err);
  }
});
```

The service knows nothing about HTTP — it just takes an `id` and returns a `User | null`. The route handles the HTTP-specific concerns: status codes, error forwarding, JSON serialization.

</details>

### 5a. The `buildWhereClause` Helper

Most analytics queries take the same set of filters: date range, category, device. Without a helper, every query would repeat 20 lines of "if startDate is set, add it to the WHERE." So we extract that into one function.

> **Key Concept: Building Dynamic WHERE Clauses Safely**
> **Technical:** A WHERE clause that includes only some filters is dynamic — its shape depends on which filters are present. The naïve approach is string concatenation: `where += "AND startDate >= '" + startDate + "'"`. That is a SQL injection vulnerability. The safe pattern is to build the clause as a string with `$1, $2, ...` placeholders while accumulating the actual values into a parallel `params` array.
>
> **Plain English:** Think of placeholders as numbered slots. The SQL says "compare this column to slot 1." Then the params array says "slot 1 holds this value." The database wires them together — the value never becomes part of the SQL grammar.

Create `server/src/services/analyticsService.ts` (we'll build it up in pieces — start with the helper):

```typescript
import pool from '../db/index';
import type {
  OverviewStats,
  TimeSeriesPoint,
  CategoryBreakdown,
  DeviceBreakdown,
  TopPage,
  PaginatedEvents,
  AnalyticsFilters,
} from '../types/index';

// --- Helper: Build WHERE clause from filters ---

function buildWhereClause(filters: AnalyticsFilters): {
  clause: string;
  params: (string | number)[];
  nextParam: number;
} {
  const conditions: string[] = [];
  const params: (string | number)[] = [];
  let paramIndex = 1;

  if (filters.startDate) {
    conditions.push(`created_at >= $${paramIndex}`);
    params.push(filters.startDate);
    paramIndex++;
  }

  if (filters.endDate) {
    conditions.push(`created_at < ($${paramIndex}::date + interval '1 day')`);
    params.push(filters.endDate);
    paramIndex++;
  }

  if (filters.category) {
    conditions.push(`category = $${paramIndex}`);
    params.push(filters.category);
    paramIndex++;
  }

  if (filters.device) {
    conditions.push(`device = $${paramIndex}`);
    params.push(filters.device);
    paramIndex++;
  }

  const clause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

  return { clause, params, nextParam: paramIndex };
}
```

### Reading `buildWhereClause` Line-by-Line

```typescript
import pool from '../db/index';
import type {
  OverviewStats,
  // ...
  AnalyticsFilters,
} from '../types/index';
```

- First import pulls in the shared pool.
- Second is `import type { ... }` — a TypeScript-only construct. It says "I'm importing these **only for type-checking**; don't include them in the compiled JavaScript." Types vanish at runtime; only the compiler uses them. The `type` keyword makes that explicit and speeds up builds.

```typescript
function buildWhereClause(filters: AnalyticsFilters): {
  clause: string;
  params: (string | number)[];
  nextParam: number;
} {
```

- `filters: AnalyticsFilters` — the parameter is typed by a named type (defined elsewhere in `types/index.ts`).
- The return type is an **inline object type**. Read it as "this function returns an object with three fields: `clause` (a string), `params` (an array of strings or numbers), and `nextParam` (a number)."

```typescript
const conditions: string[] = [];
const params: (string | number)[] = [];
let paramIndex = 1;
```

Three accumulators:

- `conditions` — an array of SQL fragments like `'created_at >= $1'`. We'll join them with `' AND '` at the end.
- `params` — a **parallel** array of the actual values. Fragment `N` in `conditions` refers to slot `N` in `params`.
- `paramIndex` — tracks the next placeholder number. Starts at 1 because PostgreSQL placeholders are 1-indexed (`$1`, `$2`, ...).

```typescript
if (filters.startDate) {
  conditions.push(`created_at >= $${paramIndex}`);
  params.push(filters.startDate);
  paramIndex++;
}
```

- `if (filters.startDate)` — only run if the field was provided. If it's `undefined`, this block skips entirely.
- `conditions.push(...)` — adds a new fragment like `'created_at >= $1'` to the end of the array.
- Backticks again: `$${paramIndex}` expands to `$1` on the first call, `$2` on the next, etc.
- `params.push(filters.startDate)` — push the actual value into the same slot.
- `paramIndex++` — increment for next time. `++` is shorthand for `= paramIndex + 1`.

The key invariant: **`conditions[i]` always refers to `params[i]`**. Keeping them synchronized as you build up is how parameterized queries work.

```typescript
if (filters.endDate) {
  conditions.push(`created_at < ($${paramIndex}::date + interval '1 day')`);
  params.push(filters.endDate);
  paramIndex++;
}
```

Same shape, but the SQL fragment is tricky: `$1::date + interval '1 day'`. Breaking that down:

- `$1::date` — PostgreSQL's **cast** syntax. Says "treat this value as a `date` type."
- `+ interval '1 day'` — PostgreSQL can add an interval to a date to get a new date.
- `created_at < (endDate + 1 day)` means "include the entire `endDate`, not just midnight of that day." If a user passes `endDate=2025-01-31`, we want events at 3pm on the 31st to still match — so we compare against the *start* of Feb 1 using `<`.

```typescript
const clause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

return { clause, params, nextParam: paramIndex };
```

- If we accumulated any conditions, prefix them with `WHERE` and join with `' AND '`. Example output: `WHERE created_at >= $1 AND category = $2`.
- If no filters were given, `clause` is an empty string — safe to interpolate into the query without breaking syntax.
- `return { clause, params, nextParam: paramIndex }` uses **shorthand property names** again: `{ clause }` is `{ clause: clause }`. `nextParam: paramIndex` renames it on the way out (callers see `.nextParam`, not `.paramIndex`).

> **Why return `nextParam`?**
> Some queries need to add more placeholders *after* the WHERE clause (for example, pagination's `LIMIT $5 OFFSET $6`). Returning the next available index lets callers continue numbering correctly.

### 5b. Aggregation Queries

> **Key Concept: SQL Aggregation**
> **Technical:** Aggregation functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) collapse many rows into one summary value, optionally grouped by another column with `GROUP BY`. PostgreSQL's `FILTER (WHERE ...)` clause lets you apply a condition to a single aggregate without affecting others — counting only `signup` events while in the same query summing only `purchase` revenue.
>
> **Plain English:** Aggregation is "tell me about a group, not individual rows." Instead of "give me every row," you ask "how many rows? What is the total? What is the average per category?" `FILTER` is a way to ask several different "how many" questions in one query without separate subqueries.

Add the aggregation functions to `analyticsService.ts`:

```typescript
// --- Overview Stats ---

export async function getOverview(filters: AnalyticsFilters): Promise<OverviewStats> {
  const { clause, params } = buildWhereClause(filters);

  const result = await pool.query(
    `SELECT
       COUNT(*) FILTER (WHERE event_type = 'page_view')  AS total_page_views,
       COUNT(*) FILTER (WHERE event_type = 'signup')      AS total_signups,
       COALESCE(SUM(value), 0)                            AS total_revenue,
       MIN(created_at)                                     AS period_start,
       MAX(created_at)                                     AS period_end
     FROM analytics_events
     ${clause}`,
    params
  );

  const row = result.rows[0];

  // Calculate average session duration (simulated from data)
  const sessionResult = await pool.query(
    `SELECT COUNT(DISTINCT session_id) AS sessions,
            EXTRACT(EPOCH FROM MAX(created_at) - MIN(created_at)) AS total_seconds
     FROM analytics_events
     ${clause}`,
    params
  );

  const sessions = Number(sessionResult.rows[0].sessions) || 1;
  const totalSeconds = Number(sessionResult.rows[0].total_seconds) || 0;
  const avgDuration = Math.round(totalSeconds / sessions);

  return {
    total_page_views: Number(row.total_page_views),
    total_signups: Number(row.total_signups),
    total_revenue: Number(row.total_revenue),
    avg_session_duration: Math.min(avgDuration, 600), // Cap at 10 minutes
    period_start: row.period_start,
    period_end: row.period_end,
  };
}

// --- Time Series ---

export async function getTimeSeries(
  filters: AnalyticsFilters
): Promise<TimeSeriesPoint[]> {
  const { clause, params } = buildWhereClause(filters);

  const result = await pool.query(
    `SELECT
       DATE(created_at) AS date,
       COUNT(*) FILTER (WHERE event_type = 'page_view') AS page_views,
       COUNT(*) FILTER (WHERE event_type = 'signup')     AS signups,
       COALESCE(SUM(value), 0)                           AS revenue
     FROM analytics_events
     ${clause}
     GROUP BY DATE(created_at)
     ORDER BY date`,
    params
  );

  return result.rows.map((row) => ({
    date: row.date.toISOString().split('T')[0],
    page_views: Number(row.page_views),
    signups: Number(row.signups),
    revenue: Number(row.revenue),
  }));
}

// --- Category Breakdown ---

export async function getCategories(
  filters: AnalyticsFilters
): Promise<CategoryBreakdown[]> {
  const { clause, params } = buildWhereClause(filters);

  const result = await pool.query(
    `SELECT
       category,
       COUNT(*)               AS count,
       COALESCE(SUM(value), 0) AS revenue
     FROM analytics_events
     ${clause}
     GROUP BY category
     ORDER BY count DESC`,
    params
  );

  return result.rows.map((row) => ({
    category: row.category,
    count: Number(row.count),
    revenue: Number(row.revenue),
  }));
}

// --- Device Breakdown ---

export async function getDevices(
  filters: AnalyticsFilters
): Promise<DeviceBreakdown[]> {
  const { clause, params } = buildWhereClause(filters);

  const result = await pool.query(
    `SELECT
       device,
       COUNT(*) AS count
     FROM analytics_events
     ${clause}
     GROUP BY device
     ORDER BY count DESC`,
    params
  );

  const total = result.rows.reduce((sum, row) => sum + Number(row.count), 0);

  return result.rows.map((row) => ({
    device: row.device,
    count: Number(row.count),
    percentage: total > 0 ? Number(row.count) / total : 0,
  }));
}

// --- Top Pages ---

export async function getTopPages(
  filters: AnalyticsFilters
): Promise<TopPage[]> {
  const { clause, params } = buildWhereClause(filters);

  const result = await pool.query(
    `SELECT
       page,
       COUNT(*)                     AS views,
       COUNT(DISTINCT session_id)   AS unique_sessions
     FROM analytics_events
     ${clause.length > 0 ? clause + ' AND' : 'WHERE'} event_type = 'page_view'
     GROUP BY page
     ORDER BY views DESC
     LIMIT 10`,
    params
  );

  return result.rows.map((row) => ({
    page: row.page,
    views: Number(row.views),
    unique_sessions: Number(row.unique_sessions),
  }));
}
```

### Reading `getOverview` Line-by-Line

```typescript
export async function getOverview(filters: AnalyticsFilters): Promise<OverviewStats> {
```

- `export async function` — exported, marked async because we use `await` inside.
- `Promise<OverviewStats>` — returns a promise that will eventually resolve to an `OverviewStats` object (a shape defined in `types/index.ts`). TypeScript will enforce that the value we `return` matches that shape.

```typescript
const { clause, params } = buildWhereClause(filters);
```

**Destructuring the helper's return value.** `buildWhereClause` returns an object with three fields; we're pulling out only `clause` and `params` for this query. (`nextParam` exists on the return value but we don't need it here.)

```typescript
const result = await pool.query(
  `SELECT
     COUNT(*) FILTER (WHERE event_type = 'page_view')  AS total_page_views,
     COUNT(*) FILTER (WHERE event_type = 'signup')      AS total_signups,
     COALESCE(SUM(value), 0)                            AS total_revenue,
     MIN(created_at)                                     AS period_start,
     MAX(created_at)                                     AS period_end
   FROM analytics_events
   ${clause}`,
  params
);
```

- `` `... ${clause}` `` — backticks with template substitution. `${clause}` drops in either `'WHERE created_at >= $1 AND ...'` or an empty string. Either way, the SQL stays valid.
- `pool.query(sql, params)` — two-argument form. The driver inserts the `params` values safely into the placeholders at send time. **Values never become part of the SQL grammar.**
- `await` pauses until PostgreSQL responds. Without `await`, `result` would be a promise, not the answer.

Now the SQL itself — this is the first time you're seeing several aggregation features:

- `COUNT(*) FILTER (WHERE event_type = 'page_view')` — count only rows matching the inner condition. You can stack multiple of these in one query, each filtering differently.
- `AS total_page_views` — **column alias**. Renames the output column so `result.rows[0].total_page_views` reads cleanly.
- `COALESCE(SUM(value), 0)` — `SUM` returns `NULL` when there are no rows. `COALESCE(a, b)` returns `a` if it's not null, else `b`. So this reads as "sum of values, or 0 if no rows."
- `MIN(created_at)`, `MAX(created_at)` — earliest and latest timestamps in the filtered set.

```typescript
const row = result.rows[0];
```

`pool.query` returns `{ rows: [...], rowCount: N, ... }`. Since this query returns exactly one row of aggregates, we grab `rows[0]`.

```typescript
const sessions = Number(sessionResult.rows[0].sessions) || 1;
const totalSeconds = Number(sessionResult.rows[0].total_seconds) || 0;
const avgDuration = Math.round(totalSeconds / sessions);
```

- `Number(x)` — convert string to number. The pg driver returns `BIGINT` and `DECIMAL` as strings to preserve precision. Casting with `Number(...)` gives you a real JavaScript number.
- `|| 1` — **logical OR with a fallback**. If the left side is falsy (0, null, undefined, NaN), use the right side. Here it prevents division by zero: if `sessions` is 0, fall back to 1.
- `Math.round(...)` — standard rounding to the nearest integer.

```typescript
return {
  total_page_views: Number(row.total_page_views),
  total_signups: Number(row.total_signups),
  total_revenue: Number(row.total_revenue),
  avg_session_duration: Math.min(avgDuration, 600),
  period_start: row.period_start,
  period_end: row.period_end,
};
```

- Build and return a plain object matching the `OverviewStats` type.
- `Math.min(a, b)` — returns the smaller of two values. We cap `avg_session_duration` at 600 seconds (10 minutes) to avoid absurd numbers when the data is sparse.
- Timestamps are returned as-is — the pg driver gives us `Date` objects automatically, which `JSON.stringify` serializes to ISO strings when Express sends them.

### Reading `getTimeSeries` — The `.map(...)` Pattern

```typescript
return result.rows.map((row) => ({
  date: row.date.toISOString().split('T')[0],
  page_views: Number(row.page_views),
  signups: Number(row.signups),
  revenue: Number(row.revenue),
}));
```

- `result.rows.map((row) => ({...}))` — `.map` is an array method that **transforms each item**, returning a new array. For each row from the DB, we produce a cleaned-up object.
- `(row) => ({...})` — arrow function. Note the **parentheses around the object**: `=> ({...})`. Without them, the curly braces would be interpreted as a function body, not an object literal. You must wrap an object literal in parens when returning it from a concise arrow function.
- `row.date.toISOString().split('T')[0]` — chained methods:
  - `.toISOString()` turns a `Date` into a string like `"2025-02-28T00:00:00.000Z"`.
  - `.split('T')` breaks that string into `["2025-02-28", "00:00:00.000Z"]`.
  - `[0]` grabs the first piece — just the date part.
- `Number(row.page_views)` — remember, pg returns numeric strings; convert to real numbers.

### Reading the Other Aggregates

All follow the same shape: `buildWhereClause` → `await pool.query(...)` → `.map(row => ({...}))`. The only things that change are the SQL and the output field names. Once you've internalized the pattern in `getOverview`, the others are variations.

**Key SQL patterns to internalize:**

| Pattern | What it does | When to reach for it |
|---------|-------------|----------------------|
| `COUNT(*) FILTER (WHERE event_type = 'signup')` | Count only rows matching the inner condition | When you want multiple conditional counts in one query without subqueries |
| `COALESCE(SUM(value), 0)` | If `SUM` returns `NULL` (no rows), return `0` | Whenever a sum could legally be empty — never let `NULL` leak to the frontend |
| `DATE(created_at)` | Strip the time off a timestamp, leaving just the date | Grouping events by day for time-series charts |
| `COUNT(DISTINCT session_id)` | Count unique sessions, not total events | Anywhere you care about "unique users" or "unique anything" |
| `GROUP BY ... ORDER BY count DESC` | Bucket rows, then sort buckets by aggregate | Top-N lists (top pages, top categories) |
| `LIMIT 10` | Take the first 10 rows after sorting | Top-N lists, paginated previews |
| `EXTRACT(EPOCH FROM ...)` | Convert a duration to seconds | Anywhere you need numeric time arithmetic |

> **Why `Number(row.total_revenue)`?**
> The PostgreSQL driver returns numeric types as JavaScript `string`, not `number`, to preserve precision (PostgreSQL `BIGINT` and `DECIMAL` can exceed JavaScript's safe integer range). Casting with `Number(...)` converts back to a JavaScript number for the API response. If you skip this cast, the JSON contains `"total_revenue": "42891.40"` (string) and your frontend math breaks silently.

> ⚠️ **Common Mistake: The `WHERE / AND` Bug in `getTopPages`**
> Look at this line: `${clause.length > 0 ? clause + ' AND' : 'WHERE'} event_type = 'page_view'`. We need an extra condition (`event_type = 'page_view'`), but whether to write `AND` or `WHERE` depends on whether other filters exist. If `clause` is empty (no filters), we need a fresh `WHERE`. If it already starts with `WHERE`, we need `AND`. Forgetting this produces invalid SQL like `WHERE WHERE event_type = 'page_view'`.

### 5c. Pagination

> **Key Concept: Offset Pagination**
> **Technical:** `LIMIT N OFFSET M` returns at most `N` rows, skipping the first `M`. To get a "page" of results: page `1` is `LIMIT 20 OFFSET 0`, page `2` is `LIMIT 20 OFFSET 20`, page `3` is `LIMIT 20 OFFSET 40`. You also need a separate `COUNT(*)` query so the frontend knows the total number of pages.
>
> **Plain English:** Pagination is how you avoid sending 10,000 rows to the frontend. You send 20 at a time and let the user click "next." The catch: you also need to tell them how many pages exist total, which requires a second query that ignores `LIMIT`.

#### 🏗️ Your Turn

Why do we need TWO queries (count + rows) instead of one?

<details>
<summary>See the answer</summary>

A single query with `LIMIT 20` returns at most 20 rows — there is no way to know whether there are 21 rows total or 21 million. The count is information the database has but won't share unless you ask. So you do two queries:

- `SELECT COUNT(*) FROM ... WHERE ...` — total rows matching filters (no LIMIT)
- `SELECT * FROM ... WHERE ... ORDER BY ... LIMIT $N OFFSET $M` — the page itself

The frontend then computes `total_pages = Math.ceil(total / limit)` to render pagination controls.

There are alternatives (cursor pagination, keyset pagination) that avoid the count query, but they have different tradeoffs and don't support "jump to page 47." Offset pagination is the simplest model and what most APIs use.

</details>

Add pagination to `analyticsService.ts`:

```typescript
// --- Paginated Events ---

export async function getEvents(
  filters: AnalyticsFilters,
  page: number = 1,
  limit: number = 20
): Promise<PaginatedEvents> {
  const { clause, params, nextParam } = buildWhereClause(filters);
  const offset = (page - 1) * limit;

  // Get total count
  const countResult = await pool.query(
    `SELECT COUNT(*) FROM analytics_events ${clause}`,
    params
  );
  const total = Number(countResult.rows[0].count);

  // Get paginated events
  const eventsResult = await pool.query(
    `SELECT * FROM analytics_events
     ${clause}
     ORDER BY created_at DESC
     LIMIT $${nextParam} OFFSET $${nextParam + 1}`,
    [...params, limit, offset]
  );

  return {
    events: eventsResult.rows,
    total,
    page,
    limit,
    total_pages: Math.ceil(total / limit),
  };
}
```

### Reading `getEvents` Line-by-Line

```typescript
export async function getEvents(
  filters: AnalyticsFilters,
  page: number = 1,
  limit: number = 20
): Promise<PaginatedEvents> {
```

- Three parameters. The `= 1` and `= 20` are **default values**. If the caller doesn't pass them, they use those defaults.
- Returns a promise resolving to a `PaginatedEvents` object.

```typescript
const { clause, params, nextParam } = buildWhereClause(filters);
const offset = (page - 1) * limit;
```

- Destructure all three fields from the helper this time — we need `nextParam` for pagination placeholders.
- `(page - 1) * limit` — classic offset math. Page 1 skips 0 rows; page 2 skips `limit` rows; page 3 skips `2*limit`, and so on.

```typescript
const countResult = await pool.query(
  `SELECT COUNT(*) FROM analytics_events ${clause}`,
  params
);
const total = Number(countResult.rows[0].count);
```

Query 1: total matching rows, ignoring pagination. `COUNT(*)` always returns one row with a `count` column.

```typescript
const eventsResult = await pool.query(
  `SELECT * FROM analytics_events
   ${clause}
   ORDER BY created_at DESC
   LIMIT $${nextParam} OFFSET $${nextParam + 1}`,
  [...params, limit, offset]
);
```

Query 2: the actual page of rows.

- `$${nextParam}` and `$${nextParam + 1}` — we're using the two placeholder slots **after** the ones the filters took. If filters used `$1, $2`, `nextParam` is 3, so this line becomes `LIMIT $3 OFFSET $4`.
- `[...params, limit, offset]` — **spread operator** followed by two more values. This says "copy every value from `params` into a new array, then append `limit`, then `offset`." The resulting array's positions line up exactly with the placeholder numbers in the SQL.

```typescript
return {
  events: eventsResult.rows,
  total,
  page,
  limit,
  total_pages: Math.ceil(total / limit),
};
```

- `events: eventsResult.rows` — the raw rows.
- `total, page, limit` — shorthand properties (`total: total` etc.).
- `Math.ceil(total / limit)` — round **up**. If there are 47 events and the page size is 20, `47/20 = 2.35` → `Math.ceil(2.35) = 3` total pages.

---

> **Session Break** — The service layer is complete. Save your work and take a break.
> When you return, you will build the routes that wire HTTP up to these service functions.

---

## 6. Analytics Routes

> **Key Concept: Routes as Thin HTTP Adapters**
> **Technical:** With the service layer doing the data work, route handlers become very thin: parse query params, call a service, send the result. Every route follows the same shape (try / await service / send / catch). This sameness is a feature — readers can scan a route and immediately see what it does.
>
> **Plain English:** A good route handler is boring. It does one thing: translate HTTP into a function call and back. If you find yourself doing real logic in a route, that logic probably belongs in a service.

### 🏗️ Your Turn

Predict the shape of one of these routes. What four things does it need to do?

<details>
<summary>See the shape</summary>

1. Extract the query parameters into a `filters` object
2. Call the matching service function with those filters
3. Send the result as JSON
4. Forward any errors to the error handler with `next(err)`

That's it. Anything more belongs in the service.

</details>

Create `server/src/routes/analytics.ts`:

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { validateAnalyticsQuery } from '../middleware/validator';
import { strictLimiter } from '../middleware/rateLimiter';
import * as analyticsService from '../services/analyticsService';
import type { AnalyticsFilters } from '../types/index';

const router = Router();

// Apply validation to all analytics routes
router.use(validateAnalyticsQuery);

// --- Helper: Extract filters from query params ---

function getFilters(req: Request): AnalyticsFilters {
  return {
    startDate: req.query.startDate as string | undefined,
    endDate: req.query.endDate as string | undefined,
    category: req.query.category as string | undefined,
    device: req.query.device as string | undefined,
  };
}

// --- Routes ---

// GET /api/analytics/overview
router.get('/overview', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const stats = await analyticsService.getOverview(filters);
    res.json(stats);
  } catch (err) {
    next(err);
  }
});

// GET /api/analytics/time-series
router.get('/time-series', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const data = await analyticsService.getTimeSeries(filters);
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// GET /api/analytics/categories
router.get('/categories', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const data = await analyticsService.getCategories(filters);
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// GET /api/analytics/devices
router.get('/devices', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const data = await analyticsService.getDevices(filters);
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// GET /api/analytics/top-pages
router.get('/top-pages', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const data = await analyticsService.getTopPages(filters);
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// GET /api/analytics/events (paginated — stricter rate limit)
router.get('/events', strictLimiter, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const page = Number(req.query.page) || 1;
    const limit = Number(req.query.limit) || 20;
    const data = await analyticsService.getEvents(filters, page, limit);
    res.json(data);
  } catch (err) {
    next(err);
  }
});

export default router;
```

### Reading This File Line-by-Line

```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { validateAnalyticsQuery } from '../middleware/validator';
import { strictLimiter } from '../middleware/rateLimiter';
import * as analyticsService from '../services/analyticsService';
import type { AnalyticsFilters } from '../types/index';
```

- `Router` is Express's **mini-app** — a standalone request router you can mount at any path. We'll build a router here and attach it in `index.ts` at `/api/analytics`.
- The fourth import is new syntax: `import * as analyticsService from '...'`. This is a **namespace import**. Instead of pulling specific names with curly braces, we grab everything the file exports and bundle it under one name. Then we call `analyticsService.getOverview(...)`, `analyticsService.getTimeSeries(...)`, etc. — reads well at call sites.

```typescript
const router = Router();
```

Call the `Router` factory (no `new` — it's a factory function, not a class). We now have a fresh router to configure.

```typescript
router.use(validateAnalyticsQuery);
```

`router.use(middleware)` registers middleware that runs for **every** route registered on this router. Without this line, you'd have to add `validateAnalyticsQuery` as an argument to every `router.get(...)` call below. Registering it once at the top is cleaner and harder to forget.

```typescript
function getFilters(req: Request): AnalyticsFilters {
  return {
    startDate: req.query.startDate as string | undefined,
    endDate: req.query.endDate as string | undefined,
    category: req.query.category as string | undefined,
    device: req.query.device as string | undefined,
  };
}
```

A little helper so we don't repeat this block in every route.

- `req.query.startDate as string | undefined` — type assertion. `req.query` values can technically be strings, arrays, or undefined. We tell TypeScript "treat this as either a string or undefined" — because we already validated the shape in middleware. Anything that didn't match the expected shape would have been rejected before reaching this handler.

```typescript
router.get('/overview', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const filters = getFilters(req);
    const stats = await analyticsService.getOverview(filters);
    res.json(stats);
  } catch (err) {
    next(err);
  }
});
```

The canonical route shape. Break it down:

- `router.get(path, handler)` — register a GET handler for `path`. The full path becomes `/api/analytics/overview` after we mount the router at `/api/analytics` in `index.ts`.
- The handler is an **async arrow function**. It needs to be async because we `await` the service call.
- `try { ... } catch (err) { next(err); }` — **error forwarding**. If the service throws (DB down, bad SQL, etc.), we catch it and pass the error to `next`. Calling `next(err)` with a non-null argument **skips all normal middleware and jumps to the error handler**. This is how async errors reach your centralized error handler.
- `res.json(stats)` — serializes `stats` to JSON and sends it with status 200.

```typescript
router.get('/events', strictLimiter, async (req, res, next) => {
```

Same shape, but notice the extra argument: `strictLimiter`. `router.get` accepts any number of middleware arguments between the path and the final handler. They run in order. So for this route only, Express runs: `validateAnalyticsQuery` (from `router.use` up top) → `strictLimiter` → the handler.

```typescript
const page = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 20;
```

- `Number(req.query.page)` — convert the string query param to a number. If it's undefined, the result is `NaN`.
- `|| 1` — fallback: `NaN` is falsy, so `|| 1` supplies a default. If the user passes `?page=3`, we get `3`; if they don't pass `page` at all, we get `1`.

```typescript
export default router;
```

Export the configured router as the file's default export. `index.ts` will import it and mount it under `/api/analytics`.

> ⚠️ **Common Mistake: Forgetting `next(err)` in Async Handlers**
> If an async route handler throws and you don't `catch` it, the promise rejects silently. Express never sees the error, the client's request hangs until it times out, and your error handler never fires. The `try / catch / next(err)` pattern is mandatory in Express 4 async routes.

---

## 7. Server Entry Point — Wiring It All Together

> **Key Concept: Middleware Order Matters**
> **Technical:** Express runs middleware in the order they are registered. Logger before everything (so it logs everything). Rate limiter before routes (so blocked requests don't hit the database). JSON parser before routes (so `req.body` is populated). Routes in the middle. Error handler last (so it catches errors from anything above).
>
> **Plain English:** Order is everything. If you put the bouncer after the bar, drunk people get drinks first and *then* get carded. Get the order right and the system works. Get it wrong and you have either security holes or broken behavior.

### 🏗️ Your Turn

Predict: what would break if you registered the rate limiter *after* `app.use('/api/analytics', analyticsRoutes)`?

<details>
<summary>See the answer</summary>

The rate limiter would never run for analytics routes, because Express stops at the first matching handler that sends a response. The route handler at `/api/analytics/...` matches and responds, ending the request before the rate limiter ever gets a chance.

The same logic applies to the error handler: if you register it *before* routes, it never sees route errors because routes haven't run yet. **Error handlers must be last.**

</details>

Create `server/src/index.ts`:

```typescript
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { httpLogger, logger } from './middleware/logger';
import { generalLimiter } from './middleware/rateLimiter';
import { errorHandler } from './middleware/errorHandler';
import analyticsRoutes from './routes/analytics';
import pool from './db/index';

dotenv.config({ path: '../.env' });

const app = express();
const PORT = process.env.PORT || 3001;

// --- Middleware stack (order matters!) ---
app.use(httpLogger);                                          // 1. Log every request
app.use(generalLimiter);                                      // 2. Rate limit
app.use(cors({ origin: process.env.CORS_ORIGIN }));          // 3. Allow frontend
app.use(express.json());                                      // 4. Parse JSON bodies

// --- Routes ---
app.use('/api/analytics', analyticsRoutes);

// --- Health check ---
app.get('/api/health', async (_req, res) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({
      status: 'ok',
      database: 'connected',
      timestamp: result.rows[0].now,
    });
  } catch {
    res.status(503).json({ status: 'error', database: 'disconnected' });
  }
});

// --- Error handler (must be last) ---
app.use(errorHandler);

// --- Start server ---
app.listen(PORT, () => {
  logger.info(`Server running on http://localhost:${PORT}`);
});

export default app;
```

### Reading This File Line-by-Line

```typescript
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { httpLogger, logger } from './middleware/logger';
import { generalLimiter } from './middleware/rateLimiter';
import { errorHandler } from './middleware/errorHandler';
import analyticsRoutes from './routes/analytics';
import pool from './db/index';
```

A parade of imports. Three defaults (`express`, `cors`, `dotenv`), three named imports from our own middleware, then a default import of the router we just built. Every one is something we're about to assemble into a complete server.

```typescript
dotenv.config({ path: '../.env' });
```

Load environment variables **before** anything else tries to read them. If you read `process.env.PORT` before calling `dotenv.config()`, you'll get `undefined` — timing matters.

```typescript
const app = express();
const PORT = process.env.PORT || 3001;
```

- `express()` — call the factory to create a new Express app. This is your main HTTP server object.
- `process.env.PORT || 3001` — read the `PORT` env var, fall back to `3001` locally. Hosting platforms like Render or Heroku set `PORT` automatically; locally, `3001` is a safe default.

```typescript
app.use(httpLogger);
app.use(generalLimiter);
app.use(cors({ origin: process.env.CORS_ORIGIN }));
app.use(express.json());
```

`app.use(middleware)` registers middleware globally — it runs for **every** request, in the order you register it. Here's what each line does:

1. **`httpLogger`** — our pino-based logger. Every incoming request gets logged (except `/api/health`).
2. **`generalLimiter`** — counts requests per IP. If a client exceeds 100/15 min, this layer sends a 429 and stops the request from reaching the route.
3. **`cors(...)`** — CORS (Cross-Origin Resource Sharing) is a browser-enforced security rule. If your frontend at `http://localhost:5173` tries to fetch from `http://localhost:3001`, the browser blocks it unless the server sends `Access-Control-Allow-Origin: http://localhost:5173` back. `cors({ origin: ... })` adds those headers. `process.env.CORS_ORIGIN` holds the allowed origin for your deployment.
4. **`express.json()`** — parses JSON request bodies. After this middleware, `req.body` contains the decoded object for `POST`/`PUT` requests with `Content-Type: application/json`. Without it, `req.body` is `undefined`.

```typescript
app.use('/api/analytics', analyticsRoutes);
```

Mount the router at `/api/analytics`. A route registered as `/overview` inside the router becomes `/api/analytics/overview` at the app level. This is how Express "nests" routers into prefixes.

```typescript
app.get('/api/health', async (_req, res) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({
      status: 'ok',
      database: 'connected',
      timestamp: result.rows[0].now,
    });
  } catch {
    res.status(503).json({ status: 'error', database: 'disconnected' });
  }
});
```

- `app.get(...)` registers a route directly on the app (no separate router).
- `_req` — underscore prefix because we don't use the request object. Just a signal for readers/linters.
- `pool.query('SELECT NOW()')` — the simplest possible SQL. `NOW()` is a Postgres function returning the current time. If the DB is reachable, this succeeds.
- `catch { ... }` — TypeScript 4+ lets you omit the error variable when you don't need it.
- `503 Service Unavailable` — the right status when a dependency (the DB) is down, not the server itself.

```typescript
app.use(errorHandler);
```

Error handler registered **last**. Because of Express's special arity detection (four parameters = error handler), this middleware only fires when someone calls `next(err)` with a non-null error.

```typescript
app.listen(PORT, () => {
  logger.info(`Server running on http://localhost:${PORT}`);
});
```

- `app.listen(port, callback)` — start the HTTP server on `port`. The callback fires once it's actually listening.
- `logger.info(...)` — structured info-level log. Uses our pino logger (not `console.log`).

```typescript
export default app;
```

Export `app` as default. Exporting is optional in the entry file — we don't import it anywhere — but it's useful if you later write integration tests that spin up the app without `.listen()`.

> **Spatial Note: The Health Check Has Special Status**
> Notice the health check tries a real database query. It is not enough to say "Express is running" — you want to know "Express *and* the database are reachable." If the DB is down, return `503 Service Unavailable` so load balancers can stop routing to this instance.

---

## 8. Test the API

Start the server:

```bash
cd server && npm run dev
```

Open a separate terminal so the server can keep running. Test each endpoint:

### Health Check

```bash
curl http://localhost:3001/api/health | jq
```

Expected:

```json
{
  "status": "ok",
  "database": "connected",
  "timestamp": "2025-03-01T..."
}
```

### Overview Stats

```bash
curl "http://localhost:3001/api/analytics/overview" | jq
```

Expected (your numbers will vary):

```json
{
  "total_page_views": 2981,
  "total_signups": 812,
  "total_revenue": 42891.40,
  "avg_session_duration": 272,
  "period_start": "2024-12-01T...",
  "period_end": "2025-03-01T..."
}
```

### With Filters

```bash
curl "http://localhost:3001/api/analytics/overview?category=marketing&startDate=2025-02-01&endDate=2025-02-28" | jq
```

Notice how the numbers shrink — you are now only counting events that match all three filters.

### Time Series

```bash
curl "http://localhost:3001/api/analytics/time-series?startDate=2025-02-01&endDate=2025-02-28" | jq
```

You should see one row per day in February.

### Category Breakdown

```bash
curl "http://localhost:3001/api/analytics/categories" | jq
```

### Device Breakdown

```bash
curl "http://localhost:3001/api/analytics/devices" | jq
```

Expected shape:

```json
[
  { "device": "desktop", "count": 3367, "percentage": 0.62 },
  { "device": "mobile", "count": 1681, "percentage": 0.31 },
  { "device": "tablet", "count": 379, "percentage": 0.07 }
]
```

The percentages should roughly match the weighted distribution from the seed (`62% / 31% / 7%`). If they're way off, your randomness is weirdly skewed — re-seed.

### Paginated Events

```bash
curl "http://localhost:3001/api/analytics/events?page=1&limit=5" | jq
```

You should see `events` array with 5 items, plus `total`, `page`, `limit`, `total_pages` fields.

### Top Pages

```bash
curl "http://localhost:3001/api/analytics/top-pages" | jq
```

### Validation Errors

```bash
# Bad date format
curl "http://localhost:3001/api/analytics/overview?startDate=not-a-date" | jq
```

Expected: `{"error":"Invalid startDate format. Use YYYY-MM-DD."}` with status 400.

```bash
# Bad category
curl "http://localhost:3001/api/analytics/overview?category=invalid" | jq
```

Expected: `{"error":"Invalid category. Must be one of: marketing, product, support, engineering"}`.

### ✅ Checkpoint

- [ ] Health check returns `status: "ok"` and a real timestamp
- [ ] All six analytics endpoints return JSON with the expected shape
- [ ] Filtered queries return smaller numbers than unfiltered ones
- [ ] Validation rejects bad dates and categories with status 400
- [ ] Server logs (in your dev terminal) show structured request entries via pino-pretty

If anything fails:

1. **Server won't start** → check `.env` has `DATABASE_URL`, and that `npm install` completed
2. **`database: "disconnected"`** → check Postgres is running and `DATABASE_URL` is correct
3. **All endpoints return 0** → did you run the seed script?
4. **Rate limit hits early** → wait 15 minutes or restart the server (in-memory counter resets)

### 🧠 Debugging Exercise

You stop the server, delete the `errorHandler` import, restart, and hit a route that throws. What does the client see, and what does it NOT see?

<details>
<summary>See the answer</summary>

Without the error handler registered, Express falls back to its default error handler. In development, that returns the full error stack trace as HTML. In production (`NODE_ENV=production`), it returns a generic `Internal Server Error` HTML page.

What it does NOT do:
- Return JSON (so your frontend, which expects JSON, will fail to parse the response)
- Log structurally (the error is dumped to stderr in unstructured format)
- Hide internal details in production reliably

This is why a custom error handler is non-optional in any real API.

</details>

---

## 9. Commit

> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: add backend with middleware stack, service layer, and analytics API"
```

---

## 🧠 Spatial Check-In

These questions test whether you internalized the *why*, not just the *what*. Try to answer in plain English before peeking.

1. **A teammate moves all the SQL queries from `analyticsService.ts` directly into the route handlers in `analytics.ts`. The app still works. Why is this a bad change?**

<details>
<summary>Check Your Answer</summary>

Three reasons:

- **Testability** — you can no longer call `getOverview(filters)` from a unit test. Every test now has to spin up Express, fake `req`/`res`, and walk through HTTP. That makes tests slow and flaky.
- **Reusability** — if a CLI script or a scheduled job needs the same overview data, it has to either re-implement the SQL or boot the HTTP server. Neither is reasonable.
- **Readability** — a route handler with embedded SQL forces the reader to mentally separate "is this HTTP code or data code?" while reading. Splitting the layers makes each file's purpose obvious.

The change "works" — but maintainability collapses as the app grows.

</details>

2. **What does `COUNT(*) FILTER (WHERE event_type = 'signup')` do, and why is it useful?**

<details>
<summary>Check Your Answer</summary>

It counts only rows where `event_type = 'signup'`, ignoring others — and you can put several of these in one query. So you can count signups, page views, and revenue in a single query without subqueries:

```sql
SELECT
  COUNT(*) FILTER (WHERE event_type = 'page_view') AS views,
  COUNT(*) FILTER (WHERE event_type = 'signup')    AS signups,
  COALESCE(SUM(value), 0)                          AS revenue
FROM analytics_events
WHERE category = 'marketing';
```

Without `FILTER`, you'd need three separate queries or messy `CASE WHEN` expressions inside `SUM`.

</details>

3. **The `/api/analytics/events` endpoint has a stricter rate limit than `/overview`. Why?**

<details>
<summary>Check Your Answer</summary>

`/events` is more expensive: it runs two queries (a `COUNT(*)` and a paginated `SELECT *`), and the result set is larger to serialize. A client hammering this endpoint can slow the database for everyone.

`/overview` runs a single aggregation that returns one row. It is cheap to serve. Letting it use the general 100/15min limit is fine.

The deeper principle: **rate limits should match cost.** Cheap endpoints get loose limits; expensive endpoints get tight ones.

</details>

4. **Why does the error handler send `err.message` in development but `'Internal server error'` in production?**

<details>
<summary>Check Your Answer</summary>

Error messages can leak sensitive details: SQL syntax, file paths, internal hostnames, library versions. In production, attackers probe for vulnerabilities by reading error responses to map your stack. A generic message reveals nothing — but the full error is still logged server-side so you can debug.

In development, *you* are the user, so seeing the real message saves you debugging time. There are no attackers.

</details>

5. **Look at this middleware registration order. What's wrong?**

```typescript
app.use(express.json());
app.use(errorHandler);              // ← here
app.use('/api/analytics', analyticsRoutes);
app.use(httpLogger);
```

<details>
<summary>Check Your Answer</summary>

Two problems:

- **`errorHandler` is registered before routes.** It never sees route errors because the routes haven't run yet when the error handler is registered. Errors fall through to the Express default handler.
- **`httpLogger` is last.** It only logs requests *after* routes have already responded — which means it logs nothing, because Express doesn't keep running middleware after a response is sent.

Correct order:

```typescript
app.use(httpLogger);      // first — log everything
app.use(express.json());  // parse bodies
app.use('/api/analytics', analyticsRoutes);
app.use(errorHandler);    // last — catch errors
```

</details>

6. **A junior dev writes:**

```typescript
const result = await pool.query(
  `SELECT * FROM analytics_events WHERE category = '${req.query.category}'`
);
```

**What is wrong, and what is the worst thing a malicious user could do?**

<details>
<summary>Check Your Answer</summary>

This is **SQL injection**. The user-supplied `category` is concatenated directly into the SQL string, with no separation between code and data.

A malicious user could send:

```
?category=marketing'; DROP TABLE analytics_events; --
```

The resulting SQL becomes:

```sql
SELECT * FROM analytics_events WHERE category = 'marketing'; DROP TABLE analytics_events; --'
```

Two statements: the legitimate `SELECT`, then a `DROP TABLE` that destroys your data. The `--` comments out the trailing quote so PostgreSQL doesn't error.

The fix is to use parameterized queries:

```typescript
await pool.query(
  'SELECT * FROM analytics_events WHERE category = $1',
  [req.query.category]
);
```

Now `req.query.category` is a value, never SQL grammar. Even `'; DROP TABLE...'` is treated as the literal string `"'; DROP TABLE..."` for comparison.

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 4 Overview](../) | [04 — State Management →](../04-state/) |
