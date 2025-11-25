# Chapter 5: Lists, Keys, and Conditional Rendering

## Rendering arrays efficiently

## The Challenge: From Data to UI

So far, we've dealt with single pieces of state. But most applications handle collections of data: a list of users, a feed of posts, a gallery of images. The core task is transforming an array of data into a list of UI elements.

React makes this straightforward with standard JavaScript methods like `Array.prototype.map()`, but doing it correctly and efficiently requires understanding a crucial concept: **identity**. React needs to know which UI element corresponds to which piece of data, especially when that data changes.

This chapter will build a single, realistic component from the ground up, demonstrating how to render lists, why React's `key` prop is non-negotiable, and how to conditionally show, hide, or change UI based on application state.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a `NotificationPanel`. It's a perfect candidate because notifications are inherently a list, and they are dynamic: they can be added, removed, and their state (e.g., "read" or "unread") can change.

Let's start with a basic version that simply displays a hardcoded list of notifications.

**Project Structure**:
```
src/
└── components/
    └── NotificationPanel.tsx  ← Our reference implementation
```

Here is our initial data and the first, naive implementation.

```tsx
// src/components/NotificationPanel.tsx

import { useState } from 'react';

type Notification = {
  id: string;
  type: 'info' | 'warning' | 'error';
  message: string;
};

const initialNotifications: Notification[] = [
  { id: 'notif-1', type: 'info', message: 'Your profile has been updated.' },
  { id: 'notif-2', type: 'warning', message: 'Your subscription is expiring soon.' },
  { id: 'notif-3', type: 'error', message: 'Failed to upload document.' },
];

export function NotificationPanel() {
  const [notifications, setNotifications] = useState(initialNotifications);

  return (
    <div className="notification-panel">
      <h2>Notifications</h2>
      <div className="notification-list">
        {notifications.map((notification) => (
          <div className={`notification notification--${notification.type}`}>
            {notification.message}
          </div>
        ))}
      </div>
    </div>
  );
}
```

When you run this, it appears to work. You'll see the three notifications rendered on the screen. However, if you open your browser's developer console, you'll see a critical warning. This is our first failure.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The UI displays the three notifications as expected. Visually, nothing appears broken.

**Browser Console Output**:
```
Warning: Each child in a list should have a unique "key" prop.

Check the render method of `NotificationPanel`.
    at div
    at NotificationPanel (http://localhost:3000/static/js/bundle.js:...)
    ...
```

**React DevTools Evidence**:
The component tree shows the `NotificationPanel` with three `div` children inside `notification-list`. React has rendered them, but it's unhappy about it.

**Let's parse this evidence**:

1.  **What the user experiences**: Everything looks fine. This is a deceptive kind of failure—it's a performance and correctness time bomb, not an immediate crash.
2.  **What the console reveals**: React is explicitly telling us the problem. When it sees an array of elements returned from a `.map()`, it needs a `key` prop on each element to distinguish them.
3.  **Root cause identified**: We are rendering a list of elements without providing the `key` prop that React requires for list reconciliation.
4.  **Why the current approach can't solve this**: Without keys, React doesn't know if the first `div` in the list is the *same* `div` on the next render. If we add, remove, or re-sort items, React has no stable identity to track which item is which. It might resort to re-creating all the DOM elements from scratch, which is inefficient and can lead to bugs.
5.  **What we need**: A way to provide a stable, unique identifier for each rendered element in the list so React can track it across renders.

## Why keys matter (and what happens when you get them wrong)

## Iteration 1: The Index-as-Key Anti-Pattern

The console warning is clear: we need a `key`. A common, but often incorrect, first attempt is to use the array index from the `.map()` function. Let's try it.

**Before** (Iteration 0):

```tsx
// src/components/NotificationPanel.tsx - Version 0
// ...
{notifications.map((notification) => (
  <div className={`notification notification--${notification.type}`}>
    {notification.message}
  </div>
))}
// ...
```

**After** (Iteration 1):

```tsx
// src/components/NotificationPanel.tsx - Version 1
// ...
{notifications.map((notification, index) => (
  <div 
    key={index} // ← Added index as key
    className={`notification notification--${notification.type}`}
  >
    {notification.message}
  </div>
))}
// ...
```

The console warning is gone! Problem solved, right?

**Not so fast.** Using the index as a key is one of the most common and subtle bugs in React. It silences the warning but creates deeper issues.

### The Limitation of Index-as-Key

The `key` prop is not for you; it's for React. It's a hint that helps React identify which items have changed, are added, or are removed. A key should be **stable** (it doesn't change for a given item), **predictable**, and **unique**.

An item's index in an array is *not* stable. If you delete the first item, the second item's index becomes `0`. If you sort the array, every item's index can change. By using `key={index}`, you are telling React that the item's identity is tied to its *position*, not the data it represents.

### New Scenario: Dismissing a Notification

Let's prove this is a problem. We'll add a button to each notification to dismiss it. To make the bug obvious, we'll also add an uncontrolled `<input>` field inside each notification item. This input will hold its own state, separate from React's state, which will visually expose the underlying issue.

```tsx
// src/components/NotificationPanel.tsx - Version 1.1 (demonstrating failure)

import { useState } from 'react';

// ... (Notification type and initialNotifications are the same)

// Let's create a sub-component for clarity
function NotificationItem({ notification, onDismiss }) {
  return (
    <div 
      // The anti-pattern we are testing
      className={`notification notification--${notification.type}`}
    >
      {notification.message}
      {/* This input will help us visualize the bug */}
      <input type="text" placeholder="Add a note..." />
      <button onClick={onDismiss}>Dismiss</button>
    </div>
  );
}

export function NotificationPanel() {
  const [notifications, setNotifications] = useState(initialNotifications);

  const handleDismiss = (idToDismiss: string) => {
    setNotifications(current => current.filter(n => n.id !== idToDismiss));
  };

  return (
    <div className="notification-panel">
      <h2>Notifications</h2>
      <div className="notification-list">
        {notifications.map((notification, index) => (
          <NotificationItem
            key={index} // ← The problematic key
            notification={notification}
            onDismiss={() => handleDismiss(notification.id)}
          />
        ))}
      </div>
    </div>
  );
}
```

Now, let's run the failure scenario:

1.  Run the app.
2.  In the first notification's input field ("Your profile has been updated."), type "Checked this".
3.  In the second notification's input field ("Your subscription is expiring soon."), type "Need to renew".
4.  Now, click the "Dismiss" button on the **first** notification.

### Diagnostic Analysis: The Index-Key Failure

**Browser Behavior**:
When you dismiss the first notification, the list shrinks from three items to two. However, you'll see something strange:

- The text "Your profile has been updated." disappears.
- The text "Your subscription is expiring soon." moves up to the first position.
- The text "Failed to upload document." moves up to the second position.
- **Crucially**, the input field in the first position still says "Checked this", and the second input field still says "Need to renew". The input state did not move with the notification message it was next to. The last notification item, along with its empty input field, is what actually got removed from the DOM.

**React DevTools Evidence**:
- **Before Dismiss**: The component tree shows three `NotificationItem` components with keys `0`, `1`, and `2`.
- **After Dismiss**: The component tree shows two `NotificationItem` components with keys `0` and `1`.
- React sees that it used to have components for keys `0, 1, 2` and now it only needs components for keys `0, 1`. Its most efficient move is to simply discard the component and DOM node for `key=2`. It then updates the props for the components at `key=0` and `key=1` with the new data at `index=0` and `index=1`. The components themselves (and their internal state, like our input) are preserved.

**Let's parse this evidence**:

1.  **What the user experiences**: The UI is now in a corrupted state. The data (notification message) and the component's internal state (the input value) are mismatched.
2.  **What DevTools reveals**: React didn't remove the first component. It removed the *last* component and just passed new props to the first two.
3.  **Root cause identified**: Using the index as a key tells React that `NotificationItem key={0}` is the same conceptual component before and after the state update. React re-renders it with new props (`notification` object for `id: 'notif-2'`), but since it's an uncontrolled input, its internal state is preserved.
4.  **Why the current approach can't solve this**: The index key breaks the link between the data's identity and the component's identity. It forces React to make incorrect assumptions about which component corresponds to which piece of data.
5.  **What we need**: A key that is tied to the data itself. A key that is stable and unique for each notification, no matter its position in the array.

### Iteration 2: The Correct Solution (Stable, Unique Keys)

The solution is to use a piece of data that uniquely identifies each notification. In our `Notification` type, the `id` property is perfect for this.

**Before** (Iteration 1.1):

```tsx
// src/components/NotificationPanel.tsx - Version 1.1
// ...
{notifications.map((notification, index) => (
  <NotificationItem
    key={index} // ← The problem
    notification={notification}
    onDismiss={() => handleDismiss(notification.id)}
  />
))}
// ...
```

**After** (Iteration 2):

```tsx
// src/components/NotificationPanel.tsx - Version 2
// ...
{notifications.map((notification) => (
  <NotificationItem
    key={notification.id} // ← The solution: use a stable, unique ID
    notification={notification}
    onDismiss={() => handleDismiss(notification.id)}
  />
))}
// ...
```

### Verification

Let's re-run the exact same test scenario:

1.  Run the app.
2.  Type "Checked this" into the first notification's input.
3.  Type "Need to renew" into the second notification's input.
4.  Click "Dismiss" on the **first** notification.

**Expected vs. Actual Improvement**:
- **Expected**: The first notification, along with its input field containing "Checked this", should disappear entirely. The second notification, with its input field containing "Need to renew", should remain as the first item in the list.
- **Actual**: This is exactly what happens! The UI behaves correctly. React, using the stable `id` as the key, correctly identified that the component with `key="notif-1"` was removed and updated the DOM accordingly, preserving the correct state for the remaining components.

### When to Apply This Solution

- **When to use stable IDs for keys**: **Almost always.** Whenever you are rendering a list from an array of objects, and those objects have a unique identifier, use it as the key. This is the correct, default approach.
- **When is `key={index}` acceptable?**: Only in the rare case that all of the following are true:
    1. The list and its items are purely static; they will never be re-ordered, filtered, or have items added/removed from the middle.
    2. The items in the list have no IDs.
    3. The component is simple and has no internal state of its own.

Even then, it's often better to generate temporary IDs if possible. As a rule of thumb: **avoid index as key**.

## Conditional rendering patterns

## Handling Dynamic UI States

Our `NotificationPanel` is better, but it's still naive. What happens if the list of notifications is empty? Or while we're fetching them from a server? A robust component must handle all possible states, not just the "happy path" where data exists. This is where conditional rendering comes in.

Let's enhance our `NotificationPanel` to handle an empty state.

### Pattern 1: Ternary Operator (`condition ? <A /> : <B />`)

The ternary operator is perfect for simple if-else logic directly inside your JSX. If the notifications array is empty, we want to show a message; otherwise, we show the list.

**Before** (Iteration 2):

```tsx
// src/components/NotificationPanel.tsx - Version 2
// ...
return (
  <div className="notification-panel">
    <h2>Notifications</h2>
    <div className="notification-list">
      {notifications.map((notification) => (
        <NotificationItem /* ... */ />
      ))}
    </div>
  </div>
);
// ...
```

**After** (Iteration 3.1):

```tsx
// src/components/NotificationPanel.tsx - Version 3.1
// ...
export function NotificationPanel() {
  const [notifications, setNotifications] = useState(initialNotifications);
  // ... handleDismiss function

  return (
    <div className="notification-panel">
      <h2>Notifications</h2>
      <div className="notification-list">
        {notifications.length > 0 ? (
          notifications.map((notification) => (
            <NotificationItem
              key={notification.id}
              notification={notification}
              onDismiss={() => handleDismiss(notification.id)}
            />
          ))
        ) : (
          <p className="empty-state">No new notifications.</p>
        )}
      </div>
    </div>
  );
}
```

Now, if you change `initialNotifications` to an empty array `[]`, the panel will correctly display "No new notifications."

### Pattern 2: Logical AND (`condition && <Expression />`)

This pattern is ideal for situations where you want to render something *only if* a condition is true, and render nothing otherwise. Let's add an "unread" indicator to our notifications. We'll add a `read` property to our data.

```typescript
// src/components/NotificationPanel.tsx - Updated types and data

type Notification = {
  id: string;
  type: 'info' | 'warning' | 'error';
  message: string;
  read: boolean; // ← Added property
};

const initialNotifications: Notification[] = [
  { id: 'notif-1', type: 'info', message: 'Your profile has been updated.', read: false },
  { id: 'notif-2', type: 'warning', message: 'Your subscription is expiring soon.', read: false },
  { id: 'notif-3', type: 'error', message: 'Failed to upload document.', read: true },
];
```

```tsx
// src/components/NotificationPanel.tsx - Version 3.2 (in NotificationItem)

function UnreadIndicator() {
  return <span className="unread-dot" aria-label="unread"></span>;
}

function NotificationItem({ notification, onDismiss }) {
  return (
    <div className={`notification notification--${notification.type} ${notification.read ? 'is-read' : ''}`}>
      {!notification.read && <UnreadIndicator />} {/* ← Logical AND */}
      <span className="message">{notification.message}</span>
      <input type="text" placeholder="Add a note..." />
      <button onClick={onDismiss}>Dismiss</button>
    </div>
  );
}
```

This works perfectly. The `UnreadIndicator` component will only be rendered if `!notification.read` is true.

#### Common Failure Mode: The `0` Gotcha

A classic trap with the `&&` pattern occurs when the condition is a number. In JavaScript, `0` is "falsy", but it's not `null` or `undefined`. React renders the number `0` as text in the DOM.

**Symptom**: You see a `0` unexpectedly rendered on your page.

**Example Failure**:
```tsx
const messageCount = 0;

return (
  <div>
    {messageCount && <p>You have new messages!</p>}
  </div>
);
```
**Browser Behavior**: The page will render `<div>0</div>`.

**Root Cause**: The expression `0 && <p>...</p>` evaluates to `0` in JavaScript, which React then renders.

**Solution**: Ensure your condition is a true boolean.
```tsx
// Fix 1: Explicit comparison
{messageCount > 0 && <p>You have new messages!</p>}

// Fix 2: Coerce to boolean
{Boolean(messageCount) && <p>You have new messages!</p>}
```

### Pattern 3: `if` Statements and Early Returns

When a component has multiple, mutually exclusive states (like loading, error, and success), nesting ternaries can become messy. A cleaner pattern is to use "guard clauses" or "early returns" at the top of your component.

Let's simulate loading and error states in our `NotificationPanel`.

```tsx
// src/components/NotificationPanel.tsx - Version 3.3

import { useState, useEffect } from 'react';

// ... Notification type, NotificationItem, etc.

export function NotificationPanel() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // Simulate a network request
    const fetchNotifications = async () => {
      try {
        await new Promise(resolve => setTimeout(resolve, 1500)); // Fake delay
        // To test the error state, uncomment the next line:
        // throw new Error("Could not fetch notifications.");
        setNotifications(initialNotifications);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    };
    fetchNotifications();
  }, []);

  const handleDismiss = (idToDismiss: string) => {
    setNotifications(current => current.filter(n => n.id !== idToDismiss));
  };

  // --- Early Return Guard Clauses ---
  if (isLoading) {
    return <div className="panel-loading">Loading notifications...</div>;
  }

  if (error) {
    return <div className="panel-error">Error: {error}</div>;
  }
  // ------------------------------------

  return (
    <div className="notification-panel">
      <h2>Notifications ({notifications.length})</h2>
      <div className="notification-list">
        {notifications.length > 0 ? (
          notifications.map((notification) => (
            <NotificationItem
              key={notification.id}
              notification={notification}
              onDismiss={() => handleDismiss(notification.id)}
            />
          ))
        ) : (
          <p className="empty-state">No new notifications.</p>
        )}
      </div>
    </div>
  );
}
```

This code is much easier to read. The main `return` statement at the end can assume that loading is finished and there are no errors. It only has to worry about the "success" and "empty" states.

### Decision Framework: Choosing a Pattern

| Scenario                               | Best Pattern                               | Example                                                              |
| -------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------- |
| Simple "if-this-then-that-else-other"  | Ternary Operator (`? :`)                   | `isLoggedIn ? <UserProfile /> : <LoginForm />`                       |
| "Show this or show nothing"            | Logical AND (`&&`)                         | `isAdmin && <AdminDashboardLink />`                                  |
| Multiple exclusive top-level states    | `if` statements / Early Returns            | `if (loading) ...`, `if (error) ...`, `return <DataView />`          |
| Filtering items within a list          | Pre-filter array before `.map()`           | `items.filter(item => item.isActive).map(...)`                       |

## Avoiding unnecessary re-renders

## The Performance Cost of Change

Our component is now robust, but is it performant? Every time a component's state or props change, it re-renders. This is usually fine, but in a list, if the parent component re-renders, it can cause *every single item in the list* to re-render, even if those items haven't changed.

Let's introduce a new feature to demonstrate this problem: a filter input that lets the user search their notifications.

### Iteration 4: Introducing a Parent State Change

We'll add a simple text input to the `NotificationPanel`. As the user types, we'll update a `filterText` state variable.

```tsx
// src/components/NotificationPanel.tsx - Version 4 (demonstrating failure)

// In NotificationItem, add a console.log to see when it renders
function NotificationItem({ notification, onDismiss }) {
  console.log(`Rendering NotificationItem: ${notification.id}`);
  // ... rest of the component is the same
  return ( /* ... */ );
}

export function NotificationPanel() {
  // ... all the existing state and logic ...
  const [filterText, setFilterText] = useState('');

  // ... useEffect for fetching ...
  // ... handleDismiss ...

  if (isLoading) { /* ... */ }
  if (error) { /* ... */ }

  const filteredNotifications = notifications.filter(n => 
    n.message.toLowerCase().includes(filterText.toLowerCase())
  );

  return (
    <div className="notification-panel">
      <h2>Notifications ({filteredNotifications.length})</h2>
      
      <input 
        type="text"
        placeholder="Filter notifications..."
        value={filterText}
        onChange={(e) => setFilterText(e.target.value)}
        className="filter-input"
      />

      <div className="notification-list">
        {filteredNotifications.length > 0 ? (
          filteredNotifications.map((notification) => (
            <NotificationItem
              key={notification.id}
              notification={notification}
              onDismiss={() => handleDismiss(notification.id)}
            />
          ))
        ) : (
          <p className="empty-state">No matching notifications.</p>
        )}
      </div>
    </div>
  );
}
```

Now, run the app and open the console. Start typing in the "Filter notifications..." input box.

### Diagnostic Analysis: The Re-render Cascade

**Browser Behavior**:
The filtering works correctly. As you type, the list of notifications updates to show only matching items.

**Browser Console Output**:
With every single keystroke, you will see a flood of logs:
```
Rendering NotificationItem: notif-1
Rendering NotificationItem: notif-2
Rendering NotificationItem: notif-3
Rendering NotificationItem: notif-1
Rendering NotificationItem: notif-2
Rendering NotificationItem: notif-3
... and so on for every key press
```

**React DevTools Profiler Evidence**:
If you record a profiling session while typing, you'll see that with each keystroke, the `NotificationPanel` re-renders (which is correct, its state changed), and consequently, *all* of the visible `NotificationItem` components also re-render, even though their own `notification` prop has not changed.

**Let's parse this evidence**:

1.  **What the user experiences**: The UI works, but for very long lists or complex list items, it might feel sluggish.
2.  **What the console reveals**: Every `NotificationItem` is re-rendering on every keystroke.
3.  **Root cause identified**: When `NotificationPanel`'s state (`filterText`) changes, it re-renders. During its render, it calls `.map()` on the `filteredNotifications` array, creating a new set of `NotificationItem` elements. React compares the new elements to the old ones. Even though the props might be identical, React re-renders the child components by default.
4.  **Why the current approach can't solve this**: By default, React's philosophy is to re-render children when a parent re-renders. We need a way to opt out of this behavior.
5.  **What we need**: A way to tell React, "Hey, this `NotificationItem` component is pure. If its props don't change, don't bother re-rendering it."

### The Solution: `React.memo`

React provides a Higher-Order Component called `React.memo` for exactly this purpose. It "memoizes" the component, meaning it saves the result of the last render and reuses it if the props are the same on the next render.

**Before** (Iteration 4):

```tsx
// src/components/NotificationPanel.tsx - Version 4
function NotificationItem({ notification, onDismiss }) {
  console.log(`Rendering NotificationItem: ${notification.id}`);
  // ...
}
```

**After** (Iteration 5):

```tsx
// src/components/NotificationPanel.tsx - Version 5
import React, { useState, useEffect } from 'react'; // Import React

// ...

const NotificationItem = React.memo(function NotificationItem({ notification, onDismiss }) {
  console.log(`Rendering NotificationItem: ${notification.id}`);
  // ... same component implementation
  return (
    <div className={`notification notification--${notification.type} ${notification.read ? 'is-read' : ''}`}>
      {!notification.read && <UnreadIndicator />}
      <span className="message">{notification.message}</span>
      <input type="text" placeholder="Add a note..." />
      <button onClick={onDismiss}>Dismiss</button>
    </div>
  );
});

// ... NotificationPanel remains the same
```

### Verification

Clear your console and type in the filter input again.

**Expected vs. Actual Improvement**:
- **Expected**: The `console.log` for `NotificationItem` should not fire when typing in the filter, because the props for each item (`notification` and `onDismiss`) are not changing.
- **Actual**: The console is silent! The items no longer re-render on every keystroke. The Profiler confirms this, showing the `NotificationItem` components as grayed out ("did not render").

**Limitation Preview**: `React.memo` performs a shallow comparison of props. If you pass complex objects or functions that are re-created on every render (like an inline arrow function `onClick={() => {}}`), memoization will fail. This is a deeper topic we will explore fully in Chapter 10: Performance Optimization. For now, know that `memo` is your first tool for preventing unnecessary list item re-renders.

## Synthesis - The Complete Journey

## The Journey: From Problem to Solution

We've taken a simple component and progressively made it more robust, correct, and performant by encountering and solving a series of common problems.

| Iteration | Failure Mode                               | Technique Applied          | Result                                       |
| --------- | ------------------------------------------ | -------------------------- | -------------------------------------------- |
| 0         | Missing `key` prop                         | None                       | Console warning, inefficient updates         |
| 1         | Using `index` as key                       | `key={index}`              | State corruption on item deletion/reorder    |
| 2         | Incorrect UI state on data changes         | `key={notification.id}`    | Correct, stable rendering and state handling |
| 3         | No handling for empty/loading/error states | Conditional Rendering      | Robust UI that handles all data states       |
| 4         | Unnecessary re-renders of list items       | `React.memo`               | Optimized performance on parent state change |

### Final Implementation

Here is the complete, production-ready version of our `NotificationPanel` that incorporates all the lessons learned.

```tsx
// src/components/NotificationPanel.tsx - Final Version

import React, { useState, useEffect } from 'react';

// --- Types and Initial Data ---
type Notification = {
  id: string;
  type: 'info' | 'warning' | 'error';
  message: string;
  read: boolean;
};

const initialNotifications: Notification[] = [
  { id: 'notif-1', type: 'info', message: 'Your profile has been updated.', read: false },
  { id: 'notif-2', type: 'warning', message: 'Your subscription is expiring soon.', read: false },
  { id: 'notif-3', type: 'error', message: 'Failed to upload document.', read: true },
];

// --- Child Components ---
function UnreadIndicator() {
  return <span className="unread-dot" aria-label="unread"></span>;
}

const NotificationItem = React.memo(function NotificationItem({
  notification,
  onDismiss,
}: {
  notification: Notification;
  onDismiss: (id: string) => void;
}) {
  // console.log(`Rendering NotificationItem: ${notification.id}`); // For debugging
  return (
    <div className={`notification notification--${notification.type} ${notification.read ? 'is-read' : ''}`}>
      {!notification.read && <UnreadIndicator />}
      <span className="message">{notification.message}</span>
      <input type="text" placeholder="Add a note..." />
      <button onClick={() => onDismiss(notification.id)}>Dismiss</button>
    </div>
  );
});

// --- Main Panel Component ---
export function NotificationPanel() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filterText, setFilterText] = useState('');

  useEffect(() => {
    const fetchNotifications = async () => {
      try {
        await new Promise(resolve => setTimeout(resolve, 1500));
        setNotifications(initialNotifications);
      } catch (err) {
        setError('Could not fetch notifications.');
      } finally {
        setIsLoading(false);
      }
    };
    fetchNotifications();
  }, []);

  const handleDismiss = (idToDismiss: string) => {
    setNotifications(current => current.filter(n => n.id !== idToDismiss));
  };

  if (isLoading) {
    return <div className="panel-loading">Loading notifications...</div>;
  }

  if (error) {
    return <div className="panel-error">Error: {error}</div>;
  }

  const filteredNotifications = notifications.filter(n =>
    n.message.toLowerCase().includes(filterText.toLowerCase())
  );

  return (
    <div className="notification-panel">
      <h2>Notifications ({filteredNotifications.length})</h2>
      
      <input
        type="text"
        placeholder="Filter notifications..."
        value={filterText}
        onChange={(e) => setFilterText(e.target.value)}
        className="filter-input"
      />

      <div className="notification-list">
        {filteredNotifications.length > 0 ? (
          filteredNotifications.map((notification) => (
            <NotificationItem
              key={notification.id}
              notification={notification}
              onDismiss={handleDismiss}
            />
          ))
        ) : (
          <p className="empty-state">No matching notifications.</p>
        )}
      </div>
    </div>
  );
}
```

### Lessons Learned

1.  **Keys are for Identity, Not Position**: The `key` prop is React's way of tracking an element's identity across renders. Always use a stable, unique ID from your data (`item.id`), and never use the array index (`index`) for dynamic lists.
2.  **Render Every State**: A component isn't complete until it can correctly render all its possible states: loading, error, empty, and data-filled. Use conditional rendering patterns to handle each case explicitly.
3.  **Default Re-renders Can Be Costly**: Be aware that parent state changes will cause child components in a list to re-render by default. Use `React.memo` as a first-line defense to prevent this for components that receive stable props.
