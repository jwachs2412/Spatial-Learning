`Level 5` **Step 5 of 9** — Frontend

# 05 — Frontend: Board UI, Redux Slices, and Card Details

## Spatial Orientation

The frontend connects to every backend module. React Router handles page-level navigation. Redux manages auth tokens, board lists, and card state. The API service layer centralizes all HTTP calls with automatic token injection.

```
┌───────────────────────────────────────────────────────────────────────┐
│  BrowserRouter                                                        │
│  ├── /login  → LoginPage                                             │
│  ├── /register → RegisterPage                                        │
│  ├── /boards → ProtectedRoute → Layout                               │
│  │               └── BoardList (grid of boards, create form)         │
│  └── /boards/:id → ProtectedRoute → Layout                          │
│                      └── BoardView                                    │
│                          ├── ListColumn (per list)                    │
│                          │   ├── CardItem (per card)                  │
│                          │   └── "Add card" form                     │
│                          └── CardDetailModal (on card click)         │
│                              ├── Description editor                   │
│                              ├── Assignee dropdown                    │
│                              └── Comments list                        │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Redux Store and Hooks

These are carried from Level 4 — same pattern, new slices.

> [!IMPORTANT]
> **You should be in:** `collab-board/client/`

Create `src/app/store.ts`:

```typescript
import { configureStore } from '@reduxjs/toolkit';
import authReducer from '../features/auth/authSlice';
import boardsReducer from '../features/boards/boardsSlice';
import boardReducer from '../features/board/boardSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    boards: boardsReducer,
    board: boardReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

Create `src/app/hooks.ts`:

```typescript
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

---

## Step 2: API Service Layer

This file centralizes every API call. Every function attaches the JWT token from `localStorage`.

Create `src/services/api.ts`:

```typescript
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

// --- Token management ---

export function getToken(): string | null {
  return localStorage.getItem('token');
}

export function setToken(token: string): void {
  localStorage.setItem('token', token);
}

export function removeToken(): void {
  localStorage.removeItem('token');
}

// --- Headers ---

function authHeaders(): Record<string, string> {
  const token = getToken();
  return {
    'Content-Type': 'application/json',
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
  };
}

// --- Fetch helper ---

async function fetchJSON<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const res = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...authHeaders(),
      ...options.headers,
    },
  });

  if (!res.ok) {
    const body = await res.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(body.error || `HTTP ${res.status}`);
  }

  // 204 No Content — nothing to parse
  if (res.status === 204) return undefined as T;

  return res.json();
}

// --- Auth ---

export async function login(email: string, password: string) {
  return fetchJSON<{ user: any; token: string }>('/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  });
}

export async function register(
  email: string,
  password: string,
  displayName: string
) {
  return fetchJSON<{ user: any; token: string }>('/auth/register', {
    method: 'POST',
    body: JSON.stringify({ email, password, displayName }),
  });
}

export async function getMe() {
  return fetchJSON<any>('/auth/me');
}

// --- Boards ---

export async function getBoards() {
  return fetchJSON<any[]>('/boards');
}

export async function getBoard(id: number) {
  return fetchJSON<any>(`/boards/${id}`);
}

export async function createBoard(name: string, description?: string) {
  return fetchJSON<any>('/boards', {
    method: 'POST',
    body: JSON.stringify({ name, description: description || null }),
  });
}

export async function deleteBoard(id: number) {
  return fetchJSON<void>(`/boards/${id}`, { method: 'DELETE' });
}

// --- Lists ---

export async function createList(boardId: number, name: string) {
  return fetchJSON<any>(`/boards/${boardId}/lists`, {
    method: 'POST',
    body: JSON.stringify({ name }),
  });
}

// --- Cards ---

export async function createCard(
  listId: number,
  title: string,
  description?: string
) {
  return fetchJSON<any>(`/lists/${listId}/cards`, {
    method: 'POST',
    body: JSON.stringify({ title, description }),
  });
}

export async function updateCard(cardId: number, updates: Record<string, any>) {
  return fetchJSON<any>(`/cards/${cardId}`, {
    method: 'PUT',
    body: JSON.stringify(updates),
  });
}

export async function moveCard(
  cardId: number,
  targetListId: number,
  targetPosition: number
) {
  return fetchJSON<any>(`/cards/${cardId}/move`, {
    method: 'PUT',
    body: JSON.stringify({ targetListId, targetPosition }),
  });
}

export async function deleteCard(cardId: number) {
  return fetchJSON<void>(`/cards/${cardId}`, { method: 'DELETE' });
}

// --- Comments ---

export async function addComment(cardId: number, content: string) {
  return fetchJSON<any>(`/cards/${cardId}/comments`, {
    method: 'POST',
    body: JSON.stringify({ content }),
  });
}

export async function deleteComment(commentId: number) {
  return fetchJSON<void>(`/comments/${commentId}`, { method: 'DELETE' });
}

// --- Members ---

export async function addMember(boardId: number, email: string) {
  return fetchJSON<any>(`/boards/${boardId}/members`, {
    method: 'POST',
    body: JSON.stringify({ email }),
  });
}

export async function getMembers(boardId: number) {
  return fetchJSON<any[]>(`/boards/${boardId}/members`);
}
```

> [!NOTE]
> **Technical:** The `fetchJSON` helper handles three concerns: (1) attaching the auth token to every request via `authHeaders()`, (2) parsing JSON responses, and (3) throwing on non-OK status codes so Redux thunks can catch errors. The `204` check prevents JSON parse errors on delete endpoints.
>
> **Plain English:** Instead of writing `fetch()` + `headers` + `res.json()` in every function, one helper does it all. Every API function becomes a clean one-liner that just describes what it wants — the plumbing is invisible.

---

## Step 3: Auth Slice

Create `src/features/auth/authSlice.ts`:

```typescript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { SafeUser } from '../../types';
import * as api from '../../services/api';

// --- State ---

interface AuthState {
  user: SafeUser | null;
  token: string | null;
  loading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  token: api.getToken(),       // Restore token from localStorage on load
  loading: false,
  error: null,
};

// --- Thunks ---

export const loginUser = createAsyncThunk(
  'auth/login',
  async ({ email, password }: { email: string; password: string }) => {
    const data = await api.login(email, password);
    api.setToken(data.token);  // Persist to localStorage
    return data;
  }
);

export const registerUser = createAsyncThunk(
  'auth/register',
  async ({
    email,
    password,
    displayName,
  }: {
    email: string;
    password: string;
    displayName: string;
  }) => {
    const data = await api.register(email, password, displayName);
    api.setToken(data.token);
    return data;
  }
);

export const fetchCurrentUser = createAsyncThunk(
  'auth/fetchCurrentUser',
  async () => {
    return await api.getMe();
  }
);

export const logoutUser = createAsyncThunk('auth/logout', async () => {
  api.removeToken();
});

// --- Slice ---

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    // Login
    builder.addCase(loginUser.pending, (state) => {
      state.loading = true;
      state.error = null;
    });
    builder.addCase(loginUser.fulfilled, (state, action) => {
      state.loading = false;
      state.user = action.payload.user;
      state.token = action.payload.token;
    });
    builder.addCase(loginUser.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || 'Login failed';
    });

    // Register
    builder.addCase(registerUser.pending, (state) => {
      state.loading = true;
      state.error = null;
    });
    builder.addCase(registerUser.fulfilled, (state, action) => {
      state.loading = false;
      state.user = action.payload.user;
      state.token = action.payload.token;
    });
    builder.addCase(registerUser.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || 'Registration failed';
    });

    // Fetch current user
    builder.addCase(fetchCurrentUser.fulfilled, (state, action) => {
      state.user = action.payload;
    });
    builder.addCase(fetchCurrentUser.rejected, (state) => {
      state.user = null;
      state.token = null;
      api.removeToken();
    });

    // Logout
    builder.addCase(logoutUser.fulfilled, (state) => {
      state.user = null;
      state.token = null;
    });
  },
});

// --- Selectors ---

export const selectUser = (state: RootState) => state.auth.user;
export const selectToken = (state: RootState) => state.auth.token;
export const selectIsAuthenticated = (state: RootState) => !!state.auth.token;
export const selectAuthLoading = (state: RootState) => state.auth.loading;
export const selectAuthError = (state: RootState) => state.auth.error;

export default authSlice.reducer;
```

> [!WARNING]
> **The token lives in two places:** Redux state (for reactive UI) and `localStorage` (for persistence across refreshes). On app load, `initialState.token` reads from `localStorage`. On login, the thunk writes to both. On logout, the thunk clears both. If these ever get out of sync, the user sees a logged-in UI but gets 401 errors — which is why `fetchCurrentUser.rejected` clears everything.

---

## Step 4: Boards Slice

Create `src/features/boards/boardsSlice.ts`:

```typescript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { Board } from '../../types';
import * as api from '../../services/api';

interface BoardsState {
  boards: Board[];
  loading: boolean;
  error: string | null;
}

const initialState: BoardsState = {
  boards: [],
  loading: false,
  error: null,
};

// --- Thunks ---

export const fetchBoards = createAsyncThunk('boards/fetchAll', async () => {
  return await api.getBoards();
});

export const createBoard = createAsyncThunk(
  'boards/create',
  async ({ name, description }: { name: string; description?: string }) => {
    return await api.createBoard(name, description);
  }
);

export const deleteBoard = createAsyncThunk(
  'boards/delete',
  async (id: number) => {
    await api.deleteBoard(id);
    return id;
  }
);

// --- Slice ---

const boardsSlice = createSlice({
  name: 'boards',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(fetchBoards.pending, (state) => {
      state.loading = true;
      state.error = null;
    });
    builder.addCase(fetchBoards.fulfilled, (state, action) => {
      state.loading = false;
      state.boards = action.payload;
    });
    builder.addCase(fetchBoards.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || 'Failed to load boards';
    });

    builder.addCase(createBoard.fulfilled, (state, action) => {
      state.boards.unshift(action.payload);
    });

    builder.addCase(deleteBoard.fulfilled, (state, action) => {
      state.boards = state.boards.filter((b) => b.id !== action.payload);
    });
  },
});

// --- Selectors ---

export const selectBoards = (state: RootState) => state.boards.boards;
export const selectBoardsLoading = (state: RootState) => state.boards.loading;

export default boardsSlice.reducer;
```

---

## Step 5: Board Slice (Single Board with Lists and Cards)

This slice manages the currently-viewed board — the one at `/boards/:id`.

Create `src/features/board/boardSlice.ts`:

```typescript
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { BoardWithLists, CardWithAssignee } from '../../types';
import * as api from '../../services/api';

interface BoardState {
  currentBoard: BoardWithLists | null;
  loading: boolean;
  error: string | null;
}

const initialState: BoardState = {
  currentBoard: null,
  loading: false,
  error: null,
};

// --- Thunks ---

export const fetchBoard = createAsyncThunk(
  'board/fetch',
  async (id: number) => {
    return await api.getBoard(id);
  }
);

export const createList = createAsyncThunk(
  'board/createList',
  async ({ boardId, name }: { boardId: number; name: string }) => {
    return await api.createList(boardId, name);
  }
);

export const createCard = createAsyncThunk(
  'board/createCard',
  async ({
    listId,
    title,
    description,
  }: {
    listId: number;
    title: string;
    description?: string;
  }) => {
    return await api.createCard(listId, title, description);
  }
);

export const updateCard = createAsyncThunk(
  'board/updateCard',
  async ({
    cardId,
    updates,
  }: {
    cardId: number;
    updates: Record<string, any>;
  }) => {
    return await api.updateCard(cardId, updates);
  }
);

export const moveCardAsync = createAsyncThunk(
  'board/moveCard',
  async ({
    cardId,
    targetListId,
    targetPosition,
  }: {
    cardId: number;
    targetListId: number;
    targetPosition: number;
  }) => {
    return await api.moveCard(cardId, targetListId, targetPosition);
  }
);

export const deleteCard = createAsyncThunk(
  'board/deleteCard',
  async ({ cardId, listId }: { cardId: number; listId: number }) => {
    await api.deleteCard(cardId);
    return { cardId, listId };
  }
);

// --- Slice ---

const boardSlice = createSlice({
  name: 'board',
  initialState,
  reducers: {
    clearBoard(state) {
      state.currentBoard = null;
      state.error = null;
    },

    // Optimistic move — Step 6 (Drag & Drop) will dispatch this
    moveCardOptimistic(
      state,
      action: PayloadAction<{
        cardId: number;
        sourceListId: number;
        targetListId: number;
        targetPosition: number;
      }>
    ) {
      if (!state.currentBoard) return;

      const { cardId, sourceListId, targetListId, targetPosition } =
        action.payload;

      // Find the card in the source list
      const sourceList = state.currentBoard.lists.find(
        (l) => l.id === sourceListId
      );
      if (!sourceList) return;

      const cardIndex = sourceList.cards.findIndex((c) => c.id === cardId);
      if (cardIndex === -1) return;

      // Remove card from source
      const [card] = sourceList.cards.splice(cardIndex, 1);

      // Update card's list_id
      card.list_id = targetListId;

      // Insert into target list
      const targetList = state.currentBoard.lists.find(
        (l) => l.id === targetListId
      );
      if (!targetList) return;

      targetList.cards.splice(targetPosition, 0, card);

      // Recalculate positions in both lists
      sourceList.cards.forEach((c, i) => (c.position = i));
      targetList.cards.forEach((c, i) => (c.position = i));
    },
  },
  extraReducers: (builder) => {
    builder.addCase(fetchBoard.pending, (state) => {
      state.loading = true;
      state.error = null;
    });
    builder.addCase(fetchBoard.fulfilled, (state, action) => {
      state.loading = false;
      state.currentBoard = action.payload;
    });
    builder.addCase(fetchBoard.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message || 'Failed to load board';
    });

    // Create list — add to the board's lists array
    builder.addCase(createList.fulfilled, (state, action) => {
      if (state.currentBoard) {
        state.currentBoard.lists.push({ ...action.payload, cards: [] });
      }
    });

    // Create card — add to the correct list
    builder.addCase(createCard.fulfilled, (state, action) => {
      if (!state.currentBoard) return;
      const list = state.currentBoard.lists.find(
        (l) => l.id === action.payload.list_id
      );
      if (list) {
        list.cards.push({ ...action.payload, assignee: null });
      }
    });

    // Update card — replace in the correct list
    builder.addCase(updateCard.fulfilled, (state, action) => {
      if (!state.currentBoard) return;
      for (const list of state.currentBoard.lists) {
        const idx = list.cards.findIndex((c) => c.id === action.payload.id);
        if (idx !== -1) {
          list.cards[idx] = {
            ...list.cards[idx],
            ...action.payload,
          };
          break;
        }
      }
    });

    // Delete card — remove from the correct list
    builder.addCase(deleteCard.fulfilled, (state, action) => {
      if (!state.currentBoard) return;
      const list = state.currentBoard.lists.find(
        (l) => l.id === action.payload.listId
      );
      if (list) {
        list.cards = list.cards.filter((c) => c.id !== action.payload.cardId);
      }
    });

    // Move card (async confirmation) — refetch to get server truth
    builder.addCase(moveCardAsync.rejected, (state) => {
      // On failure the optimistic update is wrong — refetch will fix it.
      // The component will re-dispatch fetchBoard.
      state.error = 'Failed to move card. Refreshing board...';
    });
  },
});

export const { clearBoard, moveCardOptimistic } = boardSlice.actions;

// --- Selectors ---

export const selectCurrentBoard = (state: RootState) =>
  state.board.currentBoard;
export const selectBoardLoading = (state: RootState) => state.board.loading;
export const selectBoardError = (state: RootState) => state.board.error;

export default boardSlice.reducer;
```

> [!NOTE]
> **Technical:** The `moveCardOptimistic` reducer uses Immer (built into Redux Toolkit) to mutate state directly — `splice`, `push`, index assignment. Immer tracks these mutations and produces immutable updates under the hood. The async `moveCardAsync` thunk sends the change to the server. If it fails, the component refetches the entire board to restore server truth.
>
> **Plain English:** When you drag a card, two things happen: (1) the card moves instantly in the UI via `moveCardOptimistic`, and (2) a background request tells the server. If the server says "no," we reload the board and the card snaps back. The user never waits.

---

## Step 6: Auth Pages

### Login Page

Create `src/features/auth/LoginPage.tsx`:

```typescript
import { useState, FormEvent } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import { loginUser, selectAuthLoading, selectAuthError } from './authSlice';

export default function LoginPage() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const loading = useAppSelector(selectAuthLoading);
  const error = useAppSelector(selectAuthError);

  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const result = await dispatch(loginUser({ email, password }));
    if (loginUser.fulfilled.match(result)) {
      navigate('/boards');
    }
  }

  return (
    <div className="auth-page">
      <form className="auth-form" onSubmit={handleSubmit}>
        <h1 className="auth-title">Log In to CollabBoard</h1>

        {error && <p className="auth-error">{error}</p>}

        <label className="auth-label">
          Email
          <input
            type="email"
            className="auth-input"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </label>

        <label className="auth-label">
          Password
          <input
            type="password"
            className="auth-input"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </label>

        <button className="auth-button" type="submit" disabled={loading}>
          {loading ? 'Logging in...' : 'Log In'}
        </button>

        <p className="auth-switch">
          No account? <Link to="/register">Register</Link>
        </p>
      </form>
    </div>
  );
}
```

### Register Page

Create `src/features/auth/RegisterPage.tsx`:

```typescript
import { useState, FormEvent } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  registerUser,
  selectAuthLoading,
  selectAuthError,
} from './authSlice';

export default function RegisterPage() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const loading = useAppSelector(selectAuthLoading);
  const error = useAppSelector(selectAuthError);

  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [displayName, setDisplayName] = useState('');

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const result = await dispatch(
      registerUser({ email, password, displayName })
    );
    if (registerUser.fulfilled.match(result)) {
      navigate('/boards');
    }
  }

  return (
    <div className="auth-page">
      <form className="auth-form" onSubmit={handleSubmit}>
        <h1 className="auth-title">Create Account</h1>

        {error && <p className="auth-error">{error}</p>}

        <label className="auth-label">
          Display Name
          <input
            type="text"
            className="auth-input"
            value={displayName}
            onChange={(e) => setDisplayName(e.target.value)}
            required
          />
        </label>

        <label className="auth-label">
          Email
          <input
            type="email"
            className="auth-input"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </label>

        <label className="auth-label">
          Password
          <input
            type="password"
            className="auth-input"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
            minLength={6}
          />
        </label>

        <button className="auth-button" type="submit" disabled={loading}>
          {loading ? 'Creating account...' : 'Register'}
        </button>

        <p className="auth-switch">
          Already have an account? <Link to="/login">Log In</Link>
        </p>
      </form>
    </div>
  );
}
```

---

## Step 7: Board List Page

Create `src/features/boards/BoardList.tsx`:

```typescript
import { useEffect, useState, FormEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  fetchBoards,
  createBoard,
  selectBoards,
  selectBoardsLoading,
} from './boardsSlice';

export default function BoardList() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const boards = useAppSelector(selectBoards);
  const loading = useAppSelector(selectBoardsLoading);

  const [name, setName] = useState('');
  const [showForm, setShowForm] = useState(false);

  useEffect(() => {
    dispatch(fetchBoards());
  }, [dispatch]);

  async function handleCreate(e: FormEvent) {
    e.preventDefault();
    if (!name.trim()) return;
    const result = await dispatch(createBoard({ name: name.trim() }));
    if (createBoard.fulfilled.match(result)) {
      setName('');
      setShowForm(false);
    }
  }

  if (loading && boards.length === 0) {
    return <div className="boards-loading">Loading boards...</div>;
  }

  return (
    <div className="boards-page">
      <div className="boards-header">
        <h1>Your Boards</h1>
        <button
          className="boards-create-btn"
          onClick={() => setShowForm(!showForm)}
        >
          + New Board
        </button>
      </div>

      {showForm && (
        <form className="boards-create-form" onSubmit={handleCreate}>
          <input
            type="text"
            className="boards-create-input"
            placeholder="Board name..."
            value={name}
            onChange={(e) => setName(e.target.value)}
            autoFocus
          />
          <button className="boards-create-submit" type="submit">
            Create
          </button>
        </form>
      )}

      <div className="boards-grid">
        {boards.map((board) => (
          <div
            key={board.id}
            className="board-card"
            onClick={() => navigate(`/boards/${board.id}`)}
          >
            <h2 className="board-card-name">{board.name}</h2>
            {board.description && (
              <p className="board-card-desc">{board.description}</p>
            )}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Step 8: Board View Components

### BoardView — the main board page

Create `src/features/board/BoardView.tsx`:

```typescript
import { useEffect, useState, FormEvent } from 'react';
import { useParams } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  fetchBoard,
  createList,
  clearBoard,
  selectCurrentBoard,
  selectBoardLoading,
} from './boardSlice';
import ListColumn from './ListColumn';
import CardDetailModal from './CardDetailModal';

export default function BoardView() {
  const { id } = useParams<{ id: string }>();
  const dispatch = useAppDispatch();
  const board = useAppSelector(selectCurrentBoard);
  const loading = useAppSelector(selectBoardLoading);

  const [newListName, setNewListName] = useState('');
  const [selectedCardId, setSelectedCardId] = useState<number | null>(null);

  useEffect(() => {
    if (id) dispatch(fetchBoard(Number(id)));
    return () => { dispatch(clearBoard()); };
  }, [dispatch, id]);

  function handleCreateList(e: FormEvent) {
    e.preventDefault();
    if (!newListName.trim() || !board) return;
    dispatch(createList({ boardId: board.id, name: newListName.trim() }));
    setNewListName('');
  }

  if (loading && !board) {
    return <div className="board-loading">Loading board...</div>;
  }

  if (!board) {
    return <div className="board-loading">Board not found.</div>;
  }

  return (
    <div className="board-view">
      <h1 className="board-view-title">{board.name}</h1>

      <div className="board-lists">
        {board.lists.map((list) => (
          <ListColumn
            key={list.id}
            list={list}
            onCardClick={(cardId) => setSelectedCardId(cardId)}
          />
        ))}

        {/* Add list form */}
        <div className="list-column list-column--add">
          <form onSubmit={handleCreateList}>
            <input
              type="text"
              className="add-list-input"
              placeholder="+ Add list..."
              value={newListName}
              onChange={(e) => setNewListName(e.target.value)}
            />
          </form>
        </div>
      </div>

      {selectedCardId !== null && (
        <CardDetailModal
          cardId={selectedCardId}
          boardId={board.id}
          onClose={() => setSelectedCardId(null)}
        />
      )}
    </div>
  );
}
```

### ListColumn — a single list with its cards

Create `src/features/board/ListColumn.tsx`:

```typescript
import { useState, FormEvent } from 'react';
import { useAppDispatch } from '../../app/hooks';
import { createCard } from './boardSlice';
import CardItem from './CardItem';
import type { ListWithCards } from '../../types';

interface ListColumnProps {
  list: ListWithCards;
  onCardClick: (cardId: number) => void;
}

export default function ListColumn({ list, onCardClick }: ListColumnProps) {
  const dispatch = useAppDispatch();
  const [title, setTitle] = useState('');
  const [showForm, setShowForm] = useState(false);

  function handleAdd(e: FormEvent) {
    e.preventDefault();
    if (!title.trim()) return;
    dispatch(createCard({ listId: list.id, title: title.trim() }));
    setTitle('');
    setShowForm(false);
  }

  return (
    <div className="list-column">
      <h3 className="list-column-header">{list.name}</h3>

      <div className="list-cards">
        {list.cards.map((card) => (
          <CardItem
            key={card.id}
            card={card}
            onClick={() => onCardClick(card.id)}
          />
        ))}
      </div>

      {showForm ? (
        <form className="add-card-form" onSubmit={handleAdd}>
          <input
            type="text"
            className="add-card-input"
            placeholder="Card title..."
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            autoFocus
          />
          <div className="add-card-actions">
            <button className="add-card-submit" type="submit">
              Add
            </button>
            <button
              className="add-card-cancel"
              type="button"
              onClick={() => setShowForm(false)}
            >
              Cancel
            </button>
          </div>
        </form>
      ) : (
        <button
          className="add-card-trigger"
          onClick={() => setShowForm(true)}
        >
          + Add a card
        </button>
      )}
    </div>
  );
}
```

### CardItem — card preview in a list

Create `src/features/board/CardItem.tsx`:

```typescript
import type { CardWithAssignee } from '../../types';

interface CardItemProps {
  card: CardWithAssignee;
  onClick: () => void;
}

export default function CardItem({ card, onClick }: CardItemProps) {
  return (
    <div className="card-item" onClick={onClick}>
      <span className="card-item-title">{card.title}</span>
      {card.assignee && (
        <span className="card-item-assignee" title={card.assignee.display_name}>
          {card.assignee.display_name.charAt(0).toUpperCase()}
        </span>
      )}
    </div>
  );
}
```

### CardDetailModal — full card view with comments

Create `src/features/board/CardDetailModal.tsx`:

```typescript
import { useEffect, useState, FormEvent } from 'react';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  updateCard,
  deleteCard,
  fetchBoard,
  selectCurrentBoard,
} from './boardSlice';
import Modal from '../ui/Modal';
import * as api from '../../services/api';
import type { CardDetail as CardDetailType, SafeUser } from '../../types';

interface CardDetailModalProps {
  cardId: number;
  boardId: number;
  onClose: () => void;
}

export default function CardDetailModal({
  cardId,
  boardId,
  onClose,
}: CardDetailModalProps) {
  const dispatch = useAppDispatch();
  const board = useAppSelector(selectCurrentBoard);

  const [card, setCard] = useState<CardDetailType | null>(null);
  const [members, setMembers] = useState<SafeUser[]>([]);
  const [comment, setComment] = useState('');
  const [description, setDescription] = useState('');
  const [editingDesc, setEditingDesc] = useState(false);

  // Fetch card detail and board members
  useEffect(() => {
    api.getMembers(boardId).then(setMembers).catch(() => {});

    // We don't have a getCardDetail in the slice — use the API directly
    // The card detail endpoint returns comments
    fetch(
      `${import.meta.env.VITE_API_URL || 'http://localhost:3001/api'}/cards/${cardId}`,
      { headers: { Authorization: `Bearer ${api.getToken()}` } }
    )
      .then((res) => res.json())
      .then((data) => {
        setCard(data);
        setDescription(data.description || '');
      })
      .catch(() => {});
  }, [cardId, boardId]);

  async function handleSaveDescription() {
    if (!card) return;
    await dispatch(
      updateCard({ cardId: card.id, updates: { description } })
    );
    setCard({ ...card, description });
    setEditingDesc(false);
  }

  async function handleAssign(assigneeId: number | null) {
    if (!card) return;
    await dispatch(
      updateCard({ cardId: card.id, updates: { assignee_id: assigneeId } })
    );
    const assignee = assigneeId
      ? members.find((m) => m.id === assigneeId) || null
      : null;
    setCard({ ...card, assignee_id: assigneeId, assignee });
  }

  async function handleAddComment(e: FormEvent) {
    e.preventDefault();
    if (!comment.trim() || !card) return;
    const newComment = await api.addComment(card.id, comment.trim());
    setCard({ ...card, comments: [...card.comments, newComment] });
    setComment('');
  }

  async function handleDeleteComment(commentId: number) {
    if (!card) return;
    await api.deleteComment(commentId);
    setCard({
      ...card,
      comments: card.comments.filter((c) => c.id !== commentId),
    });
  }

  async function handleDelete() {
    if (!card || !board) return;
    await dispatch(deleteCard({ cardId: card.id, listId: card.list_id }));
    onClose();
  }

  if (!card) {
    return (
      <Modal onClose={onClose}>
        <div className="card-detail-loading">Loading card...</div>
      </Modal>
    );
  }

  return (
    <Modal onClose={onClose}>
      <div className="card-detail">
        <h2 className="card-detail-title">{card.title}</h2>

        {/* Description */}
        <div className="card-detail-section">
          <h3>Description</h3>
          {editingDesc ? (
            <div className="card-detail-desc-edit">
              <textarea
                className="card-detail-textarea"
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                rows={3}
              />
              <div className="card-detail-desc-actions">
                <button onClick={handleSaveDescription}>Save</button>
                <button onClick={() => setEditingDesc(false)}>Cancel</button>
              </div>
            </div>
          ) : (
            <p
              className="card-detail-desc"
              onClick={() => setEditingDesc(true)}
            >
              {card.description || 'Click to add a description...'}
            </p>
          )}
        </div>

        {/* Assignee */}
        <div className="card-detail-section">
          <h3>Assignee</h3>
          <select
            className="card-detail-select"
            value={card.assignee_id || ''}
            onChange={(e) =>
              handleAssign(e.target.value ? Number(e.target.value) : null)
            }
          >
            <option value="">Unassigned</option>
            {members.map((m) => (
              <option key={m.id} value={m.id}>
                {m.display_name}
              </option>
            ))}
          </select>
        </div>

        {/* Comments */}
        <div className="card-detail-section">
          <h3>Comments</h3>
          <div className="card-detail-comments">
            {card.comments.map((c) => (
              <div key={c.id} className="comment">
                <div className="comment-header">
                  <strong>{c.user.display_name}</strong>
                  <button
                    className="comment-delete"
                    onClick={() => handleDeleteComment(c.id)}
                  >
                    x
                  </button>
                </div>
                <p className="comment-content">{c.content}</p>
              </div>
            ))}
          </div>

          <form className="comment-form" onSubmit={handleAddComment}>
            <input
              type="text"
              className="comment-input"
              placeholder="Add a comment..."
              value={comment}
              onChange={(e) => setComment(e.target.value)}
            />
            <button className="comment-submit" type="submit">
              Send
            </button>
          </form>
        </div>

        {/* Delete */}
        <button className="card-detail-delete" onClick={handleDelete}>
          Delete Card
        </button>
      </div>
    </Modal>
  );
}
```

---

## Step 9: UI Components

### Modal

Create `src/features/ui/Modal.tsx`:

```typescript
import { ReactNode, useEffect } from 'react';

interface ModalProps {
  children: ReactNode;
  onClose: () => void;
}

export default function Modal({ children, onClose }: ModalProps) {
  // Close on Escape key
  useEffect(() => {
    function handleKey(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose();
    }
    window.addEventListener('keydown', handleKey);
    return () => window.removeEventListener('keydown', handleKey);
  }, [onClose]);

  return (
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <button className="modal-close" onClick={onClose}>
          x
        </button>
        {children}
      </div>
    </div>
  );
}
```

### Layout

Create `src/features/ui/Layout.tsx`:

```typescript
import { ReactNode } from 'react';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import { logoutUser, selectUser } from '../auth/authSlice';
import { useNavigate } from 'react-router-dom';

interface LayoutProps {
  children: ReactNode;
}

export default function Layout({ children }: LayoutProps) {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const user = useAppSelector(selectUser);

  async function handleLogout() {
    await dispatch(logoutUser());
    navigate('/login');
  }

  return (
    <div className="app-layout">
      <header className="app-header">
        <h1 className="app-header-title">CollabBoard</h1>
        <div className="app-header-right">
          {user && <span className="app-header-user">{user.display_name}</span>}
          <button className="app-header-logout" onClick={handleLogout}>
            Log Out
          </button>
        </div>
      </header>
      <main className="app-main">{children}</main>
    </div>
  );
}
```

### ProtectedRoute

Create `src/features/ui/ProtectedRoute.tsx`:

```typescript
import { ReactNode, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAppSelector, useAppDispatch } from '../../app/hooks';
import {
  selectIsAuthenticated,
  selectUser,
  fetchCurrentUser,
} from '../auth/authSlice';

interface ProtectedRouteProps {
  children: ReactNode;
}

export default function ProtectedRoute({ children }: ProtectedRouteProps) {
  const navigate = useNavigate();
  const dispatch = useAppDispatch();
  const isAuthenticated = useAppSelector(selectIsAuthenticated);
  const user = useAppSelector(selectUser);

  useEffect(() => {
    if (!isAuthenticated) {
      navigate('/login');
    } else if (!user) {
      dispatch(fetchCurrentUser());
    }
  }, [isAuthenticated, user, navigate, dispatch]);

  if (!isAuthenticated) return null;

  return <>{children}</>;
}
```

---

## Step 10: App Component with React Router

Update `src/App.tsx`:

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { Provider } from 'react-redux';
import { store } from './app/store';
import LoginPage from './features/auth/LoginPage';
import RegisterPage from './features/auth/RegisterPage';
import BoardList from './features/boards/BoardList';
import BoardView from './features/board/BoardView';
import Layout from './features/ui/Layout';
import ProtectedRoute from './features/ui/ProtectedRoute';
import './App.css';

export default function App() {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/register" element={<RegisterPage />} />

          <Route
            path="/boards"
            element={
              <ProtectedRoute>
                <Layout>
                  <BoardList />
                </Layout>
              </ProtectedRoute>
            }
          />

          <Route
            path="/boards/:id"
            element={
              <ProtectedRoute>
                <Layout>
                  <BoardView />
                </Layout>
              </ProtectedRoute>
            }
          />

          <Route path="*" element={<Navigate to="/boards" replace />} />
        </Routes>
      </BrowserRouter>
    </Provider>
  );
}
```

Update `src/main.tsx`:

```typescript
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

```
DATA FLOW: User opens /boards/3

  BrowserRouter
    │
    ├── matches /boards/:id  →  id = 3
    │
    ProtectedRoute
    │  ├── token in Redux? ─ No → redirect /login
    │  └── token exists?   ─ Yes ──┐
    │                               │
    Layout                          │
    │  └── header, logout btn       │
    │                               │
    BoardView ◄─────────────────────┘
       │
       ├── useEffect → dispatch(fetchBoard(3))
       │       │
       │       └── GET /api/boards/3
       │           └── returns { ...board, lists: [{ ...list, cards: [...] }] }
       │
       ├── board.lists.map → ListColumn
       │       │
       │       └── list.cards.map → CardItem
       │
       └── CardItem click → setSelectedCardId → CardDetailModal
               │
               ├── fetch /api/cards/:id (detail with comments)
               ├── fetch /api/boards/:id/members (for assignee dropdown)
               └── renders description, assignee select, comments list
```

---

## Step 11: Styles

Replace `src/App.css`:

```css
/* === Reset & Base === */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #0f0f1a;
  color: #e0e0e0;
  line-height: 1.6;
}

/* === App Layout === */

.app-layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 24px;
  background-color: #1a1a2e;
  border-bottom: 1px solid #2a2a4a;
}

.app-header-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: #6366f1;
}

.app-header-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.app-header-user {
  font-size: 0.9rem;
  color: #888;
}

.app-header-logout {
  padding: 6px 14px;
  border: 1px solid #333;
  background: none;
  color: #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
}

.app-header-logout:hover {
  border-color: #ef4444;
  color: #ef4444;
}

.app-main {
  flex: 1;
  padding: 24px;
}

/* === Auth Pages === */

.auth-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

.auth-form {
  width: 100%;
  max-width: 400px;
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 32px;
}

.auth-title {
  font-size: 1.4rem;
  font-weight: 700;
  margin-bottom: 24px;
  color: #e0e0e0;
}

.auth-label {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 16px;
  font-size: 0.85rem;
  color: #888;
}

.auth-input {
  padding: 10px 12px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.95rem;
}

.auth-input:focus {
  outline: none;
  border-color: #6366f1;
}

.auth-button {
  width: 100%;
  padding: 12px;
  border: none;
  background-color: #6366f1;
  color: #fff;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-top: 8px;
}

.auth-button:hover {
  background-color: #4f46e5;
}

.auth-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.auth-error {
  color: #ef4444;
  font-size: 0.85rem;
  margin-bottom: 16px;
  padding: 8px 12px;
  background-color: rgba(239, 68, 68, 0.1);
  border-radius: 6px;
}

.auth-switch {
  margin-top: 16px;
  text-align: center;
  font-size: 0.85rem;
  color: #888;
}

.auth-switch a {
  color: #6366f1;
  text-decoration: none;
}

.auth-switch a:hover {
  text-decoration: underline;
}

/* === Boards Page === */

.boards-page {
  max-width: 900px;
  margin: 0 auto;
}

.boards-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 24px;
}

.boards-header h1 {
  font-size: 1.5rem;
}

.boards-create-btn {
  padding: 8px 16px;
  border: none;
  background-color: #6366f1;
  color: #fff;
  border-radius: 8px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.boards-create-btn:hover {
  background-color: #4f46e5;
}

.boards-create-form {
  display: flex;
  gap: 8px;
  margin-bottom: 24px;
}

.boards-create-input {
  flex: 1;
  padding: 10px 12px;
  border: 1px solid #333;
  background-color: #1a1a2e;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.95rem;
}

.boards-create-input:focus {
  outline: none;
  border-color: #6366f1;
}

.boards-create-submit {
  padding: 10px 20px;
  border: none;
  background-color: #6366f1;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
}

.boards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 16px;
}

.board-card {
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 20px;
  cursor: pointer;
  transition: border-color 0.2s, transform 0.2s;
}

.board-card:hover {
  border-color: #6366f1;
  transform: translateY(-2px);
}

.board-card-name {
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 4px;
}

.board-card-desc {
  font-size: 0.85rem;
  color: #888;
}

.boards-loading {
  text-align: center;
  padding: 60px;
  color: #888;
}

/* === Board View === */

.board-view {
  height: calc(100vh - 80px);
  display: flex;
  flex-direction: column;
}

.board-view-title {
  font-size: 1.4rem;
  margin-bottom: 16px;
}

.board-lists {
  display: flex;
  gap: 16px;
  overflow-x: auto;
  flex: 1;
  padding-bottom: 16px;
  align-items: flex-start;
}

.board-loading {
  text-align: center;
  padding: 60px;
  color: #888;
}

/* === List Column === */

.list-column {
  flex-shrink: 0;
  width: 280px;
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  max-height: calc(100vh - 160px);
}

.list-column--add {
  background: transparent;
  border: 1px dashed #333;
}

.list-column-header {
  font-size: 0.95rem;
  font-weight: 600;
  padding: 4px 4px 12px;
  color: #ccc;
}

.list-cards {
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow-y: auto;
  flex: 1;
}

.add-list-input {
  width: 100%;
  padding: 10px;
  border: none;
  background: transparent;
  color: #888;
  font-size: 0.9rem;
}

.add-list-input:focus {
  outline: none;
  color: #e0e0e0;
}

/* === Card Item === */

.card-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  background-color: #22223a;
  border: 1px solid #2a2a4a;
  border-radius: 8px;
  cursor: pointer;
  transition: border-color 0.2s, background-color 0.2s;
}

.card-item:hover {
  border-color: #6366f1;
  background-color: #2a2a4a;
}

.card-item-title {
  font-size: 0.9rem;
  color: #e0e0e0;
}

.card-item-assignee {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background-color: #6366f1;
  color: #fff;
  font-size: 0.75rem;
  font-weight: 700;
  flex-shrink: 0;
}

/* === Add Card === */

.add-card-trigger {
  padding: 8px;
  border: none;
  background: none;
  color: #888;
  font-size: 0.85rem;
  cursor: pointer;
  text-align: left;
  border-radius: 6px;
  transition: background-color 0.2s;
}

.add-card-trigger:hover {
  background-color: #22223a;
  color: #e0e0e0;
}

.add-card-form {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.add-card-input {
  padding: 8px 10px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.85rem;
}

.add-card-input:focus {
  outline: none;
  border-color: #6366f1;
}

.add-card-actions {
  display: flex;
  gap: 8px;
}

.add-card-submit {
  padding: 6px 14px;
  border: none;
  background-color: #6366f1;
  color: #fff;
  border-radius: 6px;
  font-size: 0.8rem;
  cursor: pointer;
}

.add-card-cancel {
  padding: 6px 14px;
  border: none;
  background: none;
  color: #888;
  font-size: 0.8rem;
  cursor: pointer;
}

/* === Modal === */

.modal-backdrop {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.modal-content {
  position: relative;
  background-color: #1a1a2e;
  border: 1px solid #2a2a4a;
  border-radius: 12px;
  padding: 24px;
  width: 90%;
  max-width: 560px;
  max-height: 80vh;
  overflow-y: auto;
}

.modal-close {
  position: absolute;
  top: 12px;
  right: 12px;
  background: none;
  border: none;
  color: #888;
  font-size: 1.1rem;
  cursor: pointer;
}

.modal-close:hover {
  color: #e0e0e0;
}

/* === Card Detail Modal === */

.card-detail-loading {
  text-align: center;
  padding: 40px;
  color: #888;
}

.card-detail-title {
  font-size: 1.3rem;
  font-weight: 700;
  margin-bottom: 20px;
}

.card-detail-section {
  margin-bottom: 20px;
}

.card-detail-section h3 {
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #888;
  margin-bottom: 8px;
}

.card-detail-desc {
  font-size: 0.9rem;
  color: #ccc;
  padding: 8px;
  border-radius: 6px;
  cursor: pointer;
  min-height: 40px;
}

.card-detail-desc:hover {
  background-color: #22223a;
}

.card-detail-textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.9rem;
  font-family: inherit;
  resize: vertical;
}

.card-detail-textarea:focus {
  outline: none;
  border-color: #6366f1;
}

.card-detail-desc-actions {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.card-detail-desc-actions button {
  padding: 6px 14px;
  border: none;
  border-radius: 6px;
  font-size: 0.8rem;
  cursor: pointer;
}

.card-detail-desc-actions button:first-child {
  background-color: #6366f1;
  color: #fff;
}

.card-detail-desc-actions button:last-child {
  background: none;
  color: #888;
}

.card-detail-select {
  padding: 8px 12px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.9rem;
  width: 100%;
}

.card-detail-select:focus {
  outline: none;
  border-color: #6366f1;
}

.card-detail-delete {
  width: 100%;
  padding: 10px;
  border: 1px solid #ef4444;
  background: none;
  color: #ef4444;
  border-radius: 8px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 8px;
}

.card-detail-delete:hover {
  background-color: #ef4444;
  color: #fff;
}

/* === Comments === */

.card-detail-comments {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 12px;
}

.comment {
  padding: 10px;
  background-color: #22223a;
  border-radius: 8px;
}

.comment-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}

.comment-header strong {
  font-size: 0.85rem;
  color: #6366f1;
}

.comment-delete {
  background: none;
  border: none;
  color: #666;
  cursor: pointer;
  font-size: 0.8rem;
}

.comment-delete:hover {
  color: #ef4444;
}

.comment-content {
  font-size: 0.85rem;
  color: #ccc;
}

.comment-form {
  display: flex;
  gap: 8px;
}

.comment-input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #333;
  background-color: #0f0f1a;
  color: #e0e0e0;
  border-radius: 6px;
  font-size: 0.85rem;
}

.comment-input:focus {
  outline: none;
  border-color: #6366f1;
}

.comment-submit {
  padding: 8px 16px;
  border: none;
  background-color: #6366f1;
  color: #fff;
  border-radius: 6px;
  font-size: 0.85rem;
  cursor: pointer;
}

/* === Responsive === */

@media (max-width: 768px) {
  .boards-grid {
    grid-template-columns: 1fr;
  }

  .board-lists {
    gap: 12px;
  }

  .list-column {
    width: 260px;
  }
}
```

---

## Step 12: Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
cd ..
git add .
git commit -m "feat: add frontend with Redux slices, auth pages, board view, and card detail modal"
```

---

## Step 13: Test It

Start both servers:

**Terminal 1:**

```bash
cd server && npm run dev
```

**Terminal 2:**

```bash
cd client && npm run dev
```

Open `http://localhost:5173`. You should see the login page. Register a new account. You'll be redirected to the boards list. Create a board, click into it, add lists, add cards, click a card to open the detail modal, add comments, assign a member.

If anything doesn't render, open the browser console and check for errors. Common issues:

| Problem | Fix |
|---------|-----|
| "Failed to fetch" | Is the backend running on port 3001? |
| Redirected to /login after refresh | Token is in localStorage but `fetchCurrentUser` fails — check JWT_SECRET matches |
| Cards don't appear after creating | Check that `createCard.fulfilled` adds to the correct list by `list_id` |
| Modal shows "Loading card..." forever | The GET `/api/cards/:id` endpoint might not be hit — check the API URL |

---

> [!TIP]
> ## Spatial Check-In

1. **Why does the auth slice store the token in BOTH Redux state and localStorage?**

<details><summary>Answer</summary>

Redux state is reactive — components re-render when it changes. But Redux state resets on page refresh. localStorage persists across refreshes but isn't reactive. By storing in both, the app gets reactivity (components see `selectIsAuthenticated` change) AND persistence (refreshing the page doesn't log you out). On load, `initialState.token` reads from localStorage to restore the session.

</details>

2. **Why does `moveCardOptimistic` exist as a separate reducer from the `moveCardAsync` thunk?**

<details><summary>Answer</summary>

Optimistic updates need to happen synchronously — the UI must update the instant the user drops a card. Thunks are asynchronous (they wait for the server). So the drag handler dispatches `moveCardOptimistic` immediately (synchronous reducer, instant UI update), then dispatches `moveCardAsync` (async thunk, sends to server). If the server rejects it, the component refetches the board to restore truth.

</details>

3. **Why does `ProtectedRoute` dispatch `fetchCurrentUser` instead of just checking for a token?**

<details><summary>Answer</summary>

A token can be expired or revoked. Checking `!!token` only confirms localStorage has a string — not that it's valid. `fetchCurrentUser` hits `GET /api/auth/me` with that token. If the server returns 401, the `rejected` handler clears the token and user, which triggers a redirect to `/login`. This catches stale tokens that localStorage alone would trust.

</details>

---

| | | |
|:---|:---:|---:|
| [← 04 — Backend](../04-backend/) | [Level 5 Overview](../) | [06 — Drag & Drop →](../06-drag-and-drop/) |
