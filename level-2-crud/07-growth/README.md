# Step 7 — Growth Review: What You Learned and How to Talk About It

## What You Just Built

TaskForge is a full-stack CRUD application with:
- **PostgreSQL database** with relational schema (projects → tasks)
- **10 REST API endpoints** with validation and error handling
- **6 React components** for full CRUD operations
- **Three-tier deployment** (Vercel + Render/Railway + Supabase/Neon)
- **Parameterized SQL queries** (preventing SQL injection)
- **Cascade deletes** (projects → tasks)
- **Connection pooling** for efficient database access

---

## Skills Demonstrated to Employers

| Skill | What You Did | Interview Talking Point |
|-------|-------------|----------------------|
| **Database Design** | Foreign key with `ON DELETE CASCADE` | "I designed a one-to-many schema where projects own tasks, with cascade deletes to maintain referential integrity" |
| **SQL Proficiency** | Parameterized queries (`$1, $2`) | "I use parameterized queries to prevent SQL injection — I never interpolate user input into SQL strings" |
| **Full CRUD API** | 10 endpoints with RESTful conventions | "Each resource has standard CRUD endpoints following REST conventions with proper HTTP status codes" |
| **Error Handling** | Centralized 4-parameter error middleware | "I centralized error handling in one middleware instead of duplicating try/catch responses in every route" |
| **Environment Variables** | `.env` gitignored to prevent secrets in repo | "I use environment variables for all configuration, with different values per environment" |
| **Connection Pooling** | `pg` Pool for efficient DB access | "I use a connection pool to reuse database connections instead of opening a new one per request" |
| **Three-Tier Deployment** | Frontend, backend, and database on separate services | "I deployed each tier independently — CDN for frontend, Node.js for backend, managed PostgreSQL for data" |

---

## Architectural Understanding

You should be able to draw this from memory:

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Browser    │ HTTP │   Server     │ SQL  │  PostgreSQL  │
│              │─────▶│              │─────▶│              │
│   React      │      │   Express    │      │   projects   │
│   Components │◀─────│   Routes     │◀─────│   tasks      │
│              │ JSON │   Validation │ Rows │              │
│   Vercel     │      │   Render     │      │   Supabase   │
└──────────────┘      └──────────────┘      └──────────────┘
  Presentation          Application            Data
  Layer                 Layer                  Layer
```

### Complete Request Lifecycle

When a user creates a task:

```
1. User types task title and clicks "Add"
2. TaskForm calls onSubmit(title)
3. App.handleCreateTask calls api.createTask(projectId, { title })
4. fetch() sends POST /api/projects/1/tasks with JSON body
5. Express receives the request
6. express.json() parses the body
7. CORS middleware checks origin
8. Task route handler validates title
9. pool.query() sends: INSERT INTO tasks (project_id, title) VALUES ($1, $2) RETURNING *
10. PostgreSQL inserts the row, generates id and timestamps
11. PostgreSQL returns the new row
12. Handler sends 201 with the task as JSON
13. fetch() resolves the Promise
14. api.ts parses JSON → Task object
15. handleCreateTask updates selectedProject state
16. React re-renders TaskList with the new task
```

---

## What's Missing (and Why)

| Missing Feature | Why | When You'll Learn It |
|----------------|-----|---------------------|
| User authentication | Adding auth changes the entire data model | Level 3 |
| Protected routes | Need auth first | Level 3 |
| Password hashing | Security concept that requires auth context | Level 3 |
| Automated tests | Focused on building first | Level 4 |
| State management library | useState is sufficient at this scale | Level 4 |
| Input sanitization | Important but secondary to core CRUD | Level 3-4 |
| Pagination | Dataset is small enough without it | Level 4 |
| React Router | Single-page with view switching is simpler for now | Level 3 |

---

## How to Present This Project

### Resume

```
TaskForge — Project Task Manager
- Designed relational PostgreSQL schema with foreign keys and cascade deletes
- Built 10 RESTful API endpoints with parameterized SQL queries (SQL injection prevention)
- Implemented centralized error handling with Express middleware
- Deployed three-tier architecture: Vercel CDN + Render backend + Supabase PostgreSQL
```

### Interview Questions

**"How do you prevent SQL injection?"**
> "I use parameterized queries with `$1, $2` placeholders. The database driver treats these as data values, never as SQL code. I never interpolate user input into SQL strings."

**"Explain the relationship between your tables."**
> "Projects and tasks have a one-to-many relationship: one project has many tasks. The `tasks` table has a `project_id` foreign key referencing `projects(id)`. I use `ON DELETE CASCADE` so deleting a project automatically removes all its tasks."

**"Why not store data in the browser?"**
> "Browser storage is local to one device and one browser. A database is the single source of truth — data persists across restarts, is accessible from any device, and can be queried efficiently with SQL."

---

## Completion Checklist

Before starting Level 3, verify:

- [ ] TaskForge is deployed and working at your Vercel URL
- [ ] Data persists after refreshing the page
- [ ] All CRUD operations work (create, read, update, delete for both projects and tasks)
- [ ] Deleting a project deletes its tasks
- [ ] Code is pushed to GitHub
- [ ] `server/package.json` has @types packages in `dependencies`
- [ ] `server/tsconfig.json` has `"types": ["node"]`
- [ ] No CORS errors in the browser console
- [ ] You can explain three-tier architecture
- [ ] You can explain why parameterized queries prevent SQL injection

All checked? You're ready for [Level 3 — VaultNote](../../level-3-auth/).

---

| | |
|:---|---:|
| [← Step 6: Deployment](../06-deployment/) | [Level 3 — VaultNote →](../../level-3-auth/) |
