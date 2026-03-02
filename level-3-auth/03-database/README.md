`Level 3` **Step 3 of 8** — Database Design

# 03 — Database Design: Users and Notes

## Spatial Orientation

```
vault-note/
├── client/          ← NOT working here right now
└── server/
    └── src/
        └── db/                ← ★ WE ARE HERE ★
            ├── schema.sql     ← Table definitions (we'll build this)
            └── index.ts       ← Database connection (we'll build this)
```

**What layer are we in?** The DATA layer. We're designing tables for users AND notes, then connecting to PostgreSQL. Same pattern as Level 2, but now with a users table and the relationship "notes belong to users."

---

## Step 1: Create the Folder Structure

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
mkdir -p server/src/db
mkdir -p server/src/routes
mkdir -p server/src/middleware
mkdir -p server/src/types
```

---

## Step 2: Design the Schema

VaultNote needs two tables:

```
┌──────────────────────────────────┐       ┌──────────────────────────────────┐
│            users                  │       │             notes                 │
├──────────────────────────────────┤       ├──────────────────────────────────┤
│ id             SERIAL PK         │───┐   │ id             SERIAL PK         │
│ email          VARCHAR(255) UNQ  │   │   │ user_id        INTEGER FK ──────┤
│ password_hash  VARCHAR(255)      │   └──▶│                REFERENCES        │
│ role           VARCHAR(20)       │       │                users(id)         │
│ created_at     TIMESTAMP         │       │                ON DELETE CASCADE  │
│                                  │       │ title          VARCHAR(200)      │
│                                  │       │ content        TEXT               │
│                                  │       │ created_at     TIMESTAMP          │
│                                  │       │ updated_at     TIMESTAMP          │
└──────────────────────────────────┘       └──────────────────────────────────┘
       ONE user                                  MANY notes
```

**The relationship**: One user has many notes. Each note belongs to exactly one user. Same one-to-many pattern as Level 2 (projects → tasks), but now the parent is a user.

**Key difference from Level 2**: The `users` table stores `password_hash`, not `password`. The plain text password never touches the database.

### Create the Schema File

In VS Code, create `server/src/db/schema.sql`:

```sql
-- VaultNote Database Schema
-- Run with: psql vaultnote < server/src/db/schema.sql

DROP TABLE IF EXISTS notes CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- ─── USERS TABLE ────────────────────────────────────────────────

CREATE TABLE users (
  id             SERIAL PRIMARY KEY,
  email          VARCHAR(255) UNIQUE NOT NULL,
  password_hash  VARCHAR(255) NOT NULL,
  role           VARCHAR(20) DEFAULT 'user',
  created_at     TIMESTAMP DEFAULT NOW()
);

-- ─── NOTES TABLE ────────────────────────────────────────────────

CREATE TABLE notes (
  id          SERIAL PRIMARY KEY,
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title       VARCHAR(200) NOT NULL,
  content     TEXT DEFAULT '',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

### New Column Types Explained

**`email VARCHAR(255) UNIQUE NOT NULL`**

| Part | What It Does |
|------|-------------|
| `VARCHAR(255)` | Text up to 255 characters |
| `UNIQUE` | No two users can have the same email |
| `NOT NULL` | Every user must have an email |

> [!NOTE]
> **Technical**: The `UNIQUE` constraint creates a unique index on the column, enforcing that no duplicate values can exist. Attempting to INSERT a duplicate email returns a constraint violation error.

> [!NOTE]
> **Plain English**: If alice@example.com already has an account, nobody else can register with that email. The database enforces this — your code doesn't need to check for duplicates manually (though it should handle the error gracefully).

**`password_hash VARCHAR(255) NOT NULL`**

The column is called `password_hash`, not `password`. This is intentional — it reminds everyone reading the schema that we store hashes, never plain text passwords. bcrypt hashes are 60 characters, but 255 gives room for different hashing algorithms.

**`role VARCHAR(20) DEFAULT 'user'`**

Stores the user's role. New users get `'user'` by default. An admin can be created by manually updating the database:

```sql
UPDATE users SET role = 'admin' WHERE email = 'admin@example.com';
```

We don't build a UI for creating admins — that would be a security risk. Admin promotion happens through direct database access only.

**`user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE`**

Same pattern as Level 2's `project_id`. Every note belongs to a user. If a user is deleted, all their notes are deleted too.

---

## Step 3: Run the Schema

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
psql vaultnote < server/src/db/schema.sql
```

Expected output:

```
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
```

### Verify

```bash
psql vaultnote -c "\dt"
```

You should see both `users` and `notes` tables.

```bash
psql vaultnote -c "\d users"
```

Verify the columns match the schema.

---

## Step 4: Build the Database Connection

Same pattern as Level 2. Create `server/src/db/index.ts`:

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

pool.query('SELECT NOW()')
  .then(() => {
    console.log('Connected to PostgreSQL');
  })
  .catch((err) => {
    console.error('Failed to connect to PostgreSQL:', err.message);
    process.exit(1);
  });

export default pool;
```

Identical to Level 2. The connection pool doesn't change — it's the routes that are different.

---

## Step 5: Test the Connection

Create a temporary `server/src/index.ts` to test:

```typescript
import dotenv from 'dotenv';
dotenv.config();

import pool from './db';

async function testConnection() {
  try {
    const result = await pool.query('SELECT NOW() as current_time');
    console.log('Database time:', result.rows[0].current_time);

    const tables = await pool.query(`
      SELECT table_name
      FROM information_schema.tables
      WHERE table_schema = 'public'
    `);
    console.log('Tables:', tables.rows.map(r => r.table_name));

    process.exit(0);
  } catch (err) {
    console.error('Connection test failed:', err);
    process.exit(1);
  }
}

testConnection();
```

Run it:

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
cd server && npx ts-node src/index.ts && cd ..
```

Expected:

```
Connected to PostgreSQL
Database time: 2026-03-02T...
Tables: [ 'users', 'notes' ]
```

---

## Step 6: Commit

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add server/src/db/
git commit -m "feat: add database schema with users and notes tables"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why is the column called `password_hash` and not `password`?**

<details><summary>Answer</summary>

To make it clear that we store hashed passwords, never plain text. The name reminds everyone — developers, DBAs, auditors — that the actual password is never in the database.

</details>

2. **What does the UNIQUE constraint on `email` do?**

<details><summary>Answer</summary>

Prevents two users from registering with the same email. The database rejects duplicate insertions with a constraint violation error.

</details>

3. **How does the `user_id` foreign key in `notes` relate to auth?**

<details><summary>Answer</summary>

Every note belongs to a user. When a user makes a request, the auth middleware identifies them (via JWT). The route handler then queries: `SELECT * FROM notes WHERE user_id = authenticated_user_id`. This is how we ensure users only see their own notes.

</details>

---

| | | |
|:---|:---:|---:|
| [← 02 — Project Setup](../02-project-setup/) | [Level 3 Overview](../) | [04 — Authentication →](../04-auth/) |
