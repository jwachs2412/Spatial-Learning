`Level 3` **Step 6 of 8** — Building the Frontend

# 06 — Building the Frontend: Auth Pages, Routing, and Notes UI

## Spatial Orientation

```
vault-note/
├── client/              ← ★ WE ARE HERE ★
│   └── src/
│       ├── context/           ← Auth state management (we'll build this)
│       │   └── AuthContext.tsx
│       ├── components/        ← Reusable UI pieces (we'll build these)
│       │   ├── ProtectedRoute.tsx
│       │   ├── NoteForm.tsx
│       │   ├── NoteList.tsx
│       │   └── NoteEditor.tsx
│       ├── pages/             ← Route-level pages (we'll build these)
│       │   ├── LoginPage.tsx
│       │   ├── RegisterPage.tsx
│       │   └── NotesPage.tsx
│       ├── services/          ← Backend communication (we'll build this)
│       │   └── api.ts
│       ├── types/             ← Shared types (we'll build this)
│       │   └── index.ts
│       ├── App.tsx            ← Router setup (we'll rewrite this)
│       ├── App.css            ← Styles (we'll rewrite this)
│       └── main.tsx           ← Entry point (we'll update this)
└── server/              ← Already built ✓
```

**What's new vs Level 2 frontend?**
- `context/` — React Context for auth state (token + user, shared across all components)
- `pages/` — page-level components (one per route)
- `ProtectedRoute` — redirects unauthenticated users to login
- React Router for multi-page navigation

---

## Step 1: Define Shared Types

**Where?** `client/src/types/index.ts`

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
mkdir -p client/src/types
mkdir -p client/src/services
mkdir -p client/src/components
mkdir -p client/src/pages
mkdir -p client/src/context
```

Create `client/src/types/index.ts`:

```typescript
export interface User {
  id: number;
  email: string;
  role: string;
  created_at: string;
}

export interface Note {
  id: number;
  user_id: number;
  title: string;
  content: string;
  created_at: string;
  updated_at: string;
}

export interface AuthResponse {
  token: string;
  user: User;
}

export interface CreateNoteRequest {
  title: string;
  content?: string;
}

export interface UpdateNoteRequest {
  title?: string;
  content?: string;
}
```

---

## Step 2: Build the API Service

**Where?** `client/src/services/api.ts`

This file is similar to Level 2's, but now includes auth endpoints and sends the JWT token with every protected request.

```typescript
import { AuthResponse, Note, CreateNoteRequest, UpdateNoteRequest } from '../types';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

// Helper: get stored token
function getToken(): string | null {
  return localStorage.getItem('token');
}

// Helper: create headers with auth token
function authHeaders(): Record<string, string> {
  const token = getToken();
  return {
    'Content-Type': 'application/json',
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
  };
}

// ─── AUTH API ───────────────────────────────────────────────────

export async function register(email: string, password: string): Promise<AuthResponse> {
  const response = await fetch(`${API_URL}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Registration failed');
  }
  return response.json();
}

export async function login(email: string, password: string): Promise<AuthResponse> {
  const response = await fetch(`${API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Login failed');
  }
  return response.json();
}

export async function getMe(): Promise<AuthResponse['user']> {
  const response = await fetch(`${API_URL}/auth/me`, {
    headers: authHeaders(),
  });
  if (!response.ok) throw new Error('Not authenticated');
  return response.json();
}

// ─── NOTES API ──────────────────────────────────────────────────

export async function getNotes(): Promise<Note[]> {
  const response = await fetch(`${API_URL}/notes`, {
    headers: authHeaders(),
  });
  if (!response.ok) throw new Error('Failed to fetch notes');
  return response.json();
}

export async function getNote(id: number): Promise<Note> {
  const response = await fetch(`${API_URL}/notes/${id}`, {
    headers: authHeaders(),
  });
  if (!response.ok) throw new Error('Failed to fetch note');
  return response.json();
}

export async function createNote(data: CreateNoteRequest): Promise<Note> {
  const response = await fetch(`${API_URL}/notes`, {
    method: 'POST',
    headers: authHeaders(),
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to create note');
  }
  return response.json();
}

export async function updateNote(id: number, data: UpdateNoteRequest): Promise<Note> {
  const response = await fetch(`${API_URL}/notes/${id}`, {
    method: 'PUT',
    headers: authHeaders(),
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to update note');
  }
  return response.json();
}

export async function deleteNote(id: number): Promise<void> {
  const response = await fetch(`${API_URL}/notes/${id}`, {
    method: 'DELETE',
    headers: authHeaders(),
  });
  if (!response.ok) throw new Error('Failed to delete note');
}
```

### Key Pattern: `authHeaders()`

Every protected request includes `Authorization: Bearer <token>`. The `authHeaders()` helper reads the token from localStorage and builds the headers object. If there's no token, the header is omitted and the backend returns 401.

---

## Step 3: Build the Auth Context

**Where?** `client/src/context/AuthContext.tsx`

In Levels 1 and 2, all state lived in App. That still works, but auth state is special — it's needed by almost every component (the nav bar, the route guard, the API service). React Context lets us share auth state without passing it through every layer of props.

> [!NOTE]
> **Technical**: React Context provides a way to pass data through the component tree without prop drilling. A Provider component at the top makes data available to any descendant via the useContext hook.

> [!NOTE]
> **Plain English**: Instead of passing `user` and `token` as props from App → Page → Component → SubComponent, we put them in a "shared space" (context) that any component can access directly. It's like a global variable, but managed by React.

```tsx
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import { User } from '../types';
import * as api from '../services/api';

interface AuthContextType {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(localStorage.getItem('token'));
  const [isLoading, setIsLoading] = useState(true);

  // On mount, check if we have a stored token and verify it
  useEffect(() => {
    if (token) {
      api.getMe()
        .then((userData) => {
          setUser(userData);
        })
        .catch(() => {
          // Token is invalid or expired — clear it
          localStorage.removeItem('token');
          setToken(null);
          setUser(null);
        })
        .finally(() => {
          setIsLoading(false);
        });
    } else {
      setIsLoading(false);
    }
  }, []);

  async function handleLogin(email: string, password: string) {
    const response = await api.login(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  }

  async function handleRegister(email: string, password: string) {
    const response = await api.register(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  }

  function handleLogout() {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  }

  return (
    <AuthContext.Provider
      value={{
        user,
        token,
        isLoading,
        login: handleLogin,
        register: handleRegister,
        logout: handleLogout,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook for consuming auth context
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### How Auth State Flows

```
AuthProvider (wraps entire app)
│
├── Stores: user, token, isLoading
├── Provides: login(), register(), logout()
│
├── On mount: checks localStorage for existing token
│   → If found: calls GET /api/auth/me to verify
│   → If valid: sets user state (user stays logged in across refreshes)
│   → If invalid: clears token (forces re-login)
│
└── Any component can call useAuth():
    const { user, login, logout } = useAuth();
```

---

> [!TIP]
> **Session Break** — You've built the API service layer and Auth Context with token management. Save your work and take a break.
> When you return, you'll build the protected route component and auth pages.

---

## Step 4: Build the ProtectedRoute Component

**Where?** `client/src/components/ProtectedRoute.tsx`

This component wraps pages that require authentication. If the user isn't logged in, it redirects to `/login`.

```tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

interface ProtectedRouteProps {
  children: React.ReactNode;
}

export function ProtectedRoute({ children }: ProtectedRouteProps) {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <p className="loading">Loading...</p>;
  }

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  return <>{children}</>;
}
```

**`<Navigate to="/login" replace />`** — React Router's redirect component. `replace` means it replaces the current entry in browser history, so pressing "back" doesn't bring you back to the protected page.

---

## Step 5: Build the Login Page

**Where?** `client/src/pages/LoginPage.tsx`

```tsx
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setIsSubmitting(true);

    try {
      await login(email, password);
      navigate('/notes');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="auth-page">
      <form onSubmit={handleSubmit} className="auth-form">
        <h2>Log In</h2>
        <div className="form-group">
          <label htmlFor="email">Email</label>
          <input
            id="email"
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="you@example.com"
            required
          />
        </div>
        <div className="form-group">
          <label htmlFor="password">Password</label>
          <input
            id="password"
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="••••••••"
            required
          />
        </div>
        {error && <p className="error-message">{error}</p>}
        <button type="submit" className="submit-button" disabled={isSubmitting}>
          {isSubmitting ? 'Logging in...' : 'Log In'}
        </button>
        <p className="auth-link">
          Don't have an account? <Link to="/register">Register</Link>
        </p>
      </form>
    </div>
  );
}
```

### New React Router Concepts

**`useNavigate()`** — Returns a function you can call to navigate programmatically: `navigate('/notes')` sends the user to the notes page after login.

**`<Link to="/register">`** — Like an `<a>` tag but without a full page reload. React Router intercepts the click and renders the register page component.

---

## Step 6: Build the Register Page

**Where?** `client/src/pages/RegisterPage.tsx`

```tsx
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function RegisterPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [error, setError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const { register } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    if (password !== confirmPassword) {
      setError('Passwords do not match');
      return;
    }

    if (password.length < 8) {
      setError('Password must be at least 8 characters');
      return;
    }

    setIsSubmitting(true);

    try {
      await register(email, password);
      navigate('/notes');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Registration failed');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="auth-page">
      <form onSubmit={handleSubmit} className="auth-form">
        <h2>Create Account</h2>
        <div className="form-group">
          <label htmlFor="email">Email</label>
          <input
            id="email"
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="you@example.com"
            required
          />
        </div>
        <div className="form-group">
          <label htmlFor="password">Password</label>
          <input
            id="password"
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="At least 8 characters"
            required
            minLength={8}
          />
        </div>
        <div className="form-group">
          <label htmlFor="confirmPassword">Confirm Password</label>
          <input
            id="confirmPassword"
            type="password"
            value={confirmPassword}
            onChange={(e) => setConfirmPassword(e.target.value)}
            placeholder="Type password again"
            required
          />
        </div>
        {error && <p className="error-message">{error}</p>}
        <button type="submit" className="submit-button" disabled={isSubmitting}>
          {isSubmitting ? 'Creating account...' : 'Create Account'}
        </button>
        <p className="auth-link">
          Already have an account? <Link to="/login">Log in</Link>
        </p>
      </form>
    </div>
  );
}
```

---

> [!TIP]
> **Session Break** — You've built the Login and Register pages with form handling and error display. Save your work and take a break.
> When you return, you'll build the Notes page with full CRUD functionality.

---

## Step 7: Build the Notes Page

**Where?** `client/src/pages/NotesPage.tsx`

This is the main protected page. It shows the user's notes with full CRUD — create, read, update, delete.

```tsx
import { useState, useEffect } from 'react';
import { Note } from '../types';
import * as api from '../services/api';
import { useAuth } from '../context/AuthContext';

export function NotesPage() {
  const { user, logout } = useAuth();
  const [notes, setNotes] = useState<Note[]>([]);
  const [selectedNote, setSelectedNote] = useState<Note | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Form state
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [isEditing, setIsEditing] = useState(false);

  useEffect(() => {
    loadNotes();
  }, []);

  async function loadNotes() {
    try {
      setIsLoading(true);
      const data = await api.getNotes();
      setNotes(data);
    } catch (err) {
      console.error('Failed to load notes:', err);
    } finally {
      setIsLoading(false);
    }
  }

  async function handleCreate(e: React.FormEvent) {
    e.preventDefault();
    if (title.trim().length === 0) return;

    try {
      const newNote = await api.createNote({ title: title.trim(), content: content.trim() });
      setNotes((prev) => [newNote, ...prev]);
      setTitle('');
      setContent('');
    } catch (err) {
      console.error('Failed to create note:', err);
    }
  }

  function handleSelect(note: Note) {
    setSelectedNote(note);
    setTitle(note.title);
    setContent(note.content);
    setIsEditing(true);
  }

  async function handleUpdate(e: React.FormEvent) {
    e.preventDefault();
    if (!selectedNote || title.trim().length === 0) return;

    try {
      const updated = await api.updateNote(selectedNote.id, {
        title: title.trim(),
        content: content.trim(),
      });
      setNotes((prev) => prev.map((n) => (n.id === updated.id ? updated : n)));
      setSelectedNote(null);
      setTitle('');
      setContent('');
      setIsEditing(false);
    } catch (err) {
      console.error('Failed to update note:', err);
    }
  }

  async function handleDelete(id: number) {
    if (!confirm('Delete this note?')) return;

    try {
      await api.deleteNote(id);
      setNotes((prev) => prev.filter((n) => n.id !== id));
      if (selectedNote?.id === id) {
        setSelectedNote(null);
        setTitle('');
        setContent('');
        setIsEditing(false);
      }
    } catch (err) {
      console.error('Failed to delete note:', err);
    }
  }

  function handleCancel() {
    setSelectedNote(null);
    setTitle('');
    setContent('');
    setIsEditing(false);
  }

  return (
    <div className="notes-page">
      <header className="notes-header">
        <div>
          <h1>VaultNote</h1>
          <p>{user?.email}</p>
        </div>
        <button className="logout-button" onClick={logout}>
          Log Out
        </button>
      </header>

      <div className="notes-layout">
        <section className="notes-sidebar">
          <h2>Your Notes</h2>
          {isLoading ? (
            <p className="loading">Loading...</p>
          ) : notes.length === 0 ? (
            <p className="empty-state">No notes yet. Create one!</p>
          ) : (
            <div className="note-list">
              {notes.map((note) => (
                <div
                  key={note.id}
                  className={`note-card ${selectedNote?.id === note.id ? 'active' : ''}`}
                  onClick={() => handleSelect(note)}
                >
                  <div className="note-card-content">
                    <h3>{note.title}</h3>
                    <p>{note.content.substring(0, 80)}{note.content.length > 80 ? '...' : ''}</p>
                    <time>
                      {new Date(note.updated_at).toLocaleDateString('en-US', {
                        month: 'short',
                        day: 'numeric',
                      })}
                    </time>
                  </div>
                  <button
                    className="delete-button"
                    onClick={(e) => {
                      e.stopPropagation();
                      handleDelete(note.id);
                    }}
                    title="Delete note"
                  >
                    ✕
                  </button>
                </div>
              ))}
            </div>
          )}
        </section>

        <section className="notes-editor">
          <form onSubmit={isEditing ? handleUpdate : handleCreate}>
            <input
              type="text"
              value={title}
              onChange={(e) => setTitle(e.target.value)}
              placeholder="Note title"
              className="note-title-input"
              maxLength={200}
            />
            <textarea
              value={content}
              onChange={(e) => setContent(e.target.value)}
              placeholder="Start writing..."
              className="note-content-input"
              rows={12}
            />
            <div className="editor-actions">
              {isEditing && (
                <button type="button" className="cancel-button" onClick={handleCancel}>
                  Cancel
                </button>
              )}
              <button
                type="submit"
                className="submit-button"
                disabled={title.trim().length === 0}
              >
                {isEditing ? 'Save Changes' : 'Create Note'}
              </button>
            </div>
          </form>
        </section>
      </div>
    </div>
  );
}
```

---

> [!TIP]
> **Session Break** — You've built the Notes page with create, edit, and delete functionality. Save your work and take a break.
> When you return, you'll wire everything together with App routing and add styles.

---

## Step 8: Build the App Component with Routing

**Where?** `client/src/App.tsx`

Open `client/src/App.tsx` and replace its contents:

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { ProtectedRoute } from './components/ProtectedRoute';
import { LoginPage } from './pages/LoginPage';
import { RegisterPage } from './pages/RegisterPage';
import { NotesPage } from './pages/NotesPage';
import './App.css';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/register" element={<RegisterPage />} />
          <Route
            path="/notes"
            element={
              <ProtectedRoute>
                <NotesPage />
              </ProtectedRoute>
            }
          />
          <Route path="*" element={<Navigate to="/notes" replace />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

### React Router Explained

```
URL:          Component:         Protected?
──────        ──────────         ──────────
/login        LoginPage          No
/register     RegisterPage       No
/notes        NotesPage          Yes (ProtectedRoute)
/*            → redirect /notes  —
```

**`<BrowserRouter>`** — Wraps the app to enable routing. Uses the browser's History API.

**`<Routes>`** — Container for route definitions. Only the first matching route renders.

**`<Route path="/login" element={<LoginPage />} />`** — When the URL is `/login`, render `LoginPage`.

**`<Route path="*">`** — Catch-all for any URL not matched above. Redirects to `/notes`.

**`<ProtectedRoute>`** — Wraps `NotesPage`. If the user isn't logged in, redirects to `/login`. If they are, renders the page.

---

## Step 9: Add Styles

Open `client/src/App.css` and replace its contents:

```css
/* ─── GLOBAL RESET ───────────────────────────────────────── */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    Oxygen, Ubuntu, sans-serif;
  background-color: #0f172a;
  color: #e2e8f0;
  line-height: 1.6;
}

/* ─── AUTH PAGES ─────────────────────────────────────────── */
.auth-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.auth-form {
  background: #1e293b;
  border-radius: 12px;
  padding: 2rem;
  width: 100%;
  max-width: 400px;
}

.auth-form h2 {
  text-align: center;
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  color: #38bdf8;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  color: #94a3b8;
  font-size: 0.9rem;
}

.form-group input {
  width: 100%;
  padding: 0.75rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
}

.form-group input:focus {
  outline: none;
  border-color: #38bdf8;
}

.submit-button {
  width: 100%;
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  background: #38bdf8;
  color: #0f172a;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  margin-top: 0.5rem;
}

.submit-button:hover {
  background: #7dd3fc;
}

.submit-button:disabled {
  background: #334155;
  color: #64748b;
  cursor: not-allowed;
}

.auth-link {
  text-align: center;
  margin-top: 1rem;
  color: #94a3b8;
  font-size: 0.9rem;
}

.auth-link a {
  color: #38bdf8;
  text-decoration: none;
}

.auth-link a:hover {
  text-decoration: underline;
}

/* ─── NOTES PAGE ─────────────────────────────────────────── */
.notes-page {
  max-width: 1000px;
  margin: 0 auto;
  padding: 1rem;
  min-height: 100vh;
}

.notes-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #334155;
}

.notes-header h1 {
  font-size: 1.5rem;
  color: #38bdf8;
}

.notes-header p {
  color: #94a3b8;
  font-size: 0.85rem;
}

.logout-button {
  background: none;
  border: 1px solid #475569;
  color: #94a3b8;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
}

.logout-button:hover {
  border-color: #f87171;
  color: #f87171;
}

/* ─── NOTES LAYOUT ───────────────────────────────────────── */
.notes-layout {
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 1.5rem;
}

@media (max-width: 768px) {
  .notes-layout {
    grid-template-columns: 1fr;
  }
}

/* ─── NOTES SIDEBAR ──────────────────────────────────────── */
.notes-sidebar h2 {
  font-size: 1.1rem;
  margin-bottom: 0.75rem;
}

.note-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.note-card {
  background: #1e293b;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  border: 2px solid transparent;
  transition: border-color 0.2s;
}

.note-card:hover {
  border-color: #334155;
}

.note-card.active {
  border-color: #38bdf8;
}

.note-card-content {
  flex: 1;
  overflow: hidden;
}

.note-card h3 {
  font-size: 0.95rem;
  margin-bottom: 0.15rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.note-card p {
  color: #94a3b8;
  font-size: 0.8rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.note-card time {
  font-size: 0.75rem;
  color: #64748b;
}

/* ─── NOTES EDITOR ───────────────────────────────────────── */
.notes-editor form {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
}

.note-title-input {
  width: 100%;
  padding: 0.75rem;
  border: none;
  border-bottom: 2px solid #334155;
  border-radius: 0;
  background: transparent;
  color: #e2e8f0;
  font-size: 1.25rem;
  font-weight: 600;
  font-family: inherit;
  margin-bottom: 1rem;
}

.note-title-input:focus {
  outline: none;
  border-bottom-color: #38bdf8;
}

.note-content-input {
  width: 100%;
  padding: 0.75rem;
  border: none;
  background: transparent;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
  resize: vertical;
  min-height: 200px;
  line-height: 1.6;
}

.note-content-input:focus {
  outline: none;
}

.editor-actions {
  display: flex;
  gap: 0.5rem;
  justify-content: flex-end;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #334155;
}

.cancel-button {
  padding: 0.5rem 1rem;
  border: 1px solid #475569;
  border-radius: 8px;
  background: none;
  color: #94a3b8;
  cursor: pointer;
  font-size: 0.9rem;
}

.cancel-button:hover {
  border-color: #94a3b8;
  color: #e2e8f0;
}

.editor-actions .submit-button {
  width: auto;
  padding: 0.5rem 1.5rem;
}

/* ─── DELETE BUTTON ──────────────────────────────────────── */
.delete-button {
  background: none;
  border: none;
  color: #64748b;
  font-size: 1rem;
  cursor: pointer;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  line-height: 1;
  flex-shrink: 0;
}

.delete-button:hover {
  color: #f87171;
  background: rgba(248, 113, 113, 0.1);
}

/* ─── UTILITY ────────────────────────────────────────────── */
.loading {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

.empty-state {
  color: #64748b;
  font-size: 0.9rem;
  padding: 1rem 0;
}

.error-message {
  color: #f87171;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}
```

---

## Step 10: Clean Up Boilerplate and Update Entry Point

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
rm client/src/assets/react.svg
rm client/public/vite.svg
```

Open `client/index.html` and replace:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>VaultNote — Secure Personal Notes</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Step 11: Commit

> [!IMPORTANT]
> **You should be in:** `vault-note/`

```bash
git add client/
git commit -m "feat: add React frontend with auth, routing, and notes UI"
```

---

> [!TIP]
> ## Spatial Check-In

1. **What does React Context solve that props don't?**

<details><summary>Answer</summary>

Context avoids prop drilling — passing data through every intermediate component. Auth state (user, token) is needed by many components at different depths. Context makes it accessible to any component without passing it through the tree.

</details>

2. **What does the ProtectedRoute component do?**

<details><summary>Answer</summary>

It checks if the user is authenticated (via useAuth). If yes, it renders the child page. If no, it redirects to `/login`. It prevents unauthenticated users from seeing protected pages.

</details>

3. **Where is the JWT token stored, and how is it sent with requests?**

<details><summary>Answer</summary>

Stored in localStorage. Sent via the `Authorization: Bearer <token>` header on every request to protected endpoints. The `authHeaders()` helper in api.ts handles this.

</details>

4. **What happens when a user refreshes the page?**

<details><summary>Answer</summary>

The AuthProvider reads the token from localStorage and calls `GET /api/auth/me` to verify it. If valid, the user stays logged in. If invalid or expired, the token is cleared and the user is redirected to login.

</details>

---

| | | |
|:---|:---:|---:|
| [← 05 — Building the Backend](../05-backend/) | [Level 3 Overview](../) | [07 — Deployment →](../07-deployment/) |
