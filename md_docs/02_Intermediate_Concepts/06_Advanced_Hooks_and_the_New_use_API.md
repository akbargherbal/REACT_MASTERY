# Chapter 6: Advanced Hooks and the New `use` API

## The Revolutionary `use` Hook

## Learning Objective

Understand the purpose of the new `use` hook and its unique ability to be called conditionally, breaking one of the original rules of hooks.

## Why This Matters

The `use` hook is the most significant addition to React's hook system since its introduction. It simplifies reading data from resources like Promises and Context, and its conditional nature unlocks more flexible and efficient component patterns that were previously impossible.

## Discovery Phase

In all the hooks we've seen so far—`useState`, `useEffect`, `useContext`—we've followed a strict rule: **Hooks must be called at the top level of your component.** They cannot be inside loops, conditions, or nested functions.

React 19 introduces a new, special hook that changes this rule: the `use` hook.

The `use` hook is a unified API for a single purpose: **reading a value from a resource**. A "resource" can be a Promise (which represents future data) or Context (which represents shared data). The key innovation is that `use` can be called inside conditions and loops.

Let's see a conceptual example to grasp the difference.

Here's what you **cannot** do with traditional hooks:

```jsx
import React, { useState, useContext } from "react";

const MyContext = React.createContext("default");

// THIS CODE WILL CAUSE A REACT ERROR
function InvalidConditionalHook({ shouldGetData }) {
  if (shouldGetData) {
    // Error: React Hook "useState" is called conditionally.
    const [data, setData] = useState(null);

    // Error: React Hook "useContext" is called conditionally.
    const theme = useContext(MyContext);
  }

  return <div>...</div>;
}
```

React enforces this "top-level" rule to ensure that hooks are called in the same order on every render, which is how React keeps track of state between renders.

Now, let's see how the `use` hook is different. It's designed specifically for these conditional scenarios.

```jsx
import React, { use } from 'react';

const ThemeContext = React.createContext('light');
const UserContext = React.createContext({ name: 'Guest' });

function ConditionalComponent({ useTheme, useUser }) {
let message = 'Welcome';

// ✅ CORRECT: `use` can be called conditionally
if (useTheme) {
const theme = use(ThemeContext);
message += `, enjoying the ${theme} theme`;
}

// ✅ CORRECT: Another conditional call
if (useUser) {
const user = use(UserContext);
message += ` ${user.name}`;
}

return <p>{message}!</p>;
}

export default function App() {
return (
<div>
<ThemeContext.Provider value="dark">
<UserContext.Provider value={{ name: 'Alice' }}>
<ConditionalComponent useTheme={true} useUser={true} />
<ConditionalComponent useTheme={false} useUser={true} />
<ConditionalComponent useTheme={true} useUser={false} />
</UserContext.Provider>
</ThemeContext.Provider>
);
}
```

**Rendered Output**:

```

Welcome, enjoying the dark theme Alice!
Welcome Alice!
Welcome, enjoying the dark theme!

```

This code works perfectly. The `use` hook allows the component to "opt-in" to reading a resource based on props or state. This was a major limitation of `useContext` and is a huge step forward for component flexibility.

## Deep Dive

The `use` hook's ability to be called conditionally might seem like it breaks the fundamental rules of hooks, but it's designed with a different internal mechanism. Unlike `useState`, which needs to preserve its "slot" on every render, `use` is purely for _reading_ a value that already exists somewhere else (in a context provider or a promise). React can handle this dynamic reading without losing track of component state.

Think of it this way:

- **`useState`, `useEffect`, etc.**: These hooks _create and manage stateful logic_ within a component. Their order must be consistent for React to link the state from one render to the next.
- **`use`**: This hook _reads data from an external source_. It doesn't create its own state in the same way. It's more like a special, React-aware "getter" function that can suspend the component.

This new capability allows for more efficient components. A component can avoid the work of subscribing to and processing context if it doesn't need it for the current render.

### Common Confusion: Is `use` a replacement for all other hooks?

**You might think**: I should replace all my `useState` and `useEffect` calls with `use`.

**Actually**: No. The `use` hook is not a general-purpose replacement for other hooks. It serves a very specific purpose: to read values from resources. You still need `useState` for managing component state, `useEffect` for side effects, and `useReducer` for complex state logic.

**Why the confusion happens**: The name "use" is very generic, and its ability to be called conditionally makes it seem like a "better" hook.

**How to remember**: Think of `use` as `useResource`. Its job is to **use a resource**, like a promise or context. If you aren't reading from a resource, you need a different hook.

## Production Perspective

The introduction of `use` simplifies many patterns and improves performance.

**When professionals will choose this**:

- **Conditional Data Fetching**: Kicking off a data fetch only after a user interaction, right inside the render logic.
- **Dynamic Context Consumption**: A component might only need theme context when in "edit mode". Using `use(ThemeContext)` inside an `if (isEditing)` block is cleaner and more performant than `useContext` at the top level.
- **A/B Testing or Feature Flags**: A component can conditionally read from a feature flag context to alter its behavior, without subscribing to that context if the feature is disabled.

**Trade-offs**:

- ✅ **Advantage**: Drastically simplifies component logic by allowing you to read data where you need it.
- ✅ **Advantage**: Can improve performance by avoiding unnecessary context subscriptions.
- ⚠️ **Cost**: It's a new concept. Teams will need to learn when to use it versus traditional hooks. Overusing it could make component logic harder to follow if conditions become too nested.
- ⚠️ **Cost**: To use `use` with promises, your component architecture needs to incorporate `<Suspense>` boundaries, which we'll cover next.

## Reading Promises with `use`

## Learning Objective

Learn how to use the `use` hook with Promises to handle asynchronous operations, replacing complex `useEffect` and `useState` combinations for data fetching.

## Why This Matters

Data fetching is a core task in any React application. The traditional approach using `useEffect` and `useState` involves significant boilerplate to manage loading, error, and data states. The `use` hook, combined with React Suspense, revolutionizes this by letting you write data fetching logic as if it were synchronous.

## Discovery Phase: The Old Way

Let's start with a concrete example of fetching user data the "classic" way. We need at least three state variables: one for the data, one for the loading status, and one for any potential errors.

```jsx
import React, { useState, useEffect } from "react";

// A mock API call that returns a promise
const fetchUserProfile = (userId) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (userId === 1) {
        resolve({ id: 1, name: "Alice", email: "alice@example.com" });
      } else {
        reject(new Error("User not found"));
      }
    }, 1000);
  });
};

function UserProfileClassic({ userId }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Reset state for new fetch
    setIsLoading(true);
    setError(null);
    setUser(null);

    fetchUserProfile(userId)
      .then((data) => {
        setUser(data);
      })
      .catch((err) => {
        setError(err);
      })
      .finally(() => {
        setIsLoading(false);
      });
  }, [userId]); // Re-run effect if userId changes

  if (isLoading) {
    return <p>Loading profile...</p>;
  }

  if (error) {
    return <p>Error: {error.message}</p>;
  }

  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}

export default function App() {
  return <UserProfileClassic userId={1} />;
}
```

**Interactive Behavior**:

1. Initially, "Loading profile..." is displayed for 1 second.
2. Then, the user's profile is rendered: "Alice", "Email: alice@example.com".
3. If you were to change the `userId` prop to `2`, it would show "Loading profile..." again, then "Error: User not found".

This pattern works, but it's verbose. We're manually managing three different states and their transitions. This complexity grows with every new piece of data you need to fetch.

## Deep Dive: The New Way with `use` and Suspense

Now, let's refactor this using the `use` hook. The `use` hook can be passed a promise.

- If the promise is **pending**, React will **suspend** the rendering of the component.
- If the promise **resolves**, `use` will return the resolved **value**.
- If the promise **rejects**, `use` will **throw** the error.

This suspension and error throwing are caught by two special components: `<Suspense>` and **Error Boundaries**. For now, we'll focus on Suspense.

Let's see the code. It's shockingly simple.

```jsx
import React, { use, Suspense } from "react";

// A mock API call that returns a promise
const fetchUserProfile = (userId) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (userId === 1) {
        resolve({ id: 1, name: "Alice", email: "alice@example.com" });
      } else {
        reject(new Error("User not found"));
      }
    }, 1000);
  });
};

// NOTE: For `use` to work with promises, we need a way to cache the promise
// so it's not re-created on every render. A simple cache can be a Map.
const promiseCache = new Map();

function getProfile(userId) {
  if (!promiseCache.has(userId)) {
    promiseCache.set(userId, fetchUserProfile(userId));
  }
  return promiseCache.get(userId);
}

function UserProfile({ userId }) {
  // 1. Get the promise (from a cache or library)
  const userPromise = getProfile(userId);

  // 2. Pass the promise to `use`. React handles the rest.
  const user = use(userPromise);

  // 3. Render the success state. The loading and error states are handled by React.
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}

export default function App() {
  return (
    <div>
      <h2>User Profile</h2>
      <Suspense fallback={<p>Loading profile...</p>}>
        <UserProfile userId={1} />
      </Suspense>
    </div>
  );
}
```

**Interactive Behavior**:
The visual behavior is identical to the classic version.

1. "Loading profile..." is displayed for 1 second. (This is the `fallback` from `<Suspense>`).
2. The `UserProfile` component renders its UI.

### Component Trace

Let's trace what happens when `<UserProfile userId={1} />` renders for the first time:

1. `UserProfile` is called.
2. `getProfile(1)` is called. It creates a new promise to fetch the user data, stores it in the cache, and returns it.
3. `const user = use(userPromise);` is executed.
4. The promise is **pending**. React sees this and **suspends** `UserProfile`. It stops rendering this component.
5. React looks up the component tree for the nearest `<Suspense>` boundary.
6. It finds `<Suspense fallback={<p>Loading profile...</p>}>` and renders its `fallback` UI.
7. After 1 second, the promise **resolves**.
8. React "wakes up" and tries to render `UserProfile` again from where it left off.
9. `const user = use(userPromise);` is executed again. This time, the promise is already resolved.
10. `use` returns the resolved value: `{ id: 1, name: 'Alice', ... }`.
11. The component continues rendering and returns the JSX with the user's name and email.
12. React replaces the "Loading profile..." fallback with the fully rendered `UserProfile` component.

Notice we didn't write `if (isLoading)` or `if (error)`. React handles these states declaratively through `<Suspense>` and Error Boundaries (Chapter 18). The component code is only concerned with the "happy path"—what to render when the data is available.

### Common Confusion: Where did my data fetching library go?

**You might think**: `use(fetch(...))` is all I need, and I can forget about libraries like React Query or SWR.

**Actually**: The `use` hook is a low-level primitive. It doesn't handle caching, request deduplication, or re-fetching. In the example above, we created a very simple `promiseCache` to prevent re-fetching on every render. Production-grade data fetching libraries will integrate with `use` and Suspense to provide these features for you.

**How to remember**: `use` gives you the ability to suspend. Data fetching libraries give you the policies (caching, revalidation) for _when_ and _how_ to fetch. You will likely use a library that uses `use` under the hood.

## Production Perspective

Using `use` with Suspense is the future of data fetching in React.

**When professionals choose this**:

- For nearly all data fetching in new React 19+ applications.
- To create smoother loading experiences where different parts of the page can load independently. You can nest multiple `<Suspense>` boundaries.
- To simplify component logic and remove vast amounts of state management boilerplate.

**Trade-offs**:

- ✅ **Advantage**: Massively simplified component code. Focus on the success case.
- ✅ **Advantage**: Better UX. Suspense avoids jarring whole-page spinners and allows for granular loading states.
- ✅ **Advantage**: Eliminates entire classes of bugs, like race conditions, that are common with `useEffect` fetching.
- ⚠️ **Cost**: Requires a mental model shift. You need to think in terms of Suspense boundaries rather than `isLoading` flags.
- ⚠️ **Cost**: Requires a caching strategy for promises. You can't just call `use(fetch(...))` in a loop or on every render. This is why data fetching libraries that support Suspense are still essential.

## Conditional Context Consumption with `use`

## Learning Objective

Demonstrate how the `use` hook enables conditional and dynamic consumption of React Context, overcoming a key limitation of `useContext`.

## Why This Matters

Components often need to adapt their behavior based on props or internal state. Previously, if a component _might_ need a value from context, it had to subscribe using `useContext` on every single render, even when the value wasn't used. The `use` hook solves this, allowing components to be more efficient and logically cleaner.

## Discovery Phase: The `useContext` Limitation

Let's set up a scenario. We have a `SettingsPanel` component. If the user is an admin (indicated by an `isAdmin` prop), we want to show a special theme setting read from an `AdminThemeContext`.

Here's how we might try to build it with `useContext`.

```jsx
import React, { useContext } from "react";

const AdminThemeContext = React.createContext({ color: "red" });

function SettingsPanel({ isAdmin }) {
  // We MUST call useContext at the top level, unconditionally.
  const adminTheme = useContext(AdminThemeContext);

  return (
    <div>
      <h3>User Settings</h3>
      <p>Standard setting 1</p>
      <p>Standard setting 2</p>

      {/* We only USE the value down here, but we subscribed above. */}
      {isAdmin && (
        <div
          style={{ border: `2px solid ${adminTheme.color}`, padding: "10px" }}
        >
          <h4>Admin Controls</h4>
          <p>Theme Color: {adminTheme.color}</p>
        </div>
      )}
    </div>
  );
}

export default function App() {
  return (
    <AdminThemeContext.Provider value={{ color: "blue" }}>
      <SettingsPanel isAdmin={true} />
      <hr />
      <SettingsPanel isAdmin={false} />
    </AdminThemeContext.Provider>
  );
}
```

**Rendered Output**:

```
User Settings
Standard setting 1
Standard setting 2

Admin Controls
Theme Color: blue

---
User Settings
Standard setting 1
Standard setting 2
```

This works, but look closely at the `SettingsPanel` component when `isAdmin` is `false`. It still calls `useContext(AdminThemeContext)`. This means that even the non-admin panel is subscribed to the `AdminThemeContext`. If the context value were to change, this component would re-render unnecessarily. For a small app, this is trivial. For a large app with many components and contexts, this can lead to performance issues.

## Deep Dive: Conditional Reading with `use`

Now, let's refactor `SettingsPanel` using the `use` hook. Since `use` can be called inside an `if` statement, we can make the context subscription itself conditional.

```jsx
import React, { use } from "react";

const AdminThemeContext = React.createContext({ color: "red" });

function SettingsPanel({ isAdmin }) {
  let adminTheme = null;

  // ✅ We only read from the context IF we are an admin.
  if (isAdmin) {
    adminTheme = use(AdminThemeContext);
  }

  return (
    <div>
      <h3>User Settings</h3>
      <p>Standard setting 1</p>
      <p>Standard setting 2</p>

      {isAdmin && (
        <div
          style={{ border: `2px solid ${adminTheme.color}`, padding: "10px" }}
        >
          <h4>Admin Controls</h4>
          <p>Theme Color: {adminTheme.color}</p>
        </div>
      )}
    </div>
  );
}

export default function App() {
  return (
    <AdminThemeContext.Provider value={{ color: "blue" }}>
      <SettingsPanel isAdmin={true} />
      <hr />
      <SettingsPanel isAdmin={false} />
    </AdminThemeContext.Provider>
  );
}
```

The rendered output is identical, but the behavior is fundamentally better.

### Component Trace

Let's trace the two scenarios for the new `SettingsPanel`:

**Scenario 1: `isAdmin={true}`**

1. The `SettingsPanel` component renders.
2. The `if (isAdmin)` condition is `true`.
3. `adminTheme = use(AdminThemeContext);` is executed.
4. The hook reads the value `{ color: 'blue' }` from the context provider.
5. The component subscribes to changes in this context.
6. The admin controls are rendered using the theme color.

**Scenario 2: `isAdmin={false}`**

1. The `SettingsPanel` component renders.
2. The `if (isAdmin)` condition is `false`.
3. The `use(AdminThemeContext)` line is **never executed**.
4. The component **does not subscribe** to the `AdminThemeContext`.
5. The admin controls are not rendered.
6. If the `AdminThemeContext` value changes, this component will **not** re-render.

This is a powerful optimization. The component only pays the cost (subscription and potential re-renders) for the context it actually needs for a given render.

### Common Confusion: Should I always use `use(Context)` instead of `useContext(Context)`?

**You might think**: `use(Context)` is strictly better, so I should replace all my `useContext` calls.

**Actually**: Not necessarily. If your component _always_ needs the value from a context, `useContext(Context)` at the top level is perfectly clear and idiomatic. It signals to anyone reading the code that this component has a firm dependency on that context.

**How to remember**:

- Use `useContext(Context)` when the dependency is **unconditional**.
- Use `use(Context)` when the dependency is **conditional** (inside an `if`, a `switch`, or an early `return`).

## Production Perspective

Conditional context consumption is a significant architectural improvement for large-scale React applications.

**When professionals choose this**:

- **Feature Flags**: A component can `use(FeatureFlagContext)` inside a conditional block to enable or disable features, avoiding subscriptions for users who don't have the flag enabled.
- **Complex UI States**: In a component with multiple modes (e.g., "view", "edit", "comment"), it can conditionally `use` different contexts relevant to each mode.
- **Performance Optimization**: In complex dashboards, components can opt-out of subscribing to high-frequency update contexts if they are not currently visible or relevant, preventing cascading re-renders.

**Real-world example**: Imagine a component in a design tool like Figma. When you select an element, the component might need to `use(SelectedElementContext)`. When no element is selected, it has no need for that context. Using `use` inside `if (isSelected)` is the perfect pattern for this. It makes the component more self-contained and efficient.

## useContext for Global State

## Learning Objective

Understand how to use the `useContext` hook to consume shared data from a React Context, solving the "prop drilling" problem.

## Why This Matters

While `use` provides a new way to consume context, `useContext` remains a fundamental hook. Understanding it is crucial because it establishes the provider/consumer pattern that `use` builds upon. It's the primary tool for making data like themes, user information, or language preferences available throughout your application without passing props through many layers.

## Discovery Phase: The Problem of Prop Drilling

Imagine a simple app that needs to display the current user's name in a deeply nested component. Without Context, you have to pass the `user` object down as a prop through every single intermediate component. This is called "prop drilling."

```jsx
import React from "react";

// App has the user data
function App() {
  const user = { name: "Barbara" };
  return <Page user={user} />;
}

// Page doesn't need user, but has to pass it down
function Page({ user }) {
  return <Layout user={user} />;
}

// Layout doesn't need user, but has to pass it down
function Layout({ user }) {
  return <Header user={user} />;
}

// Header is the component that finally needs the data
function Header({ user }) {
  return <h1>Welcome, {user.name}!</h1>;
}

export default App;
```

**Rendered Output**:

```

Welcome, Barbara!

```

This works, but it's brittle and verbose. `Page` and `Layout` are cluttered with a prop they don't care about. If another component deep in the tree needed `user`, we'd have to thread it through even more components.

## Deep Dive: Solving Prop Drilling with Context

React's Context API provides a way to solve this. It creates a sort of "teleporter" for data.

The process has three steps:

1.  **Create** the context using `React.createContext()`.
2.  **Provide** a value to the context using a `<MyContext.Provider>` component, wrapping a part of your component tree.
3.  **Consume** the value in any child component using the `useContext` hook.

Let's refactor our example.

```jsx
import React, { createContext, useContext } from "react";

// 1. Create the context. We can provide a default value.
const UserContext = createContext({ name: "Guest" });

// App provides the user data
function App() {
  const user = { name: "Barbara" };
  // 2. Provide the value to the entire component tree
  return (
    <UserContext.Provider value={user}>
      <Page />
    </UserContext.Provider>
  );
}

// Page no longer needs to know about `user`
function Page() {
  return <Layout />;
}

// Layout no longer needs to know about `user`
function Layout() {
  return <Header />;
}

// Header can now directly access the data
function Header() {
  // 3. Consume the value from the nearest provider
  const user = useContext(UserContext);
  return <h1>Welcome, {user.name}!</h1>;
}

export default App;
```

**Rendered Output**:

```

Welcome, Barbara!

```

The result is the same, but the code is much cleaner. `Page` and `Layout` are simplified because they no longer act as intermediaries. `Header` is now self-sufficient; it explicitly declares its dependency on `UserContext` and gets the data directly.

### Component Trace

1.  `App` renders and wraps its children in `<UserContext.Provider value={{ name: 'Barbara' }}>`. This makes the `user` object available to any component inside it.
2.  `Page` and `Layout` render. They don't know or care about `UserContext`.
3.  `Header` renders. It calls `useContext(UserContext)`.
4.  React looks up the component tree from `Header`.
5.  It finds the `<UserContext.Provider>` in `App` and its `value` prop.
6.  `useContext` returns that value (`{ name: 'Barbara' }`).
7.  The `Header` component renders the welcome message.
8.  Crucially, if the `value` in the provider changes, React will automatically re-render any components that consume that context (like `Header`).

### Common Confusion: Does Context replace Redux or other state management libraries?

**You might think**: Context provides global state, so I don't need libraries like Redux or Zustand anymore.

**Actually**: Context is a mechanism for dependency injection, not a complete state management solution. It's excellent for low-frequency updates of global data (theme, user auth, language). However, for high-frequency updates (like state in a complex editor), it can lead to performance issues, as every component consuming the context will re-render on every change.

**How to remember**:

- **Context**: Best for making static or infrequently changing global data available. Think "read-mostly".
- **State Management Libraries (Redux, Zustand)**: Best for managing complex, frequently updated application state. They offer performance optimizations (like selectors), middleware for side effects, and better developer tools.

## Production Perspective

Context is a foundational tool in any production React application.

**When professionals choose this**:

- **Theming**: Providing a theme object (colors, fonts) to the entire app.
- **Authentication**: Sharing the current user's status and profile across all components.
- **Internationalization (i18n)**: Making the current language and translation functions available.

**Best Practices**:

- **Keep contexts focused**: Don't create one giant "AppContext" with everything. Create separate contexts for separate concerns (`ThemeContext`, `AuthContext`). This prevents components from re-rendering due to unrelated state changes.
- **Memoize provider values**: If the `value` passed to a provider is an object or array created inline (`<MyContext.Provider value={{ theme: 'dark' }}>`), it will be a new object on every render, causing all consumers to re-render. You should memoize this value with `useMemo` or `useState` to prevent unnecessary re-renders (we'll cover `useMemo` shortly).

## useReducer for Complex State Logic

## Learning Objective

Learn how to manage complex component state using the `useReducer` hook, making state transitions more predictable and maintainable.

## Why This Matters

As your components grow, managing state with multiple `useState` calls can become messy and error-prone. `useReducer` provides a more structured approach, inspired by the Redux pattern, for handling state logic. It's perfect for when the next state depends on the previous one, or when you have several related pieces of state to manage together.

## Discovery Phase: The `useState` Spaghetti

Let's build a simple shopping cart quantity selector. It needs to increment, decrement, and allow the user to set a specific quantity. Using `useState`, the logic might look like this.

```jsx
import React, { useState } from "react";

function QuantitySelector() {
  const [quantity, setQuantity] = useState(1);
  const [error, setError] = useState(null);

  const handleIncrement = () => {
    if (quantity < 10) {
      setQuantity((q) => q + 1);
      setError(null);
    } else {
      setError("Cannot add more than 10 items.");
    }
  };

  const handleDecrement = () => {
    if (quantity > 1) {
      setQuantity((q) => q - 1);
      setError(null);
    } else {
      setError("Quantity cannot be less than 1.");
    }
  };

  const handleSetQuantity = (e) => {
    const value = parseInt(e.target.value, 10);
    if (value >= 1 && value <= 10) {
      setQuantity(value);
      setError(null);
    } else {
      setQuantity(1); // Reset on invalid input
      setError("Please enter a number between 1 and 10.");
    }
  };

  return (
    <div>
      <button onClick={handleDecrement}>-</button>
      <input type="number" value={quantity} onChange={handleSetQuantity} />
      <button onClick={handleIncrement}>+</button>
      {error && <p style={{ color: "red" }}>{error}</p>}
    </div>
  );
}

export default QuantitySelector;
```

**Interactive Behavior**:

- Clicking "+" increases the number.
- Clicking "-" decreases the number.
- Typing in the input sets the number.
- An error message appears if you go outside the 1-10 range.

This works, but notice how the state logic is scattered across three different event handlers. The business rules for `quantity` and `error` are intertwined and duplicated. If we added another piece of related state, like a "last action" tracker, it would get even messier.

## Deep Dive: Centralizing Logic with `useReducer`

`useReducer` lets us consolidate all this state transition logic into a single function called a **reducer**.

A reducer is a pure function that takes two arguments: the current `state` and an `action` object. It computes and returns the _next_ state.

The hook `useReducer` returns the current `state` and a `dispatch` function. Instead of calling `setState`, you `dispatch` an action object.

Let's refactor our `QuantitySelector`.

```jsx
import React, { useReducer } from "react";

const initialState = {
  quantity: 1,
  error: null,
  max: 10,
  min: 1,
};

function quantityReducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      if (state.quantity < state.max) {
        return { ...state, quantity: state.quantity + 1, error: null };
      }
      return { ...state, error: `Cannot add more than ${state.max} items.` };

    case "DECREMENT":
      if (state.quantity > state.min) {
        return { ...state, quantity: state.quantity - 1, error: null };
      }
      return { ...state, error: `Quantity cannot be less than ${state.min}.` };

    case "SET_QUANTITY":
      const value = action.payload;
      if (value >= state.min && value <= state.max) {
        return { ...state, quantity: value, error: null };
      }
      return {
        ...state,
        quantity: state.min,
        error: `Please enter a number between ${state.min} and ${state.max}.`,
      };

    default:
      throw new Error("Unknown action type");
  }
}

function QuantitySelector() {
  const [state, dispatch] = useReducer(quantityReducer, initialState);

  return (
    <div>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>-</button>
      <input
        type="number"
        value={state.quantity}
        onChange={(e) =>
          dispatch({
            type: "SET_QUANTITY",
            payload: parseInt(e.target.value, 10),
          })
        }
      />
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
      {state.error && <p style={{ color: "red" }}>{state.error}</p>}
    </div>
  );
}

export default QuantitySelector;
```

### Component Trace

1.  **Initialization**: `useReducer(quantityReducer, initialState)` is called. React stores `initialState` and gives us back the current state (`state`) and the `dispatch` function.
2.  **User Interaction**: The user clicks the "+" button.
3.  **Dispatch**: The `onClick` handler calls `dispatch({ type: 'INCREMENT' })`.
4.  **Reducer Execution**: React calls our `quantityReducer` with the current `state` and the `action` we dispatched.
5.  **State Calculation**: Inside the reducer, the `switch` statement matches the `'INCREMENT'` type. It calculates the new state object based on the current state and returns it.
6.  **Re-render**: React receives the new state object from the reducer. It detects a change and re-renders the `QuantitySelector` component. The component now receives the updated `state` from the `useReducer` hook, and the UI reflects the new quantity.

The component's responsibility is now much clearer: it dispatches actions describing _what happened_. The reducer's responsibility is to handle the state logic, describing _how the state changes_ in response. This separation makes both parts easier to understand, test, and maintain.

### Common Confusion: Is `useReducer` just for Redux users?

**You might think**: `useReducer` is a mini-Redux, and I only need it if I'm building a huge application.

**Actually**: `useReducer` is a built-in React hook that is useful for any component with moderately complex state, regardless of whether you use Redux. It's a natural step up from `useState`. In fact, `useState` can be implemented using `useReducer` under the hood.

**How to remember**:

- `useState`: Great for simple, independent state (a boolean toggle, a single input field).
- `useReducer`: Great for complex, related state (multiple form fields with validation, state machines). Choose it when the next state depends on the previous state in a non-trivial way.

## Production Perspective

Knowing when to switch from `useState` to `useReducer` is a sign of a maturing React developer.

**When professionals choose `useReducer`**:

- **Complex Forms**: When state updates for one field can affect others (e.g., selecting a country populates a list of states).
- **State Machines**: When a component can be in one of several distinct states (e.g., `idle`, `loading`, `success`, `error`) and transitions between them are well-defined.
- **Decoupling Logic**: When the state logic is complex enough that you want to test it separately from the UI component. You can export the reducer and test it as a pure function.
- **Improving Performance**: When updates are triggered from deep within a component tree, passing `dispatch` down is often more efficient than passing callback functions, as `dispatch` has a stable identity.

## useCallback for Performance (Pre-Compiler)

## Learning Objective

Understand the purpose of `useCallback` for memoizing functions, and recognize its role in optimizing performance before the introduction of the React 19 Compiler.

## Why This Matters

In React, re-rendering is normal. However, unnecessary re-renders of child components can slow down your application. `useCallback` is a tool to prevent this by ensuring that functions passed as props don't change on every render, which is crucial when working with memoized child components.

### Legacy Pattern Notice

**Pre-React 19**: `useCallback` was a manual optimization that developers had to apply frequently to prevent performance issues, especially when passing callbacks to components wrapped in `React.memo`.

**React 19+**: The new React Compiler aims to make `useCallback` and `useMemo` obsolete by automatically memoizing components and hooks. This section teaches the concept because it's vital for understanding how React's rendering works and for maintaining older codebases. In new projects with the compiler enabled, you will rarely need to write this by hand.

## Discovery Phase: The Problem of Re-created Functions

Let's create a scenario with a parent component that has a counter, and a child component that has a button to increment that counter. We will wrap the child component in `React.memo` to prevent it from re-rendering if its props don't change.

```jsx
import React, { useState } from "react";

// React.memo prevents a re-render if props are shallowly equal
const IncrementButton = React.memo(({ onIncrement }) => {
  console.log("IncrementButton is rendering!");
  return <button onClick={onIncrement}>Increment</button>;
});

function ParentComponent() {
  const [count, setCount] = useState(0);
  const [otherState, setOtherState] = useState(false);

  // This function is re-created on every render of ParentComponent
  const handleIncrement = () => {
    setCount((c) => c + 1);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <IncrementButton onIncrement={handleIncrement} />
      <hr />
      <button onClick={() => setOtherState(!otherState)}>
        Toggle Other State
      </button>
      <p>Other State: {String(otherState)}</p>
    </div>
  );
}

export default ParentComponent;
```

**Interactive Behavior**:

1.  Open your developer console. You will see "IncrementButton is rendering!".
2.  Click the "Increment" button. The count increases, and you see "IncrementButton is rendering!" again in the console. This is expected.
3.  Now, click the "Toggle Other State" button. The `otherState` value flips, but notice the console: "IncrementButton is rendering!" appears again!

This is the problem. Even though `IncrementButton` has nothing to do with `otherState`, it re-renders. We wrapped it in `React.memo`, so why is this happening?

Because on every render of `ParentComponent`, `const handleIncrement = () => { ... }` is a **brand new function**. When React compares the props for `IncrementButton`, it sees that the `onIncrement` prop from the last render is not strictly equal (`===`) to the `onIncrement` prop from the current render. So, `React.memo` gives up, and the component re-renders.

## Deep Dive: Memoizing the Callback

`useCallback` solves this by giving us the _same function instance_ between renders, as long as its dependencies haven't changed.

`const memoizedCallback = useCallback(callbackFunction, [dependencies]);`

Let's apply it to our example.

```jsx
import React, { useState, useCallback } from "react";

const IncrementButton = React.memo(({ onIncrement }) => {
  console.log("IncrementButton is rendering!");
  return <button onClick={onIncrement}>Increment</button>;
});

function ParentComponent() {
  const [count, setCount] = useState(0);
  const [otherState, setOtherState] = useState(false);

  // `useCallback` returns a memoized version of the callback.
  // The dependency array `[]` is empty, so the same function is returned on every render.
  const handleIncrement = useCallback(() => {
    setCount((c) => c + 1);
  }, []); // Empty dependency array means the function never needs to be recreated.

  return (
    <div>
      <p>Count: {count}</p>
      <IncrementButton onIncrement={handleIncrement} />
      <hr />
      <button onClick={() => setOtherState(!otherState)}>
        Toggle Other State
      </button>
      <p>Other State: {String(otherState)}</p>
    </div>
  );
}

export default ParentComponent;
```

**Interactive Behavior**:

1.  Open your developer console. You will see "IncrementButton is rendering!".
2.  Click "Increment". The count increases, and "IncrementButton is rendering!" appears. This is still expected because the parent's state changed, causing it to re-render.
3.  Click "Toggle Other State". The `otherState` flips, but this time **nothing is logged to the console**.

Success! Because `handleIncrement` is now a stable function reference thanks to `useCallback`, `React.memo` sees that the `onIncrement` prop has not changed, and it correctly skips re-rendering the `IncrementButton`.

### The Dependency Array

The second argument to `useCallback` is crucial. It's an array of values that the callback depends on. If any of these values change between renders, `useCallback` will discard the old memoized function and create a new one with the updated values.

For example, if our callback needed to use a prop:
`const memoizedFn = useCallback(() => { console.log(someProp); }, [someProp]);`
This tells React: "Only create a new version of this function if `someProp` changes."

### Common Confusion: Should I wrap every function in `useCallback`?

**You might think**: `useCallback` is for performance, so I should use it on every function in my component.

**Actually**: No. This is a common anti-pattern. `useCallback` itself has a small cost. It's only beneficial when you are passing the function as a prop to an optimized child component (`React.memo`), or when you're using it as a dependency in another hook like `useEffect`. Applying it everywhere adds unnecessary complexity for zero gain.

**How to remember**: Use `useCallback` as a targeted optimization tool, not a default practice. Ask yourself: "Am I passing this function to a `React.memo` component?" If the answer is yes, `useCallback` is likely appropriate.

## Production Perspective

Before the React Compiler, `useCallback` was a key tool in the performance optimization toolkit.

**When professionals used this**:

- To preserve referential equality for callbacks passed to components wrapped in `React.memo`.
- To prevent effects in `useEffect` from re-running unnecessarily when they depend on a function.
- In custom hooks that return functions, to ensure the returned functions are stable for consumers of the hook.

**Trade-offs**:

- ✅ **Advantage**: Prevents unnecessary re-renders of memoized children, which can be a significant performance win in complex UIs.
- ⚠️ **Cost**: Adds complexity to the code. It's another thing to reason about.
- ⚠️ **Cost**: Incorrect dependency arrays are a common source of bugs (e.g., the callback captures a "stale" value because a dependency was missed). The React ESLint plugin is essential for catching these errors.

With the React 19 Compiler, the need for manual `useCallback` is greatly diminished, but understanding the concept of referential equality and its impact on rendering remains essential for any serious React developer.

## useMemo for Expensive Calculations (Pre-Compiler)

## Learning Objective

Understand how `useMemo` can memoize the result of an expensive calculation to avoid re-computing it on every render.

## Why This Matters

Just as re-creating functions can be problematic, re-running expensive calculations on every render can make your UI feel sluggish. `useMemo` is a hook that lets you cache the result of a calculation and only re-run it when its dependencies change.

### Legacy Pattern Notice

**Pre-React 19**: `useMemo` was the standard tool for optimizing computationally intensive operations within a component's render cycle.

**React 19+**: The React Compiler is designed to handle this automatically. It can analyze your code and memoize expensive calculations without you needing to manually wrap them in `useMemo`. Learning `useMemo` is still important for understanding the underlying performance concepts and for working on projects without the compiler.

## Discovery Phase: The Expensive Calculation Problem

Let's imagine we have a component that displays a list of numbers and allows the user to filter for only the prime numbers. Finding prime numbers can be computationally expensive.

```jsx
import React, { useState } from 'react';

// An intentionally slow function to check for primality
const isPrime = (num) => {
if (num <= 1) return false;
for (let i = 2; i <= Math.sqrt(num); i++) {
if (num % i === 0) return false;
}
return true;
};

const numberList = Array.from({ length: 10000 }, (\_, i) => i + 1);

function PrimeNumberDisplay() {
const [filter, setFilter] = useState('');
const [time, setTime] = useState(new Date());

// Update time every second to trigger re-renders
setTimeout(() => setTime(new Date()), 1000);

console.log('Component is rendering...');

// This expensive calculation runs on EVERY render
const primeNumbers = numberList.filter(isPrime);

const filteredPrimes = primeNumbers.filter(p => p.toString().includes(filter));

return (
<div>
<p>Current Time: {time.toLocaleTimeString()}</p>
<input
type="text"
placeholder="Filter primes..."
value={filter}
onChange={(e) => setFilter(e.target.value)}
/>
<p>Found {filteredPrimes.length} primes.</p>
{/_ We'll just show the first 10 for brevity _/}
<pre>{JSON.stringify(filteredPrimes.slice(0, 10))}</pre>
</div>
);
}

export default PrimeNumberDisplay;
```

**Interactive Behavior**:

1.  The component renders and displays the time and the list of primes.
2.  The time updates every second, causing the component to re-render. Notice the "Component is rendering..." log in the console every second.
3.  Now, try typing in the filter input. You might notice a significant lag. Each keystroke is slow.

Why is it slow? Because every time the component re-renders—whether from the timer updating or you typing in the input—we are re-calculating `numberList.filter(isPrime)`. This is a huge waste of CPU cycles, as the list of prime numbers itself never changes.

## Deep Dive: Memoizing the Result

`useMemo` lets us cache, or "memoize," the result of this expensive calculation.

`const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);`

React will only re-run the function passed to `useMemo` if one of the dependencies (`a` or `b`) has changed since the last render. Otherwise, it returns the cached value.

Let's fix our component.

```jsx
import React, { useState, useMemo } from 'react';

const isPrime = (num) => {
if (num <= 1) return false;
for (let i = 2; i <= Math.sqrt(num); i++) {
if (num % i === 0) return false;
}
return true;
};

const numberList = Array.from({ length: 10000 }, (\_, i) => i + 1);

function PrimeNumberDisplay() {
const [filter, setFilter] = useState('');
const [time, setTime] = useState(new Date());

setTimeout(() => setTime(new Date()), 1000);

console.log('Component is rendering...');

// The expensive calculation is wrapped in useMemo
const primeNumbers = useMemo(() => {
console.log('Calculating prime numbers... (this should be rare)');
return numberList.filter(isPrime);
}, []); // Empty dependency array means this runs only ONCE.

// This part is cheap and can run on every render
const filteredPrimes = primeNumbers.filter(p => p.toString().includes(filter));

return (
<div>
<p>Current Time: {time.toLocaleTimeString()}</p>
<input
type="text"
placeholder="Filter primes..."
value={filter}
onChange={(e) => setFilter(e.target.value)}
/>
<p>Found {filteredPrimes.length} primes.</p>
<pre>{JSON.stringify(filteredPrimes.slice(0, 10))}</pre>
</div>
);
}

export default PrimeNumberDisplay;
```

**Interactive Behavior**:

1.  On the first render, you will see "Calculating prime numbers..." in the console.
2.  The time will update every second, and "Component is rendering..." will log, but "Calculating prime numbers..." will **not** appear again.
3.  Try typing in the filter input. It should be perfectly smooth and responsive.

By wrapping the expensive part in `useMemo` with an empty dependency array `[]`, we've told React to run that calculation only once for the entire lifecycle of the component and then reuse the result on every subsequent render.

### Common Confusion: `useCallback` vs. `useMemo`

**You might think**: These two hooks seem very similar. When do I use which?

**Actually**: They are related but serve different purposes.

- `useMemo` caches the **return value** of a function.
- `useCallback` caches the **function itself**.

**How to remember**:

- `useMemo` for **values** (memoized values). Use it to avoid re-computing expensive data.
- `useCallback` for **functions** (memoized callbacks). Use it to pass stable functions as props to `React.memo` components.

In fact, `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

## Production Perspective

`useMemo` is a powerful tool but should be used judiciously.

**When professionals use `useMemo`**:

- For genuinely expensive, synchronous calculations like filtering, sorting, or mapping large arrays of data.
- To preserve referential equality for objects or arrays passed as props to memoized child components. For example, `const style = useMemo(() => ({ color: 'red' }), [])` ensures the `style` object is the same on every render.
- When profiling has shown that a specific calculation is a performance bottleneck.

**Trade-offs**:

- ✅ **Advantage**: Can dramatically improve performance by preventing expensive work on re-renders.
- ⚠️ **Cost**: Adds complexity. Every `useMemo` is another piece of code to maintain.
- ⚠️ **Cost**: It's not free. It uses memory to store the cached value and adds a small overhead for dependency checking. Don't wrap simple calculations in `useMemo`.
- ⚠️ **Cost**: Just like `useCallback`, an incorrect dependency array can lead to bugs where the component shows stale, outdated data because the calculation didn't re-run when it should have.

## useRef for DOM Access and Persistence

## Learning Objective

Understand the two primary use cases for the `useRef` hook: accessing DOM elements imperatively and creating mutable "instance variables" that persist across renders without causing re-renders.

## Why This Matters

While React's declarative nature (describing UI with state and props) is its main strength, sometimes you need to step outside that model. You might need to directly interact with a DOM element (like focusing an input) or keep track of a value without triggering a re-render. `useRef` is the "escape hatch" that provides this capability.

## Discovery Phase: Use Case 1 - Accessing the DOM

Imagine you want to build a feature where clicking a button immediately focuses a text input. This is an imperative action—you're telling the browser "do this now." You can't achieve this with state alone. This is where `useRef` comes in.

```jsx
import React, { useRef } from "react";

function FocusInput() {
  // 1. Create a ref object
  const inputRef = useRef(null);

  const handleClick = () => {
    // 3. Access the DOM node via the .current property
    if (inputRef.current) {
      inputRef.current.focus();
    }
  };

  return (
    <div>
      {/_ 2. Attach the ref to a DOM element _/}
      <input ref={inputRef} type="text" placeholder="I will be focused" />
      <button onClick={handleClick}>Focus the Input</button>
    </div>
  );
}

export default FocusInput;
```

**Interactive Behavior**:
When you click the "Focus the Input" button, the text input field immediately gains focus, and your cursor appears inside it.

### Component Trace

1.  **`const inputRef = useRef(null)`**: React creates a special object that looks like `{ current: null }`. This object will persist for the full lifetime of the component.
2.  **`<input ref={inputRef} ... />`**: When React renders the `<input>` element to the DOM, it takes the actual DOM node and assigns it to the `current` property of our `inputRef` object. So, after rendering, `inputRef` is `{ current: <the input DOM element> }`.
3.  **`handleClick`**: When the button is clicked, we access `inputRef.current`. This now holds the real DOM input element, which has methods like `.focus()`. We call this method to imperatively focus the input.

## Deep Dive: Use Case 2 - Persistent "Instance Variables"

The second major use of `useRef` is to store a value that you want to persist across renders, but which **should not trigger a re-render when it changes**. This is the key difference between `useRef` and `useState`.

Let's build a simple stopwatch. We need to store the ID of the `setInterval` timer so we can clear it later.

```jsx
import React, { useState, useRef } from "react";

function Stopwatch() {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  // Use a ref to hold the interval ID.
  // If we used useState, setting the ID would cause a re-render.
  const intervalRef = useRef(null);

  const handleStart = () => {
    if (isRunning) return;
    setIsRunning(true);
    intervalRef.current = setInterval(() => {
      setSeconds((s) => s + 1);
    }, 1000);
  };

  const handleStop = () => {
    if (!isRunning) return;
    setIsRunning(false);
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  const handleReset = () => {
    setIsRunning(false);
    clearInterval(intervalRef.current);
    intervalRef.current = null;
    setSeconds(0);
  };

  return (
    <div>
      <p>Time: {seconds}s</p>
      <button onClick={handleStart}>Start</button>
      <button onClick={handleStop}>Stop</button>
      <button onClick={handleReset}>Reset</button>
    </div>
  );
}

export default Stopwatch;
```

**Interactive Behavior**:

- Click "Start": The timer begins counting up every second.
- Click "Stop": The timer pauses.
- Click "Reset": The timer stops and resets to 0.

In this example, `intervalRef.current` is used to store the timer ID returned by `setInterval`. When we assign to `intervalRef.current`, the component does **not** re-render. This is exactly what we want. The timer ID is just an internal piece of bookkeeping; it has no direct visual representation, so changing it shouldn't trigger a UI update. The re-renders are driven only by the `seconds` state.

### Common Confusion: `useRef` vs. `useState`

**You might think**: They both store values, so when should I use one over the other?

**Actually**: The difference is fundamental to how React works.

- **`useState`**: Use for any data that is part of your component's visual output. When this data changes, you _want_ React to re-render.
- **`useRef`**: Use for data that is _not_ part of the visual output. It's for things the component needs to keep track of behind the scenes, like a timer ID, a subscription object, or a previous state value.

**How to remember**: If changing the value should update the screen, use `useState`. If not, use `useRef`.

## Production Perspective

`useRef` is an indispensable tool for bridging the declarative world of React with the imperative world of the DOM and browser APIs.

**When professionals use `useRef`**:

- **Managing Focus, Text Selection, or Media Playback**: For controlling `<input>`, `<textarea>`, `<video>`, and `<audio>` elements.
- **Integrating with Third-Party DOM Libraries**: When using a library that manipulates the DOM directly (like a charting or mapping library), you'll often use a ref to give that library a DOM element to mount itself onto.
- **Storing Previous State/Props**: A common pattern is to store the previous value of a prop inside a ref to compare it with the current value in an effect.
- **Caching expensive object instances**: Storing an instance of a class that should persist for the life of the component.

**Trade-offs**:

- ✅ **Advantage**: Provides a clean and officially supported way to work with the DOM and manage mutable state without re-renders.
- ⚠️ **Cost**: Overusing refs to manipulate the DOM can be an anti-pattern. It's often a sign that you are "fighting the framework." Always try to solve a problem with state and props first before reaching for a ref. Imperative code is generally harder to reason about than declarative code.

## useImperativeHandle

## Learning Objective

Understand how to use `useImperativeHandle` in conjunction with `ref` to expose a custom, limited API from a child component to a parent.

## Why This Matters

Sometimes, a parent component needs to call a method on a child component. This breaks React's standard top-down data flow and is generally discouraged, but it's necessary for certain use cases, like controlling a video player or a custom form input. `useImperativeHandle` is the hook that allows you to do this in a controlled way, exposing only the specific functions you want, rather than the entire component instance.

### Legacy Pattern Notice

**Pre-React 19**: To pass a `ref` to a function component, you had to wrap it in another function called `forwardRef`.

**React 19+**: Refs are now passed as regular props, so `forwardRef` is no longer needed. This simplifies the code significantly. `useImperativeHandle` is still used inside the child component to customize the value that the ref receives. This module will show the modern React 19 approach.

## Discovery Phase: The Need for an Imperative API

Let's build a `FancyInput` component. Internally, it might have multiple DOM nodes, but we want to expose a simple API to the parent: `focus()`, `clear()`, and `shake()`. The parent shouldn't need to know about the internal structure of `FancyInput`.

```jsx
import React, { useRef, useImperativeHandle, useState } from "react";
import "./FancyInput.css"; // Assume this file has CSS for a 'shake' animation

// FancyInput.css would contain something like:
// @keyframes shake { 10%, 90% { transform: translate3d(-1px, 0, 0); } ... }
// .shake { animation: shake 0.82s cubic-bezier(.36,.07,.19,.97) both; }

const FancyInput = ({ ref, ...props }) => {
  const internalInputRef = useRef(null);
  const [isShaking, setIsShaking] = useState(false);

  // useImperativeHandle customizes the instance value that is exposed to parent components when using ref.
  useImperativeHandle(ref, () => ({
    // Expose a `focus` method
    focus: () => {
      internalInputRef.current.focus();
    },
    // Expose a `clear` method
    clear: () => {
      internalInputRef.current.value = "";
    },
    // Expose a `shake` method
    shake: () => {
      setIsShaking(true);
      setTimeout(() => setIsShaking(false), 820); // Duration of the CSS animation
    },
  }));

  return (
    <input
      ref={internalInputRef}
      className={isShaking ? "shake" : ""}
      {...props}
    />
  );
};

function App() {
  const fancyInputRef = useRef(null);

  const handleFocusAndClear = () => {
    fancyInputRef.current.focus();
    fancyInputRef.current.clear();
  };

  const handleInvalidSubmit = () => {
    fancyInputRef.current.shake();
  };

  return (
    <div>
      <FancyInput ref={fancyInputRef} placeholder="Enter something..." />
      <button onClick={handleFocusAndClear}>Focus & Clear</button>
      <button onClick={handleInvalidSubmit}>Invalid Submit</button>
    </div>
  );
}

export default App;
```

**Interactive Behavior**:

- Click "Focus & Clear": The input field gets focus and its content is cleared.
- Click "Invalid Submit": The input field performs a visual "shake" animation.

## Deep Dive: How It Works

Let's break down the key parts of `useImperativeHandle`.

`useImperativeHandle(ref, createHandle, [dependencies])`

1.  **`ref`**: This is the `ref` prop passed down from the parent component.
2.  **`createHandle`**: This is a function that should return the object that will be assigned to `parentRef.current`. This object is our custom "API". In our example, it's `{ focus, clear, shake }`.
3.  **`[dependencies]` (optional)**: Just like `useEffect` or `useMemo`, this dependency array determines when the `createHandle` function should be re-run.

### Component Trace

1.  In `App`, we create a ref: `const fancyInputRef = useRef(null)`.
2.  We pass this ref to our component: `<FancyInput ref={fancyInputRef} />`.
3.  Inside `FancyInput`, the `useImperativeHandle` hook is called. It knows about the parent's `ref`.
4.  The hook executes our `createHandle` function: `() => ({ focus: ..., clear: ..., shake: ... })`.
5.  It takes the returned object and assigns it to `fancyInputRef.current` in the parent component.
6.  Now, back in `App`, `fancyInputRef.current` is not the `<input>` DOM node. Instead, it's the custom object we defined.
7.  When we call `fancyInputRef.current.shake()`, we are executing the `shake` function that we defined inside `FancyInput`.

This acts as a protective barrier. The parent component has no access to `internalInputRef` or the component's state. It can only interact with the public API we explicitly exposed.

### Common Confusion: Why not just pass the whole ref up?

**You might think**: Why not just make `internalInputRef` the ref that the parent gets? It seems simpler.

**Actually**: Exposing the internal DOM structure of your component is a bad practice. It creates a tight coupling between the parent and the child. If you were to refactor `FancyInput` to be a `<div>` with a nested `<input>`, the parent's code would break because it would no longer be getting an input element.

**How to remember**: `useImperativeHandle` allows a component to maintain its **encapsulation**. It lets the child decide what, if any, imperative actions it wants to expose to its parents, hiding its internal implementation details.

## Production Perspective

`useImperativeHandle` is an "escape hatch" and should be used sparingly.

**When professionals use this**:

- **Wrapping complex non-React widgets**: When you have a rich text editor or a video player component, you often need to expose methods like `editor.clear()` or `player.play()`.
- **Controlling animations**: Triggering animations imperatively, as in our `shake` example.
- **Managing focus across complex form fields**: A parent form might need to tell a custom `AddressInput` component to "focus the zip code field".

**Trade-offs**:

- ✅ **Advantage**: Provides a clean, explicit way to create an imperative API for a component.
- ✅ **Advantage**: Promotes encapsulation by hiding internal implementation details from the parent.
- ⚠️ **Cost**: It breaks the declarative, top-down data flow of React. This can make your application harder to reason about. Always consider if you can achieve the same result with props and state first. For example, instead of a `shake()` method, you could pass a prop `isInvalid={true}` and have the child component trigger the animation based on that prop change.

## Custom Hooks: Creating Reusable Logic

## Learning Objective

Learn how to create and use custom hooks to extract and share stateful logic between components.

## Why This Matters

Custom hooks are arguably the most powerful feature of the hooks paradigm. They allow you to solve a problem that plagued React for years: how to share stateful logic without resorting to complex patterns like Higher-Order Components or Render Props. Custom hooks make your code cleaner, more reusable, and easier to understand.

## Discovery Phase: The Problem of Duplicated Logic

Imagine we have two different components in our app that both need to track the current window width. Without custom hooks, we would have to duplicate the logic in both places.

```jsx
import React, { useState, useEffect } from "react";

function Header() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    // Cleanup function
    return () => window.removeEventListener("resize", handleResize);
  }, []); // Empty array ensures this effect runs only once

  return <header>{width > 800 ? "Desktop Header" : "Mobile Header"}</header>;
}

function Sidebar() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return <aside>{width > 600 ? "Full Sidebar" : "Collapsed Sidebar"}</aside>;
}

export default function App() {
  return (
    <div>
      <Header />
      <Sidebar />
    </div>
  );
}
```

**Interactive Behavior**:
If you resize your browser window, you'll see the text in the header and sidebar change as they cross their respective width thresholds.

This code works, but the `useState` and `useEffect` logic for tracking window width is identical in both `Header` and `Sidebar`. This is a violation of the "Don't Repeat Yourself" (DRY) principle. If we needed to add throttling to the resize event, we'd have to change it in two places.

## Deep Dive: Extracting Logic to a Custom Hook

A custom hook is simply a JavaScript function whose name starts with "use" and that can call other hooks. Let's extract our duplicated logic into a `useWindowWidth` hook.

```jsx
import React, { useState, useEffect } from "react";

// 1. Create the custom hook. It's just a function.
// Its name MUST start with "use".
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  // 3. Return the value the component needs.
  return width;
}

// --- Now our components become much simpler ---

function Header() {
  // 2. Use the custom hook just like a built-in one.
  const width = useWindowWidth();
  return <header>{width > 800 ? "Desktop Header" : "Mobile Header"}</header>;
}

function Sidebar() {
  const width = useWindowWidth();
  return <aside>{width > 600 ? "Full Sidebar" : "Collapsed Sidebar"}</aside>;
}

export default function App() {
  return (
    <div>
      <Header />
      <Sidebar />
    </div>
  );
}
```

The behavior is identical, but the code is vastly improved.

- The stateful logic is defined in one place (`useWindowWidth`).
- The components (`Header`, `Sidebar`) are now declarative and simple. They just ask for the window width and use it. They don't care _how_ that width is obtained or tracked.

### Component Trace: How Custom Hooks Share Logic (But Not State)

This is a critical concept to understand. When `Header` and `Sidebar` both call `useWindowWidth`, they are **not sharing state**. Each one gets its own independent copy of the state inside the hook.

1.  `Header` renders and calls `useWindowWidth()`. React creates a new `width` state variable for this instance.
2.  `Sidebar` renders and calls `useWindowWidth()`. React creates a _separate_, new `width` state variable for this instance.
3.  The `useEffect` inside the hook sets up two separate event listeners, one for each component's state.
4.  When the window resizes, both event listeners fire, and each one calls its own `setWidth` function, updating its own component's state.

Custom hooks share **stateful logic**, not state itself.

### Common Confusion: Is a custom hook a component?

**You might think**: It's a function that uses React features, so it's a type of component.

**Actually**: No. A component is a function that returns JSX (or other renderable output). A custom hook is a function that returns arbitrary values (a number, a string, an object, an array) and is meant to be used _inside_ other components or hooks.

**How to remember**: Components make UI. Hooks provide logic _to_ components. If it returns JSX, it's a component. If its name starts with "use" and it doesn't return JSX, it's a hook.

## Production Perspective

Custom hooks are the primary pattern for code reuse and abstraction in modern React.

**Common Custom Hooks you'll see or build**:

- `useFetch(url)`: Encapsulates data fetching, loading, and error states.
- `useLocalStorage(key, initialValue)`: A drop-in replacement for `useState` that persists the value to local storage.
- `useForm(initialState)`: Manages the state of a form, including values, errors, and submission handlers.
- `useAuth()`: Provides information about the current user from an `AuthContext`.
- `useDebounce(value, delay)`: Returns a debounced version of a value, useful for search inputs.

**Best Practices**:

- **Keep them focused**: A good hook does one thing well. `useWindowSize` is better than `useBrowserAPIs` that returns everything.
- **Flexible return values**: Return an object (`{ data, loading, error }`) or an array (`[value, setValue]`) depending on what's most ergonomic for the consumer. Arrays are great for hooks that mimic built-in ones like `useState`. Objects are better when you return many values and want them to be self-documenting.
- **Parameterize them**: Make your hooks configurable by accepting arguments, like the `url` in `useFetch`.

## Hook Composition Patterns

## Learning Objective

Learn how to compose custom hooks by building more complex hooks that use other, simpler hooks, creating powerful and reusable logic.

## Why This Matters

The true power of custom hooks is unlocked when you start treating them as building blocks. Just as you compose simple components to build complex UIs, you can compose simple hooks to build complex logic. This pattern allows you to create layers of abstraction that make your application code incredibly clean and maintainable.

## Discovery Phase: Building on Our Blocks

In the last section, we created a `useWindowWidth` hook. Let's say we also have another common need: persisting a user's preference to local storage. We can create a `useLocalStorage` hook.

```jsx
import React, { useState, useEffect } from "react";

// Hook 1: A generic hook to sync state with localStorage
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.log(error);
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.log(error);
    }
  };

  return [storedValue, setValue];
}

// Hook 2: Our hook from the previous section
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);
  return width;
}

// A component using one of our hooks
function UserPreferences() {
  const [theme, setTheme] = useLocalStorage("theme", "light");

  return (
    <div>
      <p>Current theme: {theme}</p>
      <button onClick={() => setTheme("light")}>Light Mode</button>
      <button onClick={() => setTheme("dark")}>Dark Mode</button>
    </div>
  );
}

export default UserPreferences;
```

**Interactive Behavior**:
Click "Dark Mode". The theme changes. Now, refresh the page. The theme remains "dark" because it was saved to local storage.

We now have two useful, independent hooks: `useLocalStorage` and `useWindowWidth`.

## Deep Dive: Composing Hooks

Now, let's create a new, more powerful hook by composing the two we already have. Let's build `useResponsiveTheme`. The requirement is: the theme should be "dark" on large screens and "light" on small screens, but the user should also be able to override this preference and have it saved.

This new hook will _use_ our other hooks.

```jsx
import React, { useState, useEffect } from "react";

// --- Assume useLocalStorage and useWindowWidth from previous example exist here ---
function useLocalStorage(key, initialValue) {
  /_ ... implementation ... _/;
}
function useWindowWidth() {
  /_ ... implementation ... _/;
}

// Our new, composed hook
function useResponsiveTheme() {
  // 1. Use our existing hooks to get the building blocks
  const width = useWindowWidth();
  const [preferredTheme, setPreferredTheme] = useLocalStorage("theme", null);

  // 2. Derive state based on the output of our other hooks
  const isDesktop = width > 768;
  const defaultTheme = isDesktop ? "dark" : "light";

  // 3. The final theme is the user's preference, or the responsive default
  const theme = preferredTheme || defaultTheme;

  // 4. Return the theme and a way to set the preference
  return [theme, setPreferredTheme];
}

// --- Our component is now incredibly simple ---
function App() {
  const [theme, setTheme] = useResponsiveTheme();

  const appStyle = {
    backgroundColor: theme === "dark" ? "#282c34" : "#ffffff",
    color: theme === "dark" ? "white" : "black",
    padding: "20px",
    minHeight: "100vh",
    transition: "all 0.3s",
  };

  return (
    <div style={appStyle}>
      <h1>Responsive Theming App</h1>
      <p>Current theme: {theme}</p>
      <p>Resize your window or use the buttons below.</p>
      <button onClick={() => setTheme("light")}>Set to Light</button>
      <button onClick={() => setTheme("dark")}>Set to Dark</button>
      <button onClick={() => setTheme(null)}>Reset to Auto</button>
    </div>
  );
}

export default App;
```

**Interactive Behavior**:

- If your window is wide, the theme will default to "dark". If it's narrow, it defaults to "light".
- Resizing the browser will automatically change the theme.
- If you click "Set to Light", the theme will become "light" and stay that way even if you resize the window. Your preference is saved.
- Clicking "Reset to Auto" clears your preference, and the theme becomes responsive again.

### Component Trace

1.  The `App` component calls `useResponsiveTheme()`.
2.  Inside `useResponsiveTheme`, it first calls `useWindowWidth()`. This hook sets up its own state and effect to track the window width.
3.  Next, it calls `useLocalStorage('theme', null)`. This hook sets up its own state and logic to interact with local storage.
4.  The rest of the logic in `useResponsiveTheme` simply uses the values returned by the other hooks (`width` and `preferredTheme`) to calculate the final `theme`.
5.  It returns the final `theme` and the `setPreferredTheme` function to the `App` component.

This is the beauty of composition. Our `App` component is completely unaware of `localStorage` or window resize events. It just asks for a responsive theme and gets it. All the complexity is neatly encapsulated within the `useResponsiveTheme` hook, which itself is built from smaller, single-purpose hooks.

## Production Perspective

Hook composition is the cornerstone of modern React architecture.

**When professionals use this pattern**:

- **Everywhere**. It's the standard way to build abstractions.
- To create application-specific hooks like `useCurrentUser`, which might compose `useContext` and `useFetch`.
- To build form logic hooks like `useForm`, which will compose multiple `useState` or `useReducer` calls.
- To create data-fetching hooks that also handle authentication, like `useAuthenticatedFetch`, which might compose `useAuth` and `useFetch`.

**Benefits**:

- **Separation of Concerns**: Each hook handles one piece of logic. Components focus on rendering UI.
- **Reusability**: `useLocalStorage` can be used dozens of times throughout an application.
- **Testability**: Simple, pure hooks are easy to unit test in isolation.
- **Readability**: A component that uses `const { data, isLoading } = usePaginatedPosts('react');` is much easier to understand than a component filled with all the raw state and effects for pagination and fetching.

This pattern allows you to build your own "language" for your application's logic, making development faster, safer, and more enjoyable.

## Module Synthesis 📋

## Module Synthesis: Advanced Hooks and the `use` API

This module marked a significant step up in our React journey, moving from the foundational hooks to the advanced patterns that power modern, complex applications. We've covered a wide spectrum of tools, from the revolutionary to the classic.

### Key Takeaways

1.  **The `use` Hook is a Game-Changer**: React 19's `use` hook fundamentally alters how we interact with resources.
    - It can read from **Promises**, turning asynchronous data fetching into code that looks synchronous and integrating seamlessly with **Suspense**.
    - It can read from **Context**, and crucially, it can do so **conditionally**, unlocking more efficient and flexible component designs.

2.  **Classic Hooks Remain Essential**:
    - **`useContext`**: The standard for unconditional consumption of shared, global data.
    - **`useReducer`**: The tool of choice for managing complex, related state, promoting predictable state transitions by centralizing logic in a reducer function.
    - **`useRef`**: Our "escape hatch" for imperative needs, allowing us to access DOM nodes directly and to maintain mutable values without triggering re-renders.

3.  **Manual Optimization is Becoming History**:
    - We learned about `useCallback` and `useMemo` as tools to manually optimize performance by memoizing functions and values.
    - Crucially, we framed these in the context of the new **React 19 Compiler**, which aims to make these manual optimizations unnecessary in most cases. Understanding them is key to grasping the problems the compiler solves automatically.

4.  **Custom Hooks are the Core of Modern React Architecture**:
    - The most important pattern we learned is the ability to create our own hooks to **extract and reuse stateful logic**.
    - We saw how to **compose hooks**, building powerful, high-level hooks from simpler, single-purpose ones. This is the primary way to build clean, maintainable, and scalable React applications.

### Looking Forward

You now have a powerful toolkit of hooks at your disposal. You understand not only how to manage state and effects, but how to structure and reuse logic in a clean, composable way. You've also had your first taste of React 19's biggest features, which simplify asynchronous code and component design.

In the next chapter, **Chapter 7: React 19 Compiler and Automatic Optimization**, we will dive deep into the magic that makes manual memoization with `useCallback` and `useMemo` a thing of the past. You'll learn how the compiler works, how to write compiler-friendly code, and how React is making high-performance applications the default, not the exception.
