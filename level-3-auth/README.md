# Level 3 — VaultNote (Secure Personal Notes)

## What You Will Build

VaultNote is a personal notes application where users register, log in, and manage private notes. Each user can only see their own notes. An admin role can view usage statistics. Authentication uses JWT tokens and bcrypt password hashing.

### Why This App

Authentication is the dividing line between "toy project" and "real application." This level teaches you how identity works on the web — how a server knows who is making a request, how tokens travel between client and server, where trust boundaries exist, and what "security" actually means in practice.

### What Changed From Level 2

| Level 2 (TaskForge) | Level 3 (VaultNote) |
|-----|------|
| Anyone can access any data | **Users only see their own data** |
| No user accounts | **Registration + Login** |
| Passwords not needed | **bcrypt password hashing** |
| No identity verification | **JWT token authentication** |
| All routes public | **Protected routes (frontend + backend)** |
| Single role | **User + Admin roles** |

---

## Architecture

```
┌──────────────┐      ┌──────────────────┐      ┌──────────────┐
│   FRONTEND   │      │     BACKEND      │      │   DATABASE   │
│              │      │                  │      │              │
│  Login Form  │─────▶│  Auth Middleware │─────▶│  Users table │
│  Token Store │◀─────│  Route Guards    │◀─────│  Notes table │
│  Protected   │      │  JWT Creation    │      │  Hashed PWs  │
│  Routes      │      │  JWT Validation  │      │              │
└──────────────┘      └──────────────────┘      └──────────────┘
        │                      │
   ┌────▼──────────────────────▼──────────────────────────┐
   │                  TRUST BOUNDARY                      │
   │  Above: Server-side (WE control)                     │
   │  Below: Client-side (USER controls)                  │
   │  Never trust data from below the boundary.           │
   └──────────────────────────────────────────────────────┘
```

---

## Folder Structure

```
vault-note/
├── client/
│   ├── src/
│   │   ├── components/
│   │   │   └── ProtectedRoute.tsx
│   │   ├── context/
│   │   │   └── AuthContext.tsx
│   │   ├── pages/
│   │   │   ├── LoginPage.tsx
│   │   │   ├── RegisterPage.tsx
│   │   │   └── NotesPage.tsx
│   │   ├── services/
│   │   │   └── api.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── App.tsx
│   │   └── App.css
│   └── package.json
│
├── server/
│   ├── src/
│   │   ├── db/
│   │   │   ├── schema.sql
│   │   │   └── index.ts
│   │   ├── middleware/
│   │   │   ├── authenticate.ts    ← JWT verification
│   │   │   └── errorHandler.ts
│   │   ├── routes/
│   │   │   ├── auth.ts            ← Register + Login
│   │   │   ├── notes.ts           ← Protected CRUD
│   │   │   └── admin.ts           ← Admin-only routes
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── package.json
│   └── tsconfig.json
│
├── .gitignore
└── README.md
```

---

## Session Guide

| Session | Lesson | What You Do |
|---------|--------|-------------|
| 1 | [01 — Orientation](./01-orientation/) | Understand auth, security, trust boundaries |
| 2 | [02 — Project Setup](./02-project-setup/) | Scaffold with auth dependencies |
| 3 | [03 — Database](./03-database/) | Design users + notes schema |
| 4-5 | [04 — Authentication](./04-auth/) | Build registration, login, JWT, middleware |
| 5-6 | [05 — Backend](./05-backend/) | Protected CRUD routes for notes |
| 7-8 | [06 — Frontend](./06-frontend/) | Auth pages, routing, notes UI |
| 8-9 | [07 — Deployment](./07-deployment/) | Deploy with secret management |
| 9 | [08 — Growth](./08-growth/) | Review security skills |

**Estimated total: 9–12 sessions (30–60 min each)**

---

| | |
|:---|---:|
| [← Level 2: TaskForge](../level-2-crud/) | [Step 1 — Orientation →](./01-orientation/) |
