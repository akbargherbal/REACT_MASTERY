# Chapter Appendix B: Common Pitfalls and How to Avoid Them

## Introduction: The Catalog of Failure

This appendix is your field guide to the most common ways React applications fail. Unlike the chapter-by-chapter progression where we built understanding through controlled failures, this is a reference catalog—organized by symptom, not by concept.

When your application breaks, you don't think "I need to review Chapter 4 on useEffect." You think "Why is my component rendering infinitely?" or "Why isn't my state updating?" This appendix meets you at that moment of confusion.

## How to Use This Catalog

Each pitfall follows this structure:

1. **Symptom**: What you observe in the browser
2. **Browser Evidence**: Console output, DevTools observations, Network tab patterns
3. **Root Cause**: The underlying mechanism causing the failure
4. **The Trap**: Why this mistake is so common
5. **The Fix**: Concrete solution with before/after code
6. **Prevention**: How to avoid this in the future

**Reference Implementation**: Throughout this appendix, we'll use a `TaskManager` component as our recurring example—a realistic application that manages tasks with filtering, sorting, and real-time updates. This gives us a consistent context for demonstrating each pitfall.

```tsx
// TaskManager.tsx - Our reference implementation
// We'll show how various pitfalls manifest in this component
interface Task {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: Date;
}

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  const [sortBy, setSortBy] = useState<'date' | 'priority'>('date');
  
  // We'll see this component fail in many ways...
  return (
    <div>
      <TaskList tasks={tasks} />
      <TaskFilters filter={filter} onFilterChange={setFilter} />
    </div>
  );
}
```

## State Management Pitfalls

## Pitfall 1: The Stale Closure Trap

### Symptom

Event handlers or callbacks reference old state values even after state updates. Clicking a button multiple times shows the same old value in the console, or operations use outdated data.

### Browser Evidence

**Browser Console**:
```
Current count: 0
Current count: 0
Current count: 0
// Expected: 0, 1, 2
```

**React DevTools - Components Tab**:
- State shows correct value: `count: 3`
- But console logs show: `0, 0, 0`
- Indicates closure captured old state

```tsx
// TaskManager.tsx - WRONG: Stale closure in setTimeout
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [autoSaveEnabled, setAutoSaveEnabled] = useState(true);

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks([...tasks, newTask]);
    
    // Auto-save after 2 seconds
    if (autoSaveEnabled) {
      setTimeout(() => {
        // BUG: This captures the current `tasks` array
        // When setTimeout fires, it uses the OLD tasks array
        console.log('Saving tasks:', tasks);
        saveTasks(tasks); // Saves stale data!
      }, 2000);
    }
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <TaskList tasks={tasks} />
    </div>
  );
}
```

### Root Cause

JavaScript closures capture variables from their surrounding scope at the time the function is created. When `setTimeout` creates its callback, it captures the current value of `tasks`. Even though `tasks` changes later, the callback still references the old array.

This is the **closure trap**: asynchronous operations (setTimeout, setInterval, event listeners, promises) capture state at creation time, not execution time.

### The Trap

This feels like it should work because:
1. The state update happens immediately: `setTasks([...tasks, newTask])`
2. The setTimeout is created after the update
3. You expect it to "see" the new state

But React state updates are asynchronous, and the closure captures the value at the moment the function is defined, not when it executes.

### The Fix: Use Functional Updates

When you need the latest state inside an async operation, use the functional form of setState:

```tsx
// TaskManager.tsx - CORRECT: Functional update
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [autoSaveEnabled, setAutoSaveEnabled] = useState(true);

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks(prevTasks => [...prevTasks, newTask]);
    
    if (autoSaveEnabled) {
      setTimeout(() => {
        // Use functional update to get latest state
        setTasks(currentTasks => {
          console.log('Saving tasks:', currentTasks);
          saveTasks(currentTasks); // Always has latest data
          return currentTasks; // No change, just reading
        });
      }, 2000);
    }
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <TaskList tasks={tasks} />
    </div>
  );
}
```

### Alternative: Use useRef for Reading Latest State

If you need to read state without triggering updates:

```tsx
// TaskManager.tsx - Alternative: useRef
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const tasksRef = useRef(tasks);
  
  // Keep ref in sync with state
  useEffect(() => {
    tasksRef.current = tasks;
  }, [tasks]);

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks([...tasks, newTask]);
    
    setTimeout(() => {
      // tasksRef.current always has the latest value
      console.log('Saving tasks:', tasksRef.current);
      saveTasks(tasksRef.current);
    }, 2000);
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <TaskList tasks={tasks} />
    </div>
  );
}
```

### Prevention

1. **Default to functional updates** when the new state depends on the old state
2. **Use useRef** for values you need to read in async operations but don't need to trigger re-renders
3. **Avoid capturing state** in setTimeout/setInterval—use refs or functional updates
4. **ESLint rule**: Enable `react-hooks/exhaustive-deps` to catch some closure issues

---

## Pitfall 2: Mutating State Directly

### Symptom

State appears to update in React DevTools, but the component doesn't re-render. Or the component re-renders, but the UI shows stale data. Arrays or objects seem to "lose" updates.

### Browser Evidence

**React DevTools - Components Tab**:
- State value changes: `tasks: Array(5)` → `tasks: Array(6)`
- But component doesn't re-render
- Or renders but shows old data

**Browser Console**:
```
State before: [{ id: '1', title: 'Task 1', completed: false }]
State after: [{ id: '1', title: 'Task 1', completed: false }]
// Expected: completed: true
```

```tsx
// TaskManager.tsx - WRONG: Direct mutation
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const handleToggleTask = (taskId: string) => {
    // BUG: Mutating the array directly
    const task = tasks.find(t => t.id === taskId);
    if (task) {
      task.completed = !task.completed; // Direct mutation!
      setTasks(tasks); // Same reference, React won't re-render
    }
  };

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    // BUG: Mutating the array directly
    tasks.push(newTask); // Direct mutation!
    setTasks(tasks); // Same reference, React won't re-render
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <TaskList tasks={tasks} onToggle={handleToggleTask} />
    </div>
  );
}
```

### Root Cause

React uses **reference equality** to detect state changes. When you call `setTasks(tasks)` with the same array reference, React thinks nothing changed and skips the re-render.

Even if you mutate the array's contents, the array reference itself is the same:

```typescript
const arr = [1, 2, 3];
arr.push(4); // Mutates the array
console.log(arr); // [1, 2, 3, 4]
// But `arr` is still the same reference!
```

### The Trap

This feels natural because:
1. In regular JavaScript, you mutate arrays and objects all the time
2. The mutation "works"—the data changes
3. React DevTools shows the new value
4. But React's reconciliation never triggers

### The Fix: Create New References

Always create new arrays/objects when updating state:

```tsx
// TaskManager.tsx - CORRECT: Immutable updates
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const handleToggleTask = (taskId: string) => {
    // Create a new array with the updated task
    setTasks(prevTasks =>
      prevTasks.map(task =>
        task.id === taskId
          ? { ...task, completed: !task.completed } // New object
          : task
      )
    );
  };

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    // Create a new array with the new task
    setTasks(prevTasks => [...prevTasks, newTask]);
  };

  const handleDeleteTask = (taskId: string) => {
    // Create a new array without the deleted task
    setTasks(prevTasks => prevTasks.filter(task => task.id !== taskId));
  };

  const handleUpdateTask = (taskId: string, updates: Partial<Task>) => {
    // Create a new array with the updated task
    setTasks(prevTasks =>
      prevTasks.map(task =>
        task.id === taskId
          ? { ...task, ...updates } // Merge updates into new object
          : task
      )
    );
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <TaskList 
        tasks={tasks} 
        onToggle={handleToggleTask}
        onDelete={handleDeleteTask}
        onUpdate={handleUpdateTask}
      />
    </div>
  );
}
```

### Common Immutable Update Patterns

**Arrays**:

```typescript
// Add item
setItems([...items, newItem]);
setItems(items.concat(newItem));

// Remove item
setItems(items.filter(item => item.id !== idToRemove));

// Update item
setItems(items.map(item => 
  item.id === idToUpdate 
    ? { ...item, ...updates }
    : item
));

// Insert at index
setItems([
  ...items.slice(0, index),
  newItem,
  ...items.slice(index)
]);

// Replace at index
setItems(items.map((item, i) => 
  i === index ? newItem : item
));
```

**Objects**:

```typescript
// Update property
setUser({ ...user, name: 'New Name' });

// Update nested property
setUser({
  ...user,
  address: {
    ...user.address,
    city: 'New City'
  }
});

// Add property
setUser({ ...user, newProp: 'value' });

// Remove property
const { propToRemove, ...rest } = user;
setUser(rest);
```

### Prevention

1. **Never mutate state directly**—always create new references
2. **Use spread operators** (`...`) for shallow copies
3. **Use `.map()`, `.filter()`, `.concat()`** instead of `.push()`, `.splice()`, `.pop()`
4. **For deep updates**, consider Immer library (used by Redux Toolkit)
5. **TypeScript**: Use `readonly` arrays to catch mutations at compile time

---

## Pitfall 3: Batching Confusion and Multiple setState Calls

### Symptom

Multiple state updates in the same function seem to "lose" some updates, or the component renders fewer times than expected. Or in React 18+, updates that used to batch now don't.

### Browser Evidence

**Browser Console**:
```
Render count: 1
// Expected: 3 (one for each setState)
```

**React DevTools - Profiler**:
- Shows 1 render instead of 3
- All state updates applied in single commit

```tsx
// TaskManager.tsx - Understanding batching
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  const [sortBy, setSortBy] = useState<'date' | 'priority'>('date');
  const [renderCount, setRenderCount] = useState(0);

  useEffect(() => {
    setRenderCount(c => c + 1);
    console.log('Render count:', renderCount);
  });

  const handleResetFilters = () => {
    // In React 18+, these are automatically batched
    setFilter('all');
    setSortBy('date');
    setTasks([]);
    // Only 1 re-render, not 3!
  };

  const handleAsyncReset = async () => {
    await fetchTasks(); // Async operation
    
    // In React 17, these would NOT be batched (3 renders)
    // In React 18+, these ARE batched (1 render)
    setFilter('all');
    setSortBy('date');
    setTasks([]);
  };

  return (
    <div>
      <button onClick={handleResetFilters}>Reset Filters</button>
      <button onClick={handleAsyncReset}>Async Reset</button>
      <p>Render count: {renderCount}</p>
    </div>
  );
}
```

### Root Cause

**React 18+ Automatic Batching**: React automatically batches multiple state updates into a single re-render, even in async functions, timeouts, and event handlers. This is a performance optimization.

**React 17 Batching**: Only batched updates inside React event handlers. Updates in promises, setTimeout, or native event handlers were not batched.

### The Trap

This causes confusion when:
1. You expect each `setState` to trigger a separate render (for debugging or side effects)
2. You're migrating from React 17 and see different behavior
3. You have side effects that depend on intermediate state values

### The Fix: Use flushSync for Immediate Updates (Rare)

If you truly need to force a synchronous update (very rare):

```tsx
import { flushSync } from 'react-dom';

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  const handleResetWithMeasurement = () => {
    // Force synchronous update
    flushSync(() => {
      setFilter('all');
    });
    
    // This reads the updated DOM immediately
    const height = document.getElementById('task-list')?.offsetHeight;
    console.log('Height after filter reset:', height);
    
    // This will be a separate render
    flushSync(() => {
      setTasks([]);
    });
  };

  return (
    <div>
      <button onClick={handleResetWithMeasurement}>
        Reset with Measurement
      </button>
    </div>
  );
}
```

### When Batching Causes Issues

**Problem**: You need to read DOM measurements between state updates.

**Solution**: Use `flushSync` (but this is a performance hit—avoid if possible).

**Problem**: You have a useEffect that depends on multiple state values and needs to run after each individual update.

**Solution**: Rethink your effect dependencies. Usually, you want the effect to run after all updates anyway.

### Prevention

1. **Embrace batching**—it's a performance win
2. **Don't rely on intermediate renders** for side effects
3. **Use `flushSync` sparingly**—only when you must read DOM between updates
4. **In React 17**, wrap async updates in `unstable_batchedUpdates` if needed

---

## Pitfall 4: State Initialization with Expensive Computation

### Symptom

Component initialization is slow. React DevTools Profiler shows long initial render time. The expensive computation runs on every render, not just the first one.

### Browser Evidence

**React DevTools - Profiler**:
- Initial render: 450ms
- Subsequent renders: 440ms (still running expensive computation!)
- Expected: Initial 450ms, subsequent <10ms

**Browser Console**:
```
Computing initial tasks... (runs on every render)
Computed 10000 tasks in 445ms
```

```tsx
// TaskManager.tsx - WRONG: Expensive computation on every render
function TaskManager() {
  // BUG: This function runs on EVERY render!
  const [tasks, setTasks] = useState(generateInitialTasks(10000));
  
  // Even though we only use the result once,
  // generateInitialTasks(10000) executes every time TaskManager renders
  
  return <TaskList tasks={tasks} />;
}

function generateInitialTasks(count: number): Task[] {
  console.log('Computing initial tasks...');
  const start = performance.now();
  
  const tasks: Task[] = [];
  for (let i = 0; i < count; i++) {
    tasks.push({
      id: crypto.randomUUID(),
      title: `Task ${i}`,
      completed: Math.random() > 0.5,
      priority: ['low', 'medium', 'high'][Math.floor(Math.random() * 3)] as Task['priority'],
      createdAt: new Date(Date.now() - Math.random() * 86400000),
    });
  }
  
  console.log(`Computed ${count} tasks in ${performance.now() - start}ms`);
  return tasks;
}
```

### Root Cause

When you pass a value to `useState`, React evaluates that expression on every render. Even though React only uses the initial value once, the computation still runs.

```typescript
const [state, setState] = useState(expensiveFunction());
// expensiveFunction() runs on EVERY render
```

### The Trap

This looks correct because:
1. You know `useState` only uses the initial value once
2. The state updates correctly
3. But the performance cost is hidden until you profile

### The Fix: Lazy Initialization

Pass a function to `useState` instead of a value:

```tsx
// TaskManager.tsx - CORRECT: Lazy initialization
function TaskManager() {
  // Pass a function, not a value
  // The function only runs once, on mount
  const [tasks, setTasks] = useState(() => generateInitialTasks(10000));
  
  return <TaskList tasks={tasks} />;
}

function generateInitialTasks(count: number): Task[] {
  console.log('Computing initial tasks...');
  const start = performance.now();
  
  const tasks: Task[] = [];
  for (let i = 0; i < count; i++) {
    tasks.push({
      id: crypto.randomUUID(),
      title: `Task ${i}`,
      completed: Math.random() > 0.5,
      priority: ['low', 'medium', 'high'][Math.floor(Math.random() * 3)] as Task['priority'],
      createdAt: new Date(Date.now() - Math.random() * 86400000),
    });
  }
  
  console.log(`Computed ${count} tasks in ${performance.now() - start}ms`);
  return tasks;
}
```

**Browser Console** (after fix):
```
Computing initial tasks... (only on mount)
Computed 10000 tasks in 445ms
// No more logs on subsequent renders
```

### When to Use Lazy Initialization

Use lazy initialization when:
1. Initial state requires expensive computation
2. Initial state comes from localStorage or other I/O
3. Initial state involves complex object creation

Don't bother when:
1. Initial state is a simple value: `useState(0)`, `useState('')`
2. Initial state is a literal: `useState([])`, `useState({})`

### Prevention

1. **Profile your components** with React DevTools Profiler
2. **Use lazy initialization** for any non-trivial computation
3. **ESLint rule**: Create a custom rule to catch `useState(expensiveFunction())`

## useEffect Pitfalls

## Pitfall 5: Missing Dependencies in useEffect

### Symptom

Effect uses stale values. Component doesn't update when expected. Effect doesn't re-run when it should. ESLint warning: "React Hook useEffect has a missing dependency."

### Browser Evidence

**Browser Console**:
```
Effect running with filter: all
// User changes filter to "active"
// Effect doesn't re-run, still uses "all"
```

**React DevTools - Components Tab**:
- State shows: `filter: "active"`
- But effect behavior suggests: `filter: "all"`

**ESLint Warning**:
```
React Hook useEffect has a missing dependency: 'filter'. 
Either include it or remove the dependency array.
```

```tsx
// TaskManager.tsx - WRONG: Missing dependency
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  useEffect(() => {
    // BUG: Effect uses `filter` but doesn't list it in dependencies
    const filtered = tasks.filter(task => {
      if (filter === 'active') return !task.completed;
      if (filter === 'completed') return task.completed;
      return true;
    });
    
    console.log('Filtered tasks:', filtered);
    updateUI(filtered);
  }, [tasks]); // Missing `filter`!

  return (
    <div>
      <TaskFilters filter={filter} onFilterChange={setFilter} />
      <TaskList tasks={tasks} />
    </div>
  );
}
```

### Root Cause

Effects capture values from their surrounding scope (closure). When dependencies are missing, the effect uses stale values from the render when it was created.

React's dependency array tells React when to re-run the effect. If you omit a dependency, React won't re-run the effect when that value changes.

### The Trap

You might omit dependencies because:
1. You want the effect to run only once (use `[]` instead)
2. You want to avoid infinite loops (fix the root cause instead)
3. ESLint warning seems wrong (it's usually right)
4. You think React will "figure it out" (it won't)

### The Fix: Include All Dependencies

Always include every value from the component scope that the effect uses:

```tsx
// TaskManager.tsx - CORRECT: All dependencies included
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  useEffect(() => {
    const filtered = tasks.filter(task => {
      if (filter === 'active') return !task.completed;
      if (filter === 'completed') return task.completed;
      return true;
    });
    
    console.log('Filtered tasks:', filtered);
    updateUI(filtered);
  }, [tasks, filter]); // Both dependencies included

  return (
    <div>
      <TaskFilters filter={filter} onFilterChange={setFilter} />
      <TaskList tasks={tasks} />
    </div>
  );
}
```

### When Dependencies Cause Infinite Loops

If including a dependency causes an infinite loop, the problem is not the dependency—it's that your effect is creating a new reference on every render:

```tsx
// WRONG: Infinite loop
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  
  // This object is created on every render (new reference)
  const config = { sortBy: 'date', order: 'asc' };

  useEffect(() => {
    const sorted = sortTasks(tasks, config);
    setTasks(sorted); // Updates tasks
  }, [tasks, config]); // config changes every render → infinite loop!

  return <TaskList tasks={tasks} />;
}
```

**Fix 1**: Move the object inside the effect:

```tsx
// CORRECT: Object created inside effect
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    const config = { sortBy: 'date', order: 'asc' };
    const sorted = sortTasks(tasks, config);
    setTasks(sorted);
  }, [tasks]); // No config dependency needed

  return <TaskList tasks={tasks} />;
}
```

**Fix 2**: Use useMemo to stabilize the reference:

```tsx
// CORRECT: Stable reference with useMemo
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  
  const config = useMemo(
    () => ({ sortBy: 'date', order: 'asc' }),
    [] // Only create once
  );

  useEffect(() => {
    const sorted = sortTasks(tasks, config);
    setTasks(sorted);
  }, [tasks, config]); // config is stable now

  return <TaskList tasks={tasks} />;
}
```

**Fix 3**: Use primitive values instead of objects:

```tsx
// CORRECT: Primitive dependencies
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const sortBy = 'date';
  const order = 'asc';

  useEffect(() => {
    const sorted = sortTasks(tasks, { sortBy, order });
    setTasks(sorted);
  }, [tasks, sortBy, order]); // Primitives are stable

  return <TaskList tasks={tasks} />;
}
```

### Prevention

1. **Always include all dependencies**—trust ESLint
2. **Use `eslint-plugin-react-hooks`** and don't disable the rule
3. **If you get an infinite loop**, fix the root cause (unstable references)
4. **Move objects/functions inside effects** when possible
5. **Use useMemo/useCallback** to stabilize references when necessary

---

## Pitfall 6: Effects That Should Be Event Handlers

### Symptom

Effect runs on every render or at unexpected times. Logic that should run in response to user action runs automatically. Unnecessary API calls or side effects.

### Browser Evidence

**Browser Console**:
```
Saving tasks... (runs on mount)
Saving tasks... (runs on every task change)
Saving tasks... (runs when filter changes)
// Expected: Only when user clicks "Save"
```

**Network Tab**:
- POST /api/tasks (on mount)
- POST /api/tasks (on every keystroke)
- POST /api/tasks (on filter change)
- Expected: Only on explicit save action

```tsx
// TaskManager.tsx - WRONG: Effect for user action
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [shouldSave, setShouldSave] = useState(false);

  // BUG: Using effect for something that should be an event handler
  useEffect(() => {
    if (shouldSave) {
      console.log('Saving tasks...');
      saveTasks(tasks);
      setShouldSave(false);
    }
  }, [shouldSave, tasks]);

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks([...tasks, newTask]);
    setShouldSave(true); // Trigger effect
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <button onClick={() => setShouldSave(true)}>Save</button>
    </div>
  );
}
```

### Root Cause

Effects are for **synchronizing with external systems** (APIs, DOM, subscriptions), not for responding to user actions. When you use an effect for user-triggered logic, you're fighting React's model.

The effect runs:
1. On mount (usually not desired)
2. Whenever dependencies change (too often)
3. Not in direct response to the user action (confusing)

### The Trap

You might use effects for user actions because:
1. You want to "trigger" logic from multiple places
2. You're used to imperative programming (do X, then Y)
3. You think effects are "the React way" to do side effects
4. You want to avoid passing callbacks through props

### The Fix: Use Event Handlers

Put the logic directly in the event handler:

```tsx
// TaskManager.tsx - CORRECT: Event handler for user action
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const handleSave = async () => {
    console.log('Saving tasks...');
    try {
      await saveTasks(tasks);
      toast.success('Tasks saved!');
    } catch (error) {
      toast.error('Failed to save tasks');
    }
  };

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks([...tasks, newTask]);
    // Don't auto-save, let user decide
  };

  return (
    <div>
      <TaskInput onAdd={handleAddTask} />
      <button onClick={handleSave}>Save</button>
    </div>
  );
}
```

### When You Actually Need Auto-Save

If you truly want auto-save (save after every change), use a debounced effect:

```tsx
// TaskManager.tsx - Auto-save with debounce
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    // Debounce: only save after user stops editing for 2 seconds
    const timeoutId = setTimeout(() => {
      console.log('Auto-saving tasks...');
      saveTasks(tasks);
    }, 2000);

    // Cleanup: cancel the timeout if tasks change again
    return () => clearTimeout(timeoutId);
  }, [tasks]);

  return <TaskInput onAdd={handleAddTask} />;
}
```

### Decision Framework: Effect vs. Event Handler

**Use an event handler when**:
- Logic runs in response to a specific user action
- Logic should run exactly once per action
- Logic is part of the user's intent (save, submit, delete)

**Use an effect when**:
- Synchronizing with an external system (WebSocket, DOM, browser API)
- Logic should run whenever certain data changes
- Logic is about keeping things in sync, not responding to actions

### Prevention

1. **Default to event handlers** for user actions
2. **Use effects for synchronization** only
3. **Ask**: "Is this responding to a user action or synchronizing state?"
4. **If you need auto-save**, use debouncing

---

## Pitfall 7: Forgetting Effect Cleanup

### Symptom

Memory leaks. "Can't perform a React state update on an unmounted component" warning. Subscriptions or timers continue after component unmounts. Stale data from old requests.

### Browser Evidence

**Browser Console**:
```
Warning: Can't perform a React state update on an unmounted component.
This is a no-op, but it indicates a memory leak in your application.
To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup function.
```

**Network Tab**:
- Multiple requests to /api/tasks
- Component unmounts
- Old requests complete and try to update state
- Warning appears

**Memory Profiler**:
- Memory usage increases over time
- Event listeners not removed
- Timers still running

```tsx
// TaskManager.tsx - WRONG: No cleanup
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    // BUG: No cleanup for interval
    const intervalId = setInterval(() => {
      fetchTasks().then(setTasks);
    }, 5000);
    
    // BUG: No cleanup for event listener
    window.addEventListener('online', () => {
      fetchTasks().then(setTasks);
    });
    
    // BUG: No cleanup for fetch
    fetchTasks().then(setTasks);
    
    // Missing cleanup function!
  }, []);

  return <TaskList tasks={tasks} />;
}
```

### Root Cause

When a component unmounts, any ongoing async operations (timers, subscriptions, fetch requests) continue running. If they try to update state after unmount, React warns about memory leaks.

Effects need cleanup to:
1. Cancel timers and intervals
2. Remove event listeners
3. Unsubscribe from subscriptions
4. Abort fetch requests
5. Close WebSocket connections

### The Trap

You forget cleanup because:
1. The component works fine initially
2. The warning only appears when navigating away
3. You don't test unmounting scenarios
4. You think React will "clean up automatically" (it won't)

### The Fix: Return a Cleanup Function

Always return a cleanup function from effects that create subscriptions or timers:

```tsx
// TaskManager.tsx - CORRECT: Proper cleanup
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    let isMounted = true; // Track mount status
    
    // Set up interval
    const intervalId = setInterval(() => {
      fetchTasks().then(data => {
        if (isMounted) setTasks(data);
      });
    }, 5000);
    
    // Set up event listener
    const handleOnline = () => {
      fetchTasks().then(data => {
        if (isMounted) setTasks(data);
      });
    };
    window.addEventListener('online', handleOnline);
    
    // Initial fetch with AbortController
    const abortController = new AbortController();
    fetchTasks({ signal: abortController.signal })
      .then(data => {
        if (isMounted) setTasks(data);
      })
      .catch(error => {
        if (error.name === 'AbortError') {
          console.log('Fetch aborted');
        }
      });
    
    // Cleanup function
    return () => {
      isMounted = false;
      clearInterval(intervalId);
      window.removeEventListener('online', handleOnline);
      abortController.abort();
    };
  }, []);

  return <TaskList tasks={tasks} />;
}
```

### Cleanup Patterns for Common Scenarios

**Timers**:

```typescript
useEffect(() => {
  const timeoutId = setTimeout(() => {
    // Do something
  }, 1000);
  
  return () => clearTimeout(timeoutId);
}, []);

useEffect(() => {
  const intervalId = setInterval(() => {
    // Do something repeatedly
  }, 1000);
  
  return () => clearInterval(intervalId);
}, []);
```

**Event Listeners**:

```typescript
useEffect(() => {
  const handleResize = () => {
    // Handle resize
  };
  
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

**Fetch Requests**:

```typescript
useEffect(() => {
  const abortController = new AbortController();
  
  fetch('/api/data', { signal: abortController.signal })
    .then(res => res.json())
    .then(data => setData(data))
    .catch(error => {
      if (error.name !== 'AbortError') {
        console.error(error);
      }
    });
  
  return () => abortController.abort();
}, []);
```

**Subscriptions** (WebSocket, Firebase, etc.):

```typescript
useEffect(() => {
  const unsubscribe = subscribeToTasks(tasks => {
    setTasks(tasks);
  });
  
  return unsubscribe;
}, []);
```

### Prevention

1. **Always return cleanup** from effects with subscriptions/timers
2. **Use AbortController** for fetch requests
3. **Test unmounting** scenarios in development
4. **Use React Query** for data fetching (handles cleanup automatically)
5. **ESLint rule**: Create custom rule to catch missing cleanup

---

## Pitfall 8: Running Effects on Every Render

### Symptom

Effect runs constantly. Performance degrades. Infinite loops. Console flooded with logs. Network tab shows repeated requests.

### Browser Evidence

**Browser Console**:
```
Effect running...
Effect running...
Effect running...
(repeats infinitely)
```

**React DevTools - Profiler**:
- Component renders continuously
- Each render triggers effect
- Effect triggers state update
- State update triggers render
- Infinite loop

**Network Tab**:
- Hundreds of requests per second
- Browser becomes unresponsive

```tsx
// TaskManager.tsx - WRONG: Effect runs on every render
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [sortedTasks, setSortedTasks] = useState<Task[]>([]);

  // BUG: No dependency array → runs on every render
  useEffect(() => {
    console.log('Sorting tasks...');
    const sorted = [...tasks].sort((a, b) => 
      a.createdAt.getTime() - b.createdAt.getTime()
    );
    setSortedTasks(sorted); // Triggers re-render
  }); // Missing dependency array!

  return <TaskList tasks={sortedTasks} />;
}
```

### Root Cause

Without a dependency array, effects run after **every render**. If the effect updates state, it triggers another render, which runs the effect again, creating an infinite loop.

Three forms of useEffect:

1. **No dependency array**: Runs after every render
   ```typescript
   useEffect(() => {
     // Runs after every render
   });
   ```

2. **Empty dependency array**: Runs once on mount
   ```typescript
   useEffect(() => {
     // Runs once on mount
   }, []);
   ```

3. **With dependencies**: Runs when dependencies change
   ```typescript
   useEffect(() => {
     // Runs when dep1 or dep2 changes
   }, [dep1, dep2]);
   ```

### The Trap

You omit the dependency array because:
1. You forget to add it
2. You don't understand the difference
3. You want it to run "all the time" (wrong approach)

### The Fix: Add Dependency Array

Specify when the effect should run:

```tsx
// TaskManager.tsx - CORRECT: Effect runs when tasks change
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [sortedTasks, setSortedTasks] = useState<Task[]>([]);

  useEffect(() => {
    console.log('Sorting tasks...');
    const sorted = [...tasks].sort((a, b) => 
      a.createdAt.getTime() - b.createdAt.getTime()
    );
    setSortedTasks(sorted);
  }, [tasks]); // Only run when tasks changes

  return <TaskList tasks={sortedTasks} />;
}
```

### Better: Don't Use an Effect at All

For derived state (computed from existing state), don't use an effect—compute during render:

```tsx
// TaskManager.tsx - BEST: No effect needed
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  // Compute during render, no effect needed
  const sortedTasks = useMemo(
    () => [...tasks].sort((a, b) => 
      a.createdAt.getTime() - b.createdAt.getTime()
    ),
    [tasks]
  );

  return <TaskList tasks={sortedTasks} />;
}
```

### When to Use Each Pattern

**No dependency array** (almost never):
```typescript
useEffect(() => {
  // Runs after every render
  // Use case: Debugging, logging (remove in production)
});
```

**Empty dependency array** (mount only):
```typescript
useEffect(() => {
  // Runs once on mount
  // Use case: Initial data fetch, subscriptions, one-time setup
}, []);
```

**With dependencies** (most common):
```typescript
useEffect(() => {
  // Runs when dependencies change
  // Use case: Sync with external system when data changes
}, [dep1, dep2]);
```

### Prevention

1. **Always include dependency array** (even if empty)
2. **For derived state**, use useMemo instead of useEffect
3. **ESLint rule**: `react-hooks/exhaustive-deps` catches missing arrays
4. **Ask**: "Does this need to be an effect at all?"

## Component Rendering Pitfalls

## Pitfall 9: Creating Components Inside Components

### Symptom

Component loses state on every render. Input fields lose focus while typing. Animations restart. React DevTools shows component unmounting and remounting constantly.

### Browser Evidence

**Browser Console**:
```
TaskItem mounted
TaskItem unmounted
TaskItem mounted
TaskItem unmounted
(repeats on every keystroke)
```

**React DevTools - Components Tab**:
- Component tree shows TaskItem disappearing and reappearing
- State resets to initial values
- Keys change on every render

**User Experience**:
- Typing in input loses focus after each character
- Checkboxes uncheck themselves
- Animations restart mid-way

```tsx
// TaskManager.tsx - WRONG: Component defined inside component
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  // BUG: Component defined inside render
  function TaskItem({ task }: { task: Task }) {
    const [isEditing, setIsEditing] = useState(false);
    
    return (
      <div>
        {isEditing ? (
          <input defaultValue={task.title} />
        ) : (
          <span>{task.title}</span>
        )}
        <button onClick={() => setIsEditing(!isEditing)}>
          Edit
        </button>
      </div>
    );
  }

  return (
    <div>
      {tasks.map(task => (
        <TaskItem key={task.id} task={task} />
      ))}
    </div>
  );
}
```

### Root Cause

When you define a component inside another component, React creates a **new component type** on every render. React sees `TaskItem` as a different component each time, so it unmounts the old one and mounts a new one, losing all state.

```typescript
// Render 1: TaskItem is function A
function TaskItem() { ... }

// Render 2: TaskItem is function B (different reference!)
function TaskItem() { ... }

// React thinks: "This is a different component, unmount A and mount B"
```

### The Trap

You define components inside components because:
1. You want to access parent state without passing props
2. You think it's "scoped" or "private"
3. You're used to nested functions in regular JavaScript
4. It seems convenient

### The Fix: Define Components Outside

Always define components at the top level:

```tsx
// TaskItem.tsx - CORRECT: Component defined outside
interface TaskItemProps {
  task: Task;
  onUpdate: (id: string, updates: Partial<Task>) => void;
}

function TaskItem({ task, onUpdate }: TaskItemProps) {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      {isEditing ? (
        <input 
          defaultValue={task.title}
          onBlur={(e) => {
            onUpdate(task.id, { title: e.target.value });
            setIsEditing(false);
          }}
        />
      ) : (
        <span>{task.title}</span>
      )}
      <button onClick={() => setIsEditing(!isEditing)}>
        Edit
      </button>
    </div>
  );
}

// TaskManager.tsx
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const handleUpdateTask = (id: string, updates: Partial<Task>) => {
    setTasks(tasks.map(task => 
      task.id === id ? { ...task, ...updates } : task
    ));
  };

  return (
    <div>
      {tasks.map(task => (
        <TaskItem 
          key={task.id} 
          task={task}
          onUpdate={handleUpdateTask}
        />
      ))}
    </div>
  );
}
```

### If You Need to Access Parent State

Pass it as props or use context:

```tsx
// Using context for deeply nested access
const TaskContext = createContext<{
  tasks: Task[];
  updateTask: (id: string, updates: Partial<Task>) => void;
} | null>(null);

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const updateTask = (id: string, updates: Partial<Task>) => {
    setTasks(tasks.map(task => 
      task.id === id ? { ...task, ...updates } : task
    ));
  };

  return (
    <TaskContext.Provider value={{ tasks, updateTask }}>
      <TaskList />
    </TaskContext.Provider>
  );
}

// TaskItem can now access context without prop drilling
function TaskItem({ task }: { task: Task }) {
  const context = useContext(TaskContext);
  if (!context) throw new Error('TaskItem must be used within TaskContext');
  
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      {isEditing ? (
        <input 
          defaultValue={task.title}
          onBlur={(e) => {
            context.updateTask(task.id, { title: e.target.value });
            setIsEditing(false);
          }}
        />
      ) : (
        <span>{task.title}</span>
      )}
    </div>
  );
}
```

### Prevention

1. **Never define components inside components**
2. **Define components at module level** (top of file)
3. **Use props or context** to share state
4. **ESLint rule**: `react/no-unstable-nested-components` catches this

---

## Pitfall 10: Using Index as Key

### Symptom

Wrong items get updated or deleted. Component state gets mixed up. Animations apply to wrong items. Checkboxes check wrong items after reordering.

### Browser Evidence

**User Experience**:
1. User checks checkbox on "Task 1"
2. User deletes "Task 1"
3. Checkbox appears checked on "Task 2" (wrong item!)

**React DevTools - Components Tab**:
- Keys are: `0, 1, 2, 3`
- After deletion: `0, 1, 2` (same keys, different items)
- React reuses components with same keys

```tsx
// TaskManager.tsx - WRONG: Index as key
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([
    { id: '1', title: 'Task 1', completed: false, priority: 'high', createdAt: new Date() },
    { id: '2', title: 'Task 2', completed: false, priority: 'medium', createdAt: new Date() },
    { id: '3', title: 'Task 3', completed: false, priority: 'low', createdAt: new Date() },
  ]);

  const handleDeleteTask = (index: number) => {
    setTasks(tasks.filter((_, i) => i !== index));
  };

  return (
    <div>
      {tasks.map((task, index) => (
        // BUG: Using index as key
        <TaskItem
          key={index}
          task={task}
          onDelete={() => handleDeleteTask(index)}
        />
      ))}
    </div>
  );
}

function TaskItem({ task, onDelete }: { task: Task; onDelete: () => void }) {
  const [isChecked, setIsChecked] = useState(false);
  
  return (
    <div>
      <input 
        type="checkbox"
        checked={isChecked}
        onChange={(e) => setIsChecked(e.target.checked)}
      />
      <span>{task.title}</span>
      <button onClick={onDelete}>Delete</button>
    </div>
  );
}
```

### Root Cause

React uses keys to identify which items have changed, been added, or been removed. When you use index as key:

**Before deletion**:
```
key=0 → Task 1 (checkbox: unchecked)
key=1 → Task 2 (checkbox: checked)
key=2 → Task 3 (checkbox: unchecked)
```

**After deleting Task 1**:
```
key=0 → Task 2 (React reuses key=0 component, keeps its state: unchecked)
key=1 → Task 3 (React reuses key=1 component, keeps its state: checked)
```

React thinks:
- "key=0 is still here, just update its props"
- "key=1 is still here, just update its props"
- "key=2 is gone, unmount it"

But the component state (checkbox) stays with the key, not the data!

### The Trap

You use index as key because:
1. It's the easiest thing to reach for
2. It "works" when you don't reorder/delete items
3. You don't understand what keys are for
4. ESLint doesn't warn (it only warns about missing keys)

### The Fix: Use Stable, Unique IDs

Always use a stable, unique identifier from your data:

```tsx
// TaskManager.tsx - CORRECT: Stable ID as key
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([
    { id: '1', title: 'Task 1', completed: false, priority: 'high', createdAt: new Date() },
    { id: '2', title: 'Task 2', completed: false, priority: 'medium', createdAt: new Date() },
    { id: '3', title: 'Task 3', completed: false, priority: 'low', createdAt: new Date() },
  ]);

  const handleDeleteTask = (id: string) => {
    setTasks(tasks.filter(task => task.id !== id));
  };

  return (
    <div>
      {tasks.map(task => (
        // CORRECT: Using stable ID as key
        <TaskItem
          key={task.id}
          task={task}
          onDelete={() => handleDeleteTask(task.id)}
        />
      ))}
    </div>
  );
}
```

### When Index as Key is Acceptable

Index as key is safe **only when ALL of these are true**:
1. The list is static (never reordered, added to, or removed from)
2. Items have no state
3. Items have no animations

Example of safe usage:

```tsx
// Safe: Static list, no state, no reordering
function ColorPalette() {
  const colors = ['red', 'blue', 'green']; // Never changes
  
  return (
    <div>
      {colors.map((color, index) => (
        <div key={index} style={{ backgroundColor: color }}>
          {color}
        </div>
      ))}
    </div>
  );
}
```

### Generating Stable IDs

If your data doesn't have IDs, generate them once:

```tsx
// Generate IDs when creating items
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  const handleAddTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(), // Generate stable ID
      title,
      completed: false,
      priority: 'medium',
      createdAt: new Date(),
    };
    
    setTasks([...tasks, newTask]);
  };

  return (
    <div>
      {tasks.map(task => (
        <TaskItem key={task.id} task={task} />
      ))}
    </div>
  );
}
```

### Prevention

1. **Always use stable, unique IDs as keys**
2. **Never use index as key** unless the list is truly static
3. **Generate IDs** when creating items if they don't have them
4. **Use `crypto.randomUUID()`** or a library like `nanoid`
5. **Test reordering and deletion** to catch key issues

---

## Pitfall 11: Unnecessary Re-renders from Object/Array Props

### Symptom

Child component re-renders even though its props "haven't changed." Performance degrades with many child components. React DevTools Profiler shows unnecessary renders.

### Browser Evidence

**React DevTools - Profiler**:
- Parent renders
- All children render (highlighted in yellow)
- Props appear identical in Components tab
- But references are different

**Console logs**:
```
Parent render
Child render (props: { filter: 'all', sort: 'date' })
Parent render
Child render (props: { filter: 'all', sort: 'date' })
// Props look the same, but child still re-renders
```

```tsx
// TaskManager.tsx - WRONG: New object on every render
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filterValue, setFilterValue] = useState('all');
  const [sortValue, setSortValue] = useState('date');

  return (
    <div>
      {/* BUG: New object created on every render */}
      <TaskFilters 
        config={{ filter: filterValue, sort: sortValue }}
        onConfigChange={(config) => {
          setFilterValue(config.filter);
          setSortValue(config.sort);
        }}
      />
      
      {/* BUG: New array created on every render */}
      <TaskList 
        tasks={tasks}
        priorities={['low', 'medium', 'high']}
      />
    </div>
  );
}

// Child component with React.memo
const TaskFilters = React.memo(function TaskFilters({ 
  config, 
  onConfigChange 
}: {
  config: { filter: string; sort: string };
  onConfigChange: (config: { filter: string; sort: string }) => void;
}) {
  console.log('TaskFilters render');
  return <div>{/* ... */}</div>;
});
```

### Root Cause

JavaScript objects and arrays are compared by **reference**, not by value. Even if two objects have identical contents, they're different if they're different instances:

```typescript
const obj1 = { filter: 'all', sort: 'date' };
const obj2 = { filter: 'all', sort: 'date' };

obj1 === obj2 // false! Different references
```

When you create a new object/array on every render, React.memo sees it as a "different" prop and re-renders the child.

### The Trap

You create new objects/arrays because:
1. It's convenient: `config={{ filter, sort }}`
2. You don't realize it creates a new reference
3. You think React.memo will "figure it out" (it won't)
4. You don't profile to see the unnecessary renders

### The Fix: Stabilize References with useMemo

Use useMemo to create stable references:

```tsx
// TaskManager.tsx - CORRECT: Stable references
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filterValue, setFilterValue] = useState('all');
  const [sortValue, setSortValue] = useState('date');

  // Stable config object
  const config = useMemo(
    () => ({ filter: filterValue, sort: sortValue }),
    [filterValue, sortValue]
  );

  // Stable callback
  const handleConfigChange = useCallback(
    (newConfig: { filter: string; sort: string }) => {
      setFilterValue(newConfig.filter);
      setSortValue(newConfig.sort);
    },
    []
  );

  // Stable array (only create once)
  const priorities = useMemo(
    () => ['low', 'medium', 'high'] as const,
    []
  );

  return (
    <div>
      <TaskFilters 
        config={config}
        onConfigChange={handleConfigChange}
      />
      
      <TaskList 
        tasks={tasks}
        priorities={priorities}
      />
    </div>
  );
}
```

### Alternative: Pass Primitive Values

Instead of objects, pass primitive values:

```tsx
// TaskManager.tsx - Alternative: Primitives
function TaskManager() {
  const [filterValue, setFilterValue] = useState('all');
  const [sortValue, setSortValue] = useState('date');

  return (
    <div>
      {/* Primitives are compared by value, not reference */}
      <TaskFilters 
        filter={filterValue}
        sort={sortValue}
        onFilterChange={setFilterValue}
        onSortChange={setSortValue}
      />
    </div>
  );
}

const TaskFilters = React.memo(function TaskFilters({ 
  filter,
  sort,
  onFilterChange,
  onSortChange
}: {
  filter: string;
  sort: string;
  onFilterChange: (filter: string) => void;
  onSortChange: (sort: string) => void;
}) {
  console.log('TaskFilters render');
  return <div>{/* ... */}</div>;
});
```

### When to Optimize

Don't optimize prematurely. Only use useMemo/useCallback when:
1. **Profiling shows** unnecessary re-renders
2. Child component is **expensive to render**
3. Child component is **memoized** with React.memo
4. List has **many items** (50+)

### Prevention

1. **Profile first** with React DevTools Profiler
2. **Pass primitives** when possible
3. **Use useMemo** for objects/arrays passed to memoized components
4. **Use useCallback** for functions passed to memoized components
5. **Don't optimize** until you have evidence of a problem

## TypeScript Pitfalls

## Pitfall 12: Type Assertions Hiding Bugs

### Symptom

Runtime errors that TypeScript should have caught. "Cannot read property of undefined" errors. Type errors in production but not development.

### Browser Evidence

**Browser Console**:
```
Uncaught TypeError: Cannot read properties of undefined (reading 'title')
    at TaskItem.tsx:15
```

**TypeScript Compiler**:
```
No errors found.
```

**The disconnect**: TypeScript thinks the code is safe, but it crashes at runtime.

```tsx
// TaskManager.tsx - WRONG: Type assertion hiding bug
interface Task {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: Date;
}

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    fetch('/api/tasks')
      .then(res => res.json())
      .then(data => {
        // BUG: Type assertion without validation
        setTasks(data as Task[]);
      });
  }, []);

  return (
    <div>
      {tasks.map(task => (
        <div key={task.id}>
          {/* Runtime error if task.title is undefined */}
          <h3>{task.title.toUpperCase()}</h3>
        </div>
      ))}
    </div>
  );
}
```

### Root Cause

Type assertions (`as Type`) tell TypeScript "trust me, I know this is the right type." But TypeScript doesn't verify this—it just believes you. If the runtime data doesn't match the asserted type, you get runtime errors.

```typescript
const data = { id: '1', name: 'Task' }; // Missing 'title'
const task = data as Task; // TypeScript: "OK, it's a Task"
console.log(task.title.toUpperCase()); // Runtime error!
```

### The Trap

You use type assertions because:
1. TypeScript complains and you want it to stop
2. You "know" the data is correct
3. You're dealing with external data (API, localStorage)
4. You think `as` is the same as validation (it's not)

### The Fix: Validate at Runtime

Use runtime validation with a library like Zod:

```tsx
// TaskManager.tsx - CORRECT: Runtime validation
import { z } from 'zod';

// Define schema that matches your type
const TaskSchema = z.object({
  id: z.string(),
  title: z.string(),
  completed: z.boolean(),
  priority: z.enum(['low', 'medium', 'high']),
  createdAt: z.coerce.date(), // Coerce string to Date
});

const TaskArraySchema = z.array(TaskSchema);

type Task = z.infer<typeof TaskSchema>;

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('/api/tasks')
      .then(res => res.json())
      .then(data => {
        // Validate at runtime
        const result = TaskArraySchema.safeParse(data);
        
        if (result.success) {
          setTasks(result.data);
        } else {
          console.error('Invalid task data:', result.error);
          setError('Failed to load tasks: invalid data format');
        }
      })
      .catch(err => {
        console.error('Fetch error:', err);
        setError('Failed to load tasks');
      });
  }, []);

  if (error) {
    return <div>Error: {error}</div>;
  }

  return (
    <div>
      {tasks.map(task => (
        <div key={task.id}>
          {/* Safe: TypeScript AND runtime guarantee title exists */}
          <h3>{task.title.toUpperCase()}</h3>
        </div>
      ))}
    </div>
  );
}
```

### Alternative: Type Guards

For simpler cases, write type guard functions:

```typescript
// Type guard function
function isTask(value: unknown): value is Task {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'string' &&
    'title' in value &&
    typeof value.title === 'string' &&
    'completed' in value &&
    typeof value.completed === 'boolean' &&
    'priority' in value &&
    ['low', 'medium', 'high'].includes(value.priority as string) &&
    'createdAt' in value &&
    value.createdAt instanceof Date
  );
}

function isTaskArray(value: unknown): value is Task[] {
  return Array.isArray(value) && value.every(isTask);
}

// Usage
fetch('/api/tasks')
  .then(res => res.json())
  .then(data => {
    if (isTaskArray(data)) {
      setTasks(data); // TypeScript knows it's Task[]
    } else {
      setError('Invalid task data');
    }
  });
```

### When Type Assertions Are Acceptable

Type assertions are safe when:
1. You're narrowing a type you already know: `event.target as HTMLInputElement`
2. You're working with DOM APIs: `document.getElementById('root') as HTMLDivElement`
3. You're dealing with library types that are too strict

But even then, prefer type guards when possible.

### Prevention

1. **Never use `as` with external data** (API, localStorage, user input)
2. **Use Zod or similar** for runtime validation
3. **Write type guards** for complex validation logic
4. **Prefer `unknown`** over `any` to force validation
5. **ESLint rule**: `@typescript-eslint/consistent-type-assertions` to restrict `as`

---

## Pitfall 13: The `any` Escape Hatch

### Symptom

Type errors that should be caught at compile time appear at runtime. Autocomplete stops working. Refactoring breaks things silently.

### Browser Evidence

**Browser Console**:
```
Uncaught TypeError: task.toggleComplete is not a function
    at TaskItem.tsx:23
```

**TypeScript Compiler**:
```
No errors found.
```

**VS Code**:
- No autocomplete for `task` properties
- No error when calling non-existent method

```tsx
// TaskManager.tsx - WRONG: any disables type checking
function TaskManager() {
  const [tasks, setTasks] = useState<any[]>([]); // BUG: any

  useEffect(() => {
    fetch('/api/tasks')
      .then(res => res.json())
      .then((data: any) => { // BUG: any
        setTasks(data);
      });
  }, []);

  const handleToggle = (task: any) => { // BUG: any
    // TypeScript doesn't catch this error
    task.toggleComplete(); // Runtime error: method doesn't exist
  };

  return (
    <div>
      {tasks.map((task: any) => ( // BUG: any
        <div key={task.id}>
          <h3>{task.title}</h3>
          <button onClick={() => handleToggle(task)}>Toggle</button>
        </div>
      ))}
    </div>
  );
}
```

### Root Cause

`any` disables all type checking. TypeScript treats `any` as "I don't care about types here." This defeats the entire purpose of TypeScript.

```typescript
const task: any = { id: '1', title: 'Task' };

task.title.toUpperCase(); // OK
task.nonExistent.method(); // OK (but crashes at runtime!)
task.anything.you.want.here(); // OK (but crashes at runtime!)
```

### The Trap

You use `any` because:
1. TypeScript errors are frustrating
2. You don't know the correct type
3. You're dealing with complex third-party types
4. You think "I'll fix it later" (you won't)

### The Fix: Use Proper Types

Define proper types for your data:

```tsx
// TaskManager.tsx - CORRECT: Proper types
interface Task {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: Date;
}

function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    fetch('/api/tasks')
      .then(res => res.json())
      .then((data: unknown) => { // Use unknown, not any
        const result = TaskArraySchema.safeParse(data);
        if (result.success) {
          setTasks(result.data);
        }
      });
  }, []);

  const handleToggle = (task: Task) => {
    // TypeScript catches this error at compile time
    task.toggleComplete(); // Error: Property 'toggleComplete' does not exist
    
    // Correct approach
    setTasks(tasks.map(t => 
      t.id === task.id 
        ? { ...t, completed: !t.completed }
        : t
    ));
  };

  return (
    <div>
      {tasks.map(task => (
        <div key={task.id}>
          <h3>{task.title}</h3>
          <button onClick={() => handleToggle(task)}>Toggle</button>
        </div>
      ))}
    </div>
  );
}
```

### Use `unknown` Instead of `any`

When you truly don't know the type, use `unknown`:

```typescript
// WRONG: any allows anything
function processData(data: any) {
  return data.value.toUpperCase(); // No error, crashes at runtime
}

// CORRECT: unknown forces validation
function processData(data: unknown) {
  // Error: Object is of type 'unknown'
  return data.value.toUpperCase();
  
  // Must validate first
  if (
    typeof data === 'object' &&
    data !== null &&
    'value' in data &&
    typeof data.value === 'string'
  ) {
    return data.value.toUpperCase(); // OK now
  }
  
  throw new Error('Invalid data');
}
```

### When `any` is Acceptable (Rarely)

`any` is acceptable when:
1. **Migrating JavaScript to TypeScript**: Use `any` temporarily, add `// TODO: type this` comment
2. **Dealing with truly dynamic data**: JSON.parse result (but validate immediately)
3. **Working around library bugs**: When library types are wrong (but file an issue)

### Prevention

1. **Ban `any` in your codebase**: Use ESLint rule `@typescript-eslint/no-explicit-any`
2. **Use `unknown`** when you don't know the type
3. **Define proper interfaces** for your data
4. **Use Zod** to generate types from schemas
5. **Gradually type** existing code instead of using `any`

## Next.js-Specific Pitfalls

## Pitfall 14: Using Client-Only APIs in Server Components

### Symptom

Build fails with cryptic errors. "window is not defined" or "document is not defined" errors. Component works in development but fails in production build.

### Browser Evidence

**Terminal Output** (during build):
```bash
Error: Unhandled Runtime Error
ReferenceError: window is not defined

Call Stack:
  TaskManager.tsx (15:23)
```

**Development**: Works fine
**Production build**: Fails

```tsx
// app/tasks/page.tsx - WRONG: Client API in Server Component
export default function TasksPage() {
  // BUG: Server Components can't access window
  const savedFilter = window.localStorage.getItem('taskFilter');
  
  // BUG: Server Components can't access document
  const theme = document.body.classList.contains('dark') ? 'dark' : 'light';

  return (
    <div>
      <h1>Tasks</h1>
      <p>Filter: {savedFilter}</p>
      <p>Theme: {theme}</p>
    </div>
  );
}
```

### Root Cause

Next.js App Router uses **Server Components by default**. Server Components render on the server (Node.js), which doesn't have browser APIs like `window`, `document`, `localStorage`, etc.

In development, Next.js might render on the client for fast refresh, hiding the issue. In production, it fails during build.

### The Trap

You use browser APIs because:
1. You forget you're in a Server Component
2. You're used to client-only React
3. Development works fine (false confidence)
4. The error message is confusing

### The Fix: Use Client Components

Add `'use client'` directive for components that need browser APIs:

```tsx
// app/tasks/page.tsx - Server Component (no browser APIs)
import { TaskList } from './TaskList';

export default function TasksPage() {
  // Server Component: Can fetch data, but no browser APIs
  const tasks = await fetchTasks();

  return (
    <div>
      <h1>Tasks</h1>
      <TaskList initialTasks={tasks} />
    </div>
  );
}

// components/TaskList.tsx - Client Component
'use client';

import { useState, useEffect } from 'react';

export function TaskList({ initialTasks }: { initialTasks: Task[] }) {
  const [tasks, setTasks] = useState(initialTasks);
  const [filter, setFilter] = useState('all');

  useEffect(() => {
    // Client Component: Can use browser APIs
    const savedFilter = localStorage.getItem('taskFilter');
    if (savedFilter) {
      setFilter(savedFilter);
    }
  }, []);

  useEffect(() => {
    localStorage.setItem('taskFilter', filter);
  }, [filter]);

  return (
    <div>
      {/* Client-side interactivity */}
      <select value={filter} onChange={(e) => setFilter(e.target.value)}>
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="completed">Completed</option>
      </select>
      
      {tasks.map(task => (
        <div key={task.id}>{task.title}</div>
      ))}
    </div>
  );
}
```

### Decision Framework: Server vs. Client Component

**Use Server Component when**:
- Fetching data from database/API
- Accessing backend resources
- Keeping sensitive logic on server
- Reducing client bundle size
- No interactivity needed

**Use Client Component when**:
- Using browser APIs (localStorage, window, document)
- Using React hooks (useState, useEffect, useContext)
- Handling user interactions (onClick, onChange)
- Using browser-only libraries

### Prevention

1. **Default to Server Components** in App Router
2. **Add `'use client'`** only when needed
3. **Check build output** for errors before deploying
4. **Use `typeof window !== 'undefined'`** for conditional browser code
5. **Test production builds** locally with `npm run build && npm start`

---

## Pitfall 15: Hydration Mismatches

### Symptom

Warning in console: "Hydration failed because the initial UI does not match what was rendered on the server." Content flashes or changes after page load. Styles apply incorrectly initially.

### Browser Evidence

**Browser Console**:
```
Warning: Hydration failed because the initial UI does not match 
what was rendered on the server.

Warning: Expected server HTML to contain a matching <div> in <div>.
```

**User Experience**:
- Page loads with one layout
- Suddenly shifts to different layout
- Flash of unstyled content
- Cumulative Layout Shift (CLS) issues

```tsx
// app/tasks/page.tsx - WRONG: Hydration mismatch
export default function TasksPage() {
  return (
    <div>
      <h1>Tasks</h1>
      {/* BUG: Different content on server vs. client */}
      <p>Current time: {new Date().toLocaleTimeString()}</p>
      
      {/* BUG: Random content differs between renders */}
      <p>Random number: {Math.random()}</p>
      
      {/* BUG: Browser-only API returns different values */}
      <p>Screen width: {window.innerWidth}px</p>
    </div>
  );
}
```

### Root Cause

Next.js renders your component twice:
1. **On the server**: Generates HTML
2. **On the client**: "Hydrates" the HTML (attaches event listeners, state)

If the server HTML doesn't match the client HTML, React can't hydrate correctly. This happens when:
- Using `Date.now()`, `Math.random()`, or other non-deterministic values
- Using browser APIs that don't exist on server
- Conditionally rendering based on browser state

### The Trap

You create hydration mismatches because:
1. You don't understand server vs. client rendering
2. You use time-based or random values
3. You check browser state during render
4. You use third-party libraries that assume browser environment

### The Fix: Ensure Consistent Rendering

**Option 1**: Move dynamic content to Client Component with useEffect:

```tsx
// components/CurrentTime.tsx - Client Component
'use client';

import { useState, useEffect } from 'react';

export function CurrentTime() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    // Only set time on client, after hydration
    setTime(new Date().toLocaleTimeString());
    
    const interval = setInterval(() => {
      setTime(new Date().toLocaleTimeString());
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);

  // Server renders null, client renders time after hydration
  if (!time) return <p>Loading time...</p>;
  
  return <p>Current time: {time}</p>;
}

// app/tasks/page.tsx - Server Component
import { CurrentTime } from '@/components/CurrentTime';

export default function TasksPage() {
  return (
    <div>
      <h1>Tasks</h1>
      <CurrentTime />
    </div>
  );
}
```

**Option 2**: Use `suppressHydrationWarning` for known mismatches:

```tsx
// Only use this when you KNOW the mismatch is intentional
export default function TasksPage() {
  return (
    <div>
      <h1>Tasks</h1>
      <p suppressHydrationWarning>
        Current time: {new Date().toLocaleTimeString()}
      </p>
    </div>
  );
}
```

**Option 3**: Use consistent data from server:

```tsx
// app/tasks/page.tsx - Pass server time to client
export default function TasksPage() {
  const serverTime = new Date().toISOString();

  return (
    <div>
      <h1>Tasks</h1>
      <TimeDisplay initialTime={serverTime} />
    </div>
  );
}

// components/TimeDisplay.tsx
'use client';

export function TimeDisplay({ initialTime }: { initialTime: string }) {
  const [time, setTime] = useState(initialTime);

  useEffect(() => {
    const interval = setInterval(() => {
      setTime(new Date().toISOString());
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);

  return <p>Time: {new Date(time).toLocaleTimeString()}</p>;
}
```

### Common Hydration Mismatch Causes

**1. Browser-only APIs**:
```tsx
// WRONG
<p>Width: {window.innerWidth}px</p>

// CORRECT
'use client';
const [width, setWidth] = useState<number | null>(null);
useEffect(() => {
  setWidth(window.innerWidth);
}, []);
```

**2. Date/Time**:
```tsx
// WRONG
<p>{new Date().toLocaleString()}</p>

// CORRECT
Pass server time as prop, update in useEffect
```

**3. Random values**:
```tsx
// WRONG
<div key={Math.random()}>...</div>

// CORRECT
Use stable IDs from data
```

**4. Conditional rendering based on browser state**:
```tsx
// WRONG
{typeof window !== 'undefined' && <Component />}

// CORRECT
Use useEffect to conditionally render after hydration
```

### Prevention

1. **Avoid non-deterministic values** during render
2. **Use useEffect** for browser-only logic
3. **Pass server data** to client components as props
4. **Test with JavaScript disabled** to see server HTML
5. **Check Lighthouse** for CLS issues caused by hydration mismatches

---

## Pitfall 16: Mixing Server and Client Component Patterns

### Symptom

"You're importing a component that needs X. It only works in a Client Component but none of its parents are marked with 'use client'." Build errors about importing client-only code in server components.

### Browser Evidence

**Terminal Output**:
```bash
Error: You're importing a component that needs useState. 
It only works in a Client Component but none of its parents 
are marked with "use client", so they're Server Components by default.

Import trace:
  ./app/tasks/page.tsx
  ./components/TaskList.tsx
  ./components/TaskItem.tsx
```

```tsx
// app/tasks/page.tsx - Server Component
import { TaskList } from '@/components/TaskList';

export default async function TasksPage() {
  const tasks = await fetchTasks();
  
  return (
    <div>
      <h1>Tasks</h1>
      <TaskList tasks={tasks} />
    </div>
  );
}

// components/TaskList.tsx - Server Component (no 'use client')
import { TaskItem } from './TaskItem';

export function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <div>
      {tasks.map(task => (
        <TaskItem key={task.id} task={task} />
      ))}
    </div>
  );
}

// components/TaskItem.tsx - BUG: Uses hooks but no 'use client'
import { useState } from 'react';

export function TaskItem({ task }: { task: Task }) {
  // BUG: useState in Server Component
  const [isExpanded, setIsExpanded] = useState(false);
  
  return (
    <div>
      <h3>{task.title}</h3>
      <button onClick={() => setIsExpanded(!isExpanded)}>
        {isExpanded ? 'Collapse' : 'Expand'}
      </button>
      {isExpanded && <p>{task.description}</p>}
    </div>
  );
}
```

### Root Cause

In Next.js App Router:
- Components are **Server Components by default**
- Server Components **cannot use hooks** (useState, useEffect, etc.)
- Server Components **cannot use browser APIs**
- When a Server Component imports a component that uses hooks, build fails

The `'use client'` directive marks a component and **all its children** as Client Components.

### The Trap

You mix patterns because:
1. You forget to add `'use client'`
2. You don't understand the Server/Client boundary
3. You import a Client Component from a Server Component (this is fine)
4. You import a Server Component from a Client Component (this is problematic)

### The Fix: Add 'use client' to Interactive Components

Mark components that need interactivity as Client Components:

```tsx
// app/tasks/page.tsx - Server Component (can fetch data)
import { TaskList } from '@/components/TaskList';

export default async function TasksPage() {
  const tasks = await fetchTasks();
  
  return (
    <div>
      <h1>Tasks</h1>
      <TaskList tasks={tasks} />
    </div>
  );
}

// components/TaskList.tsx - Server Component (no interactivity)
import { TaskItem } from './TaskItem';

export function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <div>
      {tasks.map(task => (
        <TaskItem key={task.id} task={task} />
      ))}
    </div>
  );
}

// components/TaskItem.tsx - CORRECT: Client Component
'use client';

import { useState } from 'react';

export function TaskItem({ task }: { task: Task }) {
  const [isExpanded, setIsExpanded] = useState(false);
  
  return (
    <div>
      <h3>{task.title}</h3>
      <button onClick={() => setIsExpanded(!isExpanded)}>
        {isExpanded ? 'Collapse' : 'Expand'}
      </button>
      {isExpanded && <p>{task.description}</p>}
    </div>
  );
}
```

### Component Composition Patterns

**Pattern 1: Server Component → Client Component** (✅ Allowed):
```tsx
// Server Component
export default function Page() {
  return <ClientComponent />;
}

// Client Component
'use client';
export function ClientComponent() {
  const [state, setState] = useState();
  return <div>...</div>;
}
```

**Pattern 2: Client Component → Server Component** (❌ Not directly):
```tsx
// WRONG: Can't import Server Component into Client Component
'use client';
export function ClientComponent() {
  return <ServerComponent />; // Error!
}
```

**Pattern 3: Client Component with Server Component as children** (✅ Allowed):
```tsx
// Server Component
export default function Page() {
  return (
    <ClientWrapper>
      <ServerComponent />
    </ClientWrapper>
  );
}

// Client Component
'use client';
export function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState();
  return <div>{children}</div>;
}
```

### Decision Framework: Where to Place 'use client'

**Place `'use client'` as low as possible in the tree**:
- ✅ Only mark interactive components
- ✅ Keep data fetching in Server Components
- ✅ Pass data down as props

**Example**:
```tsx
// ✅ GOOD: Client boundary at leaf
export default async function Page() {
  const data = await fetchData(); // Server
  return (
    <div>
      <StaticHeader /> {/* Server */}
      <InteractiveWidget data={data} /> {/* Client */}
    </div>
  );
}

// ❌ BAD: Client boundary at root
'use client';
export default function Page() {
  const [data, setData] = useState();
  useEffect(() => {
    fetchData().then(setData); // Client-side fetch
  }, []);
  return (
    <div>
      <StaticHeader /> {/* Unnecessarily client-side */}
      <InteractiveWidget data={data} />
    </div>
  );
}
```

### Prevention

1. **Default to Server Components**
2. **Add `'use client'` only when needed** (hooks, browser APIs, interactivity)
3. **Push `'use client'` down the tree** as far as possible
4. **Pass data as props** from Server to Client Components
5. **Use composition** to nest Server Components inside Client Components

## Performance Pitfalls

## Pitfall 17: Premature Optimization

### Symptom

Code is complex and hard to maintain. Performance is not noticeably better. Time spent optimizing instead of building features. Over-use of useMemo, useCallback, React.memo.

### Browser Evidence

**React DevTools - Profiler**:
- Render time: 2.3ms (before optimization)
- Render time: 2.1ms (after optimization)
- Improvement: 0.2ms (negligible)

**Code Complexity**:
- Before: 50 lines, easy to understand
- After: 120 lines, wrapped in useMemo/useCallback everywhere

```tsx
// TaskManager.tsx - WRONG: Over-optimized
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState('all');
  const [sort, setSort] = useState('date');

  // Unnecessary useMemo for simple filter
  const filteredTasks = useMemo(() => {
    return tasks.filter(task => {
      if (filter === 'active') return !task.completed;
      if (filter === 'completed') return task.completed;
      return true;
    });
  }, [tasks, filter]);

  // Unnecessary useMemo for simple sort
  const sortedTasks = useMemo(() => {
    return [...filteredTasks].sort((a, b) => {
      if (sort === 'date') {
        return b.createdAt.getTime() - a.createdAt.getTime();
      }
      return a.title.localeCompare(b.title);
    });
  }, [filteredTasks, sort]);

  // Unnecessary useCallback for simple function
  const handleToggle = useCallback((id: string) => {
    setTasks(tasks.map(task =>
      task.id === id ? { ...task, completed: !task.completed } : task
    ));
  }, [tasks]);

  // Unnecessary useCallback for simple function
  const handleDelete = useCallback((id: string) => {
    setTasks(tasks.filter(task => task.id !== id));
  }, [tasks]);

  return (
    <div>
      <TaskFilters 
        filter={filter} 
        onFilterChange={setFilter}
        sort={sort}
        onSortChange={setSort}
      />
      <TaskList 
        tasks={sortedTasks}
        onToggle={handleToggle}
        onDelete={handleDelete}
      />
    </div>
  );
}

// Unnecessary React.memo for simple component
const TaskList = React.memo(function TaskList({ 
  tasks, 
  onToggle, 
  onDelete 
}: {
  tasks: Task[];
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}) {
  return (
    <div>
      {tasks.map(task => (
        <TaskItem 
          key={task.id}
          task={task}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </div>
  );
});
```

### Root Cause

Premature optimization makes code more complex without measurable benefit. The overhead of useMemo/useCallback (checking dependencies, comparing values) can be **more expensive** than the computation you're trying to optimize.

React is already fast. Most components render in <5ms. Optimizing a 2ms render to 1ms is not worth the complexity.

### The Trap

You over-optimize because:
1. You've heard "always use useMemo/useCallback"
2. You think optimization is always good
3. You don't measure before optimizing
4. You follow cargo-cult patterns without understanding

### The Fix: Optimize Only When Needed

Start simple, optimize only when profiling shows a problem:

```tsx
// TaskManager.tsx - CORRECT: Simple, readable, fast enough
function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [filter, setFilter] = useState('all');
  const [sort, setSort] = useState('date');

  // Compute during render - fast enough for most cases
  const filteredTasks = tasks.filter(task => {
    if (filter === 'active') return !task.completed;
    if (filter === 'completed') return task.completed;
    return true;
  });

  const sortedTasks = [...filteredTasks].sort((a, b) => {
    if (sort === 'date') {
      return b.createdAt.getTime() - a.createdAt.getTime();
    }
    return a.title.localeCompare(b.title);
  });

  const handleToggle = (id: string) => {
    setTasks(tasks.map(task =>
      task.id === id ? { ...task, completed: !task.completed } : task
    ));
  };

  const handleDelete = (id: string) => {
    setTasks(tasks.filter(task => task.id !== id));
  };

  return (
    <div>
      <TaskFilters 
        filter={filter} 
        onFilterChange={setFilter}
        sort={sort}
        onSortChange={setSort}
      />
      <TaskList 
        tasks={sortedTasks}
        onToggle={handleToggle}
        onDelete={handleDelete}
      />
    </div>
  );
}

// No React.memo needed unless profiling shows re-render issues
function TaskList({ 
  tasks, 
  onToggle, 
  onDelete 
}: {
  tasks: Task[];
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}) {
  return (
    <div>
      {tasks.map(task => (
        <TaskItem 
          key={task.id}
          task={task}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </div>
  );
}
```

### When to Actually Optimize

Optimize when **profiling shows** these issues:

**1. Expensive computation** (>50ms):
```tsx
// Profile shows this takes 80ms
const expensiveResult = computeExpensiveValue(data);

// Optimize with useMemo
const expensiveResult = useMemo(
  () => computeExpensiveValue(data),
  [data]
);
```

**2. Large lists** (>100 items):
```tsx
// 1000 items, each re-renders unnecessarily
const TaskItem = React.memo(function TaskItem({ task }) {
  return <div>{task.title}</div>;
});
```

**3. Frequent re-renders** (>10/second):
```tsx
// Parent re-renders every 100ms, child doesn't need to
const ExpensiveChild = React.memo(function ExpensiveChild({ data }) {
  return <ComplexVisualization data={data} />;
});
```

### Optimization Decision Framework

**Before optimizing, ask**:
1. **Is there a performance problem?** (Measure with Profiler)
2. **Is it noticeable to users?** (>100ms is noticeable)
3. **Is this the bottleneck?** (Profile to find the real issue)
4. **Will optimization help?** (Sometimes the problem is elsewhere)

**If yes to all, then optimize. Otherwise, keep it simple.**

### Prevention

1. **Profile first** with React DevTools Profiler
2. **Measure impact** before and after optimization
3. **Start simple**, optimize only when needed
4. **Focus on user-perceived performance** (loading states, responsiveness)
5. **Remember**: Readable code > Premature optimization

## Quick Reference: Diagnostic Decision Tree

## Diagnostic Decision Tree

When your React app breaks, follow this decision tree to identify the issue:

### 1. Component Not Rendering

**Symptom**: Blank screen, component doesn't appear

**Check**:
- Browser console for errors
- React DevTools: Is component in tree?
- Network tab: Did data fetch fail?

**Common causes**:
- JavaScript error (check console)
- Conditional rendering hiding component
- CSS hiding component (check computed styles)
- Data not loaded yet (add loading state)

---

### 2. Component Not Updating

**Symptom**: UI doesn't reflect state changes

**Check**:
- React DevTools: Does state change?
- Console log: Is setState being called?
- React DevTools Profiler: Is component re-rendering?

**Common causes**:
- **Pitfall 2**: Mutating state directly
- **Pitfall 1**: Stale closure in event handler
- **Pitfall 10**: Wrong key causing React to reuse component
- **Pitfall 9**: Component defined inside component

---

### 3. Infinite Renders

**Symptom**: Browser freezes, console floods with logs

**Check**:
- React DevTools Profiler: Render count
- Console: What's logging repeatedly?
- React DevTools: Which component is rendering?

**Common causes**:
- **Pitfall 8**: useEffect with no dependency array
- **Pitfall 5**: useEffect with missing dependencies causing loop
- **Pitfall 11**: New object/array created on every render
- setState inside render (without condition)

---

### 4. Memory Leak Warning

**Symptom**: "Can't perform a React state update on an unmounted component"

**Check**:
- Console: Which component?
- Network tab: Are requests completing after unmount?
- Code: Are there timers or subscriptions?

**Common causes**:
- **Pitfall 7**: Missing cleanup in useEffect
- Fetch request completing after unmount
- Timer/interval not cleared
- Event listener not removed

---

### 5. Type Error at Runtime

**Symptom**: "Cannot read property of undefined", "X is not a function"

**Check**:
- Console: Exact error message and stack trace
- React DevTools: Component props and state
- Network tab: Is API returning expected data?

**Common causes**:
- **Pitfall 12**: Type assertion hiding bug
- **Pitfall 13**: Using `any` instead of proper types
- API returning different data than expected
- Missing null/undefined check

---

### 6. Hydration Mismatch

**Symptom**: "Hydration failed" warning, content flashes

**Check**:
- Console: Hydration warning details
- View source: What's the server HTML?
- React DevTools: What's the client HTML?

**Common causes**:
- **Pitfall 15**: Using Date.now(), Math.random(), or browser APIs
- Conditional rendering based on browser state
- Third-party library assuming browser environment

---

### 7. Build Fails

**Symptom**: `npm run build` fails with error

**Check**:
- Terminal: Full error message
- Error location: File and line number
- TypeScript errors vs. runtime errors

**Common causes**:
- **Pitfall 14**: Using browser APIs in Server Component
- **Pitfall 16**: Missing 'use client' directive
- TypeScript errors
- Missing environment variables

---

### 8. Slow Performance

**Symptom**: App feels sluggish, interactions lag

**Check**:
- React DevTools Profiler: Which components render slowly?
- Chrome Performance tab: What's blocking the main thread?
- Network tab: Are there slow requests?

**Common causes**:
- **Pitfall 17**: Premature optimization making code complex
- Missing React.memo on expensive components
- Large lists without virtualization
- Expensive computation in render
- Too many re-renders

---

## Quick Diagnostic Checklist

When debugging, systematically check:

1. **Browser Console**: Any errors or warnings?
2. **React DevTools - Components**: 
   - Is component in tree?
   - What are props/state values?
   - Is component highlighted (re-rendering)?
3. **React DevTools - Profiler**:
   - How long does render take?
   - How many times does it render?
   - What triggered the render?
4. **Network Tab**:
   - Did requests complete?
   - What's the response data?
   - Are there failed requests?
5. **Console Logs**:
   - Add strategic logs to trace execution
   - Log state before and after updates
   - Log effect runs and cleanups

---

## Common Error Messages Decoded

### "Cannot read properties of undefined (reading 'X')"

**Meaning**: You're accessing a property on `undefined`

**Fix**: Add null check or optional chaining
```tsx
// WRONG
<h1>{user.name}</h1>

// CORRECT
<h1>{user?.name ?? 'Guest'}</h1>
```

---

### "Maximum update depth exceeded"

**Meaning**: Infinite render loop

**Fix**: Check useEffect dependencies and setState calls
```tsx
// WRONG
useEffect(() => {
  setState(value); // Runs on every render
});

// CORRECT
useEffect(() => {
  setState(value);
}, [dependency]); // Only runs when dependency changes
```

---

### "Objects are not valid as a React child"

**Meaning**: Trying to render an object directly

**Fix**: Render object properties, not the object itself
```tsx
// WRONG
<div>{user}</div>

// CORRECT
<div>{user.name}</div>
```

---

### "Each child in a list should have a unique 'key' prop"

**Meaning**: Missing or duplicate keys in list

**Fix**: Use stable, unique IDs as keys
```tsx
// WRONG
{tasks.map((task, index) => <div key={index}>...</div>)}

// CORRECT
{tasks.map(task => <div key={task.id}>...</div>)}
```

---

### "Can't perform a React state update on an unmounted component"

**Meaning**: setState called after component unmounted

**Fix**: Add cleanup to useEffect
```tsx
useEffect(() => {
  let isMounted = true;
  
  fetchData().then(data => {
    if (isMounted) setState(data);
  });
  
  return () => { isMounted = false; };
}, []);
```

---

### "Rendered fewer hooks than expected"

**Meaning**: Conditional hook call

**Fix**: Never call hooks conditionally
```tsx
// WRONG
if (condition) {
  useState(0);
}

// CORRECT
const [state, setState] = useState(0);
if (condition) {
  // Use state here
}
```

## Conclusion: Building Your Debugging Intuition

## Building Your Debugging Intuition

The pitfalls in this appendix represent the most common ways React applications fail. But memorizing them isn't enough—you need to develop **debugging intuition**.

### The Expert's Debugging Process

When an expert React developer encounters a bug, they follow this mental model:

**1. Observe the symptom**
- What does the user see?
- What's the expected behavior?
- What's the actual behavior?

**2. Gather evidence**
- Browser console: Errors, warnings, logs
- React DevTools: Component tree, props, state, render count
- Network tab: Request/response data, timing
- Performance tab: What's blocking the main thread?

**3. Form a hypothesis**
- Based on the evidence, what's the most likely cause?
- Which pitfall does this match?
- What would confirm or refute this hypothesis?

**4. Test the hypothesis**
- Add strategic console.logs
- Use React DevTools to inspect state
- Simplify the code to isolate the issue
- Try the fix and verify it works

**5. Understand the root cause**
- Why did this happen?
- What mechanism caused this failure?
- How can I prevent this in the future?

### From Pitfall to Pattern

Each pitfall teaches a deeper lesson:

- **Pitfall 1 (Stale Closure)** → Understand JavaScript closures and React's render cycle
- **Pitfall 2 (Direct Mutation)** → Understand immutability and reference equality
- **Pitfall 5 (Missing Dependencies)** → Understand effect dependencies and closures
- **Pitfall 7 (Missing Cleanup)** → Understand component lifecycle and memory management
- **Pitfall 10 (Index as Key)** → Understand React's reconciliation algorithm
- **Pitfall 14 (Client APIs in Server)** → Understand Next.js rendering model

### The Path to Mastery

Mastery comes from:
1. **Encountering failures** and understanding why they happened
2. **Reading error messages** systematically to extract all information
3. **Using DevTools** to observe what's actually happening
4. **Building mental models** of how React works under the hood
5. **Recognizing patterns** across different failures

### Your Debugging Toolkit

Keep these tools sharp:

**Browser DevTools**:
- Console: Read every error message carefully
- Network: Understand request/response patterns
- Performance: Profile slow interactions
- Elements: Inspect computed styles and DOM

**React DevTools**:
- Components: Inspect props, state, hooks
- Profiler: Measure render performance
- Highlight updates: See what's re-rendering

**Code Techniques**:
- Strategic console.logs
- Simplification: Remove code until it works
- Isolation: Test components in isolation
- Type checking: Let TypeScript catch errors early

### Final Wisdom

The best debugging skill is **prevention**. Write code that:
- Uses TypeScript to catch errors at compile time
- Validates external data at runtime
- Follows React's rules (immutability, stable keys, proper dependencies)
- Is simple and readable (complexity breeds bugs)

When bugs do occur (and they will), approach them systematically:
1. Don't guess—gather evidence
2. Don't fix symptoms—understand root causes
3. Don't move on—learn the lesson

Every bug you encounter and fix makes you a better React developer. This appendix is your field guide, but experience is your teacher.

**Happy debugging!**
