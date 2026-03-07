`Level 2` **Step 5 of 7** — Building the Frontend

# 05 — Building the Frontend: Projects and Tasks UI

## Spatial Orientation

```
task-forge/
├── client/              ← ★ WE ARE HERE ★
│   └── src/
│       ├── components/        ← UI building blocks (we'll build these)
│       │   ├── ProjectForm.tsx
│       │   ├── ProjectList.tsx
│       │   ├── ProjectDetail.tsx
│       │   ├── TaskForm.tsx
│       │   ├── TaskList.tsx
│       │   └── TaskItem.tsx
│       ├── services/          ← Backend communication (we'll build this)
│       │   └── api.ts
│       ├── types/             ← Shared types (we'll build this)
│       │   └── index.ts
│       ├── App.tsx            ← Root component (we'll rewrite this)
│       ├── App.css            ← Styles (we'll rewrite this)
│       └── main.tsx           ← Entry point (leave as is)
└── server/              ← Already built ✓
```

**What layer are we in?** The CLIENT layer. This code runs in the user's browser. It's what they see and interact with. It sends HTTP requests to the backend to create, read, update, and delete data.

### Component Tree

```
App
├── ProjectForm          ← Form to create a new project
├── ProjectList          ← Shows all projects as cards
│   └── (project cards)  ← Click one to select it
└── ProjectDetail        ← Shows selected project + its tasks
    ├── TaskForm         ← Form to add a task
    └── TaskList         ← Shows all tasks for this project
        └── TaskItem     ← Single task with checkbox, edit, delete
```

**How views switch**: There's no React Router. We use a single state variable `selectedProjectId`:
- When `null` → show `ProjectList` (all projects)
- When a number → show `ProjectDetail` (one project and its tasks)

This is the simplest possible navigation pattern. React Router adds complexity we don't need for two views.

---

## Step 1: Define Shared Types

**Where?** `client/src/types/index.ts`

> [!IMPORTANT]
> **You should be in:** `task-forge/` (the project root)

```bash
mkdir -p client/src/types
mkdir -p client/src/services
mkdir -p client/src/components
```

In VS Code, create a new file at `client/src/types/index.ts`:

```typescript
// ─── DATABASE MODELS ────────────────────────────────────────────
// These match what the backend returns.

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

// ─── COMPOSITE TYPES ────────────────────────────────────────────

export interface ProjectWithTasks extends Project {
  tasks: Task[];
}

// ─── REQUEST TYPES ──────────────────────────────────────────────

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

These match the backend types exactly — same pattern as Level 1.

---

## Step 2: Build the API Service

**Where?** `client/src/services/api.ts`

This file contains every function that talks to the backend. In Level 1 you had 2 functions (`getEntries`, `createEntry`). Level 2 has 9 — one for each CRUD operation.

In VS Code, create a new file at `client/src/services/api.ts`:

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

// ─── PROJECT API ────────────────────────────────────────────────

export async function getProjects(): Promise<Project[]> {
  const response = await fetch(`${API_URL}/projects`);
  if (!response.ok) throw new Error('Failed to fetch projects');
  return response.json();
}

export async function getProject(id: number): Promise<ProjectWithTasks> {
  const response = await fetch(`${API_URL}/projects/${id}`);
  if (!response.ok) throw new Error('Failed to fetch project');
  return response.json();
}

export async function createProject(data: CreateProjectRequest): Promise<Project> {
  const response = await fetch(`${API_URL}/projects`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to create project');
  }
  return response.json();
}

export async function updateProject(
  id: number,
  data: UpdateProjectRequest
): Promise<Project> {
  const response = await fetch(`${API_URL}/projects/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to update project');
  }
  return response.json();
}

export async function deleteProject(id: number): Promise<void> {
  const response = await fetch(`${API_URL}/projects/${id}`, {
    method: 'DELETE',
  });
  if (!response.ok) throw new Error('Failed to delete project');
}

// ─── TASK API ───────────────────────────────────────────────────

export async function getTasks(projectId: number): Promise<Task[]> {
  const response = await fetch(`${API_URL}/projects/${projectId}/tasks`);
  if (!response.ok) throw new Error('Failed to fetch tasks');
  return response.json();
}

export async function createTask(
  projectId: number,
  data: CreateTaskRequest
): Promise<Task> {
  const response = await fetch(`${API_URL}/projects/${projectId}/tasks`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to create task');
  }
  return response.json();
}

export async function updateTask(
  id: number,
  data: UpdateTaskRequest
): Promise<Task> {
  const response = await fetch(`${API_URL}/tasks/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Failed to update task');
  }
  return response.json();
}

export async function deleteTask(id: number): Promise<void> {
  const response = await fetch(`${API_URL}/tasks/${id}`, {
    method: 'DELETE',
  });
  if (!response.ok) throw new Error('Failed to delete task');
}
```

### Pattern: Every API Function Follows the Same Shape

```typescript
export async function doSomething(params): Promise<ReturnType> {
  const response = await fetch(url, options);
  if (!response.ok) throw new Error('...');
  return response.json();  // or nothing for DELETE
}
```

- `fetch()` sends the request
- Check `response.ok` — throw if it failed
- Parse and return the JSON body

The only differences between functions are the HTTP method, URL, and whether there's a body. This repetition is intentional — it's simple and predictable.

---

## Step 3: Build the ProjectForm Component

**Where?** `client/src/components/ProjectForm.tsx`

A form for creating new projects. Same pattern as Level 1's `EntryForm` but with different fields.

```tsx
import { useState } from 'react';
import { CreateProjectRequest } from '../types';

interface ProjectFormProps {
  onSubmit: (data: CreateProjectRequest) => Promise<void>;
}

export function ProjectForm({ onSubmit }: ProjectFormProps) {
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (name.trim().length === 0) {
      setError('Project name is required');
      return;
    }

    setIsSubmitting(true);
    setError('');

    try {
      await onSubmit({ name: name.trim(), description: description.trim() });
      setName('');
      setDescription('');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="project-form">
      <h2>New Project</h2>
      <div className="form-group">
        <input
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          placeholder="Project name"
          maxLength={100}
        />
      </div>
      <div className="form-group">
        <textarea
          value={description}
          onChange={(e) => setDescription(e.target.value)}
          placeholder="Description (optional)"
          rows={2}
        />
      </div>
      {error && <p className="error-message">{error}</p>}
      <button type="submit" className="submit-button" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create Project'}
      </button>
    </form>
  );
}
```

Same structure as Level 1: state for each field, handleSubmit with validation, onSubmit prop that the parent provides. Nothing new here — just different fields.

---

## Step 4: Build the ProjectList Component

**Where?** `client/src/components/ProjectList.tsx`

Shows all projects as clickable cards. When you click a project, the parent component switches to the detail view.

```tsx
import { Project } from '../types';

interface ProjectListProps {
  projects: Project[];
  onSelect: (id: number) => void;
  onDelete: (id: number) => void;
}

export function ProjectList({ projects, onSelect, onDelete }: ProjectListProps) {
  if (projects.length === 0) {
    return (
      <div className="empty-state">
        <p>No projects yet. Create your first project above!</p>
      </div>
    );
  }

  return (
    <div className="project-list">
      <h2>Your Projects</h2>
      {projects.map((project) => (
        <div key={project.id} className="project-card">
          <div
            className="project-card-content"
            onClick={() => onSelect(project.id)}
          >
            <h3>{project.name}</h3>
            {project.description && (
              <p className="project-description">{project.description}</p>
            )}
            <time className="project-date">
              Created{' '}
              {new Date(project.created_at).toLocaleDateString('en-US', {
                month: 'short',
                day: 'numeric',
                year: 'numeric',
              })}
            </time>
          </div>
          <button
            className="delete-button"
            onClick={(e) => {
              e.stopPropagation();
              if (confirm('Delete this project and all its tasks?')) {
                onDelete(project.id);
              }
            }}
            title="Delete project"
          >
            ✕
          </button>
        </div>
      ))}
    </div>
  );
}
```

### New Concept: Delete With Confirmation

```typescript
onClick={(e) => {
  e.stopPropagation();  // Don't trigger the card's click (which selects the project)
  if (confirm('Delete this project and all its tasks?')) {
    onDelete(project.id);
  }
}}
```

**`e.stopPropagation()`** — The delete button is inside the clickable card. Without this, clicking delete would also trigger the card's `onClick` (selecting the project). `stopPropagation` prevents the click from "bubbling up" to the parent element.

**`confirm()`** — Shows a browser dialog asking "are you sure?" Delete is destructive — it removes the project and all its tasks permanently. A confirmation dialog prevents accidental deletion.

---

## Step 5: Build the TaskItem Component

**Where?** `client/src/components/TaskItem.tsx`

A single task with a checkbox, editable title, and delete button. This is the most interactive component — it supports three actions.

```tsx
import { useState } from 'react';
import { Task } from '../types';

interface TaskItemProps {
  task: Task;
  onToggle: (id: number, completed: boolean) => Promise<void>;
  onUpdate: (id: number, title: string) => Promise<void>;
  onDelete: (id: number) => Promise<void>;
}

export function TaskItem({ task, onToggle, onUpdate, onDelete }: TaskItemProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [editTitle, setEditTitle] = useState(task.title);

  const handleToggle = () => {
    onToggle(task.id, !task.completed);
  };

  const handleSave = async () => {
    if (editTitle.trim().length === 0) {
      setEditTitle(task.title);
      setIsEditing(false);
      return;
    }

    if (editTitle.trim() !== task.title) {
      await onUpdate(task.id, editTitle.trim());
    }
    setIsEditing(false);
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      handleSave();
    }
    if (e.key === 'Escape') {
      setEditTitle(task.title);
      setIsEditing(false);
    }
  };

  return (
    <div className={`task-item ${task.completed ? 'completed' : ''}`}>
      <input
        type="checkbox"
        checked={task.completed}
        onChange={handleToggle}
        className="task-checkbox"
      />

      {isEditing ? (
        <input
          type="text"
          value={editTitle}
          onChange={(e) => setEditTitle(e.target.value)}
          onBlur={handleSave}
          onKeyDown={handleKeyDown}
          className="task-edit-input"
          autoFocus
          maxLength={200}
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
        className="delete-button"
        onClick={() => onDelete(task.id)}
        title="Delete task"
      >
        ✕
      </button>
    </div>
  );
}
```

### Edit-in-Place Pattern

This is a common UI pattern: instead of a separate edit form, you click the text to edit it right there.

```
DISPLAY MODE:                    EDIT MODE:
──────────────                   ──────────
☐ Design homepage layout         ☐ [Design homepage layout    ]
     ↑ double-click                    ↑ type to edit
                                       ↑ Enter to save, Escape to cancel
                                       ↑ Click away (blur) to save
```

**How it works:**
1. `isEditing` state starts as `false` → shows the text as a `<span>`
2. Double-click the text → `setIsEditing(true)` → shows an `<input>`
3. Type to edit → `setEditTitle(newValue)` → input updates
4. Press Enter or click away (blur) → `handleSave()` → saves and switches back to display
5. Press Escape → revert to original title and switch back to display

**`autoFocus`** — Automatically focuses the input when it appears, so the user can start typing immediately.

---

> [!TIP]
> **Session Break** — You've built the types, API service, ProjectForm, ProjectList, and TaskItem components. Save your work and take a break.
> When you return, you'll build the remaining components and wire everything together.

---

## Step 6: Build the TaskList Component

**Where?** `client/src/components/TaskList.tsx`

Renders a list of `TaskItem` components. Passes callback functions down to each item.

```tsx
import { Task } from '../types';
import { TaskItem } from './TaskItem';

interface TaskListProps {
  tasks: Task[];
  onToggle: (id: number, completed: boolean) => Promise<void>;
  onUpdate: (id: number, title: string) => Promise<void>;
  onDelete: (id: number) => Promise<void>;
}

export function TaskList({ tasks, onToggle, onUpdate, onDelete }: TaskListProps) {
  if (tasks.length === 0) {
    return (
      <div className="empty-state">
        <p>No tasks yet. Add your first task above!</p>
      </div>
    );
  }

  const completedCount = tasks.filter((t) => t.completed).length;

  return (
    <div className="task-list">
      <div className="task-list-header">
        <h3>Tasks</h3>
        <span className="task-count">
          {completedCount}/{tasks.length} completed
        </span>
      </div>
      {tasks.map((task) => (
        <TaskItem
          key={task.id}
          task={task}
          onToggle={onToggle}
          onUpdate={onUpdate}
          onDelete={onDelete}
        />
      ))}
    </div>
  );
}
```

---

## Step 7: Build the TaskForm Component

**Where?** `client/src/components/TaskForm.tsx`

A simple form for adding tasks to a project.

```tsx
import { useState } from 'react';

interface TaskFormProps {
  onSubmit: (title: string) => Promise<void>;
}

export function TaskForm({ onSubmit }: TaskFormProps) {
  const [title, setTitle] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (title.trim().length === 0) return;

    setIsSubmitting(true);
    try {
      await onSubmit(title.trim());
      setTitle('');
    } catch (err) {
      console.error('Failed to create task:', err);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="task-form">
      <input
        type="text"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Add a task..."
        maxLength={200}
      />
      <button type="submit" disabled={isSubmitting || title.trim().length === 0}>
        {isSubmitting ? 'Adding...' : 'Add'}
      </button>
    </form>
  );
}
```

---

## Step 8: Build the ProjectDetail Component

**Where?** `client/src/components/ProjectDetail.tsx`

Shows a single project with all its tasks. This component includes edit-in-place for the project name and description, the task form, and the task list.

```tsx
import { useState } from 'react';
import { ProjectWithTasks } from '../types';
import { TaskForm } from './TaskForm';
import { TaskList } from './TaskList';

interface ProjectDetailProps {
  project: ProjectWithTasks;
  onBack: () => void;
  onUpdate: (name: string, description: string) => Promise<void>;
  onCreateTask: (title: string) => Promise<void>;
  onToggleTask: (id: number, completed: boolean) => Promise<void>;
  onUpdateTask: (id: number, title: string) => Promise<void>;
  onDeleteTask: (id: number) => Promise<void>;
}

export function ProjectDetail({
  project,
  onBack,
  onUpdate,
  onCreateTask,
  onToggleTask,
  onUpdateTask,
  onDeleteTask,
}: ProjectDetailProps) {
  const [isEditingName, setIsEditingName] = useState(false);
  const [isEditingDesc, setIsEditingDesc] = useState(false);
  const [editName, setEditName] = useState(project.name);
  const [editDesc, setEditDesc] = useState(project.description);

  const handleSaveName = async () => {
    if (editName.trim().length === 0) {
      setEditName(project.name);
      setIsEditingName(false);
      return;
    }
    if (editName.trim() !== project.name) {
      await onUpdate(editName.trim(), project.description);
    }
    setIsEditingName(false);
  };

  const handleSaveDesc = async () => {
    if (editDesc.trim() !== project.description) {
      await onUpdate(project.name, editDesc.trim());
    }
    setIsEditingDesc(false);
  };

  return (
    <div className="project-detail">
      <button className="back-button" onClick={onBack}>
        ← Back to Projects
      </button>

      <div className="project-header">
        {isEditingName ? (
          <input
            type="text"
            value={editName}
            onChange={(e) => setEditName(e.target.value)}
            onBlur={handleSaveName}
            onKeyDown={(e) => {
              if (e.key === 'Enter') handleSaveName();
              if (e.key === 'Escape') {
                setEditName(project.name);
                setIsEditingName(false);
              }
            }}
            className="edit-name-input"
            autoFocus
            maxLength={100}
          />
        ) : (
          <h2 onDoubleClick={() => setIsEditingName(true)}>{project.name}</h2>
        )}

        {isEditingDesc ? (
          <textarea
            value={editDesc}
            onChange={(e) => setEditDesc(e.target.value)}
            onBlur={handleSaveDesc}
            onKeyDown={(e) => {
              if (e.key === 'Escape') {
                setEditDesc(project.description);
                setIsEditingDesc(false);
              }
            }}
            className="edit-desc-input"
            autoFocus
            rows={2}
          />
        ) : (
          <p
            className="project-description clickable"
            onDoubleClick={() => setIsEditingDesc(true)}
          >
            {project.description || 'No description (double-click to add)'}
          </p>
        )}
      </div>

      <TaskForm onSubmit={onCreateTask} />

      <TaskList
        tasks={project.tasks}
        onToggle={onToggleTask}
        onUpdate={onUpdateTask}
        onDelete={onDeleteTask}
      />
    </div>
  );
}
```

---

> [!TIP]
> **Session Break** — You've built the TaskList, TaskForm, and ProjectDetail components. Save your work and take a break.
> When you return, you'll build the App component that orchestrates everything, add styles, and commit.

---

## Step 9: Build the App Component

**Where?** `client/src/App.tsx`

This is the root component — it manages all state, handles all API calls, and coordinates the views. Same role as Level 1's App, but with more state and more operations.

Open `client/src/App.tsx` and replace its contents:

```tsx
import { useState, useEffect } from 'react';
import { Project, ProjectWithTasks } from './types';
import * as api from './services/api';
import { ProjectForm } from './components/ProjectForm';
import { ProjectList } from './components/ProjectList';
import { ProjectDetail } from './components/ProjectDetail';
import './App.css';

function App() {
  // ─── STATE ──────────────────────────────────────────────────
  const [projects, setProjects] = useState<Project[]>([]);
  const [selectedProject, setSelectedProject] = useState<ProjectWithTasks | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState('');

  // ─── LOAD PROJECTS ON MOUNT ─────────────────────────────────
  useEffect(() => {
    loadProjects();
  }, []);

  async function loadProjects() {
    try {
      setIsLoading(true);
      const data = await api.getProjects();
      setProjects(data);
    } catch (err) {
      setError('Failed to load projects. Is the backend running?');
      console.error(err);
    } finally {
      setIsLoading(false);
    }
  }

  // ─── PROJECT OPERATIONS ─────────────────────────────────────

  async function handleCreateProject(data: { name: string; description?: string }) {
    const newProject = await api.createProject(data);
    setProjects((prev) => [newProject, ...prev]);
  }

  async function handleSelectProject(id: number) {
    try {
      const project = await api.getProject(id);
      setSelectedProject(project);
    } catch (err) {
      console.error('Failed to load project:', err);
    }
  }

  async function handleUpdateProject(name: string, description: string) {
    if (!selectedProject) return;

    const updated = await api.updateProject(selectedProject.id, { name, description });
    setSelectedProject({ ...selectedProject, ...updated });
    setProjects((prev) =>
      prev.map((p) => (p.id === updated.id ? { ...p, ...updated } : p))
    );
  }

  async function handleDeleteProject(id: number) {
    await api.deleteProject(id);
    setProjects((prev) => prev.filter((p) => p.id !== id));
    if (selectedProject?.id === id) {
      setSelectedProject(null);
    }
  }

  // ─── TASK OPERATIONS ────────────────────────────────────────

  async function handleCreateTask(title: string) {
    if (!selectedProject) return;

    const newTask = await api.createTask(selectedProject.id, { title });
    setSelectedProject({
      ...selectedProject,
      tasks: [...selectedProject.tasks, newTask],
    });
  }

  async function handleToggleTask(id: number, completed: boolean) {
    const updated = await api.updateTask(id, { completed });
    if (selectedProject) {
      setSelectedProject({
        ...selectedProject,
        tasks: selectedProject.tasks.map((t) =>
          t.id === updated.id ? updated : t
        ),
      });
    }
  }

  async function handleUpdateTask(id: number, title: string) {
    const updated = await api.updateTask(id, { title });
    if (selectedProject) {
      setSelectedProject({
        ...selectedProject,
        tasks: selectedProject.tasks.map((t) =>
          t.id === updated.id ? updated : t
        ),
      });
    }
  }

  async function handleDeleteTask(id: number) {
    await api.deleteTask(id);
    if (selectedProject) {
      setSelectedProject({
        ...selectedProject,
        tasks: selectedProject.tasks.filter((t) => t.id !== id),
      });
    }
  }

  // ─── RENDER ─────────────────────────────────────────────────
  return (
    <div className="app">
      <header className="app-header">
        <h1 onClick={() => setSelectedProject(null)} style={{ cursor: 'pointer' }}>
          TaskForge
        </h1>
        <p>Project Task Manager</p>
      </header>

      <main className="app-main">
        {isLoading && <p className="loading">Loading projects...</p>}
        {error && <p className="error-message">{error}</p>}

        {!isLoading && !error && (
          <>
            {selectedProject ? (
              <ProjectDetail
                project={selectedProject}
                onBack={() => setSelectedProject(null)}
                onUpdate={handleUpdateProject}
                onCreateTask={handleCreateTask}
                onToggleTask={handleToggleTask}
                onUpdateTask={handleUpdateTask}
                onDeleteTask={handleDeleteTask}
              />
            ) : (
              <>
                <ProjectForm onSubmit={handleCreateProject} />
                <ProjectList
                  projects={projects}
                  onSelect={handleSelectProject}
                  onDelete={handleDeleteProject}
                />
              </>
            )}
          </>
        )}
      </main>
    </div>
  );
}

export default App;
```

### State-Based View Switching

```
┌────────────────────────────────────────────────────────────────────┐
│                          App Component                              │
│                                                                    │
│  State: projects[], selectedProject, isLoading, error              │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  IF selectedProject is NULL:                                │   │
│  │  ┌──────────────┐  ┌────────────────────────────────────┐  │   │
│  │  │ ProjectForm   │  │ ProjectList                        │  │   │
│  │  │               │  │                                    │  │   │
│  │  │ Create new    │  │ Shows all projects                 │  │   │
│  │  │ project       │  │ Click one → setSelectedProject     │  │   │
│  │  └──────────────┘  └────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  IF selectedProject is SET:                                 │   │
│  │  ┌──────────────────────────────────────────────────────┐  │   │
│  │  │ ProjectDetail                                        │  │   │
│  │  │  ← Back button → setSelectedProject(null)            │  │   │
│  │  │  Project name + description (editable)               │  │   │
│  │  │  ┌────────────┐                                      │  │   │
│  │  │  │ TaskForm    │  Add tasks                           │  │   │
│  │  │  └────────────┘                                      │  │   │
│  │  │  ┌────────────────────────────────────────────────┐  │  │   │
│  │  │  │ TaskList                                        │  │  │   │
│  │  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │  │  │   │
│  │  │  │  │ TaskItem  │ │ TaskItem  │ │ TaskItem  │ ...  │  │  │   │
│  │  │  │  └──────────┘ └──────────┘ └──────────┘       │  │  │   │
│  │  │  └────────────────────────────────────────────────┘  │  │   │
│  │  └──────────────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  Data flows DOWN through props. Events flow UP through callbacks.  │
└────────────────────────────────────────────────────────────────────┘
```

### How State Updates Work

When the user creates a task:

```
1. User types in TaskForm and clicks "Add"
   │
2. TaskForm calls onSubmit("Set up database")
   │  (This is handleCreateTask from App)
   │
3. handleCreateTask sends POST to backend
   │  Backend inserts into PostgreSQL
   │  Backend returns the new task object
   │
4. App updates selectedProject.tasks:
   │  setSelectedProject({
   │    ...selectedProject,
   │    tasks: [...selectedProject.tasks, newTask]
   │  })
   │
5. React re-renders ProjectDetail → TaskList → new TaskItem appears
```

**Why state lives in App**: Both `ProjectList` and `ProjectDetail` need to know about projects. The task operations need to update `selectedProject`. By keeping all state in App, there's one source of truth. Same "lifting state up" pattern as Level 1 — just more of it.

---

> [!TIP]
> **Session Break** — You've built the App component with all state management and CRUD operations. Save your work and take a break.
> When you return, you'll add styles, clean up boilerplate, and commit the complete frontend.

---

## Step 10: Add Styles

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

/* ─── APP LAYOUT ─────────────────────────────────────────── */
.app {
  max-width: 700px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

.app-header {
  text-align: center;
  margin-bottom: 2rem;
}

.app-header h1 {
  font-size: 2.5rem;
  color: #38bdf8;
  margin-bottom: 0.25rem;
}

.app-header p {
  color: #94a3b8;
  font-size: 1.1rem;
}

/* ─── PROJECT FORM ───────────────────────────────────────── */
.project-form {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.project-form h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.form-group {
  margin-bottom: 0.75rem;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 0.75rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
  resize: vertical;
}

.form-group input:focus,
.form-group textarea:focus {
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
}

.submit-button:hover {
  background: #7dd3fc;
}

.submit-button:disabled {
  background: #334155;
  color: #64748b;
  cursor: not-allowed;
}

/* ─── PROJECT LIST ───────────────────────────────────────── */
.project-list h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.project-card {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.25rem;
  margin-bottom: 1rem;
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  transition: border-color 0.2s;
  border: 2px solid transparent;
}

.project-card:hover {
  border-color: #334155;
}

.project-card-content {
  flex: 1;
  cursor: pointer;
}

.project-card h3 {
  font-size: 1.1rem;
  margin-bottom: 0.25rem;
}

.project-description {
  color: #94a3b8;
  font-size: 0.9rem;
  margin-bottom: 0.5rem;
}

.project-date {
  font-size: 0.8rem;
  color: #64748b;
}

/* ─── PROJECT DETAIL ─────────────────────────────────────── */
.project-detail {
  /* Container for selected project view */
}

.back-button {
  background: none;
  border: none;
  color: #38bdf8;
  font-size: 1rem;
  cursor: pointer;
  padding: 0.5rem 0;
  margin-bottom: 1rem;
}

.back-button:hover {
  color: #7dd3fc;
}

.project-header {
  background: #1e293b;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.project-header h2 {
  font-size: 1.5rem;
  margin-bottom: 0.5rem;
  cursor: default;
}

.project-header .clickable {
  cursor: pointer;
}

.project-header .clickable:hover {
  color: #cbd5e1;
}

.edit-name-input {
  width: 100%;
  padding: 0.5rem;
  border: 2px solid #38bdf8;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-size: 1.5rem;
  font-weight: 700;
  font-family: inherit;
}

.edit-desc-input {
  width: 100%;
  padding: 0.5rem;
  border: 2px solid #38bdf8;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 0.9rem;
  resize: vertical;
}

/* ─── TASK FORM ──────────────────────────────────────────── */
.task-form {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.task-form input {
  flex: 1;
  padding: 0.75rem;
  border: 2px solid #334155;
  border-radius: 8px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
}

.task-form input:focus {
  outline: none;
  border-color: #38bdf8;
}

.task-form button {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  background: #38bdf8;
  color: #0f172a;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  white-space: nowrap;
}

.task-form button:hover {
  background: #7dd3fc;
}

.task-form button:disabled {
  background: #334155;
  color: #64748b;
  cursor: not-allowed;
}

/* ─── TASK LIST ──────────────────────────────────────────── */
.task-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.task-list-header h3 {
  font-size: 1.1rem;
}

.task-count {
  font-size: 0.85rem;
  color: #94a3b8;
}

/* ─── TASK ITEM ──────────────────────────────────────────── */
.task-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  background: #1e293b;
  border-radius: 8px;
  margin-bottom: 0.5rem;
}

.task-item.completed .task-title {
  text-decoration: line-through;
  color: #64748b;
}

.task-checkbox {
  width: 20px;
  height: 20px;
  accent-color: #38bdf8;
  cursor: pointer;
  flex-shrink: 0;
}

.task-title {
  flex: 1;
  cursor: default;
}

.task-edit-input {
  flex: 1;
  padding: 0.25rem 0.5rem;
  border: 2px solid #38bdf8;
  border-radius: 4px;
  background: #0f172a;
  color: #e2e8f0;
  font-family: inherit;
  font-size: 1rem;
}

/* ─── DELETE BUTTON ──────────────────────────────────────── */
.delete-button {
  background: none;
  border: none;
  color: #64748b;
  font-size: 1.1rem;
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

/* ─── EMPTY STATE ────────────────────────────────────────── */
.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

/* ─── UTILITY ────────────────────────────────────────────── */
.loading {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

.error-message {
  color: #f87171;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}
```

---

## Step 11: Clean Up Boilerplate

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
rm client/src/assets/react.svg
rm client/public/vite.svg
```

Open `client/index.html` and replace its contents:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>TaskForge — Project Task Manager</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Step 12: Commit

> [!IMPORTANT]
> **You should be in:** `task-forge/`

```bash
git add client/
git commit -m "feat: add React frontend with project and task management UI"
```

---

> [!TIP]
> ## Spatial Check-In

1. **How does the app switch between the project list view and the project detail view?**

<details><summary>Answer</summary>

A `selectedProject` state variable in App. When it's `null`, the project list is shown. When it contains a project, the detail view is shown. Clicking a project sets it; clicking "Back" sets it to `null`.

</details>

2. **Why does all state live in the App component instead of in each component?**

<details><summary>Answer</summary>

Multiple components need the same data (projects, tasks). By keeping state in App (their shared parent), there's one source of truth. This is "lifting state up" — the same pattern as Level 1.

</details>

3. **How does the edit-in-place pattern work?**

<details><summary>Answer</summary>

An `isEditing` state toggles between a display `<span>` and an editable `<input>`. Double-click activates editing. Enter or blur saves. Escape cancels.

</details>

4. **Why do we use `e.stopPropagation()` on the delete button?**

<details><summary>Answer</summary>

The delete button is inside a clickable card. Without `stopPropagation`, clicking delete would also trigger the card's click handler (selecting the project). It prevents the event from bubbling up to the parent.

</details>

---

| | | |
|:---|:---:|---:|
| [← 04 — Building the Backend](../04-backend/) | [Level 2 Overview](../) | [06 — Deployment →](../06-deployment/) |
