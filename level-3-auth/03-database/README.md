# Step 3 — Database Design: Users and Notes

## Spatial Orientation

```
vault-note/
└── server/
    └── src/
        └── db/               ← YOU ARE HERE
            ├── schema.sql      ← Users + Notes tables
            └── index.ts        ← Database connection
```

---

## 1. Design the Schema

### 🏗️ Your Turn

Design a schema for VaultNote. Think about:
- A **users** table: what do you need for authentication? (email, password hash, role)
- A **notes** table: how do you link notes to users?
- What column stores the password? (Hint: it's NOT called `password`)
- Why do we store `password_hash` instead of `password`?

<details>
<summary>See the solution</summary>

Create `server/src/db/schema.sql`:

```sql
-- VaultNote Database Schema

CREATE TABLE users (
  id             SERIAL PRIMARY KEY,
  email          VARCHAR(255) UNIQUE NOT NULL,
  password_hash  VARCHAR(255) NOT NULL,
  role           VARCHAR(20) DEFAULT 'user',
  created_at     TIMESTAMP DEFAULT NOW()
);

CREATE TABLE notes (
  id          SERIAL PRIMARY KEY,
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title       VARCHAR(200) NOT NULL,
  content     TEXT DEFAULT '',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

</details>

### Reading This Schema Line-by-Line

You already learned the basic SQL DDL grammar in Level 2 (`CREATE TABLE`, `SERIAL PRIMARY KEY`, `VARCHAR(n)`, `NOT NULL`, `DEFAULT`, `REFERENCES ... ON DELETE CASCADE`). Level 3 adds one new constraint, plus two design choices worth dwelling on.

#### The `users` Table

- `id SERIAL PRIMARY KEY` — auto-incrementing integer ID. Same as Level 2.
- `email VARCHAR(255) UNIQUE NOT NULL`
  - `UNIQUE` is **new in this lesson**. It tells the database "no two rows in this table can share the same value in this column." If someone tries to register with an email already in `users`, PostgreSQL rejects the `INSERT` with a duplicate-key error.
  - PostgreSQL automatically builds an index on a `UNIQUE` column, so login lookups by email are fast.
  - 255 characters is the historical convention for email length — well above any realistic email.
- `password_hash VARCHAR(255) NOT NULL` — a string column that will hold bcrypt's output. **Notice the name.** The column is called `password_hash`, not `password`. That's deliberate: every developer reading this code is reminded that the value is a hash, never a plaintext password.
- `role VARCHAR(20) DEFAULT 'user'` — a small string for the user's role. New rows default to `'user'`; we'd manually `UPDATE` to `'admin'` for admin accounts.
- `created_at TIMESTAMP DEFAULT NOW()` — automatic creation timestamp.

#### The `notes` Table

- `user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE`
  - Same foreign-key pattern you saw in Level 2's `tasks.project_id`.
  - Every note must belong to a real user. If a user is deleted, all their notes are deleted too.
- `title VARCHAR(200) NOT NULL` — note title.
- `content TEXT DEFAULT ''` — the body. `TEXT` (no length limit) suits long content; default empty string means inserts can omit it.

**Key design decisions:**

| Column | Why This Design |
|--------|----------------|
| `email UNIQUE NOT NULL` | `UNIQUE` prevents duplicate accounts. `NOT NULL` ensures every user has an email. |
| `password_hash` (not `password`) | Naming it `password_hash` makes it clear — we NEVER store plain-text passwords. |
| `role DEFAULT 'user'` | New accounts are regular users by default. Admin role is set manually in the database. |
| `user_id REFERENCES users(id)` | Foreign key — every note belongs to a user. |
| `ON DELETE CASCADE` | Deleting a user deletes their notes. |

### 🔐 Security Exercise

Why is the column named `password_hash` and not `password`?

<details>
<summary>Answer</summary>

Naming it `password_hash` is a reminder that this column contains a **bcrypt hash**, not a plain-text password. If a developer sees a column called `password`, they might accidentally log it, display it, or compare it with `===` instead of `bcrypt.compare()`. The name itself communicates the security requirement.

</details>

### Run the Schema

```bash
psql vaultnote < server/src/db/schema.sql
```

---

## 2. Database Connection

Create `server/src/db/index.ts`:

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
  .catch((err: Error) => {
    console.error('Failed to connect to PostgreSQL:', err.message);
    process.exit(1);
  });

export default pool;
```

This file is **identical** to Level 2 Step 3's `db/index.ts`. If any line is unfamiliar — `Pool`, `connectionString`, the promise chain `.then(...).catch(...)`, or `process.exit(1)` — see Level 2 Step 3's "Reading This File Line-by-Line" walkthrough.

> ⚠️ **Type the error parameter**: `.catch((err: Error) => { ... })` — not `.catch((err) => { ... })`. Without the type, TypeScript errors with `Parameter 'err' implicitly has an 'any' type`.

### ✅ Checkpoint

```bash
psql vaultnote -c "\dt"
```

Should show `users` and `notes` tables.

---

## 3. Commit

```bash
git add .
git commit -m "feat: add users and notes schema with database connection"
```

---

> **Session Break** — Schema ready.
> When you return, you'll build authentication in [Step 4 — Authentication](../04-auth/).

---

| | |
|:---|---:|
| [← Step 2: Project Setup](../02-project-setup/) | [Step 4 — Authentication →](../04-auth/) |
