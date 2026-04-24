# Step 5 — Building the Frontend: Projects and Tasks UI

> **New to React or forgot the basics?** The JSX, hooks, events, and state lifecycle are the same as Level 1 Step 4. If `useState`, `useEffect`, `.map`, or the controlled-input pattern feel unfamiliar, revisit Level 1's "Plain-English Primer: JSX, Hooks, and Events" before continuing.
>
> **New to Level 2 on the frontend:**
> - **Generic functions** like `request<T>(url): Promise<T>` — you've seen `<T>` in types before; here it generalizes a helper across every API call.
> - **Optional chaining (`?.`)** — `options?.headers` evaluates to `undefined` if `options` is null/undefined, instead of throwing.
> - **`...prev.tasks`** inside state updates — using spread to build a new array/object that includes every existing item plus your change. React state must be replaced, never mutated.
> - **`e.stopPropagation()`** — stops a click from bubbling up to a clickable parent.

## Spatial Orientation

```
task-forge/
└── client/
    └── src/
        ├── types/
        │   └── index.ts         ← Shared interfaces
        ├── services/
        │   └── api.ts           ← 9 API functions
        ├── components/
        │   ├── ProjectForm.tsx   ← Create new projects
        │   ├── ProjectList.tsx   ← Display project cards
        │   ├── ProjectDetail.tsx ← Single project with tasks
        │   ├── TaskForm.tsx      ← Add tasks to a project
        │   ├── TaskList.tsx      ← Display tasks
        │   └── TaskItem.tsx      ← Single task with edit/delete
        ├── App.tsx               ← Root component, state owner
        └── App.css               ← Styling
```

---

## 1. Frontend Types

<details>
<summary>See the code</summary>

Create `client/src/types/index.ts`:

```typescript
export interface Project {
  id: number;
  name: string;
  description: string;
  created_at: string;
  updated_at: string;
}

export interface Task {
  id: number;
  project_id: number;
  title: string;
  completed: boolean;
  created_at: string;
  updated_at: string;
}

export interface ProjectWithTasks extends Project {
  tasks: Task[];
}

export interface CreateProjectRequest {
  name: string;
  description?: string;
}

export interface UpdateProjectRequest {
  name?: string;
  description?: string;
}

export interface CreateTaskRequest {
  title: string;
}

export interface UpdateTaskRequest {
  title?: string;
  completed?: boolean;
}
```

</details>

---

## 2. API Service Layer

> **Key Concept: Service Layer**
> Centralizing all API calls in one file means: (1) change the base URL in one place, (2) add authentication headers in one place (Level 3), (3) handle errors consistently. Components should never contain `fetch()` calls directly.

### 🏗️ Your Turn

Build an API service with 9 functions matching your 9 data endpoints. Each function should:
- Call the correct endpoint with the correct HTTP method
- Send JSON body for POST/PUT requests
- Check `response.ok` and throw on errors
- Return the parsed JSON response

<details>
<summary>See the solution</summary>

Create `client/src/services/api.ts`:

```typescript
import {
  Project,
  ProjectWithTasks,
  Task,
  CreateProjectRequest,
  UpdateProjectRequest,
  CreateTaskRequest,
  UpdateTaskRequest,
} from '../types';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3001/api';

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${API_URL}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!response.ok) {
    const errorData = await response.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(errorData.error || `Request failed with status ${response.status}`);
  }

  // 204 No Content has no body
  if (response.status === 204) {
    return undefined as T;
  }

  return response.json();
}

// Projects
export const getProjects = () => request<Project[]>('/projects');
export const getProject = (id: number) => request<ProjectWithTasks>(`/projects/${id}`);
export const createProject = (data: CreateProjectRequest) =>
  request<Project>('/projects', { method: 'POST', body: JSON.stringify(data) });
export const updateProject = (id: number, data: UpdateProjectRequest) =>
  request<Project>(`/projects/${id}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteProject = (id: number) =>
  request<void>(`/projects/${id}`, { method: 'DELETE' });

// Tasks
export const getTasks = (projectId: number) => request<Task[]>(`/projects/${projectId}/tasks`);
export const createTask = (projectId: number, data: CreateTaskRequest) =>
  request<Task>(`/projects/${projectId}/tasks`, { method: 'POST', body: JSON.stringify(data) });
export const updateTask = (id: number, data: UpdateTaskRequest) =>
  request<Task>(`/tasks/${id}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteTask = (id: number) =>
  request<void>(`/tasks/${id}`, { method: 'DELETE' });
```

</details>

### Reading This File Line-by-Line

The imports are straightforward named imports of types. The action starts with the generic helper:

```typescript
async function request<T>(url: string, options?: RequestInit): Promise<T> {
```

- `function request<T>` — `<T>` is a **generic type parameter**. Read it as "this function works for any type `T` that the caller picks." When you write `request<Project[]>(...)`, `T` is `Project[]` for that call. When you write `request<Task>(...)`, `T` is `Task`.
- `options?: RequestInit` — `?` marks the parameter as **optional**. `RequestInit` is a built-in TypeScript type (from the DOM lib) that describes the options object `fetch` accepts: `{ method, headers, body, ... }`.
- `: Promise<T>` — returns a promise that resolves to a value of type `T`. Whatever `T` is, the caller's `await` gets back a properly typed value.

```typescript
  const response = await fetch(`${API_URL}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });
```

- `` `${API_URL}${url}` `` — build the full URL by concatenating the base and the relative path.
- `{ ...options, ... }` — spread `options` into a new object so we can override fields. If the caller passed `{ method: 'POST', body: '...' }`, those land in the new object.
- `headers: { 'Content-Type': 'application/json', ...options?.headers }` — start with the default Content-Type, then spread any caller-provided headers on top. If the caller also sets Content-Type, their version wins (later keys override earlier ones).
- `options?.headers` — **optional chaining**. `options?.headers` evaluates to `undefined` if `options` is undefined, rather than throwing. `...undefined` spreads nothing, so it's safe.

```typescript
  if (!response.ok) {
    const errorData = await response.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(errorData.error || `Request failed with status ${response.status}`);
  }
```

- `response.ok` is `true` for 200–299, `false` otherwise.
- `response.json().catch(() => ({ error: 'Request failed' }))` — try to parse the error body as JSON; if parsing fails (e.g., the server returned HTML or an empty body), fall back to a default shape. The `.catch(handler)` method on a promise runs `handler` if the promise rejects.
- `throw new Error(...)` — throw so callers can `catch` it.

```typescript
  if (response.status === 204) {
    return undefined as T;
  }

  return response.json();
}
```

- `204 No Content` — what DELETE returns on success. There's no body to parse. `return undefined as T` forces TypeScript to accept `undefined` as the generic return type. Callers who do `deleteProject(5)` use `Promise<void>` and never read the result.
- Otherwise, parse and return the JSON body.

```typescript
export const getProjects = () => request<Project[]>('/projects');
export const getProject = (id: number) => request<ProjectWithTasks>(`/projects/${id}`);
```

Each export is a one-line arrow function that calls `request<T>` with a URL and optionally a method/body. Because `request` is generic, each call locks in a specific `T`:

- `request<Project[]>('/projects')` — the returned promise is typed as `Promise<Project[]>`. Callers get autocomplete on `project.name` etc.
- `request<ProjectWithTasks>(\`/projects/${id}\`)` — templated URL with the project ID.

```typescript
export const createProject = (data: CreateProjectRequest) =>
  request<Project>('/projects', { method: 'POST', body: JSON.stringify(data) });
```

- Takes `data` of type `CreateProjectRequest`.
- Passes options as the second argument: `method: 'POST'` and `body: JSON.stringify(data)`.
- `JSON.stringify(obj)` converts a JavaScript object to a JSON string. `fetch` requires the body to be a string.

Same shape for `updateProject`, `deleteProject`, `createTask`, `updateTask`, `deleteTask`. The genericized `request` helper is the whole reason these are one-liners instead of nine separate 10-line functions.

**Key pattern: Generic `request` helper.** Instead of repeating `fetch` + headers + error handling in 9 functions, we extract it into one helper. This is the DRY principle (Don't Repeat Yourself) applied to API calls.

---

## 3. Build the Components

Build each component. Try to write them yourself first — the hints tell you what state and props each component needs.

### ProjectForm

**Props:** `onSubmit: (data: CreateProjectRequest) => Promise<void>`
**State:** `name`, `description`, `isSubmitting`, `error`

<details>
<summary>See the code</summary>

```tsx
import { useState, FormEvent } from 'react';
import { CreateProjectRequest } from '../types';

interface Props {
  onSubmit: (data: CreateProjectRequest) => Promise<void>;
}

export function ProjectForm({ onSubmit }: Props) {
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    if (!name.trim()) return;
    setError('');
    setIsSubmitting(true);

    try {
      await onSubmit({ name: name.trim(), description: description.trim() });
      setName('');
      setDescription('');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create project');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="project-form">
      {error && <p className="error">{error}</p>}
      <input
        type="text"
        placeholder="Project name"
        value={name}
        onChange={(e) => setName(e.target.value)}
        maxLength={100}
        required
      />
      <input
        type="text"
        placeholder="Description (optional)"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
      />
      <button type="submit" disabled={isSubmitting || !name.trim()}>
        {isSubmitting ? 'Creating...' : 'Create Project'}
      </button>
    </form>
  );
}
```

</details>

### ProjectList

**Props:** `projects`, `onSelect: (id: number) => void`, `onDelete: (id: number) => Promise<void>`

<details>
<summary>See the code</summary>

```tsx
import { Project } from '../types';

interface Props {
  projects: Project[];
  onSelect: (id: number) => void;
  onDelete: (id: number) => Promise<void>;
}

export function ProjectList({ projects, onSelect, onDelete }: Props) {
  if (projects.length === 0) {
    return <p className="empty-state">No projects yet. Create one above!</p>;
  }

  return (
    <div className="project-list">
      {projects.map((project) => (
        <div
          key={project.id}
          className="project-card"
          onClick={() => onSelect(project.id)}
        >
          <div className="project-card-header">
            <h3>{project.name}</h3>
            <button
              className="delete-btn"
              onClick={(e) => {
                e.stopPropagation();
                if (confirm('Delete this project and all its tasks?')) {
                  onDelete(project.id);
                }
              }}
            >
              Delete
            </button>
          </div>
          {project.description && <p>{project.description}</p>}
        </div>
      ))}
    </div>
  );
}
```

</details>

### Reading ProjectForm and ProjectList

**ProjectForm** follows the exact shape of Level 1's EntryForm: controlled inputs, async submit handler with try/catch/finally, reset on success. If this feels unfamiliar, revisit Level 1 Step 4's walkthrough of EntryForm. The only new detail: `maxLength={100}` on the name input is an HTML attribute that stops typing past 100 characters — a client-side reinforcement of the server's `VARCHAR(100)` check.

**ProjectList** is a straight `.map` over the projects array, with one subtle twist: the **nested click handlers**.

```tsx
<div
  key={project.id}
  className="project-card"
  onClick={() => onSelect(project.id)}
>
  <div className="project-card-header">
    <h3>{project.name}</h3>
    <button
      className="delete-btn"
      onClick={(e) => {
        e.stopPropagation();
        if (confirm('Delete this project and all its tasks?')) {
          onDelete(project.id);
        }
      }}
    >
```

- The whole card has `onClick` — clicking anywhere on it opens that project.
- The delete button is inside the card and has its own `onClick`. Without intervention, clicking delete would fire BOTH handlers: first the button's (bubble up), then the card's (open the project the user is trying to delete).
- `e.stopPropagation()` stops the click from bubbling up to parent elements. Now only the button's handler runs.
- `confirm('...')` — a built-in browser function that opens a native "OK / Cancel" dialog and returns `true` or `false`. Synchronous — execution pauses until the user responds.

> **Key pattern: `e.stopPropagation()`**
> The delete button is inside a clickable card. Without `stopPropagation()`, clicking delete would ALSO trigger the card's `onClick` (selecting the project). `stopPropagation()` prevents the click from bubbling up to the parent.

### TaskItem

**Props:** `task`, `onUpdate`, `onDelete`
**State:** `isEditing`, `editTitle`

<details>
<summary>See the code</summary>

```tsx
import { useState } from 'react';
import { Task } from '../types';

interface Props {
  task: Task;
  onUpdate: (id: number, data: { title?: string; completed?: boolean }) => Promise<void>;
  onDelete: (id: number) => Promise<void>;
}

export function TaskItem({ task, onUpdate, onDelete }: Props) {
  const [isEditing, setIsEditing] = useState(false);
  const [editTitle, setEditTitle] = useState(task.title);

  const handleToggle = () => {
    onUpdate(task.id, { completed: !task.completed });
  };

  const handleSaveEdit = () => {
    if (editTitle.trim() && editTitle.trim() !== task.title) {
      onUpdate(task.id, { title: editTitle.trim() });
    }
    setIsEditing(false);
  };

  return (
    <div className={`task-item ${task.completed ? 'completed' : ''}`}>
      <input
        type="checkbox"
        checked={task.completed}
        onChange={handleToggle}
      />

      {isEditing ? (
        <input
          type="text"
          value={editTitle}
          onChange={(e) => setEditTitle(e.target.value)}
          onBlur={handleSaveEdit}
          onKeyDown={(e) => {
            if (e.key === 'Enter') handleSaveEdit();
            if (e.key === 'Escape') {
              setEditTitle(task.title);
              setIsEditing(false);
            }
          }}
          autoFocus
          className="edit-input"
        />
      ) : (
        <span
          className="task-title"
          onDoubleClick={() => setIsEditing(true)}
        >
          {task.title}
        </span>
      )}

      <button
        className="delete-btn"
        onClick={() => onDelete(task.id)}
      >
        ×
      </button>
    </div>
  );
}
```

</details>

### Reading TaskItem Line-by-Line

TaskItem has the most new React patterns in Level 2. Take it slow.

```tsx
const [isEditing, setIsEditing] = useState(false);
const [editTitle, setEditTitle] = useState(task.title);
```

Two local state values: a flag for "am I in edit mode right now" and a working copy of the title while editing.

```tsx
const handleToggle = () => {
  onUpdate(task.id, { completed: !task.completed });
};
```

- `!task.completed` — the `!` operator flips a boolean. So this updates the task with the opposite of its current completed value.
- We don't await `onUpdate` here because we don't need anything back; we just fire-and-forget the update.

```tsx
const handleSaveEdit = () => {
  if (editTitle.trim() && editTitle.trim() !== task.title) {
    onUpdate(task.id, { title: editTitle.trim() });
  }
  setIsEditing(false);
};
```

- Only persist if the trimmed title is non-empty **and** actually different from the original (no pointless API call for a noop edit).
- Always leave edit mode when saving, whether or not we persisted.

```tsx
<div className={`task-item ${task.completed ? 'completed' : ''}`}>
```

Template literal for the `className`. `task.completed ? 'completed' : ''` adds the `completed` class conditionally. CSS then uses that class to strike through the text.

```tsx
<input
  type="checkbox"
  checked={task.completed}
  onChange={handleToggle}
/>
```

A **controlled checkbox**. `checked={task.completed}` keeps the checkbox in sync with the task's completion state. When the user clicks, `onChange` fires and we toggle via the parent.

```tsx
{isEditing ? (
  <input ... />
) : (
  <span ...>
    {task.title}
  </span>
)}
```

**Ternary for conditional rendering.** When you need to choose between two different elements (not just show/hide), use `condition ? <A /> : <B />`. Wrap each branch in parentheses so JSX and the ternary operator don't get tangled up.

```tsx
<input
  type="text"
  value={editTitle}
  onChange={(e) => setEditTitle(e.target.value)}
  onBlur={handleSaveEdit}
  onKeyDown={(e) => {
    if (e.key === 'Enter') handleSaveEdit();
    if (e.key === 'Escape') {
      setEditTitle(task.title);
      setIsEditing(false);
    }
  }}
  autoFocus
  className="edit-input"
/>
```

Four event handlers working together to give the edit input "desktop app" feel:

- `onChange` — update state on every keystroke. Standard controlled input.
- `onBlur` — fires when the input loses focus (user clicks elsewhere). We save.
- `onKeyDown` — fires when a key is pressed down. `e.key` is the key's name as a string: `'Enter'`, `'Escape'`, `'a'`, `'ArrowLeft'`, etc.
  - Enter → save.
  - Escape → revert the working copy to the original title and exit edit mode.
- `autoFocus` — a boolean attribute. React focuses the input as soon as it mounts, so the user can start typing immediately.

```tsx
<span
  className="task-title"
  onDoubleClick={() => setIsEditing(true)}
>
  {task.title}
</span>
```

`onDoubleClick` — native browser event for a double-click. Standard pattern for "inline edit on double-click." Enters edit mode, which triggers the ternary to swap the span for the input on the next render.

### TaskForm, TaskList, ProjectDetail

<details>
<summary>TaskForm</summary>

```tsx
import { useState, FormEvent } from 'react';

interface Props {
  onSubmit: (title: string) => Promise<void>;
}

export function TaskForm({ onSubmit }: Props) {
  const [title, setTitle] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;
    setIsSubmitting(true);
    try {
      await onSubmit(title.trim());
      setTitle('');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="task-form">
      <input
        type="text"
        placeholder="Add a task..."
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        maxLength={200}
      />
      <button type="submit" disabled={isSubmitting || !title.trim()}>
        Add
      </button>
    </form>
  );
}
```

</details>

<details>
<summary>TaskList</summary>

```tsx
import { Task } from '../types';
import { TaskItem } from './TaskItem';

interface Props {
  tasks: Task[];
  onUpdate: (id: number, data: { title?: string; completed?: boolean }) => Promise<void>;
  onDelete: (id: number) => Promise<void>;
}

export function TaskList({ tasks, onUpdate, onDelete }: Props) {
  const completed = tasks.filter((t) => t.completed).length;

  return (
    <div className="task-list">
      <p className="task-count">
        {completed}/{tasks.length} completed
      </p>
      {tasks.length === 0 ? (
        <p className="empty-state">No tasks yet. Add one above!</p>
      ) : (
        tasks.map((task) => (
          <TaskItem
            key={task.id}
            task={task}
            onUpdate={onUpdate}
            onDelete={onDelete}
          />
        ))
      )}
    </div>
  );
}
```

</details>

<details>
<summary>ProjectDetail</summary>

```tsx
import { ProjectWithTasks } from '../types';
import { TaskForm } from './TaskForm';
import { TaskList } from './TaskList';

interface Props {
  project: ProjectWithTasks;
  onBack: () => void;
  onCreateTask: (title: string) => Promise<void>;
  onUpdateTask: (id: number, data: { title?: string; completed?: boolean }) => Promise<void>;
  onDeleteTask: (id: number) => Promise<void>;
}

export function ProjectDetail({
  project,
  onBack,
  onCreateTask,
  onUpdateTask,
  onDeleteTask,
}: Props) {
  return (
    <div className="project-detail">
      <button className="back-btn" onClick={onBack}>
        ← Back to Projects
      </button>
      <h2>{project.name}</h2>
      {project.description && <p className="project-description">{project.description}</p>}
      <TaskForm onSubmit={onCreateTask} />
      <TaskList
        tasks={project.tasks}
        onUpdate={onUpdateTask}
        onDelete={onDeleteTask}
      />
    </div>
  );
}
```

</details>

---

## 4. Build the App Component

### 🏗️ Your Turn

Build the root App component that:
- Holds `projects` array and `selectedProject` in state
- Fetches projects on mount
- Handles selecting a project (fetches full project with tasks)
- Handles all CRUD operations for both projects and tasks
- Switches between list view and detail view

<details>
<summary>See the solution</summary>

Replace `client/src/App.tsx`:

```tsx
import { useState, useEffect } from 'react';
import { Project, ProjectWithTasks, CreateProjectRequest } from './types';
import * as api from './services/api';
import { ProjectForm } from './components/ProjectForm';
import { ProjectList } from './components/ProjectList';
import { ProjectDetail } from './components/ProjectDetail';
import './App.css';

function App() {
  const [projects, setProjects] = useState<Project[]>([]);
  const [selectedProject, setSelectedProject] = useState<ProjectWithTasks | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState('');

  // Fetch projects on mount
  useEffect(() => {
    loadProjects();
  }, []);

  const loadProjects = async () => {
    try {
      const data = await api.getProjects();
      setProjects(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load projects');
    } finally {
      setIsLoading(false);
    }
  };

  const selectProject = async (id: number) => {
    try {
      const project = await api.getProject(id);
      setSelectedProject(project);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load project');
    }
  };

  const handleCreateProject = async (data: CreateProjectRequest) => {
    const newProject = await api.createProject(data);
    setProjects((prev) => [newProject, ...prev]);
  };

  const handleDeleteProject = async (id: number) => {
    await api.deleteProject(id);
    setProjects((prev) => prev.filter((p) => p.id !== id));
    if (selectedProject?.id === id) {
      setSelectedProject(null);
    }
  };

  const handleCreateTask = async (title: string) => {
    if (!selectedProject) return;
    const newTask = await api.createTask(selectedProject.id, { title });
    setSelectedProject((prev) =>
      prev ? { ...prev, tasks: [...prev.tasks, newTask] } : prev
    );
  };

  const handleUpdateTask = async (id: number, data: { title?: string; completed?: boolean }) => {
    const updatedTask = await api.updateTask(id, data);
    setSelectedProject((prev) =>
      prev
        ? { ...prev, tasks: prev.tasks.map((t) => (t.id === id ? updatedTask : t)) }
        : prev
    );
  };

  const handleDeleteTask = async (id: number) => {
    await api.deleteTask(id);
    setSelectedProject((prev) =>
      prev ? { ...prev, tasks: prev.tasks.filter((t) => t.id !== id) } : prev
    );
  };

  if (isLoading) return <div className="loading">Loading...</div>;

  return (
    <div className="app">
      <header>
        <h1>TaskForge</h1>
        <p>Project Task Manager</p>
      </header>

      {error && <p className="error">{error}</p>}

      {selectedProject ? (
        <ProjectDetail
          project={selectedProject}
          onBack={() => setSelectedProject(null)}
          onCreateTask={handleCreateTask}
          onUpdateTask={handleUpdateTask}
          onDeleteTask={handleDeleteTask}
        />
      ) : (
        <>
          <ProjectForm onSubmit={handleCreateProject} />
          <ProjectList
            projects={projects}
            onSelect={selectProject}
            onDelete={handleDeleteProject}
          />
        </>
      )}
    </div>
  );
}

export default App;
```

</details>

### Reading App.tsx Line-by-Line

App owns all the state. Every child component receives data + callbacks as props and never touches API calls or state directly. Let's walk the tricky parts.

```tsx
import * as api from './services/api';
```

**Namespace import.** Instead of `import { getProjects, createProject, ... } from './services/api'` with nine names to list, we pull everything into one namespace and call `api.getProjects()`, `api.createProject(...)`. Reads well at the call site.

```tsx
const [projects, setProjects] = useState<Project[]>([]);
const [selectedProject, setSelectedProject] = useState<ProjectWithTasks | null>(null);
```

- `Project[]` — array of projects, the list-view data.
- `ProjectWithTasks | null` — **union type**. The state is either a full project (with its tasks) or `null` (no project selected). The `| null` is how we model "sometimes present, sometimes not."

```tsx
useEffect(() => {
  loadProjects();
}, []);
```

Empty dependency array → run once on mount. Calls `loadProjects` which fetches and stores the project list. (Recall from Level 1: an async function must be defined inside or adjacent to the effect because the effect callback itself can't be async.)

```tsx
const handleCreateProject = async (data: CreateProjectRequest) => {
  const newProject = await api.createProject(data);
  setProjects((prev) => [newProject, ...prev]);
};
```

- Call the API, get the saved project back (with id and timestamps).
- `setProjects((prev) => [newProject, ...prev])` — **functional updater**. Uses the latest state, not a stale closure. `[newProject, ...prev]` puts the new one at the top.

```tsx
const handleDeleteProject = async (id: number) => {
  await api.deleteProject(id);
  setProjects((prev) => prev.filter((p) => p.id !== id));
  if (selectedProject?.id === id) {
    setSelectedProject(null);
  }
};
```

- `prev.filter((p) => p.id !== id)` — returns a new array with every project whose id is NOT the deleted one. `.filter((item) => condition)` is an array method that keeps items where `condition` returns true.
- `selectedProject?.id === id` — **optional chaining** again. If `selectedProject` is `null`, `selectedProject?.id` evaluates to `undefined` (not a crash), and `undefined === id` is false. So the `if` safely handles both "no project selected" and "different project selected."
- If the deleted project happened to be the one currently open, close the detail view.

```tsx
const handleCreateTask = async (title: string) => {
  if (!selectedProject) return;
  const newTask = await api.createTask(selectedProject.id, { title });
  setSelectedProject((prev) =>
    prev ? { ...prev, tasks: [...prev.tasks, newTask] } : prev
  );
};
```

- `if (!selectedProject) return;` — guard clause. Can't add a task if no project is open.
- `setSelectedProject((prev) => prev ? {...} : prev)` — the updater function always accepts and returns a value, but we only build a new object when `prev` is non-null.
- `{ ...prev, tasks: [...prev.tasks, newTask] }` — **nested spread**. Copy every field of `prev` into a new object; then override `tasks` with a new array containing every existing task plus the new one. React requires a new object reference for state changes to register.

```tsx
const handleUpdateTask = async (id: number, data: {...}) => {
  const updatedTask = await api.updateTask(id, data);
  setSelectedProject((prev) =>
    prev
      ? { ...prev, tasks: prev.tasks.map((t) => (t.id === id ? updatedTask : t)) }
      : prev
  );
};
```

- `prev.tasks.map((t) => t.id === id ? updatedTask : t)` — walk the tasks array. For the task whose id matches, substitute the updated version; leave all others unchanged. `.map` always produces a new array of the same length.

```tsx
{selectedProject ? (
  <ProjectDetail ... />
) : (
  <>
    <ProjectForm onSubmit={handleCreateProject} />
    <ProjectList ... />
  </>
)}
```

- Ternary in JSX: if a project is selected, render `<ProjectDetail />`; otherwise render the list view.
- `<>...</>` is a **React fragment** — a no-op wrapper. When you need to return multiple elements from one branch but don't want to add a real `<div>`, wrap them in `<>` and `</>`. Fragments don't show up in the DOM.

---

## 5. Add Styles

The CSS here follows the same dark-theme pattern you saw in Level 1. If you need a refresher on how CSS rules are built (selectors, properties, units, colors), see Level 1 Step 4's "How to Read This CSS" section. Nothing new in Level 2's stylesheet — just different class names for different components.

<details>
<summary>See App.css (dark theme)</summary>

Replace `client/src/App.css`:

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #0f172a; color: #e2e8f0; line-height: 1.6; }
.app { max-width: 700px; margin: 0 auto; padding: 2rem 1rem; }
header { text-align: center; margin-bottom: 2rem; }
header h1 { font-size: 2rem; color: #38bdf8; }
header p { color: #94a3b8; }
.loading { text-align: center; padding: 4rem; color: #64748b; }
.error { background: #450a0a; color: #fca5a5; padding: 0.75rem 1rem; border-radius: 8px; margin-bottom: 1rem; }
.empty-state { text-align: center; padding: 2rem; color: #64748b; }

/* Forms */
.project-form, .task-form { display: flex; gap: 0.5rem; margin-bottom: 1.5rem; flex-wrap: wrap; }
.project-form input, .task-form input { flex: 1; min-width: 200px; padding: 0.6rem; border: 1px solid #334155; border-radius: 6px; background: #1e293b; color: #e2e8f0; font-size: 0.95rem; }
.project-form input:focus, .task-form input:focus { outline: none; border-color: #38bdf8; }
.project-form button, .task-form button { padding: 0.6rem 1.2rem; background: #0ea5e9; color: white; border: none; border-radius: 6px; cursor: pointer; font-size: 0.95rem; }
.project-form button:hover:not(:disabled), .task-form button:hover:not(:disabled) { background: #0284c7; }
.project-form button:disabled, .task-form button:disabled { opacity: 0.5; cursor: not-allowed; }

/* Project cards */
.project-list { display: flex; flex-direction: column; gap: 0.75rem; }
.project-card { background: #1e293b; border-radius: 8px; padding: 1rem; cursor: pointer; transition: border-color 0.2s; border: 1px solid transparent; }
.project-card:hover { border-color: #38bdf8; }
.project-card-header { display: flex; justify-content: space-between; align-items: center; }
.project-card h3 { font-size: 1.1rem; }
.project-card p { color: #94a3b8; font-size: 0.9rem; margin-top: 0.25rem; }

/* Project detail */
.project-detail h2 { margin-bottom: 0.5rem; }
.project-description { color: #94a3b8; margin-bottom: 1rem; }
.back-btn { background: transparent; border: 1px solid #334155; color: #94a3b8; padding: 0.4rem 0.8rem; border-radius: 6px; cursor: pointer; margin-bottom: 1rem; }
.back-btn:hover { border-color: #38bdf8; color: #e2e8f0; }

/* Tasks */
.task-count { color: #64748b; font-size: 0.85rem; margin-bottom: 0.75rem; }
.task-item { display: flex; align-items: center; gap: 0.75rem; padding: 0.6rem; background: #1e293b; border-radius: 6px; margin-bottom: 0.5rem; }
.task-item.completed .task-title { text-decoration: line-through; color: #64748b; }
.task-item input[type="checkbox"] { accent-color: #0ea5e9; width: 18px; height: 18px; cursor: pointer; }
.task-title { flex: 1; cursor: default; }
.edit-input { flex: 1; background: #0f172a; border: 1px solid #38bdf8; color: #e2e8f0; padding: 0.3rem; border-radius: 4px; font-size: 0.95rem; }
.delete-btn { background: transparent; border: none; color: #64748b; cursor: pointer; font-size: 1.1rem; padding: 0.2rem 0.5rem; }
.delete-btn:hover { color: #ef4444; }
```

</details>

---

## 6. Test and Commit

Update `client/index.html` title to `TaskForge — Project Task Manager`.

```bash
git add .
git commit -m "feat: add React frontend with full CRUD for projects and tasks"
```

### ✅ Checkpoint

- [ ] Can create projects
- [ ] Can click a project to see its tasks
- [ ] Can add tasks to a project
- [ ] Can toggle task completion (checkbox)
- [ ] Can double-click a task to edit it
- [ ] Can delete tasks and projects
- [ ] Back button returns to project list

---

> **Session Break** — Frontend complete.
> When you return, you'll deploy everything in [Step 6 — Deployment](../06-deployment/).

---

| | |
|:---|---:|
| [← Step 4: Backend](../04-backend/) | [Step 6 — Deployment →](../06-deployment/) |
