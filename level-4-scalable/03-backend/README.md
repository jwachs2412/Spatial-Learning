`Level 4` **Step 3 of 9** — Backend

# 03 — Backend: Middleware Stack and Analytics API

## Spatial Orientation

The backend has three new layers compared to previous levels: a **middleware stack** (logging, rate limiting, validation), a **service layer** (SQL queries separated from routes), and a **seed script** (generates realistic sample data).

```
REQUEST FLOW:

  Client Request
       │
       ▼
  ┌─────────────┐
  │   Logger     │  ← Records every request
  ├─────────────┤
  │ Rate Limiter │  ← Blocks excessive requests
  ├─────────────┤
  │    CORS      │  ← Allows frontend origin
  ├─────────────┤
  │ JSON Parser  │  ← Parses request body
  ├─────────────┤
  │  Validator   │  ← Validates query parameters
  ├─────────────┤
  │   Route      │  ← Calls service layer
  │   Handler    │
  ├─────────────┤
  │   Service    │  ← Executes SQL, returns data
  ├─────────────┤
  │   Error      │  ← Catches and formats errors
  │   Handler    │
  └─────────────┘
       │
       ▼
  JSON Response
```

---

## Step 1: Database Schema

> [!IMPORTANT]
> **You should be in:** `data-dash/`

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

Line-by-line breakdown:

| Column | Type | Purpose |
|--------|------|---------|
| `id` | `SERIAL PRIMARY KEY` | Auto-incrementing unique ID |
| `event_type` | `VARCHAR(50)` | What happened: page_view, signup, purchase, click |
| `category` | `VARCHAR(50)` | Business category: marketing, product, support, engineering |
| `value` | `DECIMAL(10,2)` | Revenue for purchases, 0 for other events |
| `country` | `VARCHAR(2)` | ISO country code: US, UK, DE, FR, JP |
| `device` | `VARCHAR(20)` | Device type: desktop, mobile, tablet |
| `browser` | `VARCHAR(30)` | Browser name: Chrome, Firefox, Safari, Edge |
| `page` | `VARCHAR(200)` | URL path: /home, /pricing, /dashboard, /docs |
| `session_id` | `VARCHAR(36)` | UUID grouping events from one visit |
| `created_at` | `TIMESTAMP` | When the event occurred |

> [!NOTE]
> **Technical:** Indexes (`CREATE INDEX`) are data structures that speed up queries filtering on specific columns. Without an index on `created_at`, a date range query scans every row. With an index, the database jumps directly to matching rows.
>
> **Plain English:** An index is like a book's index — instead of reading every page to find "middleware," you look up "middleware" in the index and go straight to page 47.

### Why One Table?

In production, you'd often have separate tables for different event types and pre-aggregated daily summaries for fast queries. For DataDash, one table teaches:

1. **SQL aggregation** — GROUP BY, COUNT, SUM, AVG
2. **Filtering with WHERE** — date ranges, categories, devices
3. **Index design** — which columns to index and why

Run the schema:

```bash
psql datadash < server/src/db/schema.sql
```

Expected output: `DROP TABLE` / `CREATE TABLE` / `CREATE INDEX` lines.

---

## Step 2: Database Connection

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

Same pattern as Level 2 — a connection pool shared across the app.

---

## Step 3: Seed Script

The seed script generates 90 days of realistic analytics data. Students run it once to populate the database.

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

Run the seed:

```bash
cd server && npm run seed
```

Expected output:

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

Your exact numbers will differ (random generation), but the proportions should be similar.

---

## Step 4: Middleware Stack

### Logger Middleware

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

Line-by-line:

| Line | Purpose |
|------|---------|
| `level: 'info' / 'debug'` | Production logs only important events; dev logs everything |
| `transport: pino-pretty` | Formats logs as readable text in dev (JSON in production) |
| `autoLogging.ignore` | Skips logging for the health check endpoint (it fires every 30 seconds) |

> [!NOTE]
> **Technical:** Structured logging outputs JSON objects (`{"level":"info","time":1234,"msg":"request completed","responseTime":23}`). This is machine-parseable — log aggregators (Datadog, Papertrail) can search, filter, and alert on structured fields. `console.log` outputs unstructured strings.
>
> **Plain English:** Structured logs are like spreadsheets — you can sort and filter by any column. `console.log` is like a notebook — you can only read it top to bottom.

Install pino-pretty for development:

```bash
npm install -D pino-pretty
```

### Rate Limiter Middleware

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

| Option | Value | Purpose |
|--------|-------|---------|
| `windowMs` | 15 minutes | Time window for counting requests |
| `max` | 100 | Maximum requests allowed in the window |
| `standardHeaders` | true | Sends `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` headers |
| `message` | JSON object | Response body when limit is exceeded (status 429) |

> [!NOTE]
> **Technical:** Rate limiting prevents abuse — a single client can't overwhelm your server with thousands of requests. The `windowMs` and `max` values define a sliding window. Headers tell the client how many requests they have left.
>
> **Plain English:** Rate limiting is like a bouncer at a club with a capacity limit. After 100 people enter in 15 minutes, new arrivals wait outside until the window resets.

### Validation Middleware

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

Validation middleware rejects bad requests **before they reach the route handler**. This keeps routes clean — they can trust their inputs are valid.

### Error Handler Middleware

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

Key details:
- Four parameters (`err, req, res, next`) — this is how Express identifies error-handling middleware
- The full error is logged server-side with pino (for debugging)
- The client sees a clean message — in production, never expose internal error details

---

## Step 5: Service Layer

The service layer contains all SQL queries and business logic. Routes call services; services call the database. This separation means:

1. Routes handle HTTP (parsing params, sending responses)
2. Services handle data (SQL queries, calculations)
3. You can test services independently of HTTP

Create `server/src/services/analyticsService.ts`:

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

### Key SQL Patterns

| Pattern | Example | Purpose |
|---------|---------|---------|
| `COUNT(*) FILTER (WHERE ...)` | `COUNT(*) FILTER (WHERE event_type = 'signup')` | Count only matching rows (PostgreSQL-specific) |
| `COALESCE(SUM(value), 0)` | If no rows match, return 0 instead of NULL |
| `DATE(created_at)` | Extract date from timestamp for GROUP BY |
| `COUNT(DISTINCT session_id)` | Count unique sessions, not total events |
| `LIMIT $1 OFFSET $2` | Pagination: skip rows, take rows |

> [!WARNING]
> **All query parameters use `$1`, `$2` placeholders.** Never concatenate user input into SQL strings. The `buildWhereClause` helper safely parameterizes all filter values.

---

## Step 6: Analytics Routes

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

Notice the pattern:

1. **Validation middleware** runs on every route via `router.use(validateAnalyticsQuery)`
2. **Rate limiting** is stricter on `/events` (expensive paginated query)
3. **Routes don't contain SQL** — they call the service layer
4. **Every route uses try/catch + `next(err)`** — errors flow to the error handler

---

## Step 7: Server Entry Point

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

> [!WARNING]
> **Middleware order matters.** The logger must be first (to log everything), and the error handler must be last (to catch errors from everything above). If you put the error handler before routes, it never catches route errors.

---

## Step 8: Test the API

Start the server:

```bash
cd server && npm run dev
```

In a separate terminal, test each endpoint:

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

### Time Series

```bash
curl "http://localhost:3001/api/analytics/time-series?startDate=2025-02-01&endDate=2025-02-28" | jq
```

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

### Paginated Events

```bash
curl "http://localhost:3001/api/analytics/events?page=1&limit=5" | jq
```

### Top Pages

```bash
curl "http://localhost:3001/api/analytics/top-pages" | jq
```

### Validation Errors

```bash
# Bad date format
curl "http://localhost:3001/api/analytics/overview?startDate=not-a-date" | jq
```

Expected: `{"error":"Invalid startDate format. Use YYYY-MM-DD."}`

```bash
# Bad category
curl "http://localhost:3001/api/analytics/overview?category=invalid" | jq
```

Expected: `{"error":"Invalid category. Must be one of: marketing, product, support, engineering"}`

---

## Step 9: Commit

> [!IMPORTANT]
> **You should be in:** `data-dash/`

```bash
cd ..
git add .
git commit -m "feat: add backend with middleware stack, service layer, and analytics API"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why is the service layer separate from routes?**

<details><summary>Answer</summary>

Separation of concerns. Routes handle HTTP (parsing query params, sending status codes). Services handle data (SQL queries, calculations). This means you can test services without HTTP, reuse them from different routes, and change the database layer without touching routes.

</details>

2. **What does `FILTER (WHERE event_type = 'signup')` do in SQL?**

<details><summary>Answer</summary>

It's a PostgreSQL extension to aggregate functions. `COUNT(*) FILTER (WHERE event_type = 'signup')` counts only rows where `event_type` is `'signup'`. Without FILTER, you'd need a subquery or CASE expression: `SUM(CASE WHEN event_type = 'signup' THEN 1 ELSE 0 END)`.

</details>

3. **Why does the `/events` endpoint have a stricter rate limit than `/overview`?**

<details><summary>Answer</summary>

The events endpoint returns paginated rows with a COUNT query — it's more expensive than the aggregation endpoints. A stricter rate limit prevents a client from hammering this endpoint and overloading the database. The overview endpoint aggregates in a single query and is cheaper to serve.

</details>

4. **Why does the error handler send `err.message` in development but a generic message in production?**

<details><summary>Answer</summary>

In development, seeing the actual error message helps you debug. In production, error messages might reveal internal details (database table names, file paths, SQL syntax) that help attackers. A generic "Internal server error" reveals nothing useful to an attacker while the full error is still logged server-side for debugging.

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 4 Overview](../) | [04 — State Management →](../04-state/) |
