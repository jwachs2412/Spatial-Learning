# Step 5 — Building the Frontend: Projects and Tasks UI

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

---

## 5. Add Styles

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
