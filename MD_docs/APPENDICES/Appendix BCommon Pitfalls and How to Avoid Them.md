# Chapter Appendix B: Common Pitfalls and How to Avoid Them

## Introduction: Learning from Failure

## Introduction: Learning from Failure

In software development, and especially in a complex ecosystem like React/Next.js, errors are not just obstacles; they are data. A well-understood error message or a piece of unexpected behavior is a map that can lead you to a deeper understanding of the system.

This appendix is a catalog of the most common pitfalls encountered by developers. We won't just show you the solution. Instead, for each pitfall, we will:

1.  **Create the problem**: Write code that works, but has a subtle, hidden flaw.
2.  **Trigger the failure**: Demonstrate a scenario where the flaw causes a crash, a bug, or a performance issue.
3.  **Diagnose the evidence**: Systematically analyze the browser console, React DevTools, and other signals to understand the root cause.
4.  **Implement the fix**: Apply the correct pattern or API to solve the problem.
5.  **Generalize the lesson**: Extract the underlying principle so you can avoid similar issues in the future.

By learning to recognize the *symptoms* of these common problems, you'll be able to debug your own applications faster and write more robust code from the start.

## Pitfall 1: The useEffect Dependency Trap

## Pitfall 1: The `useEffect` Dependency Trap

The `useEffect` hook is powerful, but its dependency array is the source of two of the most common bugs in React: stale data from missing dependencies, and infinite loops from unstable dependencies.

### Iteration 1: The Stale Data Bug

Let's build a component that fetches and displays user data based on a `userId` prop.

**Phase 1: Establish the Reference Implementation**

Our anchor example is a `UserProfile` component inside a `Dashboard` page. The `Dashboard` has buttons to switch between different users.

**Project Structure**:
```
src/
└── app/
    ├── page.tsx            (Our Dashboard)
    └── UserProfile.tsx     (Our component with the flaw)
```

Here is the initial, flawed implementation.

```tsx
// src/app/UserProfile.tsx

"use client";

import { useState, useEffect } from 'react';

// A mock API function
const fetchUserData = async (userId: string) => {
  console.log(`Fetching data for user ${userId}...`);
  // Simulate network delay
  await new Promise(resolve => setTimeout(resolve, 500));
  if (userId === '1') {
    return { id: '1', name: 'Alice', email: 'alice@example.com' };
  }
  if (userId === '2') {
    return { id: '2', name: 'Bob', email: 'bob@example.com' };
  }
  return { id: userId, name: 'Unknown User', email: 'unknown@example.com' };
};

interface User {
  id: string;
  name: string;
  email: string;
}

export default function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetchUserData(userId)
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, []); // <-- THE FLAW: Empty dependency array

  if (loading) {
    return <p>Loading profile...</p>;
  }

  if (!user) {
    return <p>No user data.</p>;
  }

  return (
    <div>
      <h2>User Profile</h2>
      <p><strong>ID:</strong> {user.id}</p>
      <p><strong>Name:</strong> {user.name}</p>
      <p><strong>Email:</strong> {user.email}</p>
    </div>
  );
}
```

And here is the parent `Dashboard` component that lets us switch the `userId` prop.

```tsx
// src/app/page.tsx

"use client";

import { useState } from 'react';
import UserProfile from './UserProfile';

export default function Dashboard() {
  const [currentUserId, setCurrentUserId] = useState('1');

  return (
    <main style={{ padding: '2rem' }}>
      <h1>Dashboard</h1>
      <nav>
        <button onClick={() => setCurrentUserId('1')}>View Alice (ID 1)</button>
        <button onClick={() => setCurrentUserId('2')}>View Bob (ID 2)</button>
      </nav>
      <hr />
      <UserProfile userId={currentUserId} />
    </main>
  );
}
```

**Phase 2: Progressive Problem Exposure**

The component loads Alice's data correctly on the initial render. But what happens when we click the "View Bob" button?

**Failure Demonstration**

1.  The page loads, showing "Loading profile..." then Alice's data.
2.  Click the "View Bob (ID 2)" button.
3.  The UI briefly shows "Loading profile..." again, but then displays **Alice's data**, not Bob's.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The UI shows stale data. After clicking to view Bob, the loading state flickers, but the profile for Alice is displayed again.

**Browser Console Output**:
```
Fetching data for user 1...

Warning: React Hook useEffect has a missing dependency: 'userId'. Either include it or remove the dependency array.
    at UserProfile (./src/app/UserProfile.tsx:25:3)
    ...
```

**React DevTools Evidence**:
-   **Components Tab (after clicking "View Bob")**:
    -   `Dashboard` component: State `currentUserId` is `"2"`.
    -   `UserProfile` component: Props `userId` is `"2"`.
    -   `UserProfile` component: State `user` is `{id: '1', name: 'Alice', ...}`. **This is the mismatch!** The prop is correct, but the state is stale.

**Let's parse this evidence**:

1.  **What the user experiences**: The profile doesn't update to the selected user.
    -   Expected: See Bob's profile after clicking the button.
    -   Actual: See Alice's profile again.

2.  **What the console reveals**: React is explicitly telling us what's wrong. The warning `React Hook useEffect has a missing dependency: 'userId'` is the smoking gun. It means our effect reads the `userId` prop but has not "subscribed" to its changes.

3.  **What DevTools shows**: We can confirm the diagnosis. The `UserProfile` component *is* receiving the new prop (`userId="2"`), but its internal state (`user`) is not being updated correctly because the `useEffect` that fetches data isn't running again.

4.  **Root cause identified**: The `useEffect` has an empty dependency array (`[]`), which tells React: "Only run this effect once, after the initial render, and never again."

5.  **Why the current approach can't solve this**: An empty dependency array is for one-time setup code. Our effect is not setup code; it needs to re-synchronize with the outside world (the `userId` prop) whenever it changes.

6.  **What we need**: A way to tell React to re-run the effect whenever the `userId` prop changes.

**Solution Implementation**

The fix is to follow the advice from the console warning and add `userId` to the dependency array.

**Before**:
```typescript
// src/app/UserProfile.tsx

  useEffect(() => {
    setLoading(true);
    fetchUserData(userId)
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, []); // <-- PROBLEM
```

**After**:
```typescript
// src/app/UserProfile.tsx

  useEffect(() => {
    setLoading(true);
    fetchUserData(userId)
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]); // <-- FIX
```

**Verification**

Now, when you click "View Bob (ID 2)", the console logs `Fetching data for user 2...` and Bob's profile correctly appears. The component now correctly synchronizes with its props.

---

### Iteration 2: The Infinite Loop Bug

Now let's explore the opposite problem. What happens when a dependency changes on *every single render*?

Let's modify our component to accept a `config` object prop containing fetch options.

**Phase 1: Establish the Reference Implementation**

```tsx
// src/app/UserProfileWithConfig.tsx

"use client";

import { useState, useEffect } from 'react';

// ... (fetchUserData and User interface are the same)

interface User { id: string; name: string; email: string; }
const fetchUserData = async (userId: string, options: { priority: string }) => {
  console.log(`Fetching data for user ${userId} with priority ${options.priority}`);
  await new Promise(resolve => setTimeout(resolve, 500));
  // ... (same user data logic)
  return { id: userId, name: 'User ' + userId, email: 'user@example.com' };
};

export default function UserProfileWithConfig({ userId, options }: { userId: string, options: { priority: string } }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetchUserData(userId, options)
      .then(data => {
        setUser(data);
      });
  }, [userId, options]); // <-- THE FLAW: 'options' is an object

  if (!user) return <p>Loading...</p>;

  return (
    <div>
      <h2>User Profile</h2>
      <p><strong>Name:</strong> {user.name}</p>
    </div>
  );
}
```

And the parent component that passes the `options` object.

```tsx
// src/app/page.tsx (modified)

"use client";

import { useState } from 'react';
import UserProfileWithConfig from './UserProfileWithConfig';

export default function Dashboard() {
  const [currentUserId, setCurrentUserId] = useState('1');

  // This component re-renders when its state changes.
  // On every render, a NEW options object is created.
  const fetchOptions = { priority: 'high' };

  return (
    <main style={{ padding: '2rem' }}>
      <h1>Dashboard</h1>
      <button onClick={() => setCurrentUserId(currentUserId === '1' ? '2' : '1')}>
        Switch User
      </button>
      <hr />
      <UserProfileWithConfig userId={currentUserId} options={fetchOptions} />
    </main>
  );
}
```

**Phase 2: Progressive Problem Exposure**

Run the app. What do you see?

**Failure Demonstration**

The component gets stuck in a loop. The "Loading..." message might flicker, and if you open the browser's developer tools, you'll see a storm of activity.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The UI is stuck. The browser tab may become slow or unresponsive.

**Browser Console Output**:
```
Fetching data for user 1 with priority high
Fetching data for user 1 with priority high
Fetching data for user 1 with priority high
Fetching data for user 1 with priority high
... (repeats indefinitely)
```

**Network Tab Analysis**:
-   If `fetchUserData` made a real network request, you would see hundreds of identical requests being sent to the server every few seconds.

**React DevTools Evidence**:
-   **Profiler**: If you record a profiling session, you'll see the `UserProfileWithConfig` component re-rendering continuously and rapidly. The reason for the update will be that its props changed.

**Let's parse this evidence**:

1.  **What the user experiences**: A broken, looping component.
2.  **What the console reveals**: The `fetchUserData` function is being called over and over again, without any user interaction.
3.  **What DevTools shows**: The component is trapped in a render loop.
4.  **Root cause identified**: The `options` prop is an object. In JavaScript, `{ priority: 'high' } !== { priority: 'high' }`. They are two separate objects in memory. The `Dashboard` component creates a *new* `fetchOptions` object on every single render. This new object is passed as a prop to `UserProfileWithConfig`. Inside `UserProfileWithConfig`, the `useEffect` hook compares the `options` dependency from the previous render to the current one. Since the object reference is always new, React thinks the dependency has changed and re-runs the effect. The effect calls `setUser`, which triggers a re-render, and the cycle begins again.

5.  **Why the current approach can't solve this**: We are passing an unstable dependency (a newly created object) to `useEffect`, guaranteeing it will run on every render.

6.  **What we need**: A way to provide a *stable* reference for the `options` object so that it only changes when its contents actually change.

**Solution Implementation**

We can solve this in the parent component by memoizing the `options` object with the `useMemo` hook. This ensures that the same object reference is returned as long as its own dependencies (`[]` in this case) don't change.

**Before** (Parent Component):
```tsx
// src/app/page.tsx

// ...
export default function Dashboard() {
  const [currentUserId, setCurrentUserId] = useState('1');

  // PROBLEM: New object on every render
  const fetchOptions = { priority: 'high' };

  return (
    // ...
    <UserProfileWithConfig userId={currentUserId} options={fetchOptions} />
  );
}
```

**After** (Parent Component):
```tsx
// src/app/page.tsx

import { useState, useMemo } from 'react'; // <-- Import useMemo
import UserProfileWithConfig from './UserProfileWithConfig';

export default function Dashboard() {
  const [currentUserId, setCurrentUserId] = useState('1');

  // FIX: Memoize the object so its reference is stable
  const fetchOptions = useMemo(() => ({ priority: 'high' }), []);

  return (
    // ...
    <UserProfileWithConfig userId={currentUserId} options={fetchOptions} />
  );
}
```

**Verification**

After this change, the component loads once and stops. The infinite loop is broken. The `fetchOptions` object now has a stable identity across re-renders of `Dashboard`, so `useEffect` in the child component no longer re-runs unnecessarily.

### Common Failure Modes and Their Signatures

#### Symptom: Data doesn't update when props change.

**Browser behavior**: UI shows stale information.
**Console pattern**: `Warning: React Hook useEffect has a missing dependency...`
**DevTools clues**: The component's props have updated, but its state has not.
**Root cause**: A variable used inside `useEffect` is missing from the dependency array.
**Solution**: Add the missing variable to the dependency array. Use the `exhaustive-deps` lint rule to catch this automatically.

#### Symptom: Infinite re-render loop.

**Browser behavior**: Page is stuck, flickering, or unresponsive.
**Console pattern**: A `console.log` inside the component or effect fires continuously.
**Network Tab clues**: A flood of identical network requests.
**Root cause**: A dependency in a `useEffect` array is an object, array, or function that is re-created on every render.
**Solution**: Memoize the unstable dependency in the parent component using `useMemo` (for objects/arrays) or `useCallback` (for functions).

## Pitfall 2: Direct State Mutation

## Pitfall 2: Direct State Mutation

React's re-rendering mechanism is triggered when it detects a change in state or props. For objects and arrays, this detection relies on a change in the object's *reference* (its identity in memory), not its contents. Mutating state directly breaks this mechanism, leading to a UI that doesn't update.

### Phase 1: Establish the Reference Implementation

Let's build a simple to-do list. The user can add items to a list stored in state. Our flawed version will use `Array.prototype.push` to add a new item.

```tsx
// src/app/TodoList.tsx

"use client";

import { useState } from 'react';

interface Todo {
  id: number;
  text: string;
}

export default function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Learn React' }
  ]);
  const [newTodoText, setNewTodoText] = useState('');

  const handleAddTodo = () => {
    const newId = todos.length > 0 ? Math.max(...todos.map(t => t.id)) + 1 : 1;
    const newTodo = { id: newId, text: newTodoText };

    // THE FLAW: Mutating the state array directly
    todos.push(newTodo);
    setTodos(todos); // We are setting state with the SAME array reference

    setNewTodoText('');
  };

  return (
    <div>
      <h2>Todo List</h2>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
      <input
        type="text"
        value={newTodoText}
        onChange={(e) => setNewTodoText(e.target.value)}
        placeholder="New todo..."
      />
      <button onClick={handleAddTodo}>Add Todo</button>
    </div>
  );
}
```

### Phase 2: Progressive Problem Exposure

Type something into the input field and click "Add Todo".

**Failure Demonstration**

Absolutely nothing happens in the UI. The new to-do item does not appear in the list.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The list on the screen does not change after clicking the "Add" button.

**Browser Console Output**:
There are no errors or warnings. The silence is what makes this bug so confusing for beginners.

**React DevTools Evidence**:
-   **Components Tab**:
    1.  Select the `TodoList` component.
    2.  Look at the `todos` state hook. Initially, it's an array with one item.
    3.  Click "Add Todo" in your app.
    4.  **Crucially, the state value shown in DevTools *updates*!** You will see the new to-do item in the array.
    5.  However, the component has not re-rendered. The profiler would show zero renders for this action.

**Let's parse this evidence**:

1.  **What the user experiences**: A button that doesn't work.
2.  **What the console reveals**: Nothing. This is a logic error, not a syntax error.
3.  **What DevTools shows**: This is the key. The state *data* has changed, but React didn't *detect* the change. This tells us the problem is with how we're signaling the update to React.
4.  **Root cause identified**: We are directly mutating the `todos` array with `todos.push(newTodo)`. This modifies the array in place. When we call `setTodos(todos)`, we are passing the *exact same array reference* that React already has. React performs a quick reference check (`Object.is(oldState, newState)`), sees that the reference hasn't changed, and concludes that no update is necessary, so it bails out of the re-render.
5.  **Why the current approach can't solve this**: Mutation is fundamentally incompatible with React's state update detection for objects and arrays.
6.  **What we need**: To create a *new* array containing all the old items plus the new one, and pass this new array to our state setter function.

**Solution Implementation**

We must replace the mutation with an immutable update. The spread syntax is perfect for this.

**Before**:
```typescript
// src/app/TodoList.tsx

  const handleAddTodo = () => {
    // ...
    // PROBLEM: Mutation
    todos.push(newTodo);
    setTodos(todos);
    // ...
  };
```

**After**:
```typescript
// src/app/TodoList.tsx

  const handleAddTodo = () => {
    const newId = todos.length > 0 ? Math.max(...todos.map(t => t.id)) + 1 : 1;
    const newTodo = { id: newId, text: newTodoText };

    // FIX: Create a new array
    const newTodos = [...todos, newTodo];
    setTodos(newTodos);

    setNewTodoText('');
  };
```

**Verification**

Now, when you add a new to-do, it instantly appears in the list. By creating a new array, we give `setTodos` a new reference. React detects the change and correctly re-renders the component.

### General Principle: Treat State as Immutable

-   **For Arrays**: Never use mutating methods like `push`, `pop`, `splice`, `sort`, or `reverse` on state arrays. Instead, use non-mutating alternatives like `concat`, `slice`, `filter`, `map`, or the spread syntax (`[...]`).
-   **For Objects**: Never modify properties directly (e.g., `state.user.name = 'new'`). Instead, create a new object using the spread syntax (`{...state, user: {...state.user, name: 'new'}}`).

## Pitfall 3: Misunderstanding Server vs. Client Components

## Pitfall 3: Misunderstanding Server vs. Client Components

Next.js 13+ with the App Router introduces a powerful but initially confusing distinction: Server Components and Client Components. The most common pitfall is trying to use client-side features (like hooks) in a Server Component.

### Iteration 1: Using Hooks on the Server

By default, all components inside the `app` directory are **Server Components**. They run on the server and are great for fetching data and reducing the client-side JavaScript bundle. However, they cannot use state, effects, or browser-only APIs.

**Phase 1: Establish the Reference Implementation**

A developer new to the App Router wants to add a simple counter to their homepage.

```tsx
// src/app/page.tsx (Flawed Version)

import { useState } from 'react'; // <-- This is the problem

export default function HomePage() {
  const [count, setCount] = useState(0); // <-- Hooks are client-side only

  return (
    <main>
      <h1>Welcome to my page!</h1>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </main>
  );
}
```

### Phase 2: Progressive Problem Exposure

As soon as you save this file, your Next.js development server will show an error.

**Failure Demonstration**

The application fails to render, and Next.js displays a helpful error overlay in the browser.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
A full-screen error overlay from Next.js.

**Terminal Output**:
```bash
./src/app/page.tsx
ReactServerComponentsError:

You're importing a component that needs useState. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.

Learn more: https://nextjs.org/docs/getting-started/react-essentials

   ,-[/Users/dev/project/src/app/page.tsx:1:1]
 1 | import { useState } from 'react';
   :          ^^^^^^^^^
 2 |
 3 | export default function HomePage() {
 4 |   const [count, setCount] = useState(0);
   `----

Maybe one of these should be marked as a client entry with "use client":
  ./src/app/page.tsx
```

**Let's parse this evidence**:

1.  **What the user experiences**: A clear, immediate error.
2.  **What the console reveals**: The error message is incredibly specific. It identifies the exact hook (`useState`), explains that it only works in a Client Component, and even suggests the solution: marking the file with `"use client"`.
3.  **Root cause identified**: We are attempting to use a client-side hook (`useState`) inside a Server Component. Server Components are stateless and non-interactive.
4.  **What we need**: A way to tell Next.js that this component (or at least part of it) needs to be interactive and run in the browser.

**Solution Implementation**

The fix is exactly what the error message suggests: add the `"use client"` directive to the top of the file. This marks the component and all components imported into it as part of the client-side JavaScript bundle.

**Before**:
```tsx
// src/app/page.tsx

import { useState } from 'react';

export default function HomePage() {
// ...
```

**After**:
```tsx
// src/app/page.tsx

"use client"; // <-- FIX: Mark this as a Client Component

import { useState } from 'react';

export default function HomePage() {
// ...
```

**Verification**

The page now loads and the counter works as expected.

---

### Iteration 2: The "Client Component Island" Pattern

The first solution works, but it has a performance cost. By making the entire `HomePage` a Client Component, we've sent all of its code and its dependencies to the browser, even the static parts like the `<h1>`.

A better approach is to keep as much of the page as a Server Component as possible and isolate the interactive parts into their own small Client Components. This is often called the "Client Component Island" pattern.

**Problematic Approach**: Making the whole page a client component.

```tsx
// src/app/page.tsx

"use client"; // <-- Makes the entire page a client component

import { useState } from 'react';
import Header from '../components/Header'; // Header is now also client-side
import Footer from '../components/Footer'; // Footer is now also client-side
import SomeLargeStaticComponent from '../components/SomeLargeStaticComponent';

export default function HomePage() {
  const [count, setCount] = useState(0);

  return (
    <main>
      <Header />
      <SomeLargeStaticComponent />
      <h1>Welcome to my page!</h1>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <Footer />
    </main>
  );
}
```
The issue here is that `Header`, `Footer`, and `SomeLargeStaticComponent` might be completely static, but because they are imported into a Client Component, they become part of the client bundle.

**Solution: Isolate Interactivity**

1.  Create a new, small component just for the counter.
2.  Mark *only that component* as `"use client"`.
3.  Import this new Client Component into your main Server Component page.

**Step 1: Create the Client Component**
```tsx
// src/components/CounterButton.tsx

"use client";

import { useState } from 'react';

export default function CounterButton() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

**Step 2: Use it in the Server Component Page**
```tsx
// src/app/page.tsx (Now a Server Component again)

import Header from '../components/Header';
import Footer from '../components/Footer';
import SomeLargeStaticComponent from '../components/SomeLargeStaticComponent';
import CounterButton from '../components/CounterButton'; // <-- Import the client island

export default function HomePage() {
  // This component is now a Server Component. No hooks allowed here.
  // It can do server-only things like directly accessing a database.

  return (
    <main>
      <Header />
      <SomeLargeStaticComponent />
      <h1>Welcome to my page!</h1>
      <CounterButton /> {/* <-- Render the interactive island */}
      <Footer />
    </main>
  );
}
```

**Improvement**:
-   **Performance**: The initial page load is faster because only the JavaScript for `CounterButton` is sent to the client. `HomePage`, `Header`, `Footer`, and `SomeLargeStaticComponent` are rendered to static HTML on the server.
-   **Architecture**: This encourages a clean separation of concerns between server-side data fetching/rendering and client-side interactivity.

**When to Apply This Solution**:
-   **What it optimizes for**: Minimal client-side JavaScript, faster initial page loads (Time to Interactive).
-   **What it sacrifices**: A small amount of architectural complexity (creating more, smaller components).
-   **When to choose this approach**: Almost always. This is the idiomatic way to build apps with the Next.js App Router. Start with Server Components and introduce Client Components only when you need interactivity.
-   **When to avoid this approach**: If a page is almost entirely interactive (like a complex dashboard or a web app like Figma), it might be simpler to make the whole page a Client Component.

## Synthesis: A Pitfall Decision Guide

## Synthesis: A Pitfall Decision Guide

This table summarizes the common pitfalls, their tell-tale symptoms, and the tools you can use to diagnose them. Use this as a quick reference when you encounter strange behavior in your application.

| Symptom in Browser                               | Potential Cause                               | Key Diagnostic Tool(s)                               | Solution                                                              |
| ------------------------------------------------ | --------------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------- |
| UI doesn't update after a prop changes.          | Missing `useEffect` dependency.               | React DevTools (props updated, state stale), Console Warning | Add the missing prop/variable to the `useEffect` dependency array.    |
| Page freezes, flickers, or makes endless API calls. | Unstable dependency in `useEffect` (object/fn). | Network Tab, React Profiler, Console Logs            | Memoize the dependency with `useMemo` or `useCallback` in the parent. |
| UI doesn't update after clicking a button to add/change data. | Direct state mutation (`.push()`, `obj.prop=`). | React DevTools (state value changes, but no re-render) | Use immutable patterns (spread syntax `...`, `.map`, `.filter`).      |
| Next.js error overlay: "useState only works in a Client Component". | Using hooks or event handlers in a Server Component. | Next.js Error Overlay, Terminal Output               | Add `"use client"` to the top of the file.                            |
| Slow initial load, large JS bundle for a mostly static page. | Over-using `"use client"` on large parent components. | Bundle Analyzer (`@next/bundle-analyzer`), Network Tab | Isolate interactivity into small "Client Component Islands".          |
| Many unrelated components re-render when one piece of state changes. | Prop drilling or an unstable Context value. | React Profiler (highlight updates)                   | Use component composition or stabilize Context value with `useMemo`.  |

By internalizing these patterns of failure, you move from guessing to systematic debugging. Every bug becomes a lesson that deepens your understanding of the React and Next.js mental model.
