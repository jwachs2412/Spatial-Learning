# Step 6 — Building the Frontend: Auth Pages, Routing, and Notes UI

## What's New on the Frontend in Level 3

JSX, hooks, controlled inputs, state management, and the generic `request<T>` helper all carry over from Level 1 and Level 2. If any of those feel unfamiliar, revisit Level 1 Step 4's React primer and Level 2 Step 5's `request<T>` walkthrough before continuing.

The new building blocks unique to Level 3:

- **`localStorage`** — a tiny key/value store the browser persists across page refreshes. `localStorage.setItem('token', value)` saves; `localStorage.getItem('token')` reads; `localStorage.removeItem('token')` clears. Strings only — for objects, JSON.stringify in / JSON.parse out.
- **Bearer token header** — every authenticated API request includes `Authorization: Bearer <token>`. We build it once in a helper and spread it into every `fetch` call.
- **React Context** — a way to share data across components without passing props through every level. Three pieces: `createContext(...)` to declare the shape, a `Provider` component that holds the state, and `useContext(...)` (or a custom hook) to read it.
- **Custom hooks** — functions whose name starts with `use` that wrap a hook (or several) for reuse. `useAuth()` is one we'll write; calling it from any component gives us the auth state.
- **React Router** — URL-based navigation. `<BrowserRouter>` (top-level), `<Routes>` and `<Route path="..." element={...}>` (declare paths), `<Link to="...">` (navigation links), `useNavigate()` (programmatic navigation), `<Navigate to="...">` (redirect during render).
- **`ReactNode` type** — a TypeScript type for "anything React can render": JSX elements, strings, numbers, fragments, arrays of these. Used when you accept arbitrary children as a prop.

Every code block below has a "Reading This File Line-by-Line" walkthrough using these.

## Spatial Orientation

```
vault-note/
└── client/
    └── src/
        ├── types/index.ts           ← Shared types
        ├── services/api.ts          ← API calls with auth headers
        ├── context/AuthContext.tsx   ← Shared auth state
        ├── components/
        │   └── ProtectedRoute.tsx   ← Route guard
        ├── pages/
        │   ├── LoginPage.tsx
        │   ├── RegisterPage.tsx
        │   └── NotesPage.tsx
        └── App.tsx                  ← Router setup
```

---

## 1. Types

<details>
<summary>See the code — `client/src/types/index.ts`</summary>

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

</details>

---

## 2. API Service with Auth Headers

> **Key Concept: Bearer Token Pattern**
> Every authenticated request includes the JWT in the `Authorization` header: `Authorization: Bearer <token>`. The API service reads the token from localStorage and attaches it to every request automatically.

<details>
<summary>See the code — `client/src/services/api.ts`</summary>

```typescript
import { User, Note, AuthResponse, CreateNoteRequest, UpdateNoteRequest } from '../types';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

function getToken(): string | null {
  return localStorage.getItem('token');
}

function authHeaders(): Record<string, string> {
  const token = getToken();
  return token ? { Authorization: `Bearer ${token}` } : {};
}

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${API_URL}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...authHeaders(),
      ...options?.headers,
    },
  });

  if (!response.ok) {
    const data = await response.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(data.error || `Request failed: ${response.status}`);
  }

  if (response.status === 204) return undefined as T;
  return response.json();
}

// Auth
export const register = (email: string, password: string) =>
  request<AuthResponse>('/auth/register', { method: 'POST', body: JSON.stringify({ email, password }) });
export const login = (email: string, password: string) =>
  request<AuthResponse>('/auth/login', { method: 'POST', body: JSON.stringify({ email, password }) });
export const getMe = () => request<User>('/auth/me');

// Notes
export const getNotes = () => request<Note[]>('/notes');
export const createNote = (data: CreateNoteRequest) =>
  request<Note>('/notes', { method: 'POST', body: JSON.stringify(data) });
export const updateNote = (id: number, data: UpdateNoteRequest) =>
  request<Note>(`/notes/${id}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteNote = (id: number) =>
  request<void>(`/notes/${id}`, { method: 'DELETE' });
```

</details>

### Reading This File Line-by-Line

Most of this file is the same `request<T>` generic helper from Level 2. The new bits are the two helpers at the top and the spread of `authHeaders()` into every request.

```typescript
function getToken(): string | null {
  return localStorage.getItem('token');
}
```

- `localStorage` is a built-in browser global — an object with `setItem`, `getItem`, `removeItem` methods. Each value is a string.
- `getItem('token')` returns the stored string, or `null` if no value was set.
- The return type `string | null` documents that null is possible.

```typescript
function authHeaders(): Record<string, string> {
  const token = getToken();
  return token ? { Authorization: `Bearer ${token}` } : {};
}
```

- `Record<string, string>` is a TypeScript utility type meaning "an object with string keys and string values."
- Read the body as: "if we have a token, return an object with one header; otherwise return an empty object."
- The empty-object case is important — calling `fetch` with `headers: { ..., undefined }` would crash. By returning `{}` and spreading it, we contribute nothing to the headers.

```typescript
async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${API_URL}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...authHeaders(),
      ...options?.headers,
    },
  });
```

- Same generic helper from Level 2, with one addition: `...authHeaders()` is spread into the headers object.
- Spread order matters: the default `Content-Type` is first, then `authHeaders()` (so an empty object overwrites nothing, or a token is added), then any caller-provided headers (which can override either if needed).
- Result: every authenticated request automatically includes the JWT. No component or service function has to remember to attach it.

---

## 3. Auth Context

> **Key Concept: React Context**
> Context provides a way to share data across components without passing props through every level. For auth, many components need to know: "Is the user logged in? Who are they?" Context makes this available everywhere.

<details>
<summary>See the code — `client/src/context/AuthContext.tsx`</summary>

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

  // On mount, verify existing token
  useEffect(() => {
    if (token) {
      api.getMe()
        .then(setUser)
        .catch(() => {
          // Token is invalid/expired — clear it
          localStorage.removeItem('token');
          setToken(null);
        })
        .finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, [token]);

  const login = async (email: string, password: string) => {
    const response = await api.login(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  };

  const register = async (email: string, password: string) => {
    const response = await api.register(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, isLoading, login, register, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within an AuthProvider');
  return context;
}
```

</details>

### Reading This File Line-by-Line

This is the densest new pattern in Level 3. We'll go in three chunks: **declare the context shape**, **build the provider**, **expose the custom hook**.

#### Chunk 1: Declare the Shape

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
```

- `interface AuthContextType { ... }` describes everything we'll share through the context: state (user, token, isLoading) plus action functions (login, register, logout).
- Each function's type is written `(args) => returnType`. So `login: (email: string, password: string) => Promise<void>` reads "a function taking two strings, returning a void promise."
- `createContext<AuthContextType | null>(null)` creates a Context object. The `<...>` type argument says "values flowing through this context will be either AuthContextType or null." We initialize with `null` so consumers can detect "you forgot to wrap the app in an AuthProvider."

#### Chunk 2: Build the Provider

```tsx
export function AuthProvider({ children }: { children: ReactNode }) {
```

- A component named `AuthProvider`. Its only prop is `children`, typed as `ReactNode` (anything renderable).
- Whatever components you nest inside `<AuthProvider>...</AuthProvider>` will be able to read this context.

```tsx
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(localStorage.getItem('token'));
  const [isLoading, setIsLoading] = useState(true);
```

Three state values:

- `user` starts null and gets populated after we verify the token.
- `token` is read from localStorage on first render. If the user has a saved token, we start with it; otherwise null. This is what "remember me across page refreshes" looks like in code.
- `isLoading` is true until we either confirm or reject the saved token.

```tsx
  useEffect(() => {
    if (token) {
      api.getMe()
        .then(setUser)
        .catch(() => {
          localStorage.removeItem('token');
          setToken(null);
        })
        .finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, [token]);
```

- Effect with `[token]` as the dependency: runs once on mount, and again any time `token` changes.
- If we have a token, call `api.getMe()` to verify it. The server will either return the user or 401.
  - `.then(setUser)` — shorthand for `.then((user) => setUser(user))`. Because `setUser` is itself a one-arg function, we can pass it directly.
  - `.catch(() => { ... })` — token rejected (expired, tampered, or revoked). Clear it from localStorage and from state.
  - `.finally(() => setIsLoading(false))` — run regardless of success or failure. Loading is over.
- If we don't have a token, just stop loading. Nothing to verify.

```tsx
  const login = async (email: string, password: string) => {
    const response = await api.login(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  };
```

- Three writes happen in order: persist to localStorage, update token state, update user state.
- `localStorage.setItem(key, value)` saves the string under that key. Survives page refreshes and browser restarts (until cleared or until storage limits are hit).
- Updating `token` triggers the `useEffect` above to re-run, but this time it has a verified user from the response, not a stale token from localStorage.

```tsx
  const register = async (email: string, password: string) => {
    const response = await api.register(email, password);
    localStorage.setItem('token', response.token);
    setToken(response.token);
    setUser(response.user);
  };
```

Same pattern as login. Auto-login after registration.

```tsx
  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };
```

The opposite: clear from storage, reset state.

```tsx
  return (
    <AuthContext.Provider value={{ user, token, isLoading, login, register, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

- `<AuthContext.Provider value={...}>` is how React makes a context value available to descendants. Any component anywhere inside `{children}` can call `useContext(AuthContext)` and get back this object.
- The `value={{...}}` packs all six things into one object. That object is what consumers receive.

#### Chunk 3: The Custom Hook

```tsx
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within an AuthProvider');
  return context;
}
```

- Wraps `useContext(AuthContext)` so consumers don't have to import the context object themselves.
- The null check converts a confusing "context is null" runtime bug into a clear error message: "you forgot to wrap your app in `<AuthProvider>`."
- This is the **custom hook** pattern. Any function whose name starts with `use` that calls other hooks counts as a hook. Component code calls `useAuth()` and gets back the typed object.

---

## 4. ProtectedRoute Component

```tsx
// client/src/components/ProtectedRoute.tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';
import { ReactNode } from 'react';

export function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user, isLoading } = useAuth();

  if (isLoading) return <div className="loading">Loading...</div>;
  if (!user) return <Navigate to="/login" />;

  return <>{children}</>;
}
```

### Reading This File Line-by-Line

A tiny but powerful component. It wraps any page that requires login.

- `useAuth()` returns our context object. We destructure the `user` and `isLoading` fields.
- `if (isLoading) return <div>Loading...</div>;` — while we're verifying the saved token, show a loading state. **Don't redirect during loading** — that would bounce a logged-in user to `/login` for a split-second on every refresh.
- `if (!user) return <Navigate to="/login" />;` — `<Navigate>` is React Router's "redirect during render" component. Returning it from a render function tells the router to navigate.
- `return <>{children}</>;` — passing the auth check, render whatever was wrapped. The `<>...</>` is a React fragment (no DOM wrapper).

Usage at the routing level: `<Route path="/notes" element={<ProtectedRoute><NotesPage /></ProtectedRoute>} />`. The `NotesPage` only mounts when authenticated.

---

## 5. Pages

Build three pages: LoginPage, RegisterPage, and NotesPage. Each one uses `useAuth()` from the context.

### Reading the Pages — Patterns to Notice

The page components reuse familiar patterns (controlled inputs, async submit, try/catch/finally, `setIsSubmitting`). The new Level 3 pieces appear in two places — read these once and the rest is review.

**`useNavigate()`** (from `react-router-dom`):

```tsx
const navigate = useNavigate();
// ...
await login(email, password);
navigate('/notes');
```

- `useNavigate()` returns a function. Call it with a path to navigate programmatically. Used after a successful login to send the user to `/notes`.
- This is the imperative cousin of `<Navigate to="...">`. Use `<Navigate>` inside JSX during render; use `navigate(...)` inside event handlers or effects.

**`<Link to="...">`** (from `react-router-dom`):

```tsx
<Link to="/register">Register</Link>
```

- A drop-in replacement for `<a href="...">`. Looks the same, but doesn't trigger a full page reload when clicked — React Router intercepts and updates the URL without re-fetching from the server. Much faster.
- Always use `<Link>` for in-app navigation. Use plain `<a>` only for external URLs.

**`useAuth()` consumption**:

```tsx
const { login } = useAuth();      // LoginPage destructures just `login`
const { register } = useAuth();   // RegisterPage destructures just `register`
const { user, logout } = useAuth(); // NotesPage destructures user + logout
```

Each page pulls only what it needs. Because `useAuth()` returns the same object whether you destructure one field or six, it's both expressive and lightweight.

Everything else in the pages — `useState` for form fields, `try/catch/finally` for the submit handler, `e.preventDefault()`, controlled inputs — is the exact same pattern from Level 1 Step 4 and Level 2 Step 5. Refer back to those walkthroughs if anything feels unfamiliar.

<details>
<summary>LoginPage</summary>

```tsx
import { useState, FormEvent } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: FormEvent) => {
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
        <h1>Log In to VaultNote</h1>
        {error && <p className="error">{error}</p>}
        <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} required />
        <button type="submit" disabled={isSubmitting}>{isSubmitting ? 'Logging in...' : 'Log In'}</button>
        <p className="auth-link">Don't have an account? <Link to="/register">Register</Link></p>
      </form>
    </div>
  );
}
```

</details>

<details>
<summary>RegisterPage</summary>

```tsx
import { useState, FormEvent } from 'react';
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

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError('');
    if (password.length < 8) { setError('Password must be at least 8 characters'); return; }
    if (password !== confirmPassword) { setError('Passwords do not match'); return; }
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
        <h1>Create Account</h1>
        {error && <p className="error">{error}</p>}
        <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        <input type="password" placeholder="Password (8+ characters)" value={password} onChange={(e) => setPassword(e.target.value)} required />
        <input type="password" placeholder="Confirm Password" value={confirmPassword} onChange={(e) => setConfirmPassword(e.target.value)} required />
        <button type="submit" disabled={isSubmitting}>{isSubmitting ? 'Creating...' : 'Register'}</button>
        <p className="auth-link">Already have an account? <Link to="/login">Log In</Link></p>
      </form>
    </div>
  );
}
```

</details>

<details>
<summary>NotesPage (main protected page)</summary>

```tsx
import { useState, useEffect, FormEvent } from 'react';
import { Note } from '../types';
import * as api from '../services/api';
import { useAuth } from '../context/AuthContext';

export function NotesPage() {
  const { user, logout } = useAuth();
  const [notes, setNotes] = useState<Note[]>([]);
  const [selectedNote, setSelectedNote] = useState<Note | null>(null);
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [error, setError] = useState('');

  useEffect(() => {
    api.getNotes().then(setNotes).catch((err: Error) => setError(err.message));
  }, []);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;
    setError('');
    try {
      if (selectedNote) {
        const updated = await api.updateNote(selectedNote.id, { title: title.trim(), content });
        setNotes((prev) => prev.map((n) => (n.id === updated.id ? updated : n)));
        setSelectedNote(null);
      } else {
        const created = await api.createNote({ title: title.trim(), content });
        setNotes((prev) => [created, ...prev]);
      }
      setTitle('');
      setContent('');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to save note');
    }
  };

  const handleDelete = async (id: number) => {
    if (!confirm('Delete this note?')) return;
    await api.deleteNote(id);
    setNotes((prev) => prev.filter((n) => n.id !== id));
    if (selectedNote?.id === id) { setSelectedNote(null); setTitle(''); setContent(''); }
  };

  const handleSelect = (note: Note) => {
    setSelectedNote(note);
    setTitle(note.title);
    setContent(note.content);
  };

  const handleCancel = () => {
    setSelectedNote(null);
    setTitle('');
    setContent('');
  };

  return (
    <div className="notes-page">
      <header className="notes-header">
        <h1>VaultNote</h1>
        <div className="header-right">
          <span>{user?.email}</span>
          <button onClick={logout} className="logout-btn">Log Out</button>
        </div>
      </header>

      {error && <p className="error">{error}</p>}

      <div className="notes-layout">
        <aside className="notes-sidebar">
          <h2>Your Notes</h2>
          {notes.length === 0 ? (
            <p className="empty-state">No notes yet</p>
          ) : (
            notes.map((note) => (
              <div
                key={note.id}
                className={`note-card ${selectedNote?.id === note.id ? 'active' : ''}`}
                onClick={() => handleSelect(note)}
              >
                <span className="note-title">{note.title}</span>
                <button className="delete-btn" onClick={(e) => { e.stopPropagation(); handleDelete(note.id); }}>×</button>
              </div>
            ))
          )}
        </aside>

        <main className="notes-editor">
          <form onSubmit={handleSubmit}>
            <input type="text" placeholder="Note title" value={title} onChange={(e) => setTitle(e.target.value)} maxLength={200} required />
            <textarea placeholder="Write your note..." value={content} onChange={(e) => setContent(e.target.value)} rows={12} />
            <div className="editor-actions">
              <button type="submit">{selectedNote ? 'Update' : 'Create'} Note</button>
              {selectedNote && <button type="button" onClick={handleCancel} className="cancel-btn">Cancel</button>}
            </div>
          </form>
        </main>
      </div>
    </div>
  );
}
```

</details>

---

## 6. App Component with React Router

<details>
<summary>See the code — `client/src/App.tsx`</summary>

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
          <Route path="/notes" element={<ProtectedRoute><NotesPage /></ProtectedRoute>} />
          <Route path="*" element={<Navigate to="/notes" />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

</details>

### Reading This File Line-by-Line

The whole app is wrapped in two providers and a routing block. Order matters.

```tsx
<BrowserRouter>
  <AuthProvider>
    <Routes>
      ...
    </Routes>
  </AuthProvider>
</BrowserRouter>
```

- `<BrowserRouter>` (outermost) gives the rest of the tree access to React Router's hooks and components. It listens to URL changes.
- `<AuthProvider>` wraps inside the router so any component (including pages) can call `useAuth()`. Everything inside has access to the auth context.
- `<Routes>` is the routing container. Inside, each `<Route>` declares one URL pattern.

```tsx
<Route path="/login" element={<LoginPage />} />
```

- `path` is the URL pattern (`/login`).
- `element` is the JSX to render when this URL matches. Note: it's an actual element (`<LoginPage />`), not a component reference. This is React Router v6 syntax.

```tsx
<Route path="/notes" element={<ProtectedRoute><NotesPage /></ProtectedRoute>} />
```

- `<ProtectedRoute>` wraps `<NotesPage />`. As we saw, ProtectedRoute either renders its children (if authenticated) or redirects to `/login`.
- The wrapper-around-children pattern is how you "decorate" a route with extra logic — you can stack multiple wrappers (auth, role check, layout) by nesting them.

```tsx
<Route path="*" element={<Navigate to="/notes" />} />
```

- `path="*"` is the **wildcard** — matches any URL no other route matched.
- `<Navigate to="/notes" />` redirects there. So an unknown URL like `/foo` sends the user to `/notes`. Combined with ProtectedRoute, that means an unauthenticated user typing any URL ends up at `/login`.

---

## 7. CSS

<details>
<summary>See App.css</summary>

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #0f172a; color: #e2e8f0; line-height: 1.6; }
.loading { text-align: center; padding: 4rem; color: #64748b; }
.error { background: #450a0a; color: #fca5a5; padding: 0.75rem 1rem; border-radius: 8px; margin-bottom: 1rem; }

/* Auth Pages */
.auth-page { display: flex; justify-content: center; align-items: center; min-height: 100vh; padding: 1rem; }
.auth-form { background: #1e293b; padding: 2rem; border-radius: 12px; width: 100%; max-width: 400px; }
.auth-form h1 { text-align: center; margin-bottom: 1.5rem; color: #38bdf8; }
.auth-form input { width: 100%; padding: 0.75rem; margin-bottom: 1rem; border: 1px solid #334155; border-radius: 6px; background: #0f172a; color: #e2e8f0; font-size: 1rem; }
.auth-form input:focus { outline: none; border-color: #38bdf8; }
.auth-form button { width: 100%; padding: 0.75rem; background: #0ea5e9; color: white; border: none; border-radius: 6px; font-size: 1rem; cursor: pointer; }
.auth-form button:hover:not(:disabled) { background: #0284c7; }
.auth-form button:disabled { opacity: 0.5; }
.auth-link { text-align: center; margin-top: 1rem; color: #94a3b8; font-size: 0.9rem; }
.auth-link a { color: #38bdf8; text-decoration: none; }

/* Notes Page */
.notes-page { max-width: 1000px; margin: 0 auto; padding: 1rem; }
.notes-header { display: flex; justify-content: space-between; align-items: center; padding: 1rem 0; border-bottom: 1px solid #1e293b; margin-bottom: 1rem; }
.notes-header h1 { color: #38bdf8; }
.header-right { display: flex; align-items: center; gap: 1rem; }
.header-right span { color: #94a3b8; font-size: 0.9rem; }
.logout-btn { background: transparent; border: 1px solid #334155; color: #94a3b8; padding: 0.4rem 0.8rem; border-radius: 6px; cursor: pointer; }
.logout-btn:hover { border-color: #ef4444; color: #ef4444; }

.notes-layout { display: grid; grid-template-columns: 250px 1fr; gap: 1rem; min-height: 70vh; }
.notes-sidebar { background: #1e293b; border-radius: 8px; padding: 1rem; overflow-y: auto; }
.notes-sidebar h2 { font-size: 1rem; color: #94a3b8; margin-bottom: 0.75rem; }
.note-card { display: flex; justify-content: space-between; align-items: center; padding: 0.5rem; border-radius: 6px; cursor: pointer; margin-bottom: 0.25rem; }
.note-card:hover { background: #334155; }
.note-card.active { background: #0c4a6e; }
.note-title { font-size: 0.9rem; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; flex: 1; }
.delete-btn { background: transparent; border: none; color: #64748b; cursor: pointer; font-size: 1.1rem; }
.delete-btn:hover { color: #ef4444; }
.empty-state { color: #64748b; font-size: 0.85rem; }

.notes-editor { background: #1e293b; border-radius: 8px; padding: 1.5rem; }
.notes-editor input { width: 100%; padding: 0.75rem; margin-bottom: 1rem; border: 1px solid #334155; border-radius: 6px; background: #0f172a; color: #e2e8f0; font-size: 1.1rem; }
.notes-editor textarea { width: 100%; padding: 0.75rem; border: 1px solid #334155; border-radius: 6px; background: #0f172a; color: #e2e8f0; font-family: inherit; font-size: 0.95rem; resize: vertical; margin-bottom: 1rem; }
.notes-editor input:focus, .notes-editor textarea:focus { outline: none; border-color: #38bdf8; }
.editor-actions { display: flex; gap: 0.5rem; }
.editor-actions button { padding: 0.6rem 1.2rem; border-radius: 6px; font-size: 0.95rem; cursor: pointer; }
.editor-actions button[type="submit"] { background: #0ea5e9; color: white; border: none; }
.cancel-btn { background: transparent; border: 1px solid #334155; color: #94a3b8; }

@media (max-width: 768px) {
  .notes-layout { grid-template-columns: 1fr; }
  .notes-sidebar { max-height: 200px; }
}
```

</details>

---

## 8. Commit

```bash
git add .
git commit -m "feat: add auth pages, React Router, notes UI, and auth context"
```

### ✅ Checkpoint

- [ ] Can register a new account
- [ ] Can log in with existing credentials
- [ ] Redirected to login if accessing /notes without being logged in
- [ ] Can create, edit, and delete notes
- [ ] Logging out clears the session
- [ ] Refreshing the page maintains the login (token persisted)

---

> **Session Break** — Frontend complete.
> When you return, you'll deploy in [Step 7 — Deployment](../07-deployment/).

---

| | |
|:---|---:|
| [← Step 5: Backend](../05-backend/) | [Step 7 — Deployment →](../07-deployment/) |
