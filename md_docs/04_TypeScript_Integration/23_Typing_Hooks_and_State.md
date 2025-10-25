# Chapter 23: Typing Hooks and State

## useState with TypeScript

## Learning Objective

Type `useState` for simple and complex states, including objects, arrays, and union types, by leveraging both type inference and explicit generic arguments.

## Why This Matters

`useState` is the most fundamental hook for managing state in React. Understanding how to type it correctly is the first step toward building interactive, type-safe components. Getting this right prevents a wide range of bugs, from trying to put a string into a number state to accessing non-existent properties on a state object.

## Discovery Phase

For simple primitive values, TypeScript's inference works perfectly, and you don't need to do anything special.

```jsx
import React, { useState } from "react";

function Counter() {
  // TypeScript infers the type from the initial value (0).
  // `count` is inferred as type `number`.
  // `setCount` is inferred as `React.Dispatch<React.SetStateAction<number>>`.
  const [count, setCount] = useState(0);

  const increment = () => {
    // This is type-safe. We are passing a number.
    setCount(count + 1);

    // The function update form is also typed. `c` is inferred as a number.
    setCount((c) => c + 1);
  };

  // setCount('hello'); // This would be a compile-time error.

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

This is the ideal scenario. You provide an initial value, and TypeScript handles the rest. However, what if your state can be one of several types, or starts as `null`? This is where you need to give TypeScript an explicit hint using a generic argument.

```jsx
import React, { useState } from 'react';

type User = {
  name: string;
  id: number;
};

function UserProfile() {
  // The user data might not be loaded yet, so it starts as `null`.
  // We must explicitly tell `useState` that this state can be `User` OR `null`.
  const [user, setUser] = useState<User | null>(null);

  const login = () => {
    setUser({ name: 'Alice', id: 1 });
  };

  if (!user) {
    return <button onClick={login}>Login</button>;
  }

  // Because of our type, TS knows `user` is a `User` object here.
  return <p>Welcome, {user.name}!</p>;
}
```

Without the `<User | null>` generic, `useState(null)` would infer the type as `null`, and you would never be able to call `setUser` with a user object. The generic argument is your way of telling TypeScript the full range of possible types for that state variable.

## Deep Dive

### When to Use Explicit Generics

You must provide an explicit generic argument to `useState` in two common scenarios:

1.  **When the initial state is `null` or `undefined`**, but it will hold a value of a different type later (like our `User` example).
2.  **When the initial state is an empty array `[]`**, but you want TypeScript to know the type of items the array will eventually hold.

```jsx
import React, { useState } from 'react';

type Todo = { id: number; text: string; };

function TodoList() {
  // Without `<Todo[]>`, `todos` would be inferred as `never[]`,
  // and you couldn't add any items to it.
  const [todos, setTodos] = useState<Todo[]>([]);

  const addTodo = () => {
    const newTodo = { id: Date.now(), text: 'New task' };
    setTodos([...todos, newTodo]);
  };

  return (
    <div>
      <button onClick={addTodo}>Add Todo</button>
      <ul>
        {todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
      </ul>
    </div>
  );
}
```

### Typing Object State

When your state is an object, it's best to define a `type` or `interface` for its shape.

```typescript
type Settings = {
  theme: "light" | "dark";
  notifications: boolean;
};

function SettingsPanel() {
  const [settings, setSettings] = useState<Settings>({
    theme: "light",
    notifications: true,
  });

  const toggleTheme = () => {
    // Use the spread operator to update one part of the object
    // while preserving the rest.
    setSettings((currentSettings) => ({
      ...currentSettings,
      theme: currentSettings.theme === "light" ? "dark" : "light",
    }));
  };
}
```

This ensures that you can't accidentally remove a property or add a new one that isn't part of the `Settings` type.

### Common Confusion: "Why can't I just use `useState({})` for an object?"

**You might think**: "I'll start with an empty object and add properties later."

**Actually**: `useState({})` tells TypeScript that the state is of type `{}`—an object with no properties. You will get an error if you try to access or set any properties on it later.

**Why the confusion happens**: It's a common pattern in JavaScript to initialize an empty object.

**How to remember**: With `useState`, you must declare the full shape of your state upfront. Either provide a complete initial object for inference, or use a generic argument like `useState<MyObjectType | null>(null)`.

### Production Perspective

- **Keep State Flat**: When dealing with complex object state, prefer keeping it as flat as possible. Deeply nested state objects can be difficult to update immutably.
- **Consider `useReducer`**: If you find yourself with multiple `useState` calls that are all related, or if the logic for updating your state object is complex, it's a strong signal that you should refactor to `useReducer`. We'll cover this in the next section.

## `useReducer` Type Patterns

## Learning Objective

Implement a fully-typed `useReducer` hook by defining types for the state, a discriminated union for actions, and the reducer function itself.

## Why This Matters

For complex state management, `useReducer` provides a more structured and predictable alternative to `useState`. Typing it correctly makes your state transitions explicit and bulletproof, preventing invalid state changes and making the logic easier to reason about, test, and maintain.

## Discovery Phase

Let's refactor our simple `Counter` from the previous section to use `useReducer`. This will clearly illustrate the core concepts.

```jsx
import React, { useReducer } from 'react';

// Step 1: Define the shape of the state.
type CounterState = {
  count: number;
};

// Step 2: Define the actions using a discriminated union.
// This is the most important part. It lists all possible state transitions.
type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' }
  | { type: 'add_amount'; payload: number }; // An action with a payload

// Step 3: Define the reducer function.
// It takes the current state and an action, and returns the new state.
const counterReducer = (state: CounterState, action: CounterAction): CounterState => {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    case 'add_amount':
      // TS knows `action` has a `payload` of type `number` here.
      return { count: state.count + action.payload };
    default:
      throw new Error('Unhandled action type');
  }
};

const initialState: CounterState = { count: 0 };

function Counter() {
  // Step 4: Use the hook. TypeScript infers everything from the reducer.
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      {/* `dispatch` is fully typed. It will only accept objects
          that match our `CounterAction` type. */}
      <button onClick={() => dispatch({ type: 'increment' })}>Increment</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>Decrement</button>
      <button onClick={() => dispatch({ type: 'add_amount', payload: 5 })}>Add 5</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

This pattern is incredibly robust:

1.  The `CounterAction` discriminated union acts as a contract for all possible state changes. It's impossible to dispatch an action with a typo (e.g., `dispatch({ type: 'incremennt' })`).
2.  Inside the reducer's `switch` statement, TypeScript narrows the `action` type. In the `'add_amount'` case, it knows that `action.payload` exists and is a number.
3.  The `useReducer` hook infers the types of `state` and `dispatch` from the reducer function you provide. You don't need to add any generics to the hook call itself.

## Deep Dive

### Why a Discriminated Union for Actions?

Using a discriminated union for your actions is the cornerstone of this pattern. It provides several key benefits over using, for example, a single object type with optional properties:

- **Explicitness**: It clearly enumerates every possible action.
- **Safety**: It prevents you from dispatching actions with invalid combinations of properties.
- **Type Narrowing**: It enables TypeScript's `switch` statement narrowing, which is what makes the reducer implementation so safe and provides excellent autocompletion.

### Production Perspective

- **When to Choose `useReducer`**:
  - When you have complex state logic that involves multiple sub-values.
  - When the next state depends on the previous one.
  - When you want to co-locate all state transition logic in one place.
  - When you want to optimize performance for components that trigger deep updates, as you can pass `dispatch` down instead of callbacks.
- **Organization**: For complex components, it's a common and highly recommended practice to extract the reducer, action types, and initial state into a separate file (e.g., `counterReducer.ts`). This separates the state management logic from the view logic, making both easier to manage and test.
- **Testing**: Reducer functions are "pure functions"—they take state and an action and return a new state with no side effects. This makes them incredibly easy to unit test. You can test your entire state logic without even rendering a React component.

## `useContext` with TypeScript

## Learning Objective

Create and consume a type-safe context, handling the common case of a nullable default value and creating a custom hook to provide a non-null context value.

## Why This Matters

`useContext` is React's solution for avoiding "prop drilling"—passing props down through many levels of the component tree. Typing your context correctly ensures that any component consuming it receives the expected data and functions, and it prevents runtime errors from consuming a context outside of its provider.

## Discovery Phase

Let's create a simple context to provide user authentication status to any component in our app.

```jsx
'use client';
import React, { createContext, useContext, useState } from 'react';

// Step 1: Define the shape of the context's value.
type AuthContextType = {
  isAuthenticated: boolean;
  username: string | null;
  login: (username: string) => void;
  logout: () => void;
};

// Step 2: Create the context with a generic argument and a default value.
// It's common to use `null` as the default for contexts that require a provider.
const AuthContext = createContext<AuthContextType | null>(null);

// Step 3: Create the Provider component.
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [username, setUsername] = useState<string | null>(null);

  const login = (name: string) => setUsername(name);
  const logout = () => setUsername(null);

  const value: AuthContextType = {
    isAuthenticated: username !== null,
    username,
    login,
    logout,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Step 4: Create a component that consumes the context.
function UserStatus() {
  const auth = useContext(AuthContext);

  // Step 5: Handle the nullable type.
  // `auth` is of type `AuthContextType | null`. We must check for null.
  if (!auth) {
    // This happens if `UserStatus` is rendered outside of `AuthProvider`.
    return <p>Please wrap your app in AuthProvider.</p>;
  }

  return (
    <div>
      {auth.isAuthenticated ? (
        <p>Welcome, {auth.username}! <button onClick={auth.logout}>Logout</button></p>
      ) : (
        <button onClick={() => auth.login('Alice')}>Login</button>
      )}
    </div>
  );
}

// --- USAGE ---
export default function App() {
    return (
        <AuthProvider>
            <UserStatus />
        </AuthProvider>
    )
}
```

This pattern is robust and safe. The key is `createContext<AuthContextType | null>(null)`.

- The generic `<AuthContextType | null>` tells TypeScript what types to expect.
- The `null` initial value forces any consumer of the context to perform a null check. This is a feature, not a bug! It prevents you from accidentally using the component without wrapping it in the necessary provider, which would lead to a runtime error.

## Deep Dive

### Creating a Custom Hook for the Context

Having to check for `null` in every component that consumes the context can be repetitive. A very common and highly recommended pattern is to create a custom hook that wraps `useContext` and includes the null check.

```jsx
// In the same file as the context creation...

// This custom hook provides a better developer experience.
export function useAuth() {
  const context = useContext(AuthContext);
  if (context === null) {
    // This provides a much more helpful error message.
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}

// Now, our consumer component becomes much cleaner:
function UserStatusWithHook() {
  // `auth` is now guaranteed to be of type `AuthContextType`.
  // No null check needed!
  const auth = useAuth();

  return (
    <div>
      {auth.isAuthenticated ? (
        <p>
          Welcome, {auth.username}!{" "}
          <button onClick={auth.logout}>Logout</button>
        </p>
      ) : (
        <button onClick={() => auth.login("Alice")}>Login</button>
      )}
    </div>
  );
}
```

This is the professional standard for working with context.

1.  It encapsulates the `useContext` call and the null check.
2.  It provides a clear, descriptive error if the provider is missing.
3.  It provides a non-nullable context value to the components, simplifying their implementation.

### Production Perspective

- **Provider Placement**: Place your context providers as low in the component tree as possible. Wrapping your entire application in a provider can cause unnecessary re-renders in large parts of your app when the context value changes.
- **Context for Low-Frequency Updates**: Context is best suited for global data that doesn't change often, such as theme information, user authentication, or language preferences. For high-frequency state updates, consider a more optimized state management library or local state.
- **Memoization**: If the value you provide to a context provider is an object or array literal, it will be recreated on every render, causing all consumers to re-render. You should memoize the value with `useMemo` to prevent this.

## `useRef` Generic Types

## Learning Objective

Correctly apply generic types to `useRef` for two distinct use cases: accessing DOM elements and persisting mutable values across renders.

## Why This Matters

`useRef` is a versatile hook, but its typing can be subtle because it behaves differently depending on how you initialize it. Providing the correct generic type and initial value is crucial for safely interacting with DOM nodes and for correctly typing mutable "instance variables" in your function components.

## Discovery Phase

We'll look at the two primary use cases for `useRef` side-by-side to highlight the difference in their typing.

### Use Case 1: Accessing a DOM Element

This is the most common use case. You want a reference to a DOM node, like an `<input>`, to call methods like `.focus()`.

```jsx
import React, { useRef, useEffect } from "react";

function DOMRefExample() {
  // For DOM refs, the generic is the element type.
  // The initial value MUST be `null`.
  const inputRef = useRef < HTMLInputElement > null;

  useEffect(() => {
    // Because the initial value is `null`, TypeScript correctly infers
    // the type of `inputRef.current` as `HTMLInputElement | null`.
    // We must use optional chaining `?.` or an explicit check.
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} placeholder="I will be focused" />;
}
```

**Rule for DOM Refs**: `useRef<ElementType>(null)`

### Use Case 2: Storing a Mutable Value

Sometimes you need to store a value that persists across renders but doesn't trigger a re-render when it changes. This is like an instance variable.

```jsx
import React, { useRef, useEffect } from "react";

function MutableValueRefExample() {
  // For mutable values, the generic is the value type.
  // We provide a concrete initial value.
  const renderCountRef = useRef < number > 0;

  useEffect(() => {
    // We can safely increment this on every render.
    // Mutating `.current` does NOT cause a re-render.
    renderCountRef.current += 1;
  });

  // Because we provided a non-null initial value, TypeScript knows
  // `renderCountRef.current` is always a `number`. No null check needed.

  return <p>This component has rendered {renderCountRef.current} times.</p>;
}
```

**Rule for Mutable Values**: `useRef<ValueType>(initialValue)`

## Deep Dive

### The `current` Property's Type

The key difference lies in the type of the `.current` property that TypeScript infers:

- `useRef<T>(null)` → `ref.current` is of type `T | null`.
- `useRef<T>(initialValue: T)` → `ref.current` is of type `T`.
- `useRef<T>()` (no initial value) → `ref.current` is of type `T | undefined`.

This last case is often used for values that will be set later, like a timer ID.

```javascript
import { useRef, useEffect } from 'react';

function Timer() {
  // We don't have the timer ID initially.
  const timerRef = useRef<number>(); // `timerRef.current` is `number | undefined`

  useEffect(() => {
    timerRef.current = window.setInterval(() => {
      console.log('Tick');
    }, 1000);

    return () => {
      // We need to check for undefined before clearing.
      if (timerRef.current) {
        window.clearInterval(timerRef.current);
      }
    };
  }, []);

  return <div>Timer is running (see console).</div>;
}
```

### Common Confusion: "Why is the initial value for DOM refs `null`?"

**You might think**: "The element exists, so why start with `null`?"

**Actually**: When the `useRef` line is executed during the component's render, the DOM has not been updated yet. The `<input>` element doesn't exist in the DOM at that moment. React only populates `ref.current` with the DOM node _after_ the render is complete and the element has been mounted. Therefore, its initial value must be `null` to accurately reflect its state during the initial render.

**How to remember**: `useRef` runs before the `return` statement. The `ref` attribute in JSX is handled _after_ the `return`. So, the initial value has to be empty.

### Production Perspective

- **`useImperativeHandle`**: When creating a component that forwards a ref to a custom component (not a DOM node), you'll often combine `useRef` with `useImperativeHandle`. The generic type on the ref should match the handle type you define. (See Chapter 20.4 for a full example).
- **Storing Previous State**: A classic use case for a mutable ref is to store a value from the previous render. This is often extracted into a custom hook.
  ```typescript
  function usePrevious<T>(value: T): T | undefined {
    const ref = useRef<T>();
    useEffect(() => {
      ref.current = value;
    }, [value]);
    return ref.current;
  }
  ```

## `useCallback` and `useMemo` Types (When Needed)

## Learning Objective

Correctly type `useCallback` and `useMemo` while understanding their diminished role for performance optimization in the era of the React 19 Compiler.

## Why This Matters

In previous versions of React, `useCallback` and `useMemo` were essential tools for preventing unnecessary re-renders. While the new React Compiler makes them largely obsolete for this purpose, they are still necessary for specific cases, such as maintaining a stable function identity for dependency arrays or third-party libraries. Fortunately, TypeScript's inference makes typing them extremely simple.

## Discovery Phase

### `useCallback`

`useCallback` memoizes a function instance. You provide a function and a dependency array.

```jsx
import React, { useState, useCallback } from 'react';

function Button({ onClick }: { onClick: () => void }) {
  console.log('Button rendered');
  return <button onClick={onClick}>Click Me</button>;
}
const MemoizedButton = React.memo(Button);

function CallbackExample() {
  const [count, setCount] = useState(0);

  // TypeScript infers the entire type of this function.
  // `handleClick` is typed as `() => void`.
  // No explicit typing is needed on the `useCallback` call itself.
  const handleClick = useCallback(() => {
    console.log('Button was clicked!');
    // `count` is included in the dependency array, so it's not stale.
    console.log(`Current count is ${count}`);
  }, [count]); // Dependency array

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
      <MemoizedButton onClick={handleClick} />
    </div>
  );
}
```

Because `handleClick` is wrapped in `useCallback`, it will only be a new function instance if `count` changes. This prevents the `MemoizedButton` from re-rendering every time `CallbackExample` re-renders. TypeScript infers the type of `handleClick` from the function you pass into it.

### `useMemo`

`useMemo` memoizes a value. You provide a factory function that computes the value and a dependency array.

```jsx
import React, { useState, useMemo } from 'react';

function expensiveCalculation(num: number): number {
  console.log('Performing expensive calculation...');
  // Simulate a slow computation
  let total = 0;
  for (let i = 0; i < num * 1_000_000; i++) {
    total += i;
  }
  return total;
}

function MemoExample() {
  const [count, setCount] = useState(10);
  const [otherState, setOtherState] = useState(0);

  // TypeScript infers the return type of `expensiveCalculation`.
  // `computedValue` is correctly typed as `number`.
  const computedValue = useMemo(() => {
    return expensiveCalculation(count);
  }, [count]); // Only re-calculates when `count` changes.

  return (
    <div>
      <p>Count: {count}</p>
      <p>Computed Value: {computedValue}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
      <p>Other State: {otherState}</p>
      {/* Clicking this button re-renders the component, but the expensive
          calculation does NOT run again, thanks to `useMemo`. */}
      <button onClick={() => setOtherState(c => c + 1)}>Update Other State</button>
    </div>
  );
}
```

Just like `useCallback`, `useMemo`'s return type is inferred from the factory function you provide.

## Deep Dive

### Legacy Pattern Notice: The React 19 Compiler

**Pre-React 19**: Manually wrapping functions in `useCallback` and values in `useMemo` was a standard and necessary practice for performance optimization in any non-trivial component.

**React 19+**: The React Compiler (`react-forget`) automates this process. It analyzes your component's code and automatically memoizes components, props, and values where necessary.

**This means you should no longer use `useCallback` or `useMemo` for performance optimization by default.** The compiler is better at it than humans are.

**When are they still useful?**

1.  **Stable Function Identity for Dependencies**: When passing a function to a custom hook that includes it in its dependency array, you may need `useCallback` to prevent infinite loops.
2.  **Third-Party Libraries**: When a library expects a stable function reference (e.g., an event listener for a non-React library).
3.  **Expensive Calculations**: For genuinely expensive, synchronous calculations, `useMemo` is still the correct tool to prevent re-computation on every render. The compiler may not always memoize these aggressively.
4.  **Codebases without the Compiler**: If you are working on a project where the React Compiler is not enabled, the old rules still apply.

### Production Perspective

- **Optimize Last**: Don't prematurely optimize. The React Compiler is the default. Only reach for manual memoization if you have identified a specific performance bottleneck using the React DevTools profiler.
- **Readability**: One of the biggest benefits of the compiler is improved code readability. Overuse of `useCallback` and `useMemo` can clutter components and make them harder to understand. Embrace the simpler, compiler-friendly style of writing components.

## Custom Hook Type Signatures

## Learning Objective

Write and type custom hooks with clear and explicit type signatures for their parameters and return values, establishing a reusable and self-documenting API.

## Why This Matters

Custom hooks are the primary mechanism for sharing stateful logic between components. A well-typed custom hook is like a mini-API for your application's logic. A clear signature makes the hook easy to use, prevents errors, and allows TypeScript to provide excellent autocompletion and inference in the components that consume it.

## Discovery Phase

Let's build a simple `useToggle` hook. It will manage a boolean state and provide a function to toggle it.

```javascript
import { useState, useCallback } from 'react';

// Step 1: Define the hook's signature.
// It takes an optional initial boolean value.
// It returns a tuple, just like `useState`.
export function useToggle(
  initialValue: boolean = false
): [boolean, () => void] {
  const [value, setValue] = useState(initialValue);

  // We use `useCallback` to ensure the toggle function has a stable identity.
  const toggle = useCallback(() => {
    setValue(currentValue => !currentValue);
  }, []);

  return [value, toggle];
}

// --- USAGE ---
import React from 'react';

function ToggleComponent() {
  // TypeScript infers the types from our hook's signature.
  // `isOpen` is a boolean.
  // `toggleOpen` is `() => void`.
  const [isOpen, toggleOpen] = useToggle(true);

  return (
    <div>
      <button onClick={toggleOpen}>
        {isOpen ? 'Hide' : 'Show'}
      </button>
      {isOpen && <p>Now you see me!</p>}
    </div>
  );
}
```

The key is the function signature: `useToggle(initialValue: boolean = false): [boolean, () => void]`.

- **Parameters**: We type `initialValue` as a `boolean` and give it a default value.
- **Return Value**: We explicitly type the return value as a tuple `[boolean, () => void]`. This tells consumers exactly what to expect when they destructure the array.

## Deep Dive

### Returning an Object vs. an Array

For hooks that return more than two values, or where the meaning isn't as obvious as `[state, setState]`, returning an object can be more readable.

Let's build a `useCounter` hook.

```jsx
import { useState } from 'react';

// Step 1: Define the return type as an object.
type UseCounterReturn = {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
};

// Step 2: Define the hook with the explicit return type.
export function useCounter(initialValue: number = 0): UseCounterReturn {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);

  return { count, increment, decrement, reset };
}

// --- USAGE ---
function CounterComponent() {
  // Destructuring an object makes the usage self-documenting.
  const { count, increment, decrement } = useCounter(5);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

### Array vs. Object: Which to Choose?

- **Return an array (tuple)** when your hook is a direct analogue of a built-in hook like `useState`. This allows the consumer to name the destructured variables whatever they want (e.g., `const [isOpen, toggle] = useToggle()`).
- **Return an object** when your hook returns multiple values and functions. This makes the code more self-documenting (`counter.increment()`) and allows the consumer to only destructure the parts they need. It's also easier to add new return values later without breaking the destructuring for existing consumers.

### Production Perspective

- **Hook Naming**: Always start custom hook names with `use`. This is a convention that ESLint plugins rely on to check for rules of hooks (e.g., not calling hooks conditionally).
- **Single Responsibility**: A good custom hook has a single, clear purpose. Don't try to create a "god hook" that does everything. It's better to have several smaller, focused hooks that can be composed together.
- **Documentation**: A well-typed signature _is_ a form of documentation. For complex hooks, supplement this with JSDoc comments explaining what the hook does, what its parameters are, and what it returns. Your editor will show these comments on hover.

## Generic Custom Hooks

## Learning Objective

Create flexible, reusable generic custom hooks that can operate on different data types while maintaining full type safety.

## Why This Matters

Hard-coding types in a custom hook limits its reusability. By making your custom hooks generic, you can create powerful, abstract logic that can be applied to any data structure in your application. This is the key to building a truly reusable logic layer for your frontend.

## Discovery Phase

Let's take a common use case: fetching data from an API. We want a hook that can fetch any kind of data and manage the loading, error, and data states for us. This is a perfect candidate for a generic hook.

```javascript
import { useState, useEffect } from 'react';

// Step 1: Define the shape of the state the hook will manage.
// It's generic over `T`, which represents the type of the data.
type FetchState<T> = {
  data: T | null;
  isLoading: boolean;
  error: Error | null;
};

// Step 2: Make the custom hook generic.
// `useFetch<T>` introduces the generic type parameter `T`.
export function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    isLoading: true,
    error: null,
  });

  useEffect(() => {
    // Reset state for new URL
    setState({ data: null, isLoading: true, error: null });

    const fetchData = async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        // We assume the response is JSON and can be cast to `T`.
        const data: T = await response.json();
        setState({ data, isLoading: false, error: null });
      } catch (error) {
        setState({ data: null, isLoading: false, error: error as Error });
      }
    };

    fetchData();
  }, [url]); // Re-fetch when the URL changes.

  return state;
}
```

Now let's see how a component would consume this generic hook.

```jsx
// --- USAGE ---
import React from 'react';

// Define the shape of the data we expect.
type Post = {
  id: number;
  title: string;
  body: string;
};

function PostViewer({ postId }: { postId: number }) {
  // Step 3: Provide the generic type argument when calling the hook.
  // We tell `useFetch` that `T` is `Post`.
  const { data, isLoading, error } = useFetch<Post>(
    `https://jsonplaceholder.typicode.com/posts/${postId}`
  );

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  // Because we provided `<Post>`, TypeScript knows `data` is `Post | null`.
  // After the loading/error checks, we can safely access `data.title`.
  return (
    <div>
      <h1>{data?.title}</h1>
      <p>{data?.body}</p>
    </div>
  );
}
```

This pattern is incredibly powerful:

1.  The `useFetch` hook contains all the generic logic for fetching, loading, and error handling.
2.  The `PostViewer` component is only concerned with its specific needs: what type of data it expects (`Post`) and how to render it.
3.  By providing `<Post>` as the generic argument, we create a fully type-safe link between the hook and the component. `data` is correctly typed as `Post | null`, giving us autocompletion and safety.

## Deep Dive

### Inferring Generics from Parameters

Sometimes, TypeScript can infer the generic type from the hook's arguments, so you don't have to provide it explicitly. Let's create a generic `useLocalStorage` hook.

```javascript
import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.log(error);
      return initialValue;
    }
  });

  const setValue = (value: T) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.log(error);
    }
  };

  return [storedValue, setValue];
}

// --- USAGE ---
function Settings() {
    // Here, we do NOT need to write `<boolean>`.
    // TypeScript infers that `T` is `boolean` because `initialValue` is `true`.
    const [isDarkMode, setDarkMode] = useLocalStorage('darkMode', true);

    // `isDarkMode` is correctly typed as `boolean`.
    return <button onClick={() => setDarkMode(!isDarkMode)}>Toggle Dark Mode</button>
}
```

**When do you need to be explicit?**

- **Be explicit** (`useFetch<Post>`) when the generic type only appears in the _return value_ of the hook.
- **Let it be inferred** (`useLocalStorage('key', true)`) when the generic type is used in one of the hook's _parameters_ (like `initialValue: T`).

### Production Perspective

- **Data Fetching Libraries**: Professional data fetching libraries like React Query (`useQuery`) and SWR (`useSWR`) are built around this exact generic custom hook pattern. They add more advanced features like caching, revalidation, and mutations, but the core typing strategy is the same.
- **Abstracting Logic**: Generic custom hooks are the ultimate tool for abstracting away complex but reusable logic. Any time you find yourself writing similar stateful logic in multiple components, consider if it can be extracted into a generic custom hook.

## Hook Type Inference Best Practices

## Learning Objective

Synthesize best practices for leveraging TypeScript's type inference with hooks to write code that is clean, maintainable, and minimally annotated.

## Why This Matters

The goal of using TypeScript with React is not to add type annotations everywhere. It's to add them in strategic places to enable TypeScript's powerful inference engine to do most of the work for you. Following these best practices leads to code that is both safe and highly readable.

## Discovery Phase

Let's look at two versions of the same component. The first is "over-typed," with many redundant annotations. The second follows best practices, relying on inference.

```jsx
import React, { useState, useCallback } from 'react';

// Version 1: Over-typed and verbose
function VerboseCounter() {
  // Redundant: `number` is easily inferred from `0`.
  const [count, setCount] = useState<number>(0);

  // Redundant: The callback's type is easily inferred.
  const increment: () => void = useCallback((): void => {
    // Redundant: `c` is inferred as `number`.
    setCount((c: number) => c + 1);
  }, []);

  return <button onClick={increment}>Count: {count}</button>;
}

// Version 2: Clean and inferred (Best Practice)
function CleanCounter() {
  // `count` is inferred as `number`. Perfect.
  const [count, setCount] = useState(0);

  // `increment` is inferred as `() => void`. Perfect.
  const increment = useCallback(() => {
    // `c` is inferred as `number`. Perfect.
    setCount(c => c + 1);
  }, []);

  return <button onClick={increment}>Count: {count}</button>;
}
```

Both components are equally type-safe. However, the second version is much cleaner and easier to read. It trusts TypeScript to do its job. Our goal is to write code that looks like the second version.

## Deep Dive

### A Checklist for Hook Typing

Here is a summary of the best practices we've learned, framed as a set of rules.

**1. `useState`:**

- **DO** let the type be inferred from a non-empty initial value (`useState(0)`, `useState('hello')`, `useState({ a: 1 })`).
- **DO** provide an explicit generic when the initial value is `null`, `undefined`, or an empty array (`useState<User | null>(null)`, `useState<Todo[]>([])`).

**2. `useReducer`:**

- **DO** define explicit types for your `State` and your `Action` (preferably a discriminated union).
- **DO** type the parameters and return value of your reducer function.
- **DON'T** add generics to the `useReducer` call itself. Let it be inferred from the reducer.

**3. `useContext`:**

- **DO** provide an explicit generic to `createContext` (`createContext<MyContextType | null>(null)`).
- **DON'T** add types to the `useContext` call. Let it be inferred from the context object.
- **DO** create a custom hook to encapsulate the null check.

**4. `useRef`:**

- **DO** provide an explicit generic for DOM refs and initialize with `null` (`useRef<HTMLInputElement>(null)`).
- **DO** provide an explicit generic for mutable values if the initial value is not provided (`useRef<number>()`).
- **DO** let the type be inferred if providing a concrete initial value (`useRef(0)`).

**5. `useCallback` & `useMemo`:**

- **DON'T** add explicit types to the hooks themselves. Let them be inferred from the function you pass in.

**6. Custom Hooks:**

- **DO** define explicit types for the hook's parameters and its return value. This is the "public API" of your hook.
- **DON'T** over-type the variables _inside_ your custom hook. Let inference work there.

### Production Perspective

- **Type the Boundaries**: The recurring theme is to **type the boundaries** of your logic. For components, the boundary is `props`. For hooks, it's the function signature (parameters and return value). For context, it's the `createContext` call. For reducers, it's the reducer function signature. If you type these boundaries correctly, inference can handle almost everything else.
- **Readability Trumps Verbosity**: The goal is code that is easy for humans to read and maintain. Redundant type annotations add noise and can make the code harder to follow. Trust the compiler and your editor's hover tooltips to know the inferred types.
- **Configuration is Key**: All of this relies on having a strict `tsconfig.json` (`"strict": true`). A strict configuration enables the compiler to catch more errors and perform more powerful inference, which in turn allows you to write cleaner, less annotated code.

## Module Synthesis 📋

## Module Synthesis

In this chapter, we've taken a deep dive into the heart of a component's internal logic: its state. We have systematically covered how to apply TypeScript to React's core hooks, ensuring that the state driving our components is robust, predictable, and free from common type-related bugs.

We began with **`useState`**, learning the crucial distinction between relying on inference for simple cases and providing explicit generics for nullable or empty initial states. We then leveled up to **`useReducer`**, mastering the powerful pattern of using discriminated unions for actions to create fully type-safe state machines.

We explored how to manage shared state with **`useContext`**, establishing the best practice of creating a typed context, providing a `null` default, and wrapping it in a custom hook to ensure consumers are always within a provider. We clarified the dual nature of **`useRef`**, learning the distinct typing patterns for both DOM element access and mutable instance variables.

We also addressed the legacy optimization hooks, **`useCallback` and `useMemo`**, noting their diminished importance in the face of the React 19 Compiler but confirming how easily their types are inferred. Finally, we tied everything together by learning to write and type **custom hooks**, including powerful **generic custom hooks**, which represent the pinnacle of reusable, type-safe logic in React.

## Looking Ahead

You are now capable of managing state of any complexity within your React components in a fully type-safe manner. You can create reusable, encapsulated logic with custom hooks that are as robust as those from a professional library.

In the next chapter, **Chapter 24: TypeScript with State Management**, we will look beyond React's built-in hooks. We'll explore how to integrate TypeScript with popular external state management libraries. We'll cover typing Redux with Redux Toolkit, see the elegant simplicity of Zustand's type support, and revisit the Context API with more advanced patterns to create scalable, type-safe global state solutions for large applications.
