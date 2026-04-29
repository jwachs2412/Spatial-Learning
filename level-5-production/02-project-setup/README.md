`Level 5` **Step 2 of 9** — Project Setup

# 02 — Project Setup: Modular Scaffold

> **Most of this lesson reuses commands and configs from Levels 1–4.** `mkdir`, `cd`, `git init`, the `.gitignore` patterns, the `tsconfig.json` shape, the `package.json` scripts, the Vitest configs, the shared-type interface pattern — all explained in detail in the corresponding setup lessons of earlier levels. This lesson only unpacks **what's new in Level 5**: nested brace expansion, the `@dnd-kit/*` packages, `@sentry/react`, and the deeply-nested types CollabBoard's API needs.

## Spatial Orientation

Same monorepo pattern as before — `client/` and `server/` — but with a modular backend and a `.github/workflows/` directory for CI/CD.

```
collab-board/                  ← Project root
├── client/                    ← React + Redux + dnd-kit + Sentry
│   └── src/
│       ├── app/               ← Redux store (carried from Level 4)
│       ├── features/          ← Feature folders (carried from Level 4)
│       ├── services/          ← API calls
│       ├── types/             ← Shared types
│       └── utils/             ← Formatters, helpers
├── server/                    ← Express + modular backend
│   └── src/
│       ├── modules/           ← Feature modules (auth, boards, etc.)    ← NEW
│       ├── middleware/        ← Logger, rate limiter, error handler
│       └── db/                ← Pool, schema, seed
├── .github/
│   └── workflows/
│       └── ci.yml             ← GitHub Actions pipeline               ← NEW
├── .env
└── .gitignore
```

---

## Step 1: Create the Project

> [!IMPORTANT]
> **You should be in:** your projects directory

```bash
mkdir collab-board && cd collab-board
```

> [!IMPORTANT]
> **You are now in:** `collab-board/`

```bash
git init
```

Create `.gitignore`:

```gitignore
node_modules/
dist/
.env
*.log
coverage/
```

Create a starter `README.md`:

```markdown
# CollabBoard — Collaborative Project Hub

React, TypeScript, Redux Toolkit, dnd-kit, Express, PostgreSQL

## Features
- Kanban-style boards with drag-and-drop cards
- Multi-user collaboration with board membership
- Card details with descriptions, assignments, and comments
- Optimistic UI updates for instant interaction
- Modular backend architecture
- CI/CD pipeline with GitHub Actions
- Error monitoring with Sentry

## Tech Stack
- **Frontend**: React, TypeScript, Redux Toolkit, React Router, dnd-kit, Recharts
- **Backend**: Express, TypeScript, PostgreSQL, bcrypt, JWT
- **Testing**: Vitest, React Testing Library, Supertest
- **DevOps**: GitHub Actions, Sentry
- **Deployment**: Vercel (frontend), Render (backend + database)

## Getting Started
1. `cd server && npm install && npm run dev`
2. `cd client && npm install && npm run dev`
3. Open http://localhost:5173
```

---

## Step 2: Set Up the Backend

```bash
mkdir -p server/src/{modules/{auth,boards,lists,cards,comments},middleware,db,types}
mkdir -p server/tests
```

**Reading that nested brace-expansion line:**

The shell expands the braces from the inside out. The inner `{auth,boards,lists,cards,comments}` becomes five separate words; the outer `{modules/{...},middleware,db,types}` then layers on top. The whole one-liner expands to nine `mkdir` calls:

```bash
mkdir -p server/src/modules/auth
mkdir -p server/src/modules/boards
mkdir -p server/src/modules/lists
mkdir -p server/src/modules/cards
mkdir -p server/src/modules/comments
mkdir -p server/src/middleware
mkdir -p server/src/db
mkdir -p server/src/types
```

`-p` means "create parent folders too if missing," so `server/`, `server/src/`, and `server/src/modules/` all appear automatically. One line of typing instead of nine.

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd server
npm init -y
```

> [!IMPORTANT]
> **You are now in:** `collab-board/server/`

### Install Dependencies

```bash
npm install express cors dotenv pg bcryptjs jsonwebtoken pino pino-http express-rate-limit
```

| Package | Purpose | Source |
|---------|---------|--------|
| `express`, `cors`, `dotenv`, `pg` | Core stack | Carried from Level 2 |
| `bcryptjs`, `jsonwebtoken` | Authentication | Carried from Level 3 |
| `pino`, `pino-http`, `express-rate-limit` | Middleware | Carried from Level 4 |

```bash
npm install typescript @types/node @types/express @types/cors @types/pg @types/bcryptjs @types/jsonwebtoken
```

```bash
npm install -D tsx vitest supertest @types/supertest pino-pretty
```

> [!WARNING]
> **`typescript` and all `@types/` packages MUST be in regular `dependencies`, not `devDependencies`.** Render skips `devDependencies` in production builds (`NODE_ENV=production`). If these packages are in `devDependencies`, `npm run build` (which runs `tsc`) will fail with `TS7016: Could not find declaration file` and `TS2580: Cannot find name 'process'` errors. The only exception is `@types/supertest` — it's only used in tests, never in production builds.

### TypeScript Configuration

Create `server/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### Package Scripts

Update `server/package.json` scripts:

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "seed": "tsx src/db/seed.ts",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

Create `server/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    testTimeout: 10000,
  },
});
```

---

## Step 3: Set Up the Frontend

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd ..
npm create vite@latest client -- --template react-ts
cd client
npm install
```

> [!IMPORTANT]
> **You are now in:** `collab-board/client/`

### Install Dependencies

```bash
npm install @reduxjs/toolkit react-redux react-router-dom @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities @sentry/react
```

| Package | Purpose | Source |
|---------|---------|--------|
| `@reduxjs/toolkit`, `react-redux` | State management | Carried from Level 4 |
| `react-router-dom` | Client-side routing | Carried from Level 3 |
| `@dnd-kit/core` | Drag-and-drop engine | **NEW** |
| `@dnd-kit/sortable` | Sortable lists and items | **NEW** |
| `@dnd-kit/utilities` | CSS transform helpers | **NEW** |
| `@sentry/react` | Error monitoring | **NEW** |

**The four new packages — what they actually do:**

- **`@dnd-kit/core`** — the base drag-and-drop engine. Provides `<DndContext>` (top-level wrapper), `useDraggable` (mark something as draggable), `useDroppable` (mark a target zone), and `onDragEnd` (a callback fired when the user drops). It handles all the pointer/keyboard/touch tracking under the hood.
- **`@dnd-kit/sortable`** — adds **sortable list** behavior on top of core. When you drag a card from position 2 to position 4, sortable shifts the other cards to make room and computes the new ordering. Without this, you'd manually calculate every position. Provides `<SortableContext>` and `useSortable`.
- **`@dnd-kit/utilities`** — small helper functions, primarily `CSS.Transform.toString()` for converting drag offsets into CSS transform strings. Used inside draggable components to animate them as the user drags.
- **`@sentry/react`** — the JavaScript client for Sentry, an error-monitoring service. When an unhandled error happens in production, Sentry captures it (with stack trace, user info, browser version, the URL when it happened) and sends it to your dashboard. Without Sentry, you find out about bugs from angry users; with Sentry, you find out the moment they happen.

The `@scope/name` syntax is **scoped npm packages** — packages published under an organization namespace. `@dnd-kit/core` lives under the `@dnd-kit` org. The slash isn't a folder; it's part of the package name.

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

### Create Feature-Based Folder Structure

```bash
mkdir -p src/app
mkdir -p src/features/{auth,boards,board,ui}
mkdir -p src/services
mkdir -p src/types
mkdir -p src/utils
```

### Vitest Configuration

Create `client/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test-setup.ts',
    css: true,
  },
});
```

Create `client/src/test-setup.ts`:

```typescript
import '@testing-library/jest-dom';
```

Update `client/package.json` scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

Update `client/vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
  },
});
```

---

> [!TIP]
> **Session Break** — You've scaffolded both backend and frontend with all dependencies and modular structure. Save your work and take a break.
> When you return, you'll create the database, define shared types, and set up CI/CD directories.

---

## Step 4: Create the Database

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd ..
createdb collabboard
```

Create `collab-board/.env`:

```env
DATABASE_URL=postgresql://localhost:5432/collabboard
CORS_ORIGIN=http://localhost:5173
PORT=3001
JWT_SECRET=dev-secret-change-in-production
JWT_EXPIRES_IN=7d
```

> [!WARNING]
> **`JWT_SECRET` is back** — CollabBoard includes authentication (carried from Level 3). As always, use a random secret in production.

---

## Step 5: Shared Types

Create `server/src/types/index.ts`:

```typescript
// --- Database row types ---

export interface User {
  id: number;
  email: string;
  password_hash: string;
  display_name: string;
  created_at: string;
}

export type SafeUser = Omit<User, 'password_hash'>;

export interface Board {
  id: number;
  name: string;
  description: string | null;
  owner_id: number;
  created_at: string;
  updated_at: string;
}

export interface BoardMember {
  id: number;
  board_id: number;
  user_id: number;
  role: string;
  joined_at: string;
}

export interface List {
  id: number;
  board_id: number;
  name: string;
  position: number;
  created_at: string;
}

export interface Card {
  id: number;
  list_id: number;
  title: string;
  description: string | null;
  position: number;
  assignee_id: number | null;
  created_at: string;
  updated_at: string;
}

export interface Comment {
  id: number;
  card_id: number;
  user_id: number;
  content: string;
  created_at: string;
}

// --- API request/response types ---

export interface BoardWithMembers extends Board {
  members: (SafeUser & { role: string })[];
}

export interface BoardWithLists extends Board {
  lists: ListWithCards[];
}

export interface ListWithCards extends List {
  cards: CardWithAssignee[];
}

export interface CardWithAssignee extends Card {
  assignee: SafeUser | null;
}

export interface CardDetail extends Card {
  assignee: SafeUser | null;
  comments: CommentWithUser[];
}

export interface CommentWithUser extends Comment {
  user: SafeUser;
}

export interface JWTPayload {
  userId: number;
  email: string;
}

export interface AuthRequest {
  user: JWTPayload;
}
```

Create `client/src/types/index.ts`:

```typescript
// Mirror backend types (excluding password-related types)

export interface SafeUser {
  id: number;
  email: string;
  display_name: string;
  created_at: string;
}

export interface Board {
  id: number;
  name: string;
  description: string | null;
  owner_id: number;
  created_at: string;
  updated_at: string;
}

export interface BoardWithMembers extends Board {
  members: (SafeUser & { role: string })[];
}

export interface List {
  id: number;
  board_id: number;
  name: string;
  position: number;
  created_at: string;
}

export interface Card {
  id: number;
  list_id: number;
  title: string;
  description: string | null;
  position: number;
  assignee_id: number | null;
  created_at: string;
  updated_at: string;
}

export interface CardWithAssignee extends Card {
  assignee: SafeUser | null;
}

export interface ListWithCards extends List {
  cards: CardWithAssignee[];
}

export interface BoardWithLists extends Board {
  lists: ListWithCards[];
}

export interface CardDetail extends Card {
  assignee: SafeUser | null;
  comments: CommentWithUser[];
}

export interface Comment {
  id: number;
  card_id: number;
  user_id: number;
  content: string;
  created_at: string;
}

export interface CommentWithUser extends Comment {
  user: SafeUser;
}
```

### Reading the Nested Types

Six base types match six database tables: `User`, `Board`, `BoardMember`, `List`, `Card`, `Comment`. Then **richer composed types** model what the API actually returns. Let's walk the nesting:

```typescript
export type SafeUser = Omit<User, 'password_hash'>;
```

`Omit<X, K>` (built-in TypeScript utility): the type `X` with key `K` removed. `SafeUser` is a `User` minus the password hash. Same pattern as Level 3.

```typescript
export interface BoardWithLists extends Board {
  lists: ListWithCards[];
}

export interface ListWithCards extends List {
  cards: CardWithAssignee[];
}

export interface CardWithAssignee extends Card {
  assignee: SafeUser | null;
}
```

`extends` in interfaces means "all fields of the parent, plus these new ones." So `BoardWithLists` has every Board field PLUS `lists: ListWithCards[]`. `ListWithCards` has every List field PLUS `cards: CardWithAssignee[]`. And so on, three levels deep.

When the backend serves `GET /api/boards/:id`, it joins across three tables and returns one big object matching `BoardWithLists`. The TypeScript shape is:

```typescript
{
  id: 1,
  name: 'Product Launch',
  description: '...',
  owner_id: 5,
  created_at: '...',
  updated_at: '...',
  lists: [
    {
      id: 10,
      board_id: 1,
      name: 'To Do',
      position: 0,
      created_at: '...',
      cards: [
        {
          id: 100,
          list_id: 10,
          title: 'Design mockups',
          ...
          assignee: { id: 5, email: '...', display_name: 'Alex', created_at: '...' },
        },
        // more cards...
      ],
    },
    // more lists...
  ],
}
```

> [!NOTE]
> **Technical:** The nested types (`BoardWithLists → ListWithCards → CardWithAssignee`) mirror the SQL JOIN structure. When you fetch a board, the backend JOINs across tables and nests the results. The frontend types match this nested shape exactly.
>
> **Plain English:** Think of these types as Russian nesting dolls. A board contains lists, lists contain cards, cards contain an assignee. The types describe exactly how deep the nesting goes for each API endpoint.

---

## Step 6: GitHub Actions Directory

Create the CI/CD workflow directory (we'll write the actual workflow in Step 7):

```bash
mkdir -p .github/workflows
```

---

## Step 7: Initial Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
git add .
git commit -m "feat: scaffold collab-board with modular backend structure"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why does CollabBoard need both `@dnd-kit/core` and `@dnd-kit/sortable`?**

<details><summary>Answer</summary>

`@dnd-kit/core` provides the base drag-and-drop engine — detecting when a user grabs an element, tracking movement, and detecting drop targets. `@dnd-kit/sortable` builds on top of core and adds sorted list behavior — when you drag a card between position 2 and position 4, it shifts other cards to make room. Without sortable, you'd have to calculate positions manually.

</details>

2. **Why does the backend use `modules/` instead of `routes/` and `services/` at the top level?**

<details><summary>Answer</summary>

With 5 feature areas and ~20 endpoints, flat directories become hard to navigate. You'd have auth.ts, boards.ts, lists.ts, cards.ts, and comments.ts in routes/, then the same five files in services/. Modules group related code: boards/boards.routes.ts and boards/boards.service.ts live next to each other. When you work on boards, you open one folder.

</details>

3. **Why is `@sentry/react` a production dependency (not dev)?**

<details><summary>Answer</summary>

Sentry runs in production — it catches errors from real users in the deployed app. Dev dependencies are only needed during development and testing (like TypeScript, Vitest, etc.). Sentry's React SDK is bundled into the production build and initializes when the app loads.

</details>

---

| | | |
|:---|:---:|---:|
| [← 01 — Orientation](../01-orientation/) | [Level 5 Overview](../) | [03 — Database →](../03-database/) |
