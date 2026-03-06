# Level 5 — CollabBoard: Collaborative Project Hub

> Your capstone project. Production-grade patterns, complex data, drag-and-drop, CI/CD, and error monitoring.

## What You'll Build

**CollabBoard** is a collaborative project management platform — think Trello. Teams create boards, organize cards into lists, drag cards between lists, assign members, and add comments. The backend uses modular architecture. The frontend uses optimistic updates so the UI feels instant. A CI/CD pipeline runs tests on every push. Error monitoring catches crashes in production.

This is the project that makes your portfolio **complete**.

---

## Architecture

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
│  │ - Test  │    │  │ Updates      │◀──────│  │   - auth     │ │  │    │
│  │ - Build │    │  │ React Router │       │  │   - boards   │ │  │    │
│  │         │    │  │ Redux        │       │  │   - lists    │ │  │    │
│  └─────────┘    │  └──────────────┘       │  │   - cards    │ │  │    │
│                 │                          │  │   - comments │ │  │    │
│  ┌─────────┐   │                          │  └──────┬───────┘ │  │    │
│  │ Error   │   │                          │         │         │  │    │
│  │Monitoring│◀──│──────────────────────────│  ┌──────▼───────┐ │  │    │
│  │ (Sentry)│   │                          │  │  PostgreSQL   │ │  │    │
│  │         │   │                          │  │  6 tables     │ │  │    │
│  └─────────┘   │                          │  │  many-to-many │ │  │    │
│                 │                          │  └──────────────┘ │  │    │
│                 │                          └────────────────────┘  │    │
│                 └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Optimistic Update Flow

```
USER ACTION: Drag card from "To Do" to "In Progress"
  │
  ├── 1. IMMEDIATELY update Redux store (card moves in UI — instant)
  │
  ├── 2. SEND API request to backend (PUT /api/cards/:id/move)
  │
  ├── 3a. API SUCCESS → do nothing (UI is already correct)
  │
  └── 3b. API FAILURE → ROLLBACK Redux store (card snaps back)
                         Show error notification
```

---

## Folder Structure

```
collab-board/
├── client/
│   ├── src/
│   │   ├── app/
│   │   │   ├── store.ts               ← Redux store
│   │   │   └── hooks.ts              ← Typed hooks
│   │   ├── features/
│   │   │   ├── auth/                  ← Login, register, auth slice
│   │   │   ├── boards/               ← Board list, create board
│   │   │   ├── board/                ← Board view, lists, cards, DnD
│   │   │   └── ui/                   ← Layout, modals, error boundary
│   │   ├── services/
│   │   │   └── api.ts                ← All API calls
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── utils/
│   │       └── formatters.ts
│   ├── package.json
│   ├── vite.config.ts
│   └── vitest.config.ts
├── server/
│   ├── src/
│   │   ├── modules/                   ← Feature modules          ← NEW
│   │   │   ├── auth/
│   │   │   │   ├── auth.routes.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   └── auth.middleware.ts
│   │   │   ├── boards/
│   │   │   │   ├── boards.routes.ts
│   │   │   │   └── boards.service.ts
│   │   │   ├── lists/
│   │   │   │   ├── lists.routes.ts
│   │   │   │   └── lists.service.ts
│   │   │   ├── cards/
│   │   │   │   ├── cards.routes.ts
│   │   │   │   └── cards.service.ts
│   │   │   └── comments/
│   │   │       ├── comments.routes.ts
│   │   │       └── comments.service.ts
│   │   ├── middleware/
│   │   │   ├── logger.ts
│   │   │   ├── rateLimiter.ts
│   │   │   └── errorHandler.ts
│   │   ├── db/
│   │   │   ├── index.ts
│   │   │   ├── schema.sql
│   │   │   └── seed.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── tests/
│   │   ├── boards.test.ts
│   │   └── cards.test.ts
│   └── package.json
├── .github/
│   └── workflows/
│       └── ci.yml                     ← GitHub Actions             ← NEW
├── .env
├── .gitignore
└── README.md
```

---

## Session Guide

Level 5 takes approximately **14–18 sessions** (30–60 minutes each).

| Lesson | Sessions | Notes |
|--------|----------|-------|
| 01 — Orientation | 1 | Read and absorb in one sitting |
| 02 — Project Setup | 2 | Break after backend scaffold, before frontend deps |
| 03 — Database | 2 | Break after schema, before seed script |
| 04 — Backend | 4 | Breaks after auth module, after boards module, and after cards module |
| 05 — Frontend | 5 | Breaks after auth slice, after board slice, after board view components, and after ProtectedRoute |
| 06 — Drag & Drop | 3 | Breaks after SortableCard setup and after optimistic update pattern |
| 07 — CI/CD & Monitoring | 2 | Break after CI pipeline, before Sentry |
| 08 — Deployment | 1 | Complete in one sitting |
| 09 — Growth Review | 1 | Complete in one sitting |

Look for **Session Break** markers inside each lesson — they tell you exactly when to pause.

## Learning Path

| Step | Topic | What You Build |
|------|-------|----------------|
| [01 — Orientation](./01-orientation/) | Prerequisites + architecture | Mental model for production apps |
| [02 — Project Setup](./02-project-setup/) | Scaffold + modular structure | Feature modules, auth, dnd-kit, Sentry |
| [03 — Database](./03-database/) | Complex schema + JOINs | 6 tables with many-to-many relationships |
| [04 — Backend](./04-backend/) | Feature modules | Auth, boards, lists, cards, comments modules |
| [05 — Frontend](./05-frontend/) | Board UI + card details | React Router, Redux, board view, modals |
| [06 — Drag & Drop](./06-drag-and-drop/) | dnd-kit + optimistic updates | Card dragging with instant UI feedback |
| [07 — CI/CD & Monitoring](./07-cicd-monitoring/) | GitHub Actions + Sentry | Automated pipeline + error tracking |
| [08 — Deployment](./08-deployment/) | Ship the capstone | Full production deployment |
| [09 — Growth Review](./09-growth/) | Portfolio + career readiness | How to present your complete portfolio |

---

## New Concepts in This Level

| Concept | Why Now |
|---------|---------|
| **Modular backend** | Feature modules scale better than flat route files |
| **Many-to-many relationships** | Users belong to multiple boards; boards have multiple members |
| **Complex JOINs** | Board views require data from 4+ tables in one query |
| **Drag-and-drop (dnd-kit)** | Cards move between lists — core Kanban interaction |
| **Optimistic updates** | UI updates instantly, rolls back on failure |
| **CI/CD (GitHub Actions)** | Tests run automatically on every push |
| **Error monitoring (Sentry)** | Catch and track production errors |
| **Position management** | Ordered lists and cards require position tracking |

---

## Prerequisites

- Completed [Level 1 — DevPulse](../level-1-foundations/)
- Completed [Level 2 — TaskForge](../level-2-crud/)
- Completed [Level 3 — VaultNote](../level-3-auth/)
- Completed [Level 4 — DataDash](../level-4-scalable/)

---

| | |
|:---|---:|
| [← Level 4 — DataDash](../level-4-scalable/) | [Curriculum Overview →](../00-curriculum-overview/) |
