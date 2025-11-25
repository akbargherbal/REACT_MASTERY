# Chapter 12: React Query: Server State Made Simple

## Client State vs. Server State

## The Two States of Your Application

In modern web applications, not all state is created equal. For years, we've treated all application data as a single entity, often managed by tools like `useState`, `useReducer`, or Redux. This approach, however, overlooks a fundamental distinction that is the source of countless bugs, complex boilerplate, and performance issues: the difference between **Client State** and **Server State**.

Understanding this distinction is the key to writing simpler, more robust React and Next.js applications.

### Client State: What Your App Knows

Client State is the state that is owned and controlled exclusively by the client (the user's browser). It's synchronous, predictable, and completely under your control.

**Characteristics of Client State:**

*   **Owned by the client:** It lives and dies in the browser. No external system can change it without the client's knowledge.
*   **Synchronous:** When you call `setIsOpen(true)`, the state is updated immediately (pending React's next render). There's no network latency.
*   **Predictable:** You know its exact shape and value at all times. It doesn't become "stale" unless your own code changes it.
*   **Ephemeral:** It's typically reset when the user refreshes the page.

**Examples of Client State:**

*   The current value of a form input.
*   Whether a modal dialog is open or closed.
*   The selected tab in a navigation component.
*   The current theme of the application (e.g., 'dark' or 'light').

React's built-in hooks like `useState` and `useReducer` are perfectly designed for managing client state.

### Server State: What Your App Remembers

Server State is a completely different beast. It's a snapshot or a cache of data that is persisted remotely, somewhere you don't directly control. Your application is just "borrowing" it for a while to display to the user.

**Characteristics of Server State:**

*   **Owned by the server:** It's stored in a database, miles away. You can't directly modify it; you can only request changes.
*   **Asynchronous:** Fetching or updating it requires an async API call, which introduces latency, loading states, and potential errors.
*   **Shared:** Multiple clients (and even server processes) can be changing the same data simultaneously.
*   **Can become stale:** The data you have on the client can become outdated at any moment without you knowing. The "source of truth" is remote.

**Examples of Server State:**

*   A user's profile information.
*   A list of products in an e-commerce store.
*   Blog posts fetched from a CMS.
*   The results of a search query.

### The Mismatch: Why `useState` and `useEffect` Fall Short

For years, we've managed server state using client state tools. The typical pattern looks like this:

```tsx
// The classic (and problematic) data fetching pattern
import { useState, useEffect } from 'react';

type User = { id: string; name: string };

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | undefined>(undefined);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setIsLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) {
          throw new Error('Failed to fetch user');
        }
        const data = await response.json();
        setUser(data);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };

    fetchUser();
  }, [userId]); // Refetch when userId changes

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return null;

  return <h1>{user.name}</h1>;
}
```

This code seems reasonable, but it's a house of cards. It fails to address the inherent nature of server state.

**What critical questions does this code NOT answer?**

1.  **Caching:** What if we navigate away and come back to this component? It will re-fetch the same data, even if it hasn't changed.
2.  **Stale Data:** What if the user's name was updated in another browser tab? This component will continue showing the old, stale name until the page is refreshed.
3.  **Background Updates:** Wouldn't it be nice to show the user the stale data while we silently refetch a fresh copy in the background?
4.  **Request Deduplication:** If ten components on the page all need the same user data, will they make ten identical requests? Yes.
5.  **Memory Management:** How do we manage the cache? When do we garbage-collect old data?

Trying to solve these problems with `useState` and `useEffect` leads to a mountain of complex, custom, and often buggy code. We are, in effect, rebuilding a data-fetching client from scratch inside our components.

This is the problem that **TanStack Query (formerly React Query)** was built to solve. It's not just another state management library; it's a **server state management library**. It provides the machinery to handle caching, background updates, stale data, and all the other complexities of server state, letting you focus on your application logic.

## TanStack Query (React Query) fundamentals

## From Manual Fetching to Declarative Queries

Let's see how React Query transforms data fetching from an imperative, manual process into a declarative one. We'll build a simple Todo application and progressively refactor it, demonstrating how React Query solves the problems we identified.

### Phase 1: Establish the Reference Implementation

Our anchor example will be a `TodoApp` component. To focus on the client-side logic, we'll simulate a backend API with simple functions that return Promises.

**Project Structure:**

We'll start with a simple setup.

```
src/
├── api/
│   └── todos.ts       ← Our mock API functions
└── components/
    └── TodoApp.tsx    ← Our reference implementation
```

First, let's define our mock API. This simulates the network latency and potential failures of a real backend.

```typescript
// src/api/todos.ts

export interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

// Simulate a database
let todos: Todo[] = [
  { id: 1, text: 'Learn React', completed: true },
  { id: 2, text: 'Learn React Query', completed: false },
  { id: 3, text: 'Build a cool app', completed: false },
];

let nextId = 4;

// Simulate network delay
const delay = (ms: number) => new Promise(res => setTimeout(res, ms));

export const fetchTodos = async (): Promise<Todo[]> => {
  await delay(1000);
  console.log('API: Fetched Todos', todos);
  return [...todos];
};

export const addTodo = async (text: string): Promise<Todo> => {
  await delay(1000);
  if (Math.random() < 0.1) { // Simulate a 10% chance of failure
    throw new Error('Failed to add todo');
  }
  const newTodo: Todo = { id: nextId++, text, completed: false };
  todos.push(newTodo);
  console.log('API: Added Todo', newTodo);
  return newTodo;
};
```

Now, let's build the `TodoApp` component using the classic `useState` and `useEffect` approach. This is our "naive" implementation that we will improve upon.

**Iteration 0: The `useState` + `useEffect` Approach**

```tsx
// src/components/TodoApp.tsx (Version 0)

import { useState, useEffect } from 'react';
import { fetchTodos, Todo } from '../api/todos';

export function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const getTodos = async () => {
      try {
        setIsLoading(true);
        const data = await fetchTodos();
        setTodos(data);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };
    getTodos();
  }, []); // Fetch only on mount

  if (isLoading) {
    return <div>Loading todos...</div>;
  }

  if (error) {
    return <div>An error occurred: {error.message}</div>;
  }

  return (
    <div>
      <h1>Todo List (useState + useEffect)</h1>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id} style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

This component works, but it's fragile and inefficient. Let's demonstrate its primary failure: it has no concept of "stale" data. If the data changes on the server (or in another part of our app), this component remains blissfully unaware.

### Failure Demonstration: Stale Data

Imagine the user opens your app, and then switches to another tab to check their email. While they are away, a new "Todo" is added via a push notification or another device. When they switch back to your app, they expect to see the latest data. Let's see what happens with our current implementation.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The user sees the initial list of 3 todos. We simulate a data change on the "server" (our mock API). When the user refocuses the browser window, the UI does **not** update. It continues to show the old, stale list of 3 todos. The only way to see the new todo is to manually refresh the entire page.

**Browser Console Output**:
```
API: Fetched Todos [ { id: 1, ... }, { id: 2, ... }, { id: 3, ... } ]
// ...user switches tabs, data changes on server...
// ...user switches back...
// (No new console output, no re-fetch occurs)
```

**React DevTools Evidence**:
- Component tree state: `TodoApp`
- State: `todos` array with 3 items.
- Render count: 1 (on initial load). The component never re-renders when the window is refocused.

**Let's parse this evidence**:

1.  **What the user experiences**: The app feels broken or out of date.
    -   **Expected**: The app should automatically show the latest data when I return to it.
    -   **Actual**: The app shows old data, forcing a manual refresh.

2.  **What the console reveals**: The `fetchTodos` function is only called once, at the very beginning. There are no subsequent calls.

3.  **What DevTools shows**: The component's state remains unchanged after the initial fetch. It has no trigger to re-evaluate its data.

4.  **Root cause identified**: The `useEffect` hook has an empty dependency array (`[]`), so it runs exactly once on component mount and never again.

5.  **Why the current approach can't solve this**: The `useEffect` hook is not designed for managing the lifecycle of server state. It doesn't know about browser focus events, network reconnections, or other heuristics that indicate data might be stale. We would have to build all that logic ourselves.

6.  **What we need**: A hook that understands server state, automatically refetches data when it's likely to be stale, and manages the underlying cache for us.

### Technique Introduced: `useQuery`

This is where React Query shines. Let's refactor our component to use the `useQuery` hook.

First, we need to set up the `QueryClientProvider` at the root of our application. This provides the cache to all child components.

```tsx
// src/app/layout.tsx (or your root component)

'use client'; // Required for providers

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Create a client
const queryClient = new QueryClient();

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <QueryClientProvider client={queryClient}>
          {children}
          <ReactQueryDevtools initialIsOpen={false} />
        </QueryClientProvider>
      </body>
    </html>
  );
}
```

Now we can refactor `TodoApp.tsx`.

### Iteration 1: Refactoring to `useQuery`

**Before** (Iteration 0):

```tsx
// src/components/TodoApp.tsx (Version 0 - excerpt)
import { useState, useEffect } from 'react';
import { fetchTodos, Todo } from '../api/todos';

export function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const getTodos = async () => {
      try {
        setIsLoading(true);
        const data = await fetchTodos();
        setTodos(data);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };
    getTodos();
  }, []);

  // ... render logic using isLoading, error, todos ...
}
```

**After** (Iteration 1):

```tsx
// src/components/TodoApp.tsx (Version 1)

import { useQuery } from '@tanstack/react-query';
import { fetchTodos } from '../api/todos';

export function TodoApp() {
  const { data: todos, isLoading, isError, error } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  if (isLoading) {
    return <div>Loading todos...</div>;
  }

  if (isError) {
    return <div>An error occurred: {error.message}</div>;
  }

  return (
    <div>
      <h1>Todo List (React Query)</h1>
      <ul>
        {todos?.map((todo) => (
          <li key={todo.id} style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Analysis of the Improvement

Look at how much code we deleted!
*   All three `useState` calls are gone.
*   The entire `useEffect` hook is gone.
*   The complex async function inside `useEffect` with its `try/catch/finally` block is gone.

We replaced all of that imperative logic with a single, declarative hook: `useQuery`.

**Let's break down `useQuery`:**

*   `queryKey: ['todos']`: This is the unique identifier for this piece of data. React Query uses this key to cache the data. It's an array, which allows for more complex keys like `['todos', { status: 'completed' }]`.
*   `queryFn: fetchTodos`: This is the asynchronous function that will be called to fetch the data. It must return a promise.

`useQuery` returns an object with all the information we need about the state of our data: `data`, `isLoading`, `isError`, and much more.

### Verification: Solving the Stale Data Problem

Let's re-run our failure scenario with the new `useQuery` implementation.

**Browser Behavior**:
The user sees the initial list of 3 todos. We simulate a data change on the server. When the user refocuses the browser window, the UI **instantly updates** to show the new todo.

**Browser Console Output**:
```
API: Fetched Todos [ { id: 1, ... }, { id: 2, ... }, { id: 3, ... } ]
// ...user switches tabs, data changes on server...
// ...user switches back...
API: Fetched Todos [ { id: 1, ... }, { id: 2, ... }, { id: 3, ... }, { id: 4, ... } ]
```

**React Query DevTools**:
The DevTools are a superpower. When the window is refocused, you can see the `['todos']` query transition from `fresh` -> `fetching` -> `fresh` again, and the data explorer updates with the new item.

**Expected vs. Actual Improvement**:
*   **Expected**: The app should automatically refetch data when the window is refocused.
*   **Actual**: It works perfectly out of the box. `useQuery` automatically handles `refetchOnWindowFocus` (and other triggers like `refetchOnReconnect`) by default.

**Limitation Preview**:
Our app can now display server state robustly. But what happens when we want to *change* that state? How do we add a new todo? This will introduce us to mutations.

## Queries, mutations, and invalidation

## Changing Data with Mutations

Reading data is only half the story. To build a complete application, we need to create, update, and delete data. In React Query, any function that changes data on the server is called a **mutation**.

Let's extend our `TodoApp` to include a form for adding new todos.

### Iteration 2: Adding a Todo (The Wrong Way)

First, let's try to implement this using the tools we know. We'll add a simple form and a `handleSubmit` function that calls our `addTodo` API function.

**Current state recap**: Our component correctly fetches and displays a list of todos, and automatically refetches when the window is focused.
**Current limitation**: We have no way to add a new todo to the list.
**New scenario introduction**: A user types a new todo into an input field and clicks an "Add" button.

```tsx
// src/components/TodoApp.tsx (Version 2 - Naive Mutation)

import { useQuery } from '@tanstack/react-query';
import { fetchTodos, addTodo } from '../api/todos';
import { FormEvent, useState } from 'react';

export function TodoApp() {
  const { data: todos, isLoading, isError, error } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const [newTodoText, setNewTodoText] = useState('');

  const handleSubmit = async (event: FormEvent) => {
    event.preventDefault();
    if (!newTodoText.trim()) return;

    // Call the API to add the new todo
    await addTodo(newTodoText);

    setNewTodoText('');
  };

  // ... (render logic for loading/error states) ...

  return (
    <div>
      <h1>Todo List (React Query)</h1>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={newTodoText}
          onChange={(e) => setNewTodoText(e.target.value)}
          placeholder="Add a new todo"
        />
        <button type="submit">Add Todo</button>
      </form>
      <ul>
        {todos?.map((todo) => (
          /* ... list item ... */
        ))}
      </ul>
    </div>
  );
}
```

Let's run this and see what happens.

### Failure Demonstration: Stale Query Cache

When the user types "Deploy the app" and clicks "Add Todo", the input field clears, but the list of todos on the screen does not change. The new todo is missing.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The user submits the form. The UI gives feedback that the submission happened (the input clears), but the primary data display (the todo list) does not update. It feels like the action failed, even though it succeeded on the server.

**Browser Console Output**:
```
API: Fetched Todos [ { id: 1, ... }, { id: 2, ... }, { id: 3, ... } ]
// ...user adds a new todo...
API: Added Todo { id: 4, text: 'Deploy the app', completed: false }
// (No new "Fetched Todos" log appears)
```

**Network Tab Analysis**:
- A `POST` request (simulated by our `addTodo` function) is successfully made.
- However, no subsequent `GET` request is made to refetch the list of todos.

**React Query DevTools Evidence**:
- The `['todos']` query remains `fresh` and its data array still contains only 3 items. React Query has no idea that the underlying data on the server has changed.

**Let's parse this evidence**:

1.  **What the user experiences**: The app feels broken. They perform an action, but don't see the result.
    -   **Expected**: When I add a todo, it should appear in the list.
    -   **Actual**: The todo is added on the server, but the list on the screen is stale.

2.  **What the console reveals**: The `addTodo` API call was successful. The server state has been updated. The problem is purely on the client.

3.  **What DevTools shows**: The `useQuery` cache for `['todos']` is now out of sync with the server's state of truth. Calling an external `addTodo` function doesn't automatically inform React Query that related data has changed.

4.  **Root cause identified**: We are mutating server state without telling React Query that its cached data is now invalid.

5.  **Why the current approach can't solve this**: We could try to manually refetch, or manually update the local cache, but this gets complicated fast. What if the API call fails? How do we handle loading states for the mutation itself? This leads back to the same boilerplate we escaped from.

6.  **What we need**: A dedicated hook for performing mutations that can be integrated with the query cache, allowing us to invalidate stale data and trigger refetches automatically.

### Technique Introduced: `useMutation` and Query Invalidation

React Query provides the `useMutation` hook for this exact purpose. It wraps our async mutation function and gives us tools to manage its lifecycle. The most powerful tool is **query invalidation**.

When a mutation succeeds, we can tell React Query: "The data associated with this query key is now stale. Please refetch it the next time it's needed."

### Iteration 3: Refactoring to `useMutation`

Let's fix our `TodoApp` component.

**Before** (Iteration 2):

```tsx
// src/components/TodoApp.tsx (Version 2 - excerpt)
const handleSubmit = async (event: FormEvent) => {
  event.preventDefault();
  if (!newTodoText.trim()) return;
  await addTodo(newTodoText); // Fire-and-forget API call
  setNewTodoText('');
};
```

**After** (Iteration 3):

```tsx
// src/components/TodoApp.tsx (Version 3)

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { fetchTodos, addTodo } from '../api/todos';
import { FormEvent, useState } from 'react';

export function TodoApp() {
  const queryClient = useQueryClient(); // Get the query client instance

  const { data: todos, isLoading, isError, error } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const [newTodoText, setNewTodoText] = useState('');

  const addTodoMutation = useMutation({
    mutationFn: addTodo, // The async function to call
    onSuccess: () => {
      // When the mutation is successful, invalidate the 'todos' query
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const handleSubmit = (event: FormEvent) => {
    event.preventDefault();
    if (!newTodoText.trim()) return;
    addTodoMutation.mutate(newTodoText, { // Call the mutation
      onSuccess: () => {
        setNewTodoText(''); // Clear input only on success
      }
    });
  };

  // ... (render logic) ...

  return (
    <div>
      <h1>Todo List (React Query)</h1>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={newTodoText}
          onChange={(e) => setNewTodoText(e.target.value)}
          placeholder="Add a new todo"
          disabled={addTodoMutation.isPending} // Disable input while mutating
        />
        <button type="submit" disabled={addTodoMutation.isPending}>
          {addTodoMutation.isPending ? 'Adding...' : 'Add Todo'}
        </button>
      </form>
      {addTodoMutation.isError && (
        <div style={{ color: 'red' }}>
          An error occurred: {addTodoMutation.error.message}
        </div>
      )}
      {/* ... list rendering ... */}
    </div>
  );
}
```

### Analysis of the Improvement

1.  **`useQueryClient()`**: We get access to the client instance that manages the cache.
2.  **`useMutation({...})`**: We define our mutation.
    *   `mutationFn: addTodo`: We tell it which async function to run.
    *   `onSuccess: () => ...`: This is the magic. We define a callback that runs after the mutation succeeds.
3.  **`queryClient.invalidateQueries({ queryKey: ['todos'] })`**: Inside `onSuccess`, we tell React Query to mark any query whose key matches `['todos']` as stale. This doesn't immediately refetch. It simply marks it. The next time a component renders that is using that query, React Query will see it's stale and trigger a background refetch.
4.  **`addTodoMutation.mutate(newTodoText)`**: In our event handler, we call the `mutate` function provided by the hook, passing the variables our `addTodo` function needs.
5.  **Mutation State**: The `useMutation` hook also gives us loading (`isPending`) and error states for the mutation itself, allowing us to disable the form while the request is in flight and show specific error messages.

### Verification: Automatic UI Updates

Let's run the failure scenario again.

**Browser Behavior**:
The user types "Deploy the app" and clicks "Add Todo". The button changes to "Adding..." and the form is disabled. After a 1-second delay, the button reverts, the input clears, and the new todo appears at the bottom of the list.

**Network Tab Analysis**:
1.  A `POST` request (from `addTodo`) is initiated.
2.  The `POST` request completes successfully.
3.  Immediately after, a `GET` request (from the invalidated `useQuery`) is initiated to refetch the todos.
4.  The `GET` request completes, and the UI updates with the new data.

This is the "stale-while-revalidate" pattern, a robust way to keep client and server state in sync.

**Limitation Preview**:
This works perfectly, but the user experience could be better. The user has to wait for the network roundtrip (POST, then GET) before seeing their new todo. On a slow connection, this could be several seconds. Can we make the UI feel instantaneous? Yes, with optimistic updates.

## Optimistic Updates

## Creating a Snappy UI with Optimistic Updates

Our application is now robust, but it's not as fast as it could be. The user's action (adding a todo) and the UI's reaction (showing the new todo) are separated by two network requests. This delay, while technically correct, can make an application feel sluggish.

**Optimistic Updates** are a powerful UX pattern that flips this on its head: we assume the mutation will succeed and update the UI *immediately*, before the network request even finishes.

### The Optimistic Update Flow

1.  **Before Mutating**: The user clicks "Add".
2.  **Update UI Instantly**: We immediately add a temporary version of the new todo to our local query cache. The UI re-renders and the user sees their new todo appear instantly.
3.  **Fire Mutation**: In the background, we send the actual `POST` request to the server.
4.  **On Success**: The mutation succeeds. We then trigger a real refetch of the todo list to get the "canonical" data from the server (which will include the real ID, timestamps, etc.). The temporary todo is replaced by the real one, usually without the user noticing.
5.  **On Error**: The mutation fails. This is the critical part. We must "roll back" our optimistic change by restoring the query cache to its state before the update. We then show the user an error message.

This makes the application feel instantaneous, at the cost of some added complexity to handle the rollback case.

### Iteration 4: Implementing Optimistic Updates

**Current state recap**: Adding a todo correctly invalidates the query and refetches the list, but with a noticeable delay.
**Current limitation**: The UI feels slow because it waits for the server to confirm the change.
**New scenario introduction**: We want the new todo to appear in the list the moment the user clicks the "Add Todo" button.

Let's enhance our `useMutation` hook to perform an optimistic update.

```tsx
// src/components/TodoApp.tsx (Version 4 - Optimistic Update)

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { fetchTodos, addTodo, Todo } from '../api/todos';
import { FormEvent, useState } from 'react';

export function TodoApp() {
  const queryClient = useQueryClient();

  const { data: todos, isLoading, isError, error } = useQuery<Todo[]>({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const [newTodoText, setNewTodoText] = useState('');

  const addTodoMutation = useMutation({
    mutationFn: addTodo,
    // 1. onMutate is called before the mutation function is fired
    onMutate: async (newTodoText: string) => {
      // Cancel any outgoing refetches (so they don't overwrite our optimistic update)
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot the previous value
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // Optimistically update to the new value
      queryClient.setQueryData<Todo[]>(['todos'], (old) => [
        ...(old || []),
        { id: Date.now(), text: newTodoText, completed: false }, // Temporary todo
      ]);

      // Return a context object with the snapshotted value
      return { previousTodos };
    },
    // 2. onError is called if the mutation fails
    onError: (err, newTodo, context) => {
      // Rollback to the previous value
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    // 3. onSettled is always called after the mutation is done (success or error)
    onSettled: () => {
      // Invalidate the query to refetch the real data from the server
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const handleSubmit = (event: FormEvent) => {
    event.preventDefault();
    if (!newTodoText.trim()) return;
    addTodoMutation.mutate(newTodoText, {
      onSuccess: () => {
        setNewTodoText('');
      },
    });
  };

  // ... (render logic is the same) ...
  return (
    <div>
      <h1>Todo List (Optimistic)</h1>
      {/* ... form and list ... */}
    </div>
  );
}
```

### Analysis of the New `useMutation` Logic

This is significantly more complex, so let's break it down step-by-step.

*   **`onMutate`**: This function runs *before* `mutationFn`. It's our chance to manipulate the cache.
    *   `queryClient.cancelQueries`: This is a crucial first step. It prevents any background refetches from happening while we're performing our optimistic update, which could wipe out our change.
    *   `queryClient.getQueryData`: We get the current data from the cache so we can restore it later if something goes wrong. This is our "undo" snapshot.
    *   `queryClient.setQueryData`: This is the core of the optimistic update. We are **synchronously and manually** writing to the cache. We append a temporary todo to the existing list. Note we're using `Date.now()` for a temporary ID; the server will assign the real one.
    *   `return { previousTodos }`: We return the snapshot, which will be passed as a `context` object to `onError` and `onSettled`.

*   **`onError`**: This function runs only if `mutationFn` throws an error.
    *   `queryClient.setQueryData(['todos'], context.previousTodos)`: We use the `context` object from `onMutate` to roll back the cache to its original state, effectively making the temporary todo disappear.

*   **`onSettled`**: This function runs after the mutation is complete, regardless of whether it succeeded or failed.
    *   `queryClient.invalidateQueries`: We *always* invalidate the query at the end.
        *   If it succeeded, this fetches the real data from the server, replacing our temporary todo with the canonical one.
        *   If it failed, this ensures our UI is still in sync with the server after the rollback.

### Verification: Instant UI and Graceful Failure

Let's test both the success and failure cases.

**Success Case**:
1.  Type "Ship it!" and click "Add Todo".
2.  **Instantly**, the new todo appears in the list with a temporary ID. The UI feels incredibly fast.
3.  In the Network tab, the `POST` request is sent.
4.  After 1 second, the `POST` succeeds, and a `GET` request is triggered by `onSettled`.
5.  The `GET` request completes, and the list re-renders with the final data from the server (the temporary todo is replaced by the real one). The user likely won't even notice this swap.

**Failure Case** (our mock API fails 10% of the time):
1.  Type "This will fail" and click "Add Todo".
2.  **Instantly**, the new todo appears in the list.
3.  In the Network tab, the `POST` request is sent.
4.  After 1 second, the `POST` fails (our mock API throws an error).
5.  The `onError` callback is triggered. The `['todos']` cache is rolled back to its previous state.
6.  The UI re-renders, and the temporary todo **disappears** from the list. An error message is shown.
7.  The `onSettled` callback still runs, triggering a final refetch to ensure consistency.

This pattern provides the best of both worlds: a lightning-fast UI for the user's "happy path" and a resilient, self-healing state for the "unhappy path".

## Replacing your Redux boilerplate

## Rethinking Global State

For many years, Redux (and similar libraries like Zustand or MobX) has been the default choice for managing "global" state in React applications. A primary reason for reaching for these tools was to cache server data. We wanted to fetch a list of products once and make it available to any component without re-fetching every time the component mounted.

We would create actions, reducers, selectors, and thunks just to manage the simple lifecycle of fetching, caching, and updating this server data.

Let's look at the Redux boilerplate for managing just our `todos` state.

```typescript
// A rough example of Redux boilerplate for fetching todos

// todosSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchTodos as apiFetchTodos } from '../api/todos';

export const fetchTodos = createAsyncThunk('todos/fetchTodos', async () => {
  const response = await apiFetchTodos();
  return response;
});

const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [],
    status: 'idle', // 'idle' | 'loading' | 'succeeded' | 'failed'
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message;
      });
  },
});

export default todosSlice.reducer;

// In the component:
// const dispatch = useDispatch();
// const { items, status, error } = useSelector(state => state.todos);
// useEffect(() => {
//   if (status === 'idle') {
//     dispatch(fetchTodos());
//   }
// }, [status, dispatch]);
```

This is a lot of code to solve a problem that `useQuery({ queryKey: ['todos'], queryFn: fetchTodos })` solves in a single line. And this Redux example *still* doesn't handle:

*   Refetching on window focus.
*   Invalidating the data after a mutation.
*   Request deduplication.
*   Optimistic updates.

Adding those features would require significantly more boilerplate.

### React Query as Your Server State Cache

React Query isn't just a data fetching library; it's a powerful, asynchronous state manager. The `QueryClient` you provide at the root of your app acts as a global, in-memory cache for all of your server state.

*   **Instead of a Redux store for server data...** you have the `QueryClient`.
*   **Instead of reducers and thunks...** you have `queryFn` and `mutationFn`.
*   **Instead of selectors...** you have `useQuery` with a specific `queryKey`.
*   **Instead of dispatching actions...** you call `mutate` or `invalidateQueries`.

By adopting React Query, you can often delete vast amounts of Redux boilerplate from your application.

### The New Division of Labor

This doesn't mean Redux or Zustand are obsolete. It means their role becomes clearer and more focused. A modern React application's state can be cleanly divided:

| State Type    | Characteristics                               | Best Tool                               |
|---------------|-----------------------------------------------|-----------------------------------------|
| **Server State**  | Asynchronous, shared, can be stale, cached    | **TanStack (React) Query**              |
| **Client State**  | Synchronous, owned by client, ephemeral       | **`useState`, `useReducer`, `useContext`** |
| **Global Client State** | Client state shared by many components | **Zustand, Redux, Jotai**                 |

You may find that after moving all your server state management to React Query, the remaining global *client* state is so simple that you no longer need a heavy library like Redux. Perhaps a simple `useContext` and `useReducer` combination is enough. Or maybe a lightweight library like Zustand is a better fit for managing things like theme state, authentication status, or complex form state that spans multiple pages.

### The Journey: From Problem to Solution

Let's review the evolution of our `TodoApp` component.

| Iteration | Failure Mode                               | Technique Applied             | Result                                      | Code Complexity |
|-----------|--------------------------------------------|-------------------------------|---------------------------------------------|-----------------|
| 0         | Data becomes stale, no loading/error states| `useState` + `useEffect`      | Fragile, inefficient initial state          | Medium          |
| 1         | Data still stale after mutation            | `useQuery`                    | Robust data fetching and caching            | Low             |
| 2         | UI lags behind server state after mutation | `useMutation` + Invalidation  | UI automatically syncs with server          | Medium          |
| 3         | Perceptible UI lag on slow networks        | Optimistic Updates            | UI feels instantaneous, with safe rollbacks | High            |

We started with a seemingly simple `useEffect` hook and discovered it was hiding a mountain of complexity. By progressively adopting the features of React Query, we built a component that is not only simpler at the end but also vastly more powerful and resilient.

### Final Implementation

Here is the final, production-ready version of our component, a testament to how declarative server state management can be.

```tsx
// src/components/TodoApp.tsx (Final Version)

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { fetchTodos, addTodo, Todo } from '../api/todos';
import { FormEvent, useState } from 'react';

export function TodoApp() {
  const queryClient = useQueryClient();
  const [newTodoText, setNewTodoText] = useState('');

  const { data: todos, isLoading, isError, error } = useQuery<Todo[]>({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const addTodoMutation = useMutation({
    mutationFn: addTodo,
    onMutate: async (newTodoText: string) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);
      queryClient.setQueryData<Todo[]>(['todos'], (old) => [
        ...(old || []),
        { id: Date.now(), text: newTodoText, completed: false },
      ]);
      return { previousTodos };
    },
    onError: (err, newTodo, context) => {
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const handleSubmit = (event: FormEvent) => {
    event.preventDefault();
    if (!newTodoText.trim()) return;
    addTodoMutation.mutate(newTodoText, {
      onSuccess: () => setNewTodoText(''),
    });
  };

  if (isLoading) return <div>Loading todos...</div>;
  if (isError) return <div>An error occurred: {error.message}</div>;

  return (
    <div>
      <h1>Todo List (Optimistic)</h1>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={newTodoText}
          onChange={(e) => setNewTodoText(e.target.value)}
          placeholder="Add a new todo"
          disabled={addTodoMutation.isPending}
        />
        <button type="submit" disabled={addTodoMutation.isPending}>
          {addTodoMutation.isPending ? 'Adding...' : 'Add Todo'}
        </button>
      </form>
      {addTodoMutation.isError && (
        <div style={{ color: 'red' }}>
          Failed to add todo. Please try again.
        </div>
      )}
      <ul>
        {todos?.map((todo) => (
          <li key={todo.id} style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Lessons Learned

1.  **Separate Your State**: The single most important step is to identify what is client state and what is server state.
2.  **Use the Right Tool for the Job**: Don't manage server state with client state tools. `useState` is for modals; `useQuery` is for user data.
3.  **Embrace Declarative Data Fetching**: Describe *what* data you need with `useQuery`, not *how* to get it with `useEffect`.
4.  **Invalidation over Manual Updates**: Instead of manually pushing new data into your cache after a mutation, prefer to invalidate the relevant queries and let React Query handle the refetching. This is more resilient.
5.  **Use Optimistic Updates for a Premium Feel**: For frequent and important user actions, optimistic updates can transform the user experience from "good" to "great".
