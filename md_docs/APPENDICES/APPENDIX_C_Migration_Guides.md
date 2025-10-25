# Chapter C: Appendix C: Migration Guides

## React 18 to React 19

## Learning Objective

Understand the key changes in React 19 and develop a strategy for upgrading your application from React 18.

## Why This Matters

React 19 introduces powerful new features like Actions, the `use` hook, and first-class support for Server Components. While many changes are incremental, adopting the new patterns will improve performance, reduce boilerplate, and align your application with the future of React.

## Core Migration Steps

Upgrading the core library is straightforward. The main work involves adopting the new patterns where they provide the most value.

### 1. Update Dependencies

In your `package.json`, update your React dependencies:

```json
"dependencies": {
  "react": "^19.0.0",
  "react-dom": "^19.0.0"
},
"devDependencies": {
  "@types/react": "^19.0.0",
  "@types/react-dom": "^19.0.0"
}
```

Then, run your package manager's install command (e.g., `npm install`).

### 2. Address Breaking Changes

React 19 has very few breaking changes, but you should be aware of them:

- **`ref` cleanup functions**: `ref` props can now return a cleanup function, which might affect some libraries.
- **Stricter `useEffect` cleanup**: The cleanup function from `useEffect` must be a function or undefined. Returning anything else is now an error.
- **Removal of legacy APIs**: Some older, rarely used APIs like `React.PropTypes` and `React.createFactory` have been removed.

### 3. Adopt New Features Incrementally

You don't need to rewrite your app overnight. Start by identifying areas that would benefit most from React 19 features.

#### Example: Migrating a Form to use Actions

A common pattern in React 18 is manual state management for forms. React 19's Actions simplify this significantly.

```jsx
// BEFORE: React 18 Manual Form Handling
import { useState } from "react";

function UpdateProfileForm() {
  const [name, setName] = useState("");
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState(null);

  const handleSubmit = async (event) => {
    event.preventDefault();
    setIsPending(true);
    setError(null);

    const response = await updateUser({ name });
    if (response.error) {
      setError(response.error);
    }

    setIsPending(false);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
        disabled={isPending}
      />
      <button type="submit" disabled={isPending}>
        {isPending ? "Updating..." : "Update"}
      </button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

```jsx
// AFTER: React 19 with useActionState
import { useActionState } from "react";
import { useFormStatus } from "react-dom";

// The server action function (could be in a separate file)
async function updateUser(previousState, formData) {
  const name = formData.get("name");
  // ... API call logic
  if (!name) {
    return { error: "Name is required." };
  }
  return { success: true };
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Updating..." : "Update"}
    </button>
  );
}

function UpdateProfileForm() {
  const [state, formAction] = useActionState(updateUser, null);

  return (
    <form action={formAction}>
      <input type="text" name="name" />
      <SubmitButton />
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
```

### Production Perspective

The React 19 version is not only less code but also more robust. It handles pending states declaratively with `useFormStatus` and manages form state transitions with `useActionState`. This pattern also enables progressive enhancement: the form would work even if JavaScript fails to load. Start migrating your most critical forms to Actions to see the benefits immediately.

## Removing forwardRef

## Learning Objective

Refactor components that use `React.forwardRef` to the simpler `ref`-as-a-prop pattern available in React 19.

## Why This Matters

The `forwardRef` API added boilerplate and complexity, especially with TypeScript. React 19 simplifies this by treating `ref` as a regular prop, making component code cleaner, more intuitive, and easier to type.

## The Migration Pattern

The change is purely mechanical: you remove the `forwardRef` wrapper and accept `ref` directly from the props object.

```jsx
// BEFORE: React 18 with forwardRef
import { forwardRef } from "react";

const MyInput = forwardRef(function MyInput(props, ref) {
  const { label, ...otherProps } = props;
  return (
    <label>
      {label}
      <input ref={ref} {...otherProps} />
    </label>
  );
});

// Usage
// const inputRef = useRef();
// <MyInput ref={inputRef} label="Name" />
```

```jsx
// AFTER: React 19 with ref as a prop
// No import needed for forwardRef

function MyInput(props) {
  const { label, ref, ...otherProps } = props;
  return (
    <label>
      {label}
      <input ref={ref} {...otherProps} />
    </label>
  );
}

// Usage (is identical)
// const inputRef = useRef();
// <MyInput ref={inputRef} label="Name" />
```

### TypeScript Migration

The simplification is even more pronounced in TypeScript.

```jsx
// BEFORE: TypeScript with forwardRef
import { forwardRef, ComponentPropsWithoutRef } from 'react';

type MyInputProps = {
  label: string;
} & ComponentPropsWithoutRef<'input'>;

const MyInput = forwardRef<HTMLInputElement, MyInputProps>(
  function MyInput({ label, ...otherProps }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} {...otherProps} />
      </label>
    );
  }
);
```

```jsx
// AFTER: TypeScript with ref as a prop
import { ComponentPropsWithoutRef, Ref } from 'react';

type MyInputProps = {
  label: string;
  ref?: Ref<HTMLInputElement>; // Add ref to the props type
} & ComponentPropsWithoutRef<'input'>;

function MyInput({ label, ref, ...otherProps }: MyInputProps) {
  return (
    <label>
      {label}
      <input ref={ref} {...otherProps} />
    </label>
  );
}
```

### Production Perspective

This is a low-risk, high-reward refactor. It reduces boilerplate, improves readability, and simplifies your types. You can use a codemod or a simple find-and-replace to automate most of this migration across your codebase.

## Manual Optimization to Compiler

## Learning Objective

Confidently remove manual memoization hooks (`useMemo`, `useCallback`) and allow the React 19 Compiler to handle optimization automatically.

## Why This Matters

Over-using `useMemo` and `useCallback` clutters components, introduces bugs via incorrect dependency arrays, and can sometimes even make performance worse. The React Compiler understands JavaScript deeply and can perform more sophisticated memoization than is practical to do by hand.

## The Migration Mindset

The goal is to write simple, clear, and idiomatic React code. The compiler is optimized for this. Your job is to _remove_ code.

### Example: A Memoized Component

Consider a component that passes a callback and a computed value to a child component. The "optimized" React 18 version is often verbose.

```jsx
// BEFORE: React 18 with manual memoization
import { useState, useMemo, useCallback } from "react";
import { ExpensiveChild } from "./ExpensiveChild";

function ProductPage({ product }) {
  const [quantity, setQuantity] = useState(1);

  // Memoize an expensive calculation
  const shippingInfo = useMemo(() => {
    return calculateShipping(product.weight, quantity);
  }, [product.weight, quantity]);

  // Memoize a callback function
  const handleAddToCart = useCallback(() => {
    addToCart(product.id, quantity, shippingInfo);
  }, [product.id, quantity, shippingInfo]);

  return (
    <div>
      <h1>{product.name}</h1>
      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(Number(e.target.value))}
      />
      <ExpensiveChild
        shippingInfo={shippingInfo}
        onAddToCart={handleAddToCart}
      />
    </div>
  );
}
```

```jsx
// AFTER: React 19 with the Compiler
import { useState } from "react";
import { ExpensiveChild } from "./ExpensiveChild";

// Assumes the React Compiler is enabled in your build process
function ProductPage({ product }) {
  const [quantity, setQuantity] = useState(1);

  // Just calculate the value directly
  const shippingInfo = calculateShipping(product.weight, quantity);

  // Just define the function directly
  const handleAddToCart = () => {
    addToCart(product.id, quantity, shippingInfo);
  };

  return (
    <div>
      <h1>{product.name}</h1>
      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(Number(e.target.value))}
      />
      <ExpensiveChild
        shippingInfo={shippingInfo}
        onAddToCart={handleAddToCart}
      />
    </div>
  );
}
```

### How It Works

The React Compiler is a build tool. When it processes the "After" code, it analyzes the dependencies and effectively rewrites it to be highly optimized, similar to what `useMemo` and `useCallback` did, but more precisely. It memoizes `shippingInfo`, `handleAddToCart`, and even the JSX passed to `ExpensiveChild` automatically, preventing `ExpensiveChild` from re-rendering if its props haven't _semantically_ changed.

### Production Perspective

To migrate, you first need to enable the compiler in your build tool (e.g., Vite, Next.js). Once enabled, the process is to systematically delete `useMemo` and `useCallback` hooks. Start with components that are performance bottlenecks. Use the React DevTools Profiler to confirm that your components are not re-rendering unnecessarily after the change. In almost all cases, the compiler's output will be as good or better than manual memoization.

## Class Components to Hooks

## Learning Objective

Translate the lifecycle methods and state management of a class component into a modern function component using hooks.

## Why This Matters

Function components with hooks are the modern standard in React. They are more composable, easier to test, and allow for logic reuse through custom hooks. Migrating legacy class components makes your codebase more consistent and easier for new developers to understand.

## Mapping Class Features to Hooks

| Class Component Feature      | Hook Equivalent                                 |
| ---------------------------- | ----------------------------------------------- |
| `this.state` / `constructor` | `useState` or `useReducer`                      |
| `this.setState()`            | `setState()` function from `useState`           |
| `componentDidMount`          | `useEffect(() => { ... }, [])`                  |
| `componentDidUpdate`         | `useEffect(() => { ... }, [dep1, dep2])`        |
| `componentWillUnmount`       | `useEffect(() => { return () => { ... } }, [])` |
| `this.props`                 | `props` passed as the first argument            |
| Instance methods             | Functions defined inside the component          |

### Example: Data Fetching Component

```jsx
// BEFORE: Class Component
import React from "react";

class UserProfile extends React.Component {
  state = {
    user: null,
    loading: true,
    error: null,
  };

  componentDidMount() {
    this.fetchUser(this.props.userId);
  }

  componentDidUpdate(prevProps) {
    if (prevProps.userId !== this.props.userId) {
      this.fetchUser(this.props.userId);
    }
  }

  fetchUser = async (userId) => {
    this.setState({ loading: true, error: null });
    try {
      const response = await fetch(`/api/users/${userId}`);
      const data = await response.json();
      this.setState({ user: data, loading: false });
    } catch (e) {
      this.setState({ error: e, loading: false });
    }
  };

  render() {
    const { user, loading, error } = this.state;

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error!</div>;
    if (!user) return null;

    return <h1>{user.name}</h1>;
  }
}
```

```jsx
// AFTER: Function Component with Hooks
import { useState, useEffect } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      setError(null);
      try {
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        setUser(data);
      } catch (e) {
        setError(e);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
    // This effect re-runs whenever the userId prop changes
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error!</div>;
  if (!user) return null;

  return <h1>{user.name}</h1>;
}
```

### Production Perspective

The hooks version is more concise and the logic is co-located. The `useEffect` hook clearly defines that the data fetching logic depends _only_ on `userId`. This prevents a large class of bugs common in `componentDidUpdate` where a dependency was missed. The migration can be done component by component, allowing for an incremental update of your application.

## JavaScript to TypeScript

## Learning Objective

Follow a step-by-step process to introduce TypeScript into an existing JavaScript React project.

## Why This Matters

TypeScript provides static type checking, which catches errors before your code runs. It improves code quality, maintainability, and developer experience through better autocompletion and self-documenting code.

## Incremental Migration Strategy

1.  **Install Dependencies**:

    ```bash
    npm install --save-dev typescript @types/react @types/react-dom @types/node
    ```

2.  **Create `tsconfig.json`**:
    Run `npx tsc --init` in your project root. This creates a `tsconfig.json` file. Key settings for a React project include:

    ```json
    {
      "compilerOptions": {
        "target": "ES2020",
        "lib": ["dom", "dom.iterable", "esnext"],
        "jsx": "react-jsx", // or "preserve"
        "module": "esnext",
        "moduleResolution": "node",
        "allowJs": true, // IMPORTANT: Allows JS and TS files to co-exist
        "skipLibCheck": true,
        "esModuleInterop": true,
        "strict": true, // Recommended for new TS projects
        "forceConsistentCasingInFileNames": true,
        "isolatedModules": true,
        "noEmit": true // If your build tool (Vite/Next) handles transpilation
      },
      "include": ["src"]
    }
    ```

3.  **Rename Files**:
    Start with one simple component. Rename its file from `.js` or `.jsx` to `.tsx`. Your application will continue to work thanks to `"allowJs": true`.

4.  **Add Types**:
    Fix the type errors TypeScript now reports. The most common task is typing component props.

### Example: Typing a Component

```jsx
// BEFORE: JavaScript Component (src/components/Button.jsx)
export function Button({ children, onClick, variant = "primary" }) {
  const baseStyle = "px-4 py-2 rounded";
  const variantStyle =
    variant === "primary" ? "bg-blue-500 text-white" : "bg-gray-200 text-black";

  return (
    <button className={`${baseStyle} ${variantStyle}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

```jsx
// AFTER: TypeScript Component (src/components/Button.tsx)
import type { ReactNode, MouseEventHandler } from 'react';

type ButtonProps = {
  children: ReactNode;
  onClick: MouseEventHandler<HTMLButtonElement>;
  variant?: 'primary' | 'secondary';
};

export function Button({
  children,
  onClick,
  variant = 'primary',
}: ButtonProps) {
  const baseStyle = 'px-4 py-2 rounded';
  const variantStyle =
    variant === 'primary'
      ? 'bg-blue-500 text-white'
      : 'bg-gray-200 text-black';

  return (
    <button className={`${baseStyle} ${variantStyle}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

### Production Perspective

Migrate your codebase from the "leaves" of your component tree (simple components like `Button`, `Input`) inwards towards the "root" (complex pages, layouts). This bottom-up approach allows you to build a foundation of typed components, making it easier to type the more complex components that consume them.

## Create React App to Vite

## Learning Objective

Migrate a React application from Create React App (CRA) to Vite to take advantage of its superior development experience.

## Why This Matters

Vite provides a significantly faster development server and Hot Module Replacement (HMR) by leveraging native ES modules in the browser. This results in near-instantaneous startup times and updates, drastically improving developer productivity.

## Step-by-Step Migration Guide

1.  **Install Vite Dependencies**:

    ```bash
    npm install --save-dev vite @vitejs/plugin-react
    ```

2.  **Remove `react-scripts`**:

```bash
npm uninstall react-scripts
```

3.  **Update `package.json` Scripts**:
    Replace the `scripts` section with Vite commands.

```json

// BEFORE
"scripts": {
  "start": "react-scripts start",
  "build": "react-scripts build",
  "test": "react-scripts test",
  "eject": "react-scripts eject"
},
```

```json

// AFTER
"scripts": {
  "dev": "vite", // or "start"
  "build": "vite build",
  "preview": "vite preview"
},

```

4.  **Create `vite.config.js`**:
    In your project root, create a `vite.config.js` file.

```javascript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
});
```

5.  **Move `index.html`**:
    Move `public/index.html` to the root of your project.

6.  **Update `index.html`**:
    - Remove all `%PUBLIC_URL%` placeholders. Paths should be relative from the root (e.g., `/favicon.ico`).
    - Add the script entry point at the end of the `<body>` tag.

    ```html
    <!-- Add this script tag -->
    <script type="module" src="/src/index.jsx"></script>
    ```

7.  **Handle Environment Variables**:
    If you use environment variables, rename them from `REACT_APP_` to `VITE_`. Access them in your code as `import.meta.env.VITE_YOUR_VAR`.

### Production Perspective

This migration primarily affects your development and build tooling, not your React component code. The biggest benefit is the dramatic speed increase during development. The process is well-documented and typically takes less than an hour for most projects. The productivity gains for your team will pay for that time investment very quickly.

## Pages Router to App Router (Next.js)

## Learning Objective

Understand the key differences between the Next.js Pages Router and App Router and migrate a basic page.

## Why This Matters

The App Router is the foundation for React 19's server-first features like Server Components, Streaming, and Actions within Next.js. Migrating enables you to build more performant and interactive applications by leveraging the server more effectively.

## Core Conceptual Shifts

| Feature             | Pages Router (`pages/`)                    | App Router (`app/`)                                                                    |
| ------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------- |
| **Component Model** | All components are client-side by default. | Components are **Server Components** by default. Use `'use client'` for interactivity. |
| **File Convention** | `pages/posts/[id].js` is a page.           | `app/posts/[id]/page.js` is the page UI.                                               |
| **Layouts**         | Shared layout in `_app.js`.                | Nested layouts via `layout.js` files in route segments.                                |
| **Data Fetching**   | `getServerSideProps`, `getStaticProps`.    | `fetch` in async Server Components.                                                    |

### Example: A Dynamic Post Page

```jsx
// BEFORE: Pages Router (pages/posts/[id].js)
import { fetchPostById } from "../../lib/api";

export default function PostPage({ post }) {
  // This is a client-side component
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}

export async function getServerSideProps(context) {
  const { id } = context.params;
  const post = await fetchPostById(id);
  return {
    props: { post },
  };
}
```

```jsx
// AFTER: App Router (app/posts/[id]/page.js)
import { fetchPostById } from "../../lib/api";

// This is a Server Component by default
export default async function PostPage({ params }) {
  // Data is fetched directly inside the component
  const { id } = params;
  const post = await fetchPostById(id);

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}

// Optional: Add loading and error boundaries
// app/posts/[id]/loading.js
// app/posts/[id]/error.js
```

### The `'use client'` Boundary

If a component needs interactivity (e.g., `useState`, `useEffect`, event handlers), you must add the `'use client'` directive at the top of the file. This marks it and all its children as Client Components.

```jsx
"use client"; // This is now a Client Component

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### Production Perspective

Migration can be done incrementally. The App Router can co-exist with the Pages Router. You can start by migrating a few pages to the new paradigm. The main benefit is moving data fetching and rendering logic to the server, which results in less JavaScript being sent to the client, faster initial page loads, and a better user experience.
