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
