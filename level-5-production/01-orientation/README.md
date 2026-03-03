`Level 5` **Step 1 of 9** — Orientation

# 01 — Orientation: Building for Production

## Prerequisites Gate

> [!WARNING]
> **You must have ALL FOUR previous levels deployed before starting Level 5.**
>
> Verify you have these working URLs:
>
> | Check | What to verify |
> |-------|---------------|
> | **Level 1 — DevPulse** | GitHub repo + Vercel frontend + Render backend |
> | **Level 2 — TaskForge** | GitHub repo + Vercel frontend + Render backend + database |
> | **Level 3 — VaultNote** | GitHub repo + Vercel frontend + Render backend + auth works |
> | **Level 4 — DataDash** | GitHub repo + Vercel frontend + Render backend + tests pass |
>
> **Skills check** — you should be comfortable with:
> - [ ] React components with TypeScript
> - [ ] Redux Toolkit (slices, thunks, selectors)
> - [ ] React Router (routes, navigation, protected routes)
> - [ ] Express routes with middleware
> - [ ] PostgreSQL with parameterized queries
> - [ ] JWT authentication and middleware
> - [ ] Testing with Vitest, React Testing Library, and Supertest
> - [ ] Feature-based folder organization
> - [ ] Performance optimization (React.memo, useMemo)
> - [ ] Git workflow + deployment (Vercel + Render)
>
> Level 5 uses **everything** from Levels 1-4. If anything above is incomplete, go back and finish it.

---

## What Makes This a Capstone

Levels 1-4 each focused on specific skills. Level 5 combines them all and adds the final layer: **production engineering**.

```
Level 1: "I can build a web app"
Level 2: "I can persist data"
Level 3: "I can secure an app"
Level 4: "I can test and optimize"
Level 5: "I can build something a team would ship"   ← You are here
```

What makes Level 5 different:

| Previous Levels | Level 5 |
|----------------|---------|
| Single user | Multi-user collaboration |
| Simple data (1-2 tables) | Complex data (6 tables, many-to-many) |
| Flat route files | Modular backend (feature modules) |
| Click-to-save | Drag-and-drop with optimistic updates |
| Manual deployment | CI/CD pipeline (automated tests + deploy) |
| `console.error` | Error monitoring (Sentry) |

---

## What CollabBoard Does

CollabBoard is a Kanban-style project board (like Trello). Users:

1. **Register and log in** (carried from Level 3)
2. **Create boards** ("Product Launch," "Sprint 23," etc.)
3. **Invite members** to boards
4. **Add lists** to boards ("To Do," "In Progress," "Done")
5. **Add cards** to lists (individual tasks)
6. **Drag cards** between lists (To Do → In Progress)
7. **Open card details** to add descriptions, assign members, and write comments
8. **See updates** from other board members

### Board Layout

```
┌──────────────────────────────────────────────────────────────┐
│  CollabBoard          Product Launch          [+ Add Member] │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   To Do      │  │ In Progress  │  │    Done      │       │
│  │              │  │              │  │              │       │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │       │
│  │ │Design    │ │  │ │Build API │ │  │ │Setup DB  │ │       │
│  │ │mockups   │ │  │ │endpoints │ │  │ │schema    │ │       │
│  │ │  👤 Alex │ │  │ │  👤 Sam  │ │  │ │  👤 Alex │ │       │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │       │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │              │       │
│  │ │Write     │ │  │ │Auth      │ │  │              │       │
│  │ │tests     │ │  │ │middleware│ │  │              │       │
│  │ │  💬 2    │ │  │ │  👤 Sam  │ │  │              │       │
│  │ └──────────┘ │  │ └──────────┘ │  │              │       │
│  │              │  │              │  │              │       │
│  │ [+ Add card] │  │ [+ Add card] │  │ [+ Add card] │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                               │
│  [+ Add List]                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Modular Backend Architecture

In Levels 2-4, all routes lived in flat files under `routes/`. This works for small apps, but as the API grows, it becomes hard to navigate. Level 5 organizes the backend by **feature module**:

```
FLAT (Levels 2-4):                   MODULAR (Level 5):

server/src/                           server/src/
├── routes/                           ├── modules/
│   ├── auth.ts                       │   ├── auth/
│   ├── boards.ts                     │   │   ├── auth.routes.ts
│   ├── lists.ts                      │   │   ├── auth.service.ts
│   ├── cards.ts                      │   │   └── auth.middleware.ts
│   └── comments.ts                   │   ├── boards/
├── services/                         │   │   ├── boards.routes.ts
│   ├── authService.ts                │   │   └── boards.service.ts
│   ├── boardsService.ts              │   ├── lists/
│   ├── cardsService.ts               │   │   ├── lists.routes.ts
│   └── commentsService.ts            │   │   └── lists.service.ts
└── middleware/                        │   ├── cards/
    └── authenticate.ts                │   │   ├── cards.routes.ts
                                       │   │   └── cards.service.ts
                                       │   └── comments/
                                       │       ├── comments.routes.ts
                                       │       └── comments.service.ts
                                       └── middleware/
                                           └── ...
```

> [!NOTE]
> **Technical:** Modular architecture co-locates a feature's routes and services. Each module is self-contained — you can understand the boards module by reading two files. Adding a new feature means creating a new folder, not touching existing code.
>
> **Plain English:** Instead of one giant filing cabinet with drawers labeled "routes," "services," "middleware," you have separate cabinets per department: "auth," "boards," "cards." Each department keeps its own files together.

---

## Optimistic Updates

In Levels 2-4, the pattern was: user acts → API call → wait → update UI.

Level 5 introduces: user acts → update UI instantly → API call in background → rollback if it fails.

```
PESSIMISTIC (Levels 2-4):

  User drags card
       │
       ▼
  Show loading spinner
       │
       ▼
  API call (200ms–2s)
       │
       ▼
  Update UI
       │
  Total wait: 200ms–2s    ← User feels lag


OPTIMISTIC (Level 5):

  User drags card
       │
       ├──▶ Update UI immediately (0ms)    ← User feels instant
       │
       └──▶ API call in background
               │
               ├── Success: do nothing
               │
               └── Failure: rollback UI + show error
```

> [!NOTE]
> **Technical:** Optimistic updates require saving the previous state before mutating. If the API call fails, the reducer reverts to the saved state. This works well for operations that rarely fail (moving a card, toggling a checkbox).
>
> **Plain English:** Optimistic updates are like an airline that assigns you a seat before confirming with the system. You sit down immediately. If there's a conflict, they move you — but 99% of the time, it just works and you saved time waiting.

---

## CI/CD Pipeline

Until now, you deployed manually: push code, Vercel/Render auto-deploys. Level 5 adds a **check** before deployment: GitHub Actions runs your tests, and deployment only proceeds if they pass.

```
MANUAL (Levels 1-4):

  Push code → Auto-deploy    ← Broken code can reach production


CI/CD (Level 5):

  Push code → GitHub Actions → Tests pass? → Auto-deploy
                                    │
                                    └── Tests fail? → BLOCKED
                                                      Fix and push again
```

> [!NOTE]
> **Technical:** CI (Continuous Integration) runs tests and builds on every push. CD (Continuous Deployment) automatically deploys if CI passes. GitHub Actions defines the pipeline in a YAML file (`.github/workflows/ci.yml`). The pipeline runs on GitHub's servers, not your local machine.
>
> **Plain English:** CI/CD is like a quality inspector at a factory. Every product (code push) goes through inspection (tests). Only products that pass inspection reach the store (production). This prevents defective products from reaching customers.

---

## What You Won't Build (and Why)

| Omitted Feature | Reason |
|-----------------|--------|
| **Real-time WebSockets** | Adds infrastructure complexity; polling on demand demonstrates the concept |
| **File uploads** | Requires object storage (S3) — a separate infrastructure concern |
| **Email notifications** | Requires email service (SendGrid, etc.) — beyond frontend/backend scope |
| **Mobile app** | React Native is a separate learning path |
| **Microservices** | Modular monolith is the right architecture at this scale |

---

> [!TIP]
> ## Spatial Check-In

1. **Why is a modular backend better than flat route files for CollabBoard?**

<details><summary>Answer</summary>

CollabBoard has 5 feature areas (auth, boards, lists, cards, comments) with ~20 endpoints. With flat files, you'd have 5 route files and 5 service files scattered across two directories. With modules, each feature is self-contained: boards/boards.routes.ts and boards/boards.service.ts live together. When you need to modify board logic, you open one folder.

</details>

2. **When should you NOT use optimistic updates?**

<details><summary>Answer</summary>

Don't use optimistic updates for operations that are likely to fail (creating an account — email might be taken), operations with complex server-side validation (payment processing), or operations where showing incorrect state briefly would be confusing or harmful. Optimistic updates work best for simple mutations that rarely fail: moving a card, toggling a checkbox, updating text.

</details>

3. **What does the CI/CD pipeline prevent that manual deployment doesn't?**

<details><summary>Answer</summary>

Manual deployment trusts the developer to run tests before pushing. CI/CD enforces it — tests run automatically on every push, and deployment is blocked if they fail. This is critical when multiple developers work on the same codebase. One developer's change might break another's feature, and without CI, no one notices until production breaks.

</details>

---

| | | |
|:---|:---:|---:|
| [← Level 4 — DataDash](../../level-4-scalable/) | [Level 5 Overview](../) | [02 — Project Setup →](../02-project-setup/) |
