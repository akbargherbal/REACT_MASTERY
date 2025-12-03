# Chapter 4: Side Effects and Data Fetching

## useEffect: the escape hatch

## The Problem: React's Pure Component World

In Chapter 3, we built interactive components with `useState` and event handlers. Our `UserDashboard` could update its own state when users clicked buttons. But we lived in a self-contained world—our components only responded to user actions.

Real applications need to interact with the outside world:
- Fetch data from APIs
- Subscribe to WebSocket connections
- Update the document title
- Set up timers
- Interact with browser APIs (localStorage, geolocation)
- Integrate with third-party libraries (analytics, chat widgets)

These operations are called **side effects**—they reach outside the pure, predictable world of React's rendering system.

### Why Side Effects Need Special Handling

React components are functions that return JSX. React calls these functions to figure out what to display. Here's the critical insight: **React may call your component function multiple times for a single render**.

During development, React intentionally calls components twice to help you find bugs. During rendering, React may call your component, then discard the result and call it again. This is normal and expected.

This creates a problem. What happens if you put a side effect directly in your component body?

```tsx
// ❌ WRONG: Side effect in component body
// File: src/components/UserDashboard.tsx

function UserDashboard() {
  const [user, setUser] = useState(null);
  
  // This runs every time React calls this function
  fetch('/api/user')
    .then(res => res.json())
    .then(data => setUser(data));
  
  return <div>{user?.name}</div>;
}
```

### The Failure: Infinite Render Loop

Let's run this code and observe what happens.

**Browser Behavior**:
The page loads, shows a blank screen briefly, then the browser tab becomes unresponsive. The fan on your laptop spins up. After a few seconds, the browser may display "Page Unresponsive" or crash entirely.

**Browser Console Output**:

```text
Warning: Maximum update depth exceeded. This can happen when a component 
calls setState inside useEffect, but useEffect either doesn't have a 
dependency array, or one of the dependencies changes on every render.

[Violation] 'requestAnimationFrame' handler took 1847ms
[Violation] Forced reflow while executing JavaScript took 234ms
```

**React DevTools - Profiler Tab**:
- Recorded render: `UserDashboard` rendered 847 times in 2.1 seconds
- Each render took ~2ms
- Reason: State update triggered re-render
- Pattern: Continuous rendering, never stops

**Browser DevTools - Network Tab**:
- Filter: Fetch/XHR
- Observation: 200+ requests to `/api/user` in 2 seconds
- Each request: 200ms duration, 200 OK status
- Pattern: Requests fire continuously, never stop
- Total data transferred: 4.2 MB

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the user experiences**: 
   - Expected: Dashboard loads and displays user data
   - Actual: Browser freezes, becomes unresponsive

2. **What the console reveals**:
   - Key indicator: "Maximum update depth exceeded"
   - Error location: The warning mentions `setState` and rendering
   - Translation: Something is causing infinite re-renders

3. **What DevTools shows**:
   - Component state: `user` keeps changing from `null` to data object
   - Render behavior: Component renders continuously, 847 times in 2 seconds
   - Network activity: Hundreds of identical API requests

4. **Root cause identified**: 
   The `fetch` call runs every time the component renders. When the fetch completes, it calls `setUser`, which triggers a re-render. The re-render runs the fetch again. Infinite loop.

5. **Why the current approach can't solve this**:
   We can't just "be careful" about where we put the fetch. React's rendering model requires that component functions be pure—they should not have side effects. We need a way to tell React: "Run this side effect, but only at specific times."

6. **What we need**:
   A mechanism to run side effects **after** rendering completes, with control over **when** they run again.

## Enter useEffect: The Escape Hatch

`useEffect` is React's way of saying: "After you finish rendering and updating the DOM, run this code." It's an escape hatch from the pure component world into the world of side effects.

The signature:

```typescript
useEffect(
  () => {
    // Your side effect code here
    // Runs AFTER render commits to the DOM
  },
  [/* dependency array */]
);
```

Two parts:
1. **Effect function**: The code to run
2. **Dependency array**: Controls when the effect runs

### The Mental Model: Synchronization

Think of `useEffect` as **synchronizing** your component with an external system. You're saying: "Keep this side effect in sync with these values."

- When the component mounts → run the effect
- When dependencies change → run the effect again
- When the component unmounts → clean up the effect

Let's fix our infinite loop.

## Fetching data (the naïve way)

## Iteration 1: Basic Data Fetching

We'll start with the simplest possible data fetching pattern. Our goal: load user data when the component first mounts, and never again.

**Reference Implementation**: We're building a `UserDashboard` that displays user information fetched from an API. This will be our anchor example throughout this chapter.

**Project Structure**:

```text
src/
├── components/
│   └── UserDashboard.tsx  ← Our reference implementation
├── types/
│   └── user.ts
└── app/
    └── page.tsx
```

First, let's define our data types:

```typescript
// File: src/types/user.ts

export interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
  role: 'admin' | 'user' | 'guest';
}
```

Now, the component with `useEffect`:

```tsx
// File: src/components/UserDashboard.tsx

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(data => setUser(data));
  }, []); // ← Empty dependency array
  
  if (!user) {
    return <div>Loading...</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Iteration 0 - Broken):
```tsx
function UserDashboard() {
  const [user, setUser] = useState(null);
  
  // Runs on every render
  fetch('/api/user')
    .then(res => res.json())
    .then(data => setUser(data));
  
  return <div>{user?.name}</div>;
}
```

**After** (Iteration 1):
```tsx
function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    // Runs only after first render
    fetch('/api/user')
      .then(res => res.json())
      .then(data => setUser(data));
  }, []); // ← Empty array = run once
  
  if (!user) {
    return <div>Loading...</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}
```

**Key changes**:
1. Wrapped fetch in `useEffect`
2. Added empty dependency array `[]`
3. Added loading state check
4. Added TypeScript types

### Verification: Does It Work?

**Browser Behavior**:
- Page loads
- Shows "Loading..." for ~200ms
- User data appears
- No freezing, no crashes

**Browser Console Output**:

```text
(No errors or warnings)
```

**React DevTools - Profiler Tab**:
- `UserDashboard` rendered 2 times total
  - Render 1: Initial mount, `user` is `null`
  - Render 2: After fetch completes, `user` has data
- Total time: 203ms (mostly waiting for network)

**Browser DevTools - Network Tab**:
- 1 request to `/api/user`
- Status: 200 OK
- Time: 198ms
- No additional requests

**Expected vs. Actual**:
- ✅ Component renders twice (expected: mount + data update)
- ✅ One API request (expected: fetch on mount only)
- ✅ User sees loading state, then data
- ✅ No infinite loop

### Understanding the Empty Dependency Array

The `[]` is crucial. It tells React: "Run this effect after the first render, then never again."

Let's understand what happens with different dependency arrays:

```tsx
// Pattern 1: Empty array - run once on mount
useEffect(() => {
  console.log('Runs once after first render');
}, []);

// Pattern 2: No array - run after every render
useEffect(() => {
  console.log('Runs after every render');
}); // ← Dangerous! Usually wrong

// Pattern 3: With dependencies - run when dependencies change
useEffect(() => {
  console.log('Runs when userId changes');
}, [userId]);
```

**When to use each pattern**:

| Pattern | Use Case | Example |
|---------|----------|---------|
| `[]` | One-time setup | Initial data fetch, analytics page view |
| No array | Rarely correct | Syncing with external system that changes every render |
| `[deps]` | Sync with values | Fetch data when ID changes, update title when name changes |

### Current Limitation

Our component works, but it has problems:

1. **No error handling**: What if the API request fails?
2. **No loading state**: Users see "Loading..." but can't tell if it's stuck
3. **Race conditions**: What if the component unmounts before fetch completes?
4. **No refetching**: If data changes on the server, we never know

Let's address these one by one.

## Iteration 2: Adding Error Handling

**Current state recap**: Our component fetches data on mount and displays it. But we're living in a perfect world where APIs never fail.

**Current limitation**: If the fetch fails, the component stays in "Loading..." state forever. The user has no idea what went wrong.

**New scenario introduction**: What happens if the API returns a 500 error? Or the network is offline?

Let's intentionally break our API to see the failure:

```tsx
// File: src/components/UserDashboard.tsx (Iteration 1 - still broken)

function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    // Simulate API failure
    fetch('/api/user-that-does-not-exist')
      .then(res => res.json())
      .then(data => setUser(data));
  }, []);
  
  if (!user) {
    return <div>Loading...</div>;
  }
  
  return (
    <div>
      <h1>{user.name}</h1>
    </div>
  );
}
```

### The Failure: Silent Error

**Browser Behavior**:
- Page loads
- Shows "Loading..." forever
- No indication of what went wrong
- User is stuck

**Browser Console Output**:

```text
GET http://localhost:3000/api/user-that-does-not-exist 404 (Not Found)

Uncaught (in promise) SyntaxError: Unexpected token '<' in JSON at position 0
    at UserDashboard.tsx:8
```

**React DevTools - Components Tab**:
- `UserDashboard` component selected
- State: `{ user: null }`
- Hooks: `useState` (user), `useEffect` (no cleanup)
- Component never re-renders after initial mount

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Either see data or see an error message
   - Actual: Stuck on "Loading..." with no feedback

2. **What the console reveals**:
   - Key indicator: "404 (Not Found)" - the API endpoint doesn't exist
   - Second error: "Unexpected token '<'" - we tried to parse HTML as JSON
   - Error location: Inside the `.then(res => res.json())` chain

3. **What DevTools shows**:
   - Component state: `user` is still `null`
   - Render behavior: Component rendered once, never updated
   - No state change occurred

4. **Root cause identified**:
   The fetch promise chain has no error handling. When the API returns 404, the response is HTML (an error page), not JSON. Trying to parse HTML as JSON throws an error. The error is uncaught, so `setUser` never runs, and the component stays in loading state.

5. **Why the current approach can't solve this**:
   Promise chains without `.catch()` silently swallow errors. We need explicit error handling.

6. **What we need**:
   A way to capture errors and display them to the user.

### Solution: Error State

Add error state and proper error handling:

```tsx
// File: src/components/UserDashboard.tsx (Iteration 2)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/user')
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        setUser(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, []);
  
  if (isLoading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return (
      <div style={{ color: 'red' }}>
        <h2>Error loading user data</h2>
        <p>{error}</p>
      </div>
    );
  }
  
  if (!user) {
    return <div>No user data available</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Iteration 1):
- One state variable: `user`
- No error handling
- Loading state implicit (when `user` is `null`)

**After** (Iteration 2):
- Three state variables: `user`, `error`, `isLoading`
- Explicit error handling with `.catch()`
- Check `res.ok` before parsing JSON
- Three distinct UI states: loading, error, success

### Verification: Error Handling Works

Now let's test with the broken API endpoint:

**Browser Behavior**:
- Page loads
- Shows "Loading..." briefly
- Shows error message: "Error loading user data: HTTP error! status: 404"
- User knows something went wrong

**Browser Console Output**:

```text
GET http://localhost:3000/api/user-that-does-not-exist 404 (Not Found)
(No uncaught errors)
```

**React DevTools - Components Tab**:
- State: `{ user: null, error: "HTTP error! status: 404", isLoading: false }`
- Component rendered 2 times:
  - Render 1: Initial mount, `isLoading: true`
  - Render 2: After error caught, `isLoading: false`, `error` set

**Expected vs. Actual**:
- ✅ User sees error message instead of infinite loading
- ✅ Error is caught and handled gracefully
- ✅ Console shows network error but no uncaught exceptions

### The Three States Pattern

Every async operation has three states:

1. **Loading**: Operation in progress
2. **Success**: Operation completed successfully
3. **Error**: Operation failed

Your component should handle all three explicitly:

```tsx
// The pattern
const [data, setData] = useState(null);
const [error, setError] = useState(null);
const [isLoading, setIsLoading] = useState(true);

// Three distinct UI states
if (isLoading) return <LoadingSpinner />;
if (error) return <ErrorMessage error={error} />;
return <SuccessView data={data} />;
```

### Current Limitation

Our component now handles errors, but we still have problems:

1. **Race conditions**: What if the component unmounts while the fetch is in progress?
2. **Memory leaks**: Setting state on an unmounted component causes warnings
3. **No cancellation**: We can't cancel the fetch if we don't need it anymore

Let's see this failure in action.

## Iteration 3: The Race Condition Problem

**Current state recap**: Our component fetches data on mount and handles errors. It works well in isolation.

**Current limitation**: If the component unmounts before the fetch completes, we try to set state on an unmounted component.

**New scenario introduction**: What happens if the user navigates away before the data loads?

Let's create a scenario where this happens:

```tsx
// File: src/app/page.tsx
// Parent component that can unmount UserDashboard

import { useState } from 'react';
import UserDashboard from '../components/UserDashboard';

export default function Page() {
  const [showDashboard, setShowDashboard] = useState(true);
  
  return (
    <div>
      <button onClick={() => setShowDashboard(!showDashboard)}>
        Toggle Dashboard
      </button>
      
      {showDashboard && <UserDashboard />}
    </div>
  );
}
```

### The Failure: Memory Leak Warning

**User action**: 
1. Page loads, `UserDashboard` starts fetching
2. User clicks "Toggle Dashboard" before fetch completes
3. Component unmounts
4. Fetch completes and tries to call `setUser`

**Browser Console Output**:

```text
Warning: Can't perform a React state update on an unmounted component. 
This is a no-op, but it indicates a memory leak in your application. 
To fix, cancel all subscriptions and asynchronous tasks in a useEffect 
cleanup function.
    at UserDashboard (UserDashboard.tsx:5)
```

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Component unmounts cleanly
   - Actual: Works, but console shows warning

2. **What the console reveals**:
   - Key indicator: "Can't perform a React state update on an unmounted component"
   - Translation: The fetch completed after the component was removed from the DOM
   - Location: `UserDashboard` component

3. **What's happening under the hood**:
   - Component mounts → fetch starts
   - Component unmounts → removed from DOM
   - Fetch completes → tries to call `setUser` on a component that no longer exists
   - React prevents the state update but warns about the memory leak

4. **Root cause identified**:
   We have no way to cancel the fetch or prevent the state update when the component unmounts.

5. **Why the current approach can't solve this**:
   The fetch promise continues running even after the component unmounts. We need a way to either cancel the fetch or ignore its result.

6. **What we need**:
   A cleanup mechanism that runs when the component unmounts.

This is where `useEffect` cleanup functions come in.

## Cleanup and dependencies

## The Cleanup Function: Preventing Memory Leaks

`useEffect` can return a cleanup function. React calls this function:
- Before running the effect again (if dependencies changed)
- When the component unmounts

The pattern:

```typescript
useEffect(() => {
  // Setup: run side effect
  const subscription = subscribeToData();
  
  // Cleanup: undo the side effect
  return () => {
    subscription.unsubscribe();
  };
}, [dependencies]);
```

### Iteration 4: Adding Cleanup

We can't cancel a `fetch` request mid-flight (well, we can with `AbortController`, but let's start simpler). Instead, we'll use a flag to ignore the result if the component unmounts.

```tsx
// File: src/components/UserDashboard.tsx (Iteration 4)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    let isMounted = true; // ← Flag to track if component is mounted
    
    fetch('/api/user')
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        if (isMounted) { // ← Only update state if still mounted
          setUser(data);
          setIsLoading(false);
        }
      })
      .catch(err => {
        if (isMounted) { // ← Only update state if still mounted
          setError(err.message);
          setIsLoading(false);
        }
      });
    
    // Cleanup function
    return () => {
      isMounted = false; // ← Set flag to false when unmounting
    };
  }, []);
  
  if (isLoading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return (
      <div style={{ color: 'red' }}>
        <h2>Error loading user data</h2>
        <p>{error}</p>
      </div>
    );
  }
  
  if (!user) {
    return <div>No user data available</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Iteration 3):
```tsx
useEffect(() => {
  fetch('/api/user')
    .then(res => res.json())
    .then(data => {
      setUser(data);
      setIsLoading(false);
    });
}, []);
```

**After** (Iteration 4):
```tsx
useEffect(() => {
  let isMounted = true; // ← Added
  
  fetch('/api/user')
    .then(res => res.json())
    .then(data => {
      if (isMounted) { // ← Added check
        setUser(data);
        setIsLoading(false);
      }
    });
  
  return () => {
    isMounted = false; // ← Cleanup function
  };
}, []);
```

**Key changes**:
1. Added `isMounted` flag
2. Check flag before calling `setState`
3. Return cleanup function that sets flag to `false`

### Verification: No More Warnings

**Test scenario**: Toggle dashboard on/off rapidly before fetch completes

**Browser Console Output**:

```text
(No warnings)
```

**Expected vs. Actual**:
- ✅ No memory leak warnings
- ✅ State updates only happen if component is still mounted
- ✅ Component unmounts cleanly

### Using AbortController (Modern Approach)

The `isMounted` flag works, but there's a better way: `AbortController`. This actually cancels the fetch request.

```tsx
// File: src/components/UserDashboard.tsx (Iteration 4b - Modern)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

function UserDashboard() {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    fetch('/api/user', {
      signal: abortController.signal // ← Pass abort signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        setUser(data);
        setIsLoading(false);
      })
      .catch(err => {
        // AbortError is expected when we cancel
        if (err.name === 'AbortError') {
          console.log('Fetch aborted');
          return;
        }
        setError(err.message);
        setIsLoading(false);
      });
    
    // Cleanup: abort the fetch
    return () => {
      abortController.abort();
    };
  }, []);
  
  if (isLoading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return (
      <div style={{ color: 'red' }}>
        <h2>Error loading user data</h2>
        <p>{error}</p>
      </div>
    );
  }
  
  if (!user) {
    return <div>No user data available</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Iteration 4):
- Used `isMounted` flag
- Fetch continues but result is ignored

**After** (Iteration 4b):
- Use `AbortController`
- Fetch is actually cancelled
- Handle `AbortError` separately

**Benefits of AbortController**:
- Actually cancels the network request (saves bandwidth)
- Browser stops processing the response
- More explicit intent
- Standard Web API (works outside React)

### When to Apply This Solution

**What it optimizes for**:
- Memory safety (no state updates on unmounted components)
- Network efficiency (cancelled requests don't waste bandwidth)
- Clean component lifecycle

**What it sacrifices**:
- Slightly more complex code
- Need to handle `AbortError`

**When to choose this approach**:
- Any data fetching in `useEffect`
- Components that might unmount before async operations complete
- Long-running requests (large file downloads, slow APIs)

**When to avoid this approach**:
- Synchronous effects (no async operations)
- Effects that complete instantly
- When you're using a data fetching library (React Query handles this)

## Dependencies: When to Re-run Effects

So far, we've used an empty dependency array `[]`, which means "run once on mount." But what if we need to re-fetch data when something changes?

### Iteration 5: Fetching Based on Props

**Current state recap**: Our component fetches user data on mount. It works for a single, hardcoded user.

**Current limitation**: What if we want to show different users? We need to fetch new data when the user ID changes.

**New scenario introduction**: Let's make our component accept a `userId` prop and fetch that specific user's data.

```tsx
// File: src/components/UserDashboard.tsx (Iteration 5 - Broken)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    fetch(`/api/user/${userId}`, {
      signal: abortController.signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        setUser(data);
        setIsLoading(false);
      })
      .catch(err => {
        if (err.name === 'AbortError') return;
        setError(err.message);
        setIsLoading(false);
      });
    
    return () => {
      abortController.abort();
    };
  }, []); // ← Still empty array - this is the problem!
  
  if (isLoading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return (
      <div style={{ color: 'red' }}>
        <h2>Error loading user data</h2>
        <p>{error}</p>
      </div>
    );
  }
  
  if (!user) {
    return <div>No user data available</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

Now let's use it with a changing `userId`:

```tsx
// File: src/app/page.tsx

import { useState } from 'react';
import UserDashboard from '../components/UserDashboard';

export default function Page() {
  const [userId, setUserId] = useState('user-1');
  
  return (
    <div>
      <button onClick={() => setUserId('user-1')}>User 1</button>
      <button onClick={() => setUserId('user-2')}>User 2</button>
      <button onClick={() => setUserId('user-3')}>User 3</button>
      
      <UserDashboard userId={userId} />
    </div>
  );
}
```

### The Failure: Stale Data

**User action**:
1. Page loads, shows User 1's data
2. User clicks "User 2" button
3. Dashboard still shows User 1's data

**Browser Behavior**:
- Initial load: Shows User 1 correctly
- Click "User 2": Nothing changes
- Click "User 3": Still shows User 1
- Dashboard is stuck on the first user

**Browser Console Output**:

```text
Warning: React Hook useEffect has a missing dependency: 'userId'. 
Either include it or remove the dependency array. (react-hooks/exhaustive-deps)
```

**React DevTools - Components Tab**:
- `UserDashboard` component selected
- Props: `{ userId: "user-2" }` ← Changed
- State: `{ user: { id: "user-1", name: "Alice" } }` ← Didn't change
- Effect ran: 1 time (on mount only)

**Browser DevTools - Network Tab**:
- 1 request to `/api/user/user-1`
- No additional requests when clicking buttons

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Clicking "User 2" shows User 2's data
   - Actual: Dashboard stays on User 1

2. **What the console reveals**:
   - Key indicator: "missing dependency: 'userId'"
   - Translation: Your effect uses `userId` but doesn't list it in dependencies
   - This is a lint warning from `eslint-plugin-react-hooks`

3. **What DevTools shows**:
   - Props changed: `userId` went from "user-1" to "user-2"
   - State didn't change: `user` still has User 1's data
   - Effect didn't re-run: Only 1 network request total

4. **Root cause identified**:
   The effect has an empty dependency array `[]`, so it only runs once on mount. When `userId` changes, the effect doesn't re-run, so we never fetch the new user's data.

5. **Why the current approach can't solve this**:
   Empty dependency array means "never re-run." We need to tell React: "Re-run this effect when `userId` changes."

6. **What we need**:
   Add `userId` to the dependency array.

### Solution: Add Dependencies

Fix the dependency array:

```tsx
// File: src/components/UserDashboard.tsx (Iteration 5 - Fixed)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    // Reset state when starting new fetch
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/user/${userId}`, {
      signal: abortController.signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        setUser(data);
        setIsLoading(false);
      })
      .catch(err => {
        if (err.name === 'AbortError') return;
        setError(err.message);
        setIsLoading(false);
      });
    
    return () => {
      abortController.abort();
    };
  }, [userId]); // ← Added userId to dependency array
  
  if (isLoading) {
    return <div>Loading...</div>;
  }
  
  if (error) {
    return (
      <div style={{ color: 'red' }}>
        <h2>Error loading user data</h2>
        <p>{error}</p>
      </div>
    );
  }
  
  if (!user) {
    return <div>No user data available</div>;
  }
  
  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <span>Role: {user.role}</span>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Iteration 5 - Broken):
```tsx
useEffect(() => {
  fetch(`/api/user/${userId}`, ...)
    .then(...)
}, []); // ← Empty array
```

**After** (Iteration 5 - Fixed):
```tsx
useEffect(() => {
  setIsLoading(true); // ← Reset state
  setError(null);     // ← Reset state
  
  fetch(`/api/user/${userId}`, ...)
    .then(...)
}, [userId]); // ← Added dependency
```

**Key changes**:
1. Added `userId` to dependency array
2. Reset `isLoading` and `error` at start of effect
3. Effect now re-runs when `userId` changes

### Verification: Dependencies Work

**User action**: Click through User 1, User 2, User 3 buttons

**Browser Behavior**:
- Click "User 2": Shows loading, then User 2's data
- Click "User 3": Shows loading, then User 3's data
- Click "User 1": Shows loading, then User 1's data

**Browser Console Output**:

```text
(No warnings)
```

**React DevTools - Profiler Tab**:
- Each button click triggers:
  - Render 1: `isLoading: true` (loading state)
  - Render 2: `isLoading: false`, new user data
- Effect runs 4 times total:
  - Once on mount (User 1)
  - Once when changed to User 2
  - Once when changed to User 3
  - Once when changed back to User 1

**Browser DevTools - Network Tab**:
- 4 requests total:
  - `/api/user/user-1`
  - `/api/user/user-2`
  - `/api/user/user-3`
  - `/api/user/user-1` (when clicked again)

**Expected vs. Actual**:
- ✅ Effect re-runs when `userId` changes
- ✅ New data fetched for each user
- ✅ Loading state shown during fetch
- ✅ Previous fetch cancelled when new one starts (AbortController)

## The Dependency Array Rules

React has strict rules about dependencies. The `eslint-plugin-react-hooks` plugin enforces them.

### Rule 1: Include All Values Used Inside the Effect

If your effect uses a variable, prop, or state, it must be in the dependency array.

```tsx
// ❌ WRONG: userId used but not in dependencies
useEffect(() => {
  fetch(`/api/user/${userId}`);
}, []);

// ✅ CORRECT: userId in dependencies
useEffect(() => {
  fetch(`/api/user/${userId}`);
}, [userId]);
```

### Rule 2: Don't Lie About Dependencies

Never omit a dependency to "fix" a problem. If your effect runs too often, the problem is the effect logic, not the dependencies.

```tsx
// ❌ WRONG: Omitting count to prevent re-runs
useEffect(() => {
  console.log(count);
}, []); // Logs stale count

// ✅ CORRECT: Include count
useEffect(() => {
  console.log(count);
}, [count]); // Logs current count

// ✅ ALSO CORRECT: If you don't need count, don't use it
useEffect(() => {
  console.log('Component mounted');
}, []); // No dependencies needed
```

### Rule 3: Functions and Objects Need Special Handling

Functions and objects are recreated on every render, which can cause effects to re-run unnecessarily.

```tsx
// ❌ PROBLEM: fetchUser is recreated every render
function UserDashboard({ userId }: { userId: string }) {
  const fetchUser = () => {
    return fetch(`/api/user/${userId}`);
  };
  
  useEffect(() => {
    fetchUser(); // Effect re-runs every render
  }, [fetchUser]); // fetchUser is a new function each time
}

// ✅ SOLUTION 1: Move function inside effect
function UserDashboard({ userId }: { userId: string }) {
  useEffect(() => {
    const fetchUser = () => {
      return fetch(`/api/user/${userId}`);
    };
    
    fetchUser(); // Effect only re-runs when userId changes
  }, [userId]);
}

// ✅ SOLUTION 2: Use useCallback (covered in Chapter 25)
function UserDashboard({ userId }: { userId: string }) {
  const fetchUser = useCallback(() => {
    return fetch(`/api/user/${userId}`);
  }, [userId]);
  
  useEffect(() => {
    fetchUser();
  }, [fetchUser]); // fetchUser only changes when userId changes
}
```

### When to Apply: Dependency Array Decision Tree

| Scenario | Dependency Array | Example |
|----------|------------------|---------|
| Run once on mount | `[]` | Analytics page view, initial data fetch |
| Run when value changes | `[value]` | Fetch data when ID changes |
| Run on every render | No array | Sync with external system (rare) |
| Use multiple values | `[val1, val2]` | Fetch when either ID or filter changes |
| Use function/object | Move inside effect or use `useCallback`/`useMemo` | Complex fetch logic |

## Current Limitation

Our component now handles:
- ✅ Initial data fetching
- ✅ Error handling
- ✅ Cleanup on unmount
- ✅ Re-fetching when dependencies change

But we still have issues:
1. **Race conditions with rapid changes**: What if `userId` changes twice quickly?
2. **No caching**: We re-fetch the same user multiple times
3. **Boilerplate**: Every component needs this same loading/error/data pattern

These problems are why data fetching libraries exist. But first, let's understand the remaining pitfalls.

## Common pitfalls and how to avoid them

## The Race Condition: Rapid Dependency Changes

**Current state recap**: Our component re-fetches data when `userId` changes. Each fetch is properly cancelled with `AbortController`.

**Current limitation**: What if the user clicks through users very quickly? Multiple fetches start, and they might complete out of order.

**New scenario introduction**: Let's simulate a slow API and rapid user clicks.

### The Failure: Wrong Data Displayed

Imagine this sequence:
1. User clicks "User 2" → Fetch starts (takes 2 seconds)
2. User clicks "User 3" → Previous fetch cancelled, new fetch starts (takes 1 second)
3. Fetch for User 3 completes first → Shows User 3 ✅
4. Fetch for User 2 completes second → Shows User 2 ❌

Wait, we cancelled the User 2 fetch! But what if cancellation fails, or the response was already in flight?

Let's see this in action with a simulated slow API:

```tsx
// File: src/components/UserDashboard.tsx (Iteration 6 - Demonstrating race condition)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

interface UserDashboardProps {
  userId: string;
}

// Simulate slow API with random delays
async function fetchUserWithDelay(userId: string): Promise<User> {
  const delay = Math.random() * 2000 + 500; // 500-2500ms
  await new Promise(resolve => setTimeout(resolve, delay));
  
  const response = await fetch(`/api/user/${userId}`);
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  return response.json();
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    setIsLoading(true);
    setError(null);
    
    fetchUserWithDelay(userId)
      .then(data => {
        setUser(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [userId]);
  
  if (isLoading) {
    return <div>Loading user {userId}...</div>;
  }
  
  if (error) {
    return <div style={{ color: 'red' }}>{error}</div>;
  }
  
  if (!user) {
    return <div>No user data</div>;
  }
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>User ID: {user.id}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}

export default UserDashboard;
```

### Diagnostic Analysis: Race Condition Evidence

**User action**: Rapidly click User 1 → User 2 → User 3 → User 4

**Browser Behavior**:
- Shows "Loading user 1..."
- Shows "Loading user 2..."
- Shows "Loading user 3..."
- Shows "Loading user 4..."
- Shows User 3's data (completed first)
- Then suddenly shows User 1's data (completed last)

**Browser Console Output**:

```text
Fetching user-1...
Fetching user-2...
Fetching user-3...
Fetching user-4...
User 3 loaded (1.2s)
User 4 loaded (1.5s)
User 2 loaded (1.8s)
User 1 loaded (2.3s)
```

**React DevTools - Components Tab**:
- Props: `{ userId: "user-4" }` ← Current prop
- State: `{ user: { id: "user-1", ... } }` ← Wrong user!

**Browser DevTools - Network Tab**:
- 4 requests started in quick succession
- Completed out of order: 3, 4, 2, 1
- Last response (User 1) overwrote the correct data (User 4)

### The Problem: Last Response Wins

Even with `AbortController`, race conditions can occur:
1. Cancellation might not be instant
2. Response might already be in flight
3. Server might not respect cancellation
4. Network timing is unpredictable

The last `setUser` call wins, regardless of which `userId` is current.

### Solution: Ignore Stale Responses

We need to track which fetch is current and ignore responses from old fetches:

```tsx
// File: src/components/UserDashboard.tsx (Iteration 6 - Fixed)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    let isCurrentFetch = true; // ← Track if this is the current fetch
    const abortController = new AbortController();
    
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/user/${userId}`, {
      signal: abortController.signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        if (isCurrentFetch) { // ← Only update if this is still the current fetch
          setUser(data);
          setIsLoading(false);
        }
      })
      .catch(err => {
        if (err.name === 'AbortError') return;
        if (isCurrentFetch) { // ← Only update if this is still the current fetch
          setError(err.message);
          setIsLoading(false);
        }
      });
    
    return () => {
      isCurrentFetch = false; // ← Mark this fetch as stale
      abortController.abort();
    };
  }, [userId]);
  
  if (isLoading) {
    return <div>Loading user {userId}...</div>;
  }
  
  if (error) {
    return <div style={{ color: 'red' }}>{error}</div>;
  }
  
  if (!user) {
    return <div>No user data</div>;
  }
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>User ID: {user.id}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}

export default UserDashboard;
```

### What Changed

**Before** (Race condition):
```tsx
useEffect(() => {
  fetch(`/api/user/${userId}`)
    .then(data => {
      setUser(data); // Always updates, even if stale
    });
}, [userId]);
```

**After** (Race condition fixed):
```tsx
useEffect(() => {
  let isCurrentFetch = true; // ← Added
  
  fetch(`/api/user/${userId}`)
    .then(data => {
      if (isCurrentFetch) { // ← Check before updating
        setUser(data);
      }
    });
  
  return () => {
    isCurrentFetch = false; // ← Mark as stale
  };
}, [userId]);
```

**Key changes**:
1. Added `isCurrentFetch` flag
2. Check flag before calling `setState`
3. Set flag to `false` in cleanup
4. Combined with `AbortController` for double protection

### Verification: Race Condition Fixed

**User action**: Rapidly click through users

**Browser Behavior**:
- Shows loading states
- Shows User 4's data (the current user)
- Never shows stale data from earlier fetches

**React DevTools - Components Tab**:
- Props: `{ userId: "user-4" }`
- State: `{ user: { id: "user-4", ... } }` ← Correct!

**Expected vs. Actual**:
- ✅ Only the current fetch updates state
- ✅ Stale responses are ignored
- ✅ Displayed data always matches current `userId`

## Common Failure Modes and Their Signatures

### Symptom: Infinite Loop

**Browser behavior**:
- Page freezes
- Browser tab becomes unresponsive
- Fan spins up

**Console pattern**:

```text
Warning: Maximum update depth exceeded.
```

**DevTools clues**:
- Profiler shows hundreds of renders in seconds
- Network tab shows continuous requests

**Root cause**: Side effect in component body or effect without proper dependencies

**Solution**: Move side effect into `useEffect` with correct dependency array

### Symptom: Stale Data

**Browser behavior**:
- Component shows old data
- Updates don't reflect in UI

**Console pattern**:

```text
Warning: React Hook useEffect has a missing dependency: 'userId'.
```

**DevTools clues**:
- Props changed but state didn't
- Effect ran fewer times than expected

**Root cause**: Missing dependency in `useEffect` array

**Solution**: Add all used values to dependency array

### Symptom: Memory Leak Warning

**Browser behavior**:
- Component works but console shows warnings
- Happens when navigating away quickly

**Console pattern**:

```text
Warning: Can't perform a React state update on an unmounted component.
```

**DevTools clues**:
- Component unmounted but async operation completed
- State update attempted after unmount

**Root cause**: No cleanup function in `useEffect`

**Solution**: Return cleanup function that cancels async operations or sets flag

### Symptom: Effect Runs Too Often

**Browser behavior**:
- Component works but feels slow
- Unnecessary network requests

**Console pattern**:

```text
(No errors, but Network tab shows many requests)
```

**DevTools clues**:
- Profiler shows effect running on every render
- Network tab shows duplicate requests

**Root cause**: 
- No dependency array (runs every render)
- Object/function in dependencies (recreated every render)

**Solution**: 
- Add dependency array
- Move functions inside effect
- Use `useCallback` for functions (Chapter 25)

### Symptom: Wrong Data After Rapid Changes

**Browser behavior**:
- Click through options quickly
- Wrong option's data displayed

**Console pattern**:

```text
(No errors)
```

**DevTools clues**:
- Props show current value
- State shows old value
- Network tab shows requests completed out of order

**Root cause**: Race condition - old fetch completed after new fetch

**Solution**: Use `isCurrentFetch` flag or `AbortController`

## Debugging Workflow: When Your Effect Fails

### Step 1: Observe the User Experience

**Questions to ask**:
- Does the component render at all?
- Does it show loading state?
- Does it show error state?
- Does it show stale data?
- Does it freeze or crash?

### Step 2: Check the Console

**Look for**:
- "Maximum update depth exceeded" → Infinite loop
- "Missing dependency" → Incomplete dependency array
- "Can't perform state update on unmounted component" → Missing cleanup
- Network errors → API issues

### Step 3: Inspect with React DevTools

**Components tab**:
- Check current props and state
- Verify they match what you expect
- Look for unexpected values

**Profiler tab**:
- Record a session
- Check how many times component renders
- Look for unexpected re-renders
- Check effect execution count

### Step 4: Analyze Network Activity

**Network tab**:
- Filter by Fetch/XHR
- Check request count (too many? too few?)
- Check request timing (out of order?)
- Check response status codes

### Step 5: Reproduce Minimally

**Isolate the problem**:
- Remove unrelated code
- Test with hardcoded values
- Test with simplified logic
- Verify the effect in isolation

### Step 6: Apply the Fix

**Decision tree**:
- Infinite loop? → Add `useEffect` wrapper
- Stale data? → Add missing dependencies
- Memory leak? → Add cleanup function
- Race condition? → Add `isCurrentFetch` flag
- Too many re-runs? → Check dependency array

## The Journey: From Problem to Solution

| Iteration | Failure Mode | Technique Applied | Result | Performance Impact |
|-----------|--------------|-------------------|--------|-------------------|
| 0 | Infinite render loop | None | Crashes browser | N/A |
| 1 | Loop fixed, no error handling | `useEffect` with `[]` | Works but fragile | 1 request on mount |
| 2 | Silent failures | Error state + `.catch()` | User sees errors | Same |
| 3 | Memory leak warnings | Cleanup function | Clean unmount | Same |
| 4 | Stale data on prop change | Dependency array `[userId]` | Re-fetches correctly | N requests for N users |
| 5 | Race conditions | `isCurrentFetch` flag | Always shows current data | Same, but ignores stale |

### Final Implementation

Here's our complete, production-ready data fetching component:

```tsx
// File: src/components/UserDashboard.tsx (Final - Production Ready)

import { useState, useEffect } from 'react';
import type { User } from '../types/user';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    // Track if this fetch is still current
    let isCurrentFetch = true;
    
    // Create abort controller for cancellation
    const abortController = new AbortController();
    
    // Reset state when starting new fetch
    setIsLoading(true);
    setError(null);
    
    // Fetch data
    fetch(`/api/user/${userId}`, {
      signal: abortController.signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }
        return res.json();
      })
      .then(data => {
        // Only update state if this is still the current fetch
        if (isCurrentFetch) {
          setUser(data);
          setIsLoading(false);
        }
      })
      .catch(err => {
        // Ignore abort errors (expected when cancelling)
        if (err.name === 'AbortError') {
          return;
        }
        
        // Only update state if this is still the current fetch
        if (isCurrentFetch) {
          setError(err.message);
          setIsLoading(false);
        }
      });
    
    // Cleanup function
    return () => {
      // Mark this fetch as stale
      isCurrentFetch = false;
      
      // Cancel the fetch
      abortController.abort();
    };
  }, [userId]); // Re-run when userId changes
  
  // Loading state
  if (isLoading) {
    return (
      <div className="loading">
        <div className="spinner" />
        <p>Loading user data...</p>
      </div>
    );
  }
  
  // Error state
  if (error) {
    return (
      <div className="error">
        <h2>Error loading user data</h2>
        <p>{error}</p>
        <button onClick={() => window.location.reload()}>
          Retry
        </button>
      </div>
    );
  }
  
  // No data state
  if (!user) {
    return (
      <div className="empty">
        <p>No user data available</p>
      </div>
    );
  }
  
  // Success state
  return (
    <div className="user-dashboard">
      <div className="user-header">
        <img 
          src={user.avatar} 
          alt={user.name}
          className="user-avatar"
        />
        <div className="user-info">
          <h1>{user.name}</h1>
          <p>{user.email}</p>
          <span className="user-role">{user.role}</span>
        </div>
      </div>
    </div>
  );
}

export default UserDashboard;
```

### Decision Framework: Data Fetching Patterns

When you need to fetch data in a component, choose your approach based on these criteria:

| Scenario | Pattern | Example |
|----------|---------|---------|
| Fetch once on mount | `useEffect` with `[]` | Initial page data, user profile |
| Fetch when prop changes | `useEffect` with `[prop]` | Search results, filtered lists |
| Fetch on user action | Event handler + state | Form submission, button click |
| Frequent refetching | Data fetching library | Real-time data, polling |
| Complex caching needs | React Query / SWR | Multi-page app, shared data |

### When to Apply: useEffect for Data Fetching

**What it optimizes for**:
- Simple, one-off data fetching
- Learning React fundamentals
- Full control over fetch logic

**What it sacrifices**:
- No caching (refetch every time)
- No automatic retries
- Manual loading/error state management
- Boilerplate code in every component

**When to choose this approach**:
- Learning React (understand the fundamentals first)
- Simple apps with few data fetching needs
- One-off fetches that don't need caching
- Custom fetch logic that libraries don't support

**When to avoid this approach**:
- Multiple components fetch the same data
- Need caching, retries, or background refetching
- Complex loading states (pagination, infinite scroll)
- Production apps with many API calls

**Code characteristics**:
- Setup complexity: Low (just `useEffect`)
- Maintenance burden: High (repeat pattern in every component)
- Performance impact: No caching, refetch on every mount

## Lessons Learned

### 1. Side Effects Need Isolation

React components are pure functions. Side effects must be isolated in `useEffect` to prevent infinite loops and unpredictable behavior.

### 2. Cleanup Prevents Memory Leaks

Always return a cleanup function from `useEffect` when dealing with async operations, subscriptions, or timers. This prevents state updates on unmounted components.

### 3. Dependencies Must Be Honest

The dependency array is not optional or negotiable. Include every value your effect uses. The linter is your friend—listen to it.

### 4. Race Conditions Are Real

When effects depend on changing values, responses can arrive out of order. Use flags or cancellation to ignore stale results.

### 5. Three States Are Mandatory

Every async operation has three states: loading, error, and success. Handle all three explicitly in your UI.

### 6. This Is Just the Beginning

`useEffect` with manual fetch is the foundation, but production apps use data fetching libraries (React Query, SWR) that handle caching, retries, and race conditions automatically. We'll cover these in Chapter 13.

### The Professional Pattern

A professional React developer:
- Wraps side effects in `useEffect`
- Includes all dependencies honestly
- Returns cleanup functions
- Handles loading, error, and success states
- Prevents race conditions with flags or cancellation
- Knows when to reach for a library instead

You now understand the mechanics of side effects in React. This knowledge is the foundation for everything else—data fetching libraries, custom hooks, and complex state management all build on these principles.

In the next chapter, we'll apply these patterns to more complex scenarios: rendering lists efficiently, handling conditional rendering, and understanding why keys matter.
