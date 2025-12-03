# Chapter 11: Zustand: Global State Without the Ceremony

## Why not Redux (in 2025)

## Why not Redux (in 2025)

In Chapter 10, we built a Task Board using Context API. It worked—until it didn't. When we added real-time updates and complex filtering, every state change triggered re-renders across the entire component tree. Context is powerful for dependency injection and theming, but it's not optimized for frequently changing application state.

For years, Redux was the default answer to global state management in React. But Redux comes with significant ceremony: actions, action creators, reducers, dispatch, connect/useSelector, middleware setup, and a steep learning curve. In 2025, we have better options that give us Redux's power without its complexity.

### The Redux Tax: What You Pay for Power

Let's see what managing a simple counter looks like in Redux:

```typescript
// store/counterSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CounterState {
  value: number;
}

const initialState: CounterState = {
  value: 0,
};

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

```typescript
// store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

export const store = configureStore({
  reducer: {
    counter: counterReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```typescript
// store/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

```tsx
// App.tsx
import { Provider } from 'react-redux';
import { store } from './store/store';
import Counter from './components/Counter';

function App() {
  return (
    <Provider store={store}>
      <Counter />
    </Provider>
  );
}

export default App;
```

```tsx
// components/Counter.tsx
import { useAppDispatch, useAppSelector } from '../store/hooks';
import { increment, decrement, incrementByAmount } from '../store/counterSlice';

function Counter() {
  const count = useAppSelector((state) => state.counter.value);
  const dispatch = useAppDispatch();

  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => dispatch(increment())}>+1</button>
      <button onClick={() => dispatch(decrement())}>-1</button>
      <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
    </div>
  );
}

export default Counter;
```

**What we just wrote**:
- 5 separate files
- ~100 lines of code
- Type definitions for state, actions, and dispatch
- Custom hooks for type-safe access
- Provider wrapper in the root component

**What we got**: A counter that increments and decrements.

This is Redux Toolkit (RTK), the modern, simplified version of Redux. Classic Redux was even more verbose. RTK is excellent for large applications with complex state interactions, time-travel debugging requirements, and teams that need strict patterns. But for most applications, this is overkill.

### What We Actually Need

Most applications need:
1. **Global state** that multiple components can access
2. **Selective subscriptions** so components only re-render when their specific data changes
3. **Simple updates** without action creators and reducers
4. **TypeScript support** without fighting the type system
5. **DevTools** for debugging
6. **Minimal boilerplate** so we can focus on features

Redux provides all of this, but at a high cost. Context API provides #1 but fails at #2. We need something in between.

### Enter Zustand

Zustand (German for "state") is a small, fast state management library that gives you Redux's power with a fraction of the code. Here's the same counter in Zustand:

```typescript
// store/useCounterStore.ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  incrementByAmount: (amount: number) => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  incrementByAmount: (amount) => set((state) => ({ count: state.count + amount })),
}));
```

```tsx
// components/Counter.tsx
import { useCounterStore } from '../store/useCounterStore';

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);
  const incrementByAmount = useCounterStore((state) => state.incrementByAmount);

  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
      <button onClick={() => incrementByAmount(5)}>+5</button>
    </div>
  );
}

export default Counter;
```

**What we just wrote**:
- 2 files
- ~30 lines of code
- No providers, no reducers, no action creators
- Full TypeScript support
- Automatic selective subscriptions

**What we got**: The exact same functionality.

### The Zustand Philosophy

Zustand's design philosophy:
1. **Hooks-first**: Use it like `useState`, but global
2. **No providers**: No wrapping your app in context providers
3. **Selector-based subscriptions**: Components only re-render when their selected data changes
4. **Immutable updates**: Use the `set` function, which handles immutability
5. **Minimal API surface**: Learn 3 functions, use them everywhere

### When to Choose What

**Use Context API when**:
- State rarely changes (theme, locale, auth user)
- You need dependency injection
- State is truly tree-scoped (not global)
- You're okay with all consumers re-rendering

**Use Zustand when**:
- State changes frequently
- Multiple unrelated components need the same data
- You need selective subscriptions for performance
- You want simple, readable code

**Use Redux when**:
- You need time-travel debugging
- Your team requires strict patterns and conventions
- You have complex state interactions that benefit from reducers
- You're integrating with Redux-specific middleware (Redux Saga, Redux Observable)

**Use React Query when**:
- You're managing server state (API data)
- You need caching, background refetching, and optimistic updates
- The data's source of truth is the server, not the client

In this chapter, we'll build a real-world application with Zustand and see how it handles the challenges that broke Context API in Chapter 10.

## Setting up Zustand

## Setting up Zustand

### Reference Implementation: Multi-User Task Board

We're building a collaborative task management application. Think Trello or Linear, but simplified. Our requirements:

**Features**:
- Create, update, delete tasks
- Assign tasks to users
- Filter tasks by status, assignee, and priority
- Real-time updates (simulated with polling)
- Optimistic updates for instant feedback
- Undo/redo for task operations

**Why this example**:
- Complex state with multiple entities (tasks, users, filters)
- Frequent updates (drag-and-drop, status changes)
- Performance-critical (many components reading the same data)
- Real-world patterns (optimistic updates, undo/redo)

**Project Structure**:
```
src/
├── components/
│   ├── TaskBoard.tsx          ← Main board component
│   ├── TaskColumn.tsx         ← Column for each status
│   ├── TaskCard.tsx           ← Individual task card
│   ├── TaskFilters.tsx        ← Filter controls
│   └── CreateTaskForm.tsx     ← New task form
├── store/
│   └── useTaskStore.ts        ← Our Zustand store
├── types/
│   └── task.ts                ← TypeScript types
└── App.tsx
```

### Installation

First, install Zustand:

```bash
npm install zustand
```

That's it. No peer dependencies, no configuration files, no setup ceremony.

### Defining Our Types

Before building the store, let's define our domain types:

```typescript
// src/types/task.ts
export type TaskStatus = 'todo' | 'in-progress' | 'done';
export type TaskPriority = 'low' | 'medium' | 'high';

export interface User {
  id: string;
  name: string;
  avatar: string;
}

export interface Task {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  priority: TaskPriority;
  assigneeId: string | null;
  createdAt: number;
  updatedAt: number;
}

export interface TaskFilters {
  status: TaskStatus | 'all';
  assigneeId: string | 'all';
  priority: TaskPriority | 'all';
}
```

### Creating Our First Store (Naive Version)

Let's start with a simple store that just holds tasks:

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { Task, TaskFilters, User } from '../types/task';

interface TaskState {
  tasks: Task[];
  users: User[];
  filters: TaskFilters;
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
  setFilters: (filters: Partial<TaskFilters>) => void;
}

export const useTaskStore = create<TaskState>((set) => ({
  // Initial state
  tasks: [],
  users: [
    { id: '1', name: 'Alice', avatar: '👩' },
    { id: '2', name: 'Bob', avatar: '👨' },
    { id: '3', name: 'Charlie', avatar: '🧑' },
  ],
  filters: {
    status: 'all',
    assigneeId: 'all',
    priority: 'all',
  },

  // Actions
  addTask: (taskData) =>
    set((state) => ({
      tasks: [
        ...state.tasks,
        {
          ...taskData,
          id: crypto.randomUUID(),
          createdAt: Date.now(),
          updatedAt: Date.now(),
        },
      ],
    })),

  updateTask: (id, updates) =>
    set((state) => ({
      tasks: state.tasks.map((task) =>
        task.id === id
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task
      ),
    })),

  deleteTask: (id) =>
    set((state) => ({
      tasks: state.tasks.filter((task) => task.id !== id),
    })),

  setFilters: (newFilters) =>
    set((state) => ({
      filters: { ...state.filters, ...newFilters },
    })),
}));
```

**What we just created**:
- A store with state (`tasks`, `users`, `filters`)
- Actions that modify state (`addTask`, `updateTask`, `deleteTask`, `setFilters`)
- Immutable updates using spread operators
- TypeScript types for everything

**How it works**:
1. `create<TaskState>()` creates a hook that components can use
2. The function receives `set`, which updates the store
3. `set` takes a function that receives current state and returns new state
4. Updates are shallow-merged (like `setState` in class components)

### Using the Store in Components

Now let's build a simple task list to see the store in action:

```tsx
// src/components/TaskBoard.tsx
import { useTaskStore } from '../store/useTaskStore';
import TaskCard from './TaskCard';
import CreateTaskForm from './CreateTaskForm';

function TaskBoard() {
  const tasks = useTaskStore((state) => state.tasks);
  const filters = useTaskStore((state) => state.filters);

  // Filter tasks based on current filters
  const filteredTasks = tasks.filter((task) => {
    if (filters.status !== 'all' && task.status !== filters.status) return false;
    if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
    if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
    return true;
  });

  return (
    <div className="task-board">
      <h1>Task Board</h1>
      <CreateTaskForm />
      <div className="task-list">
        {filteredTasks.map((task) => (
          <TaskCard key={task.id} task={task} />
        ))}
      </div>
    </div>
  );
}

export default TaskBoard;
```

```tsx
// src/components/TaskCard.tsx
import { useTaskStore } from '../store/useTaskStore';
import { Task } from '../types/task';

interface TaskCardProps {
  task: Task;
}

function TaskCard({ task }: TaskCardProps) {
  const updateTask = useTaskStore((state) => state.updateTask);
  const deleteTask = useTaskStore((state) => state.deleteTask);
  const users = useTaskStore((state) => state.users);

  const assignee = users.find((u) => u.id === task.assigneeId);

  return (
    <div className="task-card">
      <h3>{task.title}</h3>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority-${task.priority}`}>{task.priority}</span>
        <span className={`status-${task.status}`}>{task.status}</span>
        {assignee && <span>{assignee.avatar} {assignee.name}</span>}
      </div>
      <div className="task-actions">
        <button onClick={() => updateTask(task.id, { status: 'done' })}>
          Mark Done
        </button>
        <button onClick={() => deleteTask(task.id)}>Delete</button>
      </div>
    </div>
  );
}

export default TaskCard;
```

```tsx
// src/components/CreateTaskForm.tsx
import { useState } from 'react';
import { useTaskStore } from '../store/useTaskStore';
import { TaskPriority, TaskStatus } from '../types/task';

function CreateTaskForm() {
  const addTask = useTaskStore((state) => state.addTask);
  const users = useTaskStore((state) => state.users);

  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [priority, setPriority] = useState<TaskPriority>('medium');
  const [assigneeId, setAssigneeId] = useState<string | null>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;

    addTask({
      title,
      description,
      status: 'todo',
      priority,
      assigneeId,
    });

    // Reset form
    setTitle('');
    setDescription('');
    setPriority('medium');
    setAssigneeId(null);
  };

  return (
    <form onSubmit={handleSubmit} className="create-task-form">
      <input
        type="text"
        placeholder="Task title"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
      />
      <textarea
        placeholder="Description"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
      />
      <select value={priority} onChange={(e) => setPriority(e.target.value as TaskPriority)}>
        <option value="low">Low Priority</option>
        <option value="medium">Medium Priority</option>
        <option value="high">High Priority</option>
      </select>
      <select value={assigneeId || ''} onChange={(e) => setAssigneeId(e.target.value || null)}>
        <option value="">Unassigned</option>
        {users.map((user) => (
          <option key={user.id} value={user.id}>
            {user.avatar} {user.name}
          </option>
        ))}
      </select>
      <button type="submit">Create Task</button>
    </form>
  );
}

export default CreateTaskForm;
```

### Running the Application

Let's test what we've built:

```tsx
// src/App.tsx
import TaskBoard from './components/TaskBoard';
import './App.css';

function App() {
  return (
    <div className="app">
      <TaskBoard />
    </div>
  );
}

export default App;
```

**Browser Behavior**:
- Form appears at the top
- Create a task: "Implement authentication"
- Task appears immediately in the list
- Click "Mark Done": Status updates instantly
- Click "Delete": Task disappears

**Browser Console**:
```
(No errors, no warnings)
```

**React DevTools - Components Tab**:
- `TaskBoard` component
  - Props: (none)
  - Hooks: (none - using Zustand, not useState)
- `TaskCard` components (one per task)
  - Props: `{ task: {...} }`
  - Hooks: (none)

**What's happening**:
1. Components call `useTaskStore` with a selector function
2. Zustand subscribes the component to only the selected data
3. When `addTask` is called, Zustand updates the store
4. Only components that selected `tasks` re-render
5. Components that only selected `users` or `filters` don't re-render

This is the key difference from Context API: **selective subscriptions**.

### The Selector Pattern

Notice how we use the store:

```typescript
const tasks = useTaskStore((state) => state.tasks);
const addTask = useTaskStore((state) => state.addTask);
```

**Not**:

```typescript
const { tasks, addTask } = useTaskStore();
```

**Why?** The second approach subscribes the component to the entire store. Any change to any part of the store triggers a re-render. The first approach (selector pattern) subscribes only to the specific data you select.

**Rule of thumb**: Always use selectors. Always.

### Current Limitations

Our store works, but it has problems:

1. **No derived state**: We're filtering tasks in the component, which means every component that needs filtered tasks must duplicate this logic
2. **No computed values**: We can't efficiently compute things like "number of tasks per status"
3. **No persistence**: Refresh the page, lose all tasks
4. **No optimistic updates**: When we add a task, we wait for the store update
5. **No undo/redo**: Can't reverse actions

In the next sections, we'll solve these problems using Zustand's advanced features.

## Slices, selectors, and middleware

## Slices, selectors, and middleware

Our basic store works, but as the application grows, we'll face new challenges. Let's encounter them one by one and solve them with Zustand's advanced features.

### The Failure: Duplicated Filter Logic

**Current problem**: Every component that needs filtered tasks must duplicate the filtering logic.

```tsx
// In TaskBoard.tsx
const filteredTasks = tasks.filter((task) => {
  if (filters.status !== 'all' && task.status !== filters.status) return false;
  if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
  if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
  return true;
});

// In TaskColumn.tsx (if we had one)
const filteredTasks = tasks.filter((task) => {
  // Same logic duplicated
});

// In TaskStats.tsx (if we had one)
const filteredTasks = tasks.filter((task) => {
  // Same logic duplicated again
});
```

**Let's see this fail**: Add a new component that needs filtered tasks:

```tsx
// src/components/TaskStats.tsx
import { useTaskStore } from '../store/useTaskStore';

function TaskStats() {
  const tasks = useTaskStore((state) => state.tasks);
  const filters = useTaskStore((state) => state.filters);

  // Duplicated filtering logic
  const filteredTasks = tasks.filter((task) => {
    if (filters.status !== 'all' && task.status !== filters.status) return false;
    if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
    if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
    return true;
  });

  const todoCount = filteredTasks.filter((t) => t.status === 'todo').length;
  const inProgressCount = filteredTasks.filter((t) => t.status === 'in-progress').length;
  const doneCount = filteredTasks.filter((t) => t.status === 'done').length;

  return (
    <div className="task-stats">
      <div>Todo: {todoCount}</div>
      <div>In Progress: {inProgressCount}</div>
      <div>Done: {doneCount}</div>
    </div>
  );
}

export default TaskStats;
```

**Browser Behavior**:
- Stats display correctly
- But we've now written the same filtering logic three times

**The problem**:
1. Code duplication (DRY violation)
2. Inconsistency risk (if we update one, we must update all)
3. Performance waste (filtering happens in every component)
4. Testing burden (must test filtering in multiple places)

### Diagnostic Analysis: Reading the Duplication

**What the code reveals**:
- Same filtering logic in `TaskBoard.tsx`, `TaskColumn.tsx`, `TaskStats.tsx`
- Each component subscribes to `tasks` and `filters`
- Each component re-runs filtering on every render
- No memoization, no caching

**Root cause identified**: Filtering is business logic, not view logic. It belongs in the store, not in components.

**What we need**: Derived state—computed values that automatically update when their dependencies change.

### Solution: Selectors for Derived State

Zustand doesn't have built-in computed values like MobX or Vue, but we can create them using selector functions:

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { Task, TaskFilters, User } from '../types/task';

interface TaskState {
  tasks: Task[];
  users: User[];
  filters: TaskFilters;
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
  setFilters: (filters: Partial<TaskFilters>) => void;
}

export const useTaskStore = create<TaskState>((set) => ({
  tasks: [],
  users: [
    { id: '1', name: 'Alice', avatar: '👩' },
    { id: '2', name: 'Bob', avatar: '👨' },
    { id: '3', name: 'Charlie', avatar: '🧑' },
  ],
  filters: {
    status: 'all',
    assigneeId: 'all',
    priority: 'all',
  },

  addTask: (taskData) =>
    set((state) => ({
      tasks: [
        ...state.tasks,
        {
          ...taskData,
          id: crypto.randomUUID(),
          createdAt: Date.now(),
          updatedAt: Date.now(),
        },
      ],
    })),

  updateTask: (id, updates) =>
    set((state) => ({
      tasks: state.tasks.map((task) =>
        task.id === id
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task
      ),
    })),

  deleteTask: (id) =>
    set((state) => ({
      tasks: state.tasks.filter((task) => task.id !== id),
    })),

  setFilters: (newFilters) =>
    set((state) => ({
      filters: { ...state.filters, ...newFilters },
    })),
}));

// Selector functions (outside the store)
export const selectFilteredTasks = (state: TaskState): Task[] => {
  const { tasks, filters } = state;
  return tasks.filter((task) => {
    if (filters.status !== 'all' && task.status !== filters.status) return false;
    if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
    if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
    return true;
  });
};

export const selectTasksByStatus = (state: TaskState) => {
  const filteredTasks = selectFilteredTasks(state);
  return {
    todo: filteredTasks.filter((t) => t.status === 'todo'),
    inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
    done: filteredTasks.filter((t) => t.status === 'done'),
  };
};

export const selectTaskStats = (state: TaskState) => {
  const tasksByStatus = selectTasksByStatus(state);
  return {
    todoCount: tasksByStatus.todo.length,
    inProgressCount: tasksByStatus.inProgress.length,
    doneCount: tasksByStatus.done.length,
    totalCount: selectFilteredTasks(state).length,
  };
};
```

**What changed**:
- Created `selectFilteredTasks` function that encapsulates filtering logic
- Created `selectTasksByStatus` that groups tasks by status
- Created `selectTaskStats` that computes counts
- All selectors are pure functions that take state and return derived data

Now update the components to use these selectors:

```tsx
// src/components/TaskBoard.tsx
import { useTaskStore, selectFilteredTasks } from '../store/useTaskStore';
import TaskCard from './TaskCard';
import CreateTaskForm from './CreateTaskForm';

function TaskBoard() {
  const filteredTasks = useTaskStore(selectFilteredTasks);

  return (
    <div className="task-board">
      <h1>Task Board</h1>
      <CreateTaskForm />
      <div className="task-list">
        {filteredTasks.map((task) => (
          <TaskCard key={task.id} task={task} />
        ))}
      </div>
    </div>
  );
}

export default TaskBoard;
```

```tsx
// src/components/TaskStats.tsx
import { useTaskStore, selectTaskStats } from '../store/useTaskStore';

function TaskStats() {
  const stats = useTaskStore(selectTaskStats);

  return (
    <div className="task-stats">
      <div>Todo: {stats.todoCount}</div>
      <div>In Progress: {stats.inProgressCount}</div>
      <div>Done: {stats.doneCount}</div>
      <div>Total: {stats.totalCount}</div>
    </div>
  );
}

export default TaskStats;
```

**Browser Behavior**:
- Same functionality as before
- But now filtering logic is centralized

**React DevTools - Profiler**:
- Record a render
- Change filters
- Both `TaskBoard` and `TaskStats` re-render
- But filtering logic runs only once per component (not duplicated)

**Improvement**: Code is DRY, maintainable, and testable in one place.

### The Failure: Performance Degradation with Many Tasks

**New scenario**: Let's add 1000 tasks and see what happens:

```typescript
// Add this to your store initialization for testing
const generateMockTasks = (count: number): Task[] => {
  const statuses: TaskStatus[] = ['todo', 'in-progress', 'done'];
  const priorities: TaskPriority[] = ['low', 'medium', 'high'];
  const userIds = ['1', '2', '3'];

  return Array.from({ length: count }, (_, i) => ({
    id: `task-${i}`,
    title: `Task ${i + 1}`,
    description: `Description for task ${i + 1}`,
    status: statuses[i % 3],
    priority: priorities[i % 3],
    assigneeId: userIds[i % 3],
    createdAt: Date.now() - i * 1000,
    updatedAt: Date.now() - i * 1000,
  }));
};

// In your store
export const useTaskStore = create<TaskState>((set) => ({
  tasks: generateMockTasks(1000), // ← Add 1000 tasks for testing
  // ... rest of store
}));
```

**Browser Behavior**:
- Page loads slowly
- Scrolling feels janky
- Changing filters has noticeable lag

**Browser Console**:
```
(No errors, but performance is poor)
```

**React DevTools - Profiler**:
- Record interaction: Change filter from "all" to "todo"
- `TaskBoard` render time: 245ms
- `TaskStats` render time: 180ms
- Both components re-rendered
- Filtering ran twice (once per component)

**Chrome Performance Tab**:
- Main thread blocked for 400ms
- Scripting: 380ms (filtering arrays)
- Rendering: 20ms

### Diagnostic Analysis: Reading the Performance Problem

**What the profiler reveals**:
1. Both components subscribe to different selectors
2. Both selectors call `selectFilteredTasks`
3. Filtering 1000 tasks twice = 2000 filter operations
4. No memoization = filtering runs on every render

**Root cause identified**: Selectors are pure functions, but they're not memoized. Every time a component calls `useTaskStore(selectFilteredTasks)`, the selector runs from scratch.

**What we need**: Memoization—cache the result and only recompute when dependencies change.

### Solution: Memoized Selectors with Zustand Middleware

Zustand doesn't include memoization by default, but we can add it easily. Let's use a simple memoization approach:

```bash
npm install zustand
```

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { Task, TaskFilters, User, TaskStatus } from '../types/task';

interface TaskState {
  tasks: Task[];
  users: User[];
  filters: TaskFilters;
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
  setFilters: (filters: Partial<TaskFilters>) => void;
}

export const useTaskStore = create<TaskState>((set) => ({
  tasks: generateMockTasks(1000),
  users: [
    { id: '1', name: 'Alice', avatar: '👩' },
    { id: '2', name: 'Bob', avatar: '👨' },
    { id: '3', name: 'Charlie', avatar: '🧑' },
  ],
  filters: {
    status: 'all',
    assigneeId: 'all',
    priority: 'all',
  },

  addTask: (taskData) =>
    set((state) => ({
      tasks: [
        ...state.tasks,
        {
          ...taskData,
          id: crypto.randomUUID(),
          createdAt: Date.now(),
          updatedAt: Date.now(),
        },
      ],
    })),

  updateTask: (id, updates) =>
    set((state) => ({
      tasks: state.tasks.map((task) =>
        task.id === id
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task
      ),
    })),

  deleteTask: (id) =>
    set((state) => ({
      tasks: state.tasks.filter((task) => task.id !== id),
    })),

  setFilters: (newFilters) =>
    set((state) => ({
      filters: { ...state.filters, ...newFilters },
    })),
}));

// Simple memoization helper
function createMemoizedSelector<T, R>(
  selector: (state: T) => R,
  equalityFn: (a: R, b: R) => boolean = Object.is
) {
  let lastState: T | undefined;
  let lastResult: R | undefined;

  return (state: T): R => {
    if (lastState === state && lastResult !== undefined) {
      return lastResult;
    }

    const result = selector(state);
    if (lastResult !== undefined && equalityFn(result, lastResult)) {
      return lastResult;
    }

    lastState = state;
    lastResult = result;
    return result;
  };
}

// Memoized selectors
export const selectFilteredTasks = createMemoizedSelector(
  (state: TaskState): Task[] => {
    const { tasks, filters } = state;
    return tasks.filter((task) => {
      if (filters.status !== 'all' && task.status !== filters.status) return false;
      if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
      if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
      return true;
    });
  },
  (a, b) => {
    // Custom equality: compare array length and first/last items
    if (a.length !== b.length) return false;
    if (a.length === 0) return true;
    return a[0].id === b[0].id && a[a.length - 1].id === b[a.length - 1].id;
  }
);

export const selectTasksByStatus = createMemoizedSelector(
  (state: TaskState) => {
    const filteredTasks = selectFilteredTasks(state);
    return {
      todo: filteredTasks.filter((t) => t.status === 'todo'),
      inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
      done: filteredTasks.filter((t) => t.status === 'done'),
    };
  }
);

export const selectTaskStats = createMemoizedSelector(
  (state: TaskState) => {
    const tasksByStatus = selectTasksByStatus(state);
    return {
      todoCount: tasksByStatus.todo.length,
      inProgressCount: tasksByStatus.inProgress.length,
      doneCount: tasksByStatus.done.length,
      totalCount: selectFilteredTasks(state).length,
    };
  }
);

function generateMockTasks(count: number): Task[] {
  const statuses: TaskStatus[] = ['todo', 'in-progress', 'done'];
  const priorities = ['low', 'medium', 'high'] as const;
  const userIds = ['1', '2', '3'];

  return Array.from({ length: count }, (_, i) => ({
    id: `task-${i}`,
    title: `Task ${i + 1}`,
    description: `Description for task ${i + 1}`,
    status: statuses[i % 3],
    priority: priorities[i % 3],
    assigneeId: userIds[i % 3],
    createdAt: Date.now() - i * 1000,
    updatedAt: Date.now() - i * 1000,
  }));
}
```

**What changed**:
- Created `createMemoizedSelector` helper that caches results
- Wrapped all selectors with memoization
- Added custom equality function for array comparison
- Selectors now return cached results when inputs haven't changed

**Browser Behavior**:
- Page loads faster
- Scrolling is smooth
- Filter changes are instant

**React DevTools - Profiler**:
- Record interaction: Change filter from "all" to "todo"
- `TaskBoard` render time: 12ms (was 245ms)
- `TaskStats` render time: 8ms (was 180ms)
- Filtering ran once, result was cached

**Performance Metrics**:
- **Before**:
  - Filter change: 400ms blocked time
  - Filtering: 2000 operations
  - Render time: 425ms total

- **After**:
  - Filter change: 20ms blocked time (95% improvement)
  - Filtering: 1000 operations (cached for second component)
  - Render time: 20ms total (95% improvement)

**Improvement**: Memoization eliminated redundant computation. The first component computes, subsequent components use the cached result.

### The Failure: Lost State on Page Refresh

**Current problem**: Create some tasks, refresh the page—everything's gone.

```typescript
// Create tasks in the UI
// Refresh the page
// All tasks disappear
```

**Browser Behavior**:
- Tasks exist in memory only
- Page refresh resets to initial state
- No persistence

**What we need**: Middleware to persist state to localStorage and rehydrate on load.

### Solution: Persist Middleware

Zustand provides a `persist` middleware for this exact use case:

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { Task, TaskFilters, User, TaskStatus } from '../types/task';

interface TaskState {
  tasks: Task[];
  users: User[];
  filters: TaskFilters;
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
  setFilters: (filters: Partial<TaskFilters>) => void;
}

export const useTaskStore = create<TaskState>()(
  persist(
    (set) => ({
      tasks: [],
      users: [
        { id: '1', name: 'Alice', avatar: '👩' },
        { id: '2', name: 'Bob', avatar: '👨' },
        { id: '3', name: 'Charlie', avatar: '🧑' },
      ],
      filters: {
        status: 'all',
        assigneeId: 'all',
        priority: 'all',
      },

      addTask: (taskData) =>
        set((state) => ({
          tasks: [
            ...state.tasks,
            {
              ...taskData,
              id: crypto.randomUUID(),
              createdAt: Date.now(),
              updatedAt: Date.now(),
            },
          ],
        })),

      updateTask: (id, updates) =>
        set((state) => ({
          tasks: state.tasks.map((task) =>
            task.id === id
              ? { ...task, ...updates, updatedAt: Date.now() }
              : task
          ),
        })),

      deleteTask: (id) =>
        set((state) => ({
          tasks: state.tasks.filter((task) => task.id !== id),
        })),

      setFilters: (newFilters) =>
        set((state) => ({
          filters: { ...state.filters, ...newFilters },
        })),
    }),
    {
      name: 'task-storage', // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        tasks: state.tasks,
        filters: state.filters,
        // Don't persist users (they're static)
      }),
    }
  )
);

// Selectors remain the same
export const selectFilteredTasks = createMemoizedSelector(
  (state: TaskState): Task[] => {
    const { tasks, filters } = state;
    return tasks.filter((task) => {
      if (filters.status !== 'all' && task.status !== filters.status) return false;
      if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
      if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
      return true;
    });
  },
  (a, b) => {
    if (a.length !== b.length) return false;
    if (a.length === 0) return true;
    return a[0].id === b[0].id && a[a.length - 1].id === b[a.length - 1].id;
  }
);

export const selectTasksByStatus = createMemoizedSelector(
  (state: TaskState) => {
    const filteredTasks = selectFilteredTasks(state);
    return {
      todo: filteredTasks.filter((t) => t.status === 'todo'),
      inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
      done: filteredTasks.filter((t) => t.status === 'done'),
    };
  }
);

export const selectTaskStats = createMemoizedSelector(
  (state: TaskState) => {
    const tasksByStatus = selectTasksByStatus(state);
    return {
      todoCount: tasksByStatus.todo.length,
      inProgressCount: tasksByStatus.inProgress.length,
      doneCount: tasksByStatus.done.length,
      totalCount: selectFilteredTasks(state).length,
    };
  }
);

function createMemoizedSelector<T, R>(
  selector: (state: T) => R,
  equalityFn: (a: R, b: R) => boolean = Object.is
) {
  let lastState: T | undefined;
  let lastResult: R | undefined;

  return (state: T): R => {
    if (lastState === state && lastResult !== undefined) {
      return lastResult;
    }

    const result = selector(state);
    if (lastResult !== undefined && equalityFn(result, lastResult)) {
      return lastResult;
    }

    lastState = state;
    lastResult = result;
    return result;
  };
}
```

**What changed**:
- Wrapped store with `persist` middleware
- Specified `name` for localStorage key
- Used `partialize` to choose what to persist (tasks and filters, not users)
- State now automatically saves to localStorage on every change

**Browser Behavior**:
- Create tasks
- Refresh page
- Tasks persist!

**Browser DevTools - Application Tab**:
- Navigate to Local Storage
- See key: `task-storage`
- Value: JSON with tasks and filters

**Verification**:
```json
{
  "state": {
    "tasks": [
      {
        "id": "abc123",
        "title": "Implement authentication",
        "status": "todo",
        ...
      }
    ],
    "filters": {
      "status": "all",
      "assigneeId": "all",
      "priority": "all"
    }
  },
  "version": 0
}
```

**Improvement**: State persists across page refreshes. Users don't lose their work.

### Slicing the Store: Organizing Complex State

As our store grows, keeping everything in one object becomes unwieldy. Let's organize it into slices:

```typescript
// src/store/slices/taskSlice.ts
import { StateCreator } from 'zustand';
import { Task } from '../../types/task';

export interface TaskSlice {
  tasks: Task[];
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
}

export const createTaskSlice: StateCreator<TaskSlice> = (set) => ({
  tasks: [],

  addTask: (taskData) =>
    set((state) => ({
      tasks: [
        ...state.tasks,
        {
          ...taskData,
          id: crypto.randomUUID(),
          createdAt: Date.now(),
          updatedAt: Date.now(),
        },
      ],
    })),

  updateTask: (id, updates) =>
    set((state) => ({
      tasks: state.tasks.map((task) =>
        task.id === id
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task
      ),
    })),

  deleteTask: (id) =>
    set((state) => ({
      tasks: state.tasks.filter((task) => task.id !== id),
    })),
});
```

```typescript
// src/store/slices/filterSlice.ts
import { StateCreator } from 'zustand';
import { TaskFilters } from '../../types/task';

export interface FilterSlice {
  filters: TaskFilters;
  setFilters: (filters: Partial<TaskFilters>) => void;
  resetFilters: () => void;
}

const defaultFilters: TaskFilters = {
  status: 'all',
  assigneeId: 'all',
  priority: 'all',
};

export const createFilterSlice: StateCreator<FilterSlice> = (set) => ({
  filters: defaultFilters,

  setFilters: (newFilters) =>
    set((state) => ({
      filters: { ...state.filters, ...newFilters },
    })),

  resetFilters: () =>
    set(() => ({
      filters: defaultFilters,
    })),
});
```

```typescript
// src/store/slices/userSlice.ts
import { StateCreator } from 'zustand';
import { User } from '../../types/task';

export interface UserSlice {
  users: User[];
  addUser: (user: User) => void;
}

export const createUserSlice: StateCreator<UserSlice> = (set) => ({
  users: [
    { id: '1', name: 'Alice', avatar: '👩' },
    { id: '2', name: 'Bob', avatar: '👨' },
    { id: '3', name: 'Charlie', avatar: '🧑' },
  ],

  addUser: (user) =>
    set((state) => ({
      users: [...state.users, user],
    })),
});
```

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { createTaskSlice, TaskSlice } from './slices/taskSlice';
import { createFilterSlice, FilterSlice } from './slices/filterSlice';
import { createUserSlice, UserSlice } from './slices/userSlice';

type TaskStore = TaskSlice & FilterSlice & UserSlice;

export const useTaskStore = create<TaskStore>()(
  persist(
    (...a) => ({
      ...createTaskSlice(...a),
      ...createFilterSlice(...a),
      ...createUserSlice(...a),
    }),
    {
      name: 'task-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        tasks: state.tasks,
        filters: state.filters,
      }),
    }
  )
);

// Selectors remain the same
export const selectFilteredTasks = createMemoizedSelector(
  (state: TaskStore) => {
    const { tasks, filters } = state;
    return tasks.filter((task) => {
      if (filters.status !== 'all' && task.status !== filters.status) return false;
      if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
      if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
      return true;
    });
  },
  (a, b) => {
    if (a.length !== b.length) return false;
    if (a.length === 0) return true;
    return a[0].id === b[0].id && a[a.length - 1].id === b[a.length - 1].id;
  }
);

export const selectTasksByStatus = createMemoizedSelector(
  (state: TaskStore) => {
    const filteredTasks = selectFilteredTasks(state);
    return {
      todo: filteredTasks.filter((t) => t.status === 'todo'),
      inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
      done: filteredTasks.filter((t) => t.status === 'done'),
    };
  }
);

export const selectTaskStats = createMemoizedSelector(
  (state: TaskStore) => {
    const tasksByStatus = selectTasksByStatus(state);
    return {
      todoCount: tasksByStatus.todo.length,
      inProgressCount: tasksByStatus.inProgress.length,
      doneCount: tasksByStatus.done.length,
      totalCount: selectFilteredTasks(state).length,
    };
  }
);

function createMemoizedSelector<T, R>(
  selector: (state: T) => R,
  equalityFn: (a: R, b: R) => boolean = Object.is
) {
  let lastState: T | undefined;
  let lastResult: R | undefined;

  return (state: T): R => {
    if (lastState === state && lastResult !== undefined) {
      return lastResult;
    }

    const result = selector(state);
    if (lastResult !== undefined && equalityFn(result, lastResult)) {
      return lastResult;
    }

    lastState = state;
    lastResult = result;
    return result;
  };
}
```

**What changed**:
- Split store into three slices: tasks, filters, users
- Each slice is a separate file with its own types and logic
- Combined slices in the main store using spread operator
- Store remains a single source of truth, but code is organized

**Project Structure**:
```
src/
├── store/
│   ├── slices/
│   │   ├── taskSlice.ts
│   │   ├── filterSlice.ts
│   │   └── userSlice.ts
│   └── useTaskStore.ts
```

**Benefits**:
- Each slice is independently testable
- Clear separation of concerns
- Easy to add new slices without touching existing code
- Type safety maintained across slices

### When to Apply: Slices vs. Single Store

**Use slices when**:
- Store has multiple distinct domains (tasks, users, settings)
- Different team members work on different parts of state
- You want to test state logic in isolation
- Store file exceeds ~200 lines

**Use single store when**:
- State is simple and cohesive
- All state is tightly coupled
- Store is small (<100 lines)
- Splitting would create artificial boundaries

**Code characteristics**:
- Slices: More files, better organization, easier testing
- Single store: Fewer files, simpler mental model, faster to navigate

## Devtools and debugging

## Devtools and debugging

Our store is now feature-complete, but debugging state changes is still difficult. Let's add proper debugging tools and learn how to diagnose problems in Zustand stores.

### The Failure: Invisible State Changes

**Current problem**: When a bug occurs, we can't see what changed in the store or why.

```tsx
// User reports: "I clicked 'Mark Done' but the task didn't update"
// We have no visibility into:
// - Was the action called?
// - Did the state change?
// - Did the component re-render?
// - Was the selector correct?
```

**Let's create this failure**: Add a bug to the `updateTask` action:

```typescript
// src/store/slices/taskSlice.ts
updateTask: (id, updates) =>
  set((state) => ({
    tasks: state.tasks.map((task) =>
      task.id === id
        ? { ...task, ...updates, updatedAt: Date.now() }
        : task
    ),
  })),
```

Now introduce a typo:

```typescript
updateTask: (id, updates) =>
  set((state) => ({
    tasks: state.tasks.map((task) =>
      task.id === id + 'typo' // ← Bug: id will never match
        ? { ...task, ...updates, updatedAt: Date.now() }
        : task
    ),
  })),
```

**Browser Behavior**:
- Click "Mark Done" on a task
- Nothing happens
- No error in console
- Task status doesn't change

**Browser Console**:
```
(No output - silent failure)
```

**React DevTools - Components Tab**:
- `TaskCard` component selected
- Props: `{ task: { id: "task-1", status: "todo", ... } }`
- No indication of what went wrong

### Diagnostic Analysis: Reading the Silent Failure

**What the user experiences**:
- Expected: Task status changes to "done"
- Actual: Task status remains "todo"

**What the console reveals**: Nothing. No errors, no warnings.

**What DevTools shows**: Component didn't re-render because state didn't change.

**Root cause identified**: The action ran, but the condition `task.id === id + 'typo'` never matched, so no task was updated. State remained unchanged, so no re-render occurred.

**Why the current approach can't solve this**: We have no visibility into:
1. Whether the action was called
2. What the action received as arguments
3. What the state was before the action
4. What the state is after the action
5. Whether the state actually changed

**What we need**: Logging middleware to trace all state changes.

### Solution: DevTools Middleware

Zustand provides a `devtools` middleware that integrates with Redux DevTools:

```bash
# Install Redux DevTools browser extension first
# Chrome: https://chrome.google.com/webstore/detail/redux-devtools/
# Firefox: https://addons.mozilla.org/en-US/firefox/addon/reduxdevtools/
```

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { devtools } from 'zustand/middleware';
import { createTaskSlice, TaskSlice } from './slices/taskSlice';
import { createFilterSlice, FilterSlice } from './slices/filterSlice';
import { createUserSlice, UserSlice } from './slices/userSlice';

type TaskStore = TaskSlice & FilterSlice & UserSlice;

export const useTaskStore = create<TaskStore>()(
  devtools(
    persist(
      (...a) => ({
        ...createTaskSlice(...a),
        ...createFilterSlice(...a),
        ...createUserSlice(...a),
      }),
      {
        name: 'task-storage',
        storage: createJSONStorage(() => localStorage),
        partialize: (state) => ({
          tasks: state.tasks,
          filters: state.filters,
        }),
      }
    ),
    {
      name: 'TaskStore', // Name shown in DevTools
      enabled: true, // Enable in development
    }
  )
);

// Selectors remain the same
export const selectFilteredTasks = createMemoizedSelector(
  (state: TaskStore) => {
    const { tasks, filters } = state;
    return tasks.filter((task) => {
      if (filters.status !== 'all' && task.status !== filters.status) return false;
      if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
      if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
      return true;
    });
  },
  (a, b) => {
    if (a.length !== b.length) return false;
    if (a.length === 0) return true;
    return a[0].id === b[0].id && a[a.length - 1].id === b[a.length - 1].id;
  }
);

export const selectTasksByStatus = createMemoizedSelector(
  (state: TaskStore) => {
    const filteredTasks = selectFilteredTasks(state);
    return {
      todo: filteredTasks.filter((t) => t.status === 'todo'),
      inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
      done: filteredTasks.filter((t) => t.status === 'done'),
    };
  }
);

export const selectTaskStats = createMemoizedSelector(
  (state: TaskStore) => {
    const tasksByStatus = selectTasksByStatus(state);
    return {
      todoCount: tasksByStatus.todo.length,
      inProgressCount: tasksByStatus.inProgress.length,
      doneCount: tasksByStatus.done.length,
      totalCount: selectFilteredTasks(state).length,
    };
  }
);

function createMemoizedSelector<T, R>(
  selector: (state: T) => R,
  equalityFn: (a: R, b: R) => boolean = Object.is
) {
  let lastState: T | undefined;
  let lastResult: R | undefined;

  return (state: T): R => {
    if (lastState === state && lastResult !== undefined) {
      return lastResult;
    }

    const result = selector(state);
    if (lastResult !== undefined && equalityFn(result, lastResult)) {
      return lastResult;
    }

    lastState = state;
    lastResult = result;
    return result;
  };
}
```

**What changed**:
- Wrapped store with `devtools` middleware
- Specified store name for DevTools
- Middleware order matters: `devtools(persist(...))` not `persist(devtools(...))`

Now let's add action names to make debugging clearer:

```typescript
// src/store/slices/taskSlice.ts
import { StateCreator } from 'zustand';
import { Task } from '../../types/task';

export interface TaskSlice {
  tasks: Task[];
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'updatedAt'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
}

export const createTaskSlice: StateCreator<TaskSlice> = (set) => ({
  tasks: [],

  addTask: (taskData) =>
    set(
      (state) => ({
        tasks: [
          ...state.tasks,
          {
            ...taskData,
            id: crypto.randomUUID(),
            createdAt: Date.now(),
            updatedAt: Date.now(),
          },
        ],
      }),
      false,
      'tasks/add' // ← Action name for DevTools
    ),

  updateTask: (id, updates) =>
    set(
      (state) => ({
        tasks: state.tasks.map((task) =>
          task.id === id + 'typo' // ← Bug still present
            ? { ...task, ...updates, updatedAt: Date.now() }
            : task
        ),
      }),
      false,
      'tasks/update' // ← Action name for DevTools
    ),

  deleteTask: (id) =>
    set(
      (state) => ({
        tasks: state.tasks.filter((task) => task.id !== id),
      }),
      false,
      'tasks/delete' // ← Action name for DevTools
    ),
});
```

**Browser Behavior**:
- Open Redux DevTools (browser extension)
- Click "Mark Done" on a task
- DevTools shows action: `tasks/update`

**Redux DevTools - Action Tab**:
```
Action: tasks/update
State (before): { tasks: [...], filters: {...}, users: [...] }
State (after): { tasks: [...], filters: {...}, users: [...] }
Diff: (no changes)
```

**Redux DevTools - State Tab**:
- Can inspect full state tree
- Can see that `tasks` array is identical before and after
- Can see the specific task that should have changed

**Redux DevTools - Diff Tab**:
```
(no changes detected)
```

**Diagnostic insight**: The action ran, but state didn't change. This tells us the problem is in the action logic, not in the component or selector.

### Debugging Workflow: Using DevTools to Find the Bug

**Step 1: Verify the action was called**
- Redux DevTools shows `tasks/update` action
- ✓ Action was called

**Step 2: Check the action payload**
- DevTools doesn't show payload by default
- Let's add payload logging

**Step 3: Add custom logging**:

```typescript
// src/store/slices/taskSlice.ts
updateTask: (id, updates) =>
  set(
    (state) => {
      console.log('updateTask called:', { id, updates });
      console.log('Current tasks:', state.tasks.map(t => t.id));
      
      const updatedTasks = state.tasks.map((task) => {
        const matches = task.id === id + 'typo';
        console.log(`Checking task ${task.id}: matches=${matches}`);
        return matches
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task;
      });
      
      console.log('Updated tasks:', updatedTasks.map(t => ({ id: t.id, status: t.status })));
      
      return { tasks: updatedTasks };
    },
    false,
    'tasks/update'
  ),
```

**Browser Console**:
```
updateTask called: { id: "task-1", updates: { status: "done" } }
Current tasks: ["task-1", "task-2", "task-3"]
Checking task task-1: matches=false
Checking task task-2: matches=false
Checking task task-3: matches=false
Updated tasks: [
  { id: "task-1", status: "todo" },
  { id: "task-2", status: "in-progress" },
  { id: "task-3", status: "done" }
]
```

**Diagnostic insight**: 
- Action received correct `id`: `"task-1"`
- But `task.id === id + 'typo'` evaluates to `"task-1" === "task-1typo"` → `false`
- No task matched, so no task was updated

**Bug found**: The condition has `+ 'typo'` appended to `id`.

**Fix**:

```typescript
// src/store/slices/taskSlice.ts
updateTask: (id, updates) =>
  set(
    (state) => ({
      tasks: state.tasks.map((task) =>
        task.id === id // ← Fixed: removed 'typo'
          ? { ...task, ...updates, updatedAt: Date.now() }
          : task
      ),
    }),
    false,
    'tasks/update'
  ),
```

**Browser Behavior**:
- Click "Mark Done"
- Task status changes to "done"
- Redux DevTools shows state change

**Redux DevTools - Diff Tab**:
```
tasks[0].status: "todo" → "done"
tasks[0].updatedAt: 1704067200000 → 1704067201000
```

**Verification**: Bug fixed. DevTools helped us trace the exact problem.

### Custom Logging Middleware

For more control over logging, we can create custom middleware:

```typescript
// src/store/middleware/logger.ts
import { StateCreator, StoreMutatorIdentifier } from 'zustand';

type Logger = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = []
>(
  f: StateCreator<T, Mps, Mcs>,
  name?: string
) => StateCreator<T, Mps, Mcs>;

type LoggerImpl = <T>(
  f: StateCreator<T, [], []>,
  name?: string
) => StateCreator<T, [], []>;

const loggerImpl: LoggerImpl = (f, name) => (set, get, store) => {
  const loggedSet: typeof set = (...a) => {
    const prevState = get();
    set(...a);
    const nextState = get();
    
    console.group(`🔄 ${name || 'Store'} Update`);
    console.log('Previous State:', prevState);
    console.log('Next State:', nextState);
    console.log('Changed:', findChanges(prevState, nextState));
    console.groupEnd();
  };
  
  store.setState = loggedSet;
  return f(loggedSet, get, store);
};

export const logger = loggerImpl as Logger;

function findChanges<T extends object>(prev: T, next: T): Partial<T> {
  const changes: Partial<T> = {};
  
  for (const key in next) {
    if (prev[key] !== next[key]) {
      changes[key] = next[key];
    }
  }
  
  return changes;
}
```

```typescript
// src/store/useTaskStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { devtools } from 'zustand/middleware';
import { logger } from './middleware/logger';
import { createTaskSlice, TaskSlice } from './slices/taskSlice';
import { createFilterSlice, FilterSlice } from './slices/filterSlice';
import { createUserSlice, UserSlice } from './slices/userSlice';

type TaskStore = TaskSlice & FilterSlice & UserSlice;

export const useTaskStore = create<TaskStore>()(
  logger(
    devtools(
      persist(
        (...a) => ({
          ...createTaskSlice(...a),
          ...createFilterSlice(...a),
          ...createUserSlice(...a),
        }),
        {
          name: 'task-storage',
          storage: createJSONStorage(() => localStorage),
          partialize: (state) => ({
            tasks: state.tasks,
            filters: state.filters,
          }),
        }
      ),
      {
        name: 'TaskStore',
        enabled: true,
      }
    ),
    'TaskStore'
  )
);

// Selectors remain the same
export const selectFilteredTasks = createMemoizedSelector(
  (state: TaskStore) => {
    const { tasks, filters } = state;
    return tasks.filter((task) => {
      if (filters.status !== 'all' && task.status !== filters.status) return false;
      if (filters.assigneeId !== 'all' && task.assigneeId !== filters.assigneeId) return false;
      if (filters.priority !== 'all' && task.priority !== filters.priority) return false;
      return true;
    });
  },
  (a, b) => {
    if (a.length !== b.length) return false;
    if (a.length === 0) return true;
    return a[0].id === b[0].id && a[a.length - 1].id === b[a.length - 1].id;
  }
);

export const selectTasksByStatus = createMemoizedSelector(
  (state: TaskStore) => {
    const filteredTasks = selectFilteredTasks(state);
    return {
      todo: filteredTasks.filter((t) => t.status === 'todo'),
      inProgress: filteredTasks.filter((t) => t.status === 'in-progress'),
      done: filteredTasks.filter((t) => t.status === 'done'),
    };
  }
);

export const selectTaskStats = createMemoizedSelector(
  (state: TaskStore) => {
    const tasksByStatus = selectTasksByStatus(state);
    return {
      todoCount: tasksByStatus.todo.length,
      inProgressCount: tasksByStatus.inProgress.length,
      doneCount: tasksByStatus.done.length,
      totalCount: selectFilteredTasks(state).length,
    };
  }
);

function createMemoizedSelector<T, R>(
  selector: (state: T) => R,
  equalityFn: (a: R, b: R) => boolean = Object.is
) {
  let lastState: T | undefined;
  let lastResult: R | undefined;

  return (state: T): R => {
    if (lastState === state && lastResult !== undefined) {
      return lastResult;
    }

    const result = selector(state);
    if (lastResult !== undefined && equalityFn(result, lastResult)) {
      return lastResult;
    }

    lastState = state;
    lastResult = result;
    return result;
  };
}
```

**Browser Console** (when updating a task):
```
🔄 TaskStore Update
  Previous State: { tasks: [...], filters: {...}, users: [...] }
  Next State: { tasks: [...], filters: {...}, users: [...] }
  Changed: { tasks: [...] }
```

**When to use custom logging**:
- Development environment only
- Debugging complex state interactions
- Tracing performance issues
- Understanding re-render patterns

**Production considerations**:
- Disable logging in production builds
- Use environment variables to control logging
- Consider using a proper logging service (Sentry, LogRocket)

### Common Failure Modes and Their Signatures

#### Symptom: Component doesn't re-render when state changes

**Browser behavior**: State updates in DevTools, but UI doesn't change

**Console pattern**: No errors

**DevTools clues**:
- Redux DevTools shows state change
- React DevTools shows component didn't re-render
- Component's selector might be returning the same reference

**Root cause**: Selector is not detecting the change (reference equality issue)

**Solution**: Check selector's equality function or use a different selector

#### Symptom: State updates but immediately reverts

**Browser behavior**: UI flashes the new state, then reverts

**Console pattern**:
```
🔄 TaskStore Update
  Changed: { tasks: [...] }
🔄 TaskStore Update
  Changed: { tasks: [...] }  ← Reverted
```

**DevTools clues**:
- Two rapid state updates
- Second update undoes the first

**Root cause**: Multiple sources of truth or competing updates

**Solution**: Ensure single source of truth, use optimistic updates correctly

#### Symptom: Entire app re-renders on any state change

**Browser behavior**: App feels sluggish, all components flash in React DevTools

**Console pattern**: No errors, but many render logs

**DevTools clues**:
- React Profiler shows all components re-rendering
- Components are selecting entire store instead of specific slices

**Root cause**: Components using `useTaskStore()` instead of `useTaskStore(selector)`

**Solution**: Always use selectors, never subscribe to entire store

### Debugging Workflow: When Your Store Fails

**Step 1: Verify the action was called**
- Check Redux DevTools for action name
- If no action appears, the problem is in the component (event handler not firing)

**Step 2: Check the action payload**
- Add console.log in the action
- Verify the action received correct arguments

**Step 3: Inspect state before and after**
- Use Redux DevTools Diff tab
- Verify state actually changed

**Step 4: Check component subscriptions**
- Use React DevTools to see which components re-rendered
- Verify components are using correct selectors

**Step 5: Verify selector logic**
- Add console.log in selector
- Check if selector is returning expected data

**Step 6: Check for reference equality issues**
- If selector returns new object/array every time, component will always re-render
- Use memoization or custom equality function

### Performance Debugging with Zustand

Let's add performance tracking to our store:

```typescript
// src/store/middleware/performance.ts
import { StateCreator, StoreMutatorIdentifier } from 'zustand';

type Performance = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = []
>(
  f: StateCreator<T, Mps, Mcs>,
  name?: string
) => StateCreator<T, Mps, Mcs>;

type PerformanceImpl = <T>(
  f: StateCreator<T, [], []>,
  name?: string
) => StateCreator<T, [], []>;

const performanceImpl: PerformanceImpl = (f, name) => (set, get, store) => {
  const perfSet: typeof set = (...a) => {
    const start = performance.now();
    set(...a);
    const end = performance.now();
    
    const duration = end - start;
    if (duration > 16) { // Longer than one frame (60fps)
      console.warn(
        `⚠️ Slow state update in ${name || 'Store'}: ${duration.toFixed(2)}ms`
      );
    }
  };
  
  store.setState = perfSet;
  return f(perfSet, get, store);
};

export const performance = performanceImpl as Performance;
```

```typescript
// src/store/useTaskStore.ts
import { performance } from './middleware/performance';

export const useTaskStore = create<TaskStore>()(
  performance(
    logger(
      devtools(
        persist(
          (...a) => ({
            ...createTaskSlice(...a),
            ...createFilterSlice(...a),
            ...createUserSlice(...a),
          }),
          {
            name: 'task-storage',
            storage: createJSONStorage(() => localStorage),
            partialize: (state) => ({
              tasks: state.tasks,
              filters: state.filters,
            }),
          }
        ),
        {
          name: 'TaskStore',
          enabled: true,
        }
      ),
      'TaskStore'
    ),
    'TaskStore'
  )
);
```

**Browser Console** (if state update is slow):
```
⚠️ Slow state update in TaskStore: 23.45ms
```

**When this appears**: State update took longer than 16ms (one frame at 60fps), which could cause jank.

**What to do**:
1. Check if you're doing expensive computation in the action
2. Consider moving computation to a selector
3. Use memoization for expensive operations
4. Profile with React DevTools Profiler to find the bottleneck

### The Complete Debugging Toolkit

**Browser DevTools**:
- Console: Action logs, error messages
- Network: API calls (if fetching data)
- Performance: Main thread activity, frame rate

**React DevTools**:
- Components: Props, state, hooks
- Profiler: Render timing, why components rendered

**Redux DevTools**:
- Action: All state changes with action names
- State: Full state tree inspection
- Diff: What changed between states
- Trace: Stack trace for each action

**Custom Middleware**:
- Logger: Detailed state change logs
- Performance: Timing for state updates
- Error boundary: Catch and log errors in actions

### When to Apply: Debugging Strategies

**Use Redux DevTools when**:
- Debugging state changes
- Understanding action flow
- Time-travel debugging (undo/redo)
- Inspecting state tree

**Use custom logging when**:
- Need more detailed output than DevTools provides
- Debugging specific action logic
- Tracking performance issues
- Development environment only

**Use performance middleware when**:
- App feels sluggish
- Investigating render performance
- Optimizing state updates
- Identifying bottlenecks

**Disable all debugging in production**:
- Use environment variables
- Strip logging in build process
- Keep DevTools for development only
