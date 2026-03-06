# Curriculum Overview — All 5 Levels

## How This Program Is Structured

Each level follows the same pattern:

```
1. SPATIAL ORIENTATION  → Where are we? What are we building? What does the architecture look like?
2. CONCEPT INTRODUCTION → New ideas explained in technical terms AND plain English
3. GUIDED BUILD         → Step-by-step construction of the application
4. SPATIAL CHECK-IN     → Zoom out. Where does this code live? How does it connect?
5. DEPLOYMENT           → Ship it. Make it real. Make it public.
6. GROWTH REVIEW        → What did we learn? How do we talk about it?
```

## How to Pace Yourself

Research on skill acquisition shows that **30–60 minutes of focused, deliberate practice** outperforms marathon sessions. After about an hour of deep coding, your retention drops and mistakes increase. Spreading your learning across shorter sessions — with sleep and rest in between — produces significantly better long-term retention.

Throughout this curriculum, you'll see **Session Break** markers at natural stopping points:

> [!TIP]
> **Session Break** — You've completed [section]. Save your work and take a break.
> When you return, you'll [next section].

**When you hit a Session Break:**
1. **Commit your code** — `git add . && git commit -m "progress: [what you finished]"`
2. **Step away** — take a real break (15+ minutes minimum, ideally a few hours or overnight)
3. **Return fresh** — when you come back, re-read the last section you completed to rebuild context

**You do not have to stop at every break** — if you're in flow and energized, keep going. But if you feel your focus slipping, the breaks are there for a reason.

Across all 5 levels, expect roughly **40–55 sessions** total. Some lessons fit in a single session; longer lessons have multiple break points. Each level's README includes a Session Guide showing what to expect.

---

## LEVEL 1 — DevPulse (Developer Mood Tracker)

### What You Build
A single-page application where a developer logs their daily mood, energy level, and a short note. The frontend displays a history of entries. The backend serves and receives data through a REST API. Data lives in memory (no database yet).

### Why This App
It is simple enough to focus on fundamentals but real enough to deploy and show someone. It forces you to understand the client-server split immediately.

### Architecture Snapshot

```
┌─────────────────────────────────────────────────────────────┐
│                        THE INTERNET                         │
│                                                             │
│   ┌───────────────────┐         ┌───────────────────────┐   │
│   │   FRONTEND (React)│  HTTP   │   BACKEND (Express)   │   │
│   │                   │────────▶│                       │   │
│   │  - Components     │◀────────│  - REST API           │   │
│   │  - State          │         │  - Routes             │   │
│   │  - TypeScript     │         │  - In-memory data     │   │
│   │                   │         │                       │   │
│   │  Runs in BROWSER  │         │  Runs on SERVER       │   │
│   └───────────────────┘         └───────────────────────┘   │
│                                                             │
│   Deployed to:                  Deployed to:                │
│   Vercel / Netlify              Render / Railway            │
└─────────────────────────────────────────────────────────────┘
```

### Core Concepts Introduced
- React components and JSX
- TypeScript types and interfaces
- useState and useEffect
- HTTP requests (fetch / axios)
- Express routing
- REST API design basics
- JSON data format
- Git workflow (init, add, commit, push)
- Frontend deployment
- CORS (what it is and why it blocks you)

### Skills Demonstrated to Employers
- Can build and deploy a React + TypeScript frontend
- Understands client-server architecture
- Can create a basic REST API
- Uses Git professionally
- Understands deployment basics

---

## LEVEL 2 — TaskForge (Project Task Manager)

### What You Build
A project management tool where users can create projects, add tasks to projects, mark tasks complete, edit and delete tasks. Data persists in a PostgreSQL database. Both frontend and backend are deployed to production.

### Why This App
CRUD operations are the backbone of almost every web application. This level forces you to understand how data moves from a form, through an API, into a database, and back to the screen. It introduces the concept of persistent state that survives server restarts.

### Architecture Snapshot

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│   │   FRONTEND   │      │   BACKEND    │      │   DATABASE   │  │
│   │   (React)    │─────▶│  (Express)   │─────▶│ (PostgreSQL) │  │
│   │              │◀─────│              │◀─────│              │  │
│   │  Forms       │ HTTP │  Validation  │ SQL  │  Tables      │  │
│   │  Lists       │      │  Routes      │      │  Rows        │  │
│   │  State mgmt  │      │  Controllers │      │  Relations   │  │
│   └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                  │
│   BROWSER                SERVER                 SERVER           │
│   (user's machine)       (cloud)                (cloud)          │
└──────────────────────────────────────────────────────────────────┘
```

### Core Concepts Introduced
- PostgreSQL fundamentals
- Database schema design
- SQL queries (SELECT, INSERT, UPDATE, DELETE)
- Environment variables (.env files)
- Database connection pooling
- API design principles (RESTful conventions, status codes)
- Error handling (try/catch, error middleware)
- Backend deployment with database
- Database migrations concept
- CORS configuration in production

### Why PostgreSQL (Not MongoDB, SQLite, etc.)
- **PostgreSQL over MongoDB**: Relational data (projects → tasks) maps naturally to SQL. Most production systems use relational databases. Learning SQL transfers to MySQL, SQLite, and others. MongoDB is great for document-oriented data, but starting with SQL teaches you to think about data relationships — a critical skill.
- **PostgreSQL over SQLite**: PostgreSQL runs as a separate process, which mirrors production architecture. SQLite embeds in the application, hiding the database layer. We want you to see and understand that layer.
- **PostgreSQL over MySQL**: Comparable choice, but PostgreSQL has better TypeScript ecosystem support, better JSON handling, and is the default on most modern cloud platforms.

### Skills Demonstrated to Employers
- Database design and SQL
- Full CRUD implementation
- API design with proper status codes and error handling
- Environment configuration management
- Full-stack deployment (frontend + backend + database)

---

## LEVEL 3 — VaultNote (Secure Encrypted Notes)

### What You Build
A note-taking application where users register, log in, and manage private notes. Notes belong to users. Users cannot see other users' notes. An admin role can view usage statistics. Authentication uses JWT tokens.

### Why This App
Authentication is the dividing line between "toy project" and "real application." This level teaches you how identity works on the web — how a server knows who is making a request, how tokens travel, where trust boundaries exist, and what "security" actually means in practice.

### Architecture Snapshot

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   ┌──────────────┐      ┌──────────────────┐      ┌──────────────┐  │
│   │   FRONTEND   │      │     BACKEND      │      │   DATABASE   │  │
│   │              │      │                  │      │              │  │
│   │  Login Form  │─────▶│  Auth Middleware │─────▶│  Users table │  │
│   │  Token Store │◀─────│  Route Guards    │◀─────│  Notes table │  │
│   │  Protected   │      │  JWT Creation    │      │  Hashed PWs  │  │
│   │  Routes      │      │  JWT Validation  │      │              │  │
│   └──────────────┘      └──────────────────┘      └──────────────┘  │
│         │                        │                                   │
│         │    ┌───────────────────┘                                   │
│         │    │                                                       │
│   ┌─────▼────▼─────────────────────────────────────────────┐        │
│   │              TRUST BOUNDARY                            │        │
│   │                                                        │        │
│   │  Everything ABOVE this line: we control                │        │
│   │  Everything BELOW this line: the user controls         │        │
│   │                                                        │        │
│   │  The browser is UNTRUSTED territory.                   │        │
│   │  Never trust data from the frontend without            │        │
│   │  validating it on the backend.                         │        │
│   └────────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

### Core Concepts Introduced
- User registration and login flows
- Password hashing (bcrypt)
- JWT creation, signing, and verification
- Auth middleware pattern
- Protected routes (frontend React Router guards)
- Protected routes (backend middleware)
- Role-based access control (user vs admin)
- Trust boundaries explained
- Security best practices (OWASP basics)
- HTTP-only cookies vs localStorage for tokens
- Token refresh concepts

### Why JWT (Not Sessions, Not OAuth)
- **JWT over sessions**: JWTs are stateless — the server doesn't need to track active sessions. This teaches you how tokens encode information and how the backend validates without database lookups. Sessions are excellent (and simpler in some ways), but JWTs force you to understand cryptographic signing and token lifecycle.
- **JWT before OAuth**: OAuth is a delegation protocol (letting Google verify identity for you). You need to understand manual auth first to appreciate what OAuth automates. We build JWT from scratch here, then you can adopt OAuth/Auth0/Clerk later knowing exactly what they abstract away.

### Skills Demonstrated to Employers
- Authentication system design
- Security awareness (hashing, trust boundaries, input validation)
- JWT token management
- Protected frontend and backend routes
- Role-based access control
- Understanding of common attack vectors

---

## LEVEL 4 — DataDash (Analytics Dashboard)

### What You Build
A dashboard application that displays analytics data — charts, tables, filters, and real-time-ish updates. Multiple data sources feed into a unified view. The frontend handles complex state across many components. The backend uses middleware for logging, rate limiting, and request parsing. Full test suite included.

### Why This App
Dashboards force you to manage complex, interconnected state. Multiple components need the same data. Filters in one component affect data in another. This is where simple useState breaks down and you need architectural patterns. This level also introduces testing — the practice that separates junior developers from mid-level engineers.

### Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│   ┌────────────────────────────────────┐      ┌───────────────────┐   │
│   │           FRONTEND                 │      │     BACKEND       │   │
│   │                                    │      │                   │   │
│   │  ┌──────────┐  ┌──────────┐       │      │  ┌─────────────┐ │   │
│   │  │ Chart    │  │ Table    │       │      │  │ Middleware  │ │   │
│   │  │Component │  │Component │       │      │  │ Stack:      │ │   │
│   │  └────┬─────┘  └────┬─────┘       │      │  │  - Logger   │ │   │
│   │       │              │             │      │  │  - Auth     │ │   │
│   │  ┌────▼──────────────▼─────┐      │      │  │  - RateLimit│ │   │
│   │  │    REDUX STORE          │      │ HTTP │  │  - CORS     │ │   │
│   │  │    (Single source of    │──────┼─────▶│  │  - Validate │ │   │
│   │  │     truth)              │◀─────┼──────│  └──────┬──────┘ │   │
│   │  │                         │      │      │         │        │   │
│   │  │  - analytics slice      │      │      │  ┌──────▼──────┐ │   │
│   │  │  - filters slice        │      │      │  │ Controllers │ │   │
│   │  │  - ui slice             │      │      │  │ + Services  │ │   │
│   │  └─────────────────────────┘      │      │  └──────┬──────┘ │   │
│   └────────────────────────────────────┘      │         │        │   │
│                                                │  ┌──────▼──────┐ │   │
│   ┌────────────────────────────────────┐      │  │  Database   │ │   │
│   │          TEST LAYER                │      │  └─────────────┘ │   │
│   │                                    │      └───────────────────┘   │
│   │  - Unit tests (Vitest)             │                              │
│   │  - Component tests (RTL)           │                              │
│   │  - Integration tests (Supertest)   │                              │
│   └────────────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────────────┘
```

### Core Concepts Introduced
- Redux Toolkit (slices, thunks, selectors)
- Folder architecture at scale (feature-based organization)
- API abstraction layers (service pattern)
- Express middleware deep dive
- Logging (structured logs with pino/winston)
- Rate limiting
- Unit testing with Vitest
- Component testing with React Testing Library
- API testing with Supertest
- Performance considerations (React.memo, useMemo, useCallback)
- Production debugging techniques (source maps, error boundaries)

### Why Redux Toolkit (Not Zustand, Not Context)
- **Redux Toolkit over React Context**: Context re-renders every consumer when any value changes. For a dashboard with many interconnected components, this causes performance problems. Redux provides surgical re-renders through selectors.
- **Redux Toolkit over plain Redux**: Plain Redux requires excessive boilerplate. Redux Toolkit is the official, modern way to use Redux — it eliminates boilerplate while keeping the architecture.
- **Redux Toolkit over Zustand**: Zustand is simpler and excellent for many apps. But Redux Toolkit has a larger ecosystem (DevTools, middleware, RTK Query), better documentation, and is more commonly seen in enterprise codebases. Learning Redux transfers directly to most large-company codebases.

### Why Vitest (Not Jest)
- **Vitest over Jest**: Vitest uses the same Vite build pipeline as your app, so configuration is minimal. It's faster, has native TypeScript support, and has a Jest-compatible API. If you learn Vitest, you can use Jest — the APIs are nearly identical.

### Skills Demonstrated to Employers
- Complex state management
- Scalable project architecture
- Middleware design patterns
- Comprehensive testing strategy
- Performance optimization awareness
- Production debugging skills

---

## LEVEL 5 — CollabBoard (Collaborative Project Hub)

### What You Build
A collaborative project management platform where teams can create boards, add cards, assign members, comment on cards, and track progress. Features include drag-and-drop, optimistic UI updates, real-time-ish sync, and a modular backend with clean separation of concerns. Includes a CI/CD pipeline overview and error monitoring setup.

### Why This App
This is your portfolio capstone. It demonstrates that you can build a complex, multi-feature application with production-grade patterns. It introduces the final concepts that separate a junior from a mid-level engineer: optimistic updates, complex data relationships, modular backend architecture, and operational awareness (monitoring, CI/CD).

### Architecture Snapshot

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION SYSTEM                               │
│                                                                          │
│  ┌─────────┐    ┌──────────────────────────────────────────────────┐    │
│  │  CI/CD  │    │              APPLICATION                        │    │
│  │ Pipeline│    │                                                  │    │
│  │         │    │  ┌──────────────┐       ┌────────────────────┐  │    │
│  │ GitHub  │───▶│  │   FRONTEND   │       │     BACKEND        │  │    │
│  │ Actions │    │  │              │       │                    │  │    │
│  │         │    │  │ Drag & Drop  │ HTTP  │  ┌──────────────┐ │  │    │
│  │ - Lint  │    │  │ Optimistic   │──────▶│  │   Modules:   │ │  │    │
│  │ - Test  │    │  │ Updates      │◀──────│  │   - boards   │ │  │    │
│  │ - Build │    │  │ Complex      │       │  │   - cards    │ │  │    │
│  │ - Deploy│    │  │ State        │       │  │   - users    │ │  │    │
│  └─────────┘    │  └──────────────┘       │  │   - comments │ │  │    │
│                 │                          │  └──────┬───────┘ │  │    │
│  ┌─────────┐   │                          │         │         │  │    │
│  │ Error   │   │                          │  ┌──────▼───────┐ │  │    │
│  │Monitoring│◀──│──────────────────────────│  │  Database    │ │  │    │
│  │ (Sentry)│   │                          │  │  - boards    │ │  │    │
│  │         │   │                          │  │  - cards     │ │  │    │
│  └─────────┘   │                          │  │  - users     │ │  │    │
│                 │                          │  │  - comments  │ │  │    │
│                 │                          │  │  - members   │ │  │    │
│                 │                          │  └──────────────┘ │  │    │
│                 │                          └────────────────────┘  │    │
│                 └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Core Concepts Introduced
- Drag-and-drop UI (dnd-kit)
- Optimistic UI updates pattern
- Complex data relationships (many-to-many)
- Modular backend architecture (feature modules)
- Database joins and complex queries
- CI/CD pipeline with GitHub Actions
- Error monitoring with Sentry
- Performance profiling
- Accessibility considerations
- API documentation concepts

### Skills Demonstrated to Employers
- Complex, production-ready application design
- Advanced React patterns (optimistic updates, complex state)
- Modular backend architecture
- Understanding of CI/CD and deployment pipelines
- Operational awareness (monitoring, error tracking)
- Ability to manage complex data relationships
- Portfolio-quality project

---

## Progression Summary

```
Level 1: "I can build and deploy a web app"
Level 2: "I can design a database and build full CRUD"
Level 3: "I can implement secure authentication"
Level 4: "I can architect scalable systems and write tests"
Level 5: "I can build production-grade applications with operational awareness"
```

Each level doesn't just add features — it adds **layers of understanding**. By the end, you don't just know how to build an app. You know how to **see the entire system** — every layer, every boundary, every data flow — and make informed decisions about how to build it.

---

| | |
|:---|---:|
| [← Home](../) | [Setup Guide →](../00-setup-guide/) |
