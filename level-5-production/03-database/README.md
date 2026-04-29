`Level 5` **Step 3 of 9** — Database

# 03 — Database: Complex Schema with Relationships

## Spatial Orientation

CollabBoard has the most complex schema you've built — six tables with foreign keys, a many-to-many relationship, position tracking for ordered items, and cascading deletes.

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATABASE SCHEMA                           │
│                                                                  │
│  ┌──────────┐     ┌──────────────┐     ┌──────────┐            │
│  │  users   │────▶│ board_members │◀────│  boards  │            │
│  │          │     │  (many-to-   │     │          │            │
│  │  id ●    │     │   many)      │     │  id ●    │            │
│  │  email   │     │  board_id ○  │     │  name    │            │
│  │  password│     │  user_id ○   │     │  owner_id○──▶ users  │
│  │  display │     │  role        │     │          │            │
│  └────┬─────┘     └──────────────┘     └────┬─────┘            │
│       │                                      │                  │
│       │                                ┌─────▼─────┐            │
│       │                                │   lists   │            │
│       │                                │           │            │
│       │                                │  id ●     │            │
│       │                                │  board_id○│            │
│       │                                │  name     │            │
│       │                                │  position │            │
│       │                                └─────┬─────┘            │
│       │                                      │                  │
│       │                                ┌─────▼─────┐            │
│       │                                │   cards   │            │
│       │                                │           │            │
│       ├───────────────────────────────▶│  id ●     │            │
│       │  (assignee_id)                 │  list_id ○│            │
│       │                                │  title    │            │
│       │                                │  position │            │
│       │                                │assignee_id○            │
│       │                                └─────┬─────┘            │
│       │                                      │                  │
│       │                                ┌─────▼─────┐            │
│       └───────────────────────────────▶│ comments  │            │
│          (user_id)                     │           │            │
│                                        │  id ●     │            │
│                                        │  card_id ○│            │
│                                        │  user_id ○│            │
│                                        │  content  │            │
│                                        └───────────┘            │
│                                                                  │
│  ● = Primary Key    ○ = Foreign Key    ──▶ = References         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Schema

> [!IMPORTANT]
> **You should be in:** `collab-board/`

Create `server/src/db/schema.sql`:

```sql
-- CollabBoard database schema
-- 6 tables with foreign keys, many-to-many, and position tracking

-- Drop in reverse dependency order
DROP TABLE IF EXISTS comments;
DROP TABLE IF EXISTS cards;
DROP TABLE IF EXISTS lists;
DROP TABLE IF EXISTS board_members;
DROP TABLE IF EXISTS boards;
DROP TABLE IF EXISTS users;

-- 1. Users
CREATE TABLE users (
  id             SERIAL PRIMARY KEY,
  email          VARCHAR(255) UNIQUE NOT NULL,
  password_hash  VARCHAR(255) NOT NULL,
  display_name   VARCHAR(100) NOT NULL,
  created_at     TIMESTAMP DEFAULT NOW()
);

-- 2. Boards
CREATE TABLE boards (
  id             SERIAL PRIMARY KEY,
  name           VARCHAR(100) NOT NULL,
  description    TEXT,
  owner_id       INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at     TIMESTAMP DEFAULT NOW(),
  updated_at     TIMESTAMP DEFAULT NOW()
);

-- 3. Board Members (many-to-many: users ↔ boards)
CREATE TABLE board_members (
  id             SERIAL PRIMARY KEY,
  board_id       INTEGER NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  user_id        INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role           VARCHAR(20) DEFAULT 'member',
  joined_at      TIMESTAMP DEFAULT NOW(),
  UNIQUE(board_id, user_id)
);

-- 4. Lists (columns on a board)
CREATE TABLE lists (
  id             SERIAL PRIMARY KEY,
  board_id       INTEGER NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  name           VARCHAR(100) NOT NULL,
  position       INTEGER NOT NULL DEFAULT 0,
  created_at     TIMESTAMP DEFAULT NOW()
);

-- 5. Cards (items in a list)
CREATE TABLE cards (
  id             SERIAL PRIMARY KEY,
  list_id        INTEGER NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
  title          VARCHAR(200) NOT NULL,
  description    TEXT,
  position       INTEGER NOT NULL DEFAULT 0,
  assignee_id    INTEGER REFERENCES users(id) ON DELETE SET NULL,
  created_at     TIMESTAMP DEFAULT NOW(),
  updated_at     TIMESTAMP DEFAULT NOW()
);

-- 6. Comments
CREATE TABLE comments (
  id             SERIAL PRIMARY KEY,
  card_id        INTEGER NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  user_id        INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content        TEXT NOT NULL,
  created_at     TIMESTAMP DEFAULT NOW()
);

-- Indexes for common query patterns
CREATE INDEX idx_boards_owner      ON boards(owner_id);
CREATE INDEX idx_board_members_bid ON board_members(board_id);
CREATE INDEX idx_board_members_uid ON board_members(user_id);
CREATE INDEX idx_lists_board       ON lists(board_id);
CREATE INDEX idx_lists_position    ON lists(board_id, position);
CREATE INDEX idx_cards_list        ON cards(list_id);
CREATE INDEX idx_cards_position    ON cards(list_id, position);
CREATE INDEX idx_cards_assignee    ON cards(assignee_id);
CREATE INDEX idx_comments_card     ON comments(card_id);
```

### Reading This Schema Line-by-Line

Most of the SQL grammar (`CREATE TABLE`, `SERIAL PRIMARY KEY`, `VARCHAR`, `NOT NULL`, `DEFAULT`, `REFERENCES ... ON DELETE CASCADE`, `CREATE INDEX`) was covered in Levels 2 and 3. Level 5 introduces three new things — let's walk them.

```sql
DROP TABLE IF EXISTS comments;
DROP TABLE IF EXISTS cards;
DROP TABLE IF EXISTS lists;
DROP TABLE IF EXISTS board_members;
DROP TABLE IF EXISTS boards;
DROP TABLE IF EXISTS users;
```

**Reverse-dependency drop order**. Foreign keys mean child tables must be dropped before parent tables. Dropping `users` first would fail because `boards.owner_id` still references it. We drop in **reverse** of the order we create — comments before cards before lists before boards before users.

```sql
CREATE TABLE board_members (
  id             SERIAL PRIMARY KEY,
  board_id       INTEGER NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  user_id        INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role           VARCHAR(20) DEFAULT 'member',
  joined_at      TIMESTAMP DEFAULT NOW(),
  UNIQUE(board_id, user_id)
);
```

The **junction table** for the many-to-many relationship between users and boards. Two new ideas:

- **Two foreign keys in one row** — `board_id` references `boards`, `user_id` references `users`. Together they pair a user with a board. Each `id` (the row's own primary key) is meaningless on its own — what matters is the `(board_id, user_id)` pair.
- **`UNIQUE(board_id, user_id)`** — a **composite unique constraint**. The pair must be unique across the table. You cannot insert two rows with the same `(board_id, user_id)` combination. So a user can join a board only once. Without this, your "Add member" feature could create duplicate memberships.

```sql
assignee_id    INTEGER REFERENCES users(id) ON DELETE SET NULL,
```

**`ON DELETE SET NULL`** — a new variant of the foreign-key cascade rule. When the referenced user is deleted, instead of deleting the card too (`CASCADE`) or refusing the delete (`RESTRICT`), this **sets the column to NULL**. The card stays in the table; it just no longer has an assignee.

Used here because the real-world rule is: when someone leaves the team, their cards stay (someone else can pick them up). Compare with `comments.user_id`, which uses `CASCADE` because a comment has no meaning without its author.

```sql
CREATE INDEX idx_lists_position    ON lists(board_id, position);
CREATE INDEX idx_cards_position    ON cards(list_id, position);
```

**Composite indexes** — indexes on more than one column. The order matters: this index is optimized for queries that filter by `board_id` first, then sort by `position`. Exactly the query pattern of "show me all lists on board X in display order."

A single-column index on `position` would be useless on its own (positions repeat across boards). A composite index on `(board_id, position)` lets PostgreSQL find all lists for a board AND have them pre-sorted in one go.

### Table-by-Table Breakdown

**users** — same as Level 3, plus `display_name`

| Column | Purpose |
|--------|---------|
| `display_name` | Shown on cards and comments instead of email |

**boards** — projects / workspaces

| Column | Purpose |
|--------|---------|
| `owner_id` | The user who created the board (has admin rights) |
| `ON DELETE CASCADE` | If the owner's account is deleted, their boards are too |

**board_members** — the many-to-many relationship

| Column | Purpose |
|--------|---------|
| `board_id` + `user_id` | Links a user to a board |
| `role` | "owner" or "member" — determines permissions |
| `UNIQUE(board_id, user_id)` | Prevents adding the same user twice |

> [!NOTE]
> **Technical:** A many-to-many relationship requires a **junction table** (also called a join table or bridge table). Users can belong to multiple boards, and boards can have multiple members. The `board_members` table stores each user-board pair. Without it, you'd need an array column (PostgreSQL supports this) or denormalized data.
>
> **Plain English:** Think of a school. Students take many classes, and classes have many students. You can't put all class names in the student row or all student names in the class row. You need a third table — "enrollments" — that pairs each student with each class. `board_members` is the enrollment table for boards.

**lists** — columns on a board (To Do, In Progress, Done)

| Column | Purpose |
|--------|---------|
| `position` | Integer that determines display order (0, 1, 2...) |
| `ON DELETE CASCADE` | Deleting a board deletes all its lists |

**cards** — items in a list

| Column | Purpose |
|--------|---------|
| `list_id` | Which list this card belongs to |
| `position` | Display order within the list |
| `assignee_id` | User assigned to this card (nullable) |
| `ON DELETE SET NULL` | If the assignee is deleted, the card stays but loses its assignee |
| `ON DELETE CASCADE` (via list) | Deleting a list deletes all its cards |

**comments** — discussion on cards

| Column | Purpose |
|--------|---------|
| `card_id` | Which card this comment belongs to |
| `user_id` | Who wrote the comment |
| `ON DELETE CASCADE` | Deleting a card deletes all its comments |

### Position Tracking

Lists and cards have a `position` column for ordering. When you drag a card from position 2 to position 4:

```
BEFORE:                    AFTER:
position 0: Card A         position 0: Card A
position 1: Card B         position 1: Card C    ← shifted up
position 2: Card D  ←drag  position 2: Card E    ← shifted up
position 3: Card C         position 3: Card D    ← dropped here
position 4: Card E
```

The backend updates positions with a SQL transaction — all position changes happen atomically.

### Cascade Diagram

```
DELETE a user
  ├── CASCADE → boards they own are deleted
  │   ├── CASCADE → board_members for those boards
  │   ├── CASCADE → lists for those boards
  │   │   └── CASCADE → cards in those lists
  │   │       └── CASCADE → comments on those cards
  │   └── CASCADE → board_members for those boards
  ├── CASCADE → board_members where they're a member
  ├── CASCADE → comments they wrote
  └── SET NULL → cards they're assigned to (card stays, assignee becomes null)
```

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

---

> [!TIP]
> **Session Break** — You've designed the 6-table schema with foreign keys, many-to-many relationships, and cascade rules. Save your work and take a break.
> When you return, you'll write the seed script and verify the data with SQL queries.

---

## Step 3: Seed Script

The seed creates demo users, boards, lists, cards, and comments.

Create `server/src/db/seed.ts`:

```typescript
import bcrypt from 'bcryptjs';
import pool from './index';

async function seed(): Promise<void> {
  console.log('Seeding CollabBoard database...\n');

  // Clear existing data (reverse dependency order)
  await pool.query('DELETE FROM comments');
  await pool.query('DELETE FROM cards');
  await pool.query('DELETE FROM lists');
  await pool.query('DELETE FROM board_members');
  await pool.query('DELETE FROM boards');
  await pool.query('DELETE FROM users');

  // Reset sequences
  await pool.query("ALTER SEQUENCE users_id_seq RESTART WITH 1");
  await pool.query("ALTER SEQUENCE boards_id_seq RESTART WITH 1");
  await pool.query("ALTER SEQUENCE board_members_id_seq RESTART WITH 1");
  await pool.query("ALTER SEQUENCE lists_id_seq RESTART WITH 1");
  await pool.query("ALTER SEQUENCE cards_id_seq RESTART WITH 1");
  await pool.query("ALTER SEQUENCE comments_id_seq RESTART WITH 1");

  // --- Users ---
  const passwordHash = await bcrypt.hash('password123', 10);

  const users = await pool.query(
    `INSERT INTO users (email, password_hash, display_name) VALUES
      ('alex@example.com', $1, 'Alex Chen'),
      ('sam@example.com', $1, 'Sam Rivera'),
      ('jordan@example.com', $1, 'Jordan Lee')
     RETURNING id, email, display_name`,
    [passwordHash]
  );
  console.log('Created users:', users.rows.map(u => u.display_name).join(', '));

  const [alex, sam, jordan] = users.rows;

  // --- Boards ---
  const boards = await pool.query(
    `INSERT INTO boards (name, description, owner_id) VALUES
      ('Product Launch', 'Q2 product launch planning and execution', $1),
      ('Engineering Sprint', 'Current sprint tasks and bugs', $2)
     RETURNING id, name`,
    [alex.id, sam.id]
  );
  console.log('Created boards:', boards.rows.map(b => b.name).join(', '));

  const [productBoard, sprintBoard] = boards.rows;

  // --- Board Members ---
  // Alex owns Product Launch, Sam and Jordan are members
  // Sam owns Engineering Sprint, Alex is a member
  await pool.query(
    `INSERT INTO board_members (board_id, user_id, role) VALUES
      ($1, $2, 'owner'),
      ($1, $3, 'member'),
      ($1, $4, 'member'),
      ($5, $3, 'owner'),
      ($5, $2, 'member')`,
    [productBoard.id, alex.id, sam.id, jordan.id, sprintBoard.id]
  );
  console.log('Assigned board members');

  // --- Lists for Product Launch ---
  const productLists = await pool.query(
    `INSERT INTO lists (board_id, name, position) VALUES
      ($1, 'To Do', 0),
      ($1, 'In Progress', 1),
      ($1, 'Review', 2),
      ($1, 'Done', 3)
     RETURNING id, name`,
    [productBoard.id]
  );

  const [todo, inProgress, review, done] = productLists.rows;

  // --- Lists for Engineering Sprint ---
  const sprintLists = await pool.query(
    `INSERT INTO lists (board_id, name, position) VALUES
      ($1, 'Backlog', 0),
      ($1, 'In Progress', 1),
      ($1, 'Done', 2)
     RETURNING id, name`,
    [sprintBoard.id]
  );

  const [backlog, sprintInProgress, sprintDone] = sprintLists.rows;

  // --- Cards for Product Launch ---
  const cards = await pool.query(
    `INSERT INTO cards (list_id, title, description, position, assignee_id) VALUES
      ($1, 'Design landing page mockups', 'Create wireframes and high-fidelity designs for the launch page', 0, $5),
      ($1, 'Write launch blog post', 'Draft and edit the announcement blog post', 1, $7),
      ($1, 'Prepare demo video', NULL, 2, NULL),
      ($2, 'Build API endpoints', 'Implement REST API for the new features', 0, $6),
      ($2, 'Set up auth middleware', 'JWT authentication for the new API', 1, $6),
      ($3, 'Review PRs', 'Code review for sprint deliverables', 0, $5),
      ($4, 'Set up database schema', 'PostgreSQL tables and migrations', 0, $5),
      ($4, 'Configure CI/CD', 'GitHub Actions pipeline for testing', 1, $6)
     RETURNING id, title`,
    [todo.id, inProgress.id, review.id, done.id, alex.id, sam.id, jordan.id]
  );
  console.log(`Created ${cards.rows.length} cards for Product Launch`);

  // --- Cards for Engineering Sprint ---
  await pool.query(
    `INSERT INTO cards (list_id, title, description, position, assignee_id) VALUES
      ($1, 'Fix login redirect bug', 'Users are redirected to 404 after login', 0, $4),
      ($1, 'Add rate limiting', 'Implement express-rate-limit on auth endpoints', 1, NULL),
      ($2, 'Refactor user service', 'Extract auth logic into separate module', 0, $4),
      ($3, 'Update dependencies', 'Run npm audit and update vulnerable packages', 0, $5)`,
    [backlog.id, sprintInProgress.id, sprintDone.id, sam.id, alex.id]
  );
  console.log('Created cards for Engineering Sprint');

  // --- Comments ---
  const designCard = cards.rows[0]; // "Design landing page mockups"
  const apiCard = cards.rows[3];    // "Build API endpoints"

  await pool.query(
    `INSERT INTO comments (card_id, user_id, content) VALUES
      ($1, $3, 'I''ve started on the wireframes. Should have drafts by Friday.'),
      ($1, $4, 'Make sure to include mobile views too!'),
      ($1, $3, 'Good point — I''ll add responsive mockups.'),
      ($2, $5, 'Should we use REST or GraphQL for this?'),
      ($2, $4, 'Let''s stick with REST for now. We can migrate later if needed.')`,
    [designCard.id, apiCard.id, alex.id, sam.id, jordan.id]
  );
  console.log('Created comments');

  // --- Summary ---
  console.log('\n--- Seed Summary ---');
  console.log('Users:    3 (alex, sam, jordan — password: password123)');
  console.log('Boards:   2 (Product Launch, Engineering Sprint)');
  console.log('Lists:    7');
  console.log('Cards:   12');
  console.log('Comments:  5');
  console.log('\nDone!');

  await pool.end();
}

seed().catch((err) => {
  console.error('Seed failed:', err);
  process.exit(1);
});
```

### Reading This Seed Script

This is the most complex seed in the curriculum. Most of the patterns repeat from Level 4's seed — `pool.query`, `INSERT ... RETURNING`, the `await pool.end()` cleanup. The new pieces:

```typescript
await pool.query("ALTER SEQUENCE users_id_seq RESTART WITH 1");
```

**`ALTER SEQUENCE ... RESTART WITH 1`** — `SERIAL` columns use a hidden counter (a "sequence") to generate IDs. After deleting and re-inserting, the sequence keeps counting from where it left off. This SQL resets the counter to 1 so re-seeded IDs start fresh. Without it, your seeded users might end up with IDs 100, 101, 102 instead of 1, 2, 3.

```typescript
const [alex, sam, jordan] = users.rows;
```

**Array destructuring**. `users.rows` is an array; `[alex, sam, jordan]` pulls the first three rows into named variables. We use them below: `alex.id`, `sam.id`, `jordan.id`.

```sql
INSERT INTO board_members (board_id, user_id, role) VALUES
  ($1, $2, 'owner'),
  ($1, $3, 'member'),
  ($1, $4, 'member'),
  ($5, $3, 'owner'),
  ($5, $2, 'member')
```

**Multi-row `INSERT`**. One `INSERT` statement, five rows, all sent in a single trip to the database. Faster than five separate inserts (single round-trip, single transaction). Each parenthesized group is one row's values.

Notice how `$1` (the Product Launch board ID) is reused in three rows — the same parameter can appear multiple times in the SQL. We pass `[productBoard.id, alex.id, sam.id, jordan.id, sprintBoard.id]` as the params array.

```sql
'I''ve started on the wireframes...'
```

**Escaped single quote in SQL**. SQL strings use single quotes, so to include a literal single quote in the string, you double it. `'I''ve'` represents the string `I've`. The same letter doubled also works in many other databases.

Run the schema and seed:

```bash
psql collabboard < server/src/db/schema.sql
cd server && npm run seed
```

---

## Step 4: Verify with SQL

Open `psql collabboard` and try these queries:

### List all boards a user belongs to (JOIN)

```sql
SELECT b.id, b.name, bm.role
FROM boards b
JOIN board_members bm ON b.id = bm.board_id
WHERE bm.user_id = 1;
```

**Reading this query token-by-token:**

- `SELECT b.id, b.name, bm.role` — pick three columns from the joined result. Notice the `b.` and `bm.` prefixes — those are **table aliases** to disambiguate which table each column comes from.
- `FROM boards b` — start with the `boards` table, aliased as `b`. The space-then-letter syntax assigns the alias.
- `JOIN board_members bm ON b.id = bm.board_id` — combine each board with rows in `board_members` where `b.id` (board's own id) matches `bm.board_id` (the membership's reference to the board). One board with three members produces three rows in the joined result, one per member.
- `WHERE bm.user_id = 1` — keep only rows where this membership is for user 1. Net result: every board user 1 belongs to.

This JOIN finds boards where user 1 is a member. It returns the board name and the user's role.

### Get a board with its lists and card counts (JOIN + GROUP BY)

```sql
SELECT l.id, l.name, l.position, COUNT(c.id) AS card_count
FROM lists l
LEFT JOIN cards c ON l.id = c.list_id
WHERE l.board_id = 1
GROUP BY l.id, l.name, l.position
ORDER BY l.position;
```

`LEFT JOIN` ensures lists with zero cards still appear. `GROUP BY` groups cards by list.

### Get a card with its comments and authors (multi-table JOIN)

```sql
SELECT c.title, cm.content, u.display_name, cm.created_at
FROM cards c
JOIN comments cm ON c.id = cm.card_id
JOIN users u ON cm.user_id = u.id
WHERE c.id = 1
ORDER BY cm.created_at;
```

**Reading this multi-table JOIN:**

Three tables, two JOINs:

- `FROM cards c` — start with cards (alias `c`).
- `JOIN comments cm ON c.id = cm.card_id` — chain on each comment whose `card_id` matches a card's `id`.
- `JOIN users u ON cm.user_id = u.id` — chain again on each user whose `id` matches the comment's `user_id`. Now each result row contains: card columns + comment columns + user columns.
- `WHERE c.id = 1` — narrow to one specific card.
- `ORDER BY cm.created_at` — sort comments oldest first.

JOINs chain left-to-right. Conceptually, PostgreSQL builds an intermediate result after each JOIN, then chains again. The query optimizer reorders the actual execution using indexes — but the result is the same.

This chains two JOINs: cards → comments → users. Each comment includes the author's display name.

### Check many-to-many membership

```sql
SELECT u.display_name, b.name, bm.role
FROM board_members bm
JOIN users u ON bm.user_id = u.id
JOIN boards b ON bm.board_id = b.id
ORDER BY b.name, bm.role;
```

> [!NOTE]
> **Technical:** JOINs combine rows from multiple tables based on related columns. `INNER JOIN` returns only matching rows. `LEFT JOIN` returns all rows from the left table plus matching rows from the right (NULLs for non-matches). Multi-table JOINs chain these operations — the database engine optimizes the execution order using the indexes you created.
>
> **Plain English:** A JOIN is like matching nametags at a party. You have a list of boards and a list of members. A JOIN finds every board-member pair, connecting them by board_id. A LEFT JOIN keeps boards that have no members yet (lonely boards).

---

## Step 5: Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd ..
git add .
git commit -m "feat: add 6-table schema with many-to-many relationships and seed data"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why is `board_members` a separate table instead of an array column on `boards`?**

<details><summary>Answer</summary>

A separate table lets you query efficiently (find all boards for a user, find all members of a board), add metadata per membership (role, join date), and enforce uniqueness (UNIQUE constraint prevents duplicate memberships). Array columns can't have foreign key constraints, can't be indexed efficiently for membership lookups, and can't store per-membership metadata.

</details>

2. **Why does deleting a user SET NULL on `cards.assignee_id` instead of CASCADE?**

<details><summary>Answer</summary>

The card itself should survive when an assignee leaves. CASCADE would delete the card — losing the task and all its comments. SET NULL keeps the card but removes the assignment, so another user can be assigned. This models the real-world scenario: when an employee leaves, their tasks get reassigned, not deleted.

</details>

3. **Why do `lists` and `cards` have a `position` column?**

<details><summary>Answer</summary>

SQL tables have no inherent order — `SELECT * FROM lists` returns rows in an unpredictable order. The `position` column explicitly stores the display order. When a user drags a card from position 0 to position 2, the backend updates the position values. The frontend orders by `position` to display cards in the user's chosen order.

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 5 Overview](../) | [04 — Backend →](../04-backend/) |
