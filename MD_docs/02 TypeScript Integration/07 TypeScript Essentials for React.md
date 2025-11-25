# Chapter 7: TypeScript Essentials for React

## Why TypeScript is worth the initial friction

## The Promise and the Price

TypeScript is JavaScript with static types. This means you declare the "shape" of your data, and TypeScript's compiler checks your code against these shapes before it ever runs.

The "price" is an initial learning curve. You have to write a bit more code and learn some new syntax. The "promise" is a massive reduction in a specific, painful class of bugs: runtime errors caused by incorrect data types. It transforms these bugs from runtime surprises for your users into compile-time errors for you, the developer.

Let's see a classic JavaScript bug that TypeScript eradicates.

### The Silent Failure of JavaScript

Imagine a simple component that displays a user's initials.

**Anchor Example**: We'll start with a `UserAvatar` component written in plain JavaScript. This component will be our reference point as we explore TypeScript.

```jsx
// src/components/UserAvatar.jsx

function UserAvatar({ user }) {
  // Assumes 'user' is an object with 'firstName' and 'lastName'
  const firstNameInitial = user.firstName.charAt(0);
  const lastNameInitial = user.lastName.charAt(0);

  return (
    <div className="avatar">
      {firstNameInitial}{lastNameInitial}
    </div>
  );
}

export default UserAvatar;
```

This code works perfectly if you use it correctly:

```jsx
// src/app/page.jsx
import UserAvatar from '@/components/UserAvatar';

export default function HomePage() {
  const currentUser = { firstName: 'Jane', lastName: 'Doe' };
  return <UserAvatar user={currentUser} />; // Renders "JD"
}
```

But what happens when the data isn't what we expect? Let's say an API call returns a user object, but the `lastName` is `null`.

```jsx
// src/app/page.jsx (with problematic data)
import UserAvatar from '@/components/UserAvatar';

export default function HomePage() {
  // Data from a hypothetical API that sometimes returns nulls
  const userFromApi = { firstName: 'John', lastName: null };
  return <UserAvatar user={userFromApi} />;
}
```

Let's run this and see what happens.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The entire application crashes. The user sees a white screen, or Next.js's error overlay.

**Browser Console Output**:
```
TypeError: Cannot read properties of null (reading 'charAt')
  at UserAvatar (UserAvatar.jsx:5:1)
  at renderWithHooks (...)
  ...
```

**Let's parse this evidence**:

1.  **What the user experiences**: The app is broken. They can't see their page.
2.  **What the console reveals**: The error `TypeError: Cannot read properties of null (reading 'charAt')` is explicit. It tells us we tried to call the `.charAt()` method on something that was `null`. The stack trace points directly to line 5 of `UserAvatar.jsx`: `const lastNameInitial = user.lastName.charAt(0);`.
3.  **Root cause identified**: The component implicitly assumes `user.lastName` will always be a string, but it received `null`.
4.  **Why the current approach can't solve this**: Plain JavaScript has no built-in mechanism to enforce the "shape" of the `user` prop. The error only occurs when the code *runs* with bad data. We could add manual checks (e.g., `if (user &amp;&amp; user.lastName)`), but this is tedious, error-prone, and doesn't prevent a developer from passing a completely wrong data type, like a number.
5.  **What we need**: A way to declare our expectations for the `user` prop and have our tools check for violations *before* the code ever reaches the browser. This is precisely what TypeScript provides.

TypeScript acts as a contract. You define the contract for your component's props, and the TypeScript compiler ensures that anyone using your component adheres to that contract. This moves error detection from runtime (in the user's browser) to compile-time (in your code editor), which is fundamentally safer, faster, and cheaper to fix.

## Setting up TypeScript in your React project

## Integrating TypeScript

The best way to start a new React/Next.js project with TypeScript is to let the framework handle the setup.

### Creating a New Project

When you create a new Next.js app, it will ask if you want to use TypeScript. Simply answer "Yes".

```bash
npx create-next-app@latest my-typescript-app
# ✔ Would you like to use TypeScript? … No / Yes
# Select Yes
```

This command scaffolds a new project with all the necessary configurations for TypeScript.

### Key Files and Concepts

The setup process creates a few important files and establishes some conventions:

1.  **`tsconfig.json`**: This is the TypeScript compiler configuration file. It lives in the root of your project and tells the compiler how to treat your code. For now, the default settings are excellent. Key options include:
    *   `"target": "es5"`: Compiles your modern JavaScript down to an older version (ES5) for better browser compatibility.
    *   `"lib": ["dom", "dom.iterable", "esnext"]`: Includes standard type definitions for browser DOM APIs and modern JavaScript features.
    *   `"jsx": "preserve"`: Tells TypeScript to leave JSX syntax alone so that Next.js's compiler (Babel) can handle it.
    *   `"strict": true`: Enables a wide range of type-checking behaviors that result in more robust programs. **Always leave this on.**
    *   `"paths": { "@/*": ["./src/*"] }`: Configures the `@/` path alias for cleaner imports.

2.  **File Extensions**: Your component files will now use the `.tsx` extension instead of `.js` or `.jsx`. This tells TypeScript that the file contains JSX syntax. Non-component TypeScript files (like utility functions or type definitions) use the `.ts` extension.

3.  **Type Definitions for Libraries**: Libraries like React and Node.js are written in JavaScript. To use them in a TypeScript project, we need type definition files that describe their APIs. These are typically hosted on a community-managed repository called DefinitelyTyped and are installed as `@types/*` packages. `create-next-app` handles this for you, adding `@types/react`, `@types/node`, and `@types/react-dom` to your `package.json`.

With this setup, you're ready to start writing typed React components.

## Typing props, state, and events

## The Core Workflow: Props, State, and Events

This is where you'll spend 90% of your time with TypeScript in React. We'll build a more complex component and progressively add types to make it robust.

**Anchor Example**: A `UserProfileCard` component. It will receive a `userId`, fetch user data, manage loading and error states, and allow an action via a button click.

**Project Structure**:
```
src/
├── components/
│   └── UserProfileCard.tsx  ← Our reference implementation
└── app/
    └── page.tsx
```

### Iteration 0: The Fragile JavaScript Implementation

First, let's build the component in plain JavaScript (`.jsx`) to see its weaknesses. It fetches user data based on a `userId` prop.

```jsx
// src/components/UserProfileCard.jsx (JavaScript version)
import { useState, useEffect } from 'react';

function UserProfileCard({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchUser() {
      try {
        setLoading(true);
        // This is a mock fetch. In a real app, it would be a network request.
        const response = await new Promise(resolve => setTimeout(() => resolve({
          id: userId,
          name: 'Jane Doe',
          email: 'jane.doe@example.com',
          role: 'Admin'
        }), 1000));
        setUser(response);
      } catch (err) {
        setError('Failed to fetch user.');
      } finally {
        setLoading(false);
      }
    }
    fetchUser();
  }, [userId]);

  const handleDeactivate = (event) => {
    // Let's say we want to get a data attribute from the button
    console.log('Deactivating user:', event.target.dataset.userId);
  };

  if (loading) return <p>Loading profile...</p>;
  if (error) return <p style={{ color: 'red' }}>{error}</p>;

  return (
    <div className="card">
      <h2>{user.name}</h2>
      <p>Email: {user.email}</p>
      <p>Role: {user.role}</p>
      <button onClick={handleDeactivate} data-user-id={user.id}>
        Deactivate
      </button>
    </div>
  );
}

export default UserProfileCard;
```

This component works, but it's a minefield of potential runtime errors. Let's convert it to TypeScript and fix them one by one.

### Iteration 1: Typing Component Props

First, rename the file from `UserProfileCard.jsx` to `UserProfileCard.tsx`. Immediately, your editor might show some squiggly lines.

**Current Limitation**: The `userId` prop has an implicit `any` type. This means we could pass anything to it—a number, an object, `null`—and TypeScript wouldn't complain. This defeats the purpose of using TypeScript.

**New Scenario**: What happens if a developer uses our component but forgets the `userId` prop or passes the wrong type?

```tsx
// src/app/page.tsx (Incorrect usage)
import UserProfileCard from '@/components/UserProfileCard';

export default function HomePage() {
  // Scenario 1: Forgot the prop
  // return <UserProfileCard />;

  // Scenario 2: Passed a number instead of a string
  return <UserProfileCard userId={123} />;
}
```

### Diagnostic Analysis: Reading the Failure (Props)

**Browser Behavior**:
The component might render the loading state forever or crash, depending on how the `fetch` logic handles a non-string `userId`. In our mock example, it might appear to work, but a real API would likely fail.

**Terminal Output** (from the Next.js dev server):
```bash
# With strict mode, TypeScript will complain about the implicit 'any' type
src/components/UserProfileCard.tsx:4:29 - error TS7031:
Binding element 'userId' implicitly has an 'any' type.

4 function UserProfileCard({ userId }) {
                              ~~~~~~
```

**Let's parse this evidence**:

1.  **What the user experiences**: Unpredictable behavior. The component might not load data correctly.
2.  **What the console reveals**: TypeScript is telling us directly: "I don't know what `userId` is supposed to be. You need to define its type."
3.  **Root cause identified**: The component's props are not explicitly typed.
4.  **What we need**: A way to define a "contract" for `UserProfileCard`'s props.

**Technique Introduced**: Using `interface` or `type` to define the shape of props.

-   `interface`: An extensible way to define the shape of an object. Can be implemented by classes or extended by other interfaces.
-   `type`: A more general type alias. Can represent primitives, unions, tuples, and objects.

For component props, they are largely interchangeable. A common convention is to use `interface` for public API shapes (like props) and `type` for more specific, internal types.

**Solution Implementation**: Let's define an interface for our props.

**Before**:
```tsx
// src/components/UserProfileCard.tsx
function UserProfileCard({ userId }) {
  // ...
}
```

**After**:
```tsx
// src/components/UserProfileCard.tsx
import { useState, useEffect } from 'react';

// Define the shape of the props object
interface UserProfileCardProps {
  userId: string;
}

// Apply the type to the component's props
function UserProfileCard({ userId }: UserProfileCardProps) {
  // ... rest of the component is the same for now
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // ...
  }, [userId]);

  // ...
}

export default UserProfileCard;
```

**Verification**: Now, let's go back to our incorrect usage in `page.tsx`.

**Editor (VS Code) Evidence**:
When trying to use `<UserProfileCard userId={123} />`, you'll see a red squiggly line under `userId` with the error:
`Type 'number' is not assignable to type 'string'.`

When trying to use `<UserProfileCard />`, you'll see an error on the component name:
`Property 'userId' is missing in type '{}' but required in type 'UserProfileCardProps'.`

**Improvement**: The error is now caught *in the editor* before we even save the file. We've moved the bug from a potential runtime issue to an immediate compile-time error.

### Iteration 2: Typing State

**Current Limitation**: Our state variables (`user`, `loading`, `error`) are inferred by TypeScript from their initial values.
- `loading` is inferred as `boolean`. This is correct.
- `error` is inferred as `string | null`. This is also correct.
- `user` is inferred as `any` or `null` because it starts as `null`. When we call `setUser(response)`, TypeScript doesn't know the shape of the `response` object. This means we get no autocompletion for `user.name` and no safety if we try to access a non-existent property.

**New Scenario**: What happens if we make a typo and try to access `user.username` instead of `user.name`?

```tsx
// Inside UserProfileCard.tsx
// ...
return (
  <div className="card">
    <h2>{user.username}</h2> {/* Typo! Should be user.name */}
    {/* ... */}
  </div>
);
```

### Diagnostic Analysis: Reading the Failure (State)

**Browser Behavior**:
The component renders without the user's name. The `<h2>` tag is empty. If `user` were null, the app would crash.

**Browser Console Output**:
If `user` is not null, you see no error. The property access `user.username` just returns `undefined`, which React renders as nothing. This is a silent bug.

**Let's parse this evidence**:

1.  **What the user experiences**: Missing data on the screen.
2.  **What the console reveals**: Nothing. This is the worst kind of bug—one that fails silently.
3.  **Root cause identified**: TypeScript has no information about the shape of the `user` object in our state, so it cannot validate property access.
4.  **What we need**: A way to tell `useState` what kind of data it will hold.

**Technique Introduced**: Using generics with `useState` to specify the state's type. The syntax is `useState<Type>(initialValue)`.

**Solution Implementation**: We'll define a `User` type and use it to type our state.

**Before**:
```tsx
// src/components/UserProfileCard.tsx
const [user, setUser] = useState(null);
// ...
<h2>{user.name}</h2>
```

**After**:
```tsx
// src/components/UserProfileCard.tsx

// Define the shape of our user data
interface User {
  id: string;
  name: string;
  email: string;
  role: 'Admin' | 'User' | 'Guest'; // Union type for specific roles
}

interface UserProfileCardProps {
  userId: string;
}

function UserProfileCard({ userId }: UserProfileCardProps) {
  // Tell useState it will hold a User object or null
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null); // Explicitly typing for clarity

  useEffect(() => {
    async function fetchUser() {
      try {
        setLoading(true);
        // Mock fetch now returns a typed object
        const response: User = await new Promise(resolve => setTimeout(() => resolve({
          id: userId,
          name: 'Jane Doe',
          email: 'jane.doe@example.com',
          role: 'Admin'
        }), 1000));
        setUser(response);
      } catch (err) {
        setError('Failed to fetch user.');
      } finally {
        setLoading(false);
      }
    }
    fetchUser();
  }, [userId]);

  // ...
  
  // Now, TypeScript will check property access on `user`
  if (user) { // This check is now enforced by TypeScript!
    return (
      <div className="card">
        <h2>{user.name}</h2>
        {/* ... */}
      </div>
    );
  }
  
  return null; // Or some other fallback
}
```

**Verification**: With the typo `user.username`, the editor immediately shows an error:
`Property 'username' does not exist on type 'User'. Did you mean 'name'?`

TypeScript is not only catching the error but also suggesting the fix! Furthermore, because `user` can be `null`, TypeScript will now force you to check if `user` exists before you can access its properties, preventing the "cannot read properties of null" runtime error.

### Iteration 3: Typing Events

**Current Limitation**: In our `handleDeactivate` function, the `event` parameter is implicitly `any`. We have no type safety or autocompletion when working with it.

**New Scenario**: What if we try to access a property that doesn't exist on a button click event, like `event.target.value`? In JavaScript, this would return `undefined`. In TypeScript, we can do better.

```tsx
// Inside UserProfileCard.tsx
const handleDeactivate = (event) => { // `event` is implicitly 'any'
  console.log('Input value:', event.target.value); // Buttons don't have a `value` property like inputs do
};
```

### Diagnostic Analysis: Reading the Failure (Events)

**Editor Behavior**:
With the default `tsconfig.json`, TypeScript will flag `event` as having an implicit `any` type. If we were to manually type it as `any`, we would get no help.

**Let's parse this evidence**:

1.  **Root cause identified**: The event handler's parameter is not typed.
2.  **What we need**: A way to tell TypeScript the exact type of event we expect.

**Technique Introduced**: Using React's synthetic event types. React wraps the browser's native events in a synthetic event system for cross-browser consistency. TypeScript provides types for these.

Common event types include:
-   `React.MouseEvent`: For `onClick`, `onMouseDown`, etc.
-   `React.ChangeEvent`: For `onChange` on form inputs.
-   `React.FormEvent`: For `onSubmit` on forms.
-   `React.KeyboardEvent`: For `onKeyDown`, `onKeyUp`, etc.

These types are also generic. You can specify the HTML element the event originated from for even more specific typing, e.g., `React.MouseEvent<HTMLButtonElement>`.

**Solution Implementation**: Let's apply the correct type to our handler.

**Before**:
```tsx
const handleDeactivate = (event) => {
  console.log('Deactivating user:', event.target.dataset.userId);
};
```

**After**:
```tsx
// Inside UserProfileCard.tsx
const handleDeactivate = (event: React.MouseEvent<HTMLButtonElement>) => {
  // Now `event` is fully typed.
  // `event.currentTarget` is known to be an HTMLButtonElement.
  console.log('Deactivating user:', event.currentTarget.dataset.userId);
};
```
*Note: It's often safer to use `event.currentTarget` instead of `event.target`. `currentTarget` is always the element the event listener is attached to, while `target` can be a child element inside it.*

**Verification**:
-   **Autocompletion**: When you type `event.`, your editor will now show a list of valid properties for a mouse event (`clientX`, `clientY`, `altKey`, etc.).
-   **Type Safety**: If you try to access `event.currentTarget.value`, TypeScript will show an error: `Property 'value' does not exist on type 'HTMLButtonElement'`.

### Final Implementation (Iteration 3)
Here is our fully-typed, robust component.

```tsx
// src/components/UserProfileCard.tsx

import { useState, useEffect } from 'react';

// 1. Define types for props and state
interface User {
  id: string;
  name: string;
  email: string;
  role: 'Admin' | 'User' | 'Guest';
}

interface UserProfileCardProps {
  userId: string;
}

function UserProfileCard({ userId }: UserProfileCardProps) {
  // 2. Type state with generics
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchUser() {
      try {
        setLoading(true);
        setError(null);
        const response: User = await new Promise(resolve => setTimeout(() => resolve({
          id: userId,
          name: 'Jane Doe',
          email: 'jane.doe@example.com',
          role: 'Admin'
        }), 1000));
        setUser(response);
      } catch (err) {
        setError('Failed to fetch user.');
      } finally {
        setLoading(false);
      }
    }
    fetchUser();
  }, [userId]);

  // 3. Type event handlers
  const handleDeactivate = (event: React.MouseEvent<HTMLButtonElement>) => {
    console.log('Deactivating user:', event.currentTarget.dataset.userId);
    alert(`Deactivated user ${user?.name}`);
  };

  if (loading) return <p>Loading profile...</p>;
  if (error) return <p style={{ color: 'red' }}>{error}</p>;
  
  // TypeScript enforces this check because `user` can be `null`
  if (!user) return <p>No user data found.</p>;

  return (
    <div className="card" style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px' }}>
      <h2>{user.name}</h2>
      <p>Email: {user.email}</p>
      <p>Role: {user.role}</p>
      <button onClick={handleDeactivate} data-user-id={user.id}>
        Deactivate
      </button>
    </div>
  );
}

export default UserProfileCard;
```

## Generic components

## Building Reusable, Type-Safe Components

Sometimes you want to create a component that can work with a variety of data types while still maintaining type safety. This is where generics come in.

**Anchor Example**: A `DataTable` component. We want it to be able to display a list of users, products, orders, or anything else, without having to rewrite the component for each data type.

### Iteration 0: The Single-Purpose Component

Let's start with a `UserDataTable` that only works for our `User` type.

```tsx
// src/components/UserDataTable.tsx

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserDataTableProps {
  items: User[];
}

function UserDataTable({ items }: UserDataTableProps) {
  return (
    <table>
      <thead>
        <tr>
          <th>ID</th>
          <th>Name</th>
          <th>Email</th>
        </tr>
      </thead>
      <tbody>
        {items.map((item) => (
          <tr key={item.id}>
            <td>{item.id}</td>
            <td>{item.name}</td>
            <td>{item.email}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

This works perfectly for users. But what if we now need a table for products?

```typescript
interface Product {
  sku: string;
  name: string;
  price: number;
}
```

**Current Limitation**: Our `UserDataTable` is completely coupled to the `User` type. To create a product table, we'd have to copy, paste, and rename everything to `ProductDataTable`, violating the Don't Repeat Yourself (DRY) principle.

**What we need**: A way to create a `DataTable` component that is agnostic about the *type* of data it displays, but still understands the *shape* of that data enough to render it.

### Iteration 1: Creating a Generic `DataTable`

**Technique Introduced**: Generic Components with `<T>`.

A generic component uses a type parameter (conventionally `T`, for Type) as a placeholder for a specific type that will be provided when the component is used.

We also need a way to tell our component what to render for each column. We can't hardcode `item.name` and `item.email` anymore. We'll pass in a configuration object that defines the columns.

**Solution Implementation**: Let's refactor `UserDataTable` into a generic `DataTable`.

**Before**: The rigid `UserDataTable`.

**After**: The flexible, generic `DataTable`.

```tsx
// src/components/DataTable.tsx

import React from 'react';

// 1. Define a generic Column configuration type
export interface ColumnDefinition<T> {
  key: keyof T; // The key of the data object
  header: string; // The text to display in the table header
  render?: (item: T) => React.ReactNode; // Optional custom render function
}

// 2. Define the generic props for the DataTable
// T is a placeholder for our data type (e.g., User, Product)
// We add a constraint: T must be an object with an `id` property
interface DataTableProps<T extends { id: string | number }> {
  items: T[];
  columns: ColumnDefinition<T>[];
}

// 3. Define the generic component
// We use <T extends { id: string | number }> to ensure `item.id` exists for the `key` prop
export function DataTable<T extends { id: string | number }>({
  items,
  columns,
}: DataTableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {items.map((item) => (
          <tr key={item.id}>
            {columns.map((col) => (
              <td key={String(col.key)}>
                {col.render ? col.render(item) : (item[col.key] as React.ReactNode)}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### Verification: Using the Generic Component

Now we can use this single `DataTable` component for any data type, with full type safety.

Let's define our data types and column configurations.

```tsx
// src/app/page.tsx
import { DataTable, ColumnDefinition } from '@/components/DataTable';

// Define our data types
interface User {
  id: string;
  name: string;
  email: string;
  isActive: boolean;
}

interface Product {
  id: number;
  name: string;
  price: number;
  stock: number;
}

// Sample data
const users: User[] = [
  { id: '1', name: 'Alice', email: 'alice@example.com', isActive: true },
  { id: '2', name: 'Bob', email: 'bob@example.com', isActive: false },
];

const products: Product[] = [
  { id: 101, name: 'Laptop', price: 1200, stock: 50 },
  { id: 102, name: 'Mouse', price: 25, stock: 200 },
];

// Define columns for the User table
// TypeScript ensures `key` is a valid key of `User`
const userColumns: ColumnDefinition<User>[] = [
  { key: 'id', header: 'ID' },
  { key: 'name', header: 'User Name' },
  { key: 'email', header: 'Email Address' },
  { 
    key: 'isActive', 
    header: 'Status',
    render: (user) => (
      <span style={{ color: user.isActive ? 'green' : 'red' }}>
        {user.isActive ? 'Active' : 'Inactive'}
      </span>
    )
  },
];

// Define columns for the Product table
const productColumns: ColumnDefinition<Product>[] = [
  { key: 'id', header: 'SKU' },
  { key: 'name', header: 'Product Name' },
  { 
    key: 'price', 
    header: 'Price',
    render: (product) => `$${product.price.toFixed(2)}`
  },
  { key: 'stock', header: 'In Stock' },
];

export default function HomePage() {
  return (
    <div>
      <h1>User Management</h1>
      <DataTable items={users} columns={userColumns} />

      <h1 style={{ marginTop: '40px' }}>Product Inventory</h1>
      <DataTable items={products} columns={productColumns} />
    </div>
  );
}
```

**Improvement**:
-   **Reusability**: We have one `DataTable` component that can render any kind of data.
-   **Type Safety**: When defining `userColumns`, TypeScript knows that `T` is `User`, so it will only allow `'id'`, `'name'`, `'email'`, or `'isActive'` for the `key` property. If you typed `key: 'username'`, you would get a compile-time error.
-   **Flexibility**: The `render` function in the column definition gives us full control over how data is displayed, and it is also fully typed—the `user` parameter inside `render` is correctly typed as `User`.

Generics are a powerful tool for creating flexible, reusable, and type-safe components, forming the foundation of many component libraries.

## The 20% of TypeScript that gives you 80% of the value

## A Practical Cheat Sheet

You don't need to master every feature of TypeScript to be effective. Focusing on a few core concepts will prevent the vast majority of common bugs. Here is the essential toolkit.

### 1. Basic Types
Always use the most specific type possible. Avoid `any`.

| Type          | Usage                                       | Example                               |
| ------------- | ------------------------------------------- | ------------------------------------- |
| `string`      | Textual data                                | `const name: string = 'Alice';`       |
| `number`      | Numeric values                              | `const age: number = 30;`             |
| `boolean`     | True/false values                           | `const isAdmin: boolean = true;`      |
| `null`        | Intentional absence of a value              | `const user: User | null = null;`      |
| `undefined`   | Uninitialized variables                     | `let city: string | undefined;`         |
| `string[]`    | An array of strings                         | `const roles: string[] = ['admin'];`   |
| `any`         | **Avoid.** Opt-out of type checking.        | `let data: any;`                      |
| `unknown`     | A type-safe alternative to `any`.           | `let data: unknown;`                  |

### 2. Shaping Objects with `interface` and `type`
Use these to describe the shape of objects like props and state.

**`interface`**: Best for defining object shapes, especially for public APIs like component props. Can be extended.

```typescript
interface User {
  id: number;
  name: string;
  // Optional property
  avatarUrl?: string;
}

interface Admin extends User {
  permissions: string[];
}
```

**`type`**: More versatile. Can be used for objects, but also for unions, primitives, etc.

```typescript
type UserID = string | number;

type UserRole = 'Admin' | 'Editor' | 'Viewer';

type User = {
  id: UserID;
  role: UserRole;
};
```

### 3. Union Types (`|`)
When a value can be one of several types. This is extremely common for state that can be data, `null`, or `undefined`.

```typescript
// A variable that can be a string or a number
let id: string | number = '123';
id = 456; // This is valid

// State that holds data or is null before loading
const [user, setUser] = useState<User | null>(null);
```

### 4. Typing Functions and Events
Describe a function's arguments and return value.

**For regular functions**:

```typescript
const fetchUser = async (userId: string): Promise<User> => {
  // ... implementation
};
```

**For React event handlers**:

```tsx
const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
  // ...
};

const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
  console.log(event.target.value);
};

return (
  <>
    <button onClick={handleClick}>Click Me</button>
    <input onChange={handleChange} />
  </>
);
```

### 5. Core React Types

| Type                | Usage                                                              | Example                                                              |
| ------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| `React.ReactNode`   | Anything React can render: JSX, string, number, null, array.       | `interface Props { children: React.ReactNode; }`                     |
| `React.FC<Props>`   | A function component. Automatically types `children`.              | `const MyComponent: React.FC<Props> = ({ children }) => ...`         |
| `useState<T>`       | Typing state.                                                      | `const [count, setCount] = useState<number>(0);`                     |
| `useRef<T>`         | Typing refs.                                                       | `const inputRef = useRef<HTMLInputElement>(null);`                   |

### 6. Generics (`<T>`)
For creating reusable, type-safe functions and components.

```typescript
// A generic function
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

const firstNum = getFirst([1, 2, 3]); // Inferred as number
const firstStr = getFirst(['a', 'b']); // Inferred as string

// A generic component was covered in section 7.4
```

### 7. Key Utility Types
TypeScript comes with built-in "utility types" that help you transform other types.

| Utility Type        | Description                                                       | Example                                                              |
| ------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------------- |
| `Partial<T>`        | Makes all properties of `T` optional.                             | `const partialUser: Partial<User> = { name: 'Temp' };`               |
| `Required<T>`       | Makes all properties of `T` required.                             | `const fullUser: Required<User> = { id: 1, name: 'A', avatarUrl: '...' };` |
| `Omit<T, K>`        | Creates a type by picking all properties from `T` and removing `K`. | `type UserForCreation = Omit<User, 'id'>; // { name: string; ... }` |
| `Pick<T, K>`        | Creates a type by picking only the properties `K` from `T`.         | `type UserPreview = Pick<User, 'name' | 'avatarUrl'>;`               |

By focusing on these core concepts, you can add a powerful layer of safety and predictability to your React applications, catching bugs before they happen and making your codebase easier to understand and maintain.
