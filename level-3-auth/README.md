# Level 3 — VaultNote: Secure Personal Notes

## What You Will Build

VaultNote is a personal notes application where users register an account, log in, and manage their own private notes. Each user can only see, edit, and delete their own notes — no one else's. An admin user can view usage statistics across all users.

It has a React + TypeScript frontend with client-side routing, a Node.js + Express backend with JWT authentication, and a PostgreSQL database. All three are deployed to the internet with proper secret management.

## Why This App

Authentication is the dividing line between "learning project" and "real application." Every production app has users. Every production app has protected data. This level teaches you how identity works on the web:

1. How a server knows **who** is making a request
2. How passwords are stored safely (they're never stored in plain text)
3. How tokens travel between frontend and backend
4. Where **trust boundaries** exist and why they matter
5. How to protect routes so users can only access their own data
6. How to deploy an app with secrets that must never leak

## What You Will Learn

By the end of Level 3, you will understand:

- How user registration and login work end-to-end
- How bcrypt hashes passwords (and why you never store plain text)
- How JWT tokens are created, signed, and verified
- How auth middleware protects routes
- How React Router provides client-side navigation
- How protected routes work on both frontend and backend
- How role-based access control scopes what users can do
- Where trust boundaries exist in a web application
- How to manage secrets (JWT_SECRET) in production

## Level 3 Architecture (The Big Picture)

```
┌────────────────────────────────────────────────────────────────────────┐
│                          YOUR APPLICATION                               │
│                                                                        │
│  ┌──────────────────┐   ┌──────────────────────┐   ┌──────────────┐  │
│  │    FRONTEND       │   │      BACKEND          │   │   DATABASE    │  │
│  │  (React + TS)     │   │  (Node + Express)     │   │ (PostgreSQL)  │  │
│  │                  │   │                      │   │              │  │
│  │ ┌──────────────┐ │   │ ┌──────────────────┐ │   │ ┌──────────┐ │  │
│  │ │ React Router │ │   │ │  Auth Middleware  │ │   │ │  users   │ │  │
│  │ │              │ │   │ │  (verifies JWT)   │ │   │ │  table   │ │  │
│  │ │ /login       │ │   │ ├──────────────────┤ │   │ ├──────────┤ │  │
│  │ │ /register    │ │   │ │  Auth Routes     │ │   │ │  notes   │ │  │
│  │ │ /notes       │ │   │ │  (register/login)│ │   │ │  table   │ │  │
│  │ │ /notes/:id   │ │   │ ├──────────────────┤ │   │ └──────────┘ │  │
│  │ └──────────────┘ │   │ │  Notes Routes    │ │   │              │  │
│  │                  │   │ │  (protected CRUD) │ │   │ Passwords   │  │
│  │ Token stored in  │   │ ├──────────────────┤ │   │ stored as   │  │
│  │ localStorage     │   │ │  Error Handler   │ │   │ bcrypt      │  │
│  │                  │   │ └──────────────────┘ │   │ hashes      │  │
│  │ Sent via         │   │                      │   │              │  │
│  │ Authorization    │   │ JWT_SECRET kept in   │   │              │  │
│  │ header           │   │ environment only     │   │              │  │
│  └──────────────────┘   └──────────────────────┘   └──────────────┘  │
│                                                                        │
│               ┌────────────────────────────────────┐                  │
│               │         TRUST BOUNDARY              │                  │
│               │                                    │                  │
│               │  ABOVE: Server + Database (trusted) │                  │
│               │  BELOW: Browser (untrusted)         │                  │
│               └────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────────────┘
```

## Authentication Flow (How Identity Works)

```
REGISTRATION:
─────────────
1. User fills out register form (email + password)
   │
2. Frontend sends POST /api/auth/register
   │  Body: { email: "user@example.com", password: "mypassword" }
   │
3. Backend receives the request
   │  → Validates email and password
   │  → Hashes password with bcrypt (never stores plain text)
   │  → Inserts user into database: { email, password_hash }
   │  → Creates JWT token containing { userId, role }
   │  → Sends back: { token: "eyJhbG...", user: { id, email, role } }
   │
4. Frontend stores the token in localStorage
   │  → Redirects to /notes

LOGIN:
──────
1. User fills out login form (email + password)
   │
2. Frontend sends POST /api/auth/login
   │  Body: { email: "user@example.com", password: "mypassword" }
   │
3. Backend receives the request
   │  → Finds user by email in database
   │  → Compares password with stored hash using bcrypt.compare()
   │  → If match: creates JWT token, sends it back
   │  → If no match: sends 401 Unauthorized
   │
4. Frontend stores the token in localStorage
   │  → Redirects to /notes

AUTHENTICATED REQUESTS:
───────────────────────
1. Frontend sends request with token:
   │  GET /api/notes
   │  Authorization: Bearer eyJhbG...
   │
2. Auth middleware intercepts the request
   │  → Extracts token from Authorization header
   │  → Verifies token signature using JWT_SECRET
   │  → Decodes payload: { userId: 1, role: "user" }
   │  → Attaches user info to request: req.user = { id: 1, role: "user" }
   │  → Passes request to route handler
   │
3. Route handler runs
   │  → Uses req.user.id to scope the query:
   │     SELECT * FROM notes WHERE user_id = req.user.id
   │  → Returns only THIS user's notes
```

## Folder Structure (What Goes Where and Why)

```
vault-note/
├── client/                      ← FRONTEND
│   ├── src/
│   │   ├── components/          ← React components
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   ├── NoteForm.tsx
│   │   │   ├── NoteList.tsx
│   │   │   ├── NoteEditor.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   ├── pages/               ← Route-level page components (NEW)
│   │   │   ├── LoginPage.tsx
│   │   │   ├── RegisterPage.tsx
│   │   │   └── NotesPage.tsx
│   │   ├── context/             ← React Context for auth state (NEW)
│   │   │   └── AuthContext.tsx
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── services/
│   │   │   └── api.ts
│   │   ├── App.tsx
│   │   ├── App.css
│   │   └── main.tsx
│   ├── index.html
│   ├── package.json
│   └── vite.config.ts
│
├── server/                      ← BACKEND
│   ├── src/
│   │   ├── db/
│   │   │   ├── index.ts
│   │   │   └── schema.sql
│   │   ├── routes/
│   │   │   ├── auth.ts          ← Registration + login (NEW)
│   │   │   └── notes.ts         ← Protected CRUD for notes
│   │   ├── middleware/
│   │   │   ├── authenticate.ts  ← JWT verification middleware (NEW)
│   │   │   └── errorHandler.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── .env
│   ├── package.json
│   └── tsconfig.json
│
├── .gitignore
└── README.md
```

**What changed from Level 2?**
- `server/src/routes/auth.ts` — new route file for registration and login
- `server/src/middleware/authenticate.ts` — new middleware that verifies JWT tokens
- `client/src/pages/` — page-level components for React Router
- `client/src/context/` — React Context for managing auth state globally
- `client/src/components/ProtectedRoute.tsx` — guards routes that require login

## Learning Path

Work through these sections in order:

1. **[01 — Orientation](./01-orientation/)** — Prerequisites gate, security mental models, trust boundaries
2. **[02 — Project Setup](./02-project-setup/)** — Scaffold the project with new auth dependencies
3. **[03 — Database Design](./03-database/)** — Users table, notes table, password hashing concepts
4. **[04 — Authentication](./04-auth/)** — Registration, login, bcrypt, JWT, auth middleware
5. **[05 — Building the Backend](./05-backend/)** — Protected CRUD routes for notes
6. **[06 — Building the Frontend](./06-frontend/)** — React Router, auth pages, notes UI
7. **[07 — Deployment](./07-deployment/)** — Deploy with JWT_SECRET and security considerations
8. **[08 — Growth Review](./08-growth/)** — What you learned and how to talk about it

---

| | |
|:---|---:|
| [← Level 2 — TaskForge](../level-2-crud/) | [01 — Start Here →](./01-orientation/) |
